---
title: "_Generic Keyword in C"
categories:
  - Programming
tags:
  - C
---

## Introduction
In this post, I will present an interesting feature introduced in C11: the `_Generic` keyword. At times, it may seem like the C language is not receiving new features in new standard revisions, and changes are limited to small improvements. However, the introduction of the `_Generic` keyword is a significant addition that can enhance the way we program in C, allowing us to write less repetitive and more generic code. In this post, I will present two example usages of `_Generic`.


## Use case 1: Function Overloading
Unlike C++, C does not support function overloading, which means we have to write separate functions that perform similar operations on different data types.

To illustrate a simple example of overloading, consider the following C++ code that defines two versions of the `calculateArea` function — one for squares and the other for circles. The compiler selects the appropriate implementation based on the object type passed to the function.

```cpp
#include <iostream>

using namespace std;

struct Square_t {
    double sideLength;
};

struct Circle_t {
    double radius;
};

double calculateArea(Square_t square) {
    return square.sideLength * square.sideLength;
}

double calculateArea(Circle_t circle) {
    return 3.14159 * circle.radius * circle.radius;
}

int main() {
    Square_t square;
    square.sideLength = 5.0;
    Circle_t circle;
    circle.radius = 3.0;
    double squareArea = calculateArea(square);
    cout << "Area of the square: " << squareArea << endl;
    double circleArea = calculateArea(circle);
    cout << "Area of the circle: " << circleArea << endl;
    return 0;
}
```
Running this program will produce the following output:

```console
Area of the square: 25
Area of the circle: 28.2743
```

Now, let's focus on C. How can we achieve function overloading in C? Thanks to the `_Generic` keyword, we can somewhat "simulate" function overloading. Let's reimplement the code presented earlier in C.

First, we need to rename the functions that calculate the area since we cannot use the same names. We'll add suffixes corresponding to the type each function is designated for:

```c
double calculateAreaSquare(Square_t square) {
    return square.sideLength * square.sideLength;
}

double calculateAreaCircle(Circle_t circle) {
    return 3.14159 * circle.radius * circle.radius;
}
```

Now, here's the trick: we'll use the `_Generic` selection, which chooses the appropriate definition based on the object type passed to the macro:

```c
#define calculateArea(x) _Generic((x), \
    Square_t: calculateAreaSquare, \
    Circle_t: calculateAreaCircle \
)(x)
```

Let's break it down and analyze the syntax explanation from [cppreference.com](https://en.cppreference.com/w/c/language/generic):

> `_Generic ( controlling-expression , association-list )`
> where `association-list` is a comma-separated list of associations, each of which has the syntax `type-name : expression`.


Whenever we call `calculateArea(x)`, the compiler checks the type of `x`. When `x` is of type `Square_t`, then `calculateAreaSquare` is selected. When `x` is of type `Circle_t`, then `calculateAreaCircle` is selected. When the type of `x` is neither `Square_t` nor `Circle_t`, the compilation will fail unless a default label is defined, which must have an expression that works properly with the object of the given type. The `(x)` at the end of the definition is necessary for the correct function call syntax.

The complete code is as follows. Note that the code in the `main` function is the same as its C++ counterpart, except for the functions used to print to standard output:

```c
#include <stdio.h>

typedef struct  {
    double sideLength;
} Square_t;

typedef struct {
    double radius;
} Circle_t;

double calculateAreaSquare(Square_t square) {
    return square.sideLength * square.sideLength;
}

double calculateAreaCircle(Circle_t circle) {
    return 3.14159 * circle.radius * circle.radius;
}

#define calculateArea(x) _Generic((x), \
    Square_t: calculateAreaSquare, \
    Circle_t: calculateAreaCircle \
)(x)

int main() {
    Square_t square;
    square.sideLength = 5.0;
    Circle_t circle;
    circle.radius = 3.0;
    double squareArea = calculateArea(square);
    printf("Area of the square: %f\n", squareArea);
    double circleArea = calculateArea(circle);
    printf("Area of the circle: %f\n", circleArea);
    return 0;
}
```

Running this program produces the following output:

```console
Area of the square: 25.000000
Area of the circle: 28.274310
```

## Use Case 2: Portable Drivers and Libraries Interfaces

### Usual Solution: Function Pointer

Suppose we are implementing a library that needs to be platform-independent, such as a command-line interface library for an embedded system. Let's assume that UART is our standard input/output. To ensure portability, the library code cannot use platform-specific functions that directly read from or write to UART.

One of the techniques used to make the code portable is passing function pointers to the library. These function pointers point to functions that implement generic operations, such as writing a character to standard output in a platform-specific way. Here's an example of how it's done:
```c
/* sys_shell.c library source file */

/* Static storage of the function pointer
 * for the entire lifetime of the library
 */
static int (*print_to_stdout_fptr)(const char*);

int SysShell_init(int (*print_to_stdout)(const char*)) {
    if (NULL == print_to_stdout) {
        return -1;
    }

    print_to_stdout_fptr = print_to_stdout;
}
```

The user should initialize the library as follows:
```c
int Uart_SendString(const char* str) {
    /* Custom implementation for putting the str to UART */
    return 0;
}

int main() {
    SysShell_init(Uart_SendString)
    return 0;
}
```

