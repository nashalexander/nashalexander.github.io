---
title: "How to understand C++ Containers Intuitively to Master LeetCode"
date: 2024-12-22
image: cover.webp
categories:
    - Learning
tags:
    - C++
    - LeetCode
---

## Intro

The journey of conquering LeetCode is a task for many who aim to do well at software interviews. The struggles encountered while taking on these challenges can vary quite drastically from those typically experienced in day-to-day work life.

For C++, those struggles can quickly occur when attempting to create solutions using STL collections efficiently. Questions such as, "is a vector best here, or should I use an array?" or "Would this be a queue or a deque?" may arise when considering the best way to solve a problem and achieve optimal big O complexity. Personally, I had a lot of these questions in my head when first attempting these challenges in C++. Understanding a solution at a high level and even using C with manually implemented data structures was one thing, but leveraging C++ STL and concepts felt like another subject all together.

If you come from a C background, you may already think in terms of how the application should function at a fundamental computer level. Things like allocating memory, data locality, and leveraging pointers are common ideas when dealing directly with C. With a fundamental understanding of C++ collections, this knowledge can be leveraged efficiently by using the concepts of a C implementation as the backbone of a solution. Once you can conceptualize how the solution would work in a pure C manner, C++ collections that satisfy those requirements can be leveraged to accelerate the creation of your solution.

Thinking of problems in a C-first way, and then applying C++ containers to satisfy those ideas, can be a game-changer.

## Understanding C++ Containers

### [Vector](https://en.cppreference.com/w/cpp/container/vector)

The deceiving member functions of `push_back` and `pop_back` may lead you to believe a vector is implemented as a linked list. This is not the case, a vector is essentially a fancy contiguous array with an unbounded size. The differences between a traditional array in C and a C++ vector include the class interface and how each handles its memory allocations.

A traditional C array allocates exactly the size defined when it is declared. When an attempt is made to access an element beyond its size, the array would be reaching beyond the bounds of its memory and the operation would trigger a segfault.

A vector works differently. When a vector object is created it pre-reserves a buffer of memory for an unspecified length of elements. This pre-reserved block can be checked with the member function `capacity`. As new values are added to this vector, it is filling in this reserved contiguous block of memory.

Now, what happens when this reserved amount of memory is exceeded? When a vector has run out of reserved space it will reserve a new larger contiguous space of memory. Afterwards the vector will perform a complete copy of its data to the new chunk of reserved space. It will then free the old buffer it no longer requires.

For example, if you create a vector that happens to reserve 5000 elements worth of space and you then go to append a 5001th value, you would then trigger a complete reallocation of that vector. The vector would reserve a brand new block of memory, let's say one that holds 10,000 integers, then it would copy over the previous 5000 integers to this new continuous block of memory. It would then free the old memory block.

```c++
std::vector<int> vec;
vec.size();     // 0
vec.capacity(); // Let's say this returns 5000

for(int i = 0 ; i < 5000 ; i++){
    vec.emplace_back(i);
}
vec.size();     // 5000
vec.capacity(); // 5000

vec.emplace_back(1); // Triggers a reallocation of the vector
vec.size();     // 5001
vec.capacity(); // 10000
```

This property of vectors to reallocate memory has its benefits and its trade offs. Because the memory is stored in a contiguous memory block, its access can benefit from CPU caching and avoid cache misses when processing large amounts of data sequentially. However, because of its reallocation property, the container pays a cost when using large amounts of data.  


### [Deque](https://en.cppreference.com/w/cpp/container/deque)

Another interesting container to analyze is the double-ended queue, deque. Deceivingly this container is also not implemented as a linked list. Deque is similar to vector in that when an object of type deque is declared, it reserves a contagious block of memory.

A key difference between deque and vector is the algorithm they perform when they run out of reserved memory space. Deque is actually collection of separate memory blocks.

A deque, once it goes beyond its reserved memory block, does not waste the time to reallocate and copy its contents anywhere else. This container leaves the contents as they are in their specific memory region. Instead, it allocates a brand new memory region for the new data. The deque itself holds pointers to the start of each separate memory region it contains.  Thus, accessing a value by index requires dereferencing two pointers: one for the location of the memory block containing the index, and another for the index value itself.

