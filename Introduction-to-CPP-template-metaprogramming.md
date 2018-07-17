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
==========================

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

The Basics
==========

Assume a maximum of two integers, as well as two floating point numbers is
needed in a larger application One could write the following functions to
compute the maximums:

__\[[try it](https://godbolt.org/g/SR88ke)\]__
```c++
int max(int x, int y) {
    return x > y ? x : y;
}

float max(float x, float y) {
    return x > y ? x : y;
}

int main() {
    int mi = max(3, 5);
    float mf = max(3.2f, 1.2f);
    rest_of_code(mi, mf);
}
```

The compiler will figure out which function to call for which input types, and
generate the following assembly, containing the two versions of max:

```asm
max(int, int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     eax, DWORD PTR [rbp-4]
        cmp     eax, DWORD PTR [rbp-8]
        jle     .L2
        mov     eax, DWORD PTR [rbp-4]
        jmp     .L4
.L2:
        mov     eax, DWORD PTR [rbp-8]
.L4:
        pop     rbp
        ret
max(float, float):
        push    rbp
        mov     rbp, rsp
        movss   DWORD PTR [rbp-4], xmm0
        movss   DWORD PTR [rbp-8], xmm1
        movss   xmm0, DWORD PTR [rbp-4]
        comiss  xmm0, DWORD PTR [rbp-8]
        jbe     .L11
        movss   xmm0, DWORD PTR [rbp-4]
        jmp     .L9
.L11:
        movss   xmm0, DWORD PTR [rbp-8]
.L9:
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     esi, 5
        mov     edi, 3
        call    max(int, int)
        mov     DWORD PTR [rbp-4], eax
        movss   xmm1, DWORD PTR .LC0[rip]
        movss   xmm0, DWORD PTR .LC1[rip]
        call    max(float, float)
        movd    eax, xmm0
        mov     DWORD PTR [rbp-8], eax
        movss   xmm0, DWORD PTR [rbp-8]
        mov     eax, DWORD PTR [rbp-4]
        mov     edi, eax
        call    rest_of_code(int, float)
        mov     eax, 0
        leave
        ret
.LC0:
        .long   1067030938
.LC1:
        .long   1078774989
```

Obviously, the implementations of the two functions are equivalent, up to the
type of input arguments and the return type. Furthermore, the implementation
will be the same for any other datatype that supports the _less-than_ operator.
To remove this code duplication and support any datatype, we will write our
first metaprogram:

__\[[try it](https://godbolt.org/g/icUnxh)\]__
```c++
template <typename T>
T max(T x, T y) {
    return x > y ? x : y;
}

int main() {
    // TODO
}
```

Here, we create a "function template" that the compiler can use to instantiate
a function for any type. Metaprogramming in C++ is realized (almost) entirely
using templates. A template is created using the `template` keyword, followed
by a list of comma separated template parameters, enclosed in _angle brackets_
(`< >`). The template in this example contains only a single parameter, which is
a type parameter (denoted by the keyword `typename`) and referred in the body of
the template by identifier `T`.

A function template is not a function, but a blueprint for a function. Thus, if
the above code is compiled (before completing the body of the main function),
the compiler will not generate any code for the `max` function template:

```asm
main:
        mov     eax, 0
        ret
```

A function is only generated when a template is _instantiated_ by providing 
substitutions for the template parameters as in the code below:

__\[[try it](https://godbolt.org/g/gkHX7U)\]__
```c++
template <typename T>
T max(T x, T y) {
    return x > y ? x : y;
}

int main() {
    int mi = max<int>(3, 5);
    float mf = max<float>(3.2f, 1.2f);
    rest_of_code(mi, mf);
}
```

In this example the `max` template is instantiated first by substituting the
parameter `T` with `int` (denoted by `T/int`), and then by substituting `T`
with `float` (`T/float`). The compiler will generate a separate _implicit
specialization_ of a function called `max` from the template `max` for each
distinct combination of template argument substitutions used in the program.
Thus, the resulting binary will contain two `max` functions corresponding to
the two substitutions, as can be seen in the assembly:

```asm
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     esi, 5
        mov     edi, 3
        call    int max<int>(int, int)
        mov     DWORD PTR [rbp-4], eax
        movss   xmm1, DWORD PTR .LC0[rip]
        movss   xmm0, DWORD PTR .LC1[rip]
        call    float max<float>(float, float)
        movd    eax, xmm0
        mov     DWORD PTR [rbp-8], eax
        movss   xmm0, DWORD PTR [rbp-8]
        mov     eax, DWORD PTR [rbp-4]
        mov     edi, eax
        call    rest_of_code(int, float)
        mov     eax, 0
        leave
        ret
int max<int>(int, int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     eax, DWORD PTR [rbp-4]
        cmp     eax, DWORD PTR [rbp-8]
        jle     .L4
        mov     eax, DWORD PTR [rbp-4]
        jmp     .L6
.L4:
        mov     eax, DWORD PTR [rbp-8]
.L6:
        pop     rbp
        ret
float max<float>(float, float):
        push    rbp
        mov     rbp, rsp
        movss   DWORD PTR [rbp-4], xmm0
        movss   DWORD PTR [rbp-8], xmm1
        movss   xmm0, DWORD PTR [rbp-4]
        comiss  xmm0, DWORD PTR [rbp-8]
        jbe     .L13
        movss   xmm0, DWORD PTR [rbp-4]
        jmp     .L11
.L13:
        movss   xmm0, DWORD PTR [rbp-8]
.L11:
        pop     rbp
        ret
.LC0:
        .long   1067030938
.LC1:
        .long   1078774989
```

Automatic template argument deduction
-------------------------------------

In some cases the compiler can automatically deduce (type) template arguments
from the types of function call arguments. This is the case in the above
example. Thus, the above code can be simplified to the following equivalent
form:

__\[[try it](https://godbolt.org/g/tc6H3b)\]__
```c++
template <typename T>
T max(T x, T y) {
    return x > y ? x : y;
}

int main() {
    int mi = max(3, 5);
    float mf = max(3.2f, 1.2f);
    rest_of_code(mi, mf);
}
```

The assembly generated is equivalent to the previous one, and the
correct version of the `max` function gets called:

```asm
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     esi, 5
        mov     edi, 3
        call    int max<int>(int, int)
        mov     DWORD PTR [rbp-4], eax
        movss   xmm1, DWORD PTR .LC0[rip]
        movss   xmm0, DWORD PTR .LC1[rip]
        call    float max<float>(float, float)
        movd    eax, xmm0
        mov     DWORD PTR [rbp-8], eax
        movss   xmm0, DWORD PTR [rbp-8]
        mov     eax, DWORD PTR [rbp-4]
        mov     edi, eax
        call    rest_of_code(int, float)
        mov     eax, 0
        leave
        ret
int max<int>(int, int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     eax, DWORD PTR [rbp-4]
        cmp     eax, DWORD PTR [rbp-8]
        jle     .L4
        mov     eax, DWORD PTR [rbp-4]
        jmp     .L6
.L4:
        mov     eax, DWORD PTR [rbp-8]
.L6:
        pop     rbp
        ret
float max<float>(float, float):
        push    rbp
        mov     rbp, rsp
        movss   DWORD PTR [rbp-4], xmm0
        movss   DWORD PTR [rbp-8], xmm1
        movss   xmm0, DWORD PTR [rbp-4]
        comiss  xmm0, DWORD PTR [rbp-8]
        jbe     .L13
        movss   xmm0, DWORD PTR [rbp-4]
        jmp     .L11
.L13:
        movss   xmm0, DWORD PTR [rbp-8]
.L11:
        pop     rbp
        ret
.LC0:
        .long   1067030938
.LC1:
        .long   1078774989
```

The exact situations when this can be done are formally described in
[__\[todo.reference.to.standard\]__]()
of the standard. A less formal (but possibly slightly inaccurate)
description can be found on [cppreference](https://en.cppreference.com/w/cpp/language/template_argument_deduction).

Type templates
--------------

In addition to function templates, it is also possible to create type
templates, which create a type when instantiated. For example, the following is
a type template which can be used to instantiate structures containing pairs of
types:

__\[[try it](https://godbolt.org/g/T9ipKx)\]__
```c++
template <typename T, typename U>
struct pair {
    pair() = default;
    pair(T f, U s) : first{f}, second{s} {}
    T first;
    U second;
};

int main() {
    // try commenting out the main function
    // and see the assembly output
    //*
    pair<int, float> p(3, 1.2f);
    rest_of_code(p.first, p.second);
    //*/
}
```

As with function templates, type templates are not types themselves. Only when
they are instantiated with actual template parameters does the compiler create
an implicit specialization (and compiles the methods of the type) with template
parameters substituted with the arguments passed to the template. As with
function templates, a distinct type is created for each unique combination of
substitutions used in the program.
Unlike function templates, (before C++17) there is no automatic template
argument deduction for type templates, so all template arguments have to be
passed to a type template.

Other kinds of template parameters
----------------------------------

Template parameters are not limited to types. Available kinds of arguments are:

- types (specified using the `typename` or `class` keyword)
- all types representing integers (`short`, `int`, `long`, `long long`, and
  `unsigned` variants)
- enumerations
- pointers

For example, a type template representing a fixed-size array which provides a
richer interface than the nativelly supported C array can be created by using a
type template parameter representing the type of elements stored in the array,
and an integer template parameter representing the size of the array:

__\[[try it](https://ideone.com/lbIjdo)\]__
```c++
template <typename T, int N>
class array {
public:
    array() = default;
    array(std::initializer_list<T> list) {
        std::copy(std::begin(list), std::end(list), data);
    }

    const T& operator[](int index) const { return data[index]; }
    T& operator[](int index) { return data[index]; }

    int size() const { return N; }

    const T* begin() const { return data; }
    T* begin() { return data; }

    const T* end() const { return data + N; }
    T* end() { return data + N; }

private:
    T data[N];
};

int main() {
    array<int, 5> a{1, 2, 3, 4, 5};
    int res = 0;
    for (auto x : a) {
        res += x; 
    }
    std::cout << res << std::endl;  // 15
}
```

Non-type template parameters have to be compile-time constants. Thus, the
following code will not work:

__\[[try it](https://godbolt.org/g/ks8NLJ)\]__
```c++
int main() {
    int size;
    std::cin >> size;
    array<int, size> a;
};
```

Default template parameters
---------------------------

Similarly to functions parameters, template parameters can also have default
arguments, which are used if an argument is not supplied for that parameter.
Default arguments are set by adding an equal sign after the parameter
identifier, followed by the default argument. Unlike default function
arguments, default template arguments can use previously specified template
parameters in their definition.

For example, when implementing a smart pointer that manages a resource and
automatically releases it using the RAII idiom, the user has to tell the
resource manager how the resource is to be released. This can be done by using
a _deleter_ functor, which will be called in the resource manager's destructor.
However, most of the time, the resource will be a dynamically allocated object,
which should be freed using the `delete` keyword.  Thus, it makes sense to use
a deleter that will call `delete` by default, and only require the user to
specify the deleter in case special logic is required when deleting the object.
Bellow is a simple implementation of such a resource manager template called
`unique_ptr`, and how it can be used to manage a dynamically allocated integer,
memory allocated using `malloc`, or a C-style file opened via `fopen`.

__\[[try it](https://ideone.com/GkqeGT)\]__
```c++
template <typename T>
struct default_delete {
    void operator ()(T *ptr) {
        delete ptr;
    }
};

template <typename T, typename Deleter = default_delete<T>>
class unique_ptr {
public:
    unique_ptr(T* resource = nullptr, Deleter deleter = Deleter{})
        : resource{resource}, deleter{deleter} {}

    unique_ptr(const unique_ptr&) = delete;
    unique_ptr(unique_ptr&& other) 
        : resource{std::move(other.resource)},
          deleter{std::move(other.deleter)} {}

    unique_ptr& operator=(const unique_ptr&) = delete;
    unique_ptr& operator=(unique_ptr&& other) {
        using std::swap;
        swap(other.resource, resource);
        swap(other.deleter, deleter);
    }

    ~unique_ptr() {
        deleter(resource);
    }

    T* get() { return resource; }
    const T* get() const { return resource; }

private:
    T* resource;
    Deleter deleter;
};

int main() {
    // using default deleter
    unique_ptr<int>  int_ptr(new int{});

    // using std::free to delete dynamically allocated memory
    unique_ptr<int, void(*)(void *)> array_ptr(
        reinterpret_cast<int *>(std::malloc(5 * sizeof(int))),
        std::free);

    // using std::fclose to close an opened file
    unique_ptr<FILE, int(*)(FILE *)> file_ptr(
        std::fopen("my_file.txt", "w"),
        std::fclose);

    // use int_ptr, array_ptr, file_ptr here

    // these resources get released here, at the end of scope
}
```

`decltype` and `declval`
------------------------

While not directly related to template metaprogramming, the `decltype` keyword
and the `declval` template function are most often used in conjunction with
them, so they are covered in this tutorial, to make sure they are well
understood before continuing with more complicated examples.

The `decltype` keyword is used to obtain the type of a variable or an
expression:

__\[[try it](https://godbolt.org/g/KmYCvr)\]__
```c++
int x = 15;
const decltype(x) &y = x;
static_assert(is_same<decltype(3), int>::value);
static_assert(is_same<decltype(2*x + 3.0*y), double>::value);
static_assert(is_same<decltype(x), int>::value);
static_assert(is_same<decltype(y), const int&>::value);
```

An important property of `decltype` is that the expression used as its argument
creates an _unevaluated context_. This means that the expression passed as
argument is never evaluated; only the type of the expression is computed. As a
result, functions and operators used in the expression do not have to be
defined, but only declared. Additionally, if a function is only used within
`decltype`, it does not have to be linked to the final executable.

While `decltype` does not make much sense in the example above (we know what
are the types of those expressions, so why not just use them directly?), its
usefulness becomes obvious when combining it with templates. For example,
let's look at a general `mad` function, which takes 3 arguments, multiplies the
first two, adds the third to the result, and returns the obtained value.
The question is what is the return type of this function.

```c++
template <typename T, typename U, typename V>
???? mad(T t, U u, V v) { return t * u + v; }
```

The answer is obviously "whatever the return type of `t * u + v` is". Thus,
`decltype` can be used to obtain this type:

__\[[try it](https://godbolt.org/g/RGLyn7)\]__
```c++
template <typename T, typename U, typename V>
decltype(t * u + v) mad(T t, U u, V v) { return t * u + v; }
```

Unfortunately, compiling the above code will result in an error since
parameters `t`, `u` and `v` are used before they are declared.
However, the types `T`, `U` and `V` are available. Thus, a possible solution
is to create temporary variables of those types in the argument to `decltype`
(they will not be evaluated as this is an _unevaluated context_):

__\[[try it](https://godbolt.org/g/zLDGUs)\]__
```c++
template <typename T, typename U, typename V>
decltype(T{} * U{} + V{}) mad(T t, U u, V v) { return t * u + v; }
```

This solution _usually_ works, but is not perfect as: 1) `T`, `U` or `V` may
not have a default constructor which will result in a compilation error and
2) this is not exactly equivalent to the original expression, as temporary
objects used in this solution are rvalues, while the named objects in the
function are lvalues.

Thus, a more general way of obtaining values from types (the reverse of
`decltype`) is needed. Fortunately, this does not require a new keyword, as
just a function template declaration is sufficient:

__\[[try it](https://godbolt.org/g/QyoorS)\]__
```c++
template <typename T>
T& declval();
```

Since `declval` will only be used in an unevaluated context, there is no need
to provide its implementation, which automatically solves problem 1), as now
constructors do not have to be called. Furthermore, 2) is also solved, since
the return type is `T&`, `declval<T>()` will be an lvalue to `T`, and if an
rvalue is needed, it can be obtained using `declval<T&&>()` (see [reference
collapsing rules](https://en.cppreference.com/w/cpp/language/reference#Reference_collapsing)).

With `declval`, the `mad` template can finally be properly implemented:

__\[[try it](https://godbolt.org/g/sDQBwK)\]__
```c++
template <typename T, typename U, typename V>
decltype(declval<T>() * declval<U>() + declval<V>()) mad(T t, U u, V v) {
    return t * u + v;
}
```

Since the above is a common problem in template metaprogramming, C++ provides
an alternative function definition syntax that removes the need for `declval`
in this case. Instead of providing the return type before the function
declaration, the declaration can begin with `auto`, and the return type
provided by using the arrow (`->`) operator and specifying the return type
after the function declaration (but before the function body):

__\[[try it](https://godbolt.org/g/rqz5ug)\]__
```c++
template <typename T, typename U, typename V>
auto mad(T t, U u, V v) -> decltype(t * u + v) { return t * u + v; }
```

(From C++14 onward, the compiler can also automatically deduce the return type
from the `return` statement, so providing the return type with the arrow
operator is optional.)

However, there are situations when variables of the required types are not
available, so using `declval` is the only viable option. The following "trait"
class which can be used to obtain return types of various operations is such
an example:

__\[[try it](https://godbolt.org/g/Wy2nto)\]__
```c++
template <typename T, typename U>
struct operation_result {
    using add_result = decltype(declval<T>() + declval<U>());
    using mul_result = decltype(declval<T>() * declval<U>());
    // ...
};
```

Metafunctions
=============

Let us take a closer look at type templates. For example:

```c++
template <typename T, int N>
struct foo {
    T t[N];
};
```

The type template `foo` can be instantiated with a type `T` and an integer `N`
to produce a new type `foo<T, N>`. Thus, `foo` can be viewed as a function
evaluated at compile time (a "metafunction"), which takes a type and an integer
parameter and produces a type result:

```
foo : type x int -> type
```

Thus, familiar concepts of functions and operations on functions (e.g. function
composition, evaluation) can be extended to templates. This is the first
step towards understanding templates not just as "blueprints" for the compiler,
but as a separate "metalanguage" within the language (C++) itself. The rest of
this tutorial will introduce advanced features of templates which can be used
to implement flow control and iteration in this metalanguage. However, the result
will be a purely functional language, so metaprograms written in it will look
quite a bit different than regular C++ programs (though avid Haskell lovers
will feel right at home).

Before delving into advanced features, we need to solve a small problem with
metafunctions: they always produce __new__ types, and it is impossible for a
type template to return a type already defined somewhere else. The reasons for
this drawback are mostly historical, as templates were in fact first designed to
provide "blueprints", and only later user for metaprogramming. Fortunately, there
is a simple workaround for this problem: metafunctions that want to return an
already existing type will return a new type (as that is why they have to do),
which only has a single type member. This member is a type alias for the already
existing type they actually return. By convention, this member will be called
`type`.

As an example, we can create a simple metafunction called `void_type` that
takes any type, and always returns `void`:

__\[[try it](https://godbolt.org/g/R4mNZ9)\]__
```c++
template <typename T>
struct void_type { using type = void; };

static_assert(is_same<void_type<int>::type, void>::value);
static_assert(is_same<void_type<const char *>::type, void>::value);
```

Even though this function seems quite useless at first glance, it will prove to
be a powerful tool later on. A variation of it has even been integrated
into the C++17 standard under the name `std::void_t`.

Also notice that the type `T` is not used. As in regular functions, it is
allowed to have unused parameters, and in that case the parameter identifier
can be omitted:

__\[[try it](https://godbolt.org/g/85LyQV)\]__
```c++
template<typename> struct void_type { using type = void; };
```

However, `void_type<int>` and `void_type<double>` are still distinct types,
even though `void_type<int>::type` and `void_type<double>::type` are aliases of
the same type:

__\[[try it](https://godbolt.org/g/pSDCy9)\]__
```c++
static_assert(is_same<void_type<int>, void_type<int>>::value);
static_assert(!is_same<void_type<int>, void_type<double>>::value);
```

Metafunctions that return values
--------------------------------

The idea from the previous section can be used to implement metafunctions
that return values by using a compile-time static member variable instead of a
member type alias. Since the C++11 standard, making sure that this variable
has been computed at compile time can be done by using the `constexpr` keyword.
The convention is to call this member `value`. For example, the following
metafunction returns the size of the larger object:

__\[[try it](https://godbolt.org/g/uYMt2N)\]__
```c++
template <typename U, typename T>
struct size_of_larger {
    static constexpr auto value = sizeof(U) > sizeof(T) ?
        sizeof(U) : sizeof(T);
};

static_assert(size_of_larger<int, double>::value == 8, "");
static_assert(size_of_larger<double, char>::value == 8, "");
```

`constexpr` functons
--------------------

If a metafunction contains only non-type parameters, and returns a non-type
result, an equivalent runtime version of the function can always be implemented.
For example, here is a function and a metafunction that sum two integers:

__\[[try it](https://ideone.com/N6Fdgj)\]__
```c++
int sum(int x, int y) { return x + y; }

template<int X, int Y>
struct msum {
    static constexpr auto value = X + Y;
};


volatile int x = 3;
volatile int y = 5;

int main() {
    assert(sum(x, y) == 8);
    static_assert(msum<3, 5>::value == 8);
}
```

As this can cause significant code duplication, C++11 introduced `constexpr`
functions, which combine the two implementations into one. If all the arguments
of such a function are compile-time constants, then the function will be
evaluated at compile-time, and its result will be a compile-time constant. The
syntax for writing a `constexpr` function is the same as for the regular
function, with the addition of the `constexpr` qualifier:

__\[[try it](https://ideone.com/hH8wzC)\]__
```c++
constexpr int sum(int x, int y) { return x + y; }
 
 
volatile int x = 3;
volatile int y = 5;
 
int main() {
    assert(sum(x, y) == 8);
    static_assert(sum(3, 5) == 8);
}
```

However, not every function can be easily transformed into a `constexpr`
function by just adding the `constexpr` keyword: a `constexpr` function can
only use other `contexpr` function in its implementation, and its body must be
compose of only a single (return) statement (the latter requirement has been
lifted in C++14).

Distinguishing between type and data members
--------------------------------------------

Types and values are used in different contexts, but the syntax for accessing
type members of a type is equivalent to the syntax for accessing its data members.
Usually, the compiler can only deduce if a member of a type template is a
data or a type member if the argument list does not contain template parameters.
If it does, the compiler assumes it is a data member. If it is a type member,
the `typename` keyword can be used to convey that information to the compiler.

__\[[try it](https://godbolt.org/g/3WKHy5)\]__
```c++
template <typename T>
struct foo {
    using baz = T;
};

template <typename T>
struct bar {
    static constexpr auto baz = T{};
};

// foo<int>::baz is a type
// bar<int>::baz is a value

template <typename T>
void test() {
    foo<int>::baz x;         // ok, the compiler knows foo<int>::baz is a type
    x = bar<int>::baz;       // ok, the compiler knows bar<int>::baz is a value
    // foo<T>::baz y;     // error, the compiler assumes foo<T>::baz is a value
    typename foo<T>::baz y;  // ok, explicitly saying foo<T>::baz is a type
    y = bar<T>::baz;         // ok, the compiler assume bar<T>::baz is a value
}
```

Why the compiler cannot deduce this correctly will become clear in the
[Explicit specialization](#Explicit-specialization) section. For now, it is enough
to remember this rule.

Argument binding and `integral_constant`
----------------------------------------

It is often useful to bind some arguments of a function to values and produce a
new function that has less parameters. Mathematically, for a function
`g : (x, y) -> g(x, y)`, one of its arguments can be bound to produce
`g_x : y -> g(x, y)` or `g_y : x -> g(x, y)`. Binding
metafunctions could be done by creating a new function that calls the original
function (similarly to the way this is done for regular functions):

__\[[try it](https://godbolt.org/g/Z7xpmQ)\]__
```c++
template <typename T, int N>
struct g {
    using type = T[N];
};

template <typename T>
struct g_5 {
    using type = typename g<T, 5>::type;
};

template <int N>
struct g_int {
    using type = typename g<int, N>::type;
};

static_assert(is_same<g_5<int>::type, g<int, 5>::type>::value, "");
static_assert(is_same<g_int<5>::type, g<int, 5>::type>::value, "");
```

However, with metafunctions the same can be achieved in a more
elegant way by exploiting inheritance:

__\[[try it](https://godbolt.org/g/ZbyTdV)\]__
```c++
template <typename T>
struct g_5 : g<T, 5> {};

template <int N>
struct g_int : g<int, N> {};
```

Here, `g_5` and `g_int` inherit the type `type` from `g<T, 5>` and
`g<int, N>` respectively, which then becomes their "return value".

A useful example of this is a metafunction `integral_constant` which maps a
type and a value of that type into that same value:

__\[[try it](https://godbolt.org/g/69ih4Z)\]__
```c++
template <typename T, T V>
struct integral_constant {
    static constexpr T value = V;
};

static_assert(is_same<
    decltype(integral_constant<int, 5>::value),
    const int>::value, "");
static_assert(integral_constant<int, 5>::value == 5, "");
```

This utility function can be used to simplify the implementation of
metafunctions which return a value, as they can simply inherit a specialization
of this class with the type set to the return type, and the value to the
expression that produces the return value.
Two further types that inherit from `integral_constant` are useful when
implementing functions that return a boolean:

__\[[try it](https://godbolt.org/g/w97b8C)\]__
```c++
struct true_type : integral_constant<bool, true> {};

struct false_type : integral_constant<bool, false> {};

static_assert(true_type::value == true, "");
static_assert(false_type::value == false, "");
```

`integral_constant` can also be used to "typify" a value (i.e. use a type to
encode a value) - more on that later.

Explicit specialization
=======================

The previous sections set the stage and introduced the concepts needed to write
metafunctions. This section describes the last missing piece of the puzzle
which will allow us to implement `if` statements and perform iteration needed
to write truly general metaprograms.

Enter "explicit specialization". The first section of this tutorial described
how template types and functions have to be instantiated by providing them with
arguments to produce actual types and functions (called "specializations"),
which can then be used in the program. This is done implicitly by the compiler
using the "blueprint" defined by the template. However, C++ allows us to
explicitly instantiate a particular specialization in order to change the
"blueprint" for that particular set of template arguments.
A specialization is done by declaring a function / type template with the same
name as the original template, but by providing an empty list of template
parameters and listing the arguments for which the template is specialized in
angle brackets after the name of the function/type.

For example, the `mad` template we used before could be explicitly specialized
for `float` and `double` values to use the built-in compiler instruction for
combined multiply-add:

```c++
template <typename T, typename U, typename V>
auto mad(T t, U u, V v) -> decltype(t * u + v) { return t * u + v; }

template <>
float mad<float, float, float>(float t, float u, float v) {
    return intrinsic_float_mad(t, u, v);
}

template <>
double mad<double, double, double>(double t, double u, double v) {
    return intrinsic_double_mad(t, u, v);
}


int main() {
    auto x = mad(2.3, 1, 2.2);   // uses an implicit spec. of the base template
    auto y = mad(2.3, 1.0, 2.2); // uses the explicit double spec.
    auto z = mad(2.3f, 1.0f, 2.2f); // uses the explicit float spec.
}
```

In the above code the compiler will do the following to find out which
implementation to use:

1.  All overloads (both template, and non-template, but not explicit
    specializations!) of the `mad` function are considered and the best match
    is found according to rules described in [standard-reference]().
2.  Once the best match is found, and template parameters substituted with
    arguments if the match is a template, it's explicit specializations are
    considered. If one of them matches the substituted arguments, that
    specialization is instantiated, otherwise the compiler uses the base
    template to instantiate an implicit specialization.

Explicit specializations ere not exclusive to function templates, but work with
type templates as well. The same rules for explicit specialization selection
work with type templates as with function templates (though since types cannot
be overloaded, the first step is trivial). For example, the `atomic` class
template can be used to implement atomic operations on objects which will be
used from multiple threads. The default implementation has to provide the
possibility of atomically storing, and loading an object, which can be
implemented using mutexes:

```c++
template <typename T>
class atomic {
public:
    atomic() = default;
    atomic(T value) : data{value} {} // construction is not atomic
    atomic(atomic&) = delete;

    T operator=(const T &value) {
        std::lock_guard<std::mutex> guard(m);
        data = value;
        return value;
    }

    operator T() {
        std::lock_guard<std::mutex> guard(m);
        auto tmp = data;
        return tmp;
    }

private:
    T data;
    std::mutex m;
};
```

However, for some built-in types (e.g. `int`) the hardware has special
instructions which are guaranteed to be atomic, so mutexes do not have to be
used, so an explicit specialization can be created to handle this case more
efficiently. In addition, intrinsics for special operations may exist for this
type, so additional members can be added to the specialization (some members
may also be removed, though this is usually not very useful):

```c++
template <>
class atomic<int> {
public:
    atomic() = default;
    atomic(int value) : data{value} {} 
    atomic(atomic&) = delete;

    int operator=(int value) {
        return data = value;
    }

    operator int() {
        return data;
    }

    int fetch_add(int value) {
        return intrinsic_atomic_add(&data, value);
    }

private:
    int data;
};
```

Implementing `switch`-like control flow in metafunctions
--------------------------------------------------------

Explicit specializations is the key to creating advanced metafunctions, as
they allow the selection of different implementations depending on the input
parameters, making control flow possible in metafunctions. For example, it can
be useful to figure out if a template argument is of a certain type (e.g. int).
A pseudocode for such a metafunction would look somewhat like this:

```c++
bool is_int(typename T) {
    switch T {
        case int: return true;
        default: return false;
    }
}
```

Unfortunately, the above is not valid code, but the same can be achieved with
explicit specializations. The base template can be used as the `default` case,
and explicit specializations can be used as distinct `case` statements.
Thus, the above can be implemented like this:

```c++
template <typename>
struct is_int {
    static constexpr auto value = false;
};

template <>
struct is_int<int> {
    static constexpr auto value = true;
};

int main() {
    std::cout << is_int<double>::value << std::endl; // false
    std::cout << is_int<int>::value << std::endl;    // true
}
```

The example can even be simplified by using `true_type` and `false_type`
classes defined previously:

```c++
template <typename> struct is_int : false_type {};
template <> struct is_int<int> : true_type {};
```

Using `switch`-like control flow to implement iteration
-------------------------------------------------------

As soon as control flow and functions are available in a language, iteration
can be implemented by creating recursive functions, with stopping conditions
expressed with control flow. For example, the factorial function can be
implemented as follows:

```c++
template <int N> struct fact :
    integral_constant<long, N * fact<N-1>::value> {};
template <> struct fact<0> :
    integral_constant<long, 1> {};

int main() {
    std::cout << fact<3>::value << std::endl;  // 6
    std::cout << fact<5>::value << std::endl;  // 120
};
```

Similarly, the compiler can be instructed to generate the Fibonnaci sequence:

```c++
template <int N> struct fib :
    integral_constant<long, fib<N-1>::value + fib<N-2>::value> {};
template <> struct fib<0> : integral_constant<long, 0> {};
template <> struct fib<1> : integral_constant<long, 1> {};

int main() {
    std::cout << fib<5>::value << std::endl;  // 5
    std::cout << fib<6>::value << std::endl;  // 8
}
```

However, for metafunctions dealing only with non-type parameters, C++11's
`constexpr` functions can be used instead:

```c++
constexpr long fact(int n) {
    return n == 0 ? 1 : n * fact(n - 1);
}

constexpr long fib(int n) {
    return n == 0 ? 0 : n == 1 ? 1 : fib(n-1) + fib(n-2);
}
```

Partial specialization
----------------------

While (full) explicit specialization allows different implementations for
particular combinations of template parameters, sometimes it is useful to have
a separate implementation for an entire class of template parameters. For
example, the `default_delete` and `unique_ptr` templates implemented at the
beginning of this tutorial and used in `uniqe_ptr` does not work well for C++
variable size arrays, since `delete` will be called instead of `delete[]` when
deallocating memory.  It is possible to create a specialization for each kind
of variable size array separately (e.g. `int[]`, `float[]`, `double[]`), but
this would lead to a large amount of code duplication, and it would never be
possible to cover all cases.

For this reason, C++ also allows "partial" (explicit) specializations which can
be used to provide an alternative implementation for an entire class of
template arguments. Partial specializations are created in the same way as full
specializations, but the template parameter list does not have to be empty.
Parameters used in that list are never specified when instantiating the
template, but should instead appear in the list of template arguments after the
identifier. The compiler has to be able to automatically deduce their
substitutions from the template arguments (in the same way as with function
template arguments) in order for the specialization to be valid.

A partial specialization for the implementation of `unique_ptr` from the
beginning of this tutorial can be added to properly handle variable-size
arrays:

```c++
// partial specialization for arrays of default_delete
template <typename T>
struct default_delete<T[]> {
    void operator()(T[] arr) {
        delete[] arr;
    }
};

// partial specialization of uniqe_ptr for arrays
// (to avoid adding another pointer layer to the array)
template <typename T, typename Deleter>
class unique_ptr<T[], Deleter> {
public:
    unique_ptr(T[] resource = nullptr, Deleter deleter = Deleter{})
        : resource{resource}, deleter{deleter} {}

    unique_ptr(const unique_ptr&) = delete;
    unique_ptr(unique_ptr&& other) 
        : resource{std::move(other.resource)},
          deleter{std::move(other.deleter)} {}

    unique_ptr& operator=(const unique_ptr&) = delete;
    unique_ptr& operator=(unique_ptr&& other) {
        using std::swap;
        swap(other.resource, resource);
        swap(other.deleter, deleter);
    }

    ~unique_ptr() {
        deleter(resource);
    }

    T[] get() { return resource; }
    const T[] get() const { return resource; }

private:
    T[] resource;
    Deleter deleter;
};

int main() {
    unique_ptr<int[]> arr_ptr(new int[5]); // uses the specialized version with
                                           // correct deleter
}
```

Partial specialization is not supported for function templates, but only for
class templates.  However, problems that partial specialization solves with
type templates can usually be solved for function templates using a combination
of overloading and SFINAE (explained later in this tutorial).

For class templates, the compiler will select the appropriate specialization as
follows:

1.  Arguments are substituted into the parameters of the base template, and
    default arguments are used for all unspecified parameters.
2.  Once every template parameter is substituted with an argument, all explicit
    specializations (either partial or full) are scanned for a matching
    argument list. In case of partal specializations, the parameters of the
    specializations are deduced by substituting the arguments of the base
    template into the arguments of the partial specialization. E.g. for a
    partial specialization of `foo` with template parameter `T`, and an
    argument `T[]`:
    ```c++
    template <typename T> foo {};
    template <typename T> foo<T[]> {};
    ```
    if the substitution in the base template was `T/int[]`, the substitution in
    the partial specialization wth argument `T[]` will be `T/int` (deduced by
    substituting `T[]/int[]`).
3.  If no applicable specializations are found, the base template is used to
    generate an implicit specialization for that substitution.
4.  Otherwise, the "most specialized" (as defined in [standard-reference]())
    explicit specialization is used (if there are more equaly specialized
    specializations, the program is ill-formed). If it is a full
    specialization, that specialization is used. If it is a partial
    specialization, the template parameters in that specialization are
    substituted with substitutions from step 2, and an implicit (full)
    specialization is created from the partial specialization template.

Creating advanced metafunctions using partial specialization
------------------------------------------------------------

Partial specialization can be used to create more complex metafunctions (and
first complex metafunctions in this tutorial without a simpler alternative).
For example, an `is_same` metafunction can be created to check if two types are
the same (this is a generalization of the `is_int` metafunction that works for
any type):

```c++
template <typename T, typename U> struct is_same : false_type {};
template <typename T> struct is_same<T, T> : true_type {};


// the implementation of is_int can now be simplified:
template <typename T> is_int : is_same<T, int> {};


int main() {
    std::cout << is_same<int, float>::value << std::endl; // false
    std::cout << is_same<int, int>::value << std::endl;   // true
    std::cout << is_same<int, int&>::value << std::endl;  // false
    std::cout << is_int<int&>::value << std::endl;        // false
}
```

As a second example, the `remove_extent` metafunction removes the last
dimension of the array, if the input is an array, or leaves the input type
unchanged otherwise:

```c++
template <typename T> struct remove_extent { using type = T; };
// specialization for variable-size arrays
template <typename T> struct remove_extent<T[]> { using type = T; };
// specialization for fixed-size arrays
template <typename T, int N> struct remove_extent<T[N]> { using type = T; };

int main() {
    assert(is_same<remove_extent<int[]>::type, int>::value);
    assert(is_same<remove_extent<int[5]>::type, int>::value);
    // no extent to remove
    assert(is_same<remove_extent<int>::type, int>::value);
    
    assert(is_same<remove_extent<int[5][3]>::type, int[5]>::value);
    assert(is_same<remove_extent<int[3][]>::type, int[3]>::value);
}
```

The third example combines iteration with partial specialization to count the
number of dimensions of an array:

```c++
template <typename T> rank : integral_constant<int, 0> {};
template <typename T> rank<T[]> :
    integral_constant<int, rank<T>::value + 1> {};
template <typename T, int N> rank<T[N]> : rank<T[]> {};


int main() {
    assert(rank<int>::value == 0);
    assert(rank<int[]>::value == 1);
    assert(rank<int[3]>::value == 1);
    assert(rank<int[2][3][5][]>::value == 4);
}
```

Finally, partial specialization can be used to implement compile-time linked
lists and operations on them:

```c++

// template representing the nodes of the list
template <typename, typename = void> struct node {};

// metafunction that adds elements to the beginning of the list
template <typename List, typename Value>
struct push_front {
    using type = node<Value, List>;
};

// metafunction that gets the first element of the list
template <typename List> struct front {};
template <typename Element, typename Next>
struct front<node<Element, Next>> {
    using type = Element;
};

// metafunction that removes the first element of the list
template <typename List> struct pop_front {};
template <typename Element, typename Next>
struct pop_front<node<Element, Next>> {
    using type = Next;
};

// metafunction that computes the size of the list
template <typename> struct size : integral_constant<int, 0> {};
template <typename Element, typename Next> struct size<node<Element, Next>> :
    integral_constant<int, size<Next>::value + 1> {};


int main() {
    using single_element = push_front<void, int>::type;
    assert(is_same<single_element, node<int, void>>::value);
    using list1 =
        push_front<push_front<single_element, float>::type, double>::type;
    assert(is_same<list1, node<double, node<float, node<int, void>>>>::value);
    assert(is_same<front<list1>::type, double>::value);
    using list2 = pop_front<list1>::type;
    assert(is_same<list2, node<float, node<int, void>>>::value);

    assert(size<single_element>::value == 1);
    assert(size<list1>::value == 3);
    assert(size<list2>::value == 2);
}
```

Implementing an `if`-`else` statement
-------------------------------------

The switch statement described earlier provides a way to choose an
implementation depending on a specific combination of template arguments.
However, sometimes it is necessary to select an implementation depending on
some expression being true or false. Usually, this is done using the
`if`-`else` control flow statement. While there is no such statement in the C++
template language, one can be emulated using partial specialization.
The basic idea is to use a dummy boolean template parameter, and default it to
the condition of the if statement. Then, one branch (usually the `if` branch)
is implemented in the base template, and the other one (usually the `else`
branch) in a partial specialization. The basic structure looks like this:

```c++
template </* parameters */, bool = (/* condition */)> struct foo {
    /* code to execute if condition evaluates to true */
};
template </* parameters */> struct foo</* parameters */, false> {
    /* code to execute if condition evaluates to false */
};
```

An example of this would be creating a compile-time list of two elements, with
elements sorted by their size:


```c++
//                                       v          v - not a closing bracket!
template <typename T, typename U, bool = (sizeof(T) >= sizeof(U))>
struct make_sorted_list {
    using type = node<T, node<U>>;
};
template <typename T, typename U>
struct make_sorted_list<T, U, false> {
    using type = node<U, node<T>>;
};


int main() {
    assert(is_same<make_sorted_list<double, int>::type,
                   make_sorted_list<int, double>::type
                  >::value);
}
```

SFINAE (Substitution Failure Is Not An Error)
---------------------------------------------

Substitution arguments into template parameters may result in invalid code.
For example, the substituted type may not have a member being requested, or
values of that type may not support the operations being applied to them.
The SFINAE principle states that failing to substitute an argument into a
parameter does not necessarily cause an error. In case of function templates,
the function template giving the error is discarded from the list of candidate
overloads, and the next best candidate is considered next. An error only occurs
if none of the overloads are valid candidates (or if multiple equally ranked
candidates are left at the end). In case of function templates, SFINAE applies
to both the function declaration and its definition.

For type templates, there is no overloads, but SFINAE still applies when
selecting the best specialization. In this case, however, SFINAE only applies
to the declaration, and a substitution failure in the definition does result in
an error.

One of the most common uses of SFINAE is to query various properties of types.
For example, to query if a class `T` has a member type alias called `foo` of
a specific type, a metafunction can be created whose base template returns
false, and a specialization that returns true, but its declaration is only
valid if `T` has a member called `foo`:

```c++
template <typename, typename> struct has_foo_of_type : false_type {};
template <typename T> struct has_foo_of_type<T, typename T::foo> : true_type {};

struct X {
    using foo = int;
};

struct Y {};

int main() {
    assert((has_foo_of_type<X, int>::value));
    assert((!has_foo_of_type<X, float>::value));
    assert((!has_foo_of_type<Y, int>::value));
}
```

Checking if `T` only has a type member `foo` of any type is somewhat more
complicated. Fortunately, the `void_type` metafunction finally shows its
usefulness. This function can be used to convert any type into a void type, so
the call to `foo` in the specialization only has to be converted into void, and
the default argument for the second parameter set to void.

```c++
template <typename, typename = void> struct has_foo : false_type {};
template <typename T> struct has_foo<T,
    typename void_type<typename T::foo>::type> : true_type {};

struct X { using foo = int; };
struct Y { using foo = float; };
struct Z {};

int main() {
    assert(has_foo<X>::value);
    assert(has_foo<Y>::value);
    assert(!has_foo<Z>::value);
}
```

A similar idea with a dummy template parameter can be used to check if a type
supports a certain operation. For example, to see if `+` is supported by a
type, the base template can just be defined as `false_type`, and an expression
including the `+` operation is inserted via `decltype` into the declaration of
the partial specialization.

```c++
template <typename, typename = void> struct has_add : false_type {};
template <typename T> struct has_add<T, typename void_type<
    decltype(declval<T>() + declval<T>())>::type> : true_type {};


struct X {};

int main() {
    assert(has_add<int>::value);
    assert(!has_add<X>::value);
}
```

Implementing an `if`-`elseif`-`else` statement
-----------------------------------------------

Even though nested `if`-`else` can be used to express `if`-`elseif`-`else`,
excessive nesting makes for less readable and harder to understand code,
especially in template metaprogramming, since each nesting level requires the
definition of a new identifier. Fortunately, a better alternative is possible
using SFINAE. The basic idea is to have each `if` branch of the compound
statement expressed as a partial specialization, and cause a substitution
failure if the condition is false. The base template represents the `else`
branch. Causing a substitution failure can be simplified with an
`enable_if`type template, which will define a type member named `type` if its
input is `true`, and will not define that type member if the input is `false`.


```c++
template <bool, typename T = void> struct enable_if { using type = T; }
template <typename T> struct enable_if<false, T> {};


int main() {
    using t = enable_if<true>::type; // works
    // using t2 = enable_if<false>::type; // fails to compile
}
```

Using `enable_if`, the general structure of the `if`-`else if`-`else` statement
looks as follows:

```c++
template </* parameters */, typename = void> struct foo {
    /* else branch */
};

template </* parameters */> struct foo</* parameters */,
        typename enable_if</* condition 1 */>::type> {
    /* condition 1 branch */
};

//...

template </* parameters */> struct foo</* parameters */,
        typename enable_if</* condition N */>::type> {
    /* condition N branch */
};
```

The more advanced version of the if statement can be used to implement the
merge algorithm, which, given two sorted lists, produces a new list containing
all the elements of the other two lists in the sorted order.

```c++
template <typename L1, typename L2, typename = void> struct merge {
    // nothing to do in the else branch
};

template <typename L1> struct merge<L1, void, void> {
    // if the second list is empty, the result is the first list
    using type = L1;
};

template <typename L2> struct merge<void, L2, void> {
    // if the first list is empty, the result is the second list
    using type = L2;
};

template <> struct merge<void, void, void>  {
    // need case for both list empty to avoid ambiguity
    using type = void;
};

template <typename L1, typename L2>
struct merge<L1, L2, typename enable_if<(
        sizeof(typename front<L1>::type) <=
        sizeof(typename front<L2>::type))>::type> {
    // if the first element of first list is smaller than the first element of
    // the second list, take the first element of the first list, and merge the
    // rest of the lists
    using type = node<
        typename front<L1>::type,
        typename merge<typename pop_front<L1>::type, L2>::type>;
};

template <typename L1, typename L2>
struct merge<L1, L2, typename enable_if<(
        sizeof(typename front<L1>::type) >
        sizeof(typename front<L2>::type))>::type> {
    // if the first element of first list is larger than the first element of
    // the second list, take the first element of the second list, and merge
    // the rest of the lists
    using type = node<
        typename front<L2>::type,
        typename merge<L1, typename pop_front<L2>::type>::type>;
};

int main() {
    using list1 = node<int[1], node<int[2], node<int[5], node<int[7]>>>>;
    using list2 = node<int[2], node<int[3], node<int[6], node<int[6]>>>>;
    static_assert(is_same<
        merge<list1, list2>::type,
        node<int[1], node<int[2], node<int[2], node<int[3], node<int[5],
            node<int[6], node<int[6], node<int[7]>>>>>>>>>::value);
}
```

Merge is the first step to implementing the merge sort algorithm. The only
missing part is the split algorithm, which will split the list in two halves.
Split can be implemented as two separate metafunctions, one that only takes the
first half of the values, and the other which removes the first half of the
list:

```c++
template <int K, typename L, typename = void> struct take {};

template <typename L> struct take<0, L, void> { using type = void; };

template <int K, typename First, typename Rest>
struct take<K, node<First, Rest>, typename enable_if<(K > 0)>::type> {
    using type = node<First, typename take<K - 1, Rest>::type>;
};


template <int K, typename L, typename = void> struct remove {};

template <typename L> struct remove<0, L, void> { using type = L; };

template <int K, typename First, typename Rest>
struct remove<K, node<First, Rest>, typename enable_if<(K > 0)>::type> {
    using type = typename remove<K - 1, Rest>::type;
};


int main() {
    static_assert(is_same<
        typename take<2, node<int, node<float, node<double>>>>::type,
        node<int, node<float>>>::value, "");
    static_assert(is_same<
        typename remove<2, node<int, node<float, node<double>>>>::type,
        node<double>>::value, "");
}
```

Finally, with all the components in place, the merge sort algorithm can be
implemented as follows:


```c++
template <typename L, typename = void> struct merge_sort {
    // the else branch will be the one with an empty or 1-element list - the
    // input list just has to be returned
    using type = L;
};

template <typename L>
struct merge_sort<L, typename enable_if<(size<L>::value > 1)>::type> {
    // if the list is non-trivial, it has to be split in half, the halves
    // sorted recursively, and merged back together
private:
    static constexpr auto list_size = size<L>::value;
    using first_half = typename merge_sort<
        typename take<list_size / 2, L>::type>::type;
    using second_half = typename merge_sort<
        typename remove<list_size / 2, L>::type>::type;
public:
    using type = typename merge<first_half, second_half>::type;
};


int main() {
    using unsorted =
        node<int[3], node<int[2], node<int[5], node<int[1], node<int[4]>>>>>;
    using sorted =
        node<int[1], node<int[2], node<int[3], node<int[4], node<int[5]>>>>>;
    static_assert(is_same<typename merge_sort<unsorted>::type, sorted>::value);
}
```

Type alias templates
--------------------

In addition to function and class templates, C++11 introduced type alias
templates, which are templates that, when instantiated, produce a type alias:

```c++
template <typename T> using void_t = typename void_type<T>::type;

int main() {
    static_assert(is_same<void_t<float>, typename void_type<float>::type>);
}
```

The nice part about type aliases is that even if their arguments are templates,
the compiler still knows that they produce a type, so there is no need to
specify the `typename` keyword when using them. In addition, they do allow us
to create a metafunction that returns an already existing type. This seems to
solve the problem with metafunctions that return types, and the `type` member
type alias convention: instead of creating a new type template, a type alias
template could be created instead.

Unfortunately, type alias templates do not provide as many features as type
templates, so they can only be used in some cases to replace type templates.
The major drawback is that type alias templates do not support any kind of
explicit specialization (neither full, nor partial), so they cannot be used to
implement anything that requires conditionals or iteration. In addition, older
versions of the standard (before C++14) do not specify how are unused arguments
handled. For example, the following implementation of `void_t` is not
guaranteed to trigger SFINAE behavior for its argument on older compilers:

```c++
template <typename T> using void_t = void;
```

Thus, for everything but simple cases, the implementation of the metafunction
is still done using type templates, and the resulting function is then wrapped
into a type alias template to provide a simple interface. For example:

```c++
// simplified interface for enable_if
template <bool B, typename T = void>
using enable_if_t = typename enable_if<B, T>::type;

// simplified interface for merge
template <typename L1, typename L2>
using merge_t = typename merge<L1, L2>::type;

// simplified interface for take
template <int K, typename L>
using take_t = typename take<K, L>::type;

// simplified interface for remove
template <ink K, typename L>
using remove_t = typename remove<K, L>::type;


// a simpler version of merge sort using type alias templates
template <typename L, typename = void> struct merge_sort {
    // the else branch will be the one with an empty or 1-element list - the
    // input list just has to be returned
    using type = L;
};

template <typename L>
struct merge_sort<L, enable_if_t<(size<L>::value > 1)>> {
    // if the list is non-trivial, it has to be split in half, the halves
    // sorted recursively, and merged back together
    using type = merge_t<
        typename merge_sort<take_t<size<L>::value / 2, L>>::type,
        typename merge_sort<remove_t<size<L>::value / 2, L>>::type>;
};

// simplified interface for merge_sort
template <typename L>
using merge_sort_t = typename merge_sort<L>::type;

int main() {
    using unsorted =
        node<int[3], node<int[2], node<int[5], node<int[1], node<int[4]>>>>>;
    using sorted =
        node<int[1], node<int[2], node<int[3], node<int[4], node<int[5]>>>>>;
    static_assert(is_same<merge_sort_t<unsorted>, sorted>::value);
}
```

`template template` parameters (TODO)
-------------------------------------

- modify merge sort to take a custom comparator

Variadic templates
==================

C++11 introduced templates with variable number of parameters ("variadic
templates") by introducing "template parameter packs". A template parameter
pack is a list of template parameters of the same kind (type, any type of
integer, point, or enumeration) and is specified by appending an ellipsis
(`...`) to the kind of the type in the parameter declaration. Template
parameter packs can be used to write more general templates. A simple example
is the `void_type` type template and `void_t` type alias template (this is the
version in the C++ standard), which converts any number of arguments into
`void`:

```c++
template <typename... Ts> struct void_type { using type = void; };

static_assert(is_same<void_type<int, float, bool>::type, void>::value, "");
static_assert(is_same<void_type<>::type, void>::value, "");
```

As with normal template parameters, names of template parameter packs can be
omitted if not used:

```c++
template<typename...> struct void_type { using type = void; };

static_assert(is_same<void_type<int, float, bool>::type, void>::value, "");
static_assert(is_same<void_type<>::type, void>::value, "");
```

Template parameter packs cannot be used directly, but have to be expanded into
a list of parameters by using the ellipsis operator. For example, parameter
pack expansion can be used to implement `void_t` type alias template:

```c++
template <typename... Ts> using void_t = typename void_type<Ts...>::type;

static_assert(is_same<void_t<int, float, bool>, void>::value, "");
static_assert(is_same<void_t<>, void>::value, "");
```

`void_t` can be used to wrap multiple dummy parameters that use SFINAE into a
single parameter:

```c++
// check if T has type members foo and bar
template <typename T, typename = void> has_foo_and_bar : false_type {};

template <typename T> has_foo_and_bar<T, void_t<
    decltype(typename T::foo),
    decltype(typename T::bar)>> : true_type {};


struct X { using foo = int; }
struct Y { using bar = float; }
struct Z : X, Y {};

static_assert(!has_foo_and_bar<X>::value, "");
static_assert(!has_foo_and_bar<Y>::value, "");
static_assert(has_foo_and_bar<Z>::value, "");
```

The number of parameters in the parameter pack can be checked using the
`sizeof...` operator:

```c++
template <typename... Ts> struct has_many_parameters :
    integral_constant<bool, (sizeof...(Ts) > 4)> {};

static_assert(has_many_parameters<int, bool, float, double, char>::value, "");
static_assert(!has_many_parameters<int, bool>::value, "");
```

A type template can have other parameters in addition to the parameter pack.
However, there can only be one parameter pack, and it has to be the last
parameter. This can be used to implement a metafunction which returns the first
of all the parameters passed to it:

```c++
template <typename T, typename...> using first = T;

static_assert(is_same<first<int, float, int>, int>::value, "");
static_assert(is_same<first<double, float, int>, double>::value, "");
```

Parameter packs can also be used in partial specialization. Usually, some of
the parameters of the parameter pack of the base template are mapped to
individual parameters of the specialization, while the remaining ones are
mapped to the parameter pack of the specialization. This allows implementing
iteration over parameter packs, and processing individual parameters from the
pack. For example, the `is_same` metafunction can be extended to template 
parameter packs by comparing every two adjacent elements of the pack:

```c++
template <typename...> struct is_same : true_type {};
template <typename T, typename... Rest>
struct is_same<T, T, Rest...> : is_same<T, Rest...> {};
template <typename T, typename U, typename... Rest>
struct is_same<T, U, Rest...> : false_type {};

static_assert(is_same<int>::value, "");
static_assert(is_same<int, int>::value, "");
static_assert(!is_same<int, float>::value, "");
static_assert(is_same<int, int, int>::value, "");
static_assert(!is_same<int, int, float>::value, "");
```

Specializations can contain more than one template parameter packs, but the
list after the name of the specialization still has the same restrictions on
parameter packs as the base template. However, this still means that more
complex patterns can be matched using parameter packs. For example, to check if
two `void_type` metafunctions received the same parameters, one could do the
following:

```c++
template <typename VoidType1, typename VoidType2>
struct have_same_params : true_type {};

template <typename T1, typename... Rest1, typename T2, typename... Rest2>
struct have_same_params<void_type<T1, Rest1...>, void_type<T2, Rest2...>>
    : false_type{};

template <typename T, typename... Rest1, typename... Rest2>
struct have_same_params<void_type<T, Rest1...>, void_type<T, Rest2...>>
    : have_same_params<void_type<Rest1...>, void_type<Rest2...>> {};


static_assert(
    have_same_params<void_type<int, int>, void_type<int, int>>::value);
static_assert(
    !have_same_params<void_type<int, int>, void_type<int, float>>::value);
```

Parameter transformations (e.g. applying a metafunction) can be applied to the
entire parameter pack, not just a single parameter. In this case, the result
will be a new pack, with the transformation applied to each parameter.

```c++
template <typename... Ts> struct num_args
    : integral_constant<int, sizeof...(Ts)> {};

template <typename... Ts> struct test1 : num_args<Ts...> {};
template <typename... Ts> struct test2 : num_args<void_t<Ts...>> {};
template <typename... Ts> struct test3 : num_args<void_t<Ts>...> {};

static_assert(test1<int, int, int>::value == 3, ""); // int, int, int
static_assert(test2<int, int, int>::value == 1, ""); // void
static_assert(test3<int, int, int>::value == 3, ""); // void, void, void
```

Parameter packs can also be expanded in the inheritance list:

```c++
template <typename... Ts> struct inherits : Ts... {};


struct X { using foo = void; };
struct Y { using bar = void; };

// check if inherits<X, Y> has foo and bar
template <typename T, typename = void> struct has_foo_and_bar : false_type {};
template <typename T> struct has_foo_and_bar<T, 
    void_t<typename T::foo, typename T::bar>> : true_type {};
static_assert(has_foo_and_bar<inherits<X, Y>>::value);
```

Finally, function templates can also use template parameter packs. Expanding a
template parameter pack inside the function parameter list will create a
function parameter pack, whose values get bound from the function argument
list. Similarly to template argument packs, the function parameter pack has to
be the last parameter of the function. Function parameter packs behave in the
same way as template parameter packs.

Unlike type and type alias templates, function templates can have parameter
after the template parameter pack, and those arguments can even be other
parameter packs, as long as all template parameters after the first template
parameter pack can be deduced from the function arguments.

Parameter packs in function templates can be used to implement type-safe
variadic functions:

```c++
template <typename T, typename... Rest> struct sum_type {
    using type = decltype(
        declval<T>() + declval<typename sum_type<Rest...>::type>());
};
template <typename T> struct sum_type<T> { using type = T; };

template <typename T> T sum(const T &x) { return x; }
template <typename T, typename... Rest>
typename sum_type<T, Rest...>::type sum(const T &x, const Rest &... rest) {
    return x + sum(rest...);
}


int main() {
    assert(sum(3, 2, 4) == 9);
    assert(sum(1, 2, 4, 4.5) == 11.5);
}
```

Compile-time type (and value) arrays
------------------------------------

Template parameter enable a better way to implement metafunctions on sequences
than lists implemented via the `node` template could. The only problem is that
it is not possible to pass multiple parameter packs to metafunctions, and that
it is impossible to return a parameter pack from a metafunction:

```c++
template <typename... Ts, typename... Us>  // error - multiple parameter packs
struct identity {
    using type = Ts..., Us...;  // error - cannot alias a parameter pack
};
```

However, parameter packs can stil be used for compile-time arrays if wrapped
into a type template, and combined with partial specialization to obtain the
elements from the pack:


```c++
// structure representing an array of types
template <typename...> struct type_array {};

// supported operations
template <typename...> struct push_front {};
template <typename... Params> using push_front_t =
    typename push_front<Params...>::type;

template <typename...> struct push_back {};
template <typename... Params> using push_back_t =
    typename push_back<Params...>::type;

template <typename...> struct concat {};
template <typename... Params> using concat_t =
    typename concat<Params...>::type;

template <typename...> struct size {};

template <typename...> struct front {};
template <typename... Params> using front_t =
    typename front<Params...>::type;

template <typename...> struct back {};
template <typename... Params> using back_t =
    typename back<Params...>::type;

// implementations of operations
template <typename Value, typename... Ts>
struct push_front<Value, type_array<Ts...>> {
    using type = type_array<Value, Ts...>;
};

template <typename Value, typename... Ts>
struct push_back<Value, type_array<Ts...>> {
    using type = type_array<Ts..., Value>;
};

template <typename... Ts1, typename... Ts2>
struct concat<type_array<Ts1...>, type_array<Ts2...>> {
    using type = type_array<Ts1..., Ts2...>;
};

template <typename... Ts>
struct size<type_array<Ts...>> : integral_constant<int, sizeof...(Ts)> {};

template <typename T, typename... Ts>
struct front<type_array<T, Ts...>> { using type = T; };

// this wont work, pack has to be last:
// template <typename... Ts, typename T>
// struct back<type_array<Ts..., T>>  { using type = T; };
template <typename T, typename... Ts>
struct back<type_array<T, Ts...>> : back<type_array<Ts...>> {};
template <typename T>
struct back<type_array<T>> { using type = T; };

// examples
static_assert(is_same<
    push_front_t<int, type_array<bool, char>>,
    type_array<int, bool, char>>::value, "");
static_assert(is_same<
    push_back_t<int, type_array<bool, char>>,
    type_array<bool, char, int>>::value, "");
static_assert(is_same<
    concat_t<type_array<int, bool>, type_array<char>>,
    type_array<int, bool, char>>::value, "");
static_assert(size<type_array<int, bool, char>>::value == 3);
static_assert(is_same<
    front_t<type_array<int, bool, char>>,
    int>::value);
static_assert(is_same<
    back_t<type_array
```

To implement value arrays, the values can simply be "typified" using
`integral_constant` and then added to `type_array`:

```c++
#define TYPIFY(x) integral_constant<decltype((x)), (x)>
#define DETYPIFY(x) x::value

using my_value_array = type_array<TYPIFY(3), TYPIFY(7)>;
static_assert(is_same<
    push_front_t<TYPIFY(5), my_value_array>,
    type_array<TYPIFY(5), TYPIFY(3), TYPIFY(7)>>::value);
static_assert(DETYPIFY(front_t<my_value_array>) == 3);
```

Passing type arrays to functions
--------------------------------

Handling of type arrays in metafunctions relies on partial specialization.
However, partial specialization is not supported by function templates.
Luckily, there is a way to work around this limitation. The idea is to not use
template arguments in functions directly, but to add dummy function parameters
which will be used to deduce the correct template parameters. These parameters
will never get used in the function body, and will be optimized-away by the
compiler. Their only purpose is to enable template parameters to be passed in a
more general way. For example, a function that takes a list of constant
coefficients packed into a type array and does a linear combination of these
coefficients with values known at runtime can be written as follows:

```c++
template <typename FirstCoef, typename... OtherCoefs,
          typename T, typename... Ts>
T combine(
    type_array<FirstCoef, OtherCoefs...>, // dummy parameter
    const T &value, const T &... other_values) {
    return DETYPIFY(FirstCoef) * value +
        combine(type_array<OtherCoefs...>{}, other_values...);
}


int main() {
    using coefs = type_array<TYPIFY(5), TYPIFY(2), TYPIFY(3)>;
    assert(combine(coefs{}, 1, 3, 2) == 5 + 3 * 2 + 3 * 2);
}
```

Conclusion
==========

Congratulations for making it all the way to the end of this tutorial. You
should now have the basic understanding of templates, how they can be used to
reduce the amount of code that needs to be written, how to write metafunctions
that manipulate types or values and are evaluated at compile time, including
how to write conditional statements and iterations in them, and how to use
parameter packs to write variadic functions, and create metafunctions that
operate on sequences of values.

Additionally you have seen (in some cases simplified) implementations of useful
functions that are also available in the C++ standard:

*   `std::max` (`<algorithm>`)
*   `std::pair` (`<utility>`)
*   `std::array` (`<array>`)
*   `std::default_delete` and `std::unique_ptr` (`<memory>`)
*   `std::declval` (`<utility>`)
*   `std::void_t`, `std::integral_constant`, `std::true_type`,
    `std::false_type`, `std::is_same`, `std::remove_extent`, `std::rank`
    (`<type_traits>`)
*   `std::atomic` (`<atomic>`)

The next steps in metaprogramming would include practicing writing your own
metaprograms, taking a look at the metafunctions available in the
[C++ `<type_traits>` standard header](https://en.cppreference.com/w/cpp/header/type_traits)
(trying to write some of them yourself is also a good practice of template
metaprogramming), and looking into some of the template metaprogramming
libraries available in
[boost](https://www.boost.org/doc/libs/?view=category_metaprogramming).
