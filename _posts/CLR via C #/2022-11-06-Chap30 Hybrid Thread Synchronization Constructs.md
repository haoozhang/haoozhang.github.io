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

The FCL ships with many hybrid constructs that use fancy logic to keep your threads in user mode, improving your application’s performance. Some of these hybrid constructs also avoid creating the kernel-mode construct until the first time threads contend on the construct. If threads never contend on the construct, then your application avoids the performance hit of creating the object.

### ManualResetEventSlim and SemaphoreSlim

These two constructs work exactly like their kernel-mode constructs, except that both spinning in user mode, and they both defer creating the kernel-mode construct until the first time contention occurs.

Their **Wait** methods allow to pass a **timeout** and a **CancellationToken**. Here is what these classes look like.
```c#
public class ManualResetEventSlim : IDisposable {
    public ManualResetEventSlim(Boolean initialState, Int32 spinCount);
    public void Dispose();
    public void Reset();
    public void Set();
    public Boolean Wait(Int32 millisecondsTimeout, CancellationToken     cancellationToken);
    public Boolean IsSet { get; }
    public Int32 SpinCount { get; }
    public WaitHandle WaitHandle { get; }
}   

public class SemaphoreSlim : IDisposable {
    public SemaphoreSlim(Int32 initialCount, Int32 maxCount);
    public void Dispose();
    public Int32 Release(Int32 releaseCount);
    public Boolean Wait(Int32 millisecondsTimeout, CancellationToken     cancellationToken);
    // Special method for use with async and await (see Chapter 28)
    public Task<Boolean> WaitAsync(Int32 millisecondsTimeout, CancellationToken cancellationToken);
    public Int32 CurrentCount { get; }
    public WaitHandle AvailableWaitHandle { get; }
}
```

Although there is no **AutoRestEventSlim** class, we can construct a **SemaphoreSlim** object that **maxCount** is 1.

### Monitor and Sync Blocks

Every object on the heap can have a data structure, called a **sync block**. It maintains a kernel object, the owning thread’s ID, a recursion count, and a waiting threads count. 

The **Monitor** class is a static class whose methods accept a reference to any heap object, and these methods manipulate the fields in the specified object’s sync block.

Now obviously, associating a sync block data structure with every object in the heap is quite wasteful, because most objects’ sync blocks are never used. CLR team uses a more efficient way to offer: 
+ When the CLR initializes, it allocates an array of sync blocks in native heap.
+ When an object is constructed, the object’s sync block index is initialized to -1, which indicates that it doesn’t refer to any sync block.
+ when **Monitor.Enter** is called, the CLR finds a free sync block in the array and sets the object’s sync block index to refer to the sync block.
+ When **Exit** is called, it checks whether there are any more threads waiting to use the object’s sync block. If there are no threads waiting for it, the sync block is free, **Exit** sets the object’s sync block index back to -1.

But there is a problem that each object's sync block index is implicitly public. So **Monitor.Enter** will get a public lock, which may cause thread blocked.

it is so common for developers to take a lock, do some work, and then release the lock within a single method, the C# language offers simplified syntax via its **lock** keyword. 

```c#
private void SomeMethod() {
    lock (this) {
        // This code has exclusive access to the data...
    }
}
```

It's equivalent to having written the method like this.

```c#
private void SomeMethod() {
    Boolean lockTaken = false;
    try {
        // An exception (such as ThreadAbortException) could occur here...
        Monitor.Enter(this, ref lockTaken);
        // This code has exclusive access to the data...
    }
    finally {
        if (lockTaken) Monitor.Exit(this);
    }
}
```

If an exception occurs inside the **try** block while changing state, then the state is now corrupted. When the lock is exited in the finally block, another thread will now start manipulating the corrupted state. So the **Boolean lockTaken** variable is introduced. If **Monitor.Enter** is called and successfully takes the lock, it sets **lockTaken** to true. The finally block examines **lockTaken** to know whether to call **Monitor.Exit** or not.

### ReaderWriterLockSlim

If all the threads want to read-only the data, then there is no need to block them; they should all be able to access the data concurrently. On the other hand, if a thread wants to modify the data, then this thread needs exclusive access to the data. The  The **ReaderWriterLockSlim** construct solves this problem.

+ When one thread is writing to the data, all other threads requesting access are blocked.
+ When one thread is reading from the data, other threads requesting read access are allowed to continue executing, but threads requesting write access are blocked.
+ When a thread writing to the data has completed, either a single writer thread is unblocked or all the reader threads are unblocked. If no threads are blocked, then the lock is free and available for the next reader or writer thread that wants it.
+ When all threads reading from the data have completed, a single writer thread is unblocked so it can access the data. If no threads are blocked, then the lock is free and available for the next reader or writer thread that wants it.

Here is some code that demonstrates the use of this construct.

```c#
internal sealed class Transaction : IDisposable {
    private readonly ReaderWriterLockSlim m_lock = new ReaderWriterLockSlim(LockRecursionPolicy.NoRecursion);
    private DateTime m_timeOfLastTrans;
   
    public void PerformTransaction() {
        m_lock.EnterWriteLock();
        // This code has exclusive access to the data...
        m_timeOfLastTrans = DateTime.Now;
        m_lock.ExitWriteLock();
    }
   
    public DateTime LastTransaction {
        get {
            m_lock.EnterReadLock();
            // This code has shared access to the data...
            DateTime temp = m_timeOfLastTrans;
            m_lock.ExitReadLock();
            return temp;
        } 
    }
    
    public void Dispose() { m_lock.Dispose(); }
}
```

**Reader­WriterLockSlim’s** constructor allows you to pass in a **LockRecursionPolicy** flag.

```c#
public enum LockRecursionPolicy { NoRecursion, SupportsRecursion }
```

If you pass the **SupportsRecursion** flag, then the lock will add thread ownership and recursion behaviors to the lock, which will cause performance hit. So **NoRecursion** is recommended.

### CountdownEvent

This construct blocks a thread until its internal counter reaches 0.

### Barrier

**Barrier** is used to control a set of threads that are working together
in parallel so that they can step through phases of the algorithm together. 
+ When you construct a **Barrier**, you tell it how many threads are participating in the work, and you can also pass an **Action\<Barrier>** delegate referring to code that will be invoked whenever all participants complete a phase of the work.
+ As each thread completes its phase of the work, it should call **Signal­AndWait**, which tells the **Barrier** that the thread is done and the **Barrier** blocks the thread.
+ After all participants call **SignalAndWait**, the **Barrier** invokes the delegate and then unblocks all the waiting threads so they can begin the next phase.


## Double-Check Locking 


## 