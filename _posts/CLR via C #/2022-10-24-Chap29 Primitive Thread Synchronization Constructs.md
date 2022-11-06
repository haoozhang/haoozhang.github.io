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

FCL **does not guarantee that instance methods are thread safe** because adding all the locking code would hurt performance too much. 

As mentioned earlier, when a thread constructs an object, only this thread has a reference to the object, no other thread can access that object, and no thread synchronization is required. \
However, if the thread exposes this reference, by placing in **static** field, passing into **ThreadPool.QueueUserWorkItem** or **Task**, then thread synchronization is required if the threads could perform non-read-only access.

## Primitive User-Mode and Kernel-Mode Constructs

In this section, I explain the **primitive** thread synchronization constructs. By **primitive**, I mean the simplest constructs that are available to use in your code.

There are two kinds of primitive constructs: **user-mode** and **kernel-mode**.
+ primitive user-mode constructs: \
faster than the kernel-mode constructs because they use special CPU instructions to coordinate threads. \
blocked on a user-mode primitive construct is never considered blocked, thread pool will not create a new thread to replace the temporarily blocked thread. \
If the thread holding the context never release it, the thread will always occupy the CPU. We call this a **livelock**.
+ primitive kernel-mode constructs: \
a thread uses a kernel-mode construct to acquire a resource that another thread has, Windows blocks the thread so that it is no longer wasting CPU time. \
If the thread holding the context never release it, the thread is blocked forever, and we call this a **deadlock**.

We would like to have constructs that take the best of both. That is, we'd like construct that is fast and non-blocking when there is no contention (like user-mode constructs). But when there is contention for the construct, we’d like it to be blocked by the operating system kernel. This kind of construct do exist, and are called **hybrid constructs**.

## User-Mode Constructs

The CLR guarantees that read and write the following data types are **atomic**: **Boolean, Char, (S)Byte, (U)Int16, (U)Int32, (U)IntPtr, Single,** and **reference types**. This means that all bytes within that variable are read from or written to all at once. For example,

```c#
internal static class SomeType {
    public static Int32 x = 0;
}

// A thread executes this line to assign x variable.
// Another thread cannot possibly see the value in an intermediate state, e.g., 0x01230000.
SomeType.x = 0x01234567;
```

Suppose that the **x** field is Int64 type. If a thread executes this line of code:
```
SomeType.x = 0x0123456789abcdef;
```
It's possible that another thread could query **x** and get a value of **0x0123456700000000** or **0x0000000089abcdef**, because the read and write operations are not atomic.

Although atomic access to variable guarantees that the read or write happens all at once, it does not guarantee **when** the read or write will happen due to compiler and CPU optimizations. \
The primitive user-mode constructs discussed in this section are used to enforce the timing of these atomic read and write operations. In addition, these constructs can also force atomic and timed access to **(U)Int64 and Double**.

There are two kinds of primitive user-mode thread synchronization constructs:
+ Volatile constructs
+ Interlocked constructs

### Volatile Constructs

C# compiler translates your C# constructs into Intermediate Language (IL), which is then converted by the just-in-time (JIT) compiler into native CPU instructions. During this process, the C# compiler, JIT compiler, and even CPU can optimize your code. For example, the following useless method can be compiled into nothing.
```c#
private static void OptimizedAway() {
    // Constant expression is computed at compile time resulting in zero
    Int32 value = (1 * 100) -­ (50 * 2);
    
    // If value is 0, the loop never executes
    for (Int32 x = 0; x < value; x++) {
        // There is no need to compile the code in the loop because it can never execute
       Console.WriteLine("Jeff");
    }
}
```
In this code, the compiler can see that value will always be 0, the loop will never execute. So this method could be compiled to nothing. 

However, sometimes in multiple threads, the optimization might break our intention. For example, in the following example, there are two threads that are accessing two fields.

```c#
internal sealed class ThreadsSharingData {
    private Int32 m_flag = 0;
    private Int32 m_value = 0;
   
    // This method is executed by one thread
    public void Thread1() {
        // Note: These could execute in reverse order
        m_value = 5;
        m_flag  = 1;
    }
   
    // This method is executed by another thread
    public void Thread2() {
        // Note: m_value could be read before m_flag
        if (m_flag == 1)
            Console.WriteLine(m_value);
    }
}
```
The problem with this code is that the compilers/CPU could reverse the two lines of code in **Thread1** method when compiling the code. Because reversing the two lines of code does not change the intention of the method.

