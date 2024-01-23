---
title: "Sneaky Undefined Behavior in C++ Multithreading"
date: 2023-07-18
image: cover.png
categories:
    - Language Learning
tags:
    - C++
    - Multithreading
weight: 1
---

## The Discovery

Recently while performing some unimportant updates to a C++ codebase, I noticed an interesting issue arise.

When attempting to signal some worker threads to complete, they refused to end. After investigating extensively, I could not understand how my changes had caused this issue. The modifications weren't even touching these zombie threads.

After inspecting the threads in question further, a realization hit me - the compiler had optimized out a boolean. 

### This snippet of code has a bug

```c++
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

Do you notice anything wrong here?

If you add this snippet to some test code, it most likely will compile and run just as desired. Probably, after setting run to false, it will end `workerThread` and allow `.join()` to complete.

However, if you're unlucky enough you might compile and discover the thread never ends. This interesting issue is because passing the bool to the thread's lambda function, even by reference, causes undefined behavior.

According to the compiler, the `run` variable could be safely optimized out, as clearly the while loop was always meant to be `while(true)`.

## Undefined Behavior: 90% of the time it works every time

Undefined behavior is a tricky topic in C and C++. A program might "get away" with running bug free while containing undefined behavior.

However, the entire program is technically [meaningless](https://cryptoservices.github.io/fde/2018/11/30/undefined-behavior.html). Since the results generated from the program are non-deterministic, it may work flawlessly one day, then start producing erroneous results the next.

UB is also tricky to detect. There are static analysis tools available, but even they struggle to detect all instances. It is left to the programmer to understand the cost of the code they write and to understand the repercussions of adding it to a codebase, including whether or not it has the potential to introduce undefined behavior.

### A variable of a primitive data type accessed by multiple threads without using a mutex or semaphore is undefined behavior

This is because primitive types in C++ do not have several characteristics required for safe multithreaded access. These types do not have a specified memory order or synchronization mechanism. This means when two threads attempt to access a single variable of these types, there is no defined way to ensure a data race does not occur.

## The Solution: Atomics

There are several ways to fix the issue. You could do proper locking and synchronize the main and worker threads when accessing `run`. Another solution that may provide better performance is assigning multithreaded behavior to the variable itself. This can be done by wrapping the primitive type in an [atomic variable](https://en.cppreference.com/w/cpp/atomic/atomic). After doing so, the variable can be used like any other bool, except now we have removed the undefined behavior!

### `run` is now thread safe
```c++
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

## Why not use Volatile?

For those with experience in C, a tempting solution might be to use the qualifier `volatile`. At first glance, this makes sense. If a variable is marked `volatile`, you can avoid those pesky compiler optimizations that occur as a result of undefined behavior.

However, after digging a little one can find some interesting details on this proposed solution. The `volatile` qualifier has its own flavor of undefined behavior. Namely, a [volatile variable](https://en.cppreference.com/w/c/language/volatile) is not guaranteed to have synchronization or a specified memory ordering.

### Any UB results in the entire program becoming undefined

Without these properties, accessing a volatile variable concurrently is undefined behavior. As discussed previously, introducing convenient UB into a program is dangerous and can have unforeseen impacts later in a program's lifecycle. 

## References / Further Reading
- https://cryptoservices.github.io/fde/2018/11/30/undefined-behavior.html
- https://en.cppreference.com/w/cpp/atomic/atomic
- https://en.cppreference.com/w/c/language/volatile
