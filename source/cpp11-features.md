# C++11 Features Every C++ Programmer Should Know

- pubdate: 2014-02-11
- tags: C++, language
- description: This article introduces several C++11 features that can simplify your C++ program and make you more productive.

---

Recently I've been practicing [LeetCode](http://oj.leetcode.com/) problems for technical interview. LeetCode OJ supports C++11 and I feel great writing code with it. While there are articles introducing C++11 features (for example [this one](http://www.codeproject.com/Articles/570638/Ten-Cplusplus11-Features-Every-Cplusplus-Developer)), most of them are too deep. I don't believe you need to understand move semantics before you write a well-formed C++ program.

To clarify, I'm not saying that advanced features like move semantics are not important. Rather, I find some C++11 features which not only are easy to learn but can greatly improve programming experience using C++. So, this article is about those features that every C++ programmer should absolutely check out.

Before we start: if you want to compile the examples, please make sure your favorite compiler supports C++11 and set the option `-std=c++11`. (The latest MSVC, g++ and clang has support for most of the C++11 features. Check [here](http://cpprocks.com/c11-compiler-support-shootout-visual-studio-gcc-clang-intel/) for a compatibility table)

## initializer list

C++, and also many old languages, lack support for literals for non-native types like `vector` and `map`. With C++11 initializer list, you can initialize them like this:

```cpp
vector<int> iv = {1, 2, 3, 4};
map<int, string> employees = {{1, "John"}, {2, "Mary"}};
```

Note as `map` (and also `unordered_map`, see below) stores key-value pairs internally, the syntax is like initializing a set of pairs.

What's more, the initializer list can be nested:

```cpp
map<int, vector<char>> number_pad = {
    {1, {'a', 'b', 'c'}},
    {2, {'d', 'e', 'f'}},
    ...
};
```

This not only gives you more concise code, but more importantly shows your intention better.

*Advanced Tip: you can actually use `std::initializer_list` to define literals for your own class.*

## right angle bracket

Did you notice in the previous example we omitted the space between the two `>` characters? Compilers with C++11 support will not treat that as shift operator any more. One more pitfall fixed.

## type inference (the `auto` keyword)

Remember the last time you wrote this:

```cpp
for(vector<int>::iterator it = iv.begin();it != iv.end();++it) {}
```

Have you ever wondered why the compiler forces you to specify type of `it` even though it can infer the type? (it is unambiguous that `vector<int>::begin` returns `vector<int>::iterator`)

Not any more! With C++11, you can let the compiler infer the type itself with the `auto` keyword:

```cpp
vector<int> iv = {1, 2, 3};
for(auto it = iv.begin();it != iv.end();++it) {
    cout << (*it) << endl;
}
```

This handy feature can reduce verbosity greatly. For example

```cpp
WindowController *controller = new WindowController;
auto controller = new WindowController;
```

I believe Java guys will envy this feature. ;)

*Advanced Tip: you will be able to use `auto` keyword to do function return type deduction in C++14.*

## range-based for loop

Yes, C++ finally got language level support for range-based for loop! With this, you can write the previous example as

```cpp
vector<int> iv = {1, 2, 3};
for(auto i : iv) { // type of i is int
    cout << i << endl;
}
```

Same behavior, much less code!

When copying is not desired, you can also use reference:

```cpp
list<HeavyClass> queue = {...};
for(auto &o : queue) { // type of o is HeavyClass&
    // manipulate o
}
```

## hashtable containers

C++11 introduces four new containers: `unordered_set`, `unordered_multiset`, `unordered_map` and `unordered_multimap`. Each has the exactly same interface as their counterparts without the `unordered_` prefix. The difference is that these new containers are based on hash table instead of red-black trees.

See this [discussion](http://stackoverflow.com/q/2196995/1240620) for choosing `map` or `unordered_map`.

Note: the standard committee chose the `unordered_` prefix to ensure backward compatibility with programs using non-standard `hashtable` implementation (which is available in many C++ libraries as extension).

## object construction improvement

Object construction in C++03 used to have several
restrictions. The most unintuitive one may be that you cannot delegate construction to constructors of the same class. Although you can often solve such problem with default parameter(s) or a common initialize function, sometimes you just can't. C++11 removes such restriction:

```cpp
class SomeType  {
    int number;
 
public:
    SomeType(int new_number) : number(new_number) {}
    SomeType() : SomeType(42) {}
};
```

The second improvement is that you can use base class's constructor directly, instead of writing a wrapper constructor.

```cpp
class BaseClass {
public:
    BaseClass(int value);
};
 
class DerivedClass : public BaseClass {
public:
    // C++03
    DerivedClass(int value): BaseClass(value) {}
    // C++11
    using BaseClass::BaseClass;
};
```

The third improvement is more handy: you can initialize non-static member at declaration.

```cpp
class Coordinate {
    int x = 0;
    int y = 0;
public:
    Coordinate() {}
    Coordinate(int _x, int _y): x(_x), y(_y) {}
};
```

In the above example, if the constructor with no parameter is called, the two members `x` and `y` will have the default value of 0.

Note that initialization at declaration is only possible for *non-static* members. For static members, you still have to initialize them outside the class.

## lambda functions

Functional programming (FP) has become more and more popular in recent years. Anonymous functions, which may not be an essential feature of FP, is so handy that many programmers love it. Before C++11, STL uses [functors](http://en.wikipedia.org/wiki/Functor_%28C%2B%2B%29) to emulate anonymous functions.

Consider the following program:

```cpp
struct AbsCompare {
    bool operator() (int i, int j) {
        return abs(i) < abs(j);
    }
};

int main() {
    vector<int> iv = {8, -4, 3, -2, 5, 1, -7, 9, -6};
    
    sort(iv.begin(), iv.end(), AbsCompare());
    
    for(int i : iv)
        cout << i << endl;
}
```

With C++11, you can write as this:

```cpp
int main() {
    vector<int> iv = {8, -4, 3, -2, 5, 1, -7, 9, -6};
    
    sort(iv.begin(), iv.end(), [](int i, int j) {
        return abs(i) < abs(j);
    });
    
    for(int i : iv)
        cout << i << endl;
}
```

Not surprisingly, C++11 anonymous functions support closures. You can capture outer variables in `[]`. A simple example is to find number of values less than `threshold`:

```cpp
vector<int> numbers = {1, 2, 3, 4, 5, 6};
int threshold = 4, count = 0;
for_each(numbers.begin(), numbers.end(), [threshold, &count](int i) {
    if (i < threshold) ++count;
});
cout << count << endl;
```

This is a pedagogical example, you can actually use [`count_if`](http://www.cplusplus.com/reference/algorithm/count_if/), but it demonstrates how to capture variables anyway. `threshold` is captured by value, while `count` is captured by reference - that's why we can change value of `count` inside the anonymous function. Actually, if you are lazy enough, you can use `[&]` or `[=]` to implicitly capture any external variable by reference or not. But such practice is not recommended.

What may surprise you is that anonymous functions have a different type from plain functions. They are actually specialization of type `std::function`. For example, the anonymous function used in the previous example is a `std::function<void(int)>` and, as a result, it cannot be assignment to variable of `void (*)(int)` type. If you want your function to accept both traditional and anonymous functions, remember to use templates (like what `std::sort` does).

## regular expressions

The last feature I want to introduce is regular expression facility.  A simple example would serve the purpose:

```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string haystack = "The United States is a developed country and has the world's largest national economy, with an estimated GDP in 2013 of $16.7 trillion – 23% of global nominal GDP and 19% at purchasing-power parity.";
    std::regex num_rgx("(^|\\b)\\d+($|\\b)");
    std::smatch match;

    while (std::regex_search(haystack, match, num_rgx)) {
        std::cout << "find number: " << match[0] << std::endl;
        haystack = match.suffix().str();
    }

    return 0;
}
```

Here we first constructed regular expression of numbers and a match object (it's prefixed `s` because it's for `string`, there's also a `cmatch` class for `const char*`). Then we use `regex_search` for searching in haystack for numbers. If there's a match, as you would expect, group 0 is the whole matched string, group 1 is the preceding boundary (`(^|\\b)`) and group 1 is the succeeding boundary (`($|\\b)`). You can access each group using `[]` operator.

Note in this simple example, in order to advance the matching, we use `match.suffix().str()` to get the unmatched part of `haystack`, which is rather inefficient. For performance, use [`std::regex_iterator`](http://en.cppreference.com/w/cpp/regex/regex_iterator#Example) instead.

I also found a [cheat sheet](http://cpprocks.com/regex-cheatsheet/) for C++11 regex.

---

There are actually tons of nice new features like threading, uniform initialization, strongly typed enums, tuples, raw string literals and on and on and on. C++11 is really a whole new language. For reference and a full list of C++11 new features, see [this Wikipedia article](http://en.wikipedia.org/wiki/C++11#Changes_from_the_previous_version_of_the_standard) (some examples in this article are from the article).

With coming C++14, the trend is that, the language is becoming more dynamic (at least in appearance) and programmer-friendly.