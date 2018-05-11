Use of pointers
---------------

The following two requirements guided the choice of using pointers for manipulating Ginkgo's objects:

*   Need for polymorphic classes, which reduces the amount of code that needs to be compiled compared to generics.
*   Need for shared ownership, which reduces the memory requirements of unnecessary object copies. For example, a solver object shares ownership of the system matrix with the top-level application.

### C++ type qualifiers and related problems

Every type in C++ can be described in more detail with respect to the following 4 independent categories:
*   _mutability_ - is the current thread allowed to modify the object?
*   _volatility_ - can the object be changed without direct influence from the current thread?
*   _nullability_ - is the object optional?
*   _ownership_ - what guarantees does the reference holder have about the object's lifetime, and who is responsible for disposing of the object when it is no longer needed?

The first two are binary categories, and are well separated from the last two. Every object can be qualified with the `const` and `volatile` keyword, which are completely independent from each other, and from the other two categories. Thus, the rest of this section will just insert a `cv-qualifier` keyword where `const` and/or `volatile` may appear, and will not explore these two dimension in detail.

Conceptually, _nullability_ and _ownership_ are entirely independent categories, but this is not completely true in C++, as some of the combinations are emulated, or non-existent. Nullability is a binary attribute (_null_ or _non-null_), while owership is a ternary one, and can be one of the following:

1.  _Temporary ownership_: the object is guaranteed to exist until the end of the current scope; the holder of the reference is not responsible for disposing the object.
2.  _Unique ownership_: the object is guaranteed to exist until the holder keeps a reference; the holder of the reference is responsible for disposing the object; it is guaranteed that there is only one holder with non-temporary ownership.
3.  _Shared ownership_: the object is guaranteed to exist until the holder keeps a reference; the holder of the reference is responsible for disposing the object if it is the last non-temporary owner; there might be multiple owners with non-temporary ownership.

Note that the second kind of ownership can be viewed as a special case of the third one, so it is technically not required to distinguish between them, however due to efficiency reasons and caveats of shared ownership (i.e. one owner modifying the object without the other knowing about it), they are considered as 3 separate cases.

The following table demonstrates the problems with nullability vs ownership in C++, by showing how each combination of ownership and nullability is achieved for type `T`.


|                         | non-nullable          | nullable                          |
|:-----------------------:|:---------------------:|:---------------------------------:|
| __temporary ownership__ | `T cv-qualifier &[&]` | `T cv-qualifier *`                |
| __unique ownership__    | `T cv-qualifier`      | `std::unique_ptr<T cv-qualifier>` |
| __shared ownership__    | _not supported_       | `std::shared_ptr<T cv-qualifier>` |

Thus, one of the important categories for Ginkgo: shared ownership of non-nullable objects, is not supported.

### Unique ownership of polymorphic non-nullable objects

There is another problem with non-nullable object ownership when using polymorphic types, and that is [object slicing](https://en.wikipedia.org/wiki/Object_slicing). Basically, when transferring ownership from one non-nullable object to another, the polymorphic behavior of the object is lost.

### Conclusion

Ginkgo almost exclusively needs non-nullable objects, with all three ownership schemes. However, since only temporary ownership is adequately supported for non-nullable objects, the decision is to always use nullable ones. Note that methods of non-nullable and nullable object are called differently (`.` vs `->`), and special referencing (`&`) and dereferencing (`*`) operators are needed to cast between them. For this reason using non-nullable temporary ownership in combination with nullable unique and shared ownership leads to clumsy and non-uniform code. Thus, even though it is supported, nullable temporary ownership is preferred over non-nullable temporary ownership.

See issue [#37](https://github.com/ginkgo-project/ginkgo/issues/37) for proposals and discussions about supporting nullable and non-nullable objects uniformly, with all three ownership modes.