Case 1: these two lines of code are reversed, then **Thread2** method will see that **m_flag** is 1 and then display **m_value** is 0.

Case 2: these two lines of code executes in order. When **Thread2** method executes, it may first load **m_value** (0) into CPU. Then **Thread1** method executes. But **Thread2’s** CPU register doesn’t see that **m_value** has been changed to 5.

Now, let’s talk about how to correct your code. The static **System.Threading.Volatile** class offers two static methods that look like this.
```c#
public static class Volatile {
    public static void  Write(ref Int32 location, Int32 value);
    public static Int32 Read(ref Int32 location);
}
```

In fact, these methods disable some optimizations,
+ **Volatile.Write** forces the value be written to at the point of call. In addition, all earlier load and store operation must occur before this call.
+ **Volatile.Read** method forces the value be read from at the point of the call. In addition, any later program-order loads and stores must occur after the call.

Summarize as: when threads are communicating with each other via shared memory, **write the last value by calling Volatile.Write and read the first value by calling Volatile.Read.**

So now we can fix the class by using these methods.
```c#
internal sealed class ThreadsSharingData {
    private Int32 m_flag = 0;
    private Int32 m_value = 0;

    // This method is executed by one thread
    public void Thread1() {
        // Note: 5 must be written to m_value before 1 is written to m_flag
        m_value = 5;
        Volatile.Write(ref m_flag, 1);
    }

    // This method is executed by another thread
    public void Thread2() {
        // Note: m_value must be read after m_flag is read
        if (Volatile.Read(ref m_flag) == 1)
            Console.WriteLine(m_value);
    } 
}
```

For the **Thread1** method, the **Volatile.Write** call ensures that all the writes above it are completed before a 1 is written to **m_flag**. \
For the **Thread2** method, the **Volatile.Read** call ensures that all variable reads after it start after the value in **m_flag** has been read.

### C#’s Support for Volatile

C# compiler has the **volatile** keyword to simplify use. JIT compiler ensures that all access to a volatile field are performed as volatile read and write. Furthermore, the **volatile** keyword tells the C# and JIT compilers not to cache the field in CPU, ensuring that all reads to and from the field actually cause the value to be read from memory.

Using the volatile keyword, we can rewrite the **ThreadsSharingData** class

```c#
internal sealed class ThreadsSharingData {
    private volatile Int32 m_flag = 0;
    private          Int32 m_value = 0;

    // This method is executed by one thread
    public void Thread1() {
        // Note: 5 must be written to m_value before 1 is written to m_flag
        m_value = 5;
        m_flag = 1;
    }

    // This method is executed by another thread
    public void Thread2() {
        // Note: m_value must be read after m_flag is read
        if (m_flag == 1)
            Console.WriteLine(m_value);
    }
}
```

Do NOT recommend to use C#'s **volatile** keyword, because most algorithms require few volatile read or write accesses to a field and we can access normally a field in most cases. In addition, C# does not support passing a volatile field by reference to a method. For example, if **m_amount** is defined as a **volatile** Int32, following call will generate a warning.

```c#
Boolean success = Int32.TryParse("123", out m_amount);
// The preceding line causes the C# compiler to generate a warning:
// CS0420: a reference to a volatile field will not be treated as volatile
```

### Interlocked Constructs

**Volatile.Read** method performs an atomic read operation, and its **Write** method performs an atomic write operation. That is, each method performs either an atomic read operation **or** an atomic write operation. 

System.Threading.Interlocked class’s each method performs an atomic read **and** write operation.

The most commonly used methods are those static methods that operate on Int32 variables.
```c#
public static class Interlocked {
    // return (++location)
    public static Int32 Increment(ref Int32 location);

    // return (--­­location)
    public static Int32 Decrement(ref Int32 location);

    // return (location += value)
    // Note: value can be a negative number allowing subtraction
    public static Int32 Add(ref Int32 location, Int32 value);

    // Int32 old = location; location = value; return old;
    public static Int32 Exchange(ref Int32 location, Int32 value);

    // Int32 old = location;
    // if (location == comparand) location = value;
    // return old;
    public static Int32 CompareExchange(ref Int32 location, Int32 value, Int32   comparand);
    ...
}
```

