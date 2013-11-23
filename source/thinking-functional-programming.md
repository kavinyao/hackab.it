# Thinking Functional Programming

- pubdate: 2013-05-18
- tags: functional programming, thinking, tutorial, introduction
- description: A little, practical, contrived guide to help you understand functional programming.

---

I read Loup's article [From Imperative to Functional: how to make the leap][imp-to-func] several days ago. While it's an excellent article and inspires me on several aspects, the approach it takes is a little too obscure. Since I've always wanted to write a post about functional programming (FP), I think it's the time to write down my own (and yet another) little introduction to FP, a more practical version of Loup's article as well.

[imp-to-func]: http://loup-vaillant.fr/tutorials/from-imperative-to-functional.en

## functions as value

*Programs = algorithms + data structures.* CS students have been told this since their CS101. Whether they learn C or Java as their first language (or any other imperative language), an unspoken notion that `struct`s (or something equivalent) are data structure while functions/methods are algorithms is fixated in their minds. Textbooks on C will distinguish basic types from composite types from pointers, and from functions. However, in C you can actually use functions as parameters of another function or assign to a value with pointers. (BTW, let's forget Java, it's seriously paralysed in this context.) If you've learned Linux programming, you should know the signal handler register function `signal`:

```cpp
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

// type of the function
typedef void (*sighandler_t)(int);

void my_sigint_handler(int signum)
{
    printf("Received signal %d\n", signum);
}

int main()
{
    // assign function to a variable
    sighandler_t new_handler = my_sigint_handler;
    // signal takes a function as argument, and
    // returns another function
    sighandler_t old_handler = signal(SIGINT, new_handler);
    sleep(10); // try pressing CTRL-C
    return 0;
}
```

**If you can understand this example, you've understood that functions can indeed act as values.**

However, the scope of using functions as value in C is limited in:

1. you have to do it with pointers (which is sooooo machine-friendly)
2. you cannot create functions at runtime
3. functions are always sealed environments, i.e. you cannot access non-local variables

While the first point is trivial, the rest two are important. And we will demonstrate why along the way.

## FP is about abstraction

Let's start with an example. Say our client wants three interfaces with a list as parameter:

1. create a new list by adding 1 to all items in a list, and
2. create a new list by multiplying all items in a list by 2, and
3. create a new list by first multiplying all items by 3 then adding 4

Sounds simple? Let's program them in python*:

```python
def list_add_1(l):
    new_list = []
    for i in l:
        new_list.append(i+1)
    return new_list

def list_mul_2(l):
    new_list = []
    for i in l:
        new_list.append(i*2)
    return new_list

def list_m3_a4(l):
    new_list = []
    for i in l:
        new_list.append(i*3+4)
    return new_list

# client code
list_add_1([1, 2, 3]) # -> [2, 3, 4]
list_mul_2([1, 2, 3]) # -> [2, 4, 6]
list_m3_a4([1, 2, 3]) # -> [7, 10, 13]
```

\* All the following programs are in python. Python is known to be  simple and straight-forward, so relax if you haven't learned it before. Also, I'll give notes where python-specific syntax is used.

Now let's introduce the golden rule of abstraction: **Find the common pattern and parameterize the variation**. And let's do it.

The most conspicuous pattern is the overall structure of the three functions. What's the variation? The operations on items of the list. But how we gotta parameterize an operation? By creating an OO hierarchy? Yes, that'll do the trick, but we can also use functions, remember?

```python
def list_map(op, l):
    new_list = []
    for i in l:
        new_list.append(op(i))
    return new_list

def add_1(i):
    return i+1

def mul_2(i):
    return i*2

def m3_a4(i):
    return i*3+4

# provide interfaces to our client
def list_add_1(l):
    return list_map(add_1, l)

def list_mul_2(l):
    return list_map(mul_2, l)

def list_m3_a4(l):
    return list_map(m3_a4, l)

# client code
list_add_1([1, 2, 3]) # -> [2, 3, 4]
list_mul_2([1, 2, 3]) # -> [2, 4, 6]
list_m3_a4([1, 2, 3]) # -> [7, 10, 13]
```

