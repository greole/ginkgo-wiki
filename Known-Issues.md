## Windows support
Currently, Ginkgo has no official Windows support yet. In general, it is thought that static library support should be easier than shared library on Windows. If you have a Windows computer and really need support, feel free to contribute in [issue 281](https://github.com/ginkgo-project/ginkgo/issues/281). We are actively looking for people with a Windows computer to help push this issue forward.

## Atomic operations for custom value types
When using custom value types with Ginkgo (non default ones), due to atomic operations not supporting custom value types, there may be some execution problems with some kernels. For more details, see [issue 65](https://github.com/ginkgo-project/ginkgo/issues/65).

## Using an installed static library of Ginkgo with CUDA

CMake currently requires to know that it needs to link against CUDA as soon as one of your dependencies require it (there will be an error about `CMAKE_CUDA_DEVICE_LINK_EXECUTABLE` not being set). At the time of writing, it is still an [open CMake issue](https://gitlab.kitware.com/cmake/cmake/issues/18614).
To prevent the error, you can use `enable_language(CUDA)` in your `CMakeLists.txt` before using parts of Ginkgo.

Alternatively, it is also possible to remove the CUDA link dependency from the installed file `${CMAKE_INSTALL_PREFIX}/lib/cmake/Ginkgo/GinkgoTargets.cmake`. From this:
```cmake
IMPORTED_LINK_INTERFACE_LANGUAGES_DEBUG "CUDA;CXX"
```
(It will look slightly different when Ginkgo was compiled in Release mode) to this:
```cmake
IMPORTED_LINK_INTERFACE_LANGUAGES_DEBUG "CXX"
```
We might automate this altering in the future.

This error only appears when having Ginkgo installed as a static library with CUDA, for shared libraries or when not having CUDA, the error should not occur.


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