Let me show you some code that uses the **Interlocked** methods to asynchronously query several web servers and concurrently process the returned data. It is a great model to fol- low for your own scenarios.

```c#
internal sealed class MultiWebRequests {
    // This helper class coordinates all the asynchronous operations
    private AsyncCoordinator m_ac = new AsyncCoordinator();

    // Set of web servers we want to query & their responses (Exception or Int32)
    // NOTE: Even though multiple could access this dictionary simultaneously,
    // there is no need to synchronize access to it because the keys are read­only after construction
    private Dictionary<String, Object> m_servers = new Dictionary<String, Object> {
        { "http://Wintellect.com/", null },
        { "http://Microsoft.com/",  null },
        { "http://1.1.1.1/",        null }
    };

    public MultiWebRequests(Int32 timeout = Timeout.Infinite) {
        // Asynchronously initiate all the requests all at once
        var httpClient = new HttpClient();
        foreach (var server in m_servers.Keys) {
            m_ac.AboutToBegin(1);
            httpClient.GetByteArrayAsync(server)
                .ContinueWith(task => ComputeResult(server, task));
        }
      
        // Tell AsyncCoordinator that all operations have been initiated and to call
        // AllDone when all operations complete, Cancel is called, or the timeout occurs
        m_ac.AllBegun(AllDone, timeout);
    }
   
    private void ComputeResult(String server, Task<Byte[]> task) {
        Object result;
        if (task.Exception != null) {
            result = task.Exception.InnerException;
        } else {
            // Process I/O completion here on thread pool thread(s)
            // Put your own compute­intensive algorithm here...
            result = task.Result.Length;  // This example just returns the length
        }
        
        // Save result (exception/sum) and indicate that 1 operation completed
        m_servers[server] = result;
        m_ac.JustEnded();
    }

    // Calling this method indicates that the results don't matter anymore
    public void Cancel() { m_ac.Cancel(); }
  
    // This method is called after all web servers respond, Cancel is called, or the timeout occurs
    private void AllDone(CoordinationStatus status) {
        switch (status) {
            case CoordinationStatus.Cancel:
                Console.WriteLine("Operation canceled.");
                break;
            case CoordinationStatus.Timeout:
                Console.WriteLine("Operation timed­out.");
                break;
            case CoordinationStatus.AllDone:
                Console.WriteLine("Operation completed; results below:");
                foreach (var server in m_servers) {
                    Console.Write("{0} ", server.Key);
                    Object result = server.Value;
                    if (result is Exception) {
                        Console.WriteLine("failed due to {0}.", result.GetType().Name);
                    } else {
                        Console.WriteLine("returned {0:N0} bytes.", result);
                    }
                }   
                break; 
        }
    } 
}
```

Let me first explain what this class is doing. 
1. When the **MultiWebRequests** is constructed, it initializes an **AsyncCoordinator** and a dictionary of server (and future result). 
2. Then it issues all the web requests asynchronously one by one. First call **AsyncCoordinator’s AboutToBegin** method, passing it the number of requests about to be issued.
3. Then initiates the request by calling **HttpClient’s GetByteArrayAsync**. This returns a Task and I then call **ContinueWith**, so that when the server replies, the result can be processed by **ComputeResult** method.
4. After all the web requests have been made, the **AsyncCoordinator’s AllBegun** method is called, passing **AllDone** method that should execute when all the operations complete and a timeout value.

Note that the **AllDone** method can only be called once in following cases: all requests completed, timeout, Cancel is called. So it is identified by **CoordinationStatus** variable.

Then, let’s take a look at how it works. The **Async­Coordinator** class uses **Interlocked** methods to encapsulate all the thread coordination logic.

