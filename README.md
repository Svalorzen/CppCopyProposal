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

In this proposal strong typedefs are declared and defined using a syntax very
similar to inheritance. This is to both ease the intuition about the meaning of
the code, and to emphasize that features must be added to types incrementally,
rather than by removing features to existing types.

Very simply, a strong typedef is defined in the following way:

```cpp
class type_name = type_list {
    \* additional method/member definitions *\
};
```

In this proposal we refer to the newly created class as the *type-clone* of the
original classes. We also refer to the original classes as the *type-bases* of
the *type-clone*. One cannot type-clone and use inheritance at the same time,
even though it is possible to construct an equivalent result using intermediate
helper types.

No type-base can be a primitive type. All type-bases in the `type_list` must be
aggregates, but for at most one. This is to avoid constructor conflicts.

All constructors, destructors, static and non-static member functions will be
accessible in the new class directly, their visibility (private, public,
protected) unchanged, as if they were declared in the order in which the
type-bases appear in the `type_list`. For each type-base in the `type_list`, all
mentions to that type-base in the cloned code is replaced by the type-clone.

In case of collisions between the new definitions, the new type is ill-formed.

A similar approach is taken for member attributes, with the single exception
that if they perfectly match in number, name, type, order and visibility in all
type-bases, then the type-clone gets them directly, and there is no error.

All method implementations for the type-clone would be the same as those from
the type-base it comes from. The type-clone would inherit from the same classes
its type-bases inherit from. If the type-bases inherit from the same classes,
the inheritance is collapsed (differently from how inheritance works, where
diamonds can be created).

The additional definitions that can be added in the type-clone must not conflict
with the cloned ones, or the new definition is ill-formed. Added attributes are
added after the cloned ones in terms of ordering.

### A Simple Example ###

```cpp
class MyInt {
    int x;
};

struct MyInts = MyInt {
    int y;
};

/* Equivalent to

struct MyInts {
    private:
        int x;
    public:
        int y;
};

*/

struct SumInt = MyInt {
    static SumInt operator+(SumInt lhs, SumInt rhs) { return {lhs.x + rhs.x}; }
};

struct ProdInt = MyInt {
    static ProdInt operator*(ProdInt lhs, ProdInt rhs) { return {lhs.x * rhs.x}; }
};

// Valid as SumInt and ProdInt both have a single private attribute of type int
// called x
struct SumProdInt = SumInt, ProdInt {};
// struct SumInts = MyInts, SumInt {}; Ill-formed since SumInt does not have int y;

/* Equivalent to

struct SumProdInt {
    static SumProdInt operator+(SumProdInt lhs, SumProdInt rhs) { return {lhs.x + rhs.x}; }
    static SumProdInt operator*(SumProdInt lhs, SumProdInt rhs) { return {lhs.x * rhs.x}; }
    private:
        int x;
};

*/

```

#### Interfacing with the Original Class ####

In order to interface with a type-base, simple conversion operators can be added
by the user explicitly if needed, in order to obtain the desired interface.

```cpp
struct MyIntConvertible = SumProdInt {
    operator MyInt() const { return {x}; }
};
```

### Overloads ###

As a type-clone is a new, unique type, overloads are resolved separately for
different type-clones (and type-bases).

```cpp
class Position = std::pair<double, double> {};
class Distance = std::pair<double, double> {};

Position operator+(const Position & lhs, const Distance & rhs) {
    return Position(lhs.first + rhs.first, lhs.second + rhs.second);
}

Distance operator+(const Distance & lhs, const Distance & rhs) {
    return Distance(lhs.first + rhs.first, lhs.second + rhs.second);
}

Position p(1, 1);
Distance d(1, 1);

p + d; // OK, position + distance
d + d; // OK, distance + distance
p + p; // Error, can't sum two positions
```

### Nested Types, Friends and Non-Members ###

A type-clone results in the duplication of all types and entities that directly
depend on the base-type. It also results in the duplication of aliases. It does
also include friends, although the original friend declaration was not templated
it will result in a simple declaration. In no case non-member functions are
duplicated automatically.

While in a type-clone context, all nested types can be extended as if they were
being type-cloned themselves.

```cpp

template <typename T>
void foo(T) {}

void bar(Base) {}

struct Base {
    using X = int;

    struct Nested {
        double proc();
    };

    void useNested(Nested);

    friend void foo<Base>(Base);
    friend void bar(Base);
};

struct Clone = Base {
    struct Nested {
        int func();
    };
};

/* Equivalent to

struct Clone {
    using X = int;

    struct Nested {
        double proc();
        int func();
    };

    void useNested(Nested);

    friend void foo<Clone>(Clone);
    friend void bar(Clone); // Declaration only, cannot be called as not defined
};

*/
```

Non-member non-template methods would need to be implemented by hand in order to
work with the type-clone. This proposal encourages the usage of template
non-member functions, as they perfectly fit the type of structures produced with
this proposal.

### Primitive Types ###

Strong typedefs are often discussed in terms of primitive types. However, this
proposal does *not* extend to primitive types. While doing so would be nice, it
seems to be hard to define a good, intuitive and useful way to create such
strong typedefs.

The main problem of creating a primitive type strong typedef is that the
handling of all original operators - whether they should still work or not, and
whether they should be compatible with the original type - depends heavily on
context. Serial numbers should not be multiplied, for example. This would
require removal of operators from existing types, which goes contrary to the
goals of this proposal (to have types be built incrementally).

With this proposal we support what has been done to emulate strong typing up
until now, i.e. explicitly write classes with the desired interfaces. This
proposal simply enables a simple and relatively intuitive way to do so, which
at the same time minimizes the amount of redundancy.

The choice to not extend the syntax to primitive types also mirrors the fact
that in C++ it is not possible to inherit from primitive types.


Compatibility
-------------

As the syntax is new, no old code would be affected.
