---
layout: post
title: "Beware of Variable Scope in Python 2"
date: 2013-10-21
tags: python, gotcha
description: "Python 2 handles variable scope loosely. You should especially pay attention of temporary variables in list comprehension."
---

If you've learned Python in a systematic way, you know that Python 2 has loose, if not poor, variable scope management. For example:

```python
for i in range(5):
    pass
print i # still exists!
```

You shall not be surprised as many dynamic languages do so (I've checked Ruby and PHP). And in 99% of cases, there's no problem with such behavior.

However, there's still 1% left, which might drive you mad.

## weird list comprehension

I'm taking an Artificial Intelligence course at present and a recent assignment is to implement various reinforcement learning algorithms, including [Q-Learning](http://en.wikipedia.org/wiki/Q_learning). A strange problem arises that the Q-learning implementation works well when using generator expression, but fails miserably when using list comprehension.

The scenario is like this:

```python
class QLearningAlgorithm(RLAlgorithm):
    """An implementation of Q-learning reinforcement learning algorithm."""

    def process_feedback(self, state, action, reward, new_state):
        if new_state is None:
            # current state is among end states
            V_new_state = 0.0 # the value of new_state
        else:
            possible_actions = self.get_actions(new_state)
            # this works
            V_new_state = max(self.computeQ(state, action) for action in possible_actions)
            # this doesn't work!
            # V_new_state = max([self.computeQ(state, action) for action in possible_actions])
        # code to update weights ...
```

I'll omit the theoretical stuff and let's focus on the lines with `max` function. If you know list comprehension and generator expression, using either one is equivalent to the other if fed to `max`. Having no clue, I checked the following possible factors:

1. Does calling some dependency method like `get_actions` cause a state change? No.
2. Does `max` handle `list` and generator differently? No.
3. Do we have to put generator/list comprehension outside? The same result.
4. (the guess is going wild…)

As someone with 4 years' experience in Python (at least my resume says so), I was completed challenged. Luckily, I finally figured out what's happening with [`dis`](http://docs.python.org/2/library/dis.html).

## the gotcha

I have to admit the mistake is a silly but hard to conceive one when you're focusing on fighting the theoretical complexity. And clever you may have already detected the problem.

Knowing the issue, let's consider a simpler case:

```python
def all_squares_smaller_than_v1(numbers, i):
    """Check if all squares of numbers are smaller than i."""
    max_square = max([i**2 for i in numbers])
    return max_square < i

def all_squares_smaller_than_v2(numbers, i):
    """Check if all squares of numbers are smaller than i."""
    max_square = max(i**2 for i in numbers)
    return max_square < i

# prints False
print all_squares_smaller_than_v1([1, 2, 3], 10)
# prints True
print all_squares_smaller_than_v2([1, 2, 3], 10)
```

Like the problem above, the only difference is list comprehension vs. generator expression. And sadly, the list comprehension version fails again.

Why? Let's `dis` it!

```python
import dis
import your_module

dis.dis(your_module.some_function)
```

The above is the basic usage of the `dis` module. And disassembling the byte code of the two function we get (v1 on the left and v2 on the right):

```
 41       0 LOAD_GLOBAL      0 (max)          | 46   0 LOAD_GLOBAL      0 (max)
          3 BUILD_LIST       0                |      3 LOAD_CONST       1 (<code object <genexpr> at 0x101275530, file "strange.py", line 46>)
          6 LOAD_FAST        0 (numbers)      |
          9 GET_ITER                          |
     >>  10 FOR_ITER        16 (to 29)        |      6 MAKE_FUNCTION    0
         13 STORE_FAST       1 (i)            |      9 LOAD_FAST        0 (numbers)
         16 LOAD_FAST        1 (i)            |     12 GET_ITER
         19 LOAD_CONST       1 (2)            |     13 CALL_FUNCTION    1
         22 BINARY_POWER                      |     16 CALL_FUNCTION    1
         23 LIST_APPEND      2                |     19 STORE_FAST       2 (max_square)
         26 JUMP_ABSOLUTE   10                |
     >>  29 CALL_FUNCTION    1                | 47  22 LOAD_FAST        2 (max_square)
         32 STORE_FAST       2 (max_square)   |     25 LOAD_FAST        1 (i)
                                              |     28 COMPARE_OP       0 (<)
 42      35 LOAD_FAST        2 (max_square)   |     31 RETURN_VALUE
         38 LOAD_FAST        1 (i)            |
         41 COMPARE_OP       0 (<)            |
         44 RETURN_VALUE
```

See? `all_squares_smaller_than_v1` overrides variable `i`, which happens to be a function parameter ([`STORE_FAST`](http://docs.python.org/2/library/dis.html#opcode-STORE_FAST) writes value on top of stack to specified **local variable**). The poor Python variable scope comes back!

`all_squares_smaller_than_v2` works fine as it does not involve any write to local variable. Actually, using generator expression creates a closure via [`MAKE_CLOSURE`](http://docs.python.org/2/library/dis.html#opcode-MAKE_CLOSURE) in more complex case. Let's make a little change to x and see this in action:

```python
def all_squares_smaller_than_v3(numbers, i):
    """Check if all squares of numbers are smaller than i."""
    square = lambda x: x**2
    max_square = max(square(i) for i in numbers)
    return max_square < i
```

In this version, a local variable is referenced in the generator function. If we disassemble this function, we get:

```
 51   0 LOAD_CONST       1 (<code object <lambda> at 0x10ff78db0, file "strange.py", line 51>)
      3 MAKE_FUNCTION    0
      6 STORE_DEREF      0 (square)

 52   9 LOAD_GLOBAL      0 (max)
     12 LOAD_CLOSURE     0 (square)
     15 BUILD_TUPLE      1
     18 LOAD_CONST       2 (<code object <genexpr> at 0x10ff78930, file "strange.py", line 52>)
     21 MAKE_CLOSURE     0
     24 LOAD_FAST        0 (numbers)
     27 GET_ITER
     28 CALL_FUNCTION    1
     31 CALL_FUNCTION    1
     34 STORE_FAST       2 (max_square)

 53  37 LOAD_FAST        2 (max_square)
     40 LOAD_FAST        1 (i)
     43 COMPARE_OP       0 (<)
     46 RETURN_VALUE
```

It's clear that only `square` is put in the closure because [`BUILD_TUPLE` ](http://docs.python.org/2/library/dis.html#opcode-BUILD_TUPLE) only takes 1 operand on top of the stack. It never modifies the parameter `i`.

## conclusion

I've also checked dict comprehension and set comprehension on Python 2.7.4 and it turns out they DO NOT alter local variable. So in Python 2 only list comprehension falls short! Plus, in Python 3 every type of comprehension is intact. I guess this is another incentive to move to Python 3.

So, lessons learned:

1. Python 3 rocks!
2. Prefer generator expression over list comprehension if they are equivalent.
3. Avoid naming the temporary variable(s) the same as any parameter if you have to use list comprehension.
4. `dis` always tells you the truth
