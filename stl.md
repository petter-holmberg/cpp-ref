C++ STL quick reference
=======================

This document is a collection of design guidelines when using the C++ Standard Template Library.
For brevity and simplicity of use, it does not discuss at length the *whys* of the guidelines, but instead presents references to further reading about the subject.

The guidelines are currently focused on C++98 use but features some notes about later versions.


STL overview
------------

STL is mainly based on three fundamental concepts: [Meyers96](#Meyers96) §35

1. [Containers](#Containers): Classes for holding collections of objects.
2. [Iterators](#Iterators): A mechanism for traversing containers and accessing held objects in them.
3. [Algorithms](#Algorithms): Functionality for performing operations on objects in containers, using iterators to traverse them.


<a name="Containers"></a>
Containers
----------

The STL containers can be subdivided into two categories:
1.  *Sequence containers:*
    Containers oriented towards sequential access of objects.
    The standard sequence containers are `vector`, `string`, `deque` and `list`.
2.  *Associative containers:*
    Containers oriented towards random access of objects.
    The standard associative containers are `set`, `multiset`, `map` and `multimap`.

Code should never be written with the goal to generalize the use of a specific container, so that it can be replaced without touching any code that uses it.
Instead, find the container that best matches the required use cases, and use typedefs to clairfy syntax and make container replacement easier in the event that it needs to be done. [Meyers01](#Meyers01) §2

Containers should never contain base classes, as this causes slicing (only the base part of derived classes being inserted) which is almost always an error.
To implement containers holding different types, make containers of base class pointers, or preferably of smart pointers (but never auto_ptr!)
Containers of `new`:ed pointers are prone to leak memory. The best way to avoid this problem is to use smart pointers. [Meyers01](#Meyers01) §3 §7 §8

`vector` is usually the right container to use by default when it is not obvious that another one should be used, it offers a lot of useful properties that other containers don't provide. [Sutter05](#Sutter05) §76

A *Sequence container* is the right option when element position matters, especially for insertion. Otherwise, an *Associative container* is a viable option. [Meyers01](#Meyers01) §1.

It is also useful to categorize containers in terms of memory layout: [Meyers01](#Meyers01) §1
1.  *Contiguous memory containers:*
    Containers that store multiple elements in chunks of contiguous memory.
    These containers are very efficient for ordered access of elements, but can be inefficient when it comes to insertion and deletion of elements.
    The contiguous memory containers are `vector`, `string`, and `deque`.
2.  *Node-based containers:*
    Containers that store individual elements per chunk of memory.
    These containers are very efficient at insertion and deletion, but can be inefficient when it comes to access.
    The node-based containers are `list` and all associative containers.

If it is important to avoid movement of existing container elements when inserting or erasing elements, A *Node-based container* is the only option. [Meyers01](#Meyers01) §1

If it is necessary to have the option to reliably revert insertions and erasures, a *Node-based container* is the only option. For multiple-element insertions, the only option is `list`. [Meyers01](#Meyers01) §1

If iterator, pointer and reference invalidation is a big concern, *Node-based containers* are the only ones that never invalidate them (other than ones pointing to the elements being erased). [Meyers01](#Meyers01) §1

STL containers are not thread-safe by default. The standard suggests that:
- Having multiple readers into the same container should be thread-safe.
- Having multiple writers to different containers should be thread-safe.
Not all implementations respect even these guarantees. [Meyers01](#Meyers01) §12

All standard containers implement the Strong Guarantee for all operations, with two exceptions: [Sutter99](#Sutter99) §18
1. Multi-element inserts ("iterator range" inserts) are never strongly exception-safe.
2. For `vector<T>` and `deque<T>`, all inserts and erases are strongly exception-safe only as long as T's copy constructor and assignment operator do not throw; inserting into a `vector<string>` or a `vector<vector<int>>`, for example, are not strongly exception-safe.
This means that classes having container members and using the aforementioned operations must do the work themselves to ensure that their state is predictable if exceptions occur.
To do this, insert and erase in a copy of the container, then use `swap()` to switch over to using the new version after the copy-and-change steps have succeeded.


### Sequence containers

The standard sequence containers are `vector`, `string`, `deque` and `list`.


#### vector

`vector` is the only standard container with a C-compatible layout (same as an array) and is therefore the only option to use when interfacing with a C library.

`vector` should always be preferred to a dynamically allocated array. To pass a vector safely to a C API that expects an array, do it like this: [Meyers01](#Meyers01) §13 §16

    void cApiFunction(const int* ints, size_t numInts);
    if (!v.empty()) // &v[0] undefined if vector is empty
        cApiFunction(&v[0], v.size());

Never use `vector<bool>` as a container, instead use `deque<bool>` or a non-STL alternative like `bitset<bool>`. [Meyers01](#Meyers01) §18 [Sutter02](#Sutter02) §6


#### string

`string` should always be preferred to a dynamically allocated char array. [Meyers01](#Meyers01) §13

`string` can be implemented in many different ways, so inferring something about its memory use and performance compared to char arrays without consulting the specific implementation's documentation is a bad idea. [Meyers01](#Meyers01) §15

Many `string` implementations use reference counting. If this is a problem, a `vector<char>` can be used instead. [Meyers01](#Meyers01) §13

Using `swap()` on strings invalidates iterators, pointers and references to them. [Meyers01](#Meyers01) §1

Using `c_str()` is the only safe and correct way to pass strings to a C API that expects a char array. [Meyers01](#Meyers01) §16


#### deque

While `vector` should be the default sequence container of choice, `deque` should be used when most insertions and deletions take place at the beginning or at the end of the sequence.
It offers constant-time `insert()` and `erase()` operations at both ends, uses memory in an operating system-friendly way (large deques can be split in multiple blocks of memory of a suitable size), is somewhat easier to use and inherently more efficient for growth. [Sutter02](#Sutter02) §7


### Associative containers

The standard associative containers are `set`, `multiset`, `map` and `multimap`. They all store their elements sorted, typically using some kind of balanced binary tree.

Hash table based versions (which store objects unsorted and can provide amortized constant-time complexity instead of logarithmic) are not part of STL (prior to C++11) but there are several libraries that provide them. [Meyers01](#Meyers01) §25

Once a key has been inserted into an associative container, that key must never change its relative position in the container. [Sutter02](#Sutter02) §8

The standard associative containers are optimized for a mixed combination of inserts, erasures, and lookups. But many usage scenarios look more like this:
1. Setup phase, consisting mainly of many inserts (and possibly erasures)
2. Lookup phase, consisting mainly of lookups (bulk of the time spent here!)
3. Reorganize phase, modifying/replacing data, then returning to lookup again (if needed at all)
In this type of scenario, replacing the container with a sorted `vector` is likely to improve both memory usage and speed considerably (due to caching), assuming that the lookup phase contains only lookups. [Meyers01](#Meyers01) §23

**Example (vector used as a set):**

    vector<Widget> vw;
    
    // Insertion phase
    sort(vw.begin(), vw.end());
    Widget w; // Object to look up
    
    // Lookup using lower_bound
    vector<Widget>::iterator i = lower_bound(vw.begin(), vw.end(), w);
    if (i != vw.end() && !(w < *i)) {
        ...
    }
    
    // Lookup using equal_range
    pair<vector<Widget>::iterator, vector<Widget>::iterator> range =
    equal_range(vw.begin(), vw.end(), w);
    if (range.first() != range.second()) {
        ...
    }

Object comparison in associative containers is not done using equality (i.e. `operator==` for the contained type), but equivalence, as defined by a user-defined functor (defaulting to `less`) that can be designed to sort objects in custom ways. [Meyers01](#Meyers01) §19 §20 §21


#### set

Avoid in-place key modifications in sets, it may break element ordering (if the change affects the equivalence predicate). Instead, do the following: [Meyers01](#Meyers01) §22

    set<Elem> s;
    Elem elementToChange;
    set<Elem>::iterator i = s.find(elementToChange); // Find element to change
    if (i != s.end()) {
        Elem e(*i);                                    // Copy element
        e.changeSomething();                           // Change the copy
        e.erase(i++);                                  // Erase the original
        s.insert(i, e);                                // insert the copy, hint at position if likely to be the same as the original
    }


#### multiset

Avoid in-place key modifications in multisets (see `set`).


#### map

Using `operator[]` is the most efficient method for updating existing elements in a `map`, while using `insert()` is more efficient for adding new elements. [Meyers01](#Meyers01) §24


Container functions
-------------------

The standard containers provide a compatible set of functions.


#### empty()

Tells if the container is empty in constant time.


#### size()

Gives the size of the container (number of elements). Should never be used to compare with 0 to check if empty because the `empty()` function always does this in constant time. [Meyers01](#Meyers01) §4


#### capacity()

Gives the current number of elements the container has allocated space for (usually more than `size()` for performance reasons).


#### reserve()

Should be used to avoid unnecessary reallocations when the size of the container can be anticipated.
Usually reserves more than the given number for performance reasons. `vector` does nothing if the given size is smaller than the current one, `string` may shrink the allocated memory. [Meyers01](#Meyers01) §14


#### swap()

Swaps the contents of two containers of the same type.

Can be used to efficiently trim excess capacity in `vector` and `string` containers (the "shrink-to-fit" idiom): [Meyers01](#Meyers01) §17

    vector<int>(v.begin(), v.end()).swap(v);
    string(s.begin(), s.end()).swap(s);

It can also be used to both clear a container and to reduce its capacity to the minimum possible at the same time:

    vector<int>().swap(v);
    string().swap(s);


### Range-based functions

The range-based functions takes two iterators, defining a range into a container. They should be preferred to their single-element counterparts because they are easier to write, express their intent more clearly, and exhibit better performance. [Meyers01](#Meyers01) §5

All containers offer a *Range-based constructor* and *Range-based erase()*}. All *Sequence containers* offer a *Range-based insert()* and *Range-based assign()*.


#### Range-based constructor

    container::container(InputIterator begin, inputIterator end);

When using iterator-based constructors, it is best to not use anonymous iterators but to declare them explicitly. This avoids invoking C++'s "most vexing parse" by accident: [Meyers01](#Meyers01) §6

    // Incorrect!
    list<int> data(istream_iterator<int>(file), istream_iterator<int>());
    
    // Correct, but unclear
    list<int> data((istream_iterator<int>(file)), istream_iterator<int>());
    
    // Correct and clear
    istream_iterator<int> dataBegin(file);
    istream_iterator<int> dataEnd;
    list<int> data(dataBegin, dataEnd);


#### Range-based insert()

    // For Sequence containers
    void container::insert(
        iterator position,   // Where to insert
        InputIterator begin, // Start of range to insert
        InputIterator end    // End of range to insert
    );
    
    // For Associative containers
    void container::insert(InputIterator begin, InputIterator end);

Loops using `push_front()` or `push_back()`, and `copy()` being passed `front_inserter` or `back_inserter` should typically be replaced with range-based insertion. [Meyers01](#Meyers01) §5


#### Range-based erase()

    // For Sequence containers
    iterator container::erase(iterator begin, iterator end);
    
    // For Associative containers
    void container::erase(iterator begin, iterator end);

See [Erasing](#Erasing) about how to use range-based `erase()`.


#### Range-based remove()

Only provided for Sequence containers.

See [Erasing](#Erasing) about how to use range-based remove().


#### Range-based assign()

    void container::assign(InputIterator begin, InputIterator end);


<a name="Iterators"></a>
Iterators
---------

Iterators can be classified into five categories depending on the functionality they implement:
1.  *Input iterators:*
    Read-only, single-pass container traversal.
2.  *Output iterators:*
    Write-only, single-pass container traversal.
3.  *Forward iterators:*
    Input+(Output) functionality, multi-pass forward traversal.
4.  *Bidirectional iterators:*
    Forward iterator functionality + backwards traversal.
5.  *Random access iterators:*
    Bidirectional iterator functionality + non-sequential access (classic pointers are of this category).
    Only supported by `vector`, `string` and `deque`.


All STL [Containers](#Containers) offer four types of iterators:  `iterator`, `const_iterator`, `reverse_iterator` and `const_reverse_iterator`.

An `iterator` can be implicitly converted into a `const_iterator` or a `reverse_iterator`, and a `reverse_iterator` into a `const_reverse_iterator`.
The `.base()` function of the reverse variants return their corresponding forward variant.
For container insertion or erasure, typically prefer using the normal iterator even when other options are available. [Meyers01](#Meyers01) §26

To convert from the const variants of a container's iterators into the non-const variants, use `distance()` and `advance()`: [Meyers01](#Meyers01) §27

    Container c;
    Container::const_iterator ci;
    
    Container::iterator i(c.begin());
    advance(i, distance<Container::const_iterator>(i, ci));

To emulate insertion at a position specified by a `reverse_iterator ri`, insert at the position `ri.base()`. [Meyers01](#Meyers01) §28

To erase at a position a position specified by a `reverse_iterator ri`, erase at the element preceding `ri.base()`: [Meyers01](#Meyers01) §28

    container.erase((++ri).base());

To read characters from a stream efficiently, use `istreambuf_iterator`s: [Meyers01](#Meyers01) §29

    ifstream inputFile("data.txt");
    string fileData(istreambuf_iterator<char>(inputFile), istreambuf_iterator<char>());


<a name="Algorithms"></a>
Algorithms
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
