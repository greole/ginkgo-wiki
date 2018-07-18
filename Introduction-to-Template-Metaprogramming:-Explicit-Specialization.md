Previous: [Metafunctions](./Introduction-to-Template-Metaprogramming:-Metafunctions); Next: [Variadic Templates](./Introduction-to-Template-Metaprogramming:-Variadic-Templates)

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
blueprint for that particular set of template arguments.
An explicit specialization is defined by declaring a function / type template
with the same name as the original template, but by providing an empty list
of template parameters and listing the arguments for which the template is
specialized in angle brackets after the name of the function/type.

For example, the `mad` template we used before could be explicitly specialized
for `float` and `double` values to use the built-in compiler instruction for
combined multiply-add:

__\[[try it](https://godbolt.org/g/8aqqHy)\]__
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


int test(double &x, double &y, float &z) {
    x = mad(2.3, 1, 2.2);   // uses an implicit spec. of the base template
    y = mad(2.3, 1.0, 2.2); // uses the explicit double spec.
    z = mad(2.3f, 1.0f, 2.2f); // uses the explicit float spec.
}
```

In the above code the compiler will do the following to find out which
implementation to use:

1.  All overloads (both template, and non-template, but not explicit
    specializations!) of the `mad` function are considered and the best match
    is found according to rules described in [__\[todo.add.ref.to.standard\]__](),
    or less formally on
    [cppreference](https://en.cppreference.com/w/cpp/language/overload_resolution).
2.  If the best match is a function, that function is called.
3.  If it is a function template the template parameters are substituted with the
    deduced arguments, and all explicit specializations of that function template
    are considered. If one of them matches the substituted arguments, that
    specialization is instantiated, otherwise the compiler uses the base
    template to instantiate an implicit specialization.

Explicit specializations are not exclusive to function templates, but work with
type templates as well. The same rules for explicit specialization selection
work with type templates as with function templates (since types cannot
be overloaded, the first step is trivial). For example, the `atomic` class
template can be used to implement atomic operations on objects which will be
used from multiple threads. The default implementation has to provide the
possibility of atomically storing, and loading an object, which can be
implemented using mutexes:

__\[[try it](https://godbolt.org/g/5KcGEb)\]__
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
used. An explicit specialization can be created to handle this case more
efficiently. In addition, intrinsics for special operations may exist for this
type, so additional members can be added to the specialization (some members
may also be removed, though this is usually not very useful):

__\[[try it](https://godbolt.org/g/x5h2B3)\]__
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

Explicit specialization is the key to creating advanced metafunctions, as
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
and explicit specializations can be used as distinct `case` statements:

__\[[try it](https://godbolt.org/g/wPC1JJ)\]__
```c++
template <typename>
struct is_int {
    static constexpr auto value = false;
};

template <>
struct is_int<int> {
    static constexpr auto value = true;
};

static_assert(is_int<double>::value == false, "");
static_assert(is_int<int>::value == true, "");
```

The example can even be simplified by using `true_type` and `false_type`
classes defined previously:

__\[[try it](https://godbolt.org/g/q1QusS)\]__
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

__\[[try it](https://godbolt.org/g/GTG8TM)\]__
```c++
template <int N> struct fact :
    integral_constant<long, N * fact<N-1>::value> {};
template <> struct fact<0> :
    integral_constant<long, 1> {};

static_assert(fact<3>::value == 6, "");
static_assert(fact<5>::value == 120, "");
```

Similarly, the compiler can be instructed to generate the Fibonacci sequence:

__\[[try it](https://godbolt.org/g/aYf3LE)\]__
```c++
template <int N> struct fib :
    integral_constant<long, fib<N-1>::value + fib<N-2>::value> {};
template <> struct fib<0> : integral_constant<long, 0> {};
template <> struct fib<1> : integral_constant<long, 1> {};

static_assert(fib<5>::value == 5, "");
static_assert(fib<6>::value == 8, "");
```

For simple recursive metafunctions described here, C++11's `constexpr` functions
can be used instead, but there are cases (e.g. when iterating through a complex,
recursive type) where this is not the case. Here is how the same can be achieved
with `constexpr` functions:

__\[[try it](https://godbolt.org/g/oskNGr)\]__
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
beginning of this tutorial and used in `uniqe_ptr` do not work well for
variable size arrays, since `delete` will be called instead of `delete[]` when
deallocating memory.  It is possible to create a specialization for each type
of variable size array separately (e.g. `int[]`, `float[]`, `double[]`), but
this would lead to a large amount of code duplication, and it would never be
possible to cover all cases.

For this reason, C++ also allows "partial" (explicit) specializations which can
be used to provide an alternative implementation for an entire class of
template arguments. Partial specializations are defined in the same way as full
specializations, but the template parameter list does not have to be empty, but
are used to provide "wildcards" in the argument list of the specialization.
Thus, parameters used in that list are never specified when instantiating the
template, but only appear in the list of template arguments after the
identifier. The compiler has to be able to automatically deduce their
substitutions from the template arguments (in the same way as with function
template arguments) in order for the specialization to be valid.

A partial specialization for the implementation of `unique_ptr` from the
beginning of this tutorial can be added to properly handle variable-size
arrays:

__\[[try it](https://ideone.com/CzLHU2)\]__
```c++
// partial specialization for arrays of default_delete
template <typename T>
struct default_delete<T[]> {
    void operator()(T arr[]) {
        delete[] arr;
    }
};
 
// partial specialization of uniqe_ptr for arrays
// (to avoid adding another pointer layer to the array)
template <typename T, typename Deleter>
class unique_ptr<T[], Deleter> {
public:
    unique_ptr(T resource[] = nullptr, Deleter deleter = Deleter{})
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
    template into the wildcards of the partial specialization. E.g. for a
    partial specialization of `foo` with template parameter `T`, and wildcard
    `T[]`:
    ```c++
    template <typename T> foo {};
    template <typename T> foo<T[]> {};
    ```
    if the substitution in the base template was `T/int[]`, the substitution in
    the partial specialization will be `T/int` (deduced by substituting `T[]/int[]`).
3.  If no applicable specializations are found, the base template is used to
    generate an implicit specialization for that substitution.
4.  Otherwise, the "most specialized" (as defined in
    [__\[todo.reference.to.standard\]__](), or less formally on
    [cppreference](https://en.cppreference.com/w/cpp/language/partial_specialization#Partial_ordering))
    explicit specialization is used (if there are more equaly specialized
    specializations, the program is ill-formed). If it is a full
    specialization, that specialization is used. If it is a partial
    specialization, the template parameters in that specialization are
    substituted with substitutions from step 2, and an implicit (full)
    specialization for those arguments is created from the partial specialization
    template.

Creating advanced metafunctions using partial specialization
------------------------------------------------------------

Partial specialization can be used to create more complex metafunctions.
For example, the `is_same` metafunction that is used for testing throughout
this tutorial can be created to check if two types are the same
(this is also a generalization of the `is_int` metafunction that works for
any type):

__\[[try it](https://godbolt.org/g/8CWhhe)\]__
```c++
template <typename T, typename U> struct is_same : false_type {};
template <typename T> struct is_same<T, T> : true_type {};

// the implementation of is_int can now be simplified:
template <typename T> struct is_int : is_same<T, int> {};


static_assert(!is_same<int, float>::value, "");
static_assert(is_same<int, int>::value, "");
static_assert(!is_same<int, int&>::value, "");

static_assert(is_int<int>::value, "");
static_assert(!is_int<int&>::value, "");
```

As a second example, the `remove_extent` metafunction removes the last
dimension of the array, if the input is an array, or leaves the input type
unchanged otherwise:

__\[[try it](https://godbolt.org/g/TLi95B)\]__
```c++
template <typename T> struct remove_extent { using type = T; };
// specialization for variable-size arrays
template <typename T> struct remove_extent<T[]> { using type = T; };
// specialization for fixed-size arrays
template <typename T, int N> struct remove_extent<T[N]> { using type = T; };


static_assert(is_same<remove_extent<int[]>::type, int>::value, "");
static_assert(is_same<remove_extent<int[5]>::type, int>::value, "");
// no extent to remove
static_assert(is_same<remove_extent<int>::type, int>::value, "");
static_assert(is_same<remove_extent<int[5][3]>::type, int[3]>::value, "");
static_assert(is_same<remove_extent<int[][3]>::type, int[3]>::value, "");
```

The third example combines iteration with partial specialization to count the
number of dimensions of an array (and is an example of a metafunction that
returns values, and cannot be implemented using `constexpr` functions):

__\[[try it](https://godbolt.org/g/umNXFS)\]__
```c++
template <typename T> struct rank : integral_constant<int, 0> {};
template <typename T> struct rank<T[]> :
    integral_constant<int, rank<T>::value + 1> {};
template <typename T, int N> struct rank<T[N]> : rank<T[]> {};


static_assert(rank<int>::value == 0, "");
static_assert(rank<int[]>::value == 1, "");
static_assert(rank<int[3]>::value == 1, "");
static_assert(rank<int[][2][3][5]>::value == 4, "");
```

Finally, partial specialization can be used to implement compile-time linked
lists and operations on them:

__\[[try it](https://godbolt.org/g/umNXFS)\]__
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


using single_element = push_front<void, int>::type;
static_assert(is_same<single_element, node<int, void>>::value);
using list1 =
    push_front<push_front<single_element, float>::type, double>::type;
static_assert(is_same<list1, node<double, node<float, node<int, void>>>>::value);
static_assert(is_same<front<list1>::type, double>::value);
using list2 = pop_front<list1>::type;
static_assert(is_same<list2, node<float, node<int, void>>>::value);

static_assert(size<single_element>::value == 1);
static_assert(size<list1>::value == 3);
static_assert(size<list2>::value == 2);
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

__\[[try it](https://godbolt.org/g/Gu2YqC)\]__
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


static_assert(is_same<
    make_sorted_list<double, int>::type,
    node<double, node<int>>>::value, "");
static_assert(is_same<
    make_sorted_list<double, int>::type,
    node<double, node<int>>>::value, "");
```

SFINAE (Substitution Failure Is Not An Error)
---------------------------------------------

Substituting arguments into template parameters may result in invalid code.
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
to the declaration, and a substitution failure in the definition __does__ result
in an error.

One of the most common uses of SFINAE is to query various properties of types.
For example, to query if a class `T` has a member type alias called `foo` of
a specific type, a metafunction can be created whose base template returns
false, and a specialization that returns true, but its declaration is only
valid if `T` has a member called `foo`:

__\[[try it](https://godbolt.org/g/8jKmt9)\]__
```c++
template <typename, typename> struct has_foo_of_type : false_type {};
template <typename T> struct has_foo_of_type<T, typename T::foo> : true_type {};


struct X { using foo = int; };
struct Y {};

static_assert(has_foo_of_type<X, int>::value, "");
static_assert(!has_foo_of_type<X, float>::value, "");
static_assert(!has_foo_of_type<Y, int>::value, "");
```

Checking if `T` only has a type member `foo` of any type is somewhat more
complicated. Fortunately, the `void_type` metafunction finally shows its
usefulness. This function can be used to convert any type into a void type, so
the call to `foo` in the specialization only has to be converted into void, and
the default argument for the second parameter set to void.

__\[[try it](https://godbolt.org/g/BYw1px)\]__
```c++
template <typename, typename = void> struct has_foo : false_type {};
template <typename T> struct has_foo<T,
    typename void_type<typename T::foo>::type> : true_type {};


struct X { using foo = int; };
struct Y { using foo = float; };
struct Z {};

static_assert(has_foo<X>::value);
static_assert(has_foo<Y>::value);
static_assert(!has_foo<Z>::value);
```

A similar idea with a dummy template parameter can be used to check if a type
supports a certain operation. For example, to see if `+` is supported by a
type, the base template can just be defined as `false_type`, and an expression
including the `+` operation is inserted via `decltype` into the declaration of
the partial specialization.

__\[[try it](https://godbolt.org/g/8kf9vt)\]__
```c++
template <typename, typename = void> struct has_add : false_type {};
template <typename T> struct has_add<T, typename void_type<
    decltype(declval<T>() + declval<T>())>::type> : true_type {};


struct X {};

static_assert(has_add<int>::value, "");
static_assert(!has_add<X>::value, "");
```

Implementing a compound, multiple choice `if` statement
-------------------------------------------------------

Even though nested `if`-`else` can be used to express a multiple choice `if`,
excessive nesting makes for less readable and harder to understand code,
especially in template metaprogramming, since each nesting level requires the
definition of a new identifier. Fortunately, a better alternative is possible
using SFINAE. The basic idea is to have each `if` branch of the compound
statement expressed as a partial specialization, and cause a substitution
failure if the condition is false. The base template represents the `else`
branch. To cause a substitution failure, we will define a utility type template
`enable_if`. `enable_if` will define a type member named `type` if its
input is `true`, and will not define that type member if its input is `false`:

__\[[try it](https://godbolt.org/g/3JNbke)\]__
```c++
template <bool, typename T = void> struct enable_if { using type = T; }
template <typename T> struct enable_if<false, T> {};


using t = enable_if<true>::type; // works
// using t2 = enable_if<false>::type; // fails to compile
```

Using `enable_if`, the general structure of the `if`-`elseif`-`else` statement
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

This more advanced version of the if statement can be used to implement the
merge algorithm, which, given two sorted lists, produces a new list containing
all the elements of the other two lists in the sorted order (we compare types
in the list by their size):

__\[[try it](https://godbolt.org/g/xFp18c)\]__
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


using list1 = node<int[1], node<int[2], node<int[5], node<int[7]>>>>;
using list2 = node<int[2], node<int[3], node<int[6], node<int[6]>>>>;
static_assert(is_same<
    merge<list1, list2>::type,
    node<int[1], node<int[2], node<int[2], node<int[3], node<int[5],
         node<int[6], node<int[6], node<int[7]>>>>>>>>>::value);
```

Merge is the first step to implementing the merge sort algorithm. The only
missing part is the split algorithm, which will split the list in two halves.
Split can be implemented as two separate metafunctions, one that only leaves the
first half of the values, and the other which removes the first half of the
list:

__\[[try it](https://godbolt.org/g/EBjSP2)\]__
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


static_assert(is_same<
    take<2, node<int, node<float, node<double>>>>::type,
    node<int, node<float>>>::value, "");
static_assert(is_same<
    remove<2, node<int, node<float, node<double>>>>::type,
    node<double>>::value, "");
```

Finally, with all the components in place, the merge sort algorithm can be
implemented as follows:

__\[[try it](https://godbolt.org/g/VGQpGi)\]__
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


using unsorted =
    node<int[3], node<int[2], node<int[5], node<int[1], node<int[4]>>>>>;
using sorted =
    node<int[1], node<int[2], node<int[3], node<int[4], node<int[5]>>>>>;
static_assert(is_same<merge_sort<unsorted>::type, sorted>::value);
```

Type alias templates
--------------------

In addition to function and class templates, C++11 introduced type alias
templates, which are templates that, when instantiated, produce a type alias:

__\[[try it](https://godbolt.org/g/HaACcJ)\]__
```c++
template <typename T> using void_t = typename void_type<T>::type;


static_assert(is_same<void_t<float>, void_type<float>::type>::value, "");
```

The nice part about type aliases is that even if their arguments are templates,
the compiler still knows that they produce a type, so there is no need to
specify the `typename` keyword when using them. In addition, they do allow us
to create a metafunction that returns an already existing type. This seems to
solve the problem with metafunctions that return types, and the `type` member
type alias convention: instead of creating a new type template, a type alias
template could be created instead.

Unfortunately, type alias templates do not provide as many features as type
templates, so they cannot always be used to replace type templates.
The major drawback is that they do not support any kind of explicit
specialization (neither full, nor partial), so they cannot be used to
implement anything that requires conditionals or iteration. In addition, older
versions of the standard (before C++14) do not specify how are unused arguments
handled. For example, the following implementation of `void_t` is not
guaranteed to trigger SFINAE behavior for its argument on older compilers:

```c++
template <typename T> using void_t = void;
```

Thus, for everything but simple cases, the implementation of the metafunction
is still done using type templates, and the resulting function is then wrapped
into a type alias template to provide a simpler interface. For example:

__\[[try it](https://godbolt.org/g/Cun5QW)\]__
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
template <int K, typename L>
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

using unsorted =
    node<int[3], node<int[2], node<int[5], node<int[1], node<int[4]>>>>>;
using sorted =
    node<int[1], node<int[2], node<int[3], node<int[4], node<int[5]>>>>>;
static_assert(is_same<merge_sort_t<unsorted>, sorted>::value);
```

`template template` parameters (TODO)
-------------------------------------

It is also possible to have templates that take template parameters
and instantiate the template parameter from within the definition of the
template. For example, this can be used to implement higher-order
metafunctions, like modifying the merge sort algorithm to take a custom
comparator for comparing the values.
For more details about `template template` parameters see
[cpprerence](https://en.cppreference.com/w/cpp/language/template_parameters).

Previous: [Metafunctions](./Introduction-to-Template-Metaprogramming:-Metafunctions); Next: [Variadic Templates](./Introduction-to-Template-Metaprogramming:-Variadic-Templates)