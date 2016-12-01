Duplication and Extension of Existing Classes
=============================================

Introduction
------------

This document describes a possible approach to duplicate existing functionality
while wrapping it in a new type, without the burden of inheritance and to allow
function overloads on syntactically identical but semantically different types.

The approach taken should be simple to implement and be applicable to existing
code.

Reasons
-------

- Scientific libraries where a vector of double has different behaviors
  depending on context have currently no simple way to indicate the semantic
  differences. Since a `typedef` does not allow multiple overloads on new
  typedef types - since they are still the "old" type - they have to resort to
  imperfect techniques, such as copying, wrapping or inheriting the needed type.
- Easier maintainability of code which is known to be the same, rather than
  being copy-pasted.
- Searching for "`std::vector` inheritance" yields many results of users trying
  to maintain the original interface and functionality but add one or two methods.
- Very often inheritance is misused in order to provide a copy-paste
  alternative. This can result in very deep hierarchies of types which should
  really not have anything to do with each other.

The functionality should have the following requirements:

- Can be applied to existing code.
- Should limit dependencies between new and old type as much as possible.
- Should allow for partial extensions of the old code.

Alternatives
------------

### Typedef / Using Directive ###

Using a type alias creates an alternative name for a single type. However, this
leaves no space to implement overloads that are context-specific. Nor a type can
be extended in a simple way while keeping the old interface intact.

### Inheritance ###

Inheritance requires redefinition of all constructors, and creates a stricter
dependency between two classes than what is proposed here. Classes may be
converted to a common ancestor even though that is undesired or even dangerous
in case of implicit conversions.

Inheritance may also be unwanted in order to avoid risks linked to polymorphism
and freeing data structures where the base class does not have a virtual
destructor.

### Encapsulation with Manual Exposure of Needed Methods ###

This method obviously requires a great deal of code to be rewritten in order to
wrap every single method that the old class was exposing.

In addition one needs to have intimate knowledge of the original interface in
order to be able to duplicate it correctly. Template methods, rvalue references,
possibly undocumented methods which are required in order to allow the class to
behave in the same way as before. This heightens the bar significantly for many
users, since they may not know correctly how to duplicate an interface and how
to forward parameters to the old interface correctly.

The new code also must be maintained in case the old interface changes.

### Copying the Base Class ###

This can be useful, but requires all code to be duplicated, and thus
significantly increases the burden of maintaining the code. All bugs discovered
in one class must be fixed in the other class too. All new features applied to
one class must be applied to the other too.

### Macro-expansion ###

Macro expansions can be used in order to encode the interface and implementation
of a given class just one time, and used multiple times to produce separate
classes.

This approach is unfortunately not applicable to existing code, and is very hard
to extend if one wants to copy a class but add additional functionality to it.

### Templates ###

Templates produce for each instantiation a separate type. They are unfortunately
not applicable to previously existing code. For new code, they would require the
creation of "fake" template parameters that would need to vary in order to
produce separate types. For each new instantiation the user would need to check
that in his/her whole codebase no other usages of the template with the same
fake parameters were made, or the class won't really be unique.

In addition, class extension through templates is not possible: variations would
need to be made through specialization, which itself requires copying existing
code.

Syntax
------

### Simple Case ###


Syntax could look something like this:

```cpp
class Base {
    public:
        Base() : x(0) {}
        void foo() { std::cout << "foo " << x << "\n"; }
    private:
        int x;
};

struct Copy : using Base {};

/* Equivalent to

struct Copy {
    public:
        Copy() : x(0) {}
        void foo() { std::cout << "foo " << x << "\n"; }
    private:
        int x;
};

*/
```

One cannot copy a class and inherit at the same time. If such a feature is
needed one would need to create a class ex novo, as it would be done normally.

All method implementations would be the same. The copied class would inherit
from the same classes its base class inherits from. All constructors would work
in the same way.

### Overloads ###

Duplicating an existing class should allow for new overloads on the new type,
and no ambiguity between the copied class, the old class and other copied
classes.

```cpp
class Position : using std::pair<double, double> {};
class Distance : using std::pair<double, double> {};

Position operator+(const Position & p, const Distance & d) {
    return Position(p.first + d.first, p.second + d.second);
}

Distance operator+(const Distance & lhs, const Distance & rhs) {
    return Distance(lhs.first + rhs.first, lhs.second + rhs.second);
}

// ...

Position p(1, 1);
Distance d(1, 1);

p + d; // OK
d + d; // OK
p + p; // Error
```