```c#
internal sealed class AsyncCoordinator {
    private Int32 m_opCount = 1;        // Decremented when AllBegun calls JustEnded

    private Int32 m_statusReported = 0; // 0=false, 1=true

    private Action<CoordinationStatus> m_callback;

    private Timer m_timer;

    // This method MUST be called BEFORE initiating an operation
    public void AboutToBegin(Int32 opsToAdd = 1) {
       Interlocked.Add(ref m_opCount, opsToAdd);
    }

    // This method MUST be called AFTER an operation’s result has been processed
    public void JustEnded() {
       if (Interlocked.Decrement(ref m_opCount) == 0)
          ReportStatus(CoordinationStatus.AllDone);
    }

    // This method MUST be called AFTER initiating ALL operations
    public void AllBegun(Action<CoordinationStatus> callback,
        Int32 timeout = Timeout.Infinite) {
            m_callback = callback;
        if (timeout != Timeout.Infinite)
            m_timer = new Timer(TimeExpired, null, timeout, Timeout.Infinite);
        JustEnded();
    }

    private void TimeExpired(Object o) { ReportStatus(CoordinationStatus.Timeout); }
    
    public void Cancel()               { ReportStatus(CoordinationStatus.Cancel); }
    
    private void ReportStatus(CoordinationStatus status) {
        // If status has never been reported, report it; else ignore it
        if (Interlocked.Exchange(ref m_statusReported, 1) == 0)
            m_callback(status);
    }
}
```

+ The most important field in this class is the **m_opCount** field, which tracks the number of active asynchronous operations.
+ Before each asynchronous operation is started, **AboutToBegin** is called. This method calls **Interlocked.Add** to add the number atomically.
+ As web server responses are processed, **JustEnded** is called. This method calls **Inter­locked.Decrement** to atomically subtract 1 from **m_opCount**.
+ Whichever thread happens to set **m_opCount** to 0 calls **ReportStatus**. Note that **m_opCount** field is initialized to 1 (not 0), because it ensures that **AllDone** is not invoked when the constructor calls **AllBegun**.
+ ReportStatus must make sure that only one **CoordinationStatus** is passed. It use **Interlocked.Exchange** method to determine the first thread to execute **m_callback** mwthod, and other threads will see the **m_statusReported** variable is 1, and should not invoke again.

### Implementing a Simple Spin Lock

**Interlocked** method is great, but it mostly operates on **Int32** values. If you need to operate a bunch of fields atomically, you need a way to stop all other threads and allow only one thread to execute.

Using **Interlocked** methods, we can build a thread synchronization lock.

```c#
internal struct SimpleSpinLock {
    private Int32 m_ResourceInUse; // 0=false (default), 1=true
   
    public void Enter() {
        while (true) {
            // Always set resource to in­use
            // When this thread changes it from not in­use, return
            if (Interlocked.Exchange(ref m_ResourceInUse, 1) == 0) return;
        } 
    }
   
    public void Leave() {
        // Set resource to not in­use
        Volatile.Write(ref m_ResourceInUse, 0);
    } 
}
```
Here is a class that shows how to use the **SimpleSpinLock**. 
```c#
public sealed class SomeResource {
    private SimpleSpinLock m_sl = new SimpleSpinLock();
   
    public void AccessResource() {
        m_sl.Enter();
        // Only one thread at a time can get in here to access the resource...
        m_sl.Leave();
    } 
}
```
The **SimpleSpinLock** is very simple. If two threads call **Enter** at the same time, **Interlocked.Exchange** ensures that one thread changes **m_resourceInUse** from 0 to 1 and sees that **m_resourceInUse** was 0. This thread then returns from **Enter** so that it can continue executing **AccessResource** method. The other thread will change **m_resourceInUse** from a 1 to a 1. This thread will see that it did not change **m_resourceInUse** from a 0, and this thread will now start spinning continuously, calling **Exchange** until the first thread calls **Leave**.

When the first thread is done, it calls **Leave**, which internally calls **Volatile.Write** and changes **m_resourceInUse** back to a 0. Then the spinning thread can change **m_resourceInUse** from a 0 to a 1, and get chance to return from **Enter**.

### Interlocked Anything

Many people wonder why Microsoft doesn't offer a richer set of interlocked methods that can be used in wider range of scenarios. For example, **Interlocked** class offers Multiply, Divide, Minimum, Maximum and a bunch of other methods. 

Although the **Interlocked** class doesn’t offer these methods, there is a well-known pattern that allows you to perform any operation in an atomic way by using **Interlocked.CompareExchange**.

This pattern is similar to optimistic concurrency patterns used for modifying database records. Here is an example to create atomic **Maximum** method.

