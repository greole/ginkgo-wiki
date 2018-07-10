```c++
// Templates

// godbolt.org - compiler explorer (lets you compile code with different compiler and see the assembly)

// problem: code duplication, need to write `max` for every datatype we want to use it for
int max(int x, int y) {
    return x > y ? x : y;
}

float max(float x, float y) {
    return x > y ? x : y;
}

// solution: templates (template functions -
// lets you create a blueprint for a function, from which the compiler can generate
// an actual function)

template <typename T>
T tmax(T x, T y) {
    return x > y ? x : y;
}

// ^ the compile does not generate any code from the above template
// only when we call a "specialization" of a template, the compiler will instantiate a
// function using the template

int test1(int x, int y) {
   return tmax<int>(x, y);
}

char test2(char x, char y) {
    return tmax<char>(x, y);
}

// if you want to explicitly instatiate a template without calling it:

template float tmax<float>(float x, float y);

// Automatic template parameter deduction

// The compiler will look for all "candidates" with name "tmax", find that there is a
// template functions named "tmax" and figure out it can substitute T/double to get
// the template specialization for double
double test3(double x, double y) {
    return tmax(x, y);
}

// Careful:
double test4(double x, float y) {
    // return tmax(x, y);  // fails to compile, no matchig function tmax(double, float)
    return tmax(x, static_cast<double>(y));
}

// Template classes
// The same thing as template functions, but with classes

template <typename T>
struct array2 {
    int size();
    T data[2];
};

template <typename T>
int array2<T>::size() { return 2; }

int test5(int first, int second) {
    array2<int> arr;
    array2<double> arrd;
    return arr.size() + arrd.size();
}

// Template parameter do not have to be types
// - integers
// - pointer
// - enumerations

enum bar { one, two, three };

template<int X, char *s, bar z>
char* foo() {
    return s + X + z;
}

char* test6() {
    return foo<5, nullptr, three>();
}

// we can also mix type and non-type template parameters

template<typename T, int X, T s, bar z>
T foo2() {
    return s + X + z;
}

char * test7() {
    return foo2<char *, 5, nullptr, three>();
}

// Explicit specialization
template <typename T>
T max2(T x, T y) {
    return x > y ? x : y;
}

namespace std {
// Dummy implementation to avoid large compiler output
int strcmp(const char *, const char *) {
    return 1;
}
}

template<>
const char *max2<const char *>(const char *x, const char *y) {
    return std::strcmp(x, y) > 0 ? x : y;
}

const char * test8() {
    return max2("abc", "def");
}

// same with class templates

template <>
struct array2<int[2]> {
   int size();
   template <typename T>
   T whatever();
   int data[4];
};

// template <>  // means that size() is a template function, which is being specilized
int array2<int[2]>::size() { return 4; }

template <typename T>
T array2<int[2]>::whatever() { return T{}; }

template <>  // refers to the functions specialization, not to the class spec.
int array2<int[2]>::whatever<int>() { return 5; }

// Partial (explicit) specialization

// dummy implementation to avoid 
namespace std {
template <typename T, typename U>
struct pair {
    pair() = default;
    pair(T f, U s) : first{f}, second{s} {}
    T first;
    U second;
};
}

template <typename T, typename U>
struct array2<std::pair<T, U>> {
   std::pair<T[2], U[2]> data;
};

template <typename T>
struct array2<std::pair<T, T>> {
   T data[4];
};

int test9() {
    array2<std::pair<int, float>> a;
    a.data.first[0] = 1;
    a.data.first[1] = 2;
    return a.data.first[0] + a.data.first[1];
}

void test10() {
    array2<std::pair<int, int>> x;
    x.data[2] = 5;  // the compiler selects the specialization where the least amount
                    // of substitutions is needed
}

// possible caveat

template <typename T>
struct range {
    T x;
};


template <typename T, typename U>
int foo3(T x, U y) {
    return 0;
}

template <typename T>
int foo3(range<T> x, range<T> y) {
    return 2;
}

template <typename T, typename U>
int foo3(range<T> x, range<U> y) {
    return 3;
}

int test11() {
    return foo3(range<int>{1}, range<int>{2});
}

int test12() {
    return foo3(range<int>{1}, range<double>{2.0}); 
}

// We can use template specialization to do computations at compile time:
template <int K>
struct fib {
    constexpr static auto value = fib<K-1>::value + fib<K-2>::value;
};

template <>
struct fib<0> {
    constexpr static auto value = 0;
};

template <>
struct fib<1> {
    constexpr static auto value = 1;
};

int test13() {
    return fib<7>::value;
}

// Decltpye
// - a keyword
// - allows you to get the type of a variable

int test14() {
    int x, z;
    decltype(x + z) y = 5;
    return y;
}

struct A1 {};
struct B1 {};
struct C {};
struct D {};
D operator + (A1 a, B1 b) { return D{}; }

namespace std {
template <typename T>
T& declval();
}

// std::declval
//  - not a keyword, but a function (not implemented)
//  - gets you an lvalue of a specified type, in an unevalueted context
template <typename T, typename U>
decltype(std::declval<T>() + std::declval<U>()) foo10(T a, U b) {
    auto x = std::declval<T>();
    return a + b;
}

void test16() {
    int x = foo10(5, 5);
    D d = foo10(A1{}, B1{});
}

// SFINAE (Substitution Failure Is Not An Error)
// - if the compiler fais to instantiate a explicit specialization of a template,
//   it won't cause an error, but it will just fall back to another specialization

struct A {
    int foo() { return 2; }
};

struct E {
    bool foo() { return true; }
};

struct B {
    int bar() { return 3; }
};

int f(int) {
    return 5;
}


template <typename>
struct my_void_t {
    using type = void;
};

template <typename, typename = void>
struct has_foo {
    constexpr static auto value = false;
};
template <typename T>
struct has_foo<T, typename my_void_t<decltype(std::declval<T>().foo())>::type> {
    constexpr static auto value = true;
};

bool test17() {
    return has_foo<A>::value;  // true if it has foo, false otherwise
}

bool test18() {
    return has_foo<B>::value;  // true if it has foo, false otherwise    
}

bool test19() {
    return has_foo<E>::value;
}

// Using SFINAE to select an implementation
// - small object optimization

 // if condition is true, then my_enable_if will have a type member called "type"
 // otherwise, it will not have a type member called "type"
 // "type" will be void if it exists
template <bool condition, typename T = void>
struct my_enable_if {
    using type = T;
};
template <typename T>
struct my_enable_if<false, T> {};

template <typename T, typename = void>
struct container {
    static constexpr bool is_dynamically_allocated = true;
    container() : ptr(new T) {}
    ~container() {
        delete ptr;
    }
    
    T& get() { return *ptr; }
private:
    T *ptr;
};
template <typename T>
struct container<T, typename my_enable_if<sizeof(T) <= 8>::type> {
    static constexpr bool is_dynamically_allocated = false;
    container() {}
    T& get() { return obj; }
private:
    T obj;
};

struct big_object {
    int data[100];
};

bool test20() {
    container<int> x;
    x.get() = 5;
    return container<int>::is_dynamically_allocated;
}

bool test21() {
    container<big_object> y;
    y.get().data[0] = 10;
    return container<big_object>::is_dynamically_allocated;
}

int abs(int x) {
    return x > 0 ? x : -x;
}

enum with_modifier { no_modifier, and_do_abs };

template <with_modifier modifier = no_modifier, typename T>
typename my_enable_if<modifier == no_modifier>::type
increase_by_one(T *data, int size) {
    for (int i = 0; i < size; ++i) {
        ++data[i];
    }
}

template <with_modifier modifier = no_modifier, typename T>
typename my_enable_if<modifier == and_do_abs>::type
increase_by_one(T *data, int size) {
    for (int i = 0; i < size; ++i) {
        data[i] = abs(data[i]) + 1;
    }
}

void test22() {
    int data[5] = {0, -1, -2, 3, -4};
    increase_by_one(data, 5);
    increase_by_one<and_do_abs>(data, 5);
}


// C++11 Variadic templates
// - allow us to have a variable number of template arguments

// the sizeof... keyword gives us the number of template arguments
template <typename... Ts>
int bar1(int x, Ts... params) {
    return sizeof...(Ts);
}

int test23() {
    return bar1(5, "abc", 3.2, 4, true);
}

template <int... Ints>
int bar2() {
    return sizeof...(Ints);
}

int test24() {
    return bar2<1, 2, 3, 1>();
}


namespace std {
template <typename First, typename... Rest>
struct common_type {
    using type = decltype(
        std::declval<First>() +
        std::declval<typename common_type<Rest...>::type>());
};

template <typename First>
struct common_type<First> {
    using type = First;
};
}

template <typename First>
First sum(First first) {
    return first;
}

template <typename First, typename... Rest>
typename std::common_type<First, Rest...>::type sum(First first, Rest... rest) {
    return first + sum(rest...);
}

double test25() {
    return sum(3, 4.5, 2.1, 10, 12);
}

template <typename T, T V>
struct integral_constant {
    static constexpr auto value = V;
};

struct false_type : integral_constant<bool, false> {};

struct true_type : integral_constant<bool, true> {};

template <typename T, typename U>
struct is_same : false_type {};

template <typename T>
struct is_same<T, T> : true_type {};

bool test1() {
    return is_same<int, int>::value; // true
}


bool test2() {
    return is_same<int, double>::value; // false
}
```