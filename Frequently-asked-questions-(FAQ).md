
This page is meant to answer some of the frequently asked questions users might face when starting with Ginkgo.


#### Question: I did a quick build with `cmake ..` and `make` in the build directory and I get errors like

```
[  0%] Built target generate_ginkgo_header
[  0%] Built target ginkgo_reference_device
[  0%] Building CXX object reference/CMakeFiles/ginkgo_reference.dir/base/version.cpp.o
In file included from ${SOURCE}/ginkgo/reference/base/version.cpp:33:0:
${SOURCE}/ginkgo/include/ginkgo/core/base/version.hpp: In constructor ‘gko::version_info::version_info()’:
${SOURCE}/ginkgo/include/ginkgo/core/base/version.hpp:207:42: error: cannot convert ‘gko::version’ to ‘const uint64 {aka const long unsigned int}’ in initialization
           cuda_version{get_cuda_version()}
```

This probably means that you dont have the correct version of the C++ compiler. Please be aware that Ginkgo works with only a proper C++11 compliant compiler. See the [prerequisites in README.md](https://github.com/ginkgo-project/ginkgo/blob/develop/README.md)




#### Question: When installing with `make install` it gives me errors saying that I do not have permissions to write the files to the directory.
The default installation path is `/usr/include` which you probably either do not have permissions to write to (if you are on a shared cluster) and probably not what you want as well. You can instead specify an installation path during the cmake process by adding `-DCMAKE_INSTALL_PREFIX=/your/desired/path` to the cmake command and then a normal `make install` will install the Ginkgo headers and library to this directory. In the case that you actually want to install it to `/usr/include`, then you need to do `sudo make install` and must be a `sudoer`.


#### Question: I am trying a parallel make with `make -j<procs>`. I get an error saying that I have a internal compiler error.
This probably means that you are running out of memory when doing the parallel make. In Ginkgo we try to speed up the make process as much as possible by distributing the compilation units across the files. This means that we can compiler many files at once. If your machine does not have enough memory, it is possible that it crashes with an internal compiler error. If you are lucky, you might even get a note that says that it is out of memory. You can confirm this by running `top` and `htop` while compiling Ginkgo. To resolve this you must use lesser number of processes during the `make` process.