```c#
public static Int32 Maximum(ref Int32 target, Int32 value) {
    Int32 currentVal = target, startVal, desiredVal;
    
    do {
        // Record this iteration's starting value
        startVal = currentVal;
        
        // Calculate the desired value in terms of startVal and value
        desiredVal = Math.Max(startVal, value);
        
        // NOTE: the thread could be preempted here!
        
        // if (target == startVal) target = desiredVal
        // Value prior to potential change is returned
        currentVal = Interlocked.CompareExchange(ref target, desiredVal, startVal);
        
        // If the starting value changed during this iteration, repeat
   } while (startVal != currentVal);
   
   // Return the maximum value when this thread tried to set it
   return desiredVal;
}
```

When entering the method, **currentVal** is initialized to the **target**. Then, in th loop, the **startVal** is initialized to the same value. Using **startVal**, you can perform any operation you want. But, ultimately, you must end up with a result that is placed into **desiredVal**. In this example, I simply determine whether **startVal** or **value** contains the larger value.

Now, while this operation is running, another thread could change the **target**. If this does happen, then the value in **desiredVal** is based off an old value in **startVal**, not the current value in **target**, and therefore, we should not change the **target**. To ensure that the target is changed to desiredVal if no thread changed it, we use **Interlocked.CompareExchange**. \
It checks whether **target** matches the **startVal**. If they are same, then **CompareExchange** changed the **target** to the **desiredVal**. If the **target** did change, then **CompareExchange** does not alter the **target**.

**CompareExchange** returns the value in target when **CompareExchange** is called. I placed it into **currentVal**. Then, a check is made comparing **startVal** with **currentVal**. If these values are the same, then a thread did not change **target**, **target** now contains the value in **desiredVal**, the while loop does not loop and the method returns. If **startVal** is not equal to **currentVal**, then a thread did change the value in **target** behind our thread’s back, **target** did not get changed to our value in **desiredVal**, and the while loop will loop around, this time using the new value in **currentVal** that reflects the other thread's value.

Also, we made a generic method, Morph, which encapsulates this pattern.

```c#
delegate Int32 Morpher<TResult, TArgument>(Int32 startValue, TArgument argument, out TResult morphResult);

static TResult Morph<TResult, TArgument>(ref Int32 target, TArgument argument, Morpher<TResult, TArgument> morpher) {
    TResult morphResult;
    Int32 currentVal = target, startVal, desiredVal;

    do {
        startVal = currentVal;

        // delegate
        desiredVal = morpher(startVal, argument, out morphResult);

        currentVal = Interlocked.CompareExchange(ref target, desiredVal, startVal);
    } while (startVal != currentVal);
    return morphResult;
}

```

## Kernel-Mode Constructs

Windows offers several kernel-mode constructs for synchronizing threads. The kernel-mode constructs are much slower than the user-mode constructs, Because they require coordination from the Windows operating system, each method call will cause the calling thread to transition from managed code to native user-mode code to native kernel-mode code, and return back.

However, the kernel-mode constructs offers some benefits over the user-mode constructs,
+ When a kernel-mode construct detects contention on a resource, Windows blocks the losing thread so that it is not spinning on a CPU, wasting processor resources.
+ Kernel-mode constructs can synchronize **native** and **managed** threads with each other.
+ Kernel-mode constructs can synchronize threads running in different processes on the same
machine.

The two primitive kernel-mode constructs are **Events** and **Semaphores**. Other kernel-mode constructs, such as **Mutex**, are built on top of the two primitive constructs. \
The **System.Threading** namespace offers an abstract base class called **WaitHandle**, which is a simple class that wrap a Windows kernel object handle. The FCL provides several classes derived from **WaitHandle**, the class hierarchy looks like this.

```
WaitHandle
  - EventWaitHandle
      - AutoResetEvent
      - ManualResetEvent
  - Semaphore
  - Mutex
```

