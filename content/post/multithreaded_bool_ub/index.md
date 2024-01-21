---
title: First Post
date: 2023-07-18
image: cover.jpg
categories:
    - Education
tags:
    - C++
    - Multithreading
weight: 1
---

# Sneaky Undefined Behavior in C++ Multithreading: Accessing `bool` Flags

## The Discovery

Recently I came across something interesting while working on some multithreaded C++ code. While modifying some code unrelated code, I noticed the application refused to end as expected. After running it through a debugger and digging in a bit, I noticed an interesting bug. On some code within a completely different portion of the application, the compiler had optimized out a boolean. Below is an example of the code's functionality:

```
bool run = true;
std::thread workerThread([&] {
    while(run) {
        doWork();
    }
});

std::this_thread::sleep_for(std::chrono::seconds(1));
run = false;
workerThread.join();
```

Do you notice anything wrong with this snippet? If you add this snippet to some test code, it most likely will compile and run just as desired. If you set run to false, it will end the `workerThread`. However, if you're unlucky enough you might compile and discover the thread never ends. This is because passing the bool to the thread's lambda function, even by reference, causes undefined behavior.

According to the compiler, the `run` variable could be safely optimized out, as clearly the while loop was always meant to be `while(true)`.

## Undefined Behavior: 60% of the time it works every time

Undefined behavior is a tricky topic in C and C++. A program might "get away" with running bug free while containing undefined behavior. However, the entire program is technically [meaningless](https://cryptoservices.github.io/fde/2018/11/30/undefined-behavior.html) as the results generated from it are non-deterministic. The program may work flawlessly one day, then start producing erroneous results the next. UB is also tricky to detect. 

There are static analysis tools available, but even they struggle to detect all instances. It is left to the programmer to understand the cost of the code they write and to understand the repercussions of adding it to a codebase, including whether or not it has the potential to introduce undefined behavior.

A variable accessed by multiple threads without using a mutex or semaphore is undefined behavior. This is because primitive types in C++ do not have several characteristics required for safe multithreaded access. These types do not have a specified memory order or synchronization mechanism. This means when two threads attempt to access a single variable of these types, there is no defined way to ensure a data race does not occur.

Two un-synchronized threads accessing the same variable can understandably have issues with large data types such as an `int` or `double`. But what about a `char` or `bool`? Each typically consists of a single byte, so maybe it could avoid some pitfalls of being accessed mid-write, correct? Well, not exactly. Even if you dig in to how cache works when accessed concurrently on most multicore CPUs (it usually locks by default), diving to a lower level is ignoring the initial problem completely. Regardless of the implementation of a primitive type, the c++ standard specifies this type of access as undefined behavior.

## The Solution: Atomics

There are several ways to fix the issue. You could do proper locking and synchronize the main and worker threads when accessing `run`. Another solution that may provide better performance is assigning multithreaded behavior to the variable itself. This can be done by wrapping the primitive type in an [atomic variable](https://en.cppreference.com/w/cpp/atomic/atomic). After doing so, the variable can be used like any other bool, except now we have removed the undefined behavior!

```
std::atomic<bool> run {true};
std::thread workerThread([&] {
    while(run) {
        doWork();
    }
});

std::this_thread::sleep_for(std::chrono::seconds(1));
run = false;
workerThread.join();
```

With that simple code addition, we have now made our `run` variable thread safe.

## Why not use Volatile?

For those with experience in C, a tempting solution might be to use the qualifier `volatile`. At first glance, this makes sense. If a variable is marked `volatile`, you can avoid those pesky compiler optimizations that occur as a result of undefined behavior.

However, after digging a little one can find some interesting details on this proposed solution. The `volatile` qualifier has its own flavor of undefined behavior. Namely, a [volatile variable](https://en.cppreference.com/w/c/language/volatile) is not guaranteed to have synchronization or a specified memory ordering [a]. Meaning, when accessing te variable concurrently, it is still undefined behavior. As discussed above on the subject of UB, introducing any into a program results in the entire program becoming undefined.

## References / Further Reading
- https://cryptoservices.github.io/fde/2018/11/30/undefined-behavior.html
- https://en.cppreference.com/w/cpp/atomic/atomic
- https://en.cppreference.com/w/c/language/volatile
