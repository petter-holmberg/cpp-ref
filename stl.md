C++ STL quick reference
=======================

This document is a collection of design guidelines when using the C++ Standard Template Library.
For brevity and simplicity of use, it does not discuss at length the *whys* of the guidelines, but instead presents references to further reading about the subject.

The guidelines cover C++98, C++11 and C++14, with notes on differences between the versions.


STL overview
------------

STL is mainly based on three fundamental concepts: [Meyers96](#Meyers96) §35

1.  [Containers](#Containers):
    Classes for holding collections of objects.

2.  [Iterators](#Iterators):
    A mechanism for traversing containers and accessing held objects in them.

3.  [Algorithms](#Algorithms):
    Functionality for performing operations on objects in containers, using iterators to traverse them.


<a name="Containers"></a>
Containers
----------

The STL containers can be subdivided into two categories:

1.  [Sequence containers](#SequenceContainers):
    Containers oriented towards sequential access of objects.
    The standard sequence containers are `vector`, `string`, `deque`, `list`, and (in C++11) `forward_list`.
2.  [Associative containers](#AssociativeContainers):
    Containers oriented towards random access of objects.
    The standard associative containers are `set`, `multiset`, `map`, `multimap`, and (in C++11), `unordered_set`, `unordered_multiset`, `unordered_map` and `unordered_multimap`.

Additionally, in C++11 there are [Container adaptors](#ContainerAdaptors) providing specialized interfaces to other containers: `priority_queue`, `queue` and `stack`.

Other types that provide much of what is required by standard containers include `array`, `bitset` and `valarray`.

Code should never be written with the goal to generalize the use of a specific container, so that it can be replaced without touching any code that uses it.
Instead, find the container that best matches the required use cases, and use typedefs to clarify syntax and make container replacement easier in the event that it needs to be done. [Meyers01](#Meyers01) §2

Containers should never contain base classes, as this causes slicing (only the base part of derived classes being inserted) which is almost always an error.
To implement containers holding different types, make containers of base class pointers, or preferably of smart pointers (but never `auto_ptr`!)
Containers of `new`:ed pointers are prone to leak memory. The best way to avoid this problem is to use smart pointers. [Meyers01](#Meyers01) §3 §7 §8

`vector` is usually the right container to use by default when it is not obvious that another one should be used, it offers a lot of useful properties that other containers don't provide. [Stroustrup13](#Stroustrup13) §4.4.1 §31.2 §31.4, [Sutter05](#Sutter05) §76

A [Sequence container](#SequenceContainers) is the right option when element position matters, especially for insertion. Otherwise, an [Associative container](#AssociativeContainer) is a viable option. [Meyers01](#Meyers01) §1.

It is also useful to categorize containers in terms of memory layout: [Meyers01](#Meyers01) §1

1.  *Contiguous memory containers:*
    Containers that store multiple elements in chunks of contiguous memory.
    These containers are very efficient for ordered access of elements, but can be inefficient when it comes to insertion and deletion of elements.
    The contiguous memory containers are `vector`, `string`, and `deque`.

2.  *Node-based containers:*
    Containers that store individual elements per chunk of memory.
    These containers are very efficient at insertion and deletion, but can be inefficient when it comes to access.
    The node-based containers are `list`, `forward_list` and all [Associative containers](#AssociativeContainers).

If it is important to avoid movement of existing container elements when inserting or erasing elements, A *Node-based container* is the only option. [Meyers01](#Meyers01) §1

If it is necessary to have the option to reliably revert insertions and erasures, a *Node-based container* is the only option. For multiple-element insertions, the only option is `list`. [Meyers01](#Meyers01) §1

If iterator, pointer and reference invalidation is a big concern, *Node-based containers* are the only ones that never invalidate them (other than ones pointing to the elements being erased). [Meyers01](#Meyers01) §1

STL containers are not thread-safe by default. The standard suggests that:
- Having multiple readers into the same container should be thread-safe.
- Having multiple writers to different containers should be thread-safe.
Not all implementations respect even these guarantees. [Meyers01](#Meyers01) §12

All standard containers implement the Strong Guarantee for all operations, with two exceptions: [Sutter99](#Sutter99) §18

1. Multi-element inserts ("iterator range" inserts) are never strongly exception-safe.
2. For `vector<T>` and `deque<T>`, all inserts and erases are strongly exception-safe only as long as `T`'s copy constructor and assignment operator do not throw; inserting into a `vector<string>` or a `vector<vector<int>>`, for example, are not strongly exception-safe.

This means that classes having container members and using the aforementioned operations must do the work themselves to ensure that their state is predictable if exceptions occur.
To do this, insert and erase in a copy of the container, then use `swap()` to switch over to using the new version after the copy-and-change steps have succeeded.


<a name="SequenceContainers"></a>
### Sequence containers

The standard sequence containers are `vector`, `string`, `deque`, `list`, and `forward_list`.


#### vector

`vector` is the only standard container with a C-compatible layout (same as an array) and is therefore the only option to use when interfacing with a C library.

`vector` should always be preferred to a dynamically allocated array. To pass a `vector` safely to a C API that expects an array, do it like this: [Meyers01](#Meyers01) §13 §16

    void cApiFunction(const int* ints, size_t numInts);
    if (!v.empty()) // &v[0] undefined if vector is empty
        cApiFunction(&v[0], v.size());

A `vector` only deals with properly initialized objects. This is not a requirement on `array` and built-in arrays. [Stroustrup13](#Stroustrup13) §31.4.1.3

Never use `vector<bool>` as a container, instead use `deque<bool>` or a non-STL alternative like `bitset<bool>`. [Meyers01](#Meyers01) §18 [Sutter02](#Sutter02) §6


#### string

`string` should always be preferred to a dynamically allocated char array. [Meyers01](#Meyers01) §13

`string` can be implemented in many different ways, so inferring something about its memory use and performance compared to char arrays without consulting the specific implementation's documentation is a bad idea. [Meyers01](#Meyers01) §15

Many `string` implementations use reference counting. If this is a problem, a `vector<char>` can be used instead. [Meyers01](#Meyers01) §13

Using `swap()` on `string`s invalidates iterators, pointers and references to them. [Meyers01](#Meyers01) §1

Using `c_str()` is the only safe and correct way to pass `string`s to a C API that expects a char array. [Meyers01](#Meyers01) §16


#### deque

While `vector` should be the default sequence container of choice, `deque` should be used when most insertions and deletions take place at the beginning or at the end of the sequence.
It offers constant-time `insert()` and `erase()` operations at both ends, uses memory in an operating system-friendly way (large `deque`s can be split in multiple blocks of memory of a suitable size), is somewhat easier to use and inherently more efficient for growth. [Sutter02](#Sutter02) §7


#### list

A `list` is implemented as a doubly-linked list and allows inserting and deleting elements without moving existing elements.

A `list` usually has a four-word-per-element memory overhead, and traversal of elements is significantly slower than in a `vector`. [Stroustrup13](#Stroustrup13) §31.4.2


#### forward_list

A `forward_list` is implemented as a singly-linked list and is ideal for empty and very short sequences that are typically traversed from the beginning. [Stroustrup13](#Stroustrup13) §31.4.2


<a name="AssociativeContainers"></a>
### Associative containers

The standard associative containers are `set`, `multiset`, `map`, `multimap`, and (in C++11), `unordered_set`, `unordered_multiset`, `unordered_map` and `unordered_multimap`. The ordered versions store their elements sorted, typically using some kind of balanced binary tree, and the unordered versions use hash tables with linked overflow. [Stroustrup13](#Stroustrup13) §31.4.3

The hash table based versions (which store objects unsorted and can provide amortized constant-time complexity instead of logarithmic) were not part of STL prior to C++11, but there are several libraries that provide them. [Meyers01](#Meyers01) §25

Once a key has been inserted into an associative container, that key must never change its relative position in the container. [Sutter02](#Sutter02) §8

The standard associative containers are optimized for a mixed combination of inserts, erasures, and lookups. But many usage scenarios look more like this: [Meyers01](#Meyers01) §23

1. *Setup phase*, consisting mainly of many inserts (and possibly erasures)
2. *Lookup phase*, consisting mainly of lookups (bulk of the time spent here!)
3. *Reorganize phase*, modifying/replacing data, then returning to lookup again (if needed at all)

In this type of scenario, replacing the container with a sorted `vector` is likely to improve both memory usage and speed considerably (due to caching), assuming that the lookup phase contains only lookups.

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

Object comparison in associative containers is not done using equality (i.e. `operator==` for the contained type), but equivalence, as defined by a user-defined functor (defaulting to `less()`) that can be designed to sort objects in custom ways. [Meyers01](#Meyers01) §19 §20 §21


#### set

Avoid in-place key modifications in `set`s, it may break element ordering (if the change affects the equivalence predicate). Instead, do the following: [Meyers01](#Meyers01) §22

    set<Elem> s;
    Elem elementToChange;
    set<Elem>::iterator i = s.find(elementToChange); // Find element to change
    if (i != s.end()) {
        Elem e(*i);           // Copy element
        e.changeSomething();  // Change the copy
        e.erase(i++);         // Erase the original
        s.insert(i, e);       // Insert the copy, hint at position if likely to be the same as the original
    }


#### multiset

Avoid in-place key modifications in `multiset`s (see `set`).


#### map

Using `operator[]` is the most efficient method for updating existing elements in a `map`, while using `insert()` is more efficient for adding new elements. [Meyers01](#Meyers01) §24


<a name="ContainerAdaptors"></a>
### Container adaptors

The STL container adaptors provide a different interface to an underlying container. They do not offer direct access to it and don't offers iterators or subscripting. [Stroustrup13](#Stroustrup13) §31.5


#### stack

The `stack` adaptor eliminates the non-stack operations on its container from the interface. [Stroustrup13](#Stroustrup13) §31.5.1


#### queue

The `queue` adaptor is an interface to a container that allows the insertion of elements at the `back()` and the extraction of elements at the `front()`. [Stroustrup13](#Stroustrup13) §31.5.2

#### priority_queue

The `priority_queue` adaptor is a queue in which each element is given a priority that controls the order in which the elements get to the `top()`. [Stroustrup13](#Stroustrup13)[Stroustrup13](#Stroustrup13) §31.5.3


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

All containers offer a *Range-based constructor* and *Range-based erase()*. All [Sequence containers](#SequenceContainers) offer a *Range-based insert()* and *Range-based assign()*.


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

Only provided for [Sequence containers](#SequenceContainers).

See [Erasing](#Erasing) about how to use range-based `remove()`.


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

Prefer the use of algorithms to hand-written loops whenever possible. There are many advantages: [Meyers01](#Meyers01) §43
- Efficiency: Algorithms are often more efficient than the loops programmers produce.
- Correctness: Writing loops is more subject to errors than is calling algorithms.
- Maintainability: Algorithm calls often yield code that is clearer and more straightforward than the corresponding explicit loops.

Algorithm functions like `count()`, `find()`, `lower_bound()`, `upper_bound()` and `equal_range()` often come as container member functions in addition to a non-member version.
When there are container member functions with the same names as non-member algorithm functions, prefer the member function; it is likely to be optimized for use with the specific container type and works consistently with the container's other member functions. [Meyers01](#Meyers01) §44

When using algorithms to expand containers, make sure the destination range is big enough.
A common mistake is to use the container's `.end()` iterator as the destination. This is wrong, as it points outside of the container.
The correct solution is often to use `back_inserter()`, `front_inserter()` or `inserter()`, depending on the use case and which of these are available.
Using `.reserve()` in advance, for `vector`s and `string`s, can reduce the number of reallocations.
When overwriting an entire container, the inserters are not necessary, but then use `.resize()` to ensure that for instance a `vector` can hold the new range of values. [Meyers01](#Meyers01) §30

Functors typically make better algorithm parameters than functions, enabling optimization through inlining which often outperforms corresponding algorithm functions in C.
It is also more portables, as different implementations of STL may have problems compiling code that uses algorithms used together with functions. [Meyers01](#Meyers01) §46

Using STL algorithms to solve complex problems in one go can lead to "write-only code" that is very difficult to read and understand.
Use good judgment and split up the problem or use alternative ways to solve the problem when it leads to clearer code. [Meyers01](#Meyers01) §47


<a name="Searching"></a>
### Searching

There are many STL algorithms that can be used for searching. The options can be summarized as follows: [Meyers01](#Meyers01) §45

If you just want to know if a specific value exists:
- In an unsorted range: `find()`
- In a sorted range: `binary_search()`
- In a `set` or `map`: `.count()`
- In a `multiset` or `multimap`: `.find()`

If you want to know if a specific value exists and also where it is:
- In an unsorted range: `find()`
- In a sorted range: `equal_range()`
- In a `set` or `map`: `.find()`
- In a `multiset` or `multimap`: `.find()` or `.lower_bound()` (depending on if you need only one element or specifically the first one)

If you want to find the first object with a value not preceding a specific value:
- In an unsorted range: `find_if()`
- In a sorted range: `lower_bound()`
- In a `set` or `map`: `.lower_bound()`
- In a `multiset` or `multimap`: `.lower_bound()`

If you want to find the first object with a value succeeding a specific value:
- In an unsorted range: `find_if()`
- In a sorted range: `upper_bound()`
- In a `set` or `map`: `.upper_bound()`
- In a `multiset` or `multimap`: `.upper_bound()`

If you want to count how many objects have a specific value:
- In an unsorted range: `count()`
- In a sorted range: `equal_range(), then distance()`
- In a `set` or `map`: `.count()`
- In a `multiset` or `multimap`: `.count()`

If you want to find all objects that have a specific value:
- In an unsorted range: `find() (iteratively)`
- In a sorted range: `equal_range()`
- In a `set` or `map`: `.equal_range()`
- In a `multiset` or `multimap`: `.equal_range()`


<a name="Sorting"></a>
### Sorting

There are many sorting algorithms offered by STL, and they solve different problems. Use the right one to solve the problem in the most efficient way: [Meyers01](#Meyers01) §31
- To fully sort a `vector`, `string`, `deque` or `array`, use `sort()` or `stable_sort()`.
- To put the top n elements of a `vector`, `string`, `deque` or `array` in front, use `partial_sort()`.
- To identify the element at position n or identify the top n elements without moving any, use `nth_element()`.
- To separate the elements of a standard [Sequence container](#SequenceContainers) or array into those that do and those that don't satisfy some criterion, use `partition()` or `stable_partition()`.
- To (stably) sort a `list`, use its `.sort()` member function. Performing partial sorts etc. on `list`s has to be done indirectly.

`sort()`, `partial_sort()` and `nth_element()` sorts elements with equivalent values any way they want to. `stable_sort()` is the only option that preserves the order of equivalent elements.

`nth_element()` can be used not only to find the top n elements of a range, it can also be used to find the median value at a particular percentile.

`partition()` reorders elements in a range so that all elements satisfying a particular criterion are at the beginning of the range.

The ordered [Associative containers](#AssociativeContainers) cannot be sorted (they already are).

`partition()` and `stable_partition()` require only bidirectional iterators, so they can be used on all [Sequence container](#SequenceContainers) types.

The sorting algorithms, in order of performance (best to worst) are:
1. `partition()`
2. `stable_partition()`
3. `nth_element()`
4. `partial_sort()`
5. `sort()`
6. `stable_sort()`

Many algorithms require containers to be sorted: [Meyers01](#Meyers01) §34
- `binary_search()`
- `lower_bound()`
- `upper_bound()`
- `equal_range()`
- `set_union()`
- `set_intersection()`
- `set_difference()`
- `set_symmetric_difference()`
- `merge()`
- `inplace_merge()`
- `includes()`
- `unique()` (not required but customary)
- `unique_copy()` (not required but customary)

By default, binary_search assumes that a range is sorted by `less()`, so when this is not the case it must be passed the same comparison function that the container was sorted with.


<a name="Erasing"></a>
### Erasing

To erase objects from a container, different methods should be used depending on the type of container: [Meyers01](#Meyers01) §9 §32
- For the `list` container, the best method is: `c.remove(value);`
- For contiguous memory containers (`vector`, `string`, and `deque`), the best method is the "erase-remove-idiom": `c.erase(remove(c.begin(), c.end(), value), c.end());`
- For [Associative containers](#AssociativeContainers), the only method is: `c.erase(value);`

For predicate-based erasing in [Sequence containers](#SequenceContainers), simply use `remove_if()` instead of `remove()`. For the [Associative containers](#AssociativeContainers), there are two approaches:

    // Easier but less efficient:
    Container<int> c;
    Container<int> goodValues; // Temporary containing values to keep
    remove_copy_if(c.begin(), c.end(), inserter(goodValues, goodValues.end()), badValueFunction);
    c.swap(goodValues);
    
    // Harder but more efficient:
    Container<int> c;
    for (Container<int>::iterator i = c.begin(); i != c.end(); /*nothing*/) {
        if (badValueFunction(*i))
            c.erase(i++) // Postfix increment!
        else
            ++i;
    }

To do something in addition to erasing with each element (like logging), for [Associative containers](#AssociativeContainers) the second version above just needs to be extended. For [Sequence containers](#SequenceContainers) the return value of `erase()` must to be used:

    for (Container<int>::iterator i = c.begin(); i != c.end(); ) {
        if (badValueFunction(*i)) {
            // Do something extra
            i = c.erase();
        } else
            ++i;
    }

Despite its name, `remove()` doesn't actually remove elements from containers. It just moves elements that are not to be removed up to the front of the container, overwriting as it goes, and returning the iterator to the new logical end of the range.
Feeding this into the range-based version of `erase()` will erase what's left at the end of the container. The same goes for `remove_if()` and `unique()`. For lists, the `.unique()` member function does full erasing, just like `.remove()`. [Meyers01](#Meyers01) §32 [Sutter02](#Sutter02) §2

Erasing in a container of pointers can easily lead to resource leaks. One solution is to iterate over the container, deleting and setting pointers to null, then using the erase-remove idiom to eliminate the null pointers in the container (assuming no null pointers should be kept). [Meyers01](#Meyers01) §32


<a name="Summarizing"></a>
### Summarizing

To perform custom summarizing of elements in some range, use `accumulate()` together with a stateless binary functor returning objects of the type the container holds, or `for_each()` together with a unary functor storing the sum and a function to retrieve the final result afterwards. [Meyers01](#Meyers01) §37


Smart pointers
--------------

C++11 offers a set of standard smart pointers, which simplify memory management. These are preferable to raw pointers in many contexts because they simplify client code.


### unique_ptr

`unique_ptr` is a small, fast, move-only smart pointer for managing resources wtih exclusive ownership semantics. By default it uses `delete` for destruction, but custom deleters can be specified. Stateful deleters and function pointers increase the size of `unique_ptr` objects. [Meyers14](#Meyers14) §18

`unique_ptr` should generally be the first choice of smart pointer type. Converting a `unique_ptr` to a `shared_ptr` is easy. [Meyers14](#Meyers14) §18

Prefer `make_unique` to `new` when creating `unique_ptr` objects, whenever possible. [Meyers14](#Meyers14) §21


### shared_ptr

`shared_ptr` offers convenience approaching that of garbage collection for the shared lifetime management of arbitrary resources. `shared_ptr` objects are typically twice as big as `unique_ptr` objects, incur overhead for control blocks, and require atomic reference count manipulations. By default it uses `delete` for destruction, but custom deleters can be specified. The type of the deleter has no effect on the type of `shared_ptr` objects. [Meyers14](#Meyers14) §19

Avoid creating `shared_ptr` objects directly from variables of raw pointer type (including `this` within classes), as having more than one `shared_ptr` from a single raw pointer leads to undefined behavior. Instead, use `make_shared` (if custom deleters are not needed). Alternatively, pass the result of `new` directly to the `shared_ptr` constructor. [Meyers14](#Meyers14) §19

Prefer `make_shared` or `allocate_shared` to `new` when creating `shared_ptr` objects, whenever possible. Exceptions include classes with custom memory management and systems with memory concerns, very large objects and `weak_ptr`s that outlive the corresponding `shared_ptr`s. [Meyers14](#Meyers14) §21


### weak_ptr

`weak_ptr` acts like a `shared_ptr` but allows pointing to an object that no longer exists. It can be used to construct `shared_ptr` objects, and can be useful for the implementation of caches, observer lists, and the prevention of `shared_ptr` cycles. [Meyers14](#Meyers14) §20


### auto_ptr

`auto_ptr` is a deprecated smart pointer type in C++11 (`unique_ptr` is a better alternative) and is best avoided.


Error messages
--------------

STL error messages can be very hard to decipher. The following are some useful hints: [Meyers01](#Meyers01) §49

For `vector` and `string`, iterators are sometimes pointers, so compiler diagnostics may refer to pointer types if you've made a mistake with an iterator.
For example, if your source code refers to `vector<double>::iterator`s, compiler messages will sometimes mention `double*` pointers.

Messages mentioning `back_inserter_operator`, `front_inserter_operator`, or `insert_iterator` almost always mean you've made a mistake calling `back_inserter()`, `front_inserter()`, or `inserter()`, respectively. (`back_inserter()` returns an object of type `back_inserter_iterator`, etc.)
If you didn't call these functions, some function you called (directly or indirectly) did.

Similarly, if you get a message mentioning `binder1st` or `binder2nd`, you've probably made a mistake using `bind1st()` or `bind2nd()`. (`bind1st()` returns an object of type `binder1st` etc.)

Output iterators (e.g. `ostream_iterator`s, `ostreambuf_iterator`s, and the iterators returned from `back_inserter()`, `front_inserter()` and `inserter`) do their outputting or inserting work inside assignment operators, so if you've made a mistake with one of these iterator types, you're likely to get a message complaining about something inside an assignment operator you've never heard of.

If you get an error message originating from inside the implementation of an STL algorithm (i.e. the source code giving rise to the error is in `<algorithm>`), there's probably something wrong with the types you're trying to use with that algorithm. For example, you may be passing iterators of the wrong category.

If you're using a common STL component like `vector`, `string`, or the `for_each()` algorithm, and a compiler says it has no idea what you're talking about, you've probably failed to `#include` a required header file.


Nonstandard components
----------------------

Hashed associative containers: `hash_set`, `hash_multiset`, `hash_map`, and `hash_multimap`

Singly linked list: `slist`

Container for very large strings: `rope`

Nonstandard functors and adapters: `select1st()`, `select2nd()` (very useful for `map`s and `multimap`s), `identity()`, `project1st()`, `project2nd()`, `compose1()`, `compose2()`, etc.


References
----------

<a name="Stroustrup13"></a>
[Stroustrup13]
"The C++ Programming Language, 4th Edition", ISBN 978-0-321-56384-2

<a name="Meyers96"></a>
[Meyers96]
"More Effective C++", ISBN 0-201-63371-X

<a name="Meyers01"></a>
[Meyers01]
"Effective STL", ISBN 0-201-74962-5

<a name="Meyers14"></a>
[Meyers14]
"Effective Modern C++", ISBN 978-1-491-90399-5

<a name="Sutter99"></a>
[Sutter99]
"Exceptional C++", ISBN 0-201-61562-2

<a name="Sutter02"></a>
[Sutter02]
"More Exceptional C++", ISBN 0-201-70434-X

<a name="Sutter05"></a>
[Sutter05]
"C++ coding standards - 101 Rules, Guidelines, and Best Practices", ISBN 0-321-11358-6

[The SGI STL site](http://www.sgi.com/tech/stl/)

[The STLport site](http://www.stlport.org/)

[The Boost site](http:www.boost.org/)