A common usage of the kernel-mode constructs is to create single-instance application. Here is how to implement a single-instance application.
```c#
using System;
using System.Threading;

public static class Program {
    public static void Main() {
        Boolean createdNew;
        
        // Try to create a kernel object with the specified name
        using (new Semaphore(0, 1, "SomeUniqueStringIdentifyingMyApp", out createdNew)) {
            if (createdNew) {
                // This thread created the kernel object so no other instance of this
                // application must be running. Run the rest of the application here...
            } else {
                // This thread opened an existing kernel object with the same string name;
                // another instance of this application must be running now.
                // There is nothing to do in here, let's just return from Main to terminate
                // this second instance of the application.
            } 
        }
    } 
}
```
In this code, I am using a **Semaphore**, but it would work just as well for **EventWaitHanlde** or **Mutex**. Let’s say that two instances of this process are started at exactly the same time, and both threads will attempt to create a **Semaphore** with the same string name (SomeUniqueStringIdentifyingMyApp, in my example). The Windows kernel ensures that only one thread actually creates a kernel object with the specified name; the thread that created the object will set **createdNew** variable to true.

For the second thread, Windows will see that a kernel object with the specified name already exists; the second thread does not get to create another kernel object with the same name. In this example, it will hit the **else** branch.

### Event Constructs

In fact, **Events** are simply **Boolean** variables maintained by the kernel. **When the event is false, the thread waiting on an event blocks. When the event is true, the thread unblocks.** \
There are two kinds of events.
+ When an **auto-reset event** is true, it wakes up just one blocked thread, because the kernel automatically resets the event back to false after unblocking the first thread.
+ When a **manual-reset event** is true, it unblocks all threads waiting for it because the kernel does not automatically reset the event back to false; your code must manually reset the event back to false.

```c#
public class EventWaitHandle : WaitHandle {
    public Boolean Set();    // Sets Boolean to true; always returns true
    public Boolean Reset();  // Sets Boolean to false; always returns true
}
public sealed class AutoResetEvent : EventWaitHandle {
    public AutoResetEvent(Boolean initialState);
}
public sealed class ManualResetEvent : EventWaitHandle {
    public ManualResetEvent(Boolean initialState);
}
```

Using an **auto-reset event**, we can easily create a thread synchronization lock whose behavior is similar to the **SimpleSpinLock** class I showed earlier.

```c#
internal sealed class SimpleWaitLock : IDisposable {
   private readonly AutoResetEvent m_available;
   public SimpleWaitLock() {
      m_available = new AutoResetEvent(true); // Initially free
}
   public void Enter() {
      // Block in kernel until resource available
      m_available.WaitOne();
}
   public void Leave() {
      // Let another thread access the resource
      m_available.Set();
}
   public void Dispose() { m_available.Dispose(); }
}
```

In fact, the external behavior is exactly the same; however, the performance of the two locks is different. When there is no contention on the lock, the **SimpleWaitLock** is much slower than the **SimpleSpinLock**, because every call to **SimpleWaitLock’s** **Enter** and **Leave** methods forces the calling thread to transition from managed code to the kernel and back. But when there is contention, the losing thread is blocked by the kernel and is not spinning and wasting CPU.

### Semaphore Constructs

Semaphores are simply Int32 variables maintained by the kernel. **When the semaphore is 0, A thread waiting on semaphore blocks. When the semaphore is greater than 0, the thread unblocks.** When a thread waiting on a semaphore unblocks, the kernel automatically subtracts 1 from the semaphore’s count.

```c#
public sealed class Semaphore : WaitHandle {
    public Semaphore(Int32 initialCount, Int32 maximumCount);
    public Int32 Release();   // Calls Release(1); returns previous count
    public Int32 Release(Int32 releaseCount);   // Returns previous count
}
```

Now let me summarize how these three kernel-mode primitives behave:
+ When multiple threads are waiting on an auto-reset event, setting the event causes only one thread to become unblocked.
+ When multiple threads are waiting on a manual-reset event, setting the event causes all threads to become unblocked.
+ When multiple threads are waiting on a semaphore, releasing the semaphore causes **releaseCount** threads to become unblocked.

Therefore, an auto-reset event behaves very similarly to a semaphore whose maximum count
is 1. 

Using a semaphore, we can re-implement the **SimpleWaitLock** as follows. It gives multiple threads concurrent access to a resource.