Deque does not provide a contiguous block of memory for its entire contents, thus it suffers from cache misses at the borders of its memory blocks. It does however save computation time when dealing with large amounts of data, requiring no movement of data through memory.

### Key Trade Offs

* **Vector**: Best for data that will not significantly grow in size, and requires sequential access.
* **Deque**: Best for data that will grow in size significantly, with an understanding that sequential access may have a trade off.

When an array-like collection is needed, either vector or deque could be selected. The key trade-offs list above should be considered when making the decision.

Whether or not your data structure subscribes to the traditional concept of a linked list or a queue has no relevance.

### [Container Adapters](https://en.cppreference.com/w/cpp/container#Container_adaptors)

Container adapters within C++ are not true containers. Members of the container adapter group are not unique containers, they are instead interfaces used as a wrapper around other containers. These adapters provide restricted access patterns with the added benefit of a configurable data management layer. 

Some important container adapters are `stack` and `queue`, each are used to implement common computer science data structures. Each has a container parameter in its template definition. The default is typically deque, but in some instances it may be best to leverage a vector or some other container type.

Thinking about container adapter classes as interface wrappers of underlying containers can help break up the decision of interface and implementation. Each can be individually determined between what member functions will need to be accessible (or restricted), and what the optimal memory layout and performance would look like.

## The Next Level: Chaining Containers

A powerful concept when solving complex problems is container chaining. Essentially, this means using two or more containers with different time complexity profiles to optimize data access.

### Creating a Least recently used (LRU) Cache with container chaining

Problem description of [LRU Cache](https://en.wikipedia.org/wiki/Cache_replacement_policies#LRU):

> Implement the LRUCache class:
> * LRUCache(int capacity) Initialize the LRU cache with positive size capacity.
> * int get(int key) Return the value of the key if the key exists, otherwise return -1.
> * void put(int key, int value) Update the value of the key if the key exists. Otherwise, add the key-value pair to the cache. If the number of keys exceeds the capacity from this operation, evict the least recently used key.
> The functions get and put must each run in O(1) average time complexity.

If you'd like to try your hand at solving [this problem from LeetCode](https://leetcode.com/problems/lru-cache/description/) now without any spoilers, please do so before continuing.

**Solution:**
O(1) time complexity is required for access, insertion, and deletion. In addition, the data should be stored in a queue-like data structure, with O(1) complexity to access and remove the end elements.

Containers can be mixed and matched by linking them together. For this example:
* A map and unordered_map would have O(1) time complexity for access, insertion, and deletion. However, they do not have a queue based layout and O(1) access to the end elements.
* A linked list would provide a malleable queue structure, but insertion time at some index value would be O(n) complexity as the entire linked list would need to be iterated through until the desired index is reached.

In practice, chaining may look something like the following implementation of LRU Cache. Starting with a key/value pair as our data, we can have a linked list store  

```C++
class LRUCache {
public:
    LRUCache(int capacity) : capacity(capacity)
    {}
    
    int get(int key) {
        // Check if exists
        if(values.count(key) == 0) {
            return -1;
        }

        // Should never occur, could be an assert instead
        if(cacheAddr.count(key) == 0) {
            return -1;
        }

        // Refresh cache for key
        cacheQueue.erase(cacheAddr.at(key));
        pushToCache(key);

        // Get value
        return values.at(key);
    }
    
    void put(int key, int value) {
        // Remove cache of key if it already exists
        if(cacheAddr.count(key) > 0) {
            cacheQueue.erase(cacheAddr.at(key));
        }
        
        // Add key to cache
        pushToCache(key);

        // Pop LRU cache if over capacity
        if(cacheQueue.size() > capacity){
            values.erase(cacheQueue.front());
            cacheAddr.erase(cacheQueue.front());
            cacheQueue.pop_front();
        }

        // Do put
        values[key] = value;
    }

private:
    int capacity;
    std::unordered_map<int,int> values;

    // Used to handle the queue functionality of LRU cache
    std::list<int> cacheQueue;

    // Used to store pointers to the elements in cacheQueue for O(1) refresh
    std::unordered_map<int, std::list<int>::iterator> cacheAddr;

    // Handle updating cache containers when pushing a val to cache
    void pushToCache(int key) {
        cacheQueue.push_back(key);
        cacheAddr[key] = cacheQueue.end();
        cacheAddr[key]--;
    }
};
```