![Ginkgo Library](https://raw.githubusercontent.com/ginkgo-project/ginkgo/develop/assets/logo_small.png)
## CUDA Toolkit 10.1 issues for developer
__This problem should only affect developer, as all user exposed functions should already be free of this problem.__

The CUDA compiler in Toolkit 10.1 has a bug which results in a compiler error when using a combination of a `static` function and a pointer to the class of the `static` function.
<details><summary> Here is an example of this problem </summary>

```c++
// main.cu
#include <iostream>
#include <memory>
#include <utility>


template<typename ValueType>
class Test {
public:
    static std::unique_ptr<Test> create_with_config_of(const Test *other) {
        return std::unique_ptr<Test>(new Test(other->get_var()));
        // Fix for this issue: de-referencing the pointer first
        // return std::unique_ptr<Test<ValueType>>(new Test<ValueType>((*other).get_var()));
    }
    static std::unique_ptr<Test> create() {
        return std::unique_ptr<Test>(new Test());
    }
    virtual ValueType get_var() const {
        return var_;
    }
protected:
    Test() : var_{1} {}
    Test(ValueType var) : var_{var} {}
private:
    ValueType var_;
};

int main() {
    using value_type = int;
    auto a = Test<value_type>::create();
    auto b = Test<value_type>::create_with_config_of(a.get());
    std::cout << b->get_var() << std::endl;
}
```
</details>

Compiling this with `nvcc main.cu` will result in the error:
```bash
main.cu:12:51: error: cannot call member function ‘ValueType Test<ValueType>::get_var() const [with ValueType = int]’ without object
```
However, compiling it with, for example, `g++`, it works fine.

This problem is only present when compiling a `*.cu` file because only then the CUDA compiler is involved.

### The fix
The fix to that problem is already included as a comment in the example: De-referencing the variable before calling a function (instead of using the operator `->`) prevents this compiler error all together.
So, instead of using `other->get_size()`, `(*other).get_size()` compiles fine with the CUDA compiler.