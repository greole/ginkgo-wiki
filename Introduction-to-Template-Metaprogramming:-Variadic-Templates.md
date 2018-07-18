C++11 enabled templates with variable number of parameters ("variadic
templates") by introducing "template parameter packs". A template parameter
pack is a list of template parameters of the same kind (type, any type of
integer, point, or enumeration) and is specified by appending an ellipsis
(`...`) to the kind of the type in the parameter declaration. Template
parameter packs can be used to write more general templates. A simple example
is the `void_type` type template and `void_t` type alias template (this is the
version in the C++ standard), which converts any number of arguments into
`void`:

__\[[try it](https://godbolt.org/g/V8kQaj)\]__
```c++
template <typename... Ts> struct void_type { using type = void; };

static_assert(is_same<void_type<int, float, bool>::type, void>::value, "");
static_assert(is_same<void_type<>::type, void>::value, "");
```

As with normal template parameters, names of template parameter packs can be
omitted if not used:

__\[[try it](https://godbolt.org/g/mBZQ6C)\]__
```c++
template<typename...> struct void_type { using type = void; };

static_assert(is_same<void_type<int, float, bool>::type, void>::value, "");
static_assert(is_same<void_type<>::type, void>::value, "");
```

Template parameter packs cannot be used directly, but have to be expanded into
a list of parameters by using the ellipsis operator. For example, parameter
pack expansion can be used to implement the `void_t` type alias template:

__\[[try it](https://godbolt.org/g/WFji3d)\]__
```c++
template <typename... Ts> using void_t = typename void_type<Ts...>::type;

static_assert(is_same<void_t<int, float, bool>, void>::value, "");
static_assert(is_same<void_t<>, void>::value, "");
```

`void_t` can be used to wrap multiple dummy parameters that use SFINAE into a
single parameter:

__\[[try it](https://godbolt.org/g/z7f7yW)\]__
```c++
// check if T has type members foo and bar
template <typename T, typename = void>
struct has_foo_and_bar : false_type {};

template <typename T>
struct has_foo_and_bar<T,
    void_t<typename T::foo, typename T::bar>> : true_type {};


struct X { using foo = int; };
struct Y { using bar = float; };
struct Z : X, Y {};

static_assert(!has_foo_and_bar<X>::value, "");
static_assert(!has_foo_and_bar<Y>::value, "");
static_assert(has_foo_and_bar<Z>::value, "");
```

The number of parameters in the parameter pack can be checked using the
`sizeof...` operator:

__\[[try it](https://godbolt.org/g/EYdWze)\]__
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

__\[[try it](https://godbolt.org/g/sgAfq4)\]__
```c++
template <typename T, typename...> using first_t = T;

static_assert(is_same<first_t<int, float, int>, int>::value, "");
static_assert(is_same<first_t<double, float, int>, double>::value, "");
```

Parameter packs can also be used in partial specializations. Usually, some of
the parameters of the parameter pack of the base template are matched by
individual parameters of the specialization, while the remaining ones by the
parameter pack wildcard of the specialization. This allows implementing
iteration over parameter packs, and processing individual parameters from the
pack. For example, the `is_same` metafunction can be extended to template 
parameter packs by comparing every two adjacent elements of the pack:

__\[[try it](https://godbolt.org/g/akmRgt)\]__
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

Specializations can contain more than one template parameter pack, but the
list after the name of the specialization still has the same restrictions on
parameter packs as the base template. However, this still means that more
complex patterns can be matched using parameter packs. For example, to check if
two `void_type` metafunctions received the same parameters, one could do the
following:

__\[[try it](https://godbolt.org/g/ccqfeV)\]__
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

__\[[try it](https://godbolt.org/g/LZxWVY)\]__
```c++
template <typename... Ts> struct num_args
    : integral_constant<int, sizeof...(Ts)> {};

template <typename... Ts> struct test1 : num_args<Ts...> {};
template <typename... Ts> struct test2 : num_args<void_t<Ts...>> {};
template <typename... Ts> struct test3 : num_args<void_t<Ts>...> {}; // <<<

static_assert(test1<int, int, int>::value == 3, ""); // int, int, int
static_assert(test2<int, int, int>::value == 1, ""); // void
static_assert(test3<int, int, int>::value == 3, ""); // void, void, void
```

Parameter packs can also be expanded in the inheritance list:

__\[[try it](https://godbolt.org/g/7UaL3F)\]__
```c++
template <typename... Ts> struct inherits : Ts... {};


struct X { using foo = void; };
struct Y { using bar = void; };
using foo = inherits<X, Y>::foo;
using bar = inherits<X, Y>::bar;
```

Finally, function templates can also use template parameter packs. Expanding a
template parameter pack inside the function parameter list will create a
function parameter pack, whose values get bound from the function argument
list. Similarly to template argument packs, the function parameter pack has to
be the last parameter of the function. Function parameter packs behave in the
same way as template parameter packs.

Unlike type and type alias templates, function templates can have parameters
after the template parameter pack, and those parameters can even be other
parameter packs, as long as all template parameters after the first template
parameter pack can be deduced from the function arguments.

Parameter packs in function templates can be used to implement type-safe
variadic functions:

__\[[try it](https://ideone.com/iiLS9w)\]__
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
struct concat {
    using type = Ts..., Us...;  // error - cannot alias a parameter pack
};
```

However, parameter packs can still be used for compile-time arrays if wrapped
into a type template, and combined with partial specialization to obtain the
elements from the pack:


__\[[try it](https://godbolt.org/g/R4PKJP)\]__
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
    back_t<type_array<int, bool, char>>,
    char>::value);
```

To implement value arrays, the values can simply be "typified" using
`integral_constant` and then added to `type_array`:

__\[[try it](https://godbolt.org/g/RYqVc5)\]__
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
which will be used to deduce the correct template parameters. These dummy
parameters will never get used in the function body, and will be optimized away
by the compiler. Their only purpose is to enable template parameters to be passed
in a more general way. For example, a function that takes a list of constant
coefficients packed into a type array and does a linear combination of these
coefficients with values known at runtime can be written as follows:

__\[[try it](https://ideone.com/TjtftS)\]__
```c++
template <typename FirstCoef, typename T>
T combine(type_array<FirstCoef>, const T &value) {
    return DETYPIFY(FirstCoef) * value;
}
 
template <typename FirstCoef, typename... OtherCoefs,
          typename T, typename... Ts>
T combine(
    type_array<FirstCoef, OtherCoefs...>, // dummy parameter
    const T &value, const Ts &... other_values) {
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

Congratulations for making it all the way to the end of this tutorial! You
should now have the basic understanding of templates, how they can be used to
reduce the amount of code that needs to be written, how to write metafunctions
that manipulate types or values and are evaluated at compile time, including
how to write conditional statements and iterations in metafunctons, and how to
use parameter packs to write variadic functions, and create metafunctions that
operate on sequences of values.

Additionally you have seen (in some cases simplified) implementations of useful
utilities that are also available in the C++ standard:

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