After the initialization is done, one of the library functions could look like this:
```c
int SysShell_SendWelcomeStr() {
    /* Send welcome string to standard output.
     * Standard output print functionality is provided by the
     * user and stored in print_to_stdout_fptr function pointer.
     */
    const char* welcome_str =
        "----------- Sys Shell -----------\r\n"
        "Type 'help' for more information\r\n"
        "> ";
    print_to_stdout_fptr(welcome_str);
}
```

This is a well-known and common approach. However, if you're not careful, there are some inconveniences or risks involved:
* The user of the library must make sure that `SysShell_init` is called before calling any other function of the library; otherwise, the library will not function properly.
* Passing only one function pointer is easy. However, if we have many platform-dependent functionalities that need to be passed as function pointers, the prototype of `SysShell_init` might become very long and messy, especially given the tricky function pointer syntax.
* The library requires a function pointer that takes a `const char*` string, which means that the user's implementation cannot modify the contents of the string. If we do not set strict compiler warnings, the code will compile with `int Uart_SendString(char* str)` as well, meaning that `Uart_SendString` can modify the contents of the string. This might not be expected by the library, as it could pass a string stored in a read-only location, leading to program crashes if the user's implementation tries to modify the passed string.

As mentioned above, using function pointers as an interface for portable drivers and libraries is a common practice in C programming. Below is just another way of implementing portability by utilizing the `_Generic` keyword.

### Alternative Solution: Preprocessor Definition, but with Type Safety Checks

In this solution, our library doesn't require a `SysShell_init` function to get interface function pointers as parameters. Instead, we only need the following definition in the header file. The header file can be part of the library or a specific header file provided by the user:
```c
#define SYS_SHELL_SEND_STR Uart_SendString
```

Then, `SysShell_SendWelcomeStr` uses the `SYS_SHELL_SEND_STR` definition:
```c
int SysShell_SendWelcomeStr(void) {
    const char* welcome_str =
        "----------- Sys Shell -----------\r\n"
        "Type 'help' for more information\r\n"
        "> ";
    SYS_SHELL_SEND_STR(welcome_str);
}
```
Okay, but after all, what's so special here? We all know that using the preprocessor, we can replace text so that a generic function name can be transformed into our platform-specific function name. The problem is that these preprocessor definitions can be error-prone. You can "assign" anything to `SYS_SHELL_SEND_STR`, and if the function prototype is completely wrong, then the build will surely fail. However, in the case of more subtle but dangerous mismatches, such as the already mentioned `char*` instead of `const char*`, the result might be unpredictable.

The use of `_Generic` allows us to ensure that the user implementation and the function signature are exactly as expected.

First, let's define a macro that checks if `expr` is of `expected_type`. It evaluates to 1 when the expression type matches the expected type, and 0 otherwise:
```c
#define TYPE_CHECK(expr, expected_type)\
        _Generic((expr), expected_type: 1, default: 0)

/* Our interface definition */
#define SYS_SHELL_SEND_STR Uart_SendString

/* TYPE_CHECK(SYS_SHELL_SEND_STR, int (*)(const char*)) will:
 *  - return 1 if Uart_SendString signature is
      "int Uart_SendString(const char* str)"
 *  - return 0 if Uart_SendString is of any other type,
      including function with non-const char* parameter:
      "int Uart_SendString(char* str)"
 */
```

Then we can use a static assertion, which evaluates the `TYPE_CHECK` at compile time:

```c
#include <assert.h>

static_assert(TYPE_CHECK(SYS_SHELL_SEND_STR, int (*)(const char*)),
              "Wrong SYS_SHELL_SEND_STR pointer type");
```
When a bad interface is defined, the compiler will display an error message that we write in the assertion, making it very clear for the user what is wrong. For example, the GCC error output in case of a failed assertion would be:
```console
error: static assertion failed: "Wrong SYS_SHELL_SEND_STR pointer type"
  149 | static_assert(TYPE_CHECK(SYS_SHELL_SEND_STR, int (*)(const char*)),
      | ^~~~~~~~~~~~~
```
The advantages of using the preprocessor, `_Generic`, and `static_assert` in comparison with passing function pointers to the library are:
* We do not need to implement an initialization function to pass interface function pointers, which can have complicated function prototype syntax when many function pointers are needed.
* We don't need to write NULL pointer checks everywhere in the library to ensure that the user provided a valid function pointer.
* We do not need to allocate memory for storing the pointers.
* We can check the interface function types at compile time.

The drawback of using preprocessor definitions for the interface definition is the handling of the header. The user needs to create a special header file where all the definitions are stored, and the file name must match the name expected by the library.

If you want to experiment more with the example, here's the complete code that uses `_Generic` and `static_assert`:
```c
#include <stdio.h>
#include <assert.h>

#define TYPE_CHECK(expr, expected_type)\
        _Generic((expr), expected_type: 1, default: 0)

int Uart_SendString(const char* str) {
    printf("%s", str);
    printf("\n");
    return 0;
}

#define SYS_SHELL_SEND_STR Uart_SendString

static_assert(TYPE_CHECK(SYS_SHELL_SEND_STR, int (*)(const char*)),
              "Wrong SYS_SHELL_SEND_STR pointer type");

void SysShell_SendWelcomeStr(void) {
    const char* welcome_str = "----------- Sys Shell -----------\r\n"
                              "Type 'help' for more information\r\n"
                              "> ";

    SYS_SHELL_SEND_STR(welcome_str);
}

int main() {
    SysShell_SendWelcomeStr();
    return 0;
}
```
This code produces the following output:
```console
----------- Sys Shell -----------
Type 'help' for more information
>
```
