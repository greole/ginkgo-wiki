Previous: [Building a 2D Poisson Solver](./Tutorial:-Building-a-2D-Poisson-Solver); Next: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices)

Objectives
----------

In this tutorial we will guide you though the installation process of Ginkgo. We will explain what software you need to install Ginkgo, how to build Ginkgo, run its unit tests, install it in a system folder, and finally run the examples shipped with it.

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

In the next step, you have to configure the Ginkgo installation process.


If your system supports it, the easiest way to configure your installation is to use the [CMake curses interface](https://cmake.org/cmake/help/v3.10/manual/ccmake.1.html)

```sh
ccmake ..
```
and then configure the options as desired. After having generated the CMake configuration, you initiate the build process via

```sh
make -j4
```

where `4` defines the number of cores you want to use in the parallel compilation.

The alternative way for configuring the installation process is to follow the standard cmake build procedure as

```sh
cmake -G "Unix Makefiles" [OPTIONS] .. && make
```

where you can replace `[OPTIONS]` with the desired cmake options. 
To build the reference version of all of the kernels in Ginkgo, use `-DGINKGO_BUILD_REFERENCE={ON,OFF}` (default is `ON`). 
The reference kernels are not optimized for performance, they serve as as correctnesss-check for the algorithms, and are guaranteed to compute the correct result. 
If your platform supports OpenMP and you want to use optimized OpenMP kernels, use `-DGINKGO_BUILD_OMP={ON,OFF}` (default is `OFF`).
Similarly, if your platform supports CUDA (i.e. has a CUDA-capable accelerator) 
and you want to use CUDA kernels, use `-DGINKGO_BUILD_CUDA={ON,OFF}` (default is `OFF`).

If you want to build Ginkgo's unit tests use `-DGINKGO_BUILD_TESTS={ON,OFF}` (default is `ON`). 
The unit tests are based on googletest, and if there is no googletest installation available on your system,
the cmake build process with automatically download and install googletest.

To build further examples on the functionality of Ginkgo, use `-DGINKGO_BUILD_EXAMPLES={ON,OFF}` (default is `ON`). 

The option `-DGINKGO_DEVEL_TOOLS={ON,OFF}` (default is `ON`) sets up the build system for development. 
In case you want to implement new algorithms inside Ginkgo, you **need** to set it to `ON`. If you only
plan to use Ginkgo as a library inside a larger application, you can turn it off.

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

Previous: [Building a Poisson Solver](./Tutorial:-Building-a-Poisson-Solver); Next: [Implement: Matrices](./Tutorial-2:-Implement:-Matrices)
