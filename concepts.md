C++ concepts quick reference
============================

This document is a summary of proposed core C++ concepts and a collection of design guidelines for C++ classes and algorithms. It is intended to be used as a quick reference when designing new classes/algorithms or refactoring old ones.
For brevity and simplicity of use, it does not discuss at length the *whys* of the guidelines, but instead presents references to further reading about the subject.

The guidelines cover C++98, C++11, C++14, and C++17, with notes on differences between the versions.


Template constraints
--------------------

In the C++17 concepts TS, template parameter types can be constrained to satisfy concepts. There are three syntactic forms available. [Sutton13](#Sutton13) §2

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

Aliasing the `requires` keyword with a variadic macro, or using `#define` to alias the `typename` keyword, provides easy ways to declare template constraints in anticipation of compiler support for the first two forms, treating them as documentation.

**Example:**

    #define requires(...)
    
    #define Sortable typename

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


Concept description mechanisms
------------------------------

Before (and to some extent after) C++17, Several mechanisms dealing with types are needed to describe concepts: [Stepanov09](#Stepanov09) §1.7

- Type attributes
- Type functions
- Type constructors


### Type attributes

A *type attribute* is a mapping from a type to a value describing some characteristic of the type. Examples of type attributes in C++ are the built-in type attribute `sizeof(T)`, the alignment of an object of a type, and the number of members in a `struct`.


### Type functions

A *type function* is a mapping from a type to an affiliated type. Examples of type functions is a function that given a `*T`, returns the type `T`. In C++, type functions are implemented by *traits classes*, specialized for each particular type where necessary.

**Example:**

    template<typename T>
    struct value_type {
        typedef T type;
    };
    
    // Convenience notation
    #define ValueType(T) typename value_type<T>::type


### Type constructors

A *type constructor* is a mechanism for creating a new type from one or more existing types. Examples of type constructors in C++ are the `*` (pointer) operator, `struct`, which is a n-ary type constructor, and `std::pair`, which returns a structure of two members.


Concept definitions
-------------------

The concepts presented in this section are not yet standardized and there may be substantial differences in the C++17 concepts TS. The concepts are described in greater detail in [Stroustrup12](#Stroustrup12), and some concept definitions are taken from [Niebler15](#Niebler15).

The concept definitions consist of *requirements*, which express syntax and must be checked by the compiler, and *axioms*, which express semantics and must not be checked by the compiler.


### Language concepts

Language concepts are intrinsic to the C++ programming language. [Stroustrup12](#Stroustrup12) §3.2, [Niebler15](#Niebler15) §19.2


##### Assignable

**C++ Defintion:**

    template <class T, class U>
    concept bool Assignable() {
        return Common<T, U>() && requires(T&& a, U&& b) {
            { std::forward<T>(a) = std::forward<U>(b) } -> Same<T&>;
        };
    }


##### Swappable

**C++ Defintion:**

    template <class T>
    concept bool Swappable() {
        return requires(T&& a, T&& b) {
            ranges::swap(std::forward<T>(a), std::forward<T>(b));
        };
    }
    
    template <class T, class U>
    concept bool Swappable() {
        return Swappable<T>() &&
            Swappable<U>() &&
            Common<T, U>() &&
            requires(T&& t, U&& u) {
                ranges::swap(std::forward<T>(t), std::forward<U>(u));
                ranges::swap(std::forward<U>(u), std::forward<T>(t));
            };
    }


#### Type classifications

The following concepts classify fundamental types. Their semantics are fully described in the C++ standard, using type traits.


##### Integral

The `Integral` concept is defined by the type trait `is_integral<T>::value`.

**C++ definition:**

    template <class T>
    concept bool Integral() {
        return std::is_integral<T>::value;
    }

**Example:**

    Integral<int>          // is true
    Integral<unsigned int> // is true
    Integral<double>       // is false


##### SignedIntegral

The `SignedIntegral` concept adds the `is_signed<T>::value` requirement.

**C++ definition:**

    template <class T>
    concept bool SignedIntegral() {
        return Integral<T>() && std::is_signed<T>::value;
    }

**Example:**

    SignedIntegral<int>          // is true
    SignedIntegral<signed int>   // is true
    SignedIntegral<unsigned int> // is false
    SignedIntegral<double>       // is true


##### UnsignedIntegral

The `UnsignedIntegral` concept is defined in terms of the `Integral` and `SignedIntegral` concept.

**C++ definition:**

    template <class T>
    concept bool UnsignedIntegral() {
        return Integral<T>() && !SignedIntegral<T>();
    }

**Example:**

    UnsignedIntegral<int>          // is false
    UnsignedIntegral<signed int>   // is false
    UnsignedIntegral<unsigned int> // is true
    UnsignedIntegral<double>       // is false


#### Type relations

The following concpets describe relationships between types.


##### Same

The `Same` concept is built-in to the language, described by the standard library type `is_same`. For two types `T` and `U`, `Same<T, U>` is true iff `T` and `U` denote exactly the same type after elimination of aliases. For the purposes of constraint checking, `Same<T, U>()` implies `Same<U, T>`.

**C++ definition:**

    template <class T, class U>
    concept bool Same() {
        return std::is_same<T, U>::value;
    }

**Example:**

    using Pi = int*;
    using I = int;
    Same<Pi, I> // is false
    Same<Pi, I*> // is true


##### DerivedFrom

The `DerivedFrom` concept returns true if one type is derived from another.

**C++ definition:**

    template <class T, class U>
    concept bool DerivedFrom() {
        return std::is_base_of<U, T>::value;
    }

**Example:**

    class B {};
    class D : B {};
    DerivedFrom<D, B> // is true: D is derived from B
    DerivedFrom<B, D> // is false: B is a base class of D
    DerivedFrom<B, B> // is true: a class is derived from itself


##### ConvertibleTo

The `ConvertibleTo` concept expresses the requirement that a type `T` can be implicitly converted to a `U`.

**C++ definition:**

    template <class T, class U>
    concept bool ConvertibleTo() {
        return std::is_convertible<T, U>::value;
    }

**Example:**

    ConvertibleTo<double, int>             // is true: int i = 2.7; (ok)
    ConvertibleTo<double, complex<double>> // is true: complex<double> d = 3.14; (ok)
    ConvertibleTo<complex<double>, double> // is false: double d = complex<double>2,3 (error)
    ConvertibleTo<int, int>                // is true: a type is convertible to itself
    ConvertibleTo<Derived, Base>           // is true: derived types can be converted to base types


##### Common

The `Common` concept expresses that two types `T` and `U` can both be unambiguously converted to a third type `C`, i.e. they share a *common type*. `C` can be the same type as `T` and `U` or it can be a different type:

    requirement: CommonType<T, U> (alias for the standard type trait common_type<T, U>::type)
    axiom: eq(t1, t2) <=> eq(C{t1}, C{t2})
    axiom: eq(u1, u2) <=> eq(C{u1}, C{u2})

**C++ definition:**

    template <class T, class U>
    concept bool Common() {
        return requires (T t, U u) {
            typename std::common_type_t<T, U>;
            typename std::common_type_t<U, T>;
            requires Same<std::common_type_t<U, T>, std::common_type_t<T, U>>();
            std::common_type_t<T, U>(std::forward<T>(t));
            std::common_type_t<T, U>(std::forward<U>(u));
        };
    }

Users are free to specialize `common_type` when at least one parameter is a user-defined type. Those specializations are considered by the `Common` concept.

**Example:**

    Common<int, double>           // is true
    Common<int, int>              // is true
    Common<char*, std::string>    // is true
    Common<employee, std::string> // is probably false


### Foundational concepts

Foundational concepts form the basis of a style of programming or are needed to write programs in that style. The following concepts describe the basis of the value-oriented programming style on which the STL is based. [Stroustrup12](#Stroustrup12) §3.3


##### Destructible

**C++ definition:**

    template <class T>
    concept bool Destructible() {
        return requires (T t, const T ct, T* p) {
            { t.∼T() } noexcept;
            { &t } -> Same<T*>; // not required to be equality preserving
            { &ct } -> Same<const T*>; // not required to be equality preserving
            delete p;
            delete[] p;
        };
    }


##### Constructible

**C++ definition:**

    template <class T, class ...Args>
    concept bool __ConstructibleObject = // exposition only
        Destructible<T>() && requires (Args&& ...args) {
            T{std::forward<Args>(args)...}; // not required to be equality preserving
            new T{std::forward<Args>(args)...}; // not required to be equality preserving
        };
    
    template <class T, class ...Args>
    concept bool __BindableReference = // exposition only
        std::is_reference<T>::value && requires (Args&& ...args) {
            T(std::forward<Args>(args)...);
        };
    
    template <class T, class ...Args>
    concept bool Constructible() {
        return __ConstructibleObject<T, Args...> || __BindableReference<T, Args...>;
    }


##### DefaultConstructible

**C++ definition:**

    template <class T>
    concept bool DefaultConstructible() {
        return Constructible<T>() &&
            requires (const size_t n) {
                new T[n]{}; // not required to be equality preserving
            };
    }


##### MoveConstructible

**C++ definition:**

    template <class T>
    concept bool MoveConstructible() {
        return Constructible<T, remove_cv_t<T>&&>() && 
            ConvertibleTo<remove_cv_t<T>&&, T>();
    }


##### CopyConstructible

**C++ definition:**

    template <class T>
    concept bool CopyConstructible() {
        return MoveConstructible<T>() &&
            Constructible<T, const std::remove_cv_t<T>&>() &&
            ConvertibleTo<std::remove_cv_t<T>&, T>() &&
            ConvertibleTo<const std::remove_cv_t<T>&, T>() &&
            ConvertibleTo<const std::remove_cv_t<T>&&, T>();
    }


##### Movable

**C++ definition:**

    template <class T>
    concept bool Movable() {
        return MoveConstructible<T>() &&
            Assignable<T&, T&&>() &&
            Swappable<T&>();
    }


##### Copyable

**C++ definition:**

    template <class T>
    concept bool Copyable() {
        return CopyConstructible<T>() &&
            Movable<T>() &&
            Assignable<T&, const T&>();
    }


##### Boolean

The `Boolean` concept describes the requirements on a type that is used in Boolean contexts.

**C++ definition:**

    template <class B>
    concept bool Boolean() {
        return MoveConstructible<B>() &&
            requires(const B b1, const B b2, const bool a) {
                bool(b1);
                { b1 } -> bool;
                bool(!b1);
                { !b1 } -> bool;
                { b1 && b2 } -> Same<bool>;
                { b1 && a } -> Same<bool>;
                { a && b1 } -> Same<bool>;
                { b1 || b2 } -> Same<bool>;
                { b1 || a } -> Same<bool>;
                { a || b1 } -> Same<bool>;
                { b1 == b2 } -> bool;
                { b1 != b2 } -> bool;
                { b1 == a } -> bool;
                { a == b1 } -> bool;
                { b1 != a } -> bool;
                { a != b1 } -> bool;
            };
    }


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

Let `t` and `u` be objects of types `T` and `U`. `WeaklyEqualityComparable<T, U>()` is satisfied iff:

- `t == u`, `u == t`, `t != u`, and `u != t` have the same domain.
- `bool(u == t) == bool(t == u)`.
- `bool(t != u) == !bool(t == u)`.
- `bool(u != t) == bool(t != u)`.

**C++ definition:**

    template <class T, class U>
    concept bool WeaklyEqualityComparable() {
        return requires(const T t, const U u) {
            { t == u } -> Boolean;
            { u == t } -> Boolean;
            { t != u } -> Boolean;
            { u != t } -> Boolean;
        };
    }
    
    template <class T>
    concept bool EqualityComparable() {
        return WeaklyEqualityComparable<T, T>();
    }
    
    template <class T, class U>
    concept bool EqualityComparable() {
        return Common<T, U>() &&
            EqualityComparable<T>() &&
            EqualityComparable<U>() &&
            EqualityComparable<std::common_type_t<T, U>>() &&
            WeaklyEqualityComparable<T, U>();
    }

The distinction between `EqualityComparable<T, U>()` and `WeaklyEqualityComparable<T, U>()` is purely semantic.

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

**C++ definition:**

    template <class T>
    concept bool Semiregular() {
        return Copyable<T>() &&
        DefaultConstructible<T>();
    }


##### Regular

The `Regular` concept is the union of the `Semiregular` and `EqualityComparable` concepts. [Stroustrup12](#Stroustrup12) §3.3

**C++ definition:**

    template <class T>
    concept bool Regular() {
        return Semiregular<T>() &&
        EqualityComparable<T>();
    }


##### StrictTotallyOrdered

The `StrictTotallyOrdered` concept requires The `EqualityComparable` concept plus the the inequality operators `<`, `>`, `<=`, and `>=`, with the following requirements: [Stroustrup12](#Stroustrup12) §3.3

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

**C++ definition:**

    template <class T>
    concept bool StrictTotallyOrdered() {
        return EqualityComparable<T>() &&
        requires (const T a, const T b) {
            { a < b } -> Boolean;
            { a > b } -> Boolean;
            { a <= b } -> Boolean;
            { a >= b } -> Boolean;
        };
    }
    
    template <class T, class U>
    concept bool StrictTotallyOrdered() {
        return Common<T, U>() &&
            StrictTotallyOrdered<T>() &&
            StrictTotallyOrdered<U>() &&
            StrictTotallyOrdered<std::common_type_t<T, U>>() &&
            EqualityComparable<T, U>() &&
            requires (const T t, const U u) {
                { t < u } -> Boolean;
                { t > u } -> Boolean;
                { t <= u } -> Boolean;
                { t >= u } -> Boolean;
                { u < t } -> Boolean;
                { u > t } -> Boolean;
                { u <= t } -> Boolean;
                { u >= t } -> Boolean;
            };
    }

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

Not all arguments will be well-formed for a given type. For example, `NaN` is not a well-formed
floating point value, and many types’ moved-from states are not well-formed. This does not mean that the type does not satisfy `StrictTotallyOrdered`.


### Callable concepts

Callable concepts describe requirements on function types. [Stroustrup12](#Stroustrup12) §3.4


##### Callable

The `Callable` concept describes a type whose objects can be called over a (possibly empty) sequence of arguments. `Callable`s are not `Semiregular` types; they may not exhibit the full range of capabilities as built-in value types. Minimally, they can be expected to be copy- and move-constructible but not copy- and move-assignable:

    Object:
        requirement: T* == &a (an object of this type can have its address taken and the result is a pointer to T)
        axiom: &a == addressof(a) (the result is an address to a)
        axiom: eq(a, a) (non-volatility)
    
    Destructible:
        requirement: a.~T() (destruction)
        requirement: noexcept(a.~T()) (noexcept)
    
    Initialization:
        requirement: T{a} (copy construction)
        axiom: eq(T{a}, a) (copy construction semantics)
        axiom: eq(a, b) => eq(T{move(a)}, b) (move construction semantics)
    
    Allocation:
        requirement: T* == { new T }
        requirement: delete new T
    
    Callable:
        requirement: ResultType<F f, Args args...> == f(args...) (callable with arguments)
        axiom: not_equality_preserving(f(args)) (not required to preserve equality)

**C++ definition:**

    template <class F, class...Args>
    concept bool Callable() {
        return CopyConstructible<F>() &&
            requires (F f, Args&&...args) {
                invoke(f, std::forward<Args>(args)...); // not required to be equality preserving
            };
    }

Since the invoke function call expression is not required to be equality-preserving, a function that generates random numbers may satisfy `Callable`.


##### RegularCallable

The `RegularCallable` concept describes a `Callable` that is equality-preserving, i.e. they have predictable and reasonable side-effects:

    Callable<F, Args...>
    axiom: equality_preserving(f(args))

**C++ definition:**

    template <class F, class...Args>
    concept bool RegularCallable() {
        return Callable<F, Args...>();
    }

Since the invoke function call expression is required to be equality-preserving, a function that generates random numbers does not satisfy `RegularCallable`-


##### Predicate

The `Predicate` concept describes a `RegularCallable` whose return type is `ConvertibleTo` bool:

    RegularCallable<P, Args...>
    ConvertibleTo<ResultType<P, Args...>, bool>

**C++ definition:**

    template <class F, class...Args>
    concept bool Predicate() {
        return RegularCallable<F, Args...>() &&
            Boolean<std::result_of_t<F&(Args...)>>();
    }


##### Relation

The `Relation` concept describes a binary `Predicate`.

The STL is concerned with two types of relations: *equivalence relations*, which generalize equality, and *strict weak orderings*, which generalize total orderings. In C++ these properties can only be defined for particular objects, not types, since there can be many functions with type `bool(T, T)` that are neither equivalence relations nor strict weak orderings.

    Relation<R, T1>
    Relation<R, T2>
    Common<T1, T2>
    Relation<R, CommonType<T1, T2>>
    requirement: bool r(t1, t2)
    requirement: bool r(t2, t1)
    axiom:
        r(t1, t2) <=> r(C{t1}, C{t2})
        r(t2, t1) <=> r(C{t2}, C{t1})

In summary, a `Relation` can be defined on different types if:

- `T1` and `T2` share a common type `C`
- The relation `R` is defined for all combinations of those types
- Any invocation of `r` on any combinations of types `T1`, `T2`, and `C` is equivalent to an invocation `r(C{t1}, C{t2})`

**C++ definition:**

    template <class R, class T>
    concept bool Relation() {
        return Predicate<R, T, T>();
    }
    
    template <class R, class T, class U>
    concept bool Relation() {
        return Relation<R, T>() &&
            Relation<R, U>() &&
            Common<T, U>() &&
            Relation<R, std::common_type_t<T, U>>() &&
            Predicate<R, T, U>() &&
            Predicate<R, U, T>();
    }


##### StrictWeakOrder

A `Relation` satisfies `StrictWeakOrder` iff it imposes a *strict weak ordering* on its arguments.

    axiom: (strict weak order using <)
        !(a < a) (irreflexive)
        a < b => !(b < a) (assymetric)
        a < b && b < c => a < c (transitive)
        a < b => a < c || c < b (transitivity of incomparability)

**C++ definition:**

    template <class R, class T>
    concept bool StrictWeakOrder() {
        return Relation<R, T>();
    }
    
    template <class R, class T, class U>
    concept bool StrictWeakOrder() {
        return Relation<R, T, U>();
    }


#### Operations

The following function concepts are related to numeric operations. An operation is a `RegularCallable` with a homogenous domain whose result type is convertible to its domain type. [Stroustrup12](#Stroustrup12) §3.4.2


##### UnaryOperation

The `UnaryOperation` concept describes a `RegularCallable` with one argument and a result type that is `ConvertibleTo` the type of the argument.

**C++ definition:**

    template <class Op, class T>
    concept bool UnaryOperation() {
    return RegularCallable<Op, T>() &&
        ConvertibleTo<std::result_of_t<Op(T)>, T>();
    }


##### BinaryOperation

The `BinaryOperation` concept describes a `RegularCallable` with two arguments that share a `Common` type, and a result type that is `ConvertibleTo` the `Common` type of the argument.

**C++ definition:**

    template <typename Op, typename T>
    concept bool BinaryOperation() {
        return RegularCallable<Op, T, T>() &&
            ConvertibleTo<std::result_of_t<Op(T, T)>, T>();
    }
    
    template <class Op, class T, class U>
    concept bool BinaryOperation() {
        return BinaryOperation<Op, T>() &&
            BinaryOperation<Op, U>() &&
            Common<T, U>() &&
            BinaryOperation<Op, Common<T, U>>() &&
            requires (Op op, T a, U b) {
                { op(a, b) } -> std::common_type_t<T, U>();
                { op(b, a) } -> std::common_type_t<T, U>();
        };
    }


### Iterator concepts

Iterators are one of the fundamental abstractions of the STL. They generalize the notion of pointers. The iterator design here differs from the one in the C++ standard in that it has more concepts and doesn't include an explicit notation of output iterators. In part this is because of the differentiation between iterators that require equality comparison (those in bounded ranges) and those that do not (those in weak ranges). [Stroustrup12](#Stroustrup12) §3.5


#### Iterator properties

Iterator properties deal with reading values from and writing values to iterators. [Stroustrup12](#Stroustrup12) §3.5.1


##### Readable

The `Readable` concept defines the basic properties of *input iterators*; it states what it means for a type to be readable.

    requirement: const ValueType<I>& = *i (the type has a value that can be read by dereferencing)


##### Writable

The `Writable` concept describes a requirement for writing a value to a dereferenced iterator.

    requirement: *o = value
    axiom: Readable<Out> && Same<ValueType<Out>, T> => (is_valid(*o = value) => (*o = value, eq(*o, value)))


##### IndirectlyMovable

The `IndirectlyMovable` concept describes the move relationship between the values of an input iterator `I` and an output iterator `Out`. For an input iterator `in` and an output iterator `out`,`IndirectlyMovable` requires that `*out = move(*in)`.


##### IndirectlyCopyable

The `IndirectlyMovable` concept describes the copy relationship between the values of an input iterator `I` and an output iterator `Out`. For an input iterator `in` and an output iterator `out`, `IndirectlyCopyable` requires that `*out = *in`.


#### Incrementable types

All iterators have, as a basic trait, the property that they can be incremented. [Stroustrup12](#Stroustrup12) §3.5.2


##### WeaklyIncrementable

The `WeaklyIncrementable` concept describes a `Semiregular` type that can be pre- and post-incremented, and is used in weak input and output ranges where equality comparison and equality preservation are not required.

    Associated types:
        DistanceType<I>
        Integral<DistanceType<I>>
    
    Pre-increment:
        requires: I& == ++i
        axiom: (if valid, ++i moves i to the next element)
            not_equality_preserving(++i)
            is_valid(++i) => &++i == &i
    
    Post-increment:
        requires: i++
        axiom (if valid, i++ moves i to the next element)
            not_equality_preserving(i++)
            is_valid(++i) <=> is_valid(i++)


##### Incrementable

The `Incrementable` concept describes a `Regular` type that can be pre- and post-incremented, and where both operations are equality-preserving.

    Associated types:
        DistanceType<I>
        Integral<DistanceType<I>>

    Pre-increment:
        requires: I& == ++i
        axiom:
            is_valid(++i) => &++i == &i
            equality_preserving(++i)

    Post-increment:
        requires: I = i++
        axiom:
            equality_preserving(i++)
            is_valid(++i) => (i == j => i++ = j)
            is_valid(++i) => (i == j => (i++, i) == ++j)


#### Iterator abstractions

The following concept describe iterator abstractions. [Stroustrup12](#Stroustrup12) §3.5.3


##### WeakInputIterator

The `WeakInputIterator` concept describes a `WeaklyIncrementable` and `Readable` type. It is used in weak ranges, which do not require the iterators to be `EqualityComparable`.

    requires: IteratorCategory<I>
    requires: Derived<IteratorCategory<I>, weak_input_iterator_tag>
    Readable<decltype(i++)> (dereferencing of post-increment result)


##### InputIterator

The `InputIterator` concept describes a `WeakInputIterator` that is `EqualityComparable`. It does not require that increment is an equality-preserving operation, meaning that a range can only be traversed once.


##### ForwardIterator

The `ForwardIterator` concept describes an `InputIterator` that is `Incrementable`. Because its increment operation is equality-preserving, it permits multiple passes over a range.


##### BidirectionalIterator

The `BidirectionalIterator` concept describes a `ForwardIterator` that supports the decrement operation. It abstracts the traversal of doubly linked lists.

   
    Pre-decrement:
        requires: I& == --i
        axiom: (if valid, --i moves i to the previous element)
            is_valid(--i) => &--i == &i
    
    Post-decrement:
        requires: I == i--
        axiom:
            is_valid(++i) <=> (is_valid(--(++i)) && (i == j => --(++i) == j))
            is_valid(--i) <=> (is_valid(++(--i)) && (i == j => ++(--i) == j))
    
    Increment-decrement:
        axiom:
            is_valid(++i) => (is_valid(--(++i)) && (i == j => --(++i) == j))
            is_valid(--i) => (is_valid(++(--i)) && (i == j => ++(--i) == j))


##### RandomAccessIterator

The `RandomAccessIterator` concept describes a `BidirectionalIterator` that is also `StrictTotallyOrdered`. It allows constance advancement some number of steps in either direction. The distance between `RandomAccessIterator`s can also be computed in constant time by subtracting two values.

    Difference:
        requires: DifferenceType<I> == i - j
        SignedIntegral<DifferenceType>
        ConvertibleTo<DistanceType, DifferenceType>
        axiom:
            is_valid(distance(i, j)) <=> is_valid(i - j) && is_valid(j - i)
            is_valid(i – j) => (i – j) >= 0 => i – j == distance(i, j)
            is_valid(i – j) => (i – j) < 0 => i – j == –distance(i, j)
    
    Advance:
        Addition:
            requires: I& == i += n
            requires: I == i + n
            requires: I == n + i
            axiom:
                is_valid(advance(i, n) <=> is_valid(i += n)
                is_valid(i += n) => i += n == (advance(i, n), i)
                is_valid(i += n) => &(i += n) == &i
                is_valid(i += n) => i + n == (i += n)
            axiom: is_valid(i + n) => i + n == n + i (commutativity of pointer addition)
            axiom: is_valid(i + (n + n)) => i + (n + n) == (i + n) + n (associativity of pointer addition)
            axiom: (Peano-like pointer addition)
                i + 0 == i
                is_valid(i + n) => i + n == ++(i + (n – 1))
                is_valid(++i) => (i == j => ++i != j)
    
        Subtraction:
            requires: I& == i –= n
            requires: I == i – n
            axiom: is_valid(i += –n) <=> is_valid(i –= n)
            axiom: is_valid(i –= n) => (i –= n) == (i += –n)
            axiom: is_valid(i –= n) => &(i –= n) == &i
            axiom: is_valid(i –= n) => (i – n) == (i –= n)
    
        Subscript:
            requires: ValueType<I> = i[n]
            axiom: is_valid(i + n) && is_valid(*(i + n)) => i[n] == *(i + n)


### Rearrangements

Rearrangements group together iterator requirements of algorithm families. [Stroustrup12](#Stroustrup12) §3.6


##### Permutable

The `Permutable` concept describes a requirement for permuting or rearranging the elements of an iterator range. It requires a `Semiregular` value type and `IndirectlyMovable` `ForwardIterators`.


##### Mergeable

The `Mergeable` concept describes the requirements of algorithms that merge sorted sequences into an output sequence. It requires a `Permutable` iterator range and a `StrictTotallyOrdered` or a *strict weak order* `Relation` between value types.


##### Sortable

The `Sortable` concept describes the requirements of algorithms that permute sequences of iterators into an ordered sequence. It requires a `ForwardIterator` and a `StrictTotallyOrdered` or a *strict weak order* `Relation` between value types.


References
----------

<a name="Sutton13"></a>
[Sutton13]
"Concepts Lite: Constraining Templates with Predicates", ISO N3551
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3580.pdf

<a name="Stepanov09"></a>
[Stepanov09]
"Elements of Programming", ISBN 0-321-63537-X

<a name="Stroustrup12"></a>
[Stroustrup12]
"A concept design for the STL", ISO N3551
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3351.pdf

<a name="ISO15"></a>
[ISO15]
"Programming Languages — C++ Extensions for Concepts", ISO N4549
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4549.pdf

<a name="Niebler15"></a>
[Niebler15]
"Working Draft, C++ Extensions for Ranges", ISO N4569
http://open-std.org/JTC1/SC22/WG21/docs/papers/2016/n4569.pdf
