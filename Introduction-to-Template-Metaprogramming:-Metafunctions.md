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