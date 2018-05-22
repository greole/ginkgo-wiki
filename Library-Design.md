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

1.  _Temporary ownership_: the object is guaranteed to exist until the end of the current scope; the referece holder is not responsible for disposing of the object.
2.  _Unique ownership_: the object is guaranteed to exist as long as the reference holder keeps the reference; the reference holder is responsible for disposing of the object; it is guaranteed that there is only one reference holder with non-temporary ownership.
3.  _Shared ownership_: the object is guaranteed to exist as long as the reference holder keeps the reference; the reference holder is responsible for disposing the object if it is the last non-temporary reference holder; there can be multiple reference holders with non-temporary ownership.

Note that the second kind of ownership can be viewed as a special case of the third one, so it is technically not required to distinguish between them. However, due to efficiency reasons and caveats of shared ownership (i.e. one reference holder modifying the object without the other one knowing about it), they are considered as 3 separate cases.

The following table demonstrates the problems with nullability vs ownership in C++, by showing how each combination of ownership and nullability is achieved for type `T`.


|                         | non-nullable                         | nullable                               |
|:-----------------------:|:------------------------------------:|:--------------------------------------:|
| __temporary ownership__ | `T cv-qualifier &`                   | `T cv-qualifier *` <sup>&Dagger;</sup> |
| __unique ownership__    | `T cv-qualifier` <sup>&dagger;</sup> | `std::unique_ptr<T cv-qualifier>`      |
| __shared ownership__    | _not supported_                      | `std::shared_ptr<T cv-qualifier>`      |

### <sup>&dagger;</sup> Unique ownership of polymorphic non-nullable objects

