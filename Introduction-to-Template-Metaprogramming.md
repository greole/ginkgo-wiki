The goal of this tutorial is to teach the basics of template metaprogramming in
C++. The term "metaprogramming" used here refers to instructing the compiler on
how to generate programs and then compile them, as opposed to standard
"programming" where the programmer itself writes the final program.
Metaprogramming is usually used when standard programming would include overly
repetitive and verbose programs, or when the final program is not known at the
time when the metaprogram is written (e.g. template libraries).

The target audience of this tutorial are programmers already familiar with C++,
but not yet familiar with templates and metaprogramming. Thus, the tutorial
assumes that the reader is familiar with features C++ inherits from C, as well
as C++ classes and its object-oriented facilities.

Utilities used for testing
--------------------------

In this tutorial we will often want to verify if the compiler generated the
correct code. Thus, we have to test something at compile time, as opposed to
runtime. The basic facility C++ provides to do that is the `static_assert`
keyword, which breaks the compilation if the (compile-time) expression
passed to it evaluates to false, and does nothing if it evaluates to true.
For example:

__\[[try it](https://godbolt.org/g/dUX9Mc)\]__
```c++
// continues compilation after the following line
static_assert(true, "<error message with details>");
// breaks compilation and returns the error message
static_assert(false, "<error message with details>");
```

Another utility we will often use in combination with static assert
is the `is_same<X, Y>::value` standard metafunction which returns `true` if
`X` and `Y` are the same types, and otherwise returns false.
We will also see how to implement this function ourselves in the section
about partial specialization.

__\[[try it](https://godbolt.org/g/UZQKg8)\]__
```c++
using my_int = int;
static_assert(is_same<my_int, int>::value, ""); // works
static_assert(is_same<my_int, float>::value, ""); // compilation error
```

Next: [The Basics](./Introduction-to-Template-Metaprogramming:-The-Basics)
