Duplication and Extension of Existing Classes
=============================================

Introduction
------------

This document describes a possible approach to duplicate existing functionality
while wrapping it in a new type, without the burden of inheritance and to allow
function overloads on syntactically identical but semantically different types
(also known as *strong typedef*).

The approach taken should be simple to implement and be applicable to existing
code.

Reasons
-------

- Scientific libraries where a type has different behaviors depending on context
  have currently no simple way to indicate the semantic differences. Since a
  `typedef` does not allow multiple overloads on new typedef types - since they
  are still the "old" type - they have to resort to imperfect techniques, such
  as copying, wrapping or inheriting the needed type. Examples: coordinates in a
  plane (rectangular, polar), vectors of double (probabilities, values).
- Easier maintainability of code which is known to be the same, rather than
  being copy-pasted.
- Avoiding misuse of inheritance in order to provide a copy-paste alternative.
  This can result in very deep hierarchies of types which should really not have
  anything to do with each other.
- Enabling users to use an existing and presumably correct type but partially
  extend it with context-specific methods. Examples: search for "`std::vector`
  inheritance" yields many results of users trying to maintain the original
  interface and functionality but add one or two methods.

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
produce separate types.

In addition, class extension through templates is not possible: variations would
need to be made through specialization, which itself requires copying existing
code.

Previous Work
-------------

Strong typedefs have already been proposed for the C++ language multiple times
([N1706](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2004/n1706.pdf),
[N1891](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2005/n1891.pdf),
[N3515](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3515.pdf),
[N3741](https://isocpp.org/files/papers/n3741.pdf)). These typedefs are named
*opaque typedefs*, and these papers try to explore and define exactly the
behavior that such typedefs should and would have when used to create new
types. In particular, the keywords `public`, `protected` and `private` are used
in order to create a specific relation with the original type and how is the
new type allowed to be cast back to the original type or be used in its place
during overloads.

This document shares many of the the same principles, for example (quoting from
N3741):

> - Consistent with restrictions imposed on analogous relationships such as
>   base classes underlying derived classes and integer types underlying enums,
>   an underlying type should be (1) complete and (2) not cv-qualiﬁed. We also do
>   not require that any enum type, reference type, array type, function type, or
>   pointer-to-member type be allowed as an underlying type.
> - Mutual substitutability should be always permitted by explicit request,
>   using either constructor notation or a suitable cast notation, e.g.,
>   `reinterpret_cast`. Such a type adjustment conversion between an opaque
>   type and its underlying type (in either direction) is expected to have no
>   run-time cost.

However, this document tries to propose a possibly more simple approach, where
a new language feature is introduced with the same meaning and functionality as
if the user autonomously implemented a new class him/herself, matching the
original type completely. Thus, it should result for the user more simple to
understand (as it simply matches already the already understood mechanics of
creating a new, unique type from nothing), and no new rules for type conversion
and selection on overloads have to be created.

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

One cannot copy a class and inherit at the same time. If such a class is needed
one would need to create it by hand with the desided functionality and
inheriting from the desired classes, as it would be done normally.

All method implementations would be the same. The copied class would inherit
from the same classes its base class inherits from. All constructors would work
in the same way.

### Adding New Functionality ###

Ideally one could specify additional methods, separate from that of Base, to add
upon the existing functionality.

```cpp
struct Base {
    void foo() { std::cout << "foo\n"; }
};

struct Derived : public Base {};

struct Copy : using Base {
    void bar() { std::cout << "bar\n"; }
};

struct CopyDerived : using Derived {};

/* Equivalent to

struct Copy {
    void foo() { std::cout << "foo\n"; }
    void bar() { std::cout << "bar\n"; }
};

struct CopyDerived : public Base {};

*/
```

Only new methods need to be implemented for that class.

#### Interfacing with the Original Class ####

In order to interface with the original class, simple conversion operators can
be added by the user explicitly at-will, in order to obtain the desired
interface. Note that if more types with this kind of compatibility were needed,
one would only need to implement them once, since copying the produced type
would copy the new, more compatible interface with it.

```cpp
struct Base {};

struct Copy : using Base {
    operator Base&() { return *reinterpret_cast<Base*>(this); }
};
```

This may also work even if the user adds new member attributes to the copy, as
they would get created after the base ones, possibly allowing for
`reinterpret_cast` to still work.

Note that the `reinterpret_cast` may produce undefined behaviour if substantial
alterations are added to the original class, as per standard `reinterpret_cast`
rules. In such cases one would have to manually map the copied class to the
original in some way.

```cpp
struct Base { int x; };

struct Copy : using Base {
    virtual void foo() {}

    operator Base&() {
        // Undefined behavior!
        return *reinterpret_cast<Base*>(this);
    }

    operator Base() { return Base{x}; };
};
```

In general the usual rules of `reinterpret_cast` apply to the copied classes
with respect to their general classes, exactly as if the copied class had been
implemented by hand.

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

### Templated Class Copy ###

The user might want to create a single templatized copy interface, and use it
multiple times. For example, one might want multiple copied classes which can
convert to their original. This could be done as follows:

```cpp
struct A {};

template <typename T>
struct TemplatizedCopy : using T {
    operator T&() { return *reinterpret_cast<T*>(this); }
};

// Could be used either via normal typedefs
using Copy1 = TemplatizedCopy<A>;

// Or via copy, depending on requirements.
struct Copy2 : using TemplatizedCopy<A> {};
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
struct B { T t; double u; };

template <>
struct B<double> { double y; double u; };

*/
```

The copied class can add additional specializations. Or specializations for a
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
struct C { int x; };

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
defined as a copy of the right hand side.

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
    void foo() = delete;
};

/* Equivalent to

struct Copy1 {
    Copy1() = default;
    void bar();
    void baz();
};

*/

struct Copy2 : using Base {
    Copy2(int);
    void abc();
};

/*

Equivalent to

struct Copy2 {
    Copy2(int);
    void foo();
    void bar();
    void abc();
};

*/
```

This feature could however present problems when the members changed also alter
behavior and/or variable types of non-modified member and non-member functions,
since the new behavior could be either erroneous or ambiguous.

### Copying and Extending Primitive Types (Optional) ###

The same syntax could be used in order to extend primitive types. Using the
extension that allows the modification of the copied types, this could allow for
creation of numeric types where some operations are disabled as needed.

```cpp
struct Id : using int {
    Id operator+(Id, Id) = delete;
    Id operator*(Id, Id) = delete;
    // Non-explicitly deleted operators keep their validity

    // Defining new operators with the old type can allow interoperativity
    Id operator+(Id, int);
    // We can convert the copied type to the old one.
    operator int() { return (*this) * 2; }
};

/* Equivalent to

class Id final {
    public:
        Id operator/(Id lhs, Id rhs) { return Id{lhs.v_ / rhs.v_}; }
        Id operator-(Id lhs, Id rhs) { return Id{lhs.v_ - rhs.v_}; }

        Id operator+(Id, int);
        operator int() { return v_ * 2; }
    private:
        int v_;
};

*/
```

Note that when copying from a primitive types inheritance is forbidden as the
generated copy is `final` (although it is allowed to keep copying the newly
created class).

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
