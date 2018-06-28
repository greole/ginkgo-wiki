__NOTE__: This is a temporary document we can use to write the contributing guidelines, and once it's done we can include it directly into the repository.


Code style
==========

Automatic code formatting
-------------------------

Ginkgo uses [ClangFormat](https://clang.llvm.org/docs/ClangFormat.html) (executable is usually named `clang-format`) and a custom `.clang-format` configuration file (mostly based on ClangFormat's _Google_ style) to automatically format your code. __Make sure you have ClangFormat set up and running properly__ (basically you should be able to run `make format` from Ginkgo's build directory) before committing anything that will end up in a pull request against `ginkgo-project/ginkgo` repository. In addition, you should __never__ modify the `.clang-format` configuration file shipped with Ginkgo. E.g. if ClangFormat has trouble reading this file  on your system, you should install a newer version of ClangFormat, and avoid commenting out parts of the configuration file at all costs.

ClangFormat is the primary tool that helps us achieve a uniform look of Ginkgo's codebase, while reducing the learning curve of potential contributors. However, ClangFormat configuration is not expressive enough to incorporate the entire coding style, so there are several additional rules that all contributed code should follow.

_Note_: To learn more about how ClangFormat will format your code, see existing files in Ginkgo, `.clang-format` configuration file shipped with Ginkgo, and ClangFormat's documentation.

Naming scheme
-------------

### Filenames 

Filenames use `snake_case` and use the following extensions:
*   C++ source files: `.cpp`
*   C++ header files: `.hpp`
*   CUDA source files: `.cu`
*   CUDA header files: `.cuh`
*   CMake utility files: `.cmake`
*   Shell scripts: `.sh`

_Note:_ A C++ source/header file is considered a `CUDA` file if it contains CUDA code that is not guarded with `#if` guards that disable this code in non-CUDA compilers. I.e. if a file can be compiled by a general C++ compiler, it's not considered a CUDA file.

__TODO__: Finish this section.

### Macros

C++ macros (both object-like and function-like macros) use `CAPITAL_CASE`. If they are defined in a header file, they have to start with `GKO_` to avoid name clashes (even if they are `#undef`-ed in the same file!).

### Variables

Variables use `snake_case`.

### Constants

Constants use `snake_case`.

### Functions

Functions use `snake_case`.

### Structures and classes

Structures and classes which do not experience polymorphic behaviour (i.e. do not contain virtual methods, nor members which experience polymorphic behaviour) use `snake_case`.

All other structures and classes use `CamelCase`.

### Members

All structure / class members use the same naming scheme as they would if they were not members:
*   methods use the naming scheme for functions
*   data members the naming scheme for variables or constants
*   type members for classes / structures

Additionally, non-public data members end with an underscore (`_`).

### Namespaces

Namespaces use `snake_case`.

### Template parameters

Template parameters use `CamelCase`.

Whitespace
----------

Spaces and tabs are handled by ClangFormat, but blank lines are only partially handled (the current configuration doesn't allow for more than 2 blank lines). Thus, contributors should be aware of the following rules for blank lines:

1.  Top-level statements and statements directly within namespaces are separated with 2 blank lines. The first / last statement of a namespace is separated by two blank lines from the opening / closing brace of the namespace.
    1.  _exception_: if the first __or__ the last statement in the namespace is another namespace, then no blank lines are required  
        _example_:
        ```c++
        namespace foo {


        struct x {
        };


        }  // namespace foo


        namespace bar {
        namespace baz {


        void f();


        }  // namespace baz
        }  // namespace bar
        ```

    2.  _exception_: in header files whose only purpose is to _declare_ a bunch of functions (e.g. the `*_kernel.hpp` files) these declarations can be separated by only 1 blank line (note: standard rules apply for all other statements that might be present in that file)
    3.  _exception_: "related" statement can have 1 blank line between them. "Related" is not a strictly defined adjective in this sense, but is in general one of:

        1.  overload of a same function,
        2.  function / class template and it's specializations,
        3.  macro that modifies the meaning or adds functionality to the previous / following statement.

        However, simply calling function `f` from function `g` does not imply that `f` and `g` are "related".
2.  Statements within structures / classes are separated with 1 blank line. There are no blank lines betweeen the first / last statement in the structure / class.
    1.  _exception_: there is no blank line between an access modifier (`private`, `protected`, `public`) and the following statement.  
       _example_:
        ```c++
        class foo {
        public:
            int get_x() const noexcept { return x_; }

            int &get_x() noexcept { return x_; }

        private:
            int x_;
        };
        ```

3.  Function bodies cannot have multiple consecutive blank lines, and a single blank line can only appear between two logical sections of the function.
4.  Unit tests should follow the [AAA](http://wiki.c2.com/?ArrangeActAssert) pattern, and a single blank line must appear between consecutive "A" sections. No other blank lines are allowed in unit tests.
5.  Enumeration definitions should have no blank lines between consecutive enumerators.

`#include` statement grouping
-----------------------------

In general, all include statements should be present on the top of the file, ordered in the following groups, with two blank lines between each group:

1. Related header file (e.g. `core/foo/bar.hpp` included in `core/foo/bar.cpp`, or in the unit test`core/test/foo/bar.cpp`)
2. Standard library headers (e.g. `vector`)
3. Third-party library headers (e.g. `omp.h`)
4. Other Ginkgo headers

_Example_: A file `core/base/my_file.cpp` might have an include list like this:

```c++
#include "core/base/my_file.hpp"

#include <omp.h>


#include <algorithm>
#include <vector>
#include <tuple>


#include "third_party/blas/cblas.hpp"
#include "third_party/lapack/lapack.hpp"


#include "core/base/executor.hpp"
#include "core/base/lin_op.hpp"
#include "core/base/types.hpp"
```

_Note_: ClangFormat will take care of sorting the includes alphabetically in each group.

Other Code Formatting not handled by ClangFormat
------------------------------------------------

### Control flow constructs
Single line statements should be avoided in all cases. Use of brackets is mandatory for all control flow constructs (e.g. `if`, `for`, `while`, ...).

### Variable declarations

C++ supports declaring / defining multiple variables using a single _type-specifier_.
However, this is often very confusing as references and pointers exhibit strange behavior:

```c++
template <typename T> using pointer = T *;

int *        x, y;  // x is a pointer, y is not
pointer<int> x, y;  // both x and y are pointers
```

For this reason, __always__ declare each variable on a separate line, with its own _type-specifier_.

Documentation style
-------------------

Documentation uses standard Doxygen.

###  Developer targeted notes
Make use of `@internal` doxygen tag. This can be used for any comment which is not intended for users, but is useful to better understand a piece of code.

### Whitespaces

#### After named tags such as `@param foo`
The documentation tags which use an additional name should be followed by two spaces in order to better distinguish the text from the doxygen tag. It is also possible to use a line break instead.


Project structure
=================

Ginkgo is divided into a `core` module with common functionalities independent of the architecture, and several kernel modules (`reference`, `cpu`, `gpu`) wich contain low-level computational routines for each supported architecture.

Extended header files
---------------------

Some header files from the core module have to be extended to include special functionality for specific architectures. An example of this is `core/base/math.hpp`, which has a GPU counterpart in `gpu/base/math.hpp`.
For such files you should always include the version from the module you are working on, and this file will internally include its `core` counterpart.