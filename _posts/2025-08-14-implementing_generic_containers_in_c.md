---
title: "Implementing Generic Containers in C"
excerpt: "C is a simple, low-level language, and keeping code generic requires special techniques. This post explores two approaches to building a generic stack in C."
categories:
  - Programming
tags:
  - C
---
## Introduction

In this post, I'll present some tricks that make it possible to implement a container in C that can store any
data type. No matter if it's an integer, floating point, or even a custom struct.

In C, there is no standard way to write code that works with different data types.
To illustrate the problem, let's recall how to get the absolute value
of an integer using the standard library:

```c
// Include the header where abs functions are declared:
#include <stdlib.h>

int a = -5;
int abs_a = abs(a);

long b = 14;
long abs_b = labs(b);

long long c = -42;
long long abs_c = llabs(c);
```

When we look at the implementation of each of the three functions, we
can see that their bodies are identical. The only difference is in the function
signature:

[[glibc.git]/stdlib/abs.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdlib/abs.c;h=5b75a512f9c7b6568faac4685177691bc2d39d3a;hb=HEAD):

```c
int
abs (int i)
{
  return i < 0 ? -i : i;
}
```

[[glibc.git]/stdlib/labs.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdlib/labs.c;h=dff3b5a97b3bd6c010dec56e9be24916831d1f75;hb=HEAD):

```c
long int
labs (long int i)
{
  return i < 0 ? -i : i;
}
```

[[glibc.git]/stdlib/llabs.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdlib/llabs.c;h=95279f7ebe3434b8ca5b19ad41382f09e75753dc;hb=HEAD):

```c
long long int
llabs (long long int i)
{
  return i < 0 ? -i : i;
}
```

The `abs` function is super simple, so it’s not difficult to reimplement for every integer type.
It becomes tedious, however, when we implement larger pieces of logic and need to support many types.
We could simply copy-paste the implementation for every type and call it a day.
This is the simplest solution and doesn't require doing anything sophisticated.
However, when we want to add a new feature or fix a bug, we have to apply
the same change in every version of the code - and that is definitely not fun
and can be error-prone.

In the next part of this post, I'll present a solution demonstrated on
a simple data collection (LIFO stack), and show ways we can reuse the core
data structure logic for many types.

## Example Container: Stack
### Stack Interface

Our goal is to implement a stack data structure. The features we expect:
- create/destroy the stack: functions for initializing, allocating, and
deallocating memory
- push to the stack and pop from the stack
- peek - check the value on the top of the stack without popping it
- check if the stack is empty

For simplicity, we will define the stack capacity at creation time
to avoid reallocating memory, since memory management is not our focus here.

Let's start with the interface of a stack that stores integers:

```c
#pragma once

#include <stddef.h>
#include <stdbool.h>

typedef struct Stack* Stack;

Stack Stack_create(size_t capacity);
void Stack_destroy(Stack s);

void Stack_push(Stack s, int value);
void Stack_pop(Stack s);
int Stack_peek(Stack s);
bool Stack_empty(Stack s);
```