```c#
public sealed class SimpleWaitLock : IDisposable {
    private readonly Semaphore m_available;

    public SimpleWaitLock(Int32 maxConcurrent) {
        m_available = new Semaphore(maxConcurrent, maxConcurrent);
    }
    public void Enter() {
        // Block in kernel until resource available
        m_available.WaitOne();
    }
   
    public void Leave() {
        // Let another thread access the resource
        m_available.Release(1);
    }
    
    public void Dispose() { m_available.Close(); }
}
```

### Mutex Constructs

A **Mutex** represents a mutual-exclusive lock. It works similar to an AutoResetEvent or a Semaphore with a count of 1. Because all three constructs release only one waiting thread at a time.

```c#
public sealed class Mutex : WaitHandle {
    public Mutex();
    public void ReleaseMutex();
}
```

Mutexes have some additional logic in them, which makes them more complex than the other constructs.
+ **Mutex** objects record which thread obtained it by querying the calling thread’s ID. When a thread calls **ReleaseMutex**, the **Mutex** makes sure that the calling thread is the same thread that obtained the **Mutex**.
+ **Mutex** objects maintain a recursion count indicating how many times the owning thread owns the **Mutex**. If a thread currently owns a **Mutex** and then waits on the **Mutex** again, the recursion count is incremented and the thread is allowed to continue running. When that thread calls **ReleaseMutex**, the recursion count is decremented. Only when the recursion count becomes 0 can another thread become the owner of the **Mutex**.

These additional logic have a cost. The **Mutex** object needs more memory to hold the additional thread ID and recursion count information. If an application needs or wants these additional features, then the application code could have done this itself, without using **Mutex** object.

Usually a recursive lock is needed when a method takes a lock and then calls another method that also requires the lock, as the following code demonstrates.
```c#
internal class SomeClass : IDisposable {
    private readonly Mutex m_lock = new Mutex();
    
    public void Method1() {
        m_lock.WaitOne();
        // Do whatever...
        Method2();  // Method2 recursively acquires the lock
        m_lock.ReleaseMutex();
    }
   
    public void Method2() {
        m_lock.WaitOne();
        // Do whatever...
        m_lock.ReleaseMutex();
    }
   
    public void Dispose() { m_lock.Dispose(); }
}
```

In above code, a **SomeClass** object could call **Method1**, which acquires
the **Mutex**, performs some thread-safe operation, and then calls **Method2**, which also performs some thread-safe operation. Because **Mutex** objects support recursion, the thread will acquire the lock twice and then release it twice before another thread can own the **Mutex**. If **SomeClass** has used an **AutoResetEvent** instead of a **Mutex**, then the thread would block when it called **Method2’s** **WaitOne** method.

If you need a recursive lock, then you could create one easily by using an **AutoResetEvent**.
```c#
internal sealed class RecursiveAutoResetEvent : IDisposable {
    private AutoResetEvent m_lock = new AutoResetEvent(true);
    private Int32 m_owningThreadId = 0;
    private Int32 m_recursionCount = 0;
    
    public void Enter() {
        // Obtain the calling thread's unique Int32 ID
        Int32 currentThreadId = Thread.CurrentThread.ManagedThreadId;
        
        // If the calling thread owns the lock, increment the recursion count
        if (m_owningThreadId == currentThreadId) {
            m_recursionCount++;
            return; 
        }
        
        // The calling thread doesn't own the lock, wait for it
        m_lock.WaitOne();
        // The calling now owns the lock, initialize the owning thread ID & recursion count
        m_owningThreadId = currentThreadId;
        m_recursionCount = 1;
    }
   
    public void Leave() {
        // If the calling thread doesn't own the lock, we have an error
        if (m_owningThreadId != Thread.CurrentThread.ManagedThreadId)
            throw new InvalidOperationException();
      
        // Subtract 1 from the recursion count
        if (­­m_recursionCount == 0) {
            // If the recursion count is 0, then no thread owns the lock
            m_owningThreadId = 0;
            m_lock.Set();   // Wake up 1 waiting thread (if any)
        } 
    }
    
    public void Dispose() { m_lock.Dispose(); }
}
```

Although the behavior of the **RecursiveAutoResetEvent** class is identical to that of the **Mutex** class, a **RecursiveAutoResetEvent** object will have far superior performance when a thread tries to acquire the lock recursively, because all the code is now in managed code. A thread has to transition into the Windows kernel only when first acquiring the **AutoResetEvent** or when finally releaseing it to another thread.