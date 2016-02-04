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

The concept definitions consist of *requirements*, which express syntax and must be checked by the compiler, and *axioms*, which express semantics and must not be checked by the compiler.


### Language concepts

Language concepts are intrinsic to the C++ programming language. [Stroustrup12](#Stroustrup12) §3.2


#### Type classifications

The following concepts classify fundamental types. Their semantics are fully described in the C++ standard, using type traits.


##### Integral

Starting with C++11, the `Integral` concept is defined by `is_integral<T>::value`.

**Example:**

    Integral<int>          // is true
    Integral<unsigned int> // is true
    Integral<double>       // is false


##### SignedIntegral

Starting with C++11, the `SignedIntegral` concept is defined by `is_signed<T>::value`.

**Example:**

    SignedIntegral<int>          // is true
    SignedIntegral<signed int>   // is true
    SignedIntegral<unsigned int> // is false
    SignedIntegral<double>       // is true


##### UnsignedIntegral

Starting with C++11, the `UnsignedIntegral` concept is defined by `is_unsigned<T>::value`.

**Example:**

    Integral<int>          // is true
    Integral<signed int>   // is true
    Integral<unsigned int> // is false
    Integral<double>       // is true


#### Type relations

The following concpets describe relationships between types.


##### Same

The `Same` concept is built-in to the language, described by the standard library type `is_same`. For two types `X` and `Y`, `Same<X, Y>` is true if `X` and `Y` denote exactly the same type after elimination of aliases.

**Example:**

    using Pi = int*;
    using I = int;
    Same<Pi, I> // is false
    Same<Pi, I*> // is true


##### Derived

The `Derived` concept returns true if one type is derived from another.

**Example:**

    class B {};
    class D : B {};
    Derived<D, B> // is true: D is derived from B
    Derived<B, D> // is false: B is a base class of D
    Derived<B, B> // is true: a class is derived from itself


##### Convertible

The `Convertible` concept expresses the requirement that a type `T` can be implicitly converted to a `U`.

**Example:**

    Convertible<double, int>             // is true: int i = 2.7; (ok)
    Convertible<double, complex<double>> // is true: complex<double> d = 3.14; (ok)
    Convertible<complex<double>, double> // is false: double d = complex<double>2,3 (error)
    Convertible<int, int>                // is true: a type is convertible to itself
    Convertible<Derived, Base>           // is true: derived types can be converted to base types


##### Common

The `Common` concept expresses that two types `T` and `U` can both be unambiguously converted to a third type `C`, i.e. they share a *common type*. `C` can be the same type as `T` and `U` or it can be a different type:

    requirement: CommonType<T, U> (alias for the standard type trait common_type<T, U>::type)
    axiom: eq(t1, t2) <=> eq(C{t1}, C{t2})
    axiom: eq(u1, u2) <=> eq(C{u1}, C{u2})

**Example:**

    Common<int, double>           // is true
    Common<int, int>              // is true
    Common<char*, std::string>    // is true
    Common<employee, std::string> // is probably false


### Foundational concepts

Foundational concepts form the basis of a style of programming or are needed to write programs in that style. The following concepts describe the basis of the value-oriented programming style on which the STL is based. [Stroustrup12](#Stroustrup12) §3.3


##### EqualityComparable

The ability to compare objects for equality is described by the
`EqualityComparable` concept:

    requirement: bool a == b (expression convertible to bool)
    requirement: bool a != b (expression convertible to bool)
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
        requirement: T* == &a (an object of this type can have its address taken and the result is a pointer to T)
        axiom: &a == addressof(a) (the result is an address to a)
        axiom: eq(a, a) (non-volatility)
    
    Destructible:
        requirement: a.~T() (destruction)
        requirement: noexcept(a.~T()) (noexcept)
    
    Initialization:
        requirement: T{} (default construction)
        requirement: T{a} (copy construction)
        requirement: T& == {a = b} (copy assignment)
        axiom: (copy semantics)
            eq(T{a}, a)
            eq(a = b, b)
        axiom: (move semantics (C++11))
            eq(a, b) => eq(T{move(a)}, b)
            eq(b, c) => eq(a = move(b), c)
    
    Allocation:
        requirement: T* == { new T }
        requirement: delete new T
        requirement: T* == { new T[n] }
        requirement: delete[] new T[n]

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

    requirement: bool a < b
    requirement: bool a > b
    requirement: bool a <= b
    requirement: bool a >= b
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
