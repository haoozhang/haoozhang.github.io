---
layout:     post
title:      Chapter 30. Hybrid Thread Synchronization Constructs
subtitle:   
date:       2022-12-06
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

In Chapter 29, "Primitive Thread Synchronization Constructs," I discussed the primitive user-mode and kernel-mode thread synchronization constructs. From these primitive constructs, all other thread synchronization constructs can be built. Typically, other thread synchronization constructs are built by combining the user-mode and kernel-mode constructs, and I call these **hybrid thread synchronization constructs**. \
When there is no thread contention, hybrid constructs provide the performance benefit of the primitive user-mode constructs. When multiple threads are contending for the construct at the same time, hybrid constructs also use the primitive kernel-mode constructs to provide the benefit of not spinning.

This chapter will cover:
+ how hybrid constructs are built from the various primitive con- structs.
+ many of the hybrid constructs that ship with the FCL.
+ FCL’s concurrent collection classes.
+ asynchronous synchronization constructs.

## A Simple Hybird Lock

Let me start off by showing you an example of a hybrid thread synchronization lock.

```c#
 internal sealed class SimpleHybridLock : IDisposable {
    // The Int32 is used by the primitive user­mode constructs (Interlocked methods)
    private Int32 m_waiters = 0;
    
    // The AutoResetEvent is the primitive kernel­mode construct
    private readonly AutoResetEvent m_waiterLock = new AutoResetEvent(false);
    
    public void Enter() {
        // Indicate that this thread wants the lock
        if (Interlocked.Increment(ref m_waiters) == 1)
           return; // Lock was free, no contention, just return
        
        // Another thread has the lock (contention), make this thread wait
        m_waiterLock.WaitOne();  // Bad performance hit here
        // When WaitOne returns, this thread now has the lock
    }
    
    public void Leave() {
        // This thread is releasing the lock
        if (Interlocked.Decrement(ref m_waiters) == 0)
            return; // No other threads are waiting, just return
        
        // Other threads are waiting, wake 1 of them
        m_waiterLock.Set();  // Bad performance hit here
    }
    
    public void Dispose() { m_waiterLock.Dispose(); }
}
```

The **SimpleHybridLock** contains two fields: an Int32, which will be manipulated via the primitive user-mode constructs, and an **AutoResetEvent**, which is a primitive kernel-mode construct. To get great performance, the lock tries to use the Int32 and avoid using the **AutoResetEvent** as much as possible. Note that the creating and closing **AutoResetEvent** will cause big performance hit.

The first thread to call **Enter** causes **Interlocked.Increment** to add one to the **m_waiters** field, making its value 1. This thread sees that there were zero threads waiting for this lock, so the thread returns from **Enter** very quickly. Now, if another thread comes along and calls **Enter**, this second thread increments **m_waiters** to 2 and sees that another thread has the lock, so this thread blocks by calling **WaitOne** using the **AutoResetEvent**. This operation causes the thread to transition into the kernel, which is a big performance hit. However, it can make the thread blocked, instead of spinning on the CPU.

Now let’s look at the Leave method. When a thread calls **Leave**, **Interlocked.Decrement** is called to subtract 1 from the **m_waiters**. If **m_waiters** is now 0, then no other threads are blocked inside a call to **Enter** and the thread calling **Leave** can simply return. On the other hand, if the thread calling **Leave** sees that **m_waiters** was not 1, then the thread knows that there is contention and that there is at least one other thread blocked in the kernel. This thread must wake up one (and only one) of the blocked threads. It does this by calling **Set** on **AutoResetEvent**.

## Spinning, Thread Ownership, and Recursion

Because transitions into the kernel incur such a big performance hit and threads tend to hold on a lock for very short time, we can improve the overall performance by having a thread spin in user mode for a little before having the thread transition to kernel mode. If the lock becomes available while spinning, then the transition to kernel mode is avoided.

In addition, some locks impose a limitation where the thread acquiring the lock must be the thread releasing the lock. And some locks allow the currently owning thread to require the lock recursively.

Using some fancy logic, it is possible to build a hybrid lock that offers spinning, thread ownership, and recursion. Here is what the code looks like.

```c#
internal sealed class AnotherHybridLock : IDisposable {
    // The Int32 is used by the primitive user­mode constructs (Interlocked methods)
    private Int32 m_waiters = 0;

    // The AutoResetEvent is the primitive kernel­mode construct
    private AutoResetEvent m_waiterLock = new AutoResetEvent(false);

    // This field controls spinning in an effort to improve performance
    private Int32 m_spincount = 4000;   // Arbitrarily chosen count

    // These fields indicate which thread owns the lock and how many times it owns it
    private Int32 m_owningThreadId = 0, m_recursion = 0;

    public void Enter() {
        // If calling thread already owns the lock, increment recursion count and return
        Int32 threadId = Thread.CurrentThread.ManagedThreadId;
        if (threadId == m_owningThreadId) { m_recursion++; return; }
       
        // The calling thread doesn't own the lock, try to get it
        SpinWait spinwait = new SpinWait();
        for (Int32 spinCount = 0; spinCount < m_spincount; spinCount++) {
            // If the lock was free, this thread got it; set some state and return
            if (Interlocked.CompareExchange(ref m_waiters, 1, 0) == 0) goto GotLock;
            // Black magic: give other threads a chance to run
            // in hopes that the lock will be released
            spinwait.SpinOnce();
        }   
      
        // Spinning is over and the lock was still not obtained, try one more time
        if (Interlocked.Increment(ref m_waiters) > 1) {
            // Still contention, this thread must wait
            m_waiterLock.WaitOne(); // Wait for the lock; performance hit
            // When this thread wakes, it owns the lock; set some state and return
        }
   
    GotLock:
        // When a thread gets the lock, we record its ID and
        // indicate that the thread owns the lock once
        m_owningThreadId = threadId; m_recursion = 1;
    }

    public void Leave() {
        // If the calling thread doesn't own the lock, there is a bug
        Int32 threadId = Thread.CurrentThread.ManagedThreadId;
        if (threadId != m_owningThreadId)
            throw new SynchronizationLockException("Lock not owned by calling thread");
 
        // Decrement the recursion count. If this thread still owns the lock, just return
        if (­­m_recursion > 0) return;
      
        m_owningThreadId = 0;   // No thread owns the lock now
        
        // If no other threads are waiting, just return
        if (Interlocked.Decrement(ref m_waiters) == 0)
            return;
        
        // Other threads are waiting, wake 1 of them
        m_waiterLock.Set();     // Bad performance hit here
    }
   
    public void Dispose() { m_waiterLock.Dispose(); }
}
```

As you can see, adding extra behavior to the lock increases the number of fields, which increases its memory consumption, and also decreases the lock's performance.

## Hybrid Constructs in FCL


## 