Great, now we have an abstraction called `list_map`, which *takes an operation and a list and returns a news with every item from the old list processed by the operation*. Actually, this is so common a job that many language provides builtin support for it, such as python's [`map`][python-map] and PHP's [`array_map`][php-array-map]. **Notice that we passed functions as the `op` parameter**, which is a huge leap. **A function takes functions as argument or returns function as result is called a *higher order function***. To understand the concept, you just take functions as normal values, like integers or lists, which can be passed to and returned from a function. The function `list_map` is such a function.

[python-map]: http://docs.python.org/2/library/functions.html#map
[php-array-map]: http://php.net/manual/en/function.array-map.php

Let's go on, what's next? It *seems* there's still some duplication between the three functions, but it's a little hard to explicitly name the pattern. The common pattern here is that they both *embed* an operation and *delegates* the real work to `list_map` with respective operation. Can we parameterize it? It's a little subtle, but let's go.

```python
def list_map(op, l):
    new_list = []
    for i in l:
        new_list.append(op(i))
    return new_list

def add_1(i):
    return i+1

def mul_2(i):
    return i*2

def m3_a4(i):
    return i*3+4

def bind_parameter(func, first):
    def wrapper(second):
        return func(first, second)
    return wrapper

# provide interfaces to our client
list_add_1 = bind_parameter(list_map, add_1)
list_mul_2 = bind_parameter(list_map, mul_2)
list_m3_a4 = bind_parameter(list_map, m3_a4)

# client code
list_add_1([1, 2, 3]) # -> [2, 3, 4]
list_mul_2([1, 2, 3]) # -> [2, 4, 6]
list_m3_a4([1, 2, 3]) # -> [7, 10, 13]
```

The higher order function `bind_parameter` function has the following characteristics:

1. it takes a function and it's supposed first parameter as parameters
2. (no way in C) it **creates a function** right inside its body, which takes a parameter can pass it along to `func` to fulfill `func`'s signature
3. (no way in C) the `wrapper` function **accesses two variables outside its scope** (`func` is a variable, remember?)
4. it returns the newly created function to the caller

A little mind-blowing, maybe? This illustration will help you understand:

```python
list_add_1 = bind_parameter(list_map, add_1)
     ^                         |        |
     |                         V        V
     |   def bind_parameter(  func,   first):
     |                         |        |
     |                   +-----+        |
     |                   |(3)           |(4)
     |                   |              |
     |       def wrapper(second):<------+----+
     |                   |              |    |(2)
     |(1)                |     +--------+    |
     |                   |     |             |
     |                   |     |        +----+
     |                   |     |     (5)|    |
     |                   v     v        v    |
     |           return func(first, second)  |
     |           #  list_map(add_1, [1, 2])  |
     +-------return wrapper                  |
                                             |
              +------------------------------+
              |
list_add_1([1, 2])
```

The explanation: (1) the `wrapper` function is returned and assigned to `list_add_1`. (2) the argument (`[1, 2]`) to `list_add_1` is actually passed to `wrapper` as `second`. Then, as `wrapper` magically remembers (3) `func` (`list_map`, in this case) and (4) `first` (`add_1`, in this case), it calls `func`/`list_map` with `first`/`add_1` and `second`/`[1,2]` (5) and returns its result.

You may wonder why `list_add_1` can still access `list_map` and `add_1` even after it has "left" the scope it is defined. In FP, this is called **closure - a function together with the environment it is created**. `list_add_1` is a closure so it still gains access to the variables in the environment it is defined. This behavior is impossible to achieve in traditional programming languages like C as the variable scope is determined statically. Closure overcomes the problem of static variable scope and makes it possible to compose useful new functions with variables bound to its outer environment.

