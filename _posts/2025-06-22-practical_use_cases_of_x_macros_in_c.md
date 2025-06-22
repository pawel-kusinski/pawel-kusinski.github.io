---
title: "Practical Use Cases of X Macros in C"
categories:
  - Programming
tags:
  - C
---

## Introduction
An X macro is a powerful preprocessor technique in which we define a list of data entries.
This data list is then plugged into logic blocks that are executed once per data entry.
Using X macros can help minimize writing repetitive code. While macros are often considered hard to read, once you become familiar with this pattern, it's not that intimidating anymore. It's worth having this in your design toolbox, as separating data from logic and minimizing code repetition is always beneficial.
In this post, I'll present a couple of examples of X macro usage, and as usual, my examples are focused on embedded systems.

## Use Case 1: Enum to String

### "Manual" Conversions

Let's say we have the following enum defined:
```c
typedef enum {
    SPI_ERR_OK = 0,
    SPI_ERR_BUSY,
    SPI_ERR_TIMEOUT,
    SPI_ERR_INVALID_ARG,
    SPI_ERR_NOT_INITIALIZED
} SpiErr;
```

We use this enum to indicate different errors that can be reported by an SPI driver. Since it's an enumeration,
every error code is represented as an integer. When calling any SPI driver function, we can store the error
code in a variable and print it, so that we can track what is happening in our code.

```c
SpiErr err = Spi_init();

if (err) {
    printf("ERROR: Spi_init error %d\n", err);
}
```
This log message is definitely better than no log at all; however, unless you’ve memorized
your source code, you’ll need to refer to it to decode the numeric error value into
something meaningful.

To avoid this inconvenience, we can define a function that maps all `SpiErr`
members into strings that can be printed, so we can know at a glance the nature of the error.
This enum-to-string function is usually implemented using a switch-case statement:

```c
const char* SpiErrStr(SpiErr err) {
    switch(err) {
        case SPI_ERR_OK:
            return "OK";
        case SPI_ERR_BUSY:
            return "BUSY";
        case SPI_ERR_TIMEOUT:
            return "TIMEOUT";
        case SPI_ERR_INVALID_ARG:
            return "INVALID_ARG";
        case SPI_RERR_NOT_INITIALIZED:
            return "NOT_INITIALIZED";
        default:
            return "UNKNOWN";
    }
}
```

Now our log message is much more informative, and there's no need to refer
to the source code to find out what kind of error we got:

```c
SpiErr err = Spi_init();

if (err) {
    printf("ERROR: Spi_init error %s\n", SpiErrStr(err));
}
```

### The Problem

Great — we have good log messages now. But what if we want to add a new error?
We would have to enter almost the same information in two places. First, we’d need to add
another entry to the `SpiErr` enum. Then, we’d need to add another case to our `SpiErrStr`
function to cover the new error type. This can become annoying when the enum changes often.

It can also be error-prone — if we forget to add the new case to the `SpiErrStr` function,
then a known error will be printed as `"UNKNOWN"`, which may cause confusion.
It’s also possible to make a mistake and show the wrong string for an enum value,
especially if the function was written using copy-paste.

### The Solution

#### Defining the "Database"

Let’s solve this repetitive task with X macros! The idea is to define a piece of information only once.
In this case, the list of error codes. Each error code is defined using an X macro:

```c
#define SPI_ERR_LIST \
    X(SPI_ERR_OK) \
    X(SPI_ERR_BUSY) \
    X(SPI_ERR_TIMEOUT) \
    X(SPI_ERR_INVALID_ARG) \
    X(SPI_ERR_NOT_INITIALIZED)
```

We’ve defined our data — the list of error codes.
Now we can transform this into logic and definitions.

#### Generating "typedef enum"

For the enum-to-string use case, first we need to generate the enum definition:

```c
typedef enum {
#define X(name) name,
    SPI_ERR_LIST
#undef X
} SpiErr;
```

When the preprocessor parses this, it sees `typedef enum {` — no macro yet.
Then we define `X(name)` as `name,`. So when `SPI_ERR_LIST` is processed,
each `X(...)` line turns into an enum constant followed by a comma.
After that, we `#undef X` to avoid conflicts with future uses.
Then we close the enum with `} SpiErr;`.

The above will expand to:
```c
typedef enum {
    SPI_ERR_OK,
    SPI_ERR_BUSY,
    SPI_ERR_TIMEOUT,
    SPI_ERR_INVALID_ARG,
    SPI_ERR_NOT_INITIALIZED
} SpiErr;
```

#### Generating SpiErrStr Function
Previously we saw that the `SpiErrStr` function is just a bunch of cases
that return strings. That’s repetitive. With X macros:
```c
const char* spi_err_to_str(SpiErr err) {
    switch (err) {
#define X(name) case name: return #name;
        SPI_ERR_LIST
#undef X
        default: return "UNKNOWN";
    }
}
```
Here, `X(name)` expands to a case label and a return statement.
The `#` is the "stringizing operator" — it turns a macro parameter into a string literal.

So the preprocessed output will be:
```c
const char* SpiErrStr(SpiErr err) {
    switch(err) {
        case SPI_ERR_OK:
            return "SPI_ERR_OK";
        case SPI_ERR_BUSY:
            return "SPI_ERR_BUSY";
        case SPI_ERR_TIMEOUT:
            return "SPI_ERR_TIMEOUT";
        case SPI_ERR_INVALID_ARG:
            return "SPI_ERR_INVALID_ARG";
        case SPI_RERR_NOT_INITIALIZED:
            return "SPI_ERR_NOT_INITIALIZED";
        default:
            return "UNKNOWN";
    }
}
```
But wait — those strings include the `SPI_ERR_` prefix.
If you're working on a resource-constrained system like a microcontroller, you might not want
those prefixes in the log output to save flash memory or reduce UART bandwidth.

