C++ concepts quick reference
============================

This document is a summary of proposed core C++ concepts and a collection of design guidelines for C++ classes and algorithms. It is intended to be used as a quick reference when designing new classes/algorithms or refactoring old ones.
For brevity and simplicity of use, it does not discuss at length the *whys* of the guidelines, but instead presents references to further reading about the subject.

The guidelines cover C++98, C++11, C++14, and C++17, with notes on differences between the versions.


Template constraints
--------------------

In C++17, template parameter types can be constrained to satisfy concepts. There are three syntactic forms available. [Sutton13](#Sutton13) §2

**Example:**

    // Most general form with requires clause
    template <typename C>
       requires Sortable<C>()
    void sort(C& container);
    
    // Shorthand replacing typename
    template <Sortable C>
    void sort(C& container);
    
    // Shorthand replacing template declaration
    void sort(Sortable& container);

Putting `requires` behind a comment for the first form or using `#define` to alias the `typename` keyword provides easy ways to declare template constraints in anticipation of compiler support for the first two forms.

Constraints can also be used with member functions.

**Example:**

    template<Object T, Allocator A>
    class vector {
        requires Equality_comparable<T>()
            bool operator==(const vector& x) const;
    
        requires Totally_ordered<T>()
            bool operator<(const vector& x) const;
    };

Multi-type constraints can easily be expressed using explicit form with conjunctions or shorthand constraint form with implicit conjunction: [Sutton13](#Sutton13) §2.1.1

**Example:**

    // Explicit form
    template<typename S, typename T>
        requires Sequence<S>() && Equality_comparable<T, Value_type<S>>()
    Iterator_type<S> find(S&& sequence, const T& value);
    
    // Shorthand form
    template<Sequence S, Equality_comparable<Value_Type<S>> T>
    Iterator_type<S> find(S&& sequence, const T& value);


Concept definitions
-------------------

The concepts presented in this section are not yet standardized and there may be substantial differences in C++17. The concepts are described in greater detail in [Stroustrup12](#Stroustrup12).


### Language concepts

Language concepts are intrinsic to the C++ programming language. [Stroustrup12](#Stroustrup12) §3.2


#### Type classifications

The following concepts classify fundamental types.


##### Integral


##### SignedIntegral


##### UnsignedIntegral


#### Type relations

The following concpets describe relationships between types.


##### Same


##### Derived


##### Convertible


##### Common


### Foundational concepts

Foundational concepts form the basis of a style of programming or are needed to write programs in that style. The following concepts describe the basis of the value-oriented programming style on which the STL is based. [Stroustrup12](#Stroustrup12) §3.3


##### EqualityComparable

he ability to compare objects for equality is described by the
`EqualityComparable` concept:

    bool a == b (expression convertible to bool)
    bool a != b (expression convertible to bool)
    axiom: a == b <=> eq(a, b) (specification of equality)
    axiom: (equivalence relation)
        a == a (reflexive)
        a == b => b == a (symmetric)
        a == b && b == c <=> a == c (transitive)
    axiom: a != b <=> !(a == b) (complement)

where the `eq(a, b)` function implies that `a` and `b` represent the same abstract entity (not necessarily representational or member-wise equality). Examples from the standard library include: [Stroustrup12](#Stroustrup12) §3.1.1

- Two tuples are equal if and only if they have the same number of sub-objects and all of their sub-objects compare equal using `==`.
- Two vectors are equal if and only if they have the same size and contain the same elements as if compared using the `equal` algorithm (which compares using `==`.
- Two complex numbers are equal if and only if they have equal real and imaginary components.

**Example implementation:**

    struct Rational {
        int numerator = 0;
        int denominator = 0;
    };
    
    inline bool operator==(Rational& lhs, const Rational& rhs)
    { 
        return lhs.numerator * rhs.denominator == lhs.denominator * rhs.numerator; 
    }
        
    inline bool operator!=(const Rational& lhs, const Rational& rhs)
    { 
        return !(lhs == rhs); 
    }


##### Semiregular

The `Semiregular` concept describes types that behave in regular ways except that they might not be comparable using `==`. Examples of semiregular types include all built-in C++ integral and floating point types, all trivial classes, `std::string` and standard containers of copyable types. [Stroustrup12](#Stroustrup12) §3.3

The `Semiregular` concept is defined as:

    Object:
        T* == &a (an object of this type can have its address taken and the result is a pointer to T)
        axiom: &a == addressof(a) (the result is an address to a)
        axiom: eq(a, a) (non-volatility)
    
    Destructible:
        a.~T() (destruction)
        noexcept(a.~T()) (noexcept)
    
    Initialization:
        T{} (default construction)
        T{a} (copy construction)
        T& == {a = b} (copy assignment)
        axiom: (copy semantics)
            eq(T{a}, a)
            eq(a = b, b)
        axiom: (move semantics (C++11))
            eq(a, b) => eq(T{move(a)}, b)
            eq(b, c) => eq(a = move(b), c)
    
    Allocation:
        T* == { new T }
        delete new T
        T* == { new T[n] }
        delete[] new T[n]

Note that copy semantics do not preclude shallow copies. If they did, we might not be able to conclude that pointers are copyable. Shallow-copied types may be regular with respect to copy and move semantics, but they probably have some operations that are not equality preserving. This is definitely the case with some `InputIterator`s such as `istream_iterator`.

Proper use of the `Semiregular` concept requires that it is evaluated over value types, not object types:

- Constants (`const T`) and `constexpr` objects do not have `Semiregular` types since they cannot be assigned to.
- Volatile types (`volatile T`) are not `Semiregular` since their objects’ states may by changed externally.
- Reference types types are not `Semiregular` because the syntax used for copy construction does not actually create a copy; it binds a reference to an
object.


##### Regular

The `Regular` concept is the union of the `Semiregular` and `EqualityComparable` concepts. [Stroustrup12](#Stroustrup12) §3.3


##### TotallyOrdered

The `TotallyOrdered` concept requires The `EqualityComparable` concept plus the the inequality operators `<`, `>`, `<=`, and `>=`, with the following requirements: [Stroustrup12](#Stroustrup12) §3.3

    a < b
    a > b
    a <= b
    a >= b
    axiom: (total ordering using <)
        !(a < a) (irreflexive)
        a < b => !(b < a) (assymetric)
        a < b && b < c => a < c (transitive)
        a < b || b < a || a == b (total)
    axiom: (total ordering using >, <=, >=)
        a > b <=> b < a
        a <= b <=> !(b < a)
        a => b <=> !(b > a)

**Example implementation:**

    struct Rational {
        int x = 0;
        int y = 0;
        int z = 0;
    };
    
    inline bool operator==(Rational& lhs, const Rational& rhs)
    {
        return lhs.numerator * rhs.denominator == lhs.denominator * rhs.numerator; 
    }
    
    inline bool operator!=(const Rational& lhs, const Rational& rhs)
    {
        return !(lhs == rhs); 
    }
    
    inline bool operator<(Rational& lhs, const Rational& rhs) 
    {
        return lhs.numerator * rhs.denominator < lhs.denominator * rhs.numerator; 
    }
    
    inline bool operator>(const Rational& lhs, const Rational& rhs) 
    {
        return rhs < lhs;
    }
    
    inline bool operator<=(const Rational& lhs, const Rational& rhs)
    {
        return !(lhs > rhs); 
    }
    
    inline bool operator>=(const Rational& lhs, const Rational& rhs)
    {
        return !(lhs < rhs); 
    }


### Function concepts

Function concepts describe requirements on function types. [Stroustrup12](#Stroustrup12) §3.4
 

### Iterator concepts

Iterators are one of the fundamental abstractions of the STL. They generalize the notion of pointers. [Stroustrup12](#Stroustrup12) §3.5


### Rearrangements

Rearrangements group together iterator requirements of algorithm families. [Stroustrup12](#Stroustrup12) §3.6


References
----------

<a name="Stroustrup12"></a>
[Stroustrup12]
"A concept design for the STL", ISO N3551
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3351.pdf

<a name="Sutton13"></a>
[Sutton13]
"Concepts Lite: Constraining Templates with Predicates", ISO N3551
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3580.pdf

<a name="ISO15"></a>
[ISO15]
"Programming Languages — C++ Extensions for Concepts", ISO N4549
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4549.pdf
