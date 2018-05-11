The purpose of this page is to shed some light on RTTI, as this is one of the main performance concerns of C++. The idea is to give a high-level description on how RTTI is implemented so you can asses the performance impacts of using RTTI in your implementations.

# About RTTI

A general description of what is RTTI can be found on [wikipedia](https://en.wikipedia.org/wiki/Run-time_type_information), but in short it is C++'s way of providing limited type introspection - i.e. getting information about a (unknown) type at runtime. This is mostly used in combination with polymorphism and pointers to derived classes in case the program needs to branch off depending on the concrete name of the type.

C++ standard just defines high-level interfaces for RTTI:
*   The `typeid` operator which returns an object of type `std::type_info` which describes the type of an object. These objects can be compared to determine if two objects are of the same type.
*   The `dynamic_cast` operator, which is able to perform base-to-derived and cross (i.e. sideways) casts in a polymorphic hierarchy, with runtime checking of the validity of the cast.

The actual implementation and the data layout of the structures supporting RTTI is not prescribed by the standard. However, most major compilers (gcc, clang, icc, pgi) implement the [Itanium C++ ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi.html), which, among other things, prescribes the exact layout of the data structures and function interfaces supporting RTTI. While compilers can provide their own implementation of these functions, just knowing the data layout can help us in determining what kind of algorithms the compilers can implement.

# Itanium ABI RTTI data structures

__NOTE:__ the authors of this page are not experts in compilers, or the ABI. Thus, some of the interpretations may not be completely accurate. For authoritative reference refer to the [official Itanium C++ ABI webpage](https://itanium-cxx-abi.github.io/cxx-abi/). 
 
To enable RTTI, the compiler generates an instance of (a subclass of) `std::type_info` for each class, and a pointer to that instance is added to the virtual table of the class. This instance is returned by the `typeid` operator.

The `std::type_info` object for a class `C` contains pointers to `std::type_info` objects of all direct bases of class `C`. Thus, a class hierarchy is represented as a _parent pointer DAG_.

The [RTTI section](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#rtti) of the ABI gives more details about the data structure.

`dynamic_cast` is implemented via a call to [`abi::__dynamic_cast`](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#dynamic_cast-algorithm), which receives the pointer to the object being casted, as well as `std::type_info` objects of the source static type of the object being casted, and of the destination type of the cast. A hint about the relationship of the source and destination types can also be provided by the compiler, which can simplify the search in the hierarchy. For example, in simple cases (without multiple or virtual inheritance), the compiler can provide the exact offset of the virtual table of the source object in the virtual table of the destination object, which allows the function to avoid traversing the class hierarchy DAG, and makes the check extremely simple (and O(1)).