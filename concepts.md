C++ concepts quick reference
============================

This document is a collection of design guidelines for C++ concepts and design guidelines for classes and algorithms using concepts. It is intended to be used as a quick reference when designing new concepts/classes/algorithms or refactoring old ones.
For brevity and simplicity of use, it does not discuss at length the *whys* of the guidelines, but instead presents references to further reading about the subject.

The guidelines cover C++20.


Concept definitions
-------------------

Concepts are defined as compile-time boolean expressions.

**Example:**

    template <typename T>
    concept Any = true;
    
    template <typename T>
    concept Wip = "Requirements documented here as a string literal pending implementation";

A *requires-expression* can be used inside a concept to express syntactic requirements on template arguments. There are four types of requirements. It can also be used as a standalone compile-time predicate.

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
    
    // Standalone predicate
    template <typename T>
    constexpr bool Has_member_fun = requires (T x) { { x.fun() } -> std::same_as<int>; };


Template constraints
--------------------

Template parameter types can be constrained to satisfy concepts. There are three syntactic forms available.

**Example:**

    // Most general form with requires clause
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

The `auto` keyword can also be used to describe a completely unconstrained parameter. Note that the semantics differ from when a named concept is defined. [Stroustrup17](#Stroustrup17)
§7.1

**Example:**

    concept bool Any = true; // Every type is an any
    
    void f(Any x, Any y); // x and y must take the same type
    void g(auto x, auto y); // x and y can be of different type

In pre-C++20 code, aliasing the `requires` keyword with a variadic macro, or using `#define` to alias the `typename` keyword, provides easy ways to declare template constraints in anticipation of compiler support for the first two forms, treating them as documentation. [Stroustrup17](#Stroustrup17) §5.6

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
        requires equality_comparable_with<T>
            bool operator==(const vector& x) const;
    
        requires Totally_ordered<T>
            bool operator<(const vector& x) const;
    };

Multi-type constraints can easily be expressed using explicit form with conjunctions or shorthand constraint form with implicit conjunction: [Sutton13](#Sutton13) §2.1.1

**Example:**

    // Explicit form
    template <typename S, typename T>
        requires Sequence<S> && equality_comparable_with<T, Value_type<S>>
    Iterator_type<S> find(S&& sequence, const T& value);
    
    // Shorthand form
    template <Sequence S, equality_comparable_with<Value_Type<S>> T>
    Iterator_type<S> find(S&& sequence, const T& value);


### Matching functions to concepts

Roughly speaking, C++ considers one function to be "better" than another using the following rules: [Overload16](#Overload16)

1. Functions requiring fewer or "cheaper" conversions of the arguments are better than those requiring more or costlier conversions.
2. Non-template functions are better than function template specializations.
3. One function template specialization is better than another if its parameter types are more specialized. For example, `T*` is more specialized than `T`, and so is `vector<T>`, but `T*` is not more specialized than `vector<T>`, nor is the opposite true.
4. If two functions cannot be ranked because they have equivalent conversions or are function template specializations with equivalent parameter types, then the better one is more constrained. Also, unconstrained functions are the least constrained.

In other words, constraints act as a tie-breaker for the usual overloading rules in C++. The ordering of constraints (more constrained) is essentially determined by the comparing sets of requirements for each template to determine if one is a strict superset of another.

If the difference between two concepts is purely semantic, overload resolution may still be ambiguous. This is the case for example with the standard `invocable` and `regular_invocable` concepts. To express that such a concept is a more constrained concept, traditional tag dispatch (associating a tag class, i.e. an empty class in an inheritance hierarchy) with types for the purpose of selecting appropriate overloads can be used.

**Example:**

    // Ambiguous
    template <typename T>
    concept Less_general = Most_general<T> && "has different semantics";
    
    // More constrained
    template <typename T>
    concept Less_general =
        Most_general<T> &&
        requires {
            typename semantic_category_t<T>;
            requires derived_from<T, less_general_tag>;
        };

Alternative solutions include using an associated type trait, variable template, or even an extra operation.

**Example:**

    template <typename T>
    constexpr bool is_less_general_v = false;
    
    template <>
    constexpr bool is_less_general_v<less_general_type> = true;
    
    template <typename T>
    concept Less_general = Most_general<T> && is_less_general_v<T>;

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

In C++, type attributes can be implemented with *traits classes*, specialized for each particular type where necessary.

**Example:**

    template <typename T>
    struct zero_value {
        static const T value = 0;
    };
    
    // Convenience notation using value templates (C++14)
    template <typename T>
    constexpr auto Zero = zero_value<T>::value;
    
    // Convenience notation using macro (C++98)
    #define Zero(T) zero_value<T>::value

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

**Example:**

    // int_wrapper is a type, not a type constructor
    struct int_wrapper {
        int value;
    };
    
    // wrapper is a type constructor, not a type
    template <typename T>
    struct wrapper {
        T value;
    };
    
    // wrapper<int> is a type, not a type constructor
    using IntWrapper = wrapper<int>;


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
    concept Addable = requires (T a, T b) { { a + b } -> T; };

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

In the rare case where two concepts are identical syntactically but differ semantically, either disambiguate them by adding an operation or a member type or use a traits class in the implementation of operations on them. [Stroustrup17](#Stroustrup17) §8.5, [Sutter17](#Sutter17) §T.24

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

Sometimes it is a good idea to avoid constraining class templates with concepts, as this disables the possibility to use an incomplete instance of the type in another class. A good example of this is the standard container classes. [ODwyer20](#ODwyer20)

**Example:**

    template <std::regular T>
    class BadVector {
        T* data:
    };
    
    template <typename T>
    class GoodVector {
        T* data:
    };
    
    class TreeNode {
        int root;
        BadVector<TreeNode> children; // Oops! Compiler wants to check if BadVector<TreeNode> is std::regular, but it cannot because TreeNode is incomplete at this point.
    };
    
    class TreeNode {
        int root;
        GoodVector<TreeNode> children; // Ok! TreeNode is incomplete at this point but the compiler doesn't need it yet. It will know what GoodVector<TreeNode> is once the class definition is complete.
    };

As a consequence of this, concept checks like `std::copyable<std::vector<T>>` will test as true, even if `std::copyable<T>` is false. This is because unconstrained member functions such as the copy constructor can be defined even if `T` has deleted it. Instantiating the copy constructor would of course yield a compile-time error, but concepts are only checked against interfaces, not implementations. Note that concepts can still be checked with `static_assert` inside the member functions of an unconstrained class, providing earlier error checking and better error messages.


Concept definitions
-------------------

The concepts presented in this section are standardized and thus should not be user-implemented. They merely serve to illustrate some useful implementation techniques.

The concept definitions consist of *requirements*, which express syntax and must be checked by the compiler, and *axioms*, which express semantics and must not be checked by the compiler.


### Language concepts

Language concepts are intrinsic to the C++ programming language. [Stroustrup12](#Stroustrup12) §3.2


##### swappable

`swappable` and `swappapble_with` demonstrates symmetric treatment of two similar types.

**C++ Defintion:**

    template <class T>
    concept swappable =
        requires (T& a, T& b) {
            ranges::swap(a, b);
        };
    
    template <class T, class U>
    concept swappable_with =
        common_reference_with<T, U> &&
        requires (T&& t, U&& u) {
            ranges::swap(std::forward<T>(t), std::forward<T>(t));
            ranges::swap(std::forward<U>(u), std::forward<U>(u));
            ranges::swap(std::forward<T>(t), std::forward<U>(u));
            ranges::swap(std::forward<U>(u), std::forward<T>(t));
        };


#### Type classifications

The following concepts classify fundamental types. Their semantics are fully described in the C++ standard, using type traits.


##### integral

The `integral` concept is simply defined by the type trait `is_integral<T>`.

**C++ definition:**

    template <class T>
    concept integral = std::is_integral<T>::value;

**Example:**

    integral<int>          // is true
    integral<unsigned int> // is true
    integral<double>       // is false


##### signed_integral

The `signed_integral` concept adds the `is_signed_v<T>` requirement.

**C++ definition:**

    template <class T>
    concept signed_integral = integral<T> && std::is_signed_v<T>;

**Example:**

    signed_integral<int>          // is true
    signed_integral<signed int>   // is true
    signed_integral<unsigned int> // is false
    signed_integral<double>       // is true


##### unsigned_integral

The `unsigned_integral` concept is defined in terms of the `integral` and `signed_integral` concept.

**C++ definition:**

    template <class T>
    concept unsigned_integral = integral<T> && !signed_integral<T>;

**Example:**

    unsigned_integral<int>          // is false
    unsigned_integral<signed int>   // is false
    unsigned_integral<unsigned int> // is true
    unsigned_integral<double>       // is false


#### Type relations

The following concpets describe relationships between types.


##### same_as

The `same_as` concept is built-in to the language, described by the standard library type `is_same`. For two types `T` and `U`, `Same<T, U>` is true iff `T` and `U` denote exactly the same type after elimination of aliases. For the purposes of constraint checking, `same_as<T, U>` implies `same_as<U, T>`.

**C++ definition:**

    // Detail namespace used to hide a helper concept used to define another concept
    namespace detail {
        template <class T, class U>
        concept same_helper = std::is_same_v<T, U>;
    }
 
    template <class T, class U>
    concept same_as = detail::same_helper<T, U> && detail::same_helper<U, T>;

**Example:**

    using Pi = int*;
    using I = int;
    same_as<Pi, I> // is false
    same_as<Pi, I*> // is true


##### convertible_to

The `convertible_to` concept expresses the requirement that a type `T` can be implicitly converted to a `U`.

**C++ definition:**

    // Requires-expression uses a function parameter to be able to work with most C++ types
    template <class From, class To>
    concept convertible_to =
        std::is_convertible_v<From, To> &&
        requires(std::add_rvalue_reference_t<From>(&f)()) {
            static_cast<To>(f());
        };

**Example:**

    convertible_to<double, int>             // is true: int i = 2.7; (ok)
    convertible_to<double, complex<double>> // is true: complex<double> d = 3.14; (ok)
    convertible_to<complex<double>, double> // is false: double d = complex<double>2,3 (error)
    convertible_to<int, int>                // is true: a type is convertible to itself
    convertible_to<Derived, Base>           // is true: derived types can be converted to base types


##### common_with

The `common_with` concept expresses that two types `T` and `U` can both be unambiguously converted to a third type `C`, i.e. they share a *common type*. `C` can be the same type as `T` and `U` or it can be a different type:

    requirement: std::common_type_t<T, U> (alias for the standard type trait std::common_type<T, U>::type)
    axiom: eq(t1, t2) <=> eq(C{t1}, C{t2})
    axiom: eq(u1, u2) <=> eq(C{u1}, C{u2})

**C++ definition:**

    template <class T, class U>
    concept common_with =
        same_as<common_type_t<T, U>, std::common_type_t<U, T>> &&
        convertible_to<T, std::common_type_t<T, U>> &&
        convertible_to<U, std::common_type_t<T, U>> &&
        common_reference_with<
            add_lvalue_reference_t<const T>,
            add_lvalue_reference_t<const U>> &&
        common_reference_with<
            add_lvalue_reference_t<std::common_type_t<T, U>>,
            common_reference_t<
                add_lvalue_reference_t<const T>,
                add_lvalue_reference_t<const U>>>;

Users are free to specialize `common_type` when at least one parameter is a user-defined type. Those specializations are considered by the `common_with` concept.

**Example:**

    common_with<int, double>           // is true
    common_with<int, int>              // is true
    common_with<char*, std::string>    // is true
    common_with<employee, std::string> // is probably false


### Foundational concepts

Foundational concepts form the basis of a style of programming or are needed to write programs in that style. The following concepts describe the basis of the value-oriented programming style on which the STL is based. [Stroustrup12](#Stroustrup12) §3.3


##### boolean-testable

The `boolean-testable` concept describes the requirements on a type that is used in Boolean contexts. It is an *exposition-only* concept, allowing implementers to provide an alternative implementation of

**C++ definition:**

    // exposition only
    template <class B>
    concept __boolean_testable_impl =
        std::convertible_to<B, bool>;
    
    // exposition only
    template <class B>
    concept __boolean_testable =
        __boolean_testable_impl<B> &&
        requires (B&& b) {
            { !std::forward<B>(b) } -> __boolean_testable_impl;
        };


##### equality_comparable

The ability to compare objects for equality is described by the
`equality_comparable` and `equality_comparable_with` concepts:

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

    // exposition only
    template<class T, class U>
    concept __weakly_equality_comparable_with =
        requires(const remove_reference_t<T>& t, const remove_reference_t<U>& u) {
            { t == u } -> __boolean_testable;
            { t != u } -> __boolean_testable;
            { u == t } -> __boolean_testable;
            { u != t } -> __boolean_testable;
        };
    
    template <class T>
    concept equality_comparable = __weakly_equality_comparable_with<T, T>;
    
    template <class T, class U>
    concept equality_comparable_with =
        equality_comparable<T> &&
        equality_comparable<U> &&
        std::common_reference_with<
            const std::remove_reference_t<T>&,
            const std::remove_reference_t<U>&> &&
        std::equality_comparable<
            std::common_reference_t<
                const std::remove_reference_t<T>&,
                const std::remove_reference_t<U>&>> &&
        __weakly_equality_comparable_with<T, U>;

The distinction between `equality_comparable_with<T, U>` and `__weakly_equality_comparable_with<T, U>` is purely semantic. The requirement that the expression a == b is equality preserving implies that == is reflexive, transitive, and symmetric.

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


##### semiregular

The `semiregular` concept describes types that behave in regular ways except that they might not be comparable using `==`. Examples of semiregular types include all built-in C++ integral and floating point types, all trivial classes, `std::string` and standard containers of copyable types. [Stroustrup12](#Stroustrup12) §3.3

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

Proper use of the `semiregular` concept requires that it is evaluated over value types, not object types:

- Constants (`const T`) and `constexpr` objects do not have `Semiregular` types since they cannot be assigned to.
- Volatile types (`volatile T`) are not `Semiregular` since their objects' states may by changed externally.
- Reference types types are not `Semiregular` because the syntax used for copy construction does not actually create a copy; it binds a reference to an
object.

**C++ definition:**

    template <class T>
    concept semiregular = copyable<T> && default_initializable<T>;


##### regular

The `regular` concept is the union of the `semiregular` and `equality_comparable` concepts. [Stroustrup12](#Stroustrup12) §3.3

**C++ definition:**

    template <typename T>
    concept regular = semiregular<T> && equality_comparable<T>;

Not all arguments will be well-formed for a given type. For example, `NaN` is not a well-formed
floating point value, and many types' moved-from states are not well-formed. This does not mean that the type does not satisfy `StrictTotallyOrdered`.


### Callable concepts

Callable concepts describe requirements on function objects and their arguments. [Stroustrup12](#Stroustrup12) §3.4


##### invocable

The `invocable` concept describes a type whose objects can be called over a (possibly empty) sequence of arguments. `Invocable`s are not `Semiregular` types; they may not exhibit the full range of capabilities as built-in value types. Minimally, they can be expected to be copy- and move-constructible but not copy- and move-assignable:

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
    concept invocable =
        requires (F&& f, Args&&... args) {
            invoke(std::forward<F>(f), std::forward<Args>(args)...); // not required to be equality preserving
    };


##### regular_invocable

The `regular_invocable` concept describes an `invocable` that is equality-preserving, i.e. it it is equality-preserving and does not modify the function object or the arguments. The distinction between `invocable` and `regular_invocable` is purely semantic:

    invocable<F, Args...>
    axiom: equality_preserving(f(args))

**C++ definition:**

    template <class F, class... Args>
    concept regular_invocable = invocable<F, Args...>;

Since the invoke function call expression is required to be equality-preserving, a function that generates random numbers does not satisfy `regular_invocable`.


##### predicate

The `predicate` concept describes a `regular_invocable` whose return type is `boolean-testable`:

    regular_invocable<P, Args...>
    convertible_to<ResultType<P, Args...>, bool>

**C++ definition:**

    template <class F, class... Args>
    concept predicate =
        regular_invocable<F, Args...> &&
        __boolean_testable<std::std::invoke_result_t<F, Args...)>>;

A unary `invocable` with own state, e.g. a function that alternately returns true and false irrespective of its inputs, is an example of a `pseudopredicate`. This is a useful concept that is not defined in the standard. [Stepanov09](#Stepanov09) §8.2


##### relation

The `relation` concept describes a binary `predicate`.

The STL is concerned with two types of relations: *equivalence relations*, which generalize equality, and *strict weak orderings*, which generalize total orderings. In C++ these properties can only be defined for particular objects, not types, since there can be many functions with type `bool(T, T)` that are neither equivalence relations nor strict weak orderings.

    relation<R, T1>
    relation<R, T2>
    common_with<T1, T2>
    relation<R, std::common_type_t<T1, T2>>
    requirement: bool r(t1, t2)
    requirement: bool r(t2, t1)
    axiom:
        r(t1, t2) <=> r(C{t1}, C{t2})
        r(t2, t1) <=> r(C{t2}, C{t1})

In summary, a `relation` can be defined on different types if:

- `T1` and `T2` share a common type `C`
- The relation `R` is defined for all combinations of those types
- Any invocation of `r` on any combinations of types `T1`, `T2`, and `C` is equivalent to an invocation `r(C{t1}, C{t2})`

**C++ definition:**

    template <class R, class T, class U>
    concept relation =
        predicate<R, T, T> && predicate<R, U, U> &&
        predicate<R, T, U> && predicate<R, U, T>;

A binary homogeneous `invocable` with own state, e.g. a function that alternately returns true and false irrespective of its inputs, is an example of a `pseudorelation`. This is a useful concept that is not defined in the standard. [Stepanov09](#Stepanov09) §8.2


##### equivalence_relation

A `relation` is an `equivalence_relation` iff it imposes an *equivalence relation* on its arguments.

    axiom (equivalence relation):
        rel(a, a) (reflexive)
        rel(a, b) => rel(b, a) (symmetric)
        rel(a, b) && rel(b, c) <=> rel(a, c) (transitive)

**C++ definition:**

    template <class R, class T, class U >
    concept equivalence_relation = std::relation<R, T, U>;


##### strict_weak_order

A `relation` is `strict_weak_order` iff it imposes a *strict weak ordering* on its arguments.

    axiom: (strict weak order)
        !rel(a, a) (irreflexive)
        rel(a, b) => !rel(b, a) (assymetric)
        rel(a, b) && rel(b, c) => rel(a, c) (transitive)
        rel(a, b) => rel(a, c) || rel(c, b) (transitivity of incomparability)

**C++ definition:**

    template <class R, class T, class U>
    concept strict_weak_order = relation<R, T, U>;


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

<a name="Stroustrup17"></a>
[Stroustrup17]
"Concepts: The Future of Generic Programming - or How to design good concepts and use them well"
http://stroustrup.com/good_concepts.pdf

<a name="Sutter17"></a>
[Sutter17]
"C++ Core Guidelines - T: Templates and generic programming"
https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-templates

<a name="ODwyer20"></a>
[ODwyer20]
"SFINAE special members or support incomplete types: Pick at most one"
https://quuxplusone.github.io/blog/2020/02/05/vector-is-copyable-except-when-its-not/

<a name="Overload16"></a>
[Overload16]
"Overload 136 pp6-11 - Overloading with Concepts"
https://accu.org/var/uploads/journals/Overload136.pdf
