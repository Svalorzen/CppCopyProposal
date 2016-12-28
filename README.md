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
  extend it with context-specific methods. Examples: searching for
  "`std::vector` inheritance" yields many results of users trying to maintain
  the original interface and functionality but add one or two methods.

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

We refer to the newly created class as the *type-copy* of the original class. We
also refer to the original class as the *type-base* of the *type-copy*. One
cannot type-copy a class and inherit at the same time. If such a new class was
needed one would need to create it by hand with the desired functionality and
inheriting from the desired classes, as it would be done normally.

All method implementations for the type-copy would be the same. The type-copy
would inherit from the same classes its type-base inherits from. All
constructors would work in the same way.

As the proposed syntax is also used in
[P0352R0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0352r0.pdf),
an alternative syntax could be used:

```cpp
struct Copy = Base {};
```

### Adding New Functionality ###

A type-copy is allowed to define additional methods upon definition, as it would
be done in a normal class definition. The type-copy has access to all its
members in the same way the type-base has access to its own.

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

struct Copy : using Base {
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

struct Copy : using Base {
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

Non-member methods, if not templatized, would need to be implemented by hand in
order to work with the type-copy. Friend declarations would also need to be
inserted by hand into the type-copy.

Methods to create type-copies of non-member methods have been considered, but
have been found inappropriate to bundle in this proposal. The reason is that
once a mechanism to type-copy a method has been established, it should be
general, and not only bound to type-copies. In practice it would result in the
ability of converting any function into a template. Such a change alone would
have a scope similar in size to this proposal, and so it was not included here.

Please note that while this limitation could drastically reduce the usefulness
of this feature in some cases, usually most classes are not so much dependent on
non-members that this would be a problem in practice. We argue that having
strong typing in C++ is better than not having it, even if in certain situations
it is of limited use. No silver bullet exists.

### Templated Type-Copy ###

Type-copies can be templatized.

```cpp
struct A { int x; };

template <typename T>
struct TemplatizedCopy : using T {
    static_assert(std::is_standard_layout<T>::value,
                  "Can't use this with a non-standard-layout class");

    operator T&() { return *reinterpret_cast<T*>(this); }
};

// Could be used either via normal typedefs
using Copy1 = TemplatizedCopy<A>;

// Or via copy, depending on requirements.
struct Copy2 : using TemplatizedCopy<A> {};
```

### Type-Copy of Template Classes ###

Template classes can be type-copied.

```cpp
template <typename T>
struct A {};

template <typename T>
struct B : using A<T> {};

B<int> b;
```

The type-copy must have equal or less template parameters than its type-base.
Type-copy of partial or full specializations are allowed:

```cpp
template <typename T, typename U>
struct A {};

template <typename T>
struct B : using A<T, double> {};

B<int> b;
```

When the type-base has partial specializations, the type-copy includes all those
that are still applicable on its parameters.

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

A templated type-copy is allowed to implement specializations independently of
its type-base. However, specializations already present from the type-base
cannot be overridden. Template specializations can type-copy another class.

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

### Type-Copy of Multiple Dependent Classes ###

Type-copying multiple classes using the simple syntax we have described can be
impossible if those classes depend on each other. This is because each copy
would depend on the originals, rather than on the newly created type-copies.

A type-copy is allowed to substitute any used class in its type-base for another
class, given that the substitution is a type-copy of the substituted class.

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
defined as a type-copy of the right hand side.

Similarly, this could be done to type-copy whole hierarchies of classes.

```cpp
struct A {};
struct B : A {};

struct C : using A {};
struct D : using B {
    using class C = A;
};
```

### STL Traits ###

Traits should be included in the standard library in order to determine whether
a class is a type-copy of another, or if it has been derived from a type-copy
(copies/inheritances could be nested arbitrarily).

```cpp
struct Base {};

struct Copy : using Base {};

static_assert(std::is_copy_of<Copy, Base>::value);

struct ChildCopy : public Copy {};

struct CopyChildCopy : using ChildCopy {};

static_assert(std::is_copy_base_of<Base, CopyChildCopy>::value);
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

struct IntWrapperAsInt : using IntWrapper {
    operator int() { return v_; }
};

struct IntWrapperWithSum : using IntWrapper {
    IntWrapperWithSum operator+(IntWrapperWithSum other) {
        return v_ + other.v_;
    }
};

struct IntWrapperWithAlgebra : using IntWrapperWithSum {
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