Here’s a fix: define the error list without the prefix, and add it only in the enum:
```c
#define SPI_ERR_LIST \
    X(OK) \
    X(BUSY) \
    X(TIMEOUT) \
    X(INVALID_ARG) \
    X(NOT_INITIALIZED)

typedef enum {
#define X(name) SPI_ERR_ ## name,
    SPI_ERR_LIST
#undef X
} SpiErr;

const char* spi_err_to_str(SpiErr err) {
    switch (err) {
#define X(name) case SPI_ERR_ ## name: return #name;
        SPI_ERR_LIST
#undef X
        default: return "UNKNOWN";
    }
}
```
Now the enum values keep the prefix, but the strings do not.
One downside: SPI_ERR_ appears in two places. You might think to do:
```c
#define PREFIX SPI_ERR_
#define X(name) PREFIX ## name,
```
Unfortunately, that won’t work because of how the C preprocessor handles macro expansion.
You'd need macro indirection — a more advanced topic worth its own post.


## Use Case 2: GPIO Initialization

### Writing it Manually
Let’s say we want to initialize several GPIO pins on a microcontroller.
Each pin setup might require initialization, setting direction, setting state, and enabling pull resistors.

Using Raspberry Pi Pico SDK as an example:
```c
void gpio_init(uint gpio);
void gpio_set_dir(uint gpio, bool out);
void gpio_put(uint gpio, int value);
void gpio_pull_up(uint gpio);
void gpio_pull_down(uint gpio);
```

A naive, repetitive implementation might look like:
```c
void initGpio(void) {
    printf("Initializing " "LED_RED" " pin...\n");
    gpio_init(0);
    gpio_set_dir(0, 1);
    gpio_put(0, 1);

    /* more pins... */

    printf("Initializing " "LED_GREEN" " pin...\n");
    gpio_init(2);
    gpio_set_dir(2, 1);
    gpio_put(2, 0);
}
```
### The Problem
This kind of code is hard to maintain, easy to break with copy-paste mistakes,
and doesn’t separate data from logic.

Our goal is to keep all pin configuration data in one place and define logic only once.

### The Solution
This could also be solved using struct arrays, but here we’ll use X macros.

Our GPIO "database" contains:
* label (for logging)
* pin number
* direction (`IN` or `OUT`)
* initial state (for outputs)
* pull-up setting (for inputs)



```c
// Define  GPIO pin "database" as macro data

/*   label           pin  dir  init_state  pull_up */
#define BUTTON_TEST_BOARD_OUTPUT_PINS \
    X(LED_RED,       0,   OUT,  true,      false) \
    X(LED_GREEN,     2,   OUT,  false,     false)

/*   label           pin  dir  init_state  pull_up */
#define BUTTON_TEST_BOARD_INPUT_PINS  \
    X(OK_BUTTON,     20,  IN,  false,  true) \
    X(CANCEL_BUTTON, 26,  IN,  false,  false)

// Combine both lists into one
// so we can initialize all pins together
#define BUTTON_TEST_BOARD_ALL_PINS \
    BUTTON_TEST_BOARD_OUTPUT_PINS \
    BUTTON_TEST_BOARD_INPUT_PINS
```

We can now write initGpio() like this:
```c
void initGpio(void) {
    // First pass: common setup for both input and output pins.
    // This macro is redefined just for this block.
    //
    // We temporarily define macro `X(...)` to say:
    //   - Print what we're doing
    //   - Call gpio_init
    //   - Set direction (IN or OUT)
    //   - Set initial value (for OUT pins)
    // The macro list `BUTTON_TEST_BOARD_ALL_PINS` will now "call" X(...)
    // oncew per line — expanding into real C code.
#define X(label, pin, direction, init_state, pull_up) \
    printf("Initializing " #label " pin...\n"); \
    gpio_init(pin); \
    gpio_set_dir(pin, GPIO_ ## direction); \
    gpio_put(pin, init_state);

    // Expand each pin using the above X(...) definition
    BUTTON_TEST_BOARD_ALL_PINS

    // Clean up: always undef X after use to avoid accidental reuse
#undef X

    // Second pass: only needed for input pins.
    // So we only iterate over BUTTON_TEST_BOARD_INPUT_PINS.
    // Again, we redefine X(...) with new logic — this time for pull config.
#define X(label, pin, direction, init_state, pull_up) \
    printf("Initializing pin " #label " pull-up/pull-down...\n"); \
    if (pull_up) { \
        gpio_pull_up(pin); \
    } else { \
        gpio_pull_down(pin); \
    }

    // Expand each input pin using the new X(...) logic
    BUTTON_TEST_BOARD_INPUT_PINS

    // Always undef X after use
#undef X
}
```
Now your logic is clean and your pin definitions are centralized.
If you need to change a pin, you edit one line. No risk of mismatched or duplicated logic.

There is some overhead to set this pattern up, and macros can be intimidating,
especially for those unfamiliar with X macros. But once understood, it pays off in maintainability and clarity.

## Source Code
The code snippets used in this blog post can be found on my GitHub:

[github.com/pawel-kusinski](https://github.com/pawel-kusinski/blog-code-snippets/tree/main/x_macros)
