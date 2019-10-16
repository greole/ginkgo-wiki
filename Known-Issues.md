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