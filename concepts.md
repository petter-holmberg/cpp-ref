C++ concepts quick reference
============================

This document is a summary of proposed core C++ concepts and a collection of design guidelines for C++ classes and algorithms. It is intended to be used as a quick reference when designing new classes/algorithms or refactoring old ones.
For brevity and simplicity of use, it does not discuss at length the *whys* of the guidelines, but instead presents references to further reading about the subject.

The guidelines cover C++98, C++11, C++14, C++17, and C++20, with notes on differences between the versions.


Concept definitions
-------------------

In the C++20 draft, concepts are defined as compile-time boolean expressions.

**Example:**

    template <typename T>
    concept Any = true;

In the C++ concepts TS [Sutton17](#Sutton17), concepts may also be defined as boolean functions, which allows overloading. The keyword `bool` is also required in both forms.

**Example:**

    // Expression form
    template <typename T>
    concept bool Any = true;
    
    // Function form
    template <typename T>
    concept bool Any() { return true; }

A *requires-expression* can be used inside a concept to express syntactic requirements on template arguments. There are four types of requirements.

**Example:**

    // Simple requirement
    template <typename T>
    concept C = requires (T a, T b) {
        a + b; //  C<T> is true if a + b is a valid expression
    };
    
    template <typename T, typename T::type = 0> struct S;
    template <typename T> using Ref = T&;
    
    // Type requirements
    template <typename T>
    concept C = requires {
        typename T::inner; // required nested member name
        typename S<T>;     // required class template specialization
        typename Ref<T>;   // required alias template substitution, fails if T is void
    };
    
    // Compound requirements
    template <typename T>
    concept C2 = requires (T x) {
        {*x} -> typename T::inner; // requires *x is a valid expression and is implicitly convertible to typename T::inner
        {g(x)} noexcept; // requires g(x) is a valid expression and that g(x) is non-throwing
    };
    
    template <typename U>
    concept C = sizeof(U) == 1;
    
    // Nested requirement
    template <typename T>
    concept D = requires (T t) {
        requires C<decltype(+t)>; // requires sizeof(decltype(+t)) == 1
    };


Template constraints
--------------------

In the C++20 draft, template parameter types can be constrained to satisfy concepts. In the C++ concepts TS [Sutton17](#Sutton17), there are four syntactic forms available. Only the most general form is approved in C++20 [Sutton13](#Sutton13) §2

**Example:**

    // Most general form with requires clause (approved in C++20 draft)
    template <typename C>
        requires Sortable<C>
    void sort(C& container);
    
    // Most general form with requires clause, alternative
    template <typename C>
    void sort(C& container) requires Sortable<C>;
    
    // Shorthand form, replacing typename
    template <Sortable C>
    void sort(C& container);
    
    // Abberviated form, replacing template declaration
    void sort(Sortable& container);
    
    // Template intruduction syntax
    Sortable{C}
    void sort(C& container);

In the C++ concepts TS [Sutton17](#Sutton17) the `auto` keyword can also be used to describe a completely unconstrained parameter. Note that the semantics differ from when a named concept is defined. [Stroustrup17](#Stroustrup17)
§7.1

**Example:**

    concept bool Any = true; // Every type is an any (Concepts TS syntax)
    
    void f(Any x, Any y); // x and y must take the same type
    void g(auto x, auto y); // x and y can be of different type

Aliasing the `requires` keyword with a variadic macro, or using `#define` to alias the `typename` keyword, provides easy ways to declare template constraints in anticipation of compiler support for the first two forms, treating them as documentation. [Stroustrup17](#Stroustrup17) §5.6

**Example:**

    #ifndef CONCEPTS_AVAILABLE
    #ifdef ALTERNATIVE_1
    #define requires(...)
    #elseif ALTERNATIVE_2
    #define requires //
    #endif
    #endif
    
    #ifndef CONCEPTS_AVAILABLE
    #ifdef PRE_CPP11
    #define Sortable typename
    #elseif POST_CPP11
    #define Sortable auto
    #endif
    #endif

To get a rough equivalent of concept-based overloading, `enable_if` can be used.

Constraints can also be used with member functions.

**Example:**

    template <Object T, Allocator A>
    class vector {
        requires Equality_comparable<T>
            bool operator==(const vector& x) const;
    
        requires Totally_ordered<T>
            bool operator<(const vector& x) const;
    };

Multi-type constraints can easily be expressed using explicit form with conjunctions or shorthand constraint form with implicit conjunction: [Sutton13](#Sutton13) §2.1.1

**Example:**

    // Explicit form
    template <typename S, typename T>
        requires Sequence<S> && Equality_comparable<T, Value_type<S>>
    Iterator_type<S> find(S&& sequence, const T& value);
    
    // Shorthand form
    template <Sequence S, Equality_comparable<Value_Type<S>> T>
    Iterator_type<S> find(S&& sequence, const T& value);


### Matching functions to concepts

Roughly speaking, given the Concepts TS, C++ considers one function to be "better" than another using the following rules: [Overload16](#Overload16)

1. Functions requiring fewer or "cheaper" conversions of the arguments are better than those requiring more or costlier conversions.
2. Non-template functions are better than function template specializations.
3. One function template specialization is better than another if its parameter types are more specialized. For example, `T*` is more specialized than `T`, and so is `vector<T>`, but `T*` is not more specialized than `vector<T>`, nor is the opposite true.
4. If two functions cannot be ranked because they have equivalent conversions or are function template specializations with equivalent parameter types, then the better one is more constrained. Also, unconstrained functions are the least constrained.

In other words, constraints act as a tie-breaker for the usual overloading rules in C++. The ordering of constraints (more constrained) is essentially determined by the comparing sets of requirements for each template to determine if one is a strict superset of another.

If the difference between two concepts is purely semantic, overload resolution may still be ambiguous. This is the case for example with the standard `Input_iterator` and `Forward_iterator` concepts. To express that `Forward_iterator` is a more constrained concept, traditional tag dispatch (associating a tag class (an empty class in an inheritance hierarchy) with iterator type for the purpose of selecting appropriate overloads. That associated type is its `iterator_category`.

**Example:**

    template <typename I>
    concept Input_iterator =
        Regular<I> &&
        requires (I i) {
            typename value_type_t<I>;
            { *i } -> value_type_t<I> const&;
            { ++i } -> I&;
    };
    
    // Ambiguous
    template <typename I>
    concept Forward_iterator = Input_iterator<I>;
    
    // More constrained
    template <typename I>
    concept Forward_iterator =
        Input_iterator<I> &&
        requires {
            typename iterator_category_t<I>;
            requires Derived_from<I, forward_iterator_tag>;
        };

Alternative solutions include using an associated type trait, variable template, or even an extra operation.

**Example:**

    template <typename T>
    constexpr bool is_forward_iterator_v = false;
    
    template <typename T>
    constexpr bool is_forward_iterator_v<T*> = true;
    
    template <typename I>
    concept Forward_iterator = Input_iterator<I> && is_forward_iterator_v<I*>;

Require only essential properties in a template's concepts. Requiring every operation used to be listed among the requirements makes the interface unstable. [Sutter17](#Sutter17) §T.41


### Matching types to concepts

To match types to concepts, `static_assert` that the desired concept matches. [Stroustrup17](#Stroustrup17) §5.5

**Example:**

    class My_number { /* ... */ };
    static_assert(Number<My_number>);
    static_assert(Group<My_number>);
    static_assert(Someone_elses_number<My_number>);
    
    class My_container { /* ... */ };
    static_assert(Random_access_iterator<My_container::iterator>);


Concept description mechanisms
------------------------------

Several mechanisms dealing with types are needed to describe concepts: [Stepanov09](#Stepanov09) §1.7

- Type attributes
- Type functions
- Type constructors


### Type attributes

A *type attribute* is a mapping from a type to a value describing some characteristic of the type. Examples of type attributes in C++ are the built-in type attribute `sizeof(T)`, the alignment of an object of a type, and the number of members in a `struct`.


### Type functions

A *type function* is a mapping from a type to an affiliated type. Examples of type functions is a function that given a `*T`, returns the type `T`. In C++, type functions are implemented by *traits classes*, specialized for each particular type where necessary.

**Example:**

    template <typename T>
    struct value_type {
        typedef T type;
    };
    
    // Convenience notation using alias templates (C++11)
    template <typename T>
    using ValueType = typename value_type<T>::type;
    
    // Convenience notation using macro (C++98)
    #define ValueType(T) typename value_type<T>::type


### Type constructors

A *type constructor* is a mechanism for creating a new type from one or more existing types. Examples of type constructors in C++ are the `*` (pointer) operator, `struct`, which is a n-ary type constructor, and `std::pair`, which returns a structure of two members.


Concept design guidelines
-------------------------

Ideally, a concept represents a fundamental concept in some domain, hence the name "concept". A concept has semantics; it means something; it is not just a set of unrelated operations and types. Avoid "concepts" without meaningful semantics. [Stroustrup17](#Stroustrup17) §5, [Sutter17](#Sutter17) §T.20

Specify axioms for concepts. Expressing its semantics in an informal, semi-formal, or formal way makes the concept comprehensible to readers and the effort to express it can catch conceptual errors. In general, axioms are not provable, and when they are the proof is often beyond the capability of a compiler. An axiom may not be general, but the template writer may assume that it holds for all inputs actually used (similar to a precondition). [Sutter17](#Sutter17) §T.22

**Example:**

    template <typename T>
    // The operators +, -, *, and / for a number are assumed to follow the usual mathematical rules
    // axiom(T a, T b) { a + b == b + a; a - a == 0; a * (b + c) == a * b + a * c; /*...*/ }
    concept Number = requires (T a, T b) {
        {a + b} -> T;   // the result of a + b is convertible to T
        {a - b} -> T;
        {a * b} -> T;
        {a / b} -> T;
    }

Avoid "single property concepts". People sometimes get into trouble defining something like this: [Stroustrup17](#Stroustrup17) §5.1

    template <typename T>
    concept Addable = requires (T a, T b) { { a+b } -> T; };

`Addable` is not a suitable concept for general use, it does not represent a fundamental user-level concept. If `Addable`, why not `Subtractable`? (`std::string` is not `Subtractable`, but `int*` is).

The first step to design a good concept is to consider what is a complete (necessary and sufficient) set of properties (operations, types, etc.) to match the domain concept, taking into account the semantics of that domain concept. [Stroustrup17](#Stroustrup17) §5.2, [Sutter17](#Sutter17) §T.21

A standard generic programming technique has been to specify the requirements of an algorithm to be the absolute minimum. This is not what we do to design the most useful concepts. That kind of design leads to programs where:
- Every algorithm has its own requirements (a variety that we cannot easily remember).
- Every type must be designed to match an unspecified and changing set of requirements.
- When we improve the implementation of an algorithm, we must change its requirements (part of its interface), potentially breaking code.

In this direction lies chaos. Thus, the ideal is not "minimal requirements" but "requirements expressed in terms of fundamental and complete concepts". This puts a burden on the designers of types (to match concepts), but that leads to better types and to more flexible code.
To design good concepts and to use concepts well, remember that an implementation isn't a specification - someday, someone is likely to want to improve the implementation and ideally that is done without affecting the interface. Often, we cannot change an interface because doing so would break user code.
To write maintainable and widely usable code we aim for semantic coherence, rather than minimalism for each concept and algorithm in isolation. [Stroustrup17](#Stroustrup17) §5.3

Concepts that are too simple for general use and/or lack a clear semantics can also be used as building blocks for more complete concepts.
Sometimes, we call such overly simple or incomplete concepts "constraints" to distinguish them from the "real concepts". [Stroustrup17](#Stroustrup17) §5.4

In the rare case where two concepts are identical syntactically but differ semantically (e.g., `InputIterator` and `ForwardIterator` are almost indistinguishable), either disambiguate them by adding an operation or a member type or use a traits class in the implementation of operations on them. [Stroustrup17](#Stroustrup17) §8.5, [Sutter17](#Sutter17) §T.24

Avoid complementary constraints. Functions with complementary requirements expressed using negation are brittle. [Sutter17](#Sutter17) §T.25

**Example:**

    template <typename T>
        requires !C<T> // bad
    void f();
    
    template <typename T>
        requires C<T>
    void f();
    
    
    template <typename T> // general template (good)
    void f();
    
    template <typename T> // specialization by concept (good)
        requires C<T>
    void f();


Concept definitions
-------------------

The concepts presented in this section are not yet standardized and there may be substantial changes before they are finalized. Concept definitions are taken from the Origin libraries and the Ranges TS draft ([Niebler17](#Niebler17)), with modifications to match the approved C++20 syntax. Many of the concepts are described in greater detail in [Stroustrup12](#Stroustrup12).

The concept definitions consist of *requirements*, which express syntax and must be checked by the compiler, and *axioms*, which express semantics and must not be checked by the compiler.


### Language concepts

Language concepts are intrinsic to the C++ programming language. [Stroustrup12](#Stroustrup12) §3.2, [Niebler17](#Niebler17) §19.2


##### Assignable

**C++ Defintion:**

    // (Origin)
    template <typename T, typename U>
    concept Assignable = std::is_assignable<T, U>::value;
    
    template <typename T>
    concept Move_assignable = Assignable<T&, T&&>;
    
    template <typename T>
    concept Copy_assignable = Move_assignable<T> && Assignable<T&, const T&>;
    
    // (Ranges TS)
    template <class T, class U>
    concept Assignable =
        CommonReference<const T&, const U&> &&
        requires (T&& t, U&& u) {
            { std::forward<T>(t) = std::forward<U>(u) } -> Same<T&>;
        };


##### Swappable

**C++ Defintion:**

    template <class T>
    concept Swappable =
        requires (T& a, T& b) {
            ranges::swap(a, b);
    };
    
    template <class T, class U>
    concept SwappableWith =
        CommonReference<
            const remove_reference_t<T>&,
            const remove_reference_t<U>&> &&
        requires (T&& t, U&& u) {
            ranges::swap(std::forward<T>(t), std::forward<T>(t));
            ranges::swap(std::forward<U>(u), std::forward<U>(u));
            ranges::swap(std::forward<T>(t), std::forward<U>(u));
            ranges::swap(std::forward<U>(u), std::forward<T>(t));
        };

#### Type classifications

The following concepts classify fundamental types. Their semantics are fully described in the C++ standard, using type traits.


##### Integral

The `Integral` concept is defined by the type trait `is_integral<T>`.

**C++ definition:**

    template <class T>
    concept Integral = std::is_integral<T>::value;

**Example:**

    Integral<int>          // is true
    Integral<unsigned int> // is true
    Integral<double>       // is false


##### SignedIntegral

The `SignedIntegral` concept adds the `is_signed<T>::value` requirement.

**C++ definition:**

    template <class T>
    concept SignedIntegral = Integral<T> && std::is_signed<T>::value;

**Example:**

    SignedIntegral<int>          // is true
    SignedIntegral<signed int>   // is true
    SignedIntegral<unsigned int> // is false
    SignedIntegral<double>       // is true


##### UnsignedIntegral

The `UnsignedIntegral` concept is defined in terms of the `Integral` and `SignedIntegral` concept.

**C++ definition:**

    template <class T>
    concept UnsignedIntegral = Integral<T> && !SignedIntegral<T>;

**Example:**

    UnsignedIntegral<int>          // is false
    UnsignedIntegral<signed int>   // is false
    UnsignedIntegral<unsigned int> // is true
    UnsignedIntegral<double>       // is false


#### Type relations

The following concpets describe relationships between types.


##### Same

The `Same` concept is built-in to the language, described by the standard library type `is_same`. For two types `T` and `U`, `Same<T, U>` is true iff `T` and `U` denote exactly the same type after elimination of aliases. For the purposes of constraint checking, `Same<T, U>` implies `Same<U, T>`.

**C++ definition:**

    template <class T, class U>
    concept Same = std::is_same<T, U>::value;

**Example:**

    using Pi = int*;
    using I = int;
    Same<Pi, I> // is false
    Same<Pi, I*> // is true


##### DerivedFrom

The `DerivedFrom` concept returns true if one type is derived from another.

**C++ definition:**

    template <class T, class U>
    concept DerivedFrom =
        is_base_of<U, T>::value &&
        is_convertible<remove_cv_t<T>*, remove_cv_t<U>*>::value;

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
    concept ConvertibleTo =
        is_convertible<From, To>::value &&
        requires (From (&f)()) {
            static_cast<To>(f());
        };

**Example:**

    ConvertibleTo<double, int>             // is true: int i = 2.7; (ok)
    ConvertibleTo<double, complex<double>> // is true: complex<double> d = 3.14; (ok)
    ConvertibleTo<complex<double>, double> // is false: double d = complex<double>2,3 (error)
    ConvertibleTo<int, int>                // is true: a type is convertible to itself
    ConvertibleTo<Derived, Base>           // is true: derived types can be converted to base types


##### CommonReference

The `CommonReference` concept expresses that for two types T and U, if `common_reference_t<T, U>` is well-formed and denotes a type `C` such that both `ConvertibleTo<T, C>` and `ConvertibleTo<U, C>` are satisfied, then `T` and `U` share a common reference type, `C`. Note: `C` could be the same as `T`, or `U`, or it could be a different type. `C` may be a reference type. `C` need not be unique.

**C++ definition:**

    template <class T, class U>
    concept CommonReference =
        Same<common_reference_t<T, U>, common_reference_t<U, T>> &&
        ConvertibleTo<T, common_reference_t<T, U>> &&
        ConvertibleTo<U, common_reference_t<T, U>>;

##### Common

The `Common` concept expresses that two types `T` and `U` can both be unambiguously converted to a third type `C`, i.e. they share a *common type*. `C` can be the same type as `T` and `U` or it can be a different type:

    requirement: CommonType<T, U> (alias for the standard type trait common_type<T, U>::type)
    axiom: eq(t1, t2) <=> eq(C{t1}, C{t2})
    axiom: eq(u1, u2) <=> eq(C{u1}, C{u2})

**C++ definition:**

    template <class T, class U>
    concept Common =
        Same<common_type_t<T, U>, common_type_t<U, T>> &&
        ConvertibleTo<T, common_type_t<T, U>> &&
        ConvertibleTo<U, common_type_t<T, U>> &&
        CommonReference<
            add_lvalue_reference_t<const T>,
            add_lvalue_reference_t<const U>> &&
        CommonReference<
            add_lvalue_reference_t<common_type_t<T, U>>,
            common_reference_t<
                add_lvalue_reference_t<const T>,
                add_lvalue_reference_t<const U>>>;

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

    // (Origin)
    template <typename T>
    concept Destructible =
    requires (T* t) {
        {t->~T(); }; // Note: this doesn't work with array types
    };
    
    // (Ranges TS)
    template <class T>
    concept Destructible = std::is_nothrow_destructible<T>::value;


##### Constructible

**C++ definition:**

    template <class T, class... Args>
    concept Constructible = Destructible<T> && std::is_constructible<T, Args...>::value;


##### DefaultConstructible

**C++ definition:**

    template <class T>
    concept DefaultConstructible = Constructible<T>;


##### MoveConstructible

**C++ definition:**

    // (Origin)
    template <typename T>
    concept Move_constructible = Constructible<T, T&&>;
    
    // (Ranges TS)
    template <class T>
    concept MoveConstructible = Constructible<T, T> && ConvertibleTo<T, T>;


##### CopyConstructible

**C++ definition:**

    // (Origin)
    template <typename T>
    concept Copy_constructible =
        Move_constructible<T> && Constructible<T, const T&>;
    
    // (Ranges TS)
    template <class T>
    concept CopyConstructible =
        MoveConstructible<T> &&
        Constructible<T, T&> && ConvertibleTo<T&, T> &&
        Constructible<T, const T&> && ConvertibleTo<const T&, T> &&
        Constructible<T, const T&&> && ConvertibleTo<const T, T>;


##### Movable

**C++ definition:**

    // (Origin)
    template <typename T>
    concept Movable = Move_constructible<T> && Move_assignable<T>;
    
    // (Ranges TS)
    template <class T>
    concept Movable =
        std::is_object<T>::value &&
        MoveConstructible<T> &&
        Assignable<T&, T> &&
        Swappable<T>;


##### Copyable

**C++ definition:**

    // (Origin)
    template <typename T>
    concept Copyable = Copy_constructible<T> && Copy_assignable<T>;
    
    // (Ranges TS)
    template <class T>
    concept Copyable =
        CopyConstructible<T> &&
        Movable<T> &&
        Assignable<T&, const T&>;


##### Boolean

The `Boolean` concept describes the requirements on a type that is used in Boolean contexts.

**C++ definition:**

    // (Origin)
    concept Conditional =
        requires (T t) {
            (bool)t; // Explicitly convertible
        };
    
    concept Boolean =
        Conditional<T> &&
        requires (T a, T b) {
            { !a } -> Same<bool>;
            { a && b } -> Same<bool>;
            { a || b } -> Same<bool>;
        };
    
    // (Ranges TS)
    template <class B>
    concept Boolean =
    Movable<decay_t<B>> &&
    requires (const remove_reference_t<B>& b1,
    const remove_reference_t<B>& b2, const bool a) {
        { b1 } -> ConvertibleTo<bool>&&;
        { !b1 } -> ConvertibleTo<bool>&&;
        { b1 && a } -> Same<bool>&&;
        { b1 || a } -> Same<bool>&&;
        { b1 && b2 } -> Same<bool>&&;
        { a && b2  } -> Same<bool>&&;
        { b1 || b2 } -> Same<bool>&&;
        { a || b2  } -> Same<bool>&&;
        { b1 == b2 } -> ConvertibleTo<bool>&&;
        { b1 == a  } -> ConvertibleTo<bool>&&;
        { a == b2  } -> ConvertibleTo<bool>&&;
        { b1 != b2 } -> ConvertibleTo<bool>&&;
        { b1 != a  } -> ConvertibleTo<bool>&&;
        { a != b2  } -> ConvertibleTo<bool>&&;
    };

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

- Two tuples are equal iff they have the same number of sub-objects and all of their sub-objects compare equal using `==`.
- Two vectors are equal iff they have the same size and contain the same elements as if compared using the `equal` algorithm (which compares using `==`.
- Two complex numbers are equal iff they have equal real and imaginary components.

Let `t` and `u` be objects of types `T` and `U`. `WeaklyEqualityComparable<T, U>` is satisfied iff:

- `t == u`, `u == t`, `t != u`, and `u != t` have the same domain.
- `bool(u == t) == bool(t == u)`.
- `bool(t != u) == !bool(t == u)`.
- `bool(u != t) == bool(t != u)`.

**C++ definition:**

    // (Origin)
    template <typename T>
    concept Equality_comparable =
        requires (T a, T b) {
            { a == b } -> Boolean;
            { a != b } -> Boolean;
        };
    }
    
    // (Ranges TS)
    template <class T, class U>
    concept WeaklyEqualityComparableWith =
        requires (const remove_reference_t<T>& t,
        const remove_reference_t<U>& u) {
            { t == u } -> Boolean&&;
            { t != u } -> Boolean&&;
            { u == t } -> Boolean&&;
            { u != t } -> Boolean&&;
        };
    
    template <class T>
    concept EqualityComparable = WeaklyEqualityComparableWith<T, T>;
    
    template <class T, class U>
    concept EqualityComparableWith =
        EqualityComparable<T> &&
        EqualityComparable<U> &&
        CommonReference<
            const remove_reference_t<T>&,
            const remove_reference_t<U>&> &&
        EqualityComparable<
            common_reference_t<
                const remove_reference_t<T>&,
                const remove_reference_t<U>&>> &&
        WeaklyEqualityComparableWith<T, U>;

The distinction between `EqualityComparableWith<T, U>` and `WeaklyEqualityComparableWith<T, U>` is purely semantic. The requirement that the expression a == b is equality preserving implies that == is reflexive, transitive, and symmetric.

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
- Volatile types (`volatile T`) are not `Semiregular` since their objects' states may by changed externally.
- Reference types types are not `Semiregular` because the syntax used for copy construction does not actually create a copy; it binds a reference to an
object.

**C++ definition:**

    template <class T>
    concept Semiregular = Copyable<T> && DefaultConstructible<T>;


##### Regular

The `Regular` concept is the union of the `Semiregular` and `EqualityComparable` concepts. [Stroustrup12](#Stroustrup12) §3.3

**C++ definition:**

    template <typename T>
    concept Regular = Semiregular<T> && Equality_comparable<T>;

##### StrictTotallyOrdered

The `StrictTotallyOrdered` concept requires The `EqualityComparable` concept plus the inequality operators `<`, `>`, `<=`, and `>=`, with the following requirements: [Stroustrup12](#Stroustrup12) §3.3

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

    // (Origin)
    template <typename T>
    concept Weakly_ordered =
        requires (T a, T b) {
            { a < b } -> Boolean;
            { a > b } -> Boolean;
            { a <= b } -> Boolean;
            { a >= b } -> Boolean;
        };
    
    template <typename T>
    concept Totally_ordered = Equality_comparable<T> && Weakly_ordered<T>;
    
    // (Origin)
    template <typename T>
    concept Ordered = Regular<T> && Totally_ordered<T>;
    
    // (Ranges TS)
    template <class T>
    concept StrictTotallyOrdered =
        EqualityComparable<T> &&
        requires (const remove_reference_t<T>& a, const remove_reference_t<T>& b) {
            { a < b }  -> Boolean&&;
            { a > b }  -> Boolean&&;
            { a <= b } -> Boolean&&;
            { a >= b } -> Boolean&&;
        };
    
    template <class T, class U>
    concept StrictTotallyOrderedWith =
        StrictTotallyOrdered<T> &&
        StrictTotallyOrdered<U> &&
        CommonReference<
            const remove_reference_t<T>&,
            const remove_reference_t<U>&> &&
        StrictTotallyOrdered<
            common_reference_t<
                const remove_reference_t<T>&,
                const remove_reference_t<U>&>> &&
        EqualityComparableWith<T, U> &&
        requires (const remove_reference_t<T>& t, const remove_reference_t<U>& u) {
            { t < u } -> Boolean&&;
            { t > u } -> Boolean&&;
            { t <= u } -> Boolean&&;
            { t >= u } -> Boolean&&;
            { u < t } -> Boolean&&;
            { u > t } -> Boolean&&;
            { u <= t } -> Boolean&&;
            { u >= t } -> Boolean&&;
        };

**Example implementation:**

    struct Rational {
        int numerator = 0;
        int denominator = 0;
    };
    
    inline bool operator==(const Rational& lhs, const Rational& rhs)
    {
        return lhs.numerator * rhs.denominator == lhs.denominator * rhs.numerator;
    }
    
    inline bool operator!=(const Rational& lhs, const Rational& rhs)
    {
        return !(lhs == rhs);
    }
    
    inline bool operator<(const Rational& lhs, const Rational& rhs)
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
floating point value, and many types' moved-from states are not well-formed. This does not mean that the type does not satisfy `StrictTotallyOrdered`.


### Callable concepts

Callable concepts describe requirements on function objects and their arguments. [Stroustrup12](#Stroustrup12) §3.4


##### Invocable

The `Invocable` concept describes a type whose objects can be called over a (possibly empty) sequence of arguments. `Invocable`s are not `Semiregular` types; they may not exhibit the full range of capabilities as built-in value types. Minimally, they can be expected to be copy- and move-constructible but not copy- and move-assignable:

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
    
    Invocable:
        requirement: ResultType<F f, Args args...> == f(args...) (callable with arguments)
        axiom: not_equality_preserving(f(args)) (not required to preserve equality)

**C++ definition:**

    template <class F, class... Args>
    concept Invocable =
        requires (F&& f, Args&&... args) {
            invoke(std::forward<F>(f), std::forward<Args>(args)...); // not required to be equality preserving
    };


##### RegularInvocable

The `RegularInvocable` concept describes a `Invocable` that is equality-preserving, i.e. they have predictable and reasonable side-effects:

    Invocable<F, Args...>
    axiom: equality_preserving(f(args))

**C++ definition:**

    template <class F, class... Args>
    concept RegularInvocable = Invocable<F, Args...>;

Since the invoke function call expression is required to be equality-preserving, a function that generates random numbers does not satisfy `RegularInvocable`-


##### Predicate

The `Predicate` concept describes a `RegularInvocable` whose return type is `ConvertibleTo` bool:

    RegularInvocable<P, Args...>
    ConvertibleTo<ResultType<P, Args...>, bool>

**C++ definition:**

    template <class F, class... Args>
    concept Predicate =
        RegularInvocable<F, Args...> &&
        Boolean<result_of_t<F&(Args...)>>;


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

    template <class R, class T, class U>
    concept Relation =
        Predicate<R, T, T> &&
        Predicate<R, U, U> &&
        CommonReference<
            const remove_reference_t<T>&,
            const remove_reference_t<U>&> &&
        Predicate<R,
            common_reference_t<
                const remove_reference_t<T>&,
                const remove_reference_t<U>&>,
            common_reference_t<
                const remove_reference_t<T>&,
                const remove_reference_t<U>&>> &&
        Predicate<R, T, U> &&
        Predicate<R, U, T>;


##### StrictWeakOrder

A `Relation` satisfies `StrictWeakOrder` iff it imposes a *strict weak ordering* on its arguments.

    axiom: (strict weak order using <)
        !(a < a) (irreflexive)
        a < b => !(b < a) (assymetric)
        a < b && b < c => a < c (transitive)
        a < b => a < c || c < b (transitivity of incomparability)

**C++ definition:**

    template <class R, class T, class U>
    concept StrictWeakOrder = Relation<R, T, U>;


##### Indirect invocables

The indirect invocable concepts are used to constrain those algorithms that accept invocable objects as arguments.

**C++ definition:**

    template <class F, class I>
    concept IndirectUnaryInvocable =
        Readable<I> &&
        CopyConstructible<F> &&
        Invocable<F&, value_type_t<I>&> &&
        Invocable<F&, reference_t<I>> &&
        Invocable<F&, iter_common_reference_t<I>> &&
        CommonReference<
            result_of_t<F&(value_type_t<I>&)>,
            result_of_t<F&(reference_t<I>&&)>>;
    
    template <class F, class I>
    concept IndirectRegularUnaryInvocable =
        Readable<I> &&
        CopyConstructible<F> &&
        RegularInvocable<F&, value_type_t<I>&> &&
        RegularInvocable<F&, reference_t<I>> &&
        RegularInvocable<F&, iter_common_reference_t<I>> &&
        CommonReference<
            result_of_t<F&(value_type_t<I>&)>,
            result_of_t<F&(reference_t<I>&&)>>;
    
    template <class F, class I>
    concept IndirectUnaryPredicate =
        Readable<I> &&
        CopyConstructible<F> &&
        Predicate<F&, value_type_t<I>&> &&
        Predicate<F&, reference_t<I>> &&
        Predicate<F&, iter_common_reference_t<I>>;
    
    template <class F, class I1, class I2 = I1>
    concept IndirectRelation =
        Readable<I1> &&
        Readable<I2> &&
        CopyConstructible<F> &&
        Relation<F&, value_type_t<I1>&, value_type_t<I2>&> &&
        Relation<F&, value_type_t<I1>&, reference_t<I2>> &&
        Relation<F&, reference_t<I1>, value_type_t<I2>&> &&
        Relation<F&, reference_t<I1>, reference_t<I2>> &&
        Relation<F&, iter_common_reference_t<I1>, iter_common_reference_t<I2>>;
    
    template <class F, class I1, class I2 = I1>
    concept IndirectStrictWeakOrder =
        Readable<I1> &&
        Readable<I2> &&
        CopyConstructible<F> &&
        StrictWeakOrder<F&, value_type_t<I1>&, value_type_t<I2>&> &&
        StrictWeakOrder<F&, value_type_t<I1>&, reference_t<I2>> &&
        StrictWeakOrder<F&, reference_t<I1>, value_type_t<I2>&> &&
        StrictWeakOrder<F&, reference_t<I1>, reference_t<I2>> &&
        StrictWeakOrder<F&, iter_common_reference_t<I1>, iter_common_reference_t<I2>>;


#### Operations

The following function concepts are related to numeric operations. An operation is a `RegularInvocable` with a homogenous domain whose result type is convertible to its domain type. [Stroustrup12](#Stroustrup12) §3.4.2


##### UnaryOperation

The `UnaryOperation` concept describes a `RegularInvocable` with one argument and a result type that is `ConvertibleTo` the type of the argument.


##### BinaryOperation

The `BinaryOperation` concept describes a `RegularInvocable` with two arguments that share a `Common` type, and a result type that is `ConvertibleTo` the `Common` type of the argument.


### Iterator concepts

Iterators are one of the fundamental abstractions of the STL. They generalize the notion of pointers. The iterator design here differs from the one in the C++ standard in that it has more concepts and doesn't include an explicit notation of output iterators. In part this is because of the differentiation between iterators that require equality comparison (those in bounded ranges) and those that do not (those in weak ranges). [Stroustrup12](#Stroustrup12) §3.5


#### Iterator properties

Iterator properties deal with reading values from and writing values to iterators. [Stroustrup12](#Stroustrup12) §3.5.1


##### Readable

defines the basic properties of *input iterators*; it states what it means for a type to be readable. The `Readable` concept is satisfied by types that are readable by applying operator*, including pointers, smart pointers, and iterators.

    requirement: const ValueType<I>& = *i (the type has a value that can be read by dereferencing)

**C++ definition:**

    template <class In>
    concept Readable =
        requires {
            typename value_type_t<In>;
            typename reference_t<In>;
            typename rvalue_reference_t<In>;
        } &&
        CommonReference<reference_t<In>&&, value_type_t<In>&> &&
        CommonReference<reference_t<In>&&, rvalue_reference_t<In>&&> &&
        CommonReference<rvalue_reference_t<In>&&, const value_type_t<In>&>;


##### Writable

The `Writable` concept specifies the requirements for writing a value into an iterator's referenced object.

    requirement: *o = value
    axiom: Readable<Out> && Same<ValueType<Out>, T> => (is_valid(*o = value) => (*o = value, eq(*o, value)))

**C++ definition:**

    template <class Out, class T>
    concept Writable =
        requires (Out&& o, T&& t) {
            *o = std::forward<T>(t); // not required to be equality preserving
            *std::forward<Out>(o) = std::forward<T>(t); // not required to be equality preserving
            const_cast<const reference_t<Out>&&>(*o) =
                std::forward<T>(t); // not required to be equality preserving
            const_cast<const reference_t<Out>&&>(*std::forward<Out>(o)) =
            std::forward<T>(t); // not required to be equality preserving
        };


##### IndirectlyMovable

The `IndirectlyMovable` concept specifies the relationship between a `Readable` type and a `Writable` type between which values may be moved. For an input iterator `in` and an output iterator `out`,`IndirectlyMovable` requires that `*out = move(*in)`.

The `IndirectlyMovableStorable` concept augments `IndirectlyMovable` with additional requirements enabling the transfer to be performed through an intermediate object of the `Readable` type's value type.

**C++ definition:**

    template <class In, class Out>
    concept IndirectlyMovable =
        Readable<In> &&
        Writable<Out, rvalue_reference_t<In>>;
    
    template <class In, class Out>
    concept IndirectlyMovableStorable =
        IndirectlyMovable<In, Out> &&
        Writable<Out, value_type_t<In>> &&
        Movable<value_type_t<In>> &&
        Constructible<value_type_t<In>, rvalue_reference_t<In>> &&
        Assignable<value_type_t<In>&, rvalue_reference_t<In>>


##### IndirectlyCopyable

The `IndirectlyMovable` concept specifies the relationship between a `Readable` type and a `Writable` type between which values may be copied.

The `IndirectlyCopyableStorable` concept augments `IndirectlyCopyable` with additional requirements enabling the transfer to be performed through an intermediate object of the `Readable` type's value type. It also requires the capability to make copies of values.

**C++ definition:**

    template <class In, class Out>
    concept IndirectlyCopyable =
        Readable<In> &&
        Writable<Out, reference_t<In>>;
    
    template <class In, class Out>
    concept IndirectlyCopyableStorable =
        IndirectlyCopyable<In, Out> &&
        Writable<Out, const value_type_t<In>&> &&
        Copyable<value_type_t<In>> &&
        Constructible<value_type_t<In>, reference_t<In>> &&
        Assignable<value_type_t<In>&, reference_t<In>>;


##### IndirectlySwappable

The `IndirectlySwappable` concept specifies a swappable relationship between the values referenced by two `Readable` types.

Given an object `i1` of type `I1` and an object `i2` of type `I2`, `IndirectlySwappable<I1, I2>` is satisfied if after `ranges::iter_swap(i1, i2)`, the value of `*i1` is equal to the value of `*i2` before the call, and vice
versa.

**C++ definition:**

    template <class I1, class I2 = I1>
    concept IndirectlySwappable =
        Readable<I1> &&
        Readable<I2> &&
        requires (I1&& i1, I2&& i2) {
            ranges::iter_swap(std::forward<I1>(i1), std::forward<I2>(i2));
            ranges::iter_swap(std::forward<I2>(i2), std::forward<I1>(i1));
            ranges::iter_swap(std::forward<I1>(i1), std::forward<I1>(i1));
            ranges::iter_swap(std::forward<I2>(i2), std::forward<I2>(i2));
        };


##### IndirectlyComparable

The `IndirectlyComparable` concept specifies the common requirements of algorithms that compare values from two different sequences.

**C++ definition:**

    template <class I1, class I2, class R = equal_to<>, class P1 = identity, class P2 = identity>
    concept IndirectlyComparable = IndirectRelation<R, projected<I1, P1>, projected<I2, P2>>

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

**C++ definition:**

    template <class I>
    concept WeaklyIncrementable =
        Semiregular<I> &&
        requires (I i) {
            typename difference_type_t<I>;
            requires SignedIntegral<difference_type_t<I>>;
            { ++i } -> Same<I>&; // not required to be equality preserving
            i++; // not required to be equality preserving
        };


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

**C++ definition:**

    template <class I>
    concept Incrementable =
        Regular<I> &&
        WeaklyIncrementable<I> &&
        requires (I i) {
            { i++ } -> Same<I>&&;
        };


#### Iterator abstractions

The following concept describe iterator abstractions. [Stroustrup12](#Stroustrup12) §3.5.3


##### Iterator

The `Iterator` concept forms the basis of the iterator concept taxonomy; every iterator satisfies the `Iterator` requirements. This concept specifies operations for dereferencing and incrementing an iterator. Most algorithms will require additional operations to compare iterators with sentinels, to read or write values, or to provide a richer set of iterator movements.

**C++ definition:**

    template <class I>
    concept Iterator =
        requires (I i) {
            { *i } -> auto&&; // Requires: i is dereferenceable
        } &&
        WeaklyIncrementable<I>;

The requirement that the result of dereferencing the iterator is deducible from `auto&&` means that it cannot be `void`.


##### Sentinel

The `Sentinel` concept specifies the relationship between an `Iterator` type and a `Semiregular` type whose values denote a range.

**C++ definition:**

    template <class S, class I>
    concept Sentinel =
        Semiregular<S> &&
        Iterator<I> &&
        WeaklyEqualityComparableWith<S, I>;

Let `s` and `i` be values of type `S` and `I` such that `[i,s)` denotes a range. Types `S` and `I` satisfy `Sentinel<S, I>` iff:

- `i == s` is well-defined.
- If `bool(i != s)` then `i` is dereferenceable and `[++i,s)` denotes a range.

The domain of `==` can change over time. Given an iterator `i` and sentinel `s` such that `[i,s)` denotes a range and `i != s`, `[i,s)` is not required to continue to denote a range after incrementing any iterator equal to `i`. Consequently, `i == s` is no longer required to be well-defined.


##### SizedSentinel

The `SizedSentinel` concept specifies requirements on an `Iterator` and a `Sentinel` that allow the use of the `-` operator to compute the distance between them in constant time.

The `SizedSentinel` concept is satisfied by pairs of `RandomAccessIterator`s and by counted iterators.

**C++ definition:**

    template <class S, class I>
    constexpr bool disable_sized_sentinel = false;
    
    template <class S, class I>
    concept SizedSentinel =
        Sentinel<S, I> &&
        !disable_sized_sentinel<remove_cv_t<S>, remove_cv_t<I>> &&
        requires (const I& i, const S& s) {
            { s - i } -> Same<difference_type_t<I>>;
            { i - s } -> Same<difference_type_t<I>>;
        };

Let `i` be an iterator of type `I`, and `s` a sentinel of type `S` such that `[i,s)` denotes a range. Let `N` be the smallest number of applications of `++i` necessary to make `bool(i == s)` be true. `SizedSentinel<S, I>` is satisfied iff:

- If `N` is representable by `difference_type_t<I>`, then `s - i` is well-defined and equals `N`.
- If `N` is representable by `difference_type_t<I>`, then `i - s` is well-defined and equals `N`-

`disable_sized_sentinel` provides a mechanism to enable use of sentinels and iterators with the library that meet the syntactic requirements but do not in fact satisfy `SizedSentinel`. A program that instantiates a library template that requires `SizedSentinel` with an iterator type `I` and sentinel type `S` that meet the syntactic requirements of `SizedSentinel<S, I>` but do not satisfy `SizedSentinel` is ill-formed with no diagnostic required unless
`disable_sized_sentinel<S, I>` evaluates to `true`.


##### InputIterator

The `InputIterator` concept is a refinement of `Iterator`. It defines requirements for a type whose referenced values can be read (from the requirement for `Readable`) and which can be both pre- and post-incremented. `InputIterator`s are not required to satisfy `EqualityComparable`, and do not require that increment is an equality-preserving operation, meaning that a range can only be traversed once.

**C++ definition:**

    template <class I>
    concept InputIterator =
        Iterator<I> &&
        Readable<I> &&
        requires {
            typename iterator_category_t<I>;
        } &&
        DerivedFrom<iterator_category_t<I>, input_iterator_tag>;


##### OutputIterator

The `OutputIterator` concept is a refinement of `Iterator`. It defines requirements for a type that can be used to write values (from the requirement for `Writable`) and which can be both pre- and post-incremented. However, output iterators are not required to satisfy `EqualityComparable`.

Algorithms on output iterators should never attempt to pass through the same iterator twice. They should be single pass algorithms.

**C++ definition:**

    template <class I, class T>
    concept OutputIterator =
        Iterator<I> &&
        Writable<I, T> &&
        requires (I i, T&& t) {
            *i++ = std::forward<T>(t); // not required to be equality preserving
        };


##### ForwardIterator

The `ForwardIterator` concept refines `InputIterator`, adding equality comparison. Because its increment operation is equality-preserving, it permits multiple passes over a range.

**C++ definition:**

    template <class I>
    concept ForwardIterator =
        InputIterator<I> &&
        DerivedFrom<iterator_category_t<I>, forward_iterator_tag> &&
        Incrementable<I> &&
        Sentinel<I, I>;

Two dereferenceable iterators `a` and `b` of type `X` offer the multi-pass guarantee iff:

- `a == b` implies `++a == ++b`
- The expression `([](X x){++x;}(a), *a)` is equivalent to the expression `*a`.

The requirement that `a == b` implies `++a == ++b` (which is not true for weaker iterators) and the removal of the restrictions on the number of assignments through a mutable iterator (which applies to output iterators) allow the use of multi-pass one-directional algorithms with forward iterators.


##### BidirectionalIterator

The `BidirectionalIterator` concept refines `ForwardIterator`, and adds the ability to move an iterator backward as well as forward.

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

**C++ definition:**

    template <class I>
    concept BidirectionalIterator =
        ForwardIterator<I> &&
        DerivedFrom<iterator_category_t<I>, bidirectional_iterator_tag> &&
        requires (I i) {
            { --i } -> Same<I>&;
            { i-- } -> Same<I>&&;
        };


##### RandomAccessIterator

The `RandomAccessIterator` concept refines `BidirectionalIterator` and adds support for constant-time advancement with `+=`, `+`, `-=`, and `-`, and the computation of distance in constant time with `-`. Random access iterators also support array notation via subscripting.

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

**C++ definition:**

    template <class I>
    concept RandomAccessIterator =
        BidirectionalIterator<I> &&
        DerivedFrom<iterator_category_t<I>, random_access_iterator_tag> &&
        StrictTotallyOrdered<I> &&
        SizedSentinel<I, I> &&
        requires (I i, const I j, const difference_type_t<I> n) {
            { i += n } -> Same<I>&;
            { j + n } -> Same<I>&&;
            { n + j } -> Same<I>&&;
            { i -= n } -> Same<I>&;
            { j - n } -> Same<I>&&;
            j[n];
            requires Same<decltype(j[n]), reference_t<I>>;
        };


### Rearrangements

Rearrangements group together iterator requirements of algorithm families. [Stroustrup12](#Stroustrup12) §3.6


##### Permutable

The `Permutable` concept specifies the common requirements of algorithms that reorder elements in place by moving or swapping them.

**C++ definition:**

    template <class I>
    concept Permutable =
        ForwardIterator<I> &&
        IndirectlyMovableStorable<I, I> &&
        IndirectlySwappable<I, I>;


##### Mergeable

The `Mergeable` concept specifies the requirements of algorithms that merge sorted sequences into an output sequence by copying elements.

    template <class I1, class I2, class Out,
    class R = less<>, class P1 = identity, class P2 = identity>
    concept Mergeable =
        InputIterator<I1> &&
        InputIterator<I2> &&
        WeaklyIncrementable<Out> &&
        IndirectlyCopyable<I1, Out> &&
        IndirectlyCopyable<I2, Out> &&
        IndirectStrictWeakOrder<R, projected<I1, P1>, projected<I2, P2>>;


##### Sortable

The `Sortable` concept specifies the common requirements of algorithms that permute sequences into ordered sequences (e.g., `std::sort`).

    template <class I, class R = less<>, class P = identity>
    concept Sortable =
        Permutable<I> && IndirectStrictWeakOrder<R, projected<I, P>>;


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

<a name="Sutton17"></a>
[Sutton17]
"Working Draft, C++ Extensions for Concepts", ISO N4641
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4641.pdf

<a name="Niebler17"></a>
[Niebler17]
"Programming Languages - C++ Extensions for Ranges", ISO N4685
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4685.pdf

<a name="Stroustrup17"></a>
[Stroustrup17]
"Concepts: The Future of Generic Programming - or How to design good concepts and use them well"
http://stroustrup.com/good_concepts.pdf

<a name="Sutter17"></a>
[Sutter17]
"C++ Core Guidelines - T: Templates and generic programming"
https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-templates

<a name="Overload16"></a>
[Overload16]
"Overload 136 pp6-11 - Overloading with Concepts"
https://accu.org/var/uploads/journals/Overload136.pdf

[Origin libraries](https://github.com/asutton/origin)

[range-v3](https://github.com/ericniebler/range-v3)

[cmcstl2](https://github.com/CaseyCarter/cmcstl2)
