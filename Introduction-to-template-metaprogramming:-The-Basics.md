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