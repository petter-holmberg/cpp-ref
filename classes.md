C++ classes quick reference
===========================

This document is a collection of design guidelines for C++ classes. It is intended to be used as a quick reference when designing new classes or refactoring old ones.
For brevity and simplicity of use, it does not discuss at length the *whys* of the guidelines, but instead presents references to further reading about the subject.

The guidelines are currently focused on C++98 use but features some notes about later versions.


General class design guidelines
-------------------------------

When deciding how to design a C++ class, there are several considerations to take into account.

Treat class design as type design. Before defining a new type, be sure to consider all these issues: [Meyers05](#Meyers05) §19
- How should objects of your new type be created and destroyed?
- How should object initialization differ from object assignment?
- What does it mean for objects of your new type to be passed by value?
- What are the restrictions on legal values for your new type?
- Does your new type fit into an inheritance graph?
- What kind of type conversions are allowed for your new type?
- What operators and functions make sense for the new type?
- What standard functions should be disallowed?
- Who should have access to the members of your new type?
- What is the "undeclared interface" of your new type?
- How general is your new type?
- Is a new type really what you need?


### Interface vs. Implementation

A class's interface is made up of the public (and protected) functions and data members it exposes, including non-member functions.

A class's implementation is encapsulated within the class, and includes the private functions and data members.

When designing a class, follow the Law of Second Chances: [Sutter05](#Sutter05) §36
- The most important thing to get right is the interface. Everything else can be fixed later. Get the interface wrong and you may never be allowed to fix it.

When designing a class, follow the Dependency Inversion Principle (DIP) and prefer to provide [Abstract interfaces](#AbstractInterfaces): [Sutter05](#Sutter05) §36
- High-level modules should not depend on low-level modules. Rather, both should depend upon abstractions.
- Abstractions should not depend upon details. Rather, details should depend upon abstractions.

Make interfaces easy to use correctly and hard to use incorrectly: [Meyers05](#Meyers05) §18
- Good interfaces are easy to use correctly and hard to use incorrectly. Strive for these characteristics in all interfaces.
- Ways to facilitate correct use include consistency in interfaces and behavioral compatibility with built-in types.
- Ways to prevent errors include creating new types, restricting operations on types, constraining object values, and eliminating client resource management responsibilities.

By default, all data members should be private (except if the class is an [Aggregate](#Aggregates)) and accessible only through member functions. [Sutter05](#Sutter05) §41

By default, classes should never provide functions that return handles to internal (private) members, allowing external code to modify them (even const is not safe because it can be cast away).
The exception is if compatibility with existing (legacy) code is necessary, and then the reasoning behind these exceptions should be clearly documented. [Sutter05](#Sutter05) §42


### Composition vs. Inheritance

Compositioning is to embed a member variable of one class within another class, and use the member in ways that make sense to the outer class (like providing pass-through functions to some of the inner class' interface).

Composition has meanings completely different from that of public inheritance. In the application domain, composition means "has-a", in the implementation domain, it means "is-implemented-in-terms-of". [Meyers05](#Meyers05) §38

Prefer composition to implementing [Base classes](#BaseClasses), it minimizes coupling. [Sutter02](#Sutter02) §23, [Sutter05](#Sutter05) §34


### Inheritance guidelines

Inheritance in C++ comes in three flavors:

1.  Public inheritance

    Public inheritance, models the "is-a" (or better, "works-like-a") ...or "is-implemented-in-terms-of" relationship. Everything that applies to base classes must also apply to derived classes, because every derived class object is a base class object. [Meyers05](#Meyers05) §32

    Names in derived classes hide names in base classes. Under public inheritance, this is never desirable. To make hidden names visible again, employ `using` declarations or forwarding functions. [Meyers05](#Meyers05) §33

2.  Private inheritance

    Private inheritance, models the "has-a", or "is-implemented-in-terms-of" relationship. It's usually inferior to composition, but it makes sense when a derived class needs access to protected base class members or needs to redefine inherited virtual functions. [Meyers05](#Meyers05) §39

    Unlike composition, private inheritance can enable the [Empty Base Optimization](#EBO). This can be important for library developers who strive to minimize object sizes. [Meyers05](#Meyers05) §39

3.  Protected inheritance

    Under protected inheritance, public members of the base class becomes protected under the derived class.

Use private inheritance instead of composition only when absolutely necessary, which means when: [Sutter99](#Sutter99) §15
- You need access to the class's protected members.
- You need to override a virtual function.
- The object needs to be constructed before other base sub-objects.

Never use public inheritance except to model true Liskow IS-A and WORKS-LIKE-A. All overridden member functions must require no more and promise no less. [Sutter99](#Sutter99) §22

Never inherit from a class that was not designed to be a base class.
To add more functionality to a concrete class, define free (nonmember) functions to do the job instead, and put them in the same namespace as the class they are designed to extend. [Sutter05](#Sutter05) §35

Never inherit publicly only because it allows you to reuse common code in the base class! Instead, inherit because it allows the derived class to be reused by code that uses objects of the base class polymorphically.
Before object orientation, it has always been easy for new code to call existing code. Public inheritance specifically makes it easier for existing code to seamlessly and safely call new code. (So do templates, which provide static polymorphism that can blend well with dynamic polymorphism.) [Sutter99](#Sutter99) §22 [Sutter05](#Sutter05) §37

Both classes and templates support interfaces and polymorphism. Templates can be an alternative to implementing public inheritance (static polymorphism instead of dynamic). [Meyers05](#Meyers05) §41, [Sutter05](#Sutter05) §37
- For classes, interfaces are explicit and centered on function signatures. Polymorphism occurs at runtime through virtual functions.
- For template parameters, interfaces are implicit and based on valid expressions. Polymorphism occurs during compilation through template instantiation and function overloading resolution.

When two similar concrete classes duplicate code, don't make one class the base class of the other. Instead, define a new abstract base class and make the two old classes inherit publicly from that. [Meyers96](#Meyers96) §33

Never redefine an inherited non-virtual function. [Meyers05](#Meyers05) §36

Differentiate between inheritance of interface and inheritance of implementation: [Meyers05](#Meyers05) §34
- Inheritance of interface is different from inheritance of implementation. Under public inheritance, derived classes inherit base class interfaces.
- Pure virtual functions specify inheritance of interface only.
- Simple (impure) virtual functions specify inheritance of interface plus inheritance of a default implementation.
- Non-virtual functions specify inheritance of interface plus inheritance of a mandatory implementation.

Inheritance should be used only when: [Sutter05](#Sutter05) §34
-   You want to model substitutability (public inheritance).
-   You need to override a virtual function (non-public inheritance).
-   You need to access a protected member (non-public inheritance).
    This applies to protected member functions in general, and to protected constructors in particular.
-   You need to construct the used object before, or destroy it after, a base class.
    This can be necessary when the used class provides a lock of some sort, such as a critical section or a database transaction, which must cover the entire lifetime of another base sub-object.
-   You need to share a common virtual base class, or override the construction of a virtual base class (non-public inheritance).
    The first part applies if the using class has to inherit from one of the same virtual bases as the used class. If it does not, the second part may still apply: The most-derived class is responsible for initializing all virtual base classes, and so if you need to use a different constructor or different constructor parameters for a virtual base, then you must inherit.
-   You benefit from the Empty Base Optimization and it matters to the program (non-public inheritance).
-   You need controlled polymorphism (non-public inheritance).
    Public inheritance should always model "is-a" as per the Liskov Substitution Principle (LSP).
    Non-public inheritance can express a restricted form of "is-a", even though most people identify "is-a" with public inheritance alone. Given `class Derived : private Base`, from the point of view of outside code, a Derived object "is-not-a" Base, and so of course can't be used polymorphically as a Base because of the access restrictions imposed by private inheritance.
    However, inside Derived's own member functions and friends only, a Derived object can indeed be used polymorphically as a Base (you can supply a pointer or reference to a Derived object where a Base object is expected), because members and friends have the necessary access.
    If instead of private inheritance you use protected inheritance, then the "is-a" relationship is additionally visible to further-derived classes, which means subclasses can also make use of the polymorphism.

When overriding a virtual base class function: [Sutter05](#Sutter05) §38
- Make the overriding function virtual as well.
- Make the overriding function respect the same pre- and post-conditions as the base class function. (overrides should never require more or provide less than the original function.)
- When overriding, never change the value of default arguments. [Meyers05](#Meyers05) §37
- If the base class has multiple functions with the same name but different signatures, overriding just one hides all the others in the derived class. To bring them back into scope in the derived class, add a `using BaseClass::functionName` declaration in the derived class.

Try to avoid multiple inheritance of more than one class that is not an [Abstract interfaces](#AbstractInterfaces). [Meyers05](#Meyers05) §40, [Sutter02](#Sutter02) §24, [Sutter05](#Sutter05) §36
- Multiple inheritance is more complex than single inheritance. It can lead to new ambiguity issues and to the need for virtual inheritance.
- Virtual inheritance imposes costs in size, speed, and complexity of initialization and assignment. It's most practical when virtual base classes have no data.
- Multiple inheritance does have legitimate uses. One scenario involves combining public inheritance from an Interface class with private inheritance from a class that helps with implementation.


### Non-member, non-friend functions

Non-member, non-friend functions improve encapsulation by minimizing dependencies. Whenever possible, prefer making functions non-member non-friends. [Meyers05](#Meyers05) §23

The standard requires that operators `=` `()`  `->` `[]` and must be members, and class-specific operators `new`, `new[]`, `delete`, and `delete[]` must be static members.
Use the following algorithm for determining whether a function should be a member and/or friend [Sutter05](#Sutter05) §44, [Sutter99](#Sutter99) §20:

    If: The function is one of the operators =, ->, [], or (), which according to the standard must be members:
    
        Make it a member.
    
    Else if any of:
        a) The function needs a different type as its left-hand argument (as do `operator>>`` and `operator<<`, for example).
        b) The function needs type conversions on its leftmost argument.
        c) The function can be implemented using the class's public interface alone:
    
        Make it a nonmember (and friend if needed in cases a) and b) ).
    
        If it needs to behave virtually:
    
            Add a virtual member function to provide the virtual behavior, and implement the nonmember in terms of that.
    
    Else:
    
        Make it a member.

Always define a non-member function in the same namespace as its related class. [Sutter05](#Sutter05) §57

If you need type conversions on all parameters to a function (including the one that would otherwise be pointed to by the this pointer), the function must be a non-member. [Meyers05](#Meyers05) §24


### Exception guarantees

Provide the strongest possible exception guarantee for each function, and document it. A function can usually offer a guarantee no stronger than the weakest guarantee of the functions it calls. [Meyers05](#Meyers05) §29, [Sutter02](#Sutter02) §22, [Sutter05](#Sutter05) §71

Never make exception safety an afterthought. Exception safety affects a class's design. It is never "just an implementation detail". [Sutter99](#Sutter99) §5

Observe the canonical exception-safety rules:  [Sutter99](#Sutter99) §8-18
1. Never allow an exception to escape from a destructor or from an overloaded operator delete() or operator delete[](); write every destructor and deallocation function as though it had an exception specification of `throw()` (in C++1, use `nothrow`).
2. Always use the [RAII Idiom](#RAII) to isolate resource ownership and management.
3. In each function, take all the code that might emit an exception and do all that work safely off to the side. Only then, when you know that the real work has succeeded, should you modify the program state (and clean up) using only non-throwing operations.

#### No-fail Guarantee

The No-fail Guarantee is the strongest: The function simply cannot fail. The caller doesn't need to check for any errors.
A prerequisite for any destructor, deallocation function or `swap()` function. 

#### Strong Guarantee

The Strong Guarantee is to ensure that failure leaves the program in the (visible) state it was before the call, (unwinding back to that state if necessary) so that nothing has changed in case of failure.
The immediate caller should check for failures (if it can handle them correctly on that level) but doesn't need to worry about having changed state just by making the failed call.

The Strong Guarantee can often be implemented via copy-and-swap, but the Strong Guarantee is not practical for all functions. [Meyers05](#Meyers05) §29

#### Basic Guarantee

The Basic Guarantee is to ensure that the function either totally succeeds and reaches the intended target state, or it fails and leaves the program in state that is valid (but not predictable).
The caller needs to check the state on failure and handle it appropriately.

Anything less than the Basic Guarantee is to be considered a bug!


### Coding style

Avoid using leading underscore to indicate private members, this form of naming is reserved for the compiler and may cause naming conflicts. [Sutter05](#Sutter05) §0

Keep a class and its non-member function interface in the same namespace. Keep classes and functions in different namespaces if they are not specifically intended to work together. Especially important for templated functions or operators. [Sutter05](#Sutter05) §57 §58


Class types
-----------

C++ classes come in many flavors, and depending on which one you want to implement, different rules apply. Without being clear about what type of class you want, you are not likely to implement it correctly. [Sutter05](#Sutter05) §32

A class shold neatly fall into one of these categories. If it doesn't it is probably a sign that the class is doing too much and should be divided into multiple classes.


<a name="ValueClasses"></a>
### Value classes

Value classes are intended to be used as concrete classes, not as base classes. They should be modeled after built-in types and usually be instantiated on the stack or as directly held members of other classes. [Sutter05](#Sutter05) §32

Example: `std::vector`

**Implementation:**

    class ValueClass
    {
    public:
        // Copy-constructor: Public and non-virtual
        ValueClass(const ValueClass& rhs) {}
    
        // Copy assignment operator: Public and non-virtual
        ValueClass& operator=(const ValueClass& rhs) {}
    
        // Destructor: Public and non-virtual [Meyers05](#Meyers05) §7
        ~ValueClass() {}
    
        // Assignment operator with value semantics
        ValueClass& operator=(const ValueClass& rhs) {}
    
        // No virtual functions!
    };

Never use a value class as a base class! [Sutter05](#Sutter05) §35


<a name="Aggregates"></a>
#### Aggregates

Aggregates are a special case of [Value classes](#ValueClasses) that also have:
- No user-declared constructors.
- No private or protected non-static data members.
- No base classes.
- No virtual functions.

They hold a collection of values, and don't pretend to encapsulate or provide any behavior. C-style `struct`s are Aggregates in C++. So are arrays.

Example: `std::pair`

**Implementation:**

    struct Aggregate
    {
        std::string member1;
        SomeClass member2;
    };

By default, reserve usage of the `struct` keyword for Aggregates, because it requires no extra syntax for them and makes the intent clear.

Aggregates can be instantiated with curly braces:

    Aggregate a1 = {};
    Aggregate a2 = {1, 2};


#### POD-structs

POD-structs (Plain Old Data) are a special case of [Aggregates](#Aggregates) that also have:
- No non-static data members that are non-POD-structs, non-POD-unions (or arrays of such types) and no reference members.
- No user-defined assignment operator.
- No user-defined destructor.


**Implementation:**

    struct PodStruct
    {
        int member1;
        AnotherPodStruct member2[3];
    };

POD-structs are typically compatible with C-style `struct`s.


<a name="BaseClasses"></a>
### Base classes

Base classes are the building blocks of class hierarchies. They establish interfaces through virtual functions. They are usually instantiated dynamically on the heap as part of a concrete derived class object, and used via a (smart) pointer.

**Implementation:**

    class BaseClass
    {
    public:
    
        // Destructor allowing polymorphic deletion
        virtual ~BaseClass() {}
    
        // Non-virtual functions, defining interface
    
    private:
    
        // Copy constructor not implemented to disable copying
        ValueClass(const ValueClass& rhs);
    
        // Copy assignment operator not implemented to disable copying
        ValueClass& operator=(const ValueClass& rhs);
    
        // Virtual functions (protected if needed), defining implementation details
    };

Base classes with a high cost of change should have public functions that are non-virtual rather than (non-pure) virtual.
Virtual functions should be made private, or protected if derived classes need to be able to call them (see [NVI](#NVI)). Destructors are an exception to this rule. [Sutter05](#Sutter05) §39

Base classes should always be abstract if they don't need to be used as leaf classes in an inheritance hierarchy.
If a Base class is concrete because some aspects of it require it to be, while it's also used to implement common functionality of derived classes, separate out the common parts into an abstract Base class and make the original Base class inherit from that along with the other leaf classes. [Meyers96](#Meyers96) §33

Normally, Base class destructors should be public and virtual. To prevent polymorphic deletion through Base class pointers, make the destructor non-virtual and protected. [Sutter02](#Sutter02) §27

Not all Base classes are designed to be used polymorphically. Examples are the standard container types. As a result, they don't need virtual destructors. [Meyers05](#Meyers05) §7


<a name="AbstractInterfaces"></a>
### Abstract interfaces

Abstract interfaces are classes made up entirly of (pure) virtual functions and containing no state. Usually they don't have any member function implementations.

**Implementation:**

    class Interface
    {
    public:
    
        // Public virtual destructor to allow polymorphic deletion. [Sutter02] §27
        virtual ~Interface () = 0;
    
        // Only pure virtual interface functions.
        virtual SomeType interfaceFunction() = 0;
    
        // No data members!
    };


### Traits classes

Traits classes are templates that carry information about types. They contain only typedefs and static functions, and have no modifiable state or virtual functions.
Traits classes make information about types available during compilation (as opposed to virtual functions or function pointers).
In conjunction with overloading, traits classes make it possible to perform compile-time if...else tests on types. [Meyers05](#Meyers05) §47, [Sutter02](#Sutter02) §4 [Sutter05](#Sutter05) §32

> Think of a trait as a small object whose main purpose is to carry information used by another object or algorithm to determine "policy" or "implementation details"
> -- <cite>Bjarne Stroustrup</cite>

Traits classes can solve the problem of generalizing functions on different types when templates alone aren't sufficient (because of different behavior) and you cannot use class polymorphism (because the types are primitive). 
Traits classes rely on explicit template specialization for different types.

Examples: `std::iterator_traits`, `std::numeric_limits`

**Example implementations:**

    // Traits class template to determine if a type is void
    template <typename T>
    struct is_void
    {
       static const bool value = false;
    };
    
    // Template specialization for void
    template <>
    struct is_void<void>
    {
        static const bool value = true;
    };
    
    is_void<void>(); // true
    is_void<int>();  // false
    
    // Traits class with typedef:
    template <typename T>
    struct collection_traits
    {
        typedef std::vector<T> collection_type;
    };
    
    SomeClass<collection_traits>() item;
    SomeClass::collection_type items; 


### Functor classes

A functor is a class modeled after function pointers. The convention in STL is to pass functors by value. Therefore, they should be lightweight and have valid copy-semantics. They also need to be monomorphic, i.e. not use virtual functions.
State and polymorphism can be implemented using the [Pimpl idiom](#Pimpl). [Meyers01](#Meyers01) §38

Functors that are predicates (i.e. return `bool` or something that can be implicitly converted into `bool`) should be pure functions, i.e. their return value should only depend on the input values and not some internal state. 
Defining `operator()` `const` is necessary for predicates, but internally they must also avoid accessing mutable data members, non-const local static objects, non-const objects at namespace scope and non-const global objects. 
The C++ standard doesn't guarantee that stateful predicates will work with standard algorithms! [Meyers01](#Meyers01) §39 [Sutter02](#Sutter02) §3

Functors that are made adaptable can be used in many more contexts than functors that are not. Making them adaptable simply means to define some of the typedefs `argument_type`, `first_argument_type`, `second_argument_type`, and `result_type`. The conventional way to do so is to inherit from `std::unary_function` or `std::binary_function`, depending on if the functor takes one or two arguments. These base structs are templates, taking either two or three types. The last one is the return type of `operator()`, the first one(s) are its argument type(s).
When the input parameters are `const` and not pointers, it is conventional to strip off const qualifiers for these types, while for pointers the `const` should be kept. Adaptable functors do not define more than one `operator()` function. [Meyers01](#Meyers01) §40

**Example implementations:**

    // Lightweight predicate functor
    template <typename T>
    class Predicate : public unary_function<T, bool>
    {
    public:
    
        bool operator()(const T& value) const
        {
            return value.fulfillsPredicate();
        }
    };
    
    // Heavy polymorphic functor implemented using [Pimpl]
    template <typename T>
    class Functor : public unary_function<T, void>
    {
    public:
    
        void operator()(const T& value) const
        {
            pimpl_->operator()(value);
        }
    
    private:
    
        FunctorImpl<T> *pimpl_;
    };
    
    // The implementation class, holding state
    template <typename T>
    class FunctorImpl : public unary_function<T, void>
    {
    private:
    
        // Some heavy state
        Widget w_;
        int x_;
    
        virtual ~FunctorImpl(); // Polymorphic classes need virtual destructors
    
        virtual void operator()(const T& value) const;
    
        friend class Functor<T>;
    };

To accept a functor as argument, a function needs to be defined as a template:

    template <typename F>
    void foo(F functor)
    {
        functor(...);
    }

Such a function can accept both function pointers (`&fun`), functors (`fun()`) and (in C++11) lambdas (`[](){...}`).


<a name="ExceptionClasses"></a>
### Exception classes

Exception classes can be implemented to define domain-specific exceptions.

Exceptions should be thrown by value and be caught by (const) reference. If re-thrown, it should be done with `throw;` [Meyers96](#Meyers96) §13, [Sutter05](#Sutter05) §73

**Example implementation:**

    class ExceptionClass : public std:exception
    {
    public:
    
        // No-fail constructor
        ExceptionClass() {}
    
        // No-fail copy constructor (throwing from this would abort the program!) [Sutter05] §32
        ExceptionClass(const ExceptionClass& rhs) {}
    
        // Destructor
        ~ExceptionClass() throw() {};
    
        // Virtual functions, often implements Cloning and the [Visitor Pattern] [Sutter05] §54
    }


### Policy classes

Policy classes (normally templates) are fragments of pluggable behavior. They are not usually instantiated standalone but as a base or member of another class. They may or may not have state or virtual functions.

In brief, policy-based class design fosters assembling a class (called the *host*) with complex behavior out of many little classes (called *policies*), each of which takes care of only one behavioral or structural aspect. As the name suggests, a policy establishes an interface pertaining to a specific issue. 
You can implement policies in various ways as long as you respect the policy interface.

Because you can mix and match policies, you can achieve a combinatorial set of behaviors by using a small core of elementary components.

**Example implementation:**

    // Policy class for using std::cout to print a message
    class OutputPolicyWriteToCout
    {
    protected:
    
        template <typename MessageType>
        void print(MessageType const& message) const
        {
            std::cout << message << std::endl;
        }
    };
    
    // Policy class for the "Hello World!" message in English
    class LanguagePolicyEnglish
    {
    protected:
    
        std::string message() const
        {
            return "Hello, World!";
        }
    };
    
    // Policy class for the "Hello World!" message in German
    class LanguagePolicyGerman
    {
    protected:
    
        std::string message() const
        {
            return "Hallo Welt!";
        }
    };
    
    // Host class, accepting two different policies
    template <typename OutputPolicy, typename LanguagePolicy>
    class HelloWorld : private OutputPolicy, private LanguagePolicy
    {
        using OutputPolicy::print;
        using LanguagePolicy::message;
    
    public:
    
        // Behavior method
        void run() const
        {
            print(message()); // Two policy methods
        }
    };
    
    typedef HelloWorld<OutputPolicyWriteToCout, LanguagePolicyEnglish> HelloWorldEnglish;
    HelloWorldEnglish hello_world;
    hello_world.run(); // prints "Hello, World!"
    
    typedef HelloWorld<OutputPolicyWriteToCout, LanguagePolicyGerman> HelloWorldGerman;
    HelloWorldGerman hello_world2;
    hello_world2.run(); // prints "Hallo Welt!"


### Mixin classes

A C++ mixin class is a template class that is parameterized on its [Base class](#BaseClasses), implementing some specific fragment of functionality. It is intended to be composed with other classes.

**Example implementation:**

    class ConcreteMessage
    {
    public:
    
        void print()
        {
          cout << "Hello!";
        }
    };
    
    template<typename T>
    class BoldMixin : public T
    {
    protected:
    
        void print()
        {
            cout << "<b>";
            T::print();
            cout << "</b>";
        }
    };
    
    template<typename T>
    class ItalicMixin : public T
    {
    protected:
    
        void print()
        {
            cout << "<i>";
            T::print();
            cout << "</i>";
        }
    };
    
    ConcreteMessage<ItalicMixin<BoldMixin> > italicAndBold;
    italicAndBold.print(); // Prints the concrete message in italic and bold.


### Ancillary classes

Ancillary classes support specific [idioms](#CppIdioms). They should be easy to use correctly and hard to use incorrectly.


Class-related keywords
----------------------

The following is a summary of the important class-related keywords in C++


### class, struct, union

`class`, `struct` and `union` are all keywords used to defined C++ classes.

For a class defined using `class`, the default member access level is `private`.

For a class defined using `struct`, the default member access level is `public`.

For a class defined using `union`, the default member access level is `public`. It cannot contain virtual functions or static data members, and cannot use inheritance.
It is specifically used for declaring variables that can hold data of different types simultaneously, using the same portion of memory (its size is the size of the biggest data member).
Before C++11 unions have always been restricted to hold POD types.


### public, protected, private

These keywords may refer to class member access levels, or the form of inheritance used.

- `public` data members are part of the class's interface and can be accessed by anyone.
- `protected` data members are part of the class's interface only to classes that inherit from it.
- `private` data members are accessible only to the class itself.

`protected` is no more encapsulated than public! (see `using`). [Meyers05](#Meyers05) §22

Declare data members private. It gives clients syntactically uniform access to data, affords fine-grained access control, allows invariants to be enforced, and offers class authors implementation flexibility. [Meyers05](#Meyers05) §22


### const

Use `const` proactively, it makes code simpler to understand. Declare class member functions `const` whenever possible. Avoid `const` pass-by-value parameters in function declarations though, it is equivalent and cleaner to omit it. Still, declare the same parameter `const` in the function definition if applicable, it can catch unintended changes to the parameter inside the function. [Meyers05](#Meyers05) §3, [Sutter05](#Sutter05) §15

When const and non-const member functions have essentially identical implementations, code duplication can be avoided by having the non-const version call the const version. [Meyers05](#Meyers05) §3


### virtual

`virtual` declares a function overrideable in a derived class. Accessing an object's virtual function through a base class pointer will invoke the most derived version of that function.

a "pure virtual" function is one declared as `virtual f() = 0;` and forces derived classes to provide an implementation if they are not to be abstract classes themselves.

`virtual` can also be used before the name of the base class in a derived class definition to implement virtual inheritance, solving the diamond inheritance problem of having multiple base class versions of the same function.
Under virtual inheritance, a copy of the base class will not be included in the derived class and accessed directly. Instead, it will contain a (virtual) pointer to the base class so that further derived classes can inherit from multiple classes with the same base, and not get multiple, ambiguous implementations of inherited functions.
The constructor of a base class will not be called when inherited virtually, instead it will be called by the next class that inherits non-virtually from it.


### explicit

Prevents implicit type conversions on constructors taking exactly one argument without default values. By default, always use it in this context. [Sutter05](#Sutter05) §40


### static

The `static` keyword has multiple uses in C++, which can lead to a bit of confusion:

- Used before a variable or free function declaration at file scope, it defines internal linkage. The variable or function acts like a global, but only within that file.
- Used before a variable or function inside a class, it defines one shared copy of that member by all instances of a class. When combined with `const`, it can have an initializer.
- Used before a variable inside a function, that variable retains its state between function calls.
- A globally declared anonymous union must be declared static.


### using

Introduces a name that is declared elsewhere into the declarative region where the `using` declaration appears. A class definition defines a namespace, so `using` declarations inside a class definition only applies within that class.

`using` can be used to expose protected members of base classes as public members in derived classes:

    class Base
    {
    protected:
    
        int foo; // Base::foo is protected
    };
    
    Class Derived : public Base
    {
    public:
    
        using Base::foo; // Derived::foo is public
    };


### inline

Functions declared `inline` will make the compiler replace calls to it with a copy of the function body, replacing arguments with the call values. This can optimize performance while avoiding code repetition.

Limit most inlining to small, frequently called functions. This facilitates debugging and binary upgradability, minimizes potential code bloat, and maximizes the chances of greater program speed.
Don't second-guess the compiler and declare functions `inline` prematurely. Modern compilers do inlining automatically for most small functions not explicitly declared inline.
Don't declare function templates inline just because they appear in header files. [Meyers05](#Meyers05) §30


### template

When declaring template parameters, `class` and `typename` are interchangeable. Use `typename` to identify nested dependent type names, except in base class lists or as a base class identifier in a member initialization list. [Meyers05](#Meyers05) §42

In derived class templates, refer to names in base class templates via a `this->` prefix, via `using` declarations, or via an explicit base class qualification. [Meyers05](#Meyers05) §43


### friend

The `friend` keyword can be used to override access restrictions for other classes or functions.

**Example:**

    Class Foo
    {
    private:
    
        int data;
    
        friend class Bar;      // Bar will be able to access Foo::data
        friend int Baz::fun(); // Baz::fun() (not Baz in general) will be able to access Foo::data
        friend void ::fun()    // The global fun() will be able to access Foo::data
    };

It is common to use friend functions for operator overloading, where an overloaded operator must have access to the internals of the classes that are arguments to the operator.


### operator

`operator` is not a standalone keyword, but prefix to a function name defining an overloaded operator (e.g. `Foo operator+(Foo &other);`. This name is nothing more than a glorified function name, and the function may in fact be called using it normally, but also allows it to be used with a corresponding C++ operator's syntax (see *Operators*).

### throw

Causes the compiler to inject implicit try/catch blocks around the function body to enforce via run-time checking that the function does in fact throw only the specified exceptions.

As a rule of thumb, never ever use it, unless forced to when overriding a base class virtual function that uses it and you cannot remove that too! In that case, use the identical exception specification for the overriding function. [Meyers96](#Meyers96) §14, [Sutter05](#Sutter05) §75
    
C++11 has deprecated the `throw` keyword. The `noexcept` keyword was added to supersede the empty throw specification.


Functions with special semantics
--------------------------------

Functions that tie in to built-in features of C++ require extra special care because they will be expected to work in certain ways. If implemented well, they will make classes much more powerful, but if implemented badly they have the potential to cause unexpected or undefined behavior.


### The Big Four

The Big Four functions require special treatment because they will be implemented by the compiler if you do not provide an implementation of your own. Therefore, they should not be implemented if the default compiler implementation is correct, but this choice should always be documented to assure users of your class that you know this is the case and didn't just forget or was unaware of that fact. [Meyers05](#Meyers05) §5

Explicitly disallow the use of compiler-generated functions you do not want, by declaring them private and providing no implementation. [Meyers05](#Meyers05) §6


#### Constructor

For an [Exception class](#ExceptionClasses):
- Make impossible to fail.

For a [RAII class](#RAII):
- Allocate the resource in it (or do setup for lazy allocation later).

The default constructor is the one that takes no arguments. If possible, one should be defined by the class (must be done explicitly if other constructors are defined) because otherwise the class will not be usable in arrays and STL containers, and in virtual base classes the lack of one means all derived classes must explicitly define all the base class's arguments. [Meyers96](#Meyers96) §4

If the constructor can take exactly one argument (default values may allow this for multi-argument constructors), use the `explicit` keyword to prevent implicit type conversion (almost always unwanted). [Meyers96](#Meyers96) §5, [Sutter05](#Sutter05) §40

Initialize using the initialization list rather than in the constructor body, except if you perform unmanaged resource acquisition (such as `new` expressions not immediately passed to a smart pointer). [Meyers05](#Meyers05) §4, [Sutter02](#Sutter02) §18 [Sutter05](#Sutter05) §9 §48

Always initialize arguments in the initialization list in the same order as they are declared in the class. [Meyers05](#Meyers05) §4, [Sutter05](#Sutter05) §47

Always avoid calling virtual functions in constructors. [Sutter05](#Sutter05) §49

Prevent possible resource leaks in constructors by catching all possible exceptions during allocation and releasing previously allocated resources. The destructor will not be called if the constructor fails. [Meyers96](#Meyers96) §10 [Sutter02](#Sutter02) §18

Constructor function try blocks (i.e. try/catch around the initialization list + body) are useful to translate exceptions thrown by any sub-object constructor into a different exception, but they must (re)throw something. They are not useful for any other type of function. [Sutter02](#Sutter02) §18

If the class legally can have "optional" members that may throw during construction but should not prevent class construction, use the [Pimpl Idiom](#Pimpl) to group such individual members. [Sutter02](#Sutter02) §18

Making the constructor(s) private prevents object instantiation. A function made `friend` of the class, or defined `static` in the class, will be allowed to access the private constructor and can therefore be useful as a means to control object instantiation, by holding static instances inside it. [Meyers96](#Meyers96) §26


#### Destructor

For a [Value class](#ValueClasses):
- Make public.
- Make non-virtual.

For a [RAII class](#RAII):
- Release the resource in it.

For a [Base class](#BaseClasses):
- Implementation depends on if clients should be able to delete polymorphically using a pointer to the base class or not. [Meyers05](#Meyers05) §7, [Sutter02](#Sutter02) §27, [Sutter05](#Sutter05) §50
    - Make public for polymorphic deletion, private (protected) otherwise.
    - Make virtual for polymorphic deletion, non-virtual otherwise.
- If a class has any virtual functions, it should have a virtual destructor. [Meyers05](#Meyers05) §7
- Thinking in the future tense, it's usually best to prefer allowing polymorphic deletion, even if it is not currently required. [Meyers96](#Meyers96) §32

Destructors need to release resources allocated by the class in order to prevent leaks. [Meyers96](#Meyers96) §10

Destructors must always provide the no-fail guarantee. If a destructor calls a function that may throw, always wrap the call in a try/catch block that prevents the exception from escaping. [Meyers05](#Meyers05) §8, [Meyers96](#Meyers96) §11, [Sutter02](#Sutter02) §19 [Sutter05](#Sutter05) §51

If you write the destructor, you probably need to explicitly write or disable the copy constructor and copy assignment operator. [Sutter05](#Sutter05) §52

Always avoid calling virtual functions in destructors. [Sutter05](#Sutter05) §49

A destructor implies changing state outside of the object being destroyed. If the object is performing resource allocation, then the compiler generated/supplied methods will be wrong!


#### Copy constructor

For a [Value class](#ValueClasses):
- Make public.
- Make non-virtual.

For an [Exception class](#ExceptionClasses):
- Make impossible to fail.

If you write/disable the copy constructor, also do the same for the copy assignment operator [Sutter05](#Sutter05) §52.

Copy constructors and copy assignment operators should not implement copying in terms of one of the other. Instead, put common functionality in a third function that both call. [Meyers05](#Meyers05) §12

If you write the copy constructor, and allocate or duplicate some resource in it, you should also write a destructor that releases it. [Sutter05](#Sutter05) §52

Copy construction needs to be correct (and should be cheap) for Value classes in order to make them usable in standard containers. [Meyers01](#Meyers01) §3

Abstract base classes can define the copy constructor `explicit` to allow slicing but prevent it from being done by accident. [Sutter05](#Sutter05) §54

Abstract base classes can also define a pure virtual `clone()` function (otherwise known as a virtual copy constructor), returning a pointer to a (deep) copy of the object. This can be used by the normal copy constructor to avoid slicing. [Meyers96](#Meyers96) §25, [Sutter05](#Sutter05) §54

**Example:**

    class NonSliceableComponents
    {
    public:
    
        // Copy constructor
        NonSliceableComponents(const NonSliceableComponents& rhs) 
        {
            // Deep-copy all components
            for (list<NonSliceable*>::const_iterator it = rhs.components_.begin(); it != rhs.components_.end(); ++it)
            {
                components_.push_back(it->clone());
            }
        }
    
    private:
    
        std::vector<NonSliceableComponent*> components_;
    };
    
    class NonSliceableComponent
    {
    public:
    
        virtual NonSliceable* clone() const = 0;
    };
    
    class Component : public NonSliceableComponent
    {
    public:
    
        // Virtual copy constructor, using the normal copy constructor
        virtual Component* clone() const { return new Derived(*this); }
    }


#### Copy assignment operator

For a [Value class](#ValueClasses):
- Make public.
- Make non-virtual.

If you write/disable the copy assignment operator, also do the same for the copy constructor [Sutter05](#Sutter05) §52.

If you write the copy assignment operator, and allocate or duplicate some resource in it, you should also write a destructor that releases it. [Sutter05](#Sutter05) §52

The canonical form for copy assignment implementation is to provide a no-fail `swap()` function and implement copy assignment by creating a temporary, swapping and then returning (see *swap*). [Sutter02](#Sutter02) §22

Never write a copy assignment operator that relies on a check for self-assignment in order to work properly; a copy assignment operator that uses the create-a-temporary-and-swap idiom is automatically both strongly exception-safe and safe for self-assignment. It's all right to use a self-assignment check as an optimization to avoid needless work. [Sutter99](#Sutter99) §38


### swap

Provide a no-fail `swap()` function to efficiently and infallibly swap the contents of this object with another's. It has many potential uses (primarily in [Value classes](#ValueClasses)), e.g. to implement assignment easily while maintaining the strong exception guarantee for objects composed of other objects that provide the Strong Guarantee. [Meyers05](#Meyers05) §25, [Sutter99](#Sutter99) §12 [Sutter02](#Sutter02) §22 [Sutter05](#Sutter05) §56

If you offer a member `swap()`, also offer a non-member `swap()` that calls the member. For classes (not templates), specialize `std::swap()` too. When calling `swap()`, employ a `using` declaration for `std::swap()`, then call `swap()` without namespace qualification. It's fine to totally specialize `std` templates for user-defined types, but never try to add something completely new to `std`. [Meyers05](#Meyers05) §25

**Example implementation:**

    class Derived : public Base
    {
    private:
    
        U member1_;
        int member2_;
    
    public:
    
        // Member swap (first swap bases, then members)
        void swap(T& rhs) // noexcept
        {
            using std::swap;
            B::swap(rhs);                 // Swap base class members (user-defined)
            member1_.swap(rhs.member1_);  // User-defined, using member version
            swap(member2_, rhs.member2_); // Built-in, using std::swap
        }
    
        // Copy assignment operator utilizing swap (traditional)
        T& operator=(const T& other)
        {
            T temp(other);
            swap(temp);
            return *this;
        }
    
        // Copy assignment operator utilizing swap (pass by value, more elegant in C++03 but ambiguous together with move assignment operator in C++11)
        T& operator=(T temp)
        {
            using std::swap;
            swap(temp);
            return *this;
        }
    };
    
    // Non-member swap as std template specialization
    namespace std
    {
        template <>
        void swap(T& t1, T& t2)
        {
            t1.swap(t2);
        }
    }
    
    // Template example
    template<typename T>
    class X
    {
        ...
        void swap(X<T>& rhs)
        {
            ...
        }
    };
    
    template <typename T>
    void swap(X<T>& x1, X<T>& x2)
    {
        x1.swap(x2);
    }


### Operators

By default, avoid providing implicit conversion operators of the form `operator Type()`. Instead, use named functions (like `std::string`'s `c_str()` function). [Sutter05](#Sutter05) §40

When overloading operators, provide multiple versions with different argument types, when applicable, to prevent wasteful implicit type conversions (e.g. for string comparison) [Sutter05](#Sutter05) §29


#### operator=

When implementing the assignment operator, make it non-virtual and with a specific signature, returning a reference to *this: [Meyers05](#Meyers05) §10, [Sutter05](#Sutter05) §55
- Classic version: `T& operator=(const T&);`
- Optimizer-friendly: `T& operator=(T);`

Ensure that operator= is safe for self-assignment. [Meyers05](#Meyers05) §11, [Sutter05](#Sutter05) §55

Explicitly invoke all base assignment operators and assign all data members. [Sutter05](#Sutter05) §55

Don't return const T&, because it prevents objects from being placed in std containers. [Sutter05](#Sutter05) §55


#### Binary arithmetic and bitwise operators

The binary arithmetic operators are:
- `operator+`
- `operator-`
- `operator*`
- `operator/`
- `operator%`
- `operator&`
- `operator|`
- `operator^`
- `operator<<`
- `operator>>`

If implementing one of these, implement its corresponding compound assignment operator as well, and implement it in terms of that (except if that modifies the left-hand side so significantly that it is more advantageous to do the reverse, e.g. in the case of multiplication of complex numbers). [Meyers96](#Meyers96) §22, [Sutter05](#Sutter05) §27


#### Compound assignment operators

The compound assignment operators are:
- `operator+=`
- `operator-=`
- `operator*=`
- `operator/=`
- `operator%=`
- `operator&=`
- `operator|=`
- `operator^=`
- `operator<<=`
- `operator>>=`


#### Increment and decrement operators

The prefix versions (prefer to call these when possible):

    T& T::operator++()
    {
        // perform increment
        return *this;
    }
    
    T& T::operator--()
    {
        // perform decrement
        return *this;
    }

The postfix versions (implemented in terms of the prefix versions): [Meyers96](#Meyers96) §6, [Sutter99](#Sutter99) §20 [Sutter05](#Sutter05) §28

    T T::operator++(int)
    {
        T old(*this);
        ++*this;
        return old;
    }  

    T T::operator--(int)
    {
        T old(*this);
        --*this;
        return old;
    }

Prefer use of the prefix versions since they do not require temporaries. [Meyers96](#Meyers96) §6

Don't return const in the postfix versions (used to be good advice but in C++11 this means you cannot take full advantage of rvalue references!)


#### Streaming operators

The streaming operators, `operator<<` and `operator>>` are always implemented as non-member functions. [Sutter05](#Sutter05) §57


#### operator[]

The index operator `operator[]` is often used to provide array-like access syntax for user-defined classes. STL implements `operator[]` in the `std::string` and `std::map` classes. `std::string` simply returns a character reference as a result of `operator[]` whereas `std::map` returns a reference to the value given its key. In both cases, the returned reference can be directly read or written to.
The `std::string` and `std::map` classes have no knowledge or has no control over whether the reference is used for reading or for modification. Sometimes, however, it is useful to detect how the value is being used. In this case, `operator[]` is often implemented to distinguish reads from writes by returning a proxy object. [Meyers96](#Meyers96) §30

#### operator&, operator||, operator,

Never overload these operators! [Meyers96](#Meyers96) §7, [Sutter05](#Sutter05) §30


#### operator new

Allocates enough memory to hold an object of the given type, and calls a constructor to initialize it. Can be overloaded to change the way memory gets allocated. [Meyers96](#Meyers96) §8

There are many valid reasons for writing custom versions of new and delete, including improving performance, debugging heap usage errors, and collecting heap usage information. [Meyers05](#Meyers05) §50

If called like a function (`operator new(size)`), C++ will only allocate memory, not call the object constructor. [Meyers96](#Meyers96) §8

There are three forms of operator new: "plain new", "placement new" and "nothrow new". If implementing your own `operator new`, always provide/implement all three forms. [Meyers05](#Meyers05) §52, [Sutter05](#Sutter05) §46

Always implement together with `operator delete`, except for the special in-place form. [Sutter99](#Sutter99) §36 [Sutter05](#Sutter05) §45

`operator new` should contain an infinite loop trying to allocate memory, should call the new-handler if it can't satisfy a memory request, and should handle requests for zero bytes. Class-specific versions should handle requests for larger blocks than expected. [Meyers05](#Meyers05) §51

Always explicitly declare `operator new` and `operator delete` as static functions. They are never non-static member functions. [Sutter99](#Sutter99) §36


##### plain new:

Should return a pointer to raw, uninitialized memory.

    void* operator new(std::size_t); 


##### placement new:

Used when there is already allocated memory that should be assigned to the object.

When you write a placement version of operator new, be sure to write the corresponding placement version of `operator delete`. If you don't, your program may experience subtle, intermittent memory leaks. [Meyers05](#Meyers05) §52

    void* operator new(std::size_t, void* p)
    {
        return p;
    }

**Usage example:**

    class Widget
    {
    public:
    
        Widget(int widgetSize);
        ...
    };
    
    Widget* constructWidgetInBuffer(void *buffer, int widgetSize)
    {
        return new (buffer) Widget(widgetSize); // Using placement new constructor.
    }


##### nothrow new

Nothrow new is of limited utility, because it applies only to memory allocation; associated constructor calls may still throw exceptions. [Meyers05](#Meyers05) §49

    void* operator new(std::size_t, std::nothrow_t) throw(); // nothrow in C++11


#### operator delete

Always implement together with `operator new`. [Sutter05](#Sutter05) §45

Never allow operator delete to fail. [Sutter05](#Sutter05) §51

operator delete should do nothing if passed a pointer that is null. Class-specific versions should handle blocks that are larger than expected. [Meyers05](#Meyers05) §51


#### operator new[]

Always implement together with `operator delete[]`. [Sutter05](#Sutter05) §45

Always implement all three forms. [Sutter05](#Sutter05) §46

#### operator delete[]

Always implement together with `operator new[]`. [Sutter05](#Sutter05) §45

Never allow `operator delete[]` to fail. [Sutter05](#Sutter05) §51


#### Implicit type conversion operators

Operators that take the form `operator typename() const;`, where "typename" is the name of a type (e.g. `double`).

Usually this type of operator should be avoided, because it may lead to unexpected and unwanted uses, which are difficult to diagnose. [Meyers96](#Meyers96) §5


<a name="CppIdioms"></a>
C++ idioms
----------

<a name="RAII"></a>
#### RAII

Any class that allocates a resource that must be released after use (heap memory, file handle, thread, socket etc.) should be implemented using the RAII Idiom (Resource Acquisition Is Initialization):
- Perform allocation/acquisition in the constructor (or lazily at the first method call that needs it). [Meyers96](#Meyers96) §17
- Perform deallocation/release in the destructor.
- Either provide a copy constructor and copy assignment operator with valid resource copying semantics or disable both (by making them private and non-implemented). [Meyers05](#Meyers05) §13 §14, [Sutter05](#Sutter05) §13

APIs often require access to raw resources, so RAII classes should provide some means of access to the raw resources they manage. [Meyers05](#Meyers05) §15

**Example implementation**:

    class ResourceManager
    {
    private:
    
        Resource* resource_; // The managed resource
    
    public:
    
        ResourceManger()
        {
            resource = new Resource(); // Acquisition
        }
    
        ~ResourceManager()
        {
            delete resource; // Release
        }
    
        Resource* get() const
        {
            return resource_; // Access to raw resource
        }
    
    private:
    
        Resource(const Resource&); // No implementation to disable copying
        T& operator=(const Resource&); // No implementation to disable copying
    }


<a name="Pimpl"></a>
### Pimpl

When it makes sense to completely hide internal implementation, the Pimpl (Pointer to implementation) idiom should be used. It minimizes compiler dependencies, separates interface from implementation and adds portability. The downside is that it adds complexity. [Sutter05](#Sutter05) §43

Pimpl is useful to suppress class member constructor exceptions when the class' own constructor should never be allowed to throw. [Sutter02](#Sutter02) §18

**Example implementation:**

    // Foo.hpp:
    class Foo
    {
    public:
    
        void bar();
    
    private:
    
        struct FooImpl;
        shared_ptr<FooImpl> pimpl_;
    };

    // Foo.cpp:
    struct FooImpl()
    {
        void bar()
        {
            // Do something, the actual implementation
        }
    };
    
    void Foo::bar()
    {
        pimpl_->bar(); // Calling the implementation
    }


<a name="CRTP"></a>
### Curiously Recurring Template Pattern

The Curiously Recurring Template Pattern (CRTP) is useful to extract out type-independent but type-customizable functionality in a base class and to mix in that interface/property/behavior into a derived class, customized for the derived class.

This is achieved by creating classes inheriting from a base class template, specializing it on their own type.

**Example implementation:**

    template <typename Derived>
    struct Base
    {
        void interface()
        {
            ...
            static_cast<Derived*>(this)->implementation();
            ...
        }
    
        static void static_interface()
        {
            ...
            Derived::static_implementation();
            ...
        }
     
        // The default implementation may be (if exists) or should be (otherwise) overriden by inheriting in derived classes (see below)
        void implementation();
        static void static_implementation();
    };
    
    // The Curiously Recurring Template Pattern (CRTP)
    struct Derived1 : Base<Derived1>
    {
        // This class uses Base's variant of implementation()
        //void implementation();
    
        // ... and overrides static_implementation()
        static void static_implementation();
    };
    
    struct Derived2 : Base<Derived2>
    {
        // This class overrides implementation()
        void implementation();
    
        // ... and uses Base's variant of static_implementation()
        //static void static_implementation();
    };


<a name="Visitor"></a>
### Visitor Pattern

The Visitor Pattern allows adding functionality to a set of classes without "polluting" them with a lot of extra responsibilities, and without having to perform type checking to call the right function for every type. The code for a specific functionality will also be localized to one place.

**Example implementation:**

    // A simple class hierarchy:
    
    class Base {};
    
    class Foo : public Base {};
    
    class Bar : public Base {};
    
    class Baz : public Base {};

Now, to add a `log()` functionality to all derived classes, first add an `accept(Visitor)` function to the hierarchy:
    
    class Base
    {
    public:
    
        virtual void accept(Visitor& v) = 0;
    };
    
    class Foo : public Base
    {
    public:
    
        virtual void accept(Visitor& v) { v.visit(this); }
    };
    
    class Bar : public Base
    {
    public:
    
        virtual void accept(Visitor& v) { v.visit(this); }
    };
    
    class Baz : public Base
    {
    public:
        virtual void accept(Visitor& v) { v.visit(this); }
    };

Next, create a `Visitor` [Base class](#BaseClasses) with a pure virtual `visit()` method for each `Base` type:

    class Visitor
    {
    public:
        virtual void visit(Foo* b) = 0;
        virtual void visit(Bar* b) = 0;
        virtual void visit(Baz* b) = 0;
    };

Finally, create a `LogVisitor` derived class for each derived class with the implementation (we could add more Visitors like this when we need other new functions):

    class LogVisitor : public Visitor
    {
    public:
        void visit(Foo* b)
        {
            cout << "Logging" << b->fooFunction();
        }
    
        void visit(Bar* b)
        {
            cout << "Logging" << b->barFunction();
        }
    
        void visit(Baz* b)
        {
            cout << "Logging" << b->bazFunction();
        }
    };

Usage example:

    void doLogging(Base* b)
    {
        LogVisitor v; // The Visitor we use for logging
        b->accept(v); // Do the logging on whatever derived type object we got
    }


<a name="NVI"></a>
### NVI

Classes designed using the NVI pattern (Non-Virtual Interface) can be useful for perform pre-post operations on code fragments, such as invariant checking, locks etc.

**Example implementation:**

    class Base
    {
    private:
    
        ReaderWriterLock lock_;
        SomeComplexDataType data_;
    
    public:
    
        void read_from(std::istream& in) // Note: non-virtual
        {
            lock_.acquire();
            assert(data_.check_invariants() == true); // must be true
            read_from_impl(in);
            assert(data_.check_invariants() == true); // must be true
            lock_.release();
        }
    
        void write_to(std::ostream& out) const // Note: non-virtual
        {
            lock_.acquire();
            write_to_impl(out);
            lock_.release();
        }
    
        virtual ~Base() {} // Virtual because Base is a polymorphic base class.
    
    private:
    
        virtual void read_from_impl(std::istream&) = 0;
        virtual void write_to_impl(std::ostream&) const = 0;
    };
    
    class XMLReaderWriter : public Base
    {
    private:
    
        virtual void read_from_impl (std::istream&) // Note: Not part of the client interface!
        {
            // Read XML.
        }
    
        virtual void write_to_impl (std::ostream&) const // Note: Not part of the client interface!
        {
            // Write XML.
        }
    };
    
    class TextReaderWriter : public Base
    {
    private:
    
        virtual void read_from_impl (std::istream&) { ... }
        virtual void write_to_impl (std::ostream&) const { ... }
    };


<a name="EBO"></a>
### Empty Base Optimization

In order to ensure object identity, empty classes in C++ don't take 0 bytes of memory, but C++ allows compilers to optimize memory usage when inheriting from them to ensure that they don't take any extra space. `boost::compressed_pair` is an example of using the Empty Base Optimization to save space.

**Example implementation:**

    class E1 {};
    class E2 {};
    
    // without EBCO
    class Foo
    {
        E1 e1;
        E2 e2;
        int data;
    }; // sizeof(Foo) = 8
    
    // with EBCO
    class Bar : private E1, private E2
    {
        int data;
    }; // sizeof(Bar) = 4


References
----------

<a name="Meyers05"></a>
[Meyers05]
"Effective C++", ISBN 0-321-33486-7

<a name="Meyers96"></a>
[Meyers96]
"More Effective C++", ISBN 0-201-63371-X

<a name="Meyers01"></a>
[Meyers01]
"Effective STL", ISBN 0-201-74962-5

<a name="Sutter99"></a>
[Sutter99]
"Exceptional C++", ISBN 0-201-61562-2

<a name="Sutter02"></a>
[Sutter02]
"More Exceptional C++", ISBN 0-201-70434-X

<a name="Sutter05"></a>
[Sutter05]
"C++ coding standards - 101 Rules, Guidelines, and Best Practices", ISBN 0-321-11358-6

[Generic Programming in C++](http://www.generic-programming.org/languages/cpp/)

[More C++ Idioms](http://en.wikibooks.org/wiki/More_C++_Idioms)
