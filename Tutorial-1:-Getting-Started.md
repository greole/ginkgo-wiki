Previous: [Building a 2D Poisson Solver](./Tutorial:-Building-a-2D-Poisson-Solver); Next: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices)

Objectives
----------

In this tutorial we will gude you though the installation process of Ginkgo. We will explain what software you need to install Ginkgo, how to build Ginkgo, run its unit tests, install it in a system folder, and finally run the examples shipped with it.

Prerequisites
-------------

In order to install Ginkgo, you need to meet the following requirements:

### Linux and Mac OS 

For the Ginkgo core library you need to have [_cmake 3.9_](https://cmake.org/install/) (or newer) installed as well as a C++11 compliant compiler, either [_gcc 5.3+, 6.3+, 7.3+, 8.1+_](https://gcc.gnu.org/install/index.html), [_clang 3.9+_](https://clang.llvm.org/get_started.html) or [_Apple LLVM 8.0+_](https://developer.apple.com/library/archive/documentation/CompilerTools/Conceptual/LLVMCompilerOverview/index.html)

If you want to use the Ginkgo CUDA module, you need to have _CUDA 9.0_ or newer installed. For information on how to install CUDA and on possible host compiler restrictions imposed by your version of CUDA, please see the [CUDA installation guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html) or the [CUDA installation guide for Mac Os X](https://docs.nvidia.com/cuda/cuda-installation-guide-mac-os-x/index.html).

### Windows

Windows is currently not supported, but we are working on porting the library there. 

Building ginkgo
---------------

As a first step, clone the Ginkgo repository on your local machine.

```sh
git clone https://github.com/ginkgo-project/ginkgo.git
```

Next, enter the cloned repository, create a build directory and enter it.

```sh
cd ginkgo
mkdir build
cd build
```

In the next step, you have to decide which parts of Ginkgo you want to use. The standard cmake build procedure is

```sh
cmake -G "Unix Makefiles" [OPTIONS] .. && make
```

where you can replace `[OPTIONS]` with the desired cmake options. 
To build the reference version of all of the kernels in Ginkgo, use `-DGINKGO_BUILD_REFERENCE={ON,OFF}` (default is `ON`). While these kernels are not the fastest, they can be expected to be correct and hence can be useful for testing.If you want to use optimized OpenMP kernels on the CPU, use `-DGINKGO_BUILD_CUDA={ON,OFF}` (default is `OFF`) and for optimized CUDA kernels on the GPU, use `-DGINKGO_BUILD_CUDA={ON,OFF}` (default is `OFF`).

If you want to build Ginkgo's unit tests use `-DGINKGO_BUILD_TESTS={ON,OFF}` (default is `ON`). This will download googletest if you don't have it installed yet.

To build further examples on the functionality of Ginkgo, use `-DGINKGO_BUILD_EXAMPLES={ON,OFF}` (default is `ON`). 

The option `-DGINKGO_DEVEL_TOOLS={ON,OFF}` (default is `ON`) sets up the build system for development, so you should use it in case you plan on contributing to Ginkgo.

To build Ginkgo in Debug mode with these options, the cmake build procedure would be

```sh
cmake -G "Unix Makefiles" -BDebug -DCMAKE_BUILD_TYPE=Debug -DGINKGO_DEVEL_TOOLS=ON \
         -DGINKGO_BUILD_TESTS=ON -DGINKGO_BUILD_EXAMPLES=ON -DGINKGO_BUILD_REFERENCE=ON \
	    -DGINKGO_BUILD_OMP=ON -DGINKGO_BUILD_CUDA=ON ..
cmake --build Debug -j [JOBS]
```

where you replace `[JOBS]` with the number of jobs you want to use.

In case you want to install Ginkgo into a specific location, you can use the option `-DCMAKE_INSTALL_PREFIX=path`. The default location is usually `/usr/local/`.

For a detailed description of all the available build options please see the [Installation page](https://github.com/ginkgo-project/ginkgo/blob/develop/INSTALL.md).



Running unit tests [optional]
-----------------------------

Assuming you have built Ginkgo with `-DGINKGO_BUILD_TESTS=ON`, you now can run Ginkgo's unit tests by calling

```sh
cd Debug/
make test
```

You will see tests for the Reference, OpenMP and CUDA implementations of all the routines in Ginkgo, depending on which build options you have chosen.

Installation
------------

After all tests passed you are ready to install Ginkgo into the specified path. This can simply be done with 

```sh
make install
```

If you didn't set an installation prefix with `-DCMAKE_INSTALL_PREFIX` or if the installation prefix is not writable for your user, you might have to call

```sh
sudo make install
```

to successfully install Ginkgo.

Running the examples [optional]
-------------------------------

If you have compiled Ginkgo with `-DGINKGO_BUILD_EXAMPLES=ON` you can now run the included examples.
To do so, change directories into `examples`

```sh
cd examples/
```

and choose an example. Change directories into the example directory and run the executable.

Previous: [Building a 2D Poisson Solver](./Tutorial:-Building-a-2D-Poisson-Solver); Next: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices)
