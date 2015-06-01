C++ concurrency quick reference
===============================

This document is a collection of design guidelines for C++ concurrency, using the concurrency and multithreading facilities in the C++11 and C++14 standards.
For brevity and simplicity of use, it does not discuss at length the *whys* of the guidelines, but instead presents references to further reading about the subject.

The guidelines cover C++11 and C++14, with notes on differences between the versions.

There are two main reasons to use concurrency [Williams12](#Willams12) §1.2:
- Separation of concerns
- Performance

There are two ways to use concurrency for performance:
- Dividing a single task into parts and run each in parallel (task parallellism, data parallellism).
- Using the available parallellism to solve multiple tasks at once.


Threads
-------


### Launching threads

A new thread is started by constructing a `std::thread` object, passing a function or function object as the first argument. [Williams12](#Willams12) §2.1.1:

**Example:**

    void thread_function() {
        // In the first thread
    }
    
    struct thread_function_object() {
        void operator()() const {
            // In the second thread
        }
    };
    
    struct thread_class() {
        void thread_member_function() {
            // In the third thread
        }
    };
    
    std::thread first_thread(thread_function);
    std::thread second_thread{thread_function_object()};
    thread_class obj;
    std::thread third_thread(&thread_class::thread_member_function, &obj);
    std::thread fourth_thread([](
        /* In the fourth thread */
    });


### Waiting for threads to complete

Once a thread is started, explicitly decide to wait for it to finish by calling `.join()`, or leave it to run on its own by calling `.detach()`. If none of these options are chosen, the thread's destructor will terminate the program. Calling `.join()` can only be done once for a given thread and once it's called, the `std::thread` object is no longer associated with the now finished thread. Its member function `.joinable()` will return false to indicate this. [Williams12](#Willams12) §2.1.2

To avoid program termination if an exception occurs, make sure that the thread is joined in both the normal and the exceptional case. This can be ensured using the RAII idiom: [Williams12](#Willams12) §2.1.3

**Example implementation:**

    class thread_guard {
        std::thread& thr_;
    public:
        explicit thread_guard(std::thread& thr) : thr_(thr) {}
        thread_guard(const thread_guard&) = delete;
        thread_guard& operator=(const thread_guard&) = delete;
        ~thread_guard() {
            if (thr_.joinable()) {
                thr_.join();
            }
        }
    };
    
    void thread_function() {
        // Do some work
    }
    
    void func() {
        std::thread thr(thread_function);
        thread_guard guard(thr);
        do_something_that_might_throw();
    };


### Detaching threads

Calling `.detach()` on a `std::thread` object leaves the thread to run in the background, with no means to wait for it to complete. Such threads are often called *daemon threads*. A thread for which `.joinable()` returns false cannot be detached. [Williams12](#Willams12) §2.1.3

If a thread is detached, ensure that the data accessed by the thread outlives it. One common way to ensure this is to copy the data into the thread rather than sharing it.


### Passing arguments to threads

Arguments to the thread function can be passed as additional arguments to the `std::thread` constructor. By default the arguments are copied, even if the arguments to the thread function itself are expecting references. This may lead to problems if passing pointers to local variables or temporaries, or when you actually want to pass a reference. [Williams12](#Willams12) §2.2

**Example:**

    void dont_update_str(const std::string& str) {
        std::cout << str << std::endl;
    };
    
    void oops1() {
        char buffer[] = "Hello";
        std::thread t1(dont_update_str(buffer));             // Bad, references the local buffer!
        std::thread t2(dont_update_str(std::string(buffer))) // Good, copies buffer!
        ...
    }
    
    void update_str(std::string& str_to_update) {
        str += " World";
    }
    
    void oops2() {
        std::string str("Hello");
        std::thread t1(update_str(str)); // Bad, updates a copy of str!
        std::thread t2(update_str(std::ref(str))) // Good, updates the local string!
        ...
    }

Objects supporting move semantics can also be moved into `std::thread`s to transfer ownership.

**Example:**

    void process_big_object(std::unique_ptr<big_object> obj) {
        // Do some work on obj
    }
    
    void fire_and_forget() {
        std::unique_ptr<big_object> obj(new big_object);
        std::thread t(process_big_object, std::move(obj));
        t.detach();
    }


### Transferring ownership of threads

`std::thread` objects are not copyable but they are movable, so it is possible to write functions that creates threads and returns them, or to pass created threads to other functions that wait for them to complete. Note that moving into `std::thread` objects already associated with a (non-joined, non-detached) thread will terminate the program. [Williams12](#Willams12) §2.3

**Example:**

    void thread_function() {
        // Do some work
    }
    
    std::thread make_thread() {
        return std::thread(thread_function);
    }
    
    void accept_thread(std::thread thr);
    
    void start_thread() {
        accept_thread(std::thread(thread_function));
        std::thread thr(thread_function);
        accept_thread(std::move(thr));
    }

Transfer of `std::thread` ownership can be used to ensure that threads are joined before a scope is exited:

**Example:**

    class scoped_thread {
        std::thread thr_;
    public:
        explicit scoped_thread(std::thread thr) : thr_(std::move(thr)) {
            if (!thr_.joinable()) {
                throw std::logic_error("No thread");
            }
        }
        scoped_thread(const scoped_thread&) = delete;
        scoped_thread& operator=(const scoped_thread&) = delete;
        ~scoped_thread() {
            thr_.join();
        }
    };
    
    void thread_function() {
        // Do some work
    }
    
    void func() {
        scoped_thread st(std::thread(thread_function));
        do_something_that_might_throw();
    };

`std::thread` objects can be stored in move-aware containers:

**Example:**

    std::vector<std::thread> threads {std::thread(thread_function), std::thread(thread_function)}; // Spawn two threads
    std::for_each(threads.begin(), threads.end(), std::mem_fn(&std::thread::join)); // Wait for each thread to finish, in turn.


### Choosing the number of threads at runtime

Use the `std::thread::hardware_concurrency()` function to get a hint of how many threads that can truly run concurrently for a given execution of a program. It may return 0 if this information is not available. [Williams12](#Willams12) §2.4


### Identifying threads

Thread identities are of the type `std::thread::id` and can be retrieved in two ways: [Williams12](#Willams12) §2.5
- By calling the `.get_id()` member function of a `std::thread` object (may return a default-constructed instance if this object has no associated thread).
- By calling the `std::this_thread::get_id()` function from within a thread.

`std::thread::id` objects are copyable, comparable, hashable, totally ordered, and serializable, making them useful in many contexts.


References
----------

<a name="Stroustrup13"></a>
[Stroustrup13]
"The C++ Programming Language, 4th Edition", ISBN 978-0-321-56384-2

<a name="Williams12"></a>
[Williams12]
"C++ Concurrency in Action, Practical Multithreading", ISBN 978-1-933-98877-1

<a name="Meyers14"></a>
[Meyers14]
"Effective Modern C++", ISBN 978-1-491-90399-5
