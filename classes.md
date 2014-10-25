C++ classes quick reference
===========================


General class design guidelines
-------------------------------

When deciding how to design a C++ class, there are several considerations to take into account:


### Interface vs. implementation

A class's interface is made up of the public (and protected) functions and data members it exposes.

A class's implementation is encapsulated within the class, and includes the private functions and data members.

When designing a class, follow the Law of Second Chances: [Sutter05](#Sutter05) §36
- The most important thing to get right is the interface. Everything else can be fixed later. Get the interface wrong and you may never be allowed to fix it.

When designing a class, follow the Dependency Inversion Principle (DIP) and prefer to provide abstract interfaces: [Sutter05](#Sutter05) §36
- High-level modules should not depend on low-level modules. Rather, both should depend upon abstractions.
- Abstractions should not depend upon details. Rather, details should depend upon abstractions.

Make interfaces easy to use correctly and hard to use incorrectly: [Meyers05](#Meyers05) §18
- Good interfaces are easy to use correctly and hard to use incorrectly. Strive for these characteristics in all interfaces.
- Ways to facilitate correct use include consistency in interfaces and behavioral compatibility with built-in types.
- Ways to prevent errors include creating new types, restricting operations on types, constraining object values, and eliminating client resource management responsibilities.

By default, all data members should be private (except if the class is an *aggregate*) and accessible only through member functions. [Sutter05](#Sutter05) §41

By default, classes should never provide functions that return handles to internal (private) members, allowing external code to modify them (even const is not safe because it can be cast away). The exception is if compatibility with existing (legacy) code is necessary, and then the reasoning behind these exceptions should be clearly documented. [Sutter05](#Sutter05) §42


### Composition vs. inheritance

Compositioning is to embed a member variable of one class within another class, and use the member in ways that make sense to the outer class (like providing passthrough functions to some of the inner class' interface).

Composition has meanings completely different from that of public inheritance. In the application domain, composition means "has-a", in the implementation domain, it means "is-implemented-in-terms-of". [Meyers05](#Meyers05) §38

Prefer composition to implementing base classes, it minimizes coupling. [Sutter02](#Sutter02) §23, [Sutter05](#Sutter05) §34


### Inheritance guidelines

Inheritance in C++ comes in three flavors:

1.  Public inheritance

    Public inheritance, models the "is-a" (or better, "works-like-a") ...or "is-implemented-in-terms-of" relationship. Everything that applies to base classes must also apply to derived classes, because every derived class object is a base class object. [Meyers05](#Meyers05) §32

    Names in derived classes hide names in base classes. Under public inheritance, this is never desirable. To make hidden names visible again, employ "using" declarations or forwarding functions. [Meyers05](#Meyers05) §33

2.  Private inheritance

    Private inheritance, models the "has-a", or "is-implemented-in-terms-of" relationship. It's usually inferior to composition, but it makes sense when a derived class needs access to protected base class members or needs to redefine inherited virtual functions. [Meyers05](#Meyers05) §39

    Unlike composition, private inheritance can enable the *Empty Base Optimization*. {19} This can be important for library developers who strive to minimize object sizes. [Meyers05](#Meyers05) §39

3.  Protected inheritance

    Under protected inheritance, public members of the base class becomes protected under the derived class.

Use private inheritance instead of composition only when absolutely necessary, which means when: [Sutter99](#Sutter99) §15
- You need access to the class's protected members.
- You need to override a virtual function.
- The object needs to be constructed before other base subobjects.

Never use public inheritance except to model true Liskow IS-A and WORKS-LIKE-A. All overridden member functions must require no more and promise no less. [Sutter99](#Sutter99) §22

Never inherit from a class that was not designed to be a base class. To add more functionality to a concrete class, define free (nonmember) functions to do the job instead, and put them in the same namespace as the class they are designed to extend. [Sutter05](#Sutter05) §35

Never inherit publicly only because it allows you to reuse common code in the base class! Instead, inherit because it allows the derived class to be reused by code that uses objects of the base class polymorphically. Before object orientation, it has always been easy for new code to call existing code. Public inheritance specifically makes it easier for existing code to seamlessly and safely call new code. (So do templates, which provide static polymorphism that can blend well with dynamic polymorphism.) [Sutter99](#Sutter99) §22 [Sutter05](#Sutter05) §37

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
    This can be necessary when the used class provides a lock of some sort, such as a critical section or a database transaction, which must cover the entire lifetime of another base subobject.
-   You need to share a common virtual base class, or override the construction of a virtual base class (non-public inheritance).
    The first part applies if the using class has to inherit from one of the same virtual bases as the used class. If it does not, the second part may still apply: The most-derived class is responsible for initializing all virtual base classes, and so if you need to use a different constructor or different constructor parameters for a virtual base, then you must inherit.
-   You benefit from the Empty Base Optimization and it matters to the program (non-public inheritance).
-   You need controlled polymorphism (non-public inheritance).
    Public inheritance should always model "is-a" as per the Liskov Substitution Principle (LSP). Non-public inheritance can express a restricted form of "is-a", even though most people identify "is-a" with public inheritance alone. Given "class Derived : private Base", from the point of view of outside code, a Derived object "is-not-a" Base, and so of course can't be used polymorphically as a Base because of the access restrictions imposed by private inheritance. However, inside Derived's own member functions and friends only, a Derived object can indeed be used polymorphically as a Base (you can supply a pointer or reference to a Derived object where a Base object is expected), because members and friends have the necessary access.
    If instead of private inheritance you use protected inheritance, then the "is-a" relationship is additionally visible to further-derived classes, which means subclasses can also make use of the polymorphism.

When overriding a virtual base class function: [Sutter05](#Sutter05) §38
- Make the overriding function virtual as well.
- Make the overriding function respect the same pre- and post-conditions as the base class function. (overrides should never require more or provide less than the original function.)
- When overriding, never change the value of default arguments. [Meyers05](#Meyers05) §37
- If the base class has multiple functions with the same name but different signatures, overriding just one hides all the others in the derived class. To bring them back into scope in the derived class, add a "using BaseClass::functionName" declaration in the derived class.

Try to avoid multiple inheritance of more than one class that is not an abstract interface. [Meyers05](#Meyers05) §40, [Sutter02](#Sutter02) §24, [Sutter05](#Sutter05) §36
- Multiple inheritance is more complex than single inheritance. It can lead to new ambiguity issues and to the need for virtual inheritance.
- Virtual inheritance imposes costs in size, speed, and complexity of initialization and assignment. It's most practical when virtual base classes have no data.
- Multiple inheritance does have legitimate uses. One scenario involves combining public inheritance from an Interface class with private inheritance from a class that helps with implementation.


Class types
-----------

C++ classes come in many flavors, and depending on which one you want to implement, different rules apply. Without being clear about what type of class you want, you are not likely to implement it correctly. [Sutter05](#Sutter05) §32

A class shold neatly fall into one of these categories. If it doesn't it is probably a sign that the class is doing too much and should be divided into multiple classes.


### Value classes

Value classes are intended to be used as concrete classes, not as base classes. They should be modelled after built-in types and usually be instantiated on the stack or as directly held members of other classes. [Sutter05](#Sutter05) §32

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


#### Aggregates

Aggregates are a special case of value classes that also have:
- No user-declared constructors.
- No private or protected non-static data members.
- No base classes.
- No virtual functions.

They hold a collection of values, and don't pretend to encapsulate or provide any behavior. C-style structs are aggregates in C++. So are arrays.

Example: `std::pair`

**Implementation:**

    struct Aggregate
    {
        std::string member1;
        SomeClass member2;
    };

By default, reserve usage of the "struct" keyword for aggregates, because it requires no extra syntax for them and makes the intent clear.

Aggregates can be instantiated with curly braces:

    Aggregate a1 = {};
    Aggregate a2 = {1, 2};


#### POD-structs

POD-structs (Plain Old Data) are a special case of aggregates that also have:
- No non-static data members that are non-POD-structs, non-POD-unions (or arrays of such types) and no reference members.
- No user-defined assignment operator.
- No user-defined destructor.


**Implementation:**

    struct PodStruct
    {
        int member1;
        AnotherPodStruct member2[3];
    };

POD-structs are typically compatible with C-style structs.


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

Base classes with a high cost of change should have public functions that are nonvirtual rather than (non-pure) virtual. Virtual functions should be made private, or protected if derived classes need to be able to call them (see [NVI](#NVI)). Destructors are an exception to this rule. [Sutter05](#Sutter05) §39

Base classes should always be abstract if they don't need to be used as leaf classes in an inheritance hierarchy. If a base class is concrete because some aspects of it require it to be, while it's also used to implement common functionality of derived classes, separate out the common parts into an abstract base class and make the original base class inherit from that along with the other leaf classes. [Meyers96](#Meyers96) §33

Normally, base class destructors should be public and virtual. To prevent polymorphic deletion through base class pointers, make the destructor non-virtual and protected. [Sutter02](#Sutter02) §27

Not all base classes are designed to be used polymorphically. Examples are the standard container types. As a result, they don't need virtual destructors. [Meyers05](#Meyers05) §7


### Abstract interfaces

Abstract interfaces are classes made up entirly of (pure) virtual functions and containing no state. Usually they don't have any member function implementations.

**Implementation:**

    class Interface
    {
    public:
    
        // Public virtual destructor to allow polymorphic deletion. [Sutter02](#Sutter02) §27
        virtual ~Interface () = 0;
    
        // Only pure virtual interface functions.
        virtual SomeType interfaceFunction() = 0;
    
        // No data members!
    };


### Traits classes

Traits classes are templates that carry information about types. They contain only typedefs and static functions, and have no modifiable state or virtual functions. Traits classes make information about types available during compilation (as opposed to virtual functions or function pointers). In conjunction with overloading, traits classes make it possible to perform compile-time if...else tests on types. [Meyers05](#Meyers05) §47, [Sutter02](#Sutter02) §4 [Sutter05](#Sutter05) §32

> Think of a trait as a small object whose main purpose is to carry information used by another object or algorithm to determine "policy" or "implementation details"
> -- <cite>Bjarne Stroustrup</cite>

Traits classes can solve the problem of generalizing functions on different types when templates alone aren't sufficient (because of different behavior) and you cannot use class polymorphism (because the types are primitive). Traits classes rely on explicit template specialization for different types.

Examples: `std::iterator_traits`, `std::numeric_limits`

**Example implemenations:**

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

A functor is a class modelled after function pointers. The convention in STL is to pass functors by value. Therefore, they should be lightweight and have valid copy-semantics. They also need to be monomorphic, i.e. not use virtual functions. State and polymorphism can be implemented using the [Pimpl idiom](#Pimpl). [Meyers01](#Meyers01) §38

Functors that are predicates (i.e. return bool or something that can be implicitly converted into bool) should be pure functions, i.e. their return value should only depend on the input values and not some internal state. Defining `operator()` `const` is necessary for predicates, but internally they must also avoid accessing mutable data members, non-const local static objects, non-const objects at namespace scope and non-const global objects. The C++ standard doesn't guarantee that stateful predicates will work with standard algorithms! [Meyers01](#Meyers01) §39 [Sutter02](#Sutter02) §3

Functors that are made adaptable can be used in many more contexts than functors that are not. Making them adaptable simply means to define some of the typedefs `argument_type`, `first_argument_type`, `second_argument_type`, and `result_type`. The conventional way to do so is to inherit from `std::unary_function` or `std::binary_function`, depending on if the functor takes one or two arguments. These base structs are templates, taking either two or three types. The last one is the return type of `operator()`, the first one(s) are its argument type(s). When the input parameters are `const` and not pointers, it is conventional to strip off const qualifiers for these types, while for pointers the `const` should be kept. Adaptable functors do not define more than one operator() function. [Meyers01](#Meyers01) §40

**Example implemenations:**

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
    
    // Heavy polymorphic functor implemented using Pimpl
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

Such a function can accept both function pointers (`&fun`), functors (`fun()`) and lambdas (`[]() { ... }`)).


### Exception classes

Exception classes can be implemented to define domain-specific exceptions.

Exceptions should be thrown by value and be caught by (const) reference. If re-thrown, it should be done with `throw;` [Meyers96](#Meyers96) §13, [Sutter05](#Sutter05) §73

**Example implemenation:**

    class ExceptionClass : public std:exception
    {
    public:
    
        // No-fail constructor
        ExceptionClass() {}
    
        // No-fail copy constructor (throwing from this would abort the program!) [Sutter05](#Sutter05) §32
        ExceptionClass(const ExceptionClass& rhs) {}
    
        // Destructor
        ~ExceptionClass() throw() {};
    
        // Virtual functions, often implements Cloning and the Visitor pattern [Sutter05](#Sutter05) §54
    }


### Policy classes

Policy classes (normally templates) are fragments of pluggable behavior. They are not usually instantiated standalone but as a base or member of another class. They may or may not have state or virtual functions.

In brief, policy-based class design fosters assembling a class (called the host) with complex behavior out of many little classes (called policies), each of which takes care of only one behavioral or structural aspect. As the name suggests, a policy establishes an interface pertaining to a specific issue. You can implement policies in various ways as long as you respect the policy interface.

Because you can mix and match policies, you can achieve a combinatorial set of behaviors by using a small core of elementary components.

**Implementation example:**

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
    
        // Behaviour method
        void run() const
        {
            // Two policy methods
            print(message());
        }
    };
    
    typedef HelloWorld<OutputPolicyWriteToCout, LanguagePolicyEnglish> HelloWorldEnglish;
    HelloWorldEnglish hello_world;
    hello_world.run(); // prints "Hello, World!"
    
    typedef HelloWorld<OutputPolicyWriteToCout, LanguagePolicyGerman> HelloWorldGerman;
    HelloWorldGerman hello_world2;
    hello_world2.run(); // prints "Hallo Welt!"


### Mixin classes

A C++ mixin class is a template class that is parameterized on its base class, implementing some specific fragment of functionality. It is intended to be composed with other classes.

**Implementation example:**

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

Ancillary classes support specific idioms. They should be easy to use correctly and hard to use incorrectly.


#### RAII classes

Any class that allocates a resource that must be released after use (heap memory, file handle, thread, socket etc.) should be implemented using the RAII idiom (Resource Acquisition Is Initialization): Perform allocation in the constructor (or lazily at the first method call that needs it [Meyers96](#Meyers96) §17), deallocation in the destructor, and either provide a copy constructor and copy assignment operator with valid resource copying semantics or disable both (by making them private and non-implemented). [Meyers05](#Meyers05) §13 §14, [Sutter05](#Sutter05) §13

APIs often require access to raw resources, so RAII classes should provide some means of access to the raw resources they manage. [Meyers05](#Meyers05) §15

**Implementation example**:

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
    
        Resource(const Resource&); // No implemenation to disable copying
        T& operator=(const Resource&); // No implemenation to disable copying
    }


Class-related keywords
----------------------


Functions with special semantics
--------------------------------


C++ idioms
----------


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
