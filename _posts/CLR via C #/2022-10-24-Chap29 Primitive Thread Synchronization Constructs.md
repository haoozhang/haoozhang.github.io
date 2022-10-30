---
layout:     post
title:      Chapter 29. Primitive Thread Synchronization Constructs
subtitle:   
date:       2022-10-24
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

In chapter 27, we focused on how to use existing threads to perform compute-bound operations, and chapter 28 focused on how to use threads to perform IO-bound operations. In this chapter, I focus on **thread synchronization**. 

Thread synchronization is used to prevent corruption when multiple threads access shared data **at the same time**. I emphasize **at the same time**, because thread synchronization is all about timing. If you have data that is accessed by two threads and those threads cannot possibly touch the data simultaneously, then thread synchronization is not required. For example, in chapter 28, I discussed the async functions can be executed by different threads, but it is impossible for two threads to access this same data at the same time. Therefore, no thread synchronization is required for async functions.

It's better to avoid thread synchronization, because it has many problems associated with it.
+ It is tedious and extremely error-prone. You should identify all data that could be multiple threads at the same time, and wrap all this code with additional code that acquires and releases a thread synchronization lock. If you forget to surround just one block, then the data will become corrupted.
+ It hurt performance. It takes time to acquire and release a lock because there are additional method calls. For example, let's say that you have code that adds a node to the head of a linked list.
```c#
// LinkedList class
public class Node {
    internal Node m_next;
    // Other members not shown
}

public sealed class LinkedList {
    private Node m_head;
    
    public void Add(Node newNode) {
        // The following two lines perform very fast reference assignments
        newNode.m_next = m_head;
        m_head = newNode;
    }
}
```

Now, if we want to make **Add** thread safe so that multiple threads can call it simultaneously without corrupting the linked list, then we need to have the Add method acquire and release a lock.

```c#
public sealed class LinkedList {
    private SomeKindOfLock m_lock = new SomeKindOfLock();
    private Node m_head;
    
    public void Add(Node newNode) {
        m_lock.Enter();
        // The following two lines perform very fast reference assignments
        newNode.m_next = m_head;
        m_head = newNode;
        m_lock.Leave();
    }
}
```

+ Blocking a thread causes more threads to be created, but creating a new thread is very expensive in terms of memory and performance. Besides, blocking thread will run with this new thread, which causes that Windows is scheduling more threads than CPUs, and increased context switching.

The summary of all of this is that thread synchronization is bad, and you should avoid it. To that end,
+ Avoid shared data such as **static** fields. When a thread uses the **new** operator to construct an object, the **new** operator returns a reference to the new object. At this point in time, only the thread that constructs the object has a reference to it; no other thread can access that object.
+ Try to use value types because they are always copied, so each thread operates on its own copy.
+ it is OK to have multiple threads accessing shared read-only data simultaneously. For example, after a **String** object is created, it is immutable; so many threads can access a single **String** object at the same time without any chance of the **String** object becoming corrupted.

## Class Libraries and Thread Safety

Microsoft Framework Class Library (FCL) guarantees that **all static methods are thread safe**. This means that if two threads call a static method at the same time, no data will get corrupted.

The **Console** class contains a **static** field named as **InternalSyncObject**, many of its methods acquire and release thsi lock to ensure that only one thread at a time is accessing the console. \
The **System.Math.Max** static method is thread safe even though it doesn’t take any lock. Because **Int32** is a value type, the two **Int32** values passed to **Max** are copied into it.

FCL does not guarantee that instance methods are thread safe because adding all the locking code would hurt performance too much. 

As mentioned earlier, when a thread constructs an object, only this thread has a reference to the object, no other thread can access that object, and no thread synchronization is required. \
However, if the thread exposes this reference, by placing in **static** field, passing into **ThreadPool.QueueUserWorkItem** or **Task**, then thread synchronization is required if the threads could perform non-read-only access.

## Primitive User-Mode and Kernel-Mode Constructs

In this chapter, I explain the **primitive** thread synchronization constructs. By **primitive**, I mean the simplest constructs that are available to use in your code.

There are two kinds of primitive constructs: **user-mode** and **kernel-mode**.
+ primitive user-mode constructs: \
faster than the kernel-mode constructs because they use special CPU instructions to coordinate threads. \
blocked on a user-mode primitive construct is never considered blocked, thread pool will not create a new thread to replace the temporarily blocked thread. We call this a **livelock**.
+ primitive kernel-mode constructs: \
a thread uses a kernel-mode construct to acquire a resource that another thread has, Windows blocks the thread so that it is no longer wasting CPU time. \
If the thread holding the context never release it, the thread is blocked forever, and we call this a **deadlock**.

We would like to have constructs that take the best of both. That is, we'd like construct that is fast and non-blocking when there is no contention (like user-mode constructs). But when there is contention for the construct, we’d like it to be blocked by the operating system kernel. This kind of construct do exist, and are called **hybrid constructs**.

## User-Mode Constructs


## Kernel-Mode Constructs