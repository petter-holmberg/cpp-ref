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
