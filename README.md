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

Motivation
----------

- Scientific libraries where a type has different behaviors depending on context
  have currently no simple way to indicate the semantic differences. Since a
  `typedef` does not allow a separate overload on the alias - since it is still
  the "old" type - libraries have to resort to imperfect techniques, such as
  copying, wrapping or inheriting the needed type. Examples: coordinates in a
  plane (rectangular, polar), vectors of double (probabilities, values),
  separate physical values (kilograms, speeds).
- Easier maintainability of code which is known to be the same, rather than
  forcing multiple copies of the same code to coexist.
- Avoiding misuse of inheritance in order to provide a copy-paste alternative.
  This can result in very deep hierarchies of types which should really not have
  anything to do with each other.
- Enabling users to use existing and presumably correct types and extend them
  with context-specific methods. Examples: searching for "`std::vector`
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

Inheritance requires redefinition of constructors (although easier now with the
`using` directive). Worse, it creates a tight dependency between two classes
which very often not intended. Classes may be converted to a common ancestor
even though that is undesired or even dangerous in case of implicit conversions.

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

Templates produce a new type for each instantiation. They are unfortunately not
applicable to previously existing code. For new code, they would require the
creation of "fake" template parameters that would need to vary in order to
produce separate types.

In addition, another severe limitation is that class extension through templates
is not possible: variations would need to be made either through specialization,
which itself requires copying existing code, or though inheritance, with the
previously mentioned pitfalls.

Previous Work
-------------

