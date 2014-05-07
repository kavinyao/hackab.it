---
layout: post
title: "How Linux Implements Generic Linked List"
date: 2014-01-13
tags: C, kernel, data structure
description: "This article introduces a relatively rarely seen C-implementation of generic linked list, which is actually implemented in the Linux kernel."
---

Since my junior year, I've been primarily programming in pretty high-level languages such as Python and JavaScript. While there's always some hidden cost, high-level languages really help you focus on the problem itself instead of the nuances of implementation, which usually can increase your productivity and avoid frustration.

Recently, however, I wrote plenty of C/C++. It's like back to the old days. Though it's pretty frustrating to find out how cumbersome it is to [use user-defined type as `map` keys](http://stackoverflow.com/q/17016175/1240620), I still find there's definitely enough fun in not-so-high-level programming. I can still remember that kind of thrill I had when I learned that you can use [`auto_ptr`](http://www.cplusplus.com/reference/memory/auto_ptr/) to automatically release memory. The point is, languages like C/C++ have mechanisms to tackle problems which seems difficult (or impossible) to solve at first glance. You can even fix some language design mistakes with them. *Not every programming language has such ability.*

A gem I discovered today is another way of implementing generic linked list in C. I encountered it when reading the [Pintos](http://en.wikipedia.org/wiki/Pintos) source code.

The most popular way of implementing generic linked list in C is to use something like ([via](http://pseudomuto.com/development/2013/05/02/implementing-a-generic-linked-list-in-c.html)):

```cpp
typedef struct _listNode {
    void *data;
    struct _listNode *next;
} listNode;
```

where data points to the actual element and since it's of void pointer type, it can hold any pointer. While such implementation is indeed feasible and obvious, they typically involve heavy dynamic memory allocation (DMA) of list nodes (1 for the data and 1 for the node itself). Everybody knows **DMA is costly**.

But to construct such dynamic data structure in C, you have to resort to dynamic memory in C! You might say.

Yes, and No.

Let's see the actual implementation before answering why.

The implementation begins with the definition of list node (it's a doubly-linked list):

```cpp
struct list_elem
{
    struct list_elem *prev;     /* Previous list element. */
    struct list_elem *next;     /* Next list element. */
};
```

And we have a `struct list` type to keep the head and tail of the linked list:

```cpp
struct list
{
    struct list_elem head;      /* List head, dummy node. */
    struct list_elem tail;      /* List tail, dummy node. */
};
```

With the `head` and `tail`, `prev` and `next`, we can define all possible linked list operations on these structures. For example:

```cpp
void list_insert (struct list_elem *before, struct list_elem *elem)
{
    ASSERT (is_interior (before) || is_tail (before));
    ASSERT (elem != NULL);

    elem->prev = before->prev;
    elem->next = before;
    before->prev->next = elem;
    before->prev = elem;
}
```

So where the heck is the data? Let's see.

To construct a linked list of type `struct foo`, first you put `struct elem` into it:

```cpp
struct foo
{
    struct list_elem elem;
    int bar;
    ...other members...
};
```

That seems clever as you just cast `struct list_elem *` to `struct foo *` (since `elem` is the first member of `struct foo`).

But the surprising part is that we can actually remove such restriction and put `struct list_elem elem` anywhere you like in `struct foo`. To achieve this,  you do something like this (`e` is a `struct list_elem *`):

```cpp
struct foo *f = list_entry(e, struct foo, elem);
```

And the `list_entry` is a macro defined like this:

```cpp
#define list_entry(LIST_ELEM, STRUCT, MEMBER)    \
           ((STRUCT *) ((uint8_t *) LIST_ELEM    \
                - offsetof (STRUCT, MEMBER)))
```

that is, you get the address of `e`, subtract the offset of `elem` in `struct foo` and then you get the address of the structure `e` is embedded in.

So you see the key part is how to implement `offsetof`. It is implemented as, again, a macro:

```cpp
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *) 0)->MEMBER)
```

Did you get the trick? First we take address 0 as `TYPE *`, then get the address of its member `MEMBER`. Since the base address is 0, the address of `MEMBER` equals the offset. (Note we are taking the address of `MEMBER`, not accessing it, so there's no problem using address 0.)

That's it. And with clearly defined linked list API, we can do things like:

```cpp
struct list_elem *e;

for (e = list_begin (&foo_list); e != list_end (&foo_list); e = list_next (e))
{
    struct foo *f = list_entry (e, struct foo, elem);
    // do something with f
}
```

So you see, using the above implementation, DMA could be eliminated totally in contexts where full access to all memory is granted. And an operating system kernel like Pintos is clearly among them. Even if you don't program OS kernels, you can still get a performance gain because the number of DMA per list node is reduced to 1.

The downside is obvious as well: you *have to* fuse `struct list_elem` into every user defined type that you might put into a linked list, even if in most programs the objects won't be in any linked list. But at least in the context of OS kernel, where processes and threads are organized in different linked lists, using this solution is not only fine but also natural.

Another consequence is much more undesirable, however. As every `list_elem` can only exist in *one* linked list, if you want to put the same object into several linked lists, you have to put more than one `list_elem` into its fields. But how could you estimate how many linked list you are going to put a particular object in? As a result, you end up creating wrapper `struct`s for the object.

Finally, this is actually the way how Linux kernel implements [doubly linked list](https://github.com/torvalds/linux/blob/master/include/linux/list.h) (and I think maybe Pintos just copied the design).

Hope this relatively rarely seen C-implementation of generic linked list sheds some light on your understanding of C.
