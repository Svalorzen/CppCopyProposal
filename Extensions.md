Extensions
==========

Templated Type-Clone
--------------------

Type-clones can be templatized.

```cpp
struct A { int x; };

template <typename T>
struct TemplatizedClone = T {
    static_assert(std::is_standard_layout<T>::value,
                  "Can't use this with a non-standard-layout class");

    operator T&() { return *reinterpret_cast<T*>(this); }
};

// Could be used either via normal typedefs
using Alias = TemplatizedClone<A>;

// Or via clone, depending on requirements.
struct Clone = TemplatizedClone<A> {};
```

Type-Clone of Template Classes
------------------------------

Template classes can be type-copied.

```cpp
template <typename T>
struct A {};

template <typename T>
struct B = A<T> {};

B<int> b;
```

The type-clone must have equal or less template parameters than its type-base.
Type-clone of partial or full specializations are allowed:

```cpp
template <typename T, typename U>
struct A {};

template <typename T>
struct B = A<T, double> {};

B<int> b;
```

When the type-base has partial specializations, the type-clone includes all those
that are still applicable on its parameters.

```cpp
template <typename T, typename U>
struct A { T t; U u; };

template <typename U>
struct A<double, U> { double y; U u; };

template <typename T>
struct A<T, int> { T t; char z; };

template <typename T>
struct B = A<T, double> {};

/* Equivalent to

template <typename T>
struct B { T t; double u; };

template <>
struct B<double> { double y; double u; };

*/
```

A templated type-clone is allowed to implement specializations independently of
its type-base. However, specializations already present from the type-base
cannot be overridden. Template specializations can type-clone another class.

```cpp
template <typename T>
struct A { int x; };

struct B { char c; };

template <typename T>
struct C = A<T> {};

template <>
struct C<double> = B {};

template <>
struct A<int> = C<double> {};

// Can't be done, since A<int> creates C<int>
// template<>
// struct C<int> {};

/* Equivalent to

template<>
struct A<int> { char c; };

template <typename T>
struct C { int x; };

template <>
struct C<double> { char c; };

template <>
struct C<int> { char c; };

*/
```

Type-Clone of Multiple Dependent Classes
----------------------------------------

Type-cloning multiple classes using the simple syntax we have described can be
impossible if those classes depend on each other. This is because each copy
would depend on the originals, rather than on the newly created type-copies.

A type-clone is allowed to substitute any used class in its type-base for another
class, given that the substitution is a type-clone of the substituted class.

```cpp
struct A;

struct B {
    A * a;
};

struct A {
    B b;
};

struct C;

struct D = B {
    using class C = A;
};

struct C = A {
    using class D = B;
};

/* Equivalent to

struct C;

struct D {
    C * a;
};

struct C {
    D b;
};

*/
```

`using class` has been used in order to disambiguate it from normal `using`
alias directive. `using class` is only valid when the left hand side has been
defined as a type-clone of the right hand side.

Similarly, this could be done to type-clone whole hierarchies of classes.

```cpp
struct A {};
struct B : A {};

struct C = A {};
struct D = B {
    using class C = A;
};
```

STL Traits
----------

Traits should be included in the standard library in order to determine whether
a class is a type-clone of another, or if it has been derived from a type-clone
(copies/inheritances could be nested arbitrarily).

```cpp
struct Base {};

struct Copy = Base {};

static_assert(std::is_copy_of<Copy, Base>::value);

struct ChildCopy : public Copy {};

struct CopyChildCopy = ChildCopy {};

static_assert(std::is_copy_base_of<Base, CopyChildCopy>::value);
```