Strong typedefs have already been proposed for the C++ language multiple times
([N1706](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2004/n1706.pdf),
[N1891](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2005/n1891.pdf),
[N3515](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3515.pdf),
[N3741](https://isocpp.org/files/papers/n3741.pdf)). These typedefs are named
*opaque typedefs*, and these papers try to explore and define exactly the
behavior that such typedefs should and would have when used to create new types.
In particular, the keywords `public`, `protected` and `private` are used in
order to create a specific relation with the original type to determine how the
new type is allowed to be cast back to the original type or be used in its place
during overloads.

This document shares many of the same principles, for example (quoting from
N3741):

> - Consistent with restrictions imposed on analogous relationships such as
>   base classes underlying derived classes and integer types underlying enums,
>   an underlying type should be (1) complete and (2) not cv-qualiﬁed. We also do
>   not require that any enum type, reference type, array type, function type, or
>   pointer-to-member type be allowed as an underlying type.

However, this document tries to propose a possibly more simple approach, where
a new language feature is introduced with the same meaning and functionality as
if the user autonomously implemented a new class him/herself, matching the
original type completely. Thus, it should result for the user more simple to
understand (as it simply matches already the already understood mechanics of
creating a new, unique type from nothing), and no new rules for type conversion
and selection on overloads have to be created.

Design Decisions
----------------

- It should be simple to create and extend a strong typedef of a given class as
  it is when using inheritance.
- The interface of types should be built incrementally, as it is now, and it
  should not be possible to remove methods/attributes of the original type. This
  allows to understand type hierarchies more easily.
- Strong typedefs are considered as separate types, but it should be possible to
  allow their usage in place of their original type with additional syntax.
- Do not allow automatic creation of non-member functions for strong typedefs,
  taking from non-member functions that use the original type in their
  signature. Instead, we encourage usage of non-member templated functions.
- Naive solutions such as `strong using StrongAlias = int` have been discarded
  as they are not easily generalizable without introducing unintuitive corner
  cases.

Syntax
------

### Simple Case ###

A strong typedef is defined as follows:

```cpp
class Base {
    public:
        Base() : x(0) {}
        void foo() { std::cout << "foo " << x << "\n"; }
    private:
        int x;
};

struct Copy = Base {};

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

We refer to the newly created class as the *type-copy* of the original class. We
also refer to the original class as the *type-base* of the *type-copy*. One
cannot type-copy a class and inherit at the same time. If such a new class was
needed one would need to create it by hand with the desired functionality and
inheriting from the desired classes, as it would be done normally.

All method implementations for the type-copy would be the same. The type-copy
would inherit from the same classes its type-base inherits from. All explicitly
defined constructors and destructors would work in the same way. Implicitly
defined/deleted constructors and destructors may be overridden or deleted in the
type-copy.

```cpp
struct A {
    A(int);
};
// Impossible to construct
struct B {
    A a;
};
// Can be constructed
struct C = B {
    C(int i) : a(i) {}
};
```

Unless where noted, the new class behaves with respect to all C++ rules as if it
had been implemented by hand.

### Adding New Functionality ###

A type-copy is allowed to define additional methods and attributes upon
definition, as it would be done in a normal class definition. The type-copy has
access to all its members in the same way the type-base has access to its own.

Note that additional attributes will be initialized in type-copied constructors
in the same way as if they weren't specified at all. Additional constructors can
be specified if needed.

```cpp
struct Base {
    void foo() { std::cout << "foo\n"; }
};

struct Derived : public Base {};

struct Copy = Base {
    void bar() { std::cout << "bar\n"; }

    int x;      // Unspecified on construction
    int j = 10; // 10 on construction
};

struct CopyDerived = Derived {};

/* Equivalent to

struct Copy {
    void foo() { std::cout << "foo\n"; }
    void bar() { std::cout << "bar\n"; }

    int x;
    int j = 10;
};

struct CopyDerived : public Base {};

*/
```

All newly declared methods in the type-copy must of course be implemented in
order to be used.

#### Interfacing with the Original Class ####

In order to interface with a type-base, simple conversion operators can be added
by the user explicitly if needed, in order to obtain the desired interface. Note
that if more types with this kind of compatibility were needed, one would only
need to implement them once, since recursively type-copying the type-copy would
duplicate the new, more compatible interface with it.

```cpp
struct Base {
    public:
        int x;

    private:
        double y;
};

struct Copy = Base {
    operator Base() { return Base{x, y}; }
};
```

`reinterpret_cast` may also be used to convert back to the original class,
limited by the tool's already existing rules.

In general the usual rules of `reinterpret_cast` apply to the copied classes
with respect to their general classes, exactly as if the copied class had been
implemented by hand.

### Overloads ###

A type-copy is a new, unique type, which cannot be substituted with its
type-base nor other type-copies of any kind - unless appropriate conversion
operators are explicitly added by the user. Overloads are resolved as if each
type-copy is a unique type. Once again, the general rule of "behaves as if
implemented by hand" applies.

```cpp
class Position = std::pair<double, double> {};
class Distance = std::pair<double, double> {};

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

### Nested Types, Friends and Non-Members ###

A type-copy results in the duplication of all types and entities that directly
depend on the base-type. It also results in the duplication of aliases. However,
it does **not** include friends, nor it creates type-copies of non-member
functions.

While in a type-copy context, all nested types can be extended as if they were
being type-copied themselves.

```cpp

struct Base {
    using X = int;

    struct Nested {
        double proc();
    };

    void useNested(Nested);

    friend void foo(Base);
    friend void bar() { std::cout << "bar\n"; }
};

void foo(Base b);
void baz(Base::Nested n);

struct Copy = Base {
    struct Nested {
        int func();
    };
};

/* Equivalent to

struct Copy {
    using X = int;

    struct Nested {
        double proc();
        int func();
    };

    void useNested(Nested);

};

*/
```

### Primitive Types ###

Strong typedefs are often discussed in terms of primitive types. However, this
proposal does *not* extend to primitive types. While doing so would be nice, it
seems to be hard to define a good, intuitive and useful way to create such
strong typedefs.

The main problem of creating a primitive type strong typedef is that the
handling of all original operators - whether they should still work or not, and
whether they should be compatible with the original type - depends heavily on
context. Serial numbers should not be multiplied, for example. However, manually
specifying whether each operator should be present or not is a very dull task.

While wrapping a primitive type in a small class and exposing only the needed
operators can work, this process becomes tedious to do many times, since
inheritance cannot be used lest risk unwanted conversions.

However, this proposal makes wrapping and incrementally exposing operators easy,
removes code duplication and prevents any unwanted conversions. While not
perfect in all cases, as still some code must be written to expose global
operators with base-types (as those are not copied), we argue that this system
can still be useful in dealing with this problem.

Once created, wrapper types can then be copied indefinitely when needed, and
thus allow for a robust and efficient way to substitute for a lack of primitive
type strong typedefs.

```cpp
class IntWrapper final {
    public:
        IntWrapper() : v_(0) {}
        explicit IntWrapper(int v) : v_(v) {}
        IntWrapper(const IntWrapper & other) : v_(other.v_) {}

    private:
        int v_;
};

struct IntWrapperAsInt = IntWrapper {
    operator int() { return v_; }
};

struct IntWrapperWithSum = IntWrapper {
    IntWrapperWithSum operator+(IntWrapperWithSum other) {
        return v_ + other.v_;
    }
};

struct IntWrapperWithAlgebra = IntWrapperWithSum {
    // And so on...
    IntWrapperWithAlgebra operator-(IntWrapperWithAlgebra other);
    IntWrapperWithAlgebra operator/(IntWrapperWithAlgebra other);
    IntWrapperWithAlgebra operator*(IntWrapperWithAlgebra other);
    IntWrapperWithAlgebra operator%(IntWrapperWithAlgebra other);
};
```

The choice of not extending this syntax to primitive types also mirrors the fact
that in C++ it is not possible to inherit from primitive types.

Compatibility
-------------

As the syntax is new, no old code would be affected.