### Copying Template Classes ###

Since the construct is similar to inheritance, the syntax for creating aliases
of templated classes could be the same:

```cpp
template <typename T>
struct A {};

template <typename T>
struct B : using A<T> {};

B<int> b;
```

The copied class must have the same number or less of template parameters than
the base class. Partial or full specializations of the base class can be allowed:

```cpp
template <typename T, typename U>
struct A {};

template <typename T>
struct B : using A<T, double> {};

B<int> b;
```

When the base class has partial specializations, only those who apply are copied
to the copied class.

```cpp
template <typename T, typename U>
struct A { T t; U u; };

template <typename U>
struct A<double, U> { double y; U u; };

template <typename T>
struct A<T, int> { T t; char z; };

template <typename T>
struct B : using A<T, double> {};

/* Equivalent to

template <typename T>
struct B { T t; double u; }

template <>
struct B<double> { double y; double u; };

*/
```

The copied class can add additional specialization. Or specializations for a
given class can copy another.

```cpp
template <typename T>
struct A { int x; };

struct B { char c; };

template <typename T>
struct C : using A<T> {};

template <>
struct C<double> : using B {};

template <>
struct A<int> : using C<double> {};

/* Equivalent to

template<>
struct A<int> { char c; };

template <typename T>
struct C { int x; }

template <>
struct C<double> { char c; };

*/

```

### Copying Multiple Dependent Classes ###

Copying multiple classes using the simple syntax we have described can be
impossible if those classes depend on one another. This is because each copy
would depend on the originals, rather than on the copied classes. A possible way
to specify such dependencies could be:

```cpp
struct A;

struct B {
    A * a;
};

struct A {
    B b;
};

struct C;

struct D : using B {
    using class C = A;
};

struct C : using A {
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
declared as a copy of the right hand side.

In case of a template base class using a template second class, one could
specify different copies for certain specializations;

```cpp
template <typename T>
struct A {};

template <typename T>
struct B {
    A<T> a;
};

template <typename T>
struct C : using A<T> {};

```

### Adding New Functionality ###

Ideally one could specify additional methods, separate from that of Base, to add
upon the existing functionality.

```cpp
struct Base {
    void foo() { std::cout << "foo\n"; }
};

struct Copy : using Base {
    void bar() { std::cout << "bar\n"; }
};

/* Equivalent to

struct Copy {
    void foo() { std::cout << "foo\n"; }
    void bar() { std::cout << "bar\n"; }
};

*/
```

Only new methods need to be implemented for that class.

### Substituting Existing Functionality (Optional) ###

Ideally one may want to use most of an implementation for another class, but
vary a certain number of methods. In this case, if `Copy` contains a member
function that already exists in `Base`, then that implementation is substituted
in `Copy`. This may or may not be allowed for attributes.

```cpp
struct Base {
    void foo() { std::cout << "foo\n"; }
    void bar() { std::cout << "bar\n"; }
};

struct Copy : using Base {
    void foo() { std::cout << "baz\n"; }
};

/* Equivalent to

struct Copy {
    void foo() { std::cout << "baz\n"; }
    void bar() { std::cout << "bar\n"; }
};

*/
```

A side effect of this is that it could allow for some type of "interface", where
some base class could be defined as:

```cpp
struct Base {
    Base() = delete;
    void foo();
    void bar();
};

struct Copy1 : using Base {
    Copy1() = default;
    void baz();
};

struct Copy2 : using Base {
    Copy2() = default;
    void abc();
};
```

This feature could however present problems when the members changed also alter
behavior and/or variable types of non-modified member and non-member functions,
since the new behavior could be either erroneous or ambiguous.

### STL Traits (Optional) ###

Traits could be included in the standard library in order to determine whether a
class is a copy of another, or if it has been derived from a copy
(copies/inheritances could be nested arbitrarily).

```cpp
struct Base {};

struct Copy : using Base {};

static_assert(std::is_copy<Copy, Base>::value);

struct ChildCopy : public Copy {};

struct CopyChildCopy : using ChildCopy {};

static_assert(std::is_copy_base_of<Base, CopyChildCopy>::value);
```

Compatibility
-------------

As the syntax is new, no old code would be affected.

Possible Extensions
-------------------

It could possibly be useful if a similar approach could be used in order to
extend in the same way primitive types, so that specific overloads may be
implemented for different types, while at the same time preventing the
compilation of operations that are currently allowed but should not be (e.g.
summing meters to feet).
