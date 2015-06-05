C++ concurrency quick reference
===============================

This document is a collection of design guidelines for C++ concurrency, using the concurrency and multithreading facilities in the C++11 and C++14 standards.
For brevity and simplicity of use, it does not discuss at length the *whys* of the guidelines, but instead presents references to further reading about the subject.

The guidelines cover C++11 and C++14, with notes on differences between the versions.


Concurrency overview
--------------------

There are two main reasons to use concurrency: [Williams12](#Willams12) §1.2
- Separation of concerns
- Performance

There are two ways to use concurrency for performance:
- Dividing a single task into parts and run each in parallel (task parallellism, data parallellism).
- Using the available parallellism to solve multiple tasks at once.

The C++ concurrency features refer to an activity potentially executed concurrently with other activities as a *task*. The implementation of a task is a normal function or function object.
A *thread* is the system-level representation of the computer's facility for executing a task. Threads may share the same address space, and may therefore access the same memory locations.


Tasks
-----

The task-based model of concurrency in C++ provides a high-level view that focuses on producing results concurrently, without requiring code that directly shares data between threads. Think in terms of tasks that can be executed concurrently, rather than directly in terms of threads, and prefer task-based programming to thread-based. [Meyers14](#Meyers14) §35 [Stroustrup13](#Stroustrup13) §42.4

Situations when a thread-based rather than task-based approach may be appropriate include: [Meyers14](#Meyers14) §35
- When there is a need to access to the API of the underlying threading implemenation.
- When there is a need and possibility to optimize thread usage for the application.
- When there is a need to implement threading technology beyond the C++ concurrency API.


### Futures

If a thread needs to wait for a specific one-off event, use a `std::future<>`, or a `std::shared_future<>` to represent this event. The template parameter is for the type of the associated data, with the `<void>` specialization representing situations when there is no associated data. [Williams12](#Willams12) §4.2.1

For simple asynchronous tasks, instead of launching a `std::thread` (which doesn't have a direct way of returning a calculated result), use `std::async()`. It immediately returns a `std::future<>` object, which will eventually hold the returned value of the called function. To get the value, call the `.get()` member function of the future, which will block until the result can be returned.

**Example:**

    int expensive_calculation_1(int argument);
    int expensive_calculation_2(int argument);
    
    int sum_of_expensive_calculations(int argument_1, int argument_2) {
        std::future<int> answer_1 = std::async(expensive_calculation_1, argument_1);
        std::future<int> answer_2 = std::async(expensive_calculation_2, argument_2);
        return answer_1.get() + answer_2.get();
    }

By default, the implementation may decide to not actually invoke a task asynchronously. The optional `std::async()` launch policy parameter can be used to suggest which option to choose, with the values `std::launch::async` to force the task to run asynchonously on a different thread, or `std::launch::deferred` to run it synchronously when a call to `.get()` or `.wait()` on an associated `std::future<>` or `std::shared_future<>` is invoked. `std::launch::async | std::launch::deferred` is the default. Switching policies can be helpful in debugging concurrent code. [Meyers14](#Meyers14) §36 [Stroustrup13](#Stroustrup13) §42.4.6

If the following conditions are fulfilled, the default launch policy for a task is fine: [Meyers14](#Meyers14) §36
- The task need not run concurrently with the thread calling `.get()` or `.wait()`.
- It doesn't matter which thread's `thread_local` variables are read or written.
- Either there's a guarantee that `.get()` or `.wait()` will be called on the `std::future<>` returned by `std::async()` or it's acceptable that the task may never execute.
- Code using `.wait_for()` or `.wait_until()` takes the possibility of deferred status into account. 

A `std::future<>` object needs to be protected via a mutex or some other synchronization mechanism if multiple threads have access to it. The same is true for `std::shared_future<>`, but copies of a `std::shared_future<>` referring to the same result can be used without synchronization. This is by design; a `std::future<>` models unique ownership of the asynchronous result, where only one call to `.get()` should be made, but with a `std::shared_future<>` mutliple threads can wait for the same result through their own local copy. Copies can be obtained via the `.share()` member function. [Williams12](#Willams12) §4.2.5

The final `std::future<>` or `std::shared_future<>` referring to a shared state for a non-deferred task launched via `std::async()` blocks until the task completes. [Meyers14](#Meyers14) §38


### Promises

A `std::promise<>` is a handle to a shared state. It is where a task can deposit its result to be retrieved through a `std::future<>`, acquired through its `.get_future()` member function. The member functions `.set_value()` and `.set_exception()` set the result (which can be an exception).

Don't `.set_value()` or `.set_exception()` to a `std::promise<>` twice, it will throw a `std::future_error` exception. [Stroustrup13](#Stroustrup13) §42.4.2


#### Packaged tasks

Instead of invoking `std::async()` directly, asynchronous tasks can be prepared for later invocation by wrapping them in a `std::packaged_task<>` object. A `std::packaged_task<>` holds a task and a `std::future<>` / `std::promise<>` pair. The template parameter is a function signature for the asynchronous function to call (the function's parameters and return types can differ as long as they are convertible to the given types). 
`std::packaged_task<>` is a callable type and invoking it sets its internal `std::future<>` that can be extracted with the `.get_future()` member function later to extract the result of the task. If the task returns a value, it causes a `.set_value()` on its internal `std::promise<>`. Similarly, if it throws an exception, it causes a `.set_exception()`. [Williams12](#Willams12) §4.2.2

**Example:**

    int task(int i) {
        if (i) {
            return i;
        }
        throw std::runtime_error("task(0)");
    }
    
    int run_tasks() {
        std::packaged_task<int(int)> pt1 {task};
        std::packaged_task<int(int)> pt2 {task};
    
        pt1(1); // Let pt1 call task(1) asynchronously, don't care how.
        pt2(0); // Let pt2 call task(0) asynchronously, don't care how.
    
        auto future1 = pt1.get_future(); // Will contain 1.
        auto future2 = pt2.get_future(); // Will contain exception.
    
        try {
            std::cout << future1.get() << "\n"; // Will print.
            std::cout << future2.get() << "\n"; // Will throw.
        } catch (const std::exception& ex) {
            std::cout << "Exception: " << ex.what() << "\n";
        }
    }

`std::packaged_task<>`s can be used as a building block for thread pools or other task management schemes, such as running eash task on its own thread, or running them all sequentially on a particular background thread.


Threads
-------

A `std::thread` is an abstraction of the computer hardware's notion of a computation. All threads work in the same address space. For hardware protection against data races, use some notion of a process.
Stacks are not shared between threads, so local variables are not subject to data races, unless if they are accessed through pointers. For this reason, watch out for by-reference context bindings of local variables or capturing members of an object through `[this]` in lambdas used as tasks. If in doubt, copy (`[=]`). [Stroustrup13](#Stroustrup13) §42.2 §42.4.6

Thread-based programming calls for manual management of thread exhaustion, oversubscription, load balancing, and adaptation to a new platform. [Meyers14](#Meyers14) §35


### Launching threads

A new thread is started by constructing a `std::thread` object, passing a function or function object as the first argument. A `std::thread` is intended to map one-to-one with the operating system's software threads, and is movable but not copyable. If a new software thread cannot be created, a `std::system_error` exception is thrown. [Stroustrup13](#Stroustrup13) §42.2 [Williams12](#Willams12) §2.1.1

**Example:**

    void thread_function() {
        // In the first thread.
    }
    
    struct thread_function_object() {
        void operator()() const {
            // In the second thread.
        }
    };
    
    struct thread_class() {
        void thread_member_function() {
            // In the third thread.
        }
    };
    
    std::thread first_thread(thread_function);
    std::thread second_thread {thread_function_object()};
    thread_class obj;
    std::thread third_thread(&thread_class::thread_member_function, &obj);
    std::thread fourth_thread([](
        /* In the fourth thread. */
    });

Because `std::thread` objects may start running a function immediately after they are initialized, declare them last in a class. That guarantees that at the time they are constructed, all the data members that precede them have already been initialized and can therefore safely be accessed by the asynchronously running thread that correspond to the `std::thread` data member. [Meyers14](#Meyers14) §37


### Waiting for threads to complete

Once a thread is started, explicitly decide to wait for it to finish by calling `.join()`, or leave it to run on its own by calling `.detach()`. If none of these options are chosen, the thread's destructor will terminate the program, to prevent a system thread from accidentally outliving its `std::thread` object. Avoid destroying a running thread like this by making `std::thread`s unjoinable on all paths. [Meyers14](#Meyers14) §37 [Stroustrup13](#Stroustrup13) §42.2.2

Calling `.join()` can only be done once for a given thread and once it's called, the `std::thread` object is no longer associated with the now finished thread. Its member function `.joinable()` will return false to indicate this. [Stroustrup13](#Stroustrup13) §42.2 [Williams12](#Willams12) §2.1.2

To avoid program termination if an exception occurs, make sure that the thread is joined in both the normal and the exceptional case. This can be ensured using the RAII idiom. [Stroustrup13](#Stroustrup13) §42.2.4 [Williams12](#Willams12) §2.1.3

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
        // Do some work.
    }
    
    void func() {
        std::thread thr(thread_function);
        thread_guard guard(thr);
        do_something_that_might_throw();
    };


### Detaching threads

Calling `.detach()` on a `std::thread` object leaves the thread to run in the background, with no means to wait for it to complete. Such threads are often called *daemon threads*. A thread for which `.joinable()` returns false cannot be detached. [Stroustrup13](#Stroustrup13) §42.2.5 [Williams12](#Willams12) §2.1.3

If a thread is detached, ensure that the data accessed by the thread outlives it. One common way to ensure this is to copy the data into the thread rather than sharing it.

An often better alternative to detaching threads is to move them into a "main module" of a program, access them through `unique::ptr<>`s or `shared::ptr<>`s, or place them in a `std::vector<std::thread>` to avoid losing track of them. [Stroustrup13](#Stroustrup13) §42.2.5


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
        std::thread t2(update_str(std::ref(str))); // Good, updates the local string!
        std::thread t3([&str]{ update_str(str); }); // Also good, reference wrapper updates the local string!
        ...
    }

Objects supporting move semantics can also be moved into `std::thread`s to transfer ownership.

**Example:**

    void process_big_object(std::unique_ptr<big_object> obj) {
        // Do some work on obj.
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
        // Do some work.
    }
    
    std::thread make_thread() {
        return std::thread(thread_function);
    }
    
    void accept_thread(std::thread thr);
    
    void start_thread() {
        accept_thread(std::thread(thread_function)); // Moves rvalue thread.
        std::thread thr(thread_function);
        accept_thread(std::move(thr)); // Moves lvalue thread.
    }

Transfer of `std::thread` ownership can be used to ensure that threads are joined before a scope is exited.

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

`std::thread` objects can be stored in move-aware containers.

**Example:**

    std::vector<std::thread> threads {std::thread(thread_function), std::thread(thread_function)}; // Spawn two threads.
    std::for_each(threads.begin(), threads.end(), std::mem_fn(&std::thread::join)); // Wait for each thread to finish, in turn.


### Choosing the number of threads at runtime

Use the `std::thread::hardware_concurrency()` function to get a hint of how many threads that can truly run concurrently for a given execution of a program. It may return 0 if this information is not available. [Williams12](#Willams12) §2.4


### Identifying threads

Thread identities are of the type `std::thread::id` and can be retrieved in two ways: [Williams12](#Willams12) §2.5
- By calling the `.get_id()` member function of a `std::thread` object (returns a default-constructed instance if this object has no associated thread).
- By calling the `std::this_thread::get_id()` function from within a thread.

`std::thread::id` objects are copyable, comparable, hashable, totally ordered, and serializable, making them useful in many contexts.


### Sharing data between threads


#### Mutexes

Mutexes are the most general of the data-protection mechanisms available in C++. They are represented by the `std::mutex`, `std::recursive_mutex`, `std::timed_mutex` and `std::recursive_timed_mutex` classes. Only one thread can own a mutex at once.
Mutexes are locked with the `.lock()` member function and unlocked with `.unlock()`. View mutexes as a resource; it is usually not recommended to use them directly, but indirectly through the RAII classes `std::lock_guard<>` or `std::unique_lock<>`. [Stroustrup13](#Stroustrup13) §42.3.1.4 [Williams12](#Willams12) §3.2.1

When possible, lock a mutex only while actually accessing the shared data. Try to do any processing of the data outside the lock. In particular, don't do any really time-consuming activities like file I/O while holding a lock. In general, a lock should be held for only the minimum possible time needed to perform the required operation. [Williams12](#Willams12) §3.2.8

**Example:**

    class threadsafe_list {
        std::list<int> list_;    
        std::mutex mut_;
    public: 
        void push_back(int new_value) {
            std::lock_guard<std::mutex> guard(mut_);
            list_.push_back(new_value);
        }
        bool contains(int value) {
            std::lock_guard<std::mutex> guard(mut_);
            return std::find(list_.begin(), list_.end(), value) != list_.end();
        };
        ...
    };


#### Avoiding race conditions

The best way to avoid data races is not to share data: [Stroustrup13](#Stroustrup13) §42.3
- Keep interesting data in local variables, in free store not shared with other threads, or in `thread_local` memory.
- Do not pass pointers to such data to other `std::thread`s.
- When such data needs to be processed by another `std::thread`, pass pointers to a specific section of the data and make sure not to touch that section of the data until after termination of the task.
- When none of these options are possible, use locking through a `std::mutex` or a `std::condition_variable`.

A variable declared `thread_local` is owned by and lives as long as a `std::thread`, and is not accessible from other `std::thread`s. Unless accessed through a pointer, it is therefore not subject to data races between threads, as each `std::thread` has its own copy of all `thread_local` variables.
On some systems the amount of stack storage for a `std::thread` is very limited, so global `thread_local` variables can be used as a mechanism to store large amounts of non-shared data, but since they share the logical problems of normal global variables they are usually best avoided. [Stroustrup13](#Stroustrup13) §42.2.8

Beware of (non-const) `static` class members, as they are subject to data races and are often not easily spotted. Making them `thread_local` fixes the data races but changes the semantics of the class in a way that may not be desired. [Stroustrup13](#Stroustrup13) §42.2.8

It is possible to get race conditions even for objects protected by a mutex if the interface is not carefully designed. [Williams12](#Willams12) §3.2.3

**Example:**

    template <typename T>    
    class not_threadsafe_stack {
        std::stack<T> st_;
        std::mutex mut_;
        ... // Interface like std::stack
    };
    
    void thread_function() {
        not_threadsafe_stack<int> s;
        if (!s.empty()) {
            int value = s.top(); // Bad, another thread may have emptied the stack by now!
            s.pop(); // Bad, another thread may have read the same value from the top of the stack as this thread, causing either this thread or the other thread to pop the next value without either thread having called do_something() with it!
            do_something(value);
        }
    }

Don't pass pointers or references to protected data outside the scope of the lock, whether by returning them from a function, storing them in externally visible memory, or passing them as arguments to user-supplied functions. [Williams12](#Willams12) §3.2.2

**Example:**

    class not_threadsafe_data {
        data data_;    
        std::mutex mut_;
    public: 
        data& get() {
            return data_; // Bad, caller may modify data_ through the returned reference without locking!
        }
        template <typename Function>
        bool process(Function func) {
            std::lock_guard<std::mutex> guard(data_);
            func(data_); // Bad, func may modify data_ through an internal pointer or reference later without locking!
        };
    };


#### Avoiding deadlocks

When one thread function locks a `std::mutex` and calls itself recursively, it deadlocks with itself. To solve this problem, use a `std::recursive_mutex`, which can be locked repeatedly within the same thread. [Stroustrup13](#Stroustrup13) §42.3.1.1

When two threads need access to two or more mutexes at once, they must always be locked in the same order to avoid the possibility of a deadlock. Use the `std::lock()` or `std::try_lock()` functions to ensure this instead of doing it manually. They take two or more `std::mutex` or `std::unique_lock<>` objects and lock all mutexes at once without the possibility of deadlock. [Stroustrup13](#Stroustrup13) §42.3.2 [Williams12](#Willams12) §3.2.4 §3.2.6

A `std::unique_lock<>` object offers more flexibility by allowing deferred locking or early unlocking, which can be important for performance. It can also be used to allow a function to lock a mutex and then return it to transfer ownership to the caller so it can perform additional actions under the protection of the same lock. [Williams12](#Willams12) §3.2.7

**Example:**

    class threadsafe_data {
        data data_;
        std::mutex mut_;
    public:
        ...
        friend void swap_1(const threadsafe_type& lhs, const threadsafe_type& rhs) { // Preferred approach when sufficient.
            if (&lhs != &rhs) {
                std::lock(lhs.mut_, rhs.mut_); // Lock both mutexes (and release all if one fails).
                std::lock_guard<std::mutex> lock_a(lhs.mut_, std::adopt_lock); // std::adopt_lock indicates that the mutex is already locked.
                std::lock_guard<std::mutex> lock_b(rhs.mut_, std::adopt_lock);
                swap(lhs.data_, rhs.data_);
            }
        }
        friend void swap_2(const threadsafe_type& lhs, const threadsafe_type& rhs) { // Later locking, but slightly slower and takes more memory.
            if (&lhs != &rhs) {
                std::unique_lock<std::mutex> lock_a(lhs.mut_, std::defer_lock); // std::defer_lock indicates that the mutex is not yet locked and will be locked later.
                std::unique_lock<std::mutex> lock_b(rhs.mut_, std::defer_lock);
                std::lock(lock_a, lock_b); // Both mutexes are locked here.
                swap(lhs.data_, rhs.data_);
            }
        }
    };

Deadlock can also occur if two or more threads call `.join()` on each other's `std::thread` object, causing them to wait for each other to finish. To avoid this possibility, use the following guidelines: [Williams12](#Willams12) §3.2.5
- Avoid nested locks. If you already hold a lock, never acquire a second one as a separate action. Use `std::lock` to acquire all locks at once.
- Avoid calling user-supplied code when holding a lock. That code may acquire another lock as a separate action.
- If avoiding separate locking actions is unavoidable, define a fixed order of locking that's always consistent between threads.
- Consider establishing a lock hierarchy, preventing a thread from locking a mutex if it already holds a lock from a lower-level mutex.
- Avoid waiting for a thread while holding a lock. The other thread may need to acquire the lock in order to proceed.
- Prefer to join threads in the same function that started them.


#### Protecting shared data during initialization

Use `std::call_once()` together with a `std::once_flag` object instead of a mutex for lazy initialization of expensive resources. [Stroustrup13](#Stroustrup13) §42.3.3 [Williams12](#Willams12) §3.3.1

**Example:**

    std::shared_ptr<resource> resource_ptr;
    std::once_flag resouce_flag; // Keeps track of if the resource has been initialized.
    
    void init_resource() {
        resource_ptr.reset(new resource);
    }
    
    void use_resource() {
        std::call_once(resource_flag, init_resource); // init_resource() Will only be called the first time use_resource() is called.
        resource_ptr->do_something();
    }


### Waiting for threads


#### Condition variables

Use a `std::condition_variable` to manage communication among threads. [Stroustrup13](#Stroustrup13) §42.3.4

A *condition variable* is associated with some event or other condition, and one or more threads can wait for that condition to be satisfied. When some thread has determined that the condition is satisfied, it can then notify one or more of the threads waiting on the condition variable, in order to wake them up and allow them to continue working.
When a condition variable is destroyed, all the threads waiting for it (if any) must be notified, or they may wait forever.

There are two types of condition variables: `std::condition_variable`, which only works together with a `std::mutex`, and `std::condition_variable_any`, which works together with anything that meets some minimal criteria for being mutex-like. Prefer `std::condition_variable` unless extra flexibility is required. [Williams12](#Willams12) §4.1.1

**Example:**

    std::mutex mut;
    std::queue<data_chunk> data_queue;
    std::condition_variable data_cond;
    
    void data_preparation_thread() {
        while (more_data_to_prepare()) {
            const data_chunk data = prepare_data();
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push_back(data);
            data_cond.notify_one(); // Notify any waiting thread that new data is in the queue.
        }
    }
    
    void data_processing_thread() {
        while (true) {
            std::unique_lock<std::mutex> lk(mut);
            data_cond.wait(lk, []{ return !data_queue.empty(); }); // Wait for the lambda to return true (i.e. when there is data in the queue).
            auto data = data_queue.front();
            data_queue.pop();
            lk.unlock();
            process(data);
            if (is_last_chunk(data)) {
                break;
            }
        }
    }

The `.wait()` member function requires a `std::unique_lock` because it will unlock the associated mutex if the lambda returns false, put the current thread in a waiting state and re-lock it to check again once the `.notify_one()` member function has been called by the other thread.

A "plain" `.wait()` is a `.wait()` without the predicate argument. This is a low-level operation that should always be used in a loop, as it allows the current thread to wake up "spuriously" without a notification from another thread. [Williams12](#Willams12) §4.1.1

With `notify_one()`, only one waiting thread will be notified to check its condition again. Use the `.notify_all()` member function if multiple threads should be notified to check their conditions again all at once.

Don't use a function with side-effects to check together with `.wait()`, as it may be called an intederminate number of times. [Williams12](#Willams12) §4.1.1

Consider `void` futures instead of condition variables for one-shot event communication. [Meyers14](#Meyers14) §39


#### Waiting with a time limit

The `std::condition_variable<>` and `std::future<>` variants have `.wait_for()` and `.wait_until()` member functions, with variants that just wait until signaled, or that check a supplied predicate when woken and returns only when the predicate is true or when the timeout has expired. They work with the `std::chrono` `time_point` and `duration` types.


### Putting the current thread to sleep

The `this_thread::sleep_until()` and `this_tread::sleep_for()` functions can be used to put the current thread to sleep. The `this_thread::yield()` function is used to give another `std::thread` a chance to proceed, without explicitly blocking the current thread. Usually, it is better to use `this_thread::sleep()` as this provides the scheduler with a duration, which can be used to make better choices about which threads to run when. Consider `this_thread::yield()` a feature for optimization in very rare and specialized cases. [Stroustrup13](#Stroustrup13) §42.2.6 [Williams12](#Willams12) §4.3.4

If the system clock is reset, `this_thread::wait_until()` is affected, but not `this_thread::wait_for()`.


### Avoiding thread starvation

The C++ standard does not guarantee a fair scheduling of threads, but in reality schedulers are "reasonably fair", making it extremely unlikely that a thread starves forever. [Stroustrup13](#Stroustrup13) §42.3.1


Lock-free programming
---------------------

Primitive operations that do not suffer data races, often called *atomic operations*, can be used in the implementation of higher-level concurrency mechanisms, such as locks, threads, and lock-free data structures. With the notable exception of simple atomic counters, lock-free programming is for specialists. [Stroustrup13](#Stroustrup13) §41.3

`volatile` is *not* a synchronization mechanism, it is for special memory where reads and writes should not be optimized away because it may be accessed by something external to the thread of control. [Meyers14](#Meyers14) §40 [Stroustrup13](#Stroustrup13) §41.4


### Atomics

A synchronization operation on one or more memory locations is a *consume operation*, an *acquire operation*, a *release operation*, or both an acquire and release operation.
- For an *acquire operation*, other processors will see its effect before any subsequent operation's effect.
- For a *release operation*, other processors will see every preceding operation's effect before the effect of the operation itself.
- A *consume operation* is a weaker form of an acquire operation. For a consume operation, other processors will see its effect before any subsequent operation's effect, except that effects that do not depend on the consume operation's value may happen before the consume operation.


#### std::atomic_flag

`std::atomic_flag` is the simplest atomic type, which represents a Boolean flag (one bit). It is the only atomic type that's guaranteed to be statically initialized and atomic for every implementation. It is intended as a basic building block and cannot even be used as a general Boolean flag, but can be used to implement other atomic types. `std::atomic_flag` objects must be initialized with the macro value `ATOMIC_FLAG_INIT`. [Stroustrup13](#Stroustrup13) §41.3.2.1 [Williams12](#Willams12) §5.2.2

**Example:**

    class spinlock_mutex {
        std::atomic_flag flag;
    public:
        spinlock_mutex() : flag(ATOMIC_FLAG_INIT) {}
        void lock() {
            while (flag.test_and_set(std::memory_order_acquire));
        }
        void unlock() {
            flag.clear(std::memory_order_release);
        }
    };


#### std::atomic<bool>

`std::atomic<bool>` is the simplest of the atomic integral types.


#### std::atomic<T*>

`std::atomic<T*>` is the atomic form of a pointer to some type `T`.

The C++ standard library also provides free functions for accessing instances of `std::shared_ptr<>` in an atomic fashion. The atomic operations available are *load*, *store*, *exchange*, and *compare/exchange*, taking a `std::shared_ptr<>*` as the first argument.


#### std::atomic<int>

A simple `std::atomic<int>` variable is close to ideal for a shared counter, such as a use count for a shared data structure. [Stroustrup13](#Stroustrup13) §41.3.1

**Example:**

    template <typename T>
    class shared_ptr {
        T* ptr;
        std::atomic<int>* use_count;
    public:
        ...
        ~shared_ptr {
            if (--*use_count) {
                delete ptr;
            }
        }
    };


#### std::atomic<T>

To use a user-defined type with the `std::atomic<T>` primary class template it must fulfill the following criteria: [Williams12](#Willams12) §5.2.6
- `T` must have be trivially copyable (i.e. no virtual functions or virtual base classes are allowed, allowing copying to be done using `memcpy()`).
- `T` must be bitwise equality comparable (instances must be comparable with `memcmp()`).


### Fences

The standard fences `std::atomic_thread_fence` and `std::atomic_signal_fence` act as memory barriers, restricting operation reordering across the fence according to some specified memory ordering. Fences are used in combination with atomics to enforce memory-order constraints between operations. [Stroustrup13](#Stroustrup13) §41.3.2.2 [Williams12](#Willams12) §5.3.5

**Example:**

    std::atomic_bool x, y;
    
    void write_x_then_y() {
        x.store(true, std::memory_order_relaxed); // Must happen before the load from x.
        std::atomic_thread_fence(std::memory_order_release); // Synchronizes with acquire fence.
        y.store(true, std::memory_order_relaxed);
    }
    
    void read_y_then_x() {
        while (!y.load(std::memory_order_relaxed);
        std::atomic_thread_fence(std__memory_order_acquire); // Synchronizes with release fence.
        if (x.load(std::memory_order_relaxed)) { // Must happen after the store to x.
            ...
        }
    }


Designing concurrent data structures
------------------------------------

Data structures that use mutexes, condition variables, and futures to synchronize data are called *blocking*. The application calls library functions that will suspend the execution of a thread until another thread performs an action.
Data structures that don't use blocking library functions are said to be *non-blocking*. Not all such data structures are *lock-free*.
For a data structure to qualify as *lock-free*, more than one thread must be able to access the data concurrently, but they don't have to be able to do the same operations concurrently.
A *wait-free* data structure is a *lock-free* data structure with the additional property that every thread accessing the data structure can complete its operation within a bounded number of steps, regardless of the behavior of other threads. Writing *wait-free* data structures correctly is extremely hard.

Guidelines for designing concurrent data structures: [Williams12](#Willams12) §6.1.1
- Ensure that no thread can see a state where the invariants of the data structure have been broken by the actions of another thread.
- Take care to avoid race conditions inherent in the interface to the data structure by providing functions for complete operations rather than for operation steps.
- Pay attention to how the data structure behaves in the presence of exceptions to ensure that the invariants are not broken.
- Minimize the opportunities for deadlock when using the data structure by restricting the scope of locks and avoiding nested locks where possible.

Questions to ask yourself as the data structure designer: [Williams12](#Willams12) §6.1.1
- Can the scope of locks be restricted to allow some parts of an operation to be performed outside the lock?
- Can different parts of the data structure be protected with different mutexes?
- Do all operations require the same level of protection?
- Can a simple change to the data structure improve the opportunities for concurrency without affecting the operational semantics?


### Lock-free concurrent data structures

Advantages of *lock-free* data structures: [Williams12](#Willams12) §7.1
- Enables maximum concurrency.
- Robustness. If a thread dies while holding a lock, that data structure is broken forever.
- Deadlocks are impossible (but there is still a possibility for live locks).

Disadvantages of *lock-free* data structures: [Williams12](#Willams12) §7.1
- Complexity. Considerably harder to achieve correctness.
- Overall performance may decrease, because atomic operations can me much slower than their corresponing non-atomic operations, and there is likely to be more of them in a lock-free data structure.
- Hardware has to synchronize data between threads, which can be a significant performance drain.

Guidelines for designing lock-free data structures: [Williams12](#Williams12) §7.3
- Use `std::memory_order_seq_cst` for prototyping, and relax the memory ordering constraints to optimize only once the basic operations are working.
- Use a lock-free memory reclamation scheme, e.g. waiting until no threads are accessing the data structure and deleting all objects that are pending deletion, using *hazard pointers* to identify that a thread is accessing a particular object, or reference counting the objects so that they aren't deleted until there are no outstanding references.
- Watch out for the *ABA problem*. It is particularly prevalent in algorithms that use free lists or otherwise recycle nodes rather than returning them to the allocator.
- Identify busy-wait loops and help the other thread.


References
----------

<a name="Stroustrup13"></a>
[Stroustrup13]
"The C++ Programming Language, 4th Edition", ISBN 978-0-321-56384-2

<a name="Williams12"></a>
[Williams12]
"C++ Concurrency in Action, Practical", ISBN 978-1-933-98877-1

<a name="Meyers14"></a>
[Meyers14]
"Effective Modern C++", ISBN 978-1-491-90399-5