There is a problem with non-nullable object ownership when using polymorphic types, and that is [object slicing](https://en.wikipedia.org/wiki/Object_slicing). Basically, when transferring ownership from one non-nullable object to another, the polymorphic behavior of the object is lost.

### <sup>&Dagger;</sup> Inconsistent interface of nullable object references with temporary ownership

Nullable references with unique and shared ownership provide a `.get()` method which returns a reference to an object with temporary ownership. However, a reference with temporary ownership does not provide such a method, even though it is perfectly valid (and useful in contexts when the ownership of the original reference is not known). In addition, unique and shared ownership reference can be extended via inheritance, while the temporary ownership reference cannot.

### Conclusion

Ginkgo almost exclusively needs non-nullable objects, with all three ownership schemes. However, since only temporary ownership is adequately supported for non-nullable objects, the decision is to always use nullable ones. Note that methods of non-nullable and nullable object are called differently (`.` vs `->`), and special referencing (`&`) and dereferencing (`*`) operators are needed to cast between them. For this reason using non-nullable temporary ownership in combination with nullable unique and shared ownership leads to clumsy and non-uniform code. Thus, even though it is supported, nullable temporary ownership is preferred over non-nullable temporary ownership.

See issue [#37](https://github.com/ginkgo-project/ginkgo/issues/37) for proposals and discussions about supporting nullable and non-nullable objects uniformly, with all three ownership modes.

Ginkgo's core abstract classes
==============================

__Note:__ The following text has been copied from PR [#46](https://github.com/ginkgo-project/ginkgo/pull/46) and most likely needs some extra editing to integrate into this page properly.

`PolymorphicObject`
-------------------

A `PolymorphicObject` is designed as an abstract base class for all polymorphic objects. It is executor-aware (i.e. all polymorphic object "belong" to a certain executor), and exposes virtual method for cloning and copying polymorphic objects.

It is easy to notice that for a specific (concrete) implementation of a polymorphic object, all these methods are trivially implemented using a constructor which takes an executor, and the assignment operator. For this reason, to simplify the implementation of polymorphic objects and promote code reuse, this PR provides the `EnablePolymorphicObject` and `EnableAbstractPolymorphicObject` [mixins](https://en.wikipedia.org/wiki/Mixin) that provide default implementations of these methods.

The `PolymorphicObject` class is implemented in a way that reduces the amount of conversions needed when using it. As an example, consider the `clone()` method which will create an exact copy of a polymorphic object. To make it a part of the PolymorphicObject interface it needs to have the following signature:

```c++
virtual std::unique_ptr<PolymorphicObject> clone() const = 0;
```

However, when implementing the method for a concrete polymorphic object, say a CSR matrix, we do know more about the return type - it should be a unique pointer to a CSR matrix, so we would _like_ to write something like this:

```c++
std::unique_ptr<Csr> clone() const override { /* implementation of clone */ }
```

Unfortunately, changing the return type of a virtual method is only allowed if the types are [covariant](http://en.cppreference.com/w/cpp/language/virtual#Covariant_return_types), which is not the case here.
However, we are still able to support this by implementing clone() using a helper method:

```c++
class PolymorphicObject {
public:
    // the clone method is no longer virtual
    std::unique_ptr<PolymorphicObject> clone() const { return this->clone_impl(); }
protected:
    // there is a new helper method which is virtual
    virtual std::unique_ptr<PolymorphicObject> clone_impl() const = 0;
};
```

Then, in the concrete polymorphic object, we __hide__ the `clone()` method, and __override__ the `clone_impl()` method:

```c++
class Csr {
public:
    // since this method is no longer virtual, we can change its return type
    // however, we need to cast the result of clone_impl()
    std::unique_ptr<Csr> clone() const { 
        return std::unique_ptr<Csr>(this->clone_impl().release());
    }
protected:
    // we implement clone for Csr, keeping it's signature
    std::unique_ptr<PolymorphicObject> clone_impl() const override { /* implementation of clone */ }
};
```

Both of these things are done automatically for clone and other methods when using the `EnablePolymorphicObject` mixin. For abstract classes that inherit from PolymorphicObject, (e.g. `LinOp`) we still want to hide the original `clone()` method, so it returns an instance of `LinOp` when called on a `LinOp`. For this reason, there is a `EnableAbstractPolymorphicObject` mixin that will only hide the public methods, but leave the implementation methods unimplemented.

There are three conventions here that are used throughout the code:

_Convention 1:_ All mixin classes start with `Enable` or `enable_`.  
_Convention 2:_ All virtual implementation methods end with `_impl`.  
_Convention 3:_ All mixins need a reference to the class that implements them (know also as the CRTP pattern in C++). Such template parmeters have been marked with `[CRTP parameter]` in the documentation.

Notice that _Convention 1_ seems to break the rule that class names should be nouns. This is intentional, since mixin "classes" should never be used as classes themselves, but just "included" (via inheritance) into other classes to provide specific functionality (unfortunately, C++ does't have built-in support for mixins, so misusing inheritence for this purpose is the only way of implementing them). The word "enable" was chosen to be compatible with the C++ standard library (see [`std::enable_shared_from_this`](http://en.cppreference.com/w/cpp/memory/enable_shared_from_this) for an example of a mixin in the standard library).

`LinOp`
-------

Thanks to `PolymorphicObject`, the `LinOp` interface has been simplified. It now inherits all of its memory management methods from it, and just adds the `apply()` methods. The `apply()` methods have also been enhanced by adding a [fluent interface](https://en.wikipedia.org/wiki/Fluent_interface), automatically verifying the sizes of input parameters (so the implementers don't have to do it anymore), and automatically copying the parameters to the correct executor if they were not already there.

In order to enable this, the same "trick" of having a public non-virtual `apply()` method (which is hidden in subclases), and a protected virtual `apply_impl()` method was used here.
The public method validates the sizes, and copies the parameters, and then passes the control to the `apply_impl()` method which the user has to override with their own implementation.

The `BasicLinOp` mixin has been updated accordingly, and renamed to `EnableLinOp`, to be consistent with _Convention 1_.

Finally, the `num_stored_nonzeros` property has been removed (closes #47), and the `num_rows` and `num_cols` properties have been merged into a single property `size`, of the newly created type `dim`, which has `num_rows` and `num_cols` properties, and provides some useful methods for size manipulation (more can be added if needed later). I find this easier to use, as most of the time we manipulate both dimensions together (e.g. to get transposed dimensions, it is now possible to write `transpose(size)`). All existing linear operators have been updated to use this new syntax accordingly.

`AbstractFactory` and `LinOpFactory`
------------------------------------

The `LinOpFactory` interface (which had a generate method capable of creating `LinOp`s from other `LinOp`s) has been generalized into the `AbstractFactory` interface template which creates `AbstractProductType`s from `ComponentsType`s. There is still a symbol named `LinOpFactory`, but this has now become only an alias for a specific kind of `AbstractFactory`.
The idea is that the `AbstractFactory` interface can be reused to create stopping criteria and logger factories if needed.

Another improvement to `AbstractFactory` is that it has become a first-class citizen together with `LinOp`, as it now supports standard management operations such as copying and cloning (it inherits from `PolymorphicObject`.

As most factories we had were mostly boilerplate code (set parameters, pass them to the constructor of the object being built), we now had an `EnableDefaultFactory` mixin that is able to generate a complete factory implementation, give the adequate description. For `LinOpFactory`, there is also a `GKO_ENABLE_LIN_OP_FACTORY` macro that provides a simpler interface to `EnableDefaultFactory`.
The default factory uses a fluent interface to fill the parameters of the factory, and accepts default parameters, so the users do not have to specify all of them.
All classes that use factories (except for BlockJacobi - it needs significant refactoring anyway, so I skipped it for now) have been updated to use this. See one of the solvers and the unit tests for examples of enabling a default factory for a linear operator, and using such a factory. 

Other utilities
---------------

There are a couple of minor tweaks that were added as support for the major changes, but can be useful on their own:

1.  The most significant one is the `temporary_clone` "smart pointer". This utility will make sure that the object is on the specified executor, by copying it only if it's not already there. For non-constant objects, it will also copy the data back to the original location before deallocating it. (This is useful in the new implementation of `LinOp::apply()`, and anywhere where ensuring the correct executor of an object is required.)
2. As already mentioned, the `dim` class connects both the `num_rows` and `num_cols` properties into a single property, and adds some utilities for manipulating them. The utility `size` class used in some assertions has been completely replaced by `dim`.

Changes to the code
-------------------

As you probably noticed, the is a significant number of lines changed by this PR. However, most of them are trivial search/replace changes resulting from a changed interface. To summarize them in one place these are:

1.   `get_num_rows()` and `get_num_cols()` calls have been replaced by `get_size().num_rows` and `get_size().num_cols`.
2.  constructors that take `num_rows` and `num_cols` arguments have been modified to instead take a `size` argument. This also made it possible to remove some of the constructors by providing default parameters in some of the other ones. Calls to those constructors have also been updated.
3.  The `get_num_stored_elements()` method has been added to concrete operators where it makes sense, and the assertions using this method have been removed for classes where the method doesn't makes sense.
4.  Solver factories have been removed and replaced with automatically generated ones. Since these factories have a slightly different (fluent) interface, the calls to these factories have been updated.
5.  Public `apply()` overrides in concrete operators have been replaced with protected `apply_impl()` overrides.

References
----------

The following is a list of references which explain some of the concepts used in this PR in more detail:

1.  [Method overriding vs method hiding](https://stackoverflow.com/a/19737267/9385966)
2.  [Mixins](https://en.wikipedia.org/wiki/Mixin)
3.  [CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)
4.  [Fluent interface](https://en.wikipedia.org/wiki/Fluent_interface)
5.  [Abstract factory pattern](https://en.wikipedia.org/wiki/Abstract_factory_pattern)