There’s not much to explain here - the function prototypes speak for themselves.
The complete code can be found [on my GitHub](https://github.com/pawel-kusinski/blog-code-snippets/blob/main/generic_stack/int_type/stack.h).

### Stack Internals

Before diving deep into making the stack generic, let's quickly take a look at
how it's implemented.  In the snipped below we can see the function that
creates the stack and the internal data structure to keep track on the stack
state.

```c
struct Stack {
    int* data;
    size_t capacity;
    size_t top;
};

Stack Stack_create(size_t capacity) {
    Stack s = malloc(sizeof(struct Stack));
    s->data = malloc(capacity * sizeof(int));
    s->capacity = capacity;
    s->top = 0;
    return s;
}
```

`Stack.data` points to the memory where we keep all stored elements.
`Stack.capacity` tells how many elements we can store in the buffer, and `Stack.top` keeps track of the current stack size, so we know the offset
in `Stack.data` where we can push and pop elements. It also tells us whether
the stack is empty or full.

To show how `Stack.capacity` and `Stack.top` are used, here are `Stack_push()` and `Stack_pop()`:

```c
void Stack_push(Stack s, int value) {
    if (s->top >= s->capacity) {
        return;
    }

    s->data[s->top++] = value;
}

void Stack_pop(Stack s) {
    assert(s->top);
    s->top--;
}
```

When pushing, we compare `top` with `capacity` to ensure we don’t
write outside the allocated buffer. Before popping, we check `top` to
ensure there’s something to pop.

If you want to see the other functions, they’re in [this
repo](https://github.com/pawel-kusinski/blog-code-snippets/blob/main/generic_stack/int_type/stack.c).

Now, let’s look at the first method to make our integer stack generic so it can
store any data type.

## Method 1: void* Pointer

Take a look at the `Stack_push()` function signature.
It clearly says: "I expect you to pass a value, and its type should be `int`".

```c
void Stack_push(Stack s, int value);
```

Now, how can we modify this so the function accepts any type?
There is a way - and it's called a void pointer:

```c
void Stack_push(Stack s, void* value);
```

When we pass a void pointer, `Stack_push()` only knows where the element
is stored — it doesn’t know how many bytes it occupies.
`Stack_push()` needs to copy the data we pass to it.
The missing information is the size in bytes of that element.
One option would be:

```c
void Stack_push(Stack s, void* value, size_t elem_size);
```

Here, elem_size tells how many bytes to copy from the value pointer.
However, all elements in a stack are the same type, meaning they’re all
the same size.
So a better idea is to pass `elem_size` only once, when creating the stack,
and store it in the internal data structure.

Revised stack structure and `Stack_create()`:

```c
struct Stack {
    char* data;
    size_t capacity;
    size_t top;
    size_t elem_size;
};

Stack Stack_create(size_t capacity, size_t elem_size) {
    Stack s = malloc(sizeof(struct Stack));
    s->data = malloc(capacity * elem_size);
    s->capacity = capacity;
    s->top = 0;
    s->elem_size = elem_size;
    return s;
}
```

The `data` array is now just a byte array.
A size of one byte works for any data type - from 8-bit characters to custom
structures whose size is not divisible by 2 or 4.

When creating a stack that stores up to 3 long values:

```c
size_t capacity = 3;
size_t elem_size = sizeof(long);
Stack s = Stack_create(capacity, elem_size);
```

If `sizeof(long)` is 8, the internal array will be 24 bytes long.
Note that the actual size of a type may depend on the platform.

The `Stack_push()` function for this generic stack:

```c
void Stack_push(Stack s, void* value) {
    if (s->top >= s->capacity * s->elem_size) {
        return;
    }

    memcpy(&s->data[s->top], value, s->elem_size);
    s->top += s->elem_size;
}
```

Here, `memcpy()` copies `elem_size` bytes from the `value` pointer into the
buffer starting at index `top`.
In this generic version, `top` is incremented or decremented by `elem_size`
each time we push or pop.

This method lets us implement a data structure once for all types - no extra
headers needed.
The downside is syntactic overhead. For example, `Stack_peek()` returns
a void pointer:

```c
void* Stack_peek(Stack s)
    assert(s->top)
    return (void*)&s->data[s->top - s->elem_size]
}
```

To use the value, we must cast and dereference:

```c
long val = *((long*)Stack_peek(s));
printf("val: %ld\n", val);
```

This adds extra syntax and reduces readability.

## Method 2: Macros

Another method uses the preprocessor to generate function declarations
and definitions for each type.

We want to "copy and paste" the integer stack header for different types:

```c
typedef struct Stack* Stack;
Stack Stack_create(size_t capacity);
void Stack_destroy(Stack s);
void Stack_push(Stack s, int value);
void Stack_pop(Stack s);
int Stack_peek(Stack s);
bool Stack_empty(Stack s);
```

The X-Macros technique works well here - we define the list of desired
data types in one place, and macros generate all declarations and definitions.
You can read more on X-Macros in [this post](https://www.kusinski.net/programming/practical_use_cases_of_x_macros_in_c).

### Custom Structure for Demonstration

To show that this method works with custom types, let’s define a `Point` struct:

```c
typedef struct {
    double x;
    double y;
} Point
```

### List of Stack Data Types

Here’s the list of types for which stack functions are generated:

```c
#define STACK_TYPES \
    X(double) \
    X(Point)
```
This is the only place where we add data types to support,
and all macros will generate code for it.

### X Macro in the Header File

In the header, we define the X macro. `T` is the type name.
We concatenate `Stack_` with the type name to create handle and function names.

```c
#define X(T) \
typedef struct Stack_ ## T * Stack_ ## T; \
Stack_ ## T Stack_ ## T ## _create(size_t capacity); \
void Stack_ ## T ## _destroy(Stack_ ## T s); \
void Stack_ ## T ## _push(Stack_ ## T s, T value); \
void Stack_ ## T ## _pop(Stack_ ## T s); \
T Stack_ ## T ## _peek(Stack_ ## T s); \
bool Stack_ ## T ## _empty(Stack_ ## T s);

STACK_TYPES

#undef X
```

### Header File Macro Expansion

For clarity, here’s what it expands to:

```c
typedef struct Stack_double * Stack_double;
Stack_double Stack_double_create(size_t capacity);
void Stack_double_destroy(Stack_double s);
void Stack_double_push(Stack_double s, double value);
void Stack_double_pop(Stack_double s);
double Stack_double_peek(Stack_double s);
Stack_double_empty(Stack_double s);

typedef struct Stack_Point * Stack_Point;
Stack_Point Stack_Point_create(size_t capacity);
void Stack_Point_destroy(Stack_Point s);
void Stack_Point_push(Stack_Point s, Point value);
void Stack_Point_pop(Stack_Point s);
Point Stack_Point_peek(Stack_Point s);
Stack_Point_empty(Stack_Point s);
```

### X Macros in the C file

In the .c file, we use the same technique:

```c
#define X(T) \
\
struct Stack_ ## T { \
    T* data; \
    size_t capacity; \
    size_t top; \
}; \
\
Stack_ ## T Stack_ ## T ## _create(size_t capacity) { \
    Stack_ ## T s = malloc(sizeof(struct Stack_ ## T)); \
    s->data = malloc(capacity * sizeof(T)); \
    s->capacity = capacity; \
    s->top = 0; \
    return s; \
} \
\
void Stack_ ## T ## _push(Stack_ ## T s, T value) { \
    if (s->top >= s->capacity) { \
        return; \
    } \
 \
    s->data[s->top++] = value; \
}

STACK_TYPES

#undef X
```

It expands to:

```c
struct Stack_double {
    double* data;
    size_t capacity;
    size_t top;
};

Stack_double Stack_double_create(size_t capacity) {
    Stack_double s = malloc(sizeof(struct Stack_double));
    s->data = malloc(capacity * sizeof(double));
    s->capacity = capacity;
    s->top = 0;
    return s;
}

void Stack_double_push(Stack_double s, double value) {
    if (s->top >= s->capacity) {
        return;
    }

    s->data[s->top++] = value;
}

struct Stack_Point {
    Point* data;
    size_t capacity;
    size_t top;
};

Stack_Point Stack_Point_create(size_t capacity) {
    Stack_Point s = malloc(sizeof(struct Stack_Point));
    s->data = malloc(capacity * sizeof(Point));
    s->capacity = capacity;
    s->top = 0;
    return s;
}

void Stack_Point_push(Stack_Point s, Point value) {
    if (s->top >= s->capacity) {
        return;
    }

    s->data[s->top++] = value;
}
```

### Bonus: Function Overloading

Thanks to the macros, the Stack declarations and logic are generated for any data type.
However, unlike the void pointer method, we still end up with different functions for each type.
In the example below, we push two different elements to two stacks of different types.
Because the functions are type-specific, we must ensure we call the correct variant.

```c
double val_double = 1;
Stack_double_push(s_double, val_double);

Point val_point = {.x = 1, .y = 2};
Stack_Point_push(s_point, val_point);
```

There is a way to write just `Stack_push(s, v)` regardless of the type of `s` and `v`.
To achieve this, we can use the `_Generic` keyword, which allows us to simulate
function overloading similar to what exists in other programming languages.

```c
#define Stack_push(s, v) _Generic((s), \
    Stack_double: Stack_double_push, \
    Stack_Point: Stack_Point_push \
)(s, v)
```

The `_Generic` statement checks the type of `s` (the stack handle) and, based on its type,
selects the matching function. With this definition in place, the previous example can be rewritten as:

```c
double val_double = 1;
Stack_push(s_double, val_double);

Point val_point = {.x = 1, .y = 2};
Stack_push(s_point, val_point);
```

This becomes especially useful when we decide to change the type stored in the stack.
For example, if we switch from `int` to `long`, we no longer need to replace
`Stack_int_push` with `Stack_long_push` throughout the codebase - the `_Generic` macro handles it.

Earlier I mentioned that we only need to define the `STACK_TYPES` list and all macros
will expand for all specified types. Unfortunately, the `_Generic` selection list must still
be maintained separately alongside `STACK_TYPES`. This extra maintenance is why I mark
this technique as a "bonus" solution.

More details on the `_Generic` keyword can be found in [this
post](https://www.kusinski.net/programming/c_generic_keyword)

## Comparison of Presented Methods

|                  | Void Pointer | Macros |
|------------------|:------------:|:------:|
| Code Size        |     :+1:     | :-1:   |
| Easy to use API  |     :-1:     | :+1:   |
| Code readability |     :+1:     | :-1:   |
| Modularity       |     :+1:     | :-1:   |

### Code Size

The void pointer technique uses the same implementation for all types,
so the code size does not grow as the number of supported stack types increases.

When using the macro technique, similar code is generated for every data type,
so the total code size grows as we add more types.

### Easy to Use API

Because void pointers require casting and dereferencing,
we end up with more complicated syntax even for simple functions such as getters.

Code generated by macros is easier to use,
because it is clearly defined for each data type and does not require
pointer-related operators such as `&` and `*`.

### Code Readability

Even though dealing with pointers usually involves more effort and requires extra care,
code defined using macros can be harder to read and follow - especially when
your text editor does not support automatic macro expansion views.

### Modularity

Here’s what I mean by "modularity":
Code that is modular can be compiled as a library, kept in a separate repository,
and easily reused in different projects without modification.

The void pointer implementation is 100% modular,
and can be reused for any data type without customization.

This is not the case with the macro-based implementation.
The `STACK_TYPES` list defines all supported data types.
We could add all built-in C types to that list,
but we cannot automatically include custom types defined by the user.
Therefore, the user either has to modify the original header file
or provide their own version and include it in the main header file.

## Conclusion

C is a simple, low-level language, and therefore, to keep the code generic,
we have to rely on techniques that each have their own advantages and disadvantages.
If you use a data structure only in a few places or projects,
it may not be worth applying the techniques presented here.
However, when you find yourself rewriting the same code for many different data types,
you might want to give them a try.

## Links
Complete source code for this blog post:
[github.com/pawel-kusinski](https://github.com/pawel-kusinski/blog-code-snippets/tree/main/generic_stack)

glibc stdlib source tree:
[sourceware.org](https://sourceware.org/git/?p=glibc.git;a=tree;f=stdlib;hb=HEAD)

Practical Use Cases of X Macros in C:
[kusinski.net](https://www.kusinski.net/programming/practical_use_cases_of_x_macros_in_c/)

Generic Keyword in C:
[kusinski.net](https://www.kusinski.net/programming/c_generic_keyword/)