Take a little time to consume this. You'll get it.

A side but important benefit we can get from closure is to restrict data access. Data encapsulation is a big reason why OOP is so praised, but with closure, it's so easy to achieve, without a single class. See this example:

```python
# run with python3
def x_accessors():
    x = 1
    def getter():
        return x

    def setter(_x):
        nonlocal x
        x = _x

    return getter, setter

get_x, set_x = x_accessors()

print('x is', get_x())
set_x(2)
print('x is now', get_x())
```

Because of closure, x is only accessible to `getter` and `setter`. You can get it and set it through them, but never directly access it. A little inspiration, right?

Back to `bind_parameter`. With it we can now introduce the notion of **partially applied function**. When a function taking n parameters gets less than n ones (say k) bound, it is said to be partially applied. And we need to provide it with the rest n-k parameters to get it **fully applied**. And *the process to get a partially applied function is called **currying***. Our little `bind_parameter` serves as an example of such curry function. However, it is actually a rather limited one as it can only bind the first parameter to a function with two parameters. Take a little step further we can a more flexible version of curry:

```python
def curry(func, *args):
    """bind parameters from left to right"""
    def partial(*more_args):
        full_args = args+more_args
        return func(*full_args)
    return partial

def contrived(x, y, z, p):
    return (x+y)**z/p

contrived_partial = curry(contrived, 1, 5)
print contrived_partial(2, 4)# -> 9
```

Note1: `*args` is python's way of saying variable length parameter, like `…` in C

Note2: (to whom knows python) `**kwargs` is omitted for simplicity

We now have a curry function that can bind arbitrary number of parameters! Bonus: if you understand this without any difficulty, take a look at this [more powerful `curry`][unlimited-curry] which can bind parameters for many times until the desired number of parameters are satisfied.

[unlimited-curry]: https://gist.github.com/kavinyao/5603699

Back to our little demo code again and let's look for common pattern again. And this time it's . . . `add_1`, `mul_2` and `m3_a4` -- they are all special versions of the some generic operation. Moreover, if you are a quick learner, you'd notice that they are the both partially applied (e.g. `add_1` is `add` bound with 1). So the code is further refactored into this:

```python
from operator import add, mul

def list_map(op, l):
    new_list = []
    for i in l:
        new_list.append(op(i))
    return new_list

def mul_add(a, b, x):
    return a*x+b

def curry(func, *args):
    """bind parameters from left to right"""
    def partial(*more_args):
        full_args = args+more_args
        return func(*full_args)
    return partial

add_1 = curry(add, 1)
mul_2 = curry(mul, 2)
m3_a4 = curry(mul_add, 3, 4)

# provide interfaces to our client
list_add_1 = curry(list_map, add_1)
list_mul_2 = curry(list_map, mul_2)
list_m3_a4 = curry(list_map, m3_a4)

# client code
list_add_1([1, 2, 3]) # -> [2, 3, 4]
list_mul_2([1, 2, 3]) # -> [2, 4, 6]
list_m3_a4([1, 2, 3]) # -> [7, 10, 13]
```

We are near the final step and our remaining job is to get rid of abstract out `mul_add`. Why? Because there's a pattern in it, let's see:

```python
def mul_add(a, b, x):
    return a*x+b
m3_a4 = curry(mul_add, 3, 4)

# is equivlant to
def mul_add(a, b, x):
    # reuse is *awesome*
    return add(b, mul(a, x))
m3_a4 = curry(mul_add, 3, 4)

# is equivlant to
def mul_add(a, b, x):
    mul_a = curry(mul, a)
    add_b = curry(add, b)
    return add_b(mul_a(x))
m3_a4 = curry(mul_add, 3, 4)
```

Notice the pattern? In essence, we're applying two operations on x, first mul_a and then add_b, and the operations could be any as long as it takes a single parameter and returns another. Addition? Multiplication? It doesn't matter. By a or b? That's even more extraneous. So the common pattern here is the operations, and let's parameterize 'em. Like this:

```python
def chain(op1, op2):
    def combined_ops(x):
        return op1(op2(x))
    return combined_ops

add_4 = curry(add, 4)
mul_3 = curry(mul, 3)
m3_a4 = chain(add_4, mul_3)
```

That's exactly the idea of **function composition** mentioned in the Loup's article. And since we can compose two, why not parameterize more?

```python
def compose(*ops):
    def composite(x):
        res = x
        # the ops are called in reverse order
        for op in reversed(ops):
            res = op(res)
        return res
    return composite

# bonus: if we remove the assumption that the last op takes a single value
def compose(*ops):
    def composite(*args):
        r_ops = reversed(ops)
        # only the last op takes in multiple arguments
        res = r_ops.next()(*args)
        for op in r_ops:
            res = op(res)
        return res
    return composite
```

With `compose`, we now have our final version of the demo:

```python
from operator import add, mul

def list_map(op, l):
    new_list = []
    for i in l:
        new_list.append(op(i))
    return new_list

def curry(func, *args):
    def partial(*more_args):
        full_args = args+more_args
        return func(*full_args)
    return partial

def compose(*ops):
    def composite(*args):
        r_ops = reversed(ops)
        res = r_ops.next()(*args)
        for op in r_ops:
            res = op(res)
        return res
    return composite

add_1 = curry(add, 1)
mul_2 = curry(mul, 2)
m3_a4 = compose(curry(add, 4), curry(mul, 3))

# provide interfaces to our client
list_add_1 = curry(list_map, add_1)
list_mul_2 = curry(list_map, mul_2)
list_m3_a4 = curry(list_map, m3_a4)

# client code
list_add_1([1, 2, 3]) # -> [2, 3, 4]
list_mul_2([1, 2, 3]) # -> [2, 4, 6]
list_m3_a4([1, 2, 3]) # -> [7, 10, 13]
```

Till this point, our code has no more abstraction to extract. So what's the benefit? Think for a while before you read on.

To summarize:

### separation of concerns

Remember the initial `list_add_1`? It does basically two things: 1) increment by 1 on each item of the input list, and 2) store incremented value in a new list. Ever learned about [cohesion][wiki-cohesion]? This is a typical example of coincidental cohesion, *the **worst** one*. So we split them into two separate logic: 1) apply an operation on each item of the input list, store new value in another list, and 2) input increment by 1 as the operation.

[wiki-cohesion]: http://en.wikipedia.org/wiki/Cohesion_(computer_science)

### improved reusability

Since we tried our best to extract abstractions from variations, we now have several building blocks to reuse. For example, if the customer issues a new requirement that create a new list by multiply by 3, we can write:

```python
mul_3 = curry(mul, 3)
list_mul_3 = curry(list_map, mul_3)
```

and the job is done, without any need to repeat code. This is also in accordance with the [OCP Principle][wiki-ocp].

[wiki-ocp]: http://en.wikipedia.org/wiki/Open/closed_principle

### better flexibility

With high-order functions, we can easily compose new variation upon common abstractions, hence better flexibility. Wanna multiply by 3 then add 4 then divide by 2? No problem:

```python
m3a4d2 = compose(
    curry(mul, 3),
    curry(add, 4),
    curry(div, 2)
)
list_m3a4d2 = bind_parameter(list_map, m3a4d2)
```

Finally, let's take a little recap of the essential concepts of FP:

* higher-order functions
* closure
* partially applied functions with curry
* function composition

do you remember each?

Oh! Let's not forget the last bonus: **lambda functions**, also called anonymous functions. They exist in many languages (limited in python, prevalent in JavaScript, introduced to PHP in 5.3, available in C# since 3.0 and forever delayed in Java (chuckling)). They are convenient but are essentially *syntactic sugar*, but considering programmer happiness, they are great.
