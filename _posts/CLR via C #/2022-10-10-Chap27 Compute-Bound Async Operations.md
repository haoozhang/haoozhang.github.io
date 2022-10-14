---
layout:     post
title:      Chapter 27. Compute-Bound Asynchoronous Operations
subtitle:   
date:       2022-10-10
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

In this chapter, I’ll talk about the various ways that you can perform operations asynchronously. When performing an asynchronous compute-bound operation, you execute it using other threads. Here are some examples of compute-bound operations: compiling code, spell checking, grammar checking.

## Introducing the CLR’s Thread Pool

Creating and destroying a thread is an expensive operation. Having lots of threads wastes memory resources and also hurts performance due to the operating system having to schedule and context switch. \
To improve this situation, CLR manages its thread pool, which can be treat a set of threads that available for application to use. There is one thread pool per CLR.

When the CLR initializes, the thread pool has no threads. Internally, the thread pool maintains a queue of operation requests. \
When you want to perform an async operation, you call some method to append an entry into the thread pool’s queue. The thread pool will dispatchthe entry to a thread. \
If there is no thread in thread pool, a new thread will be created. \
When the thread complete the task, it's not destoryed. Instead, it's returned to thread pool, and wait to respond to another request.

If your application makes multiple requests for the thread pool, the thread pool will try to service all requests by using just this one thread. \
However, if the speed of producing requests is faster than the processing speed of thread, additional threads will be created to serve. So, these requests can be handled by a small number of threads.

When the thread has nothing to do for some period of time, it will wake up and kill itself, to save memory resources.

So, thread pool can contain a small number of threads to avoid wasting memory resource. Also, it can contain more threads, to take advantage of multiprocessors, hyperthreaded processors, and multi-core processors. **The thread pool is heuristic**.

## Performing a Simple Compute-Bound Operation

To enqueue an asyns compute-bound operation to the thread pool, you typically call one of the following methods defined in **ThreadPool** class.

```c#
static Boolean QueueUserWorkItem(WaitCallback callBack);
static Boolean QueueUserWorkItem(WaitCallback callBack, Object state);
```

These methods enqueue a "work item" to the thread pool's queue, and then return immediately. A work item is the method identified by the **callback** parameter. The method can be passed a single parameter specified via the **state** argument. Eventually, some thread in the pool will process the work item. The callback method you write must match the **WaitCallback** delegate type, which is defined as follows.

```c#
delegate void WaitCallback(Object state);
```

The following code demonstrates how to have a thread pool thread call a method asynchronously.

```c#
using System;
using System.Threading;

public static class Program {

    public static void Main() {
        Console.WriteLine("Main thread: queuing an asynchronous operation");
        ThreadPool.QueueUserWorkItem(ComputeBoundOp, 5);
        Console.WriteLine("Main thread: Doing other work here...");
        Thread.Sleep(10000);  // Simulating other work (10 seconds)
        Console.WriteLine("Hit <Enter> to end this program...");
        Console.ReadLine();
    }
         
    // This method's signature must match the WaitCallback delegate
    private static void ComputeBoundOp(Object state) {
        // This method is executed by a thread pool thread
        Console.WriteLine("In ComputeBoundOp: state={0}", state);
        Thread.Sleep(1000);  // Simulates other work (1 second)
        // When this method returns, the thread goes back
        // to the pool and waits for another task
    }
}
```

## Execution Contexts

Every thread has an execution context data structure associated with it. When a thread executes code, some operations are affected by the execution context settings. Ideally, whenever a thread uses another (helper) thread to perform tasks, this thread's execution context should flow to to the helper thread. This ensures that any operations performed by helper thread(s) are executing with the same context settings.

By default, CLR allows to flow the initializing thread's context to next helper threads, but it may cause performance drop. This is because that the execution context contains a lot of information, the process of creating execution context and copying these information into helper thread wastes some time.

There is an **ExecutionContext** class that allows you to control how a thread’s execution context flows from one thread to another.
```c#
public sealed class ExecutionContext : IDisposable, ISerializable {
    [SecurityCritical] public static AsyncFlowControl SuppressFlow();
    public static void RestoreFlow();
    public static Boolean IsFlowSuppressed();

    // Less commonly used methods are not shown
}
```

You can use this class to suppress the flowing of an execution context, thereby improving your application’s performance. Here is an example showing how suppressing the flow of execution context affects data in a thread’s context when queuing a work item to the CLR’s thread pool.

```c#
public static void Main() {
    // Put some data into the Main thread's logical call context
    CallContext.LogicalSetData("Name", "Jeffrey");

    // Initiate some work to be done by a thread pool thread
    // The thread pool thread can access the logical call context data
    ThreadPool.QueueUserWorkItem(
       state => Console.WriteLine("Name={0}", CallContext.LogicalGetData("Name")));
    
    // Now, suppress the flowing of the Main thread's execution context
    ExecutionContext.SuppressFlow();

    // Initiate some work to be done by a thread pool thread
    // The thread pool thread CANNOT access the logical call context data
    ThreadPool.QueueUserWorkItem(
       state => Console.WriteLine("Name={0}", CallContext.LogicalGetData("Name")));
    
    // Restore the flowing of the Main thread's execution context in case
    // it employs more thread pool threads in the future
    ExecutionContext.RestoreFlow();
    
    Console.ReadLine();
}
```

When runnning the preceding code, I get the following output.
```c#
Name=Jeffrey
Name=
```

## Cooperative Cancellation

.NET framework offers a standard pattern for canceling operations. This pattern is **cooperative**, meaning that the canceling operation must both use the types mentioned in this section. Let me first explain the two main types provided in the FCL.

### CancellationTokenSource and CancellationToken

To cancel an operation, you must first create a **System.Threading.CancellationTokenSource** object. This class looks like this.

```c#
public sealed class CancellationTokenSource : IDisposable {  // A reference type
    public CancellationTokenSource();
    public Boolean IsCancellationRequested { get; }
    public CancellationToken Token { get; }
    public void Cancel();  // Internally, calls Cancel passing false
    public void Cancel(Boolean throwOnFirstException);
    ...
}
```

This object contains all the states related cancellation. After constructing a **CancellationTokenSource** (reference type) object, one or more **CancellationToken** (value type) can be obtained by querying its **Token** property, it can be passed around to your operations that to be canceled.

Here are the most useful members of the **CancellationToken** value type.

```c#
public struct CancellationToken {  // A value type
    public static CancellationToken None { get; }  // Very convenient

    public Boolean IsCancellationRequested { get; } // Called by non­Task invoked     operations
    public void ThrowIfCancellationRequested();  // Called by Task­invoked operations

    // WaitHandle is signaled when the CancellationTokenSource is canceled
    public WaitHandle WaitHandle { get; }
    // GetHashCode, Equals, operator== and operator!= members are not shown

    public Boolean CanBeCanceled { get; }  // Rarely used

    public CancellationTokenRegistration Register(Action<Object> callback, Object state,
       Boolean useSynchronizationContext);  // Simpler overloads not shown
}
```

A compute-bound operation’s loop can periodically call **CancellationToken’s IsCancellationRequested** property to know if the loop should terminate early, thereby ending the compute-bound operation.

Let me put all together with sample code.

```c#
internal static class CancellationDemo {
    public static void Main() {
        CancellationTokenSource cts = new CancellationTokenSource();

        // Pass the CancellationToken and the number­to­count­to into the operation
        ThreadPool.QueueUserWorkItem(o => Count(cts.Token, 1000));
        
        Console.WriteLine("Press <Enter> to cancel the operation.");
        Console.ReadLine();
        cts.Cancel();  // If Count returned already, Cancel has no effect on it
        // Cancel returns immediately, and the method continues running here...
        
        Console.ReadLine();
    }

    private static void Count(CancellationToken token, Int32 countTo) {
        for (Int32 count = 0; count <countTo; count++) {
            if (token.IsCancellationRequested) {
                Console.WriteLine("Count is cancelled");
                break; // Exit the loop to stop the operation
            }

            Console.WriteLine(count);
            Thread.Sleep(200);   // For demo, waste some time
        }
        Console.WriteLine("Count is done");
    }
}
```

If you want to execute an operation and prevent it from being canceled, you can pass the operation the **CancellationToken.None** property. This property returns a special **CancellationToken** instance that is not associated with any **CancellationTokenSource**, so no code can call **Cancel**.

### Register

You can call **CancellationToken's Register** method to register one or more methods, which will be invoked when a **CancellationToken** is canceled. For this method, you need to pass: 
+ an Action\<Object> delegate
+ a state value that will be passed to the callback via the delegate
+ a Boolean value indicating whether to invoke callback by using **SynchronizationContext**.

If you pass false for the **useSynchronizationContext** parameter, then the thread that calls **Cancel** will invoke all the registered methods sequentially. \
If you pass true for the **useSynchronizationContext** parameter, then the callbacks are sent to **SynchronizationContext** object. It will decide which thread call these callbacks.

**CancellationToken’s Register** method returns a **CancellationTokenRegistration**, which looks like this.
```c#
public struct CancellationTokenRegistration : IEquatable<CancellationTokenRegistration>, IDisposable {
    public void Dispose();
    // GetHashCode, Equals, operator== and operator!= members are not shown
}
```

You can call **Dispose** to remove a registered callback from the **CancellationTokenSource**, so that this callback will not be invoked when calling **CancellationTokenSource.Cancel** method.

Here is some code that demonstrates registering two callbacks with a single **CancellationTokenSource**.

```c#
var cts = new CancellationTokenSource();
cts.Token.Register(() => Console.WriteLine("Canceled 1"));
cts.Token.Register(() => Console.WriteLine("Canceled 2"));

// To test, let's just cancel it now and have the 2 callbacks execute
cts.Cancel();

// Output:
// Canceled 2
// Canceled 1
```

### Linking CancellationTokenSource

You can create a new **CancellationTokenSource** object by linking a bunch of other **CancellationTokenSource** objects. This new **CancellationTokenSource** object will be canceled when any of the linked **CancellationTokenSource** objects are canceled.

```c#
// Create a CancellationTokenSource
var cts1 = new CancellationTokenSource();
cts1.Token.Register(() => Console.WriteLine("cts1 canceled"));

// Create another CancellationTokenSource
var cts2 = new CancellationTokenSource();
cts2.Token.Register(() => Console.WriteLine("cts2 canceled"));

// Create a new CancellationTokenSource that is canceled when cts1 or ct2 is canceled
var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(cts1.Token, cts2.Token);
linkedCts.Token.Register(() => Console.WriteLine("linkedCts canceled"));

// Cancel one of the CancellationTokenSource objects (I chose cts2)
cts2.Cancel();
// Display which CancellationTokenSource objects are canceled
Console.WriteLine("cts1 canceled={0}, cts2 canceled={1}, linkedCts canceled={2}", cts1.IsCancellationRequested, cts2.IsCancellationRequested, linkedCts.IsCancellationRequested);

// Output: 
// linkedCts canceled
// cts2 canceled
// cts1 canceled=False, cts2 canceled=True, linkedCts canceled=True
```

### Cancel after a period of time

**CancellationTokenSource** gives you a way to have it cancel itself after a period of time. To take advantage of this, you can either construct a **CancellationTokenSource** object by using the constructors that accept a delay, or call **CancellationTokenSource’s CancelAfter** method.

```c#
public sealed class CancellationTokenSource : IDisposable {  // A reference type
    public CancellationTokenSource(Int32 millisecondsDelay);
    public CancellationTokenSource(TimeSpan delay);

    public void CancelAfter(Int32 millisecondsDelay);
    public void CancelAfter(TimeSpan delay);
    ...
}
```

## Tasks

Calling ThreadPool's QueueWorkItem method to initiate an asynchronous compute-bound operation is very simple. However, this method has many limitations. There is no built-in way to know when the operation has completed, and there is no way to get return value when the operation completes. To address this, Microsoft introduces **System.Threading.Tasks**.

Before, we call **ThreadPool’s QueueUserWorkItem** method, you can do the same via **Tasks**.

```c#
1) ThreadPool.QueueUserWorkItem(ComputeBoundOp, 5);
2) new Task(ComputeBoundOp, 5).Start(); // You can create a task, and start it later.
3) Task.Run( () => ComputeBoundOp(5) ); // Create a task and then start it immediately.
```

When creating a **Task**, you call a constructor, passing it an **Action** or **Action\<Object>** delegate that indicates the performed operation. If you pass a method that expects an **Object**, then you must also pass to **Task**’s constructor. \
When calling **Run**, you can pass it an **Action** or **Func\<TResult>** delegate indicating the operation you want performed. \
When calling a constructor or when calling **Run**, you can optionally pass a **CancellationToken**.

### Waiting for a Task to Complete and Getting Its Result

With tasks, we can wait for them to complete and then get their result. 

Let’s say that we have a **Sum** method that is computationally intensive if n is a large value.
```c#
private static Int32 Sum(Int32 n) {
    Int32 sum = 0;
    for (; n > 0; n­­)
        checked { sum += n; }   // if n is large, this will throw System.OverflowException
    return sum;
}
```

We can now construct a **Task\<TResult>** object, and pass the **TResult** argument as the async operation's return type. Now, after starting the task, we can wait for it and get its result.

```c#
// Create a Task (it does not start running now)
Task<Int32> t = new Task<Int32>(n => Sum((Int32)n), 1000000000);

// You can start the task sometime later
t.Start();

// Optionally, you can explicitly wait for the task to complete
t.Wait();

// You can get the result (the Result property internally calls Wait)
Console.WriteLine("The Sum is: " + t.Result); // An Int32 value
```

If the compute-bound task throws an unhandled exception, the exception will be swallowed, stored in a collection, and the thread pool thread is allowed to return to the thread pool. When the **Wait** method or the **Result** property is invoked, these members will throw a **System.Aggregate­ Exception** object.

The **AggregateException** type is used to encapsulate a collection of exception objects ( can happen if a parent task spawns multiple child tasks that throw exceptions). It contains an **Inner­Exceptions** property that returns a **ReadOnlyCollection\<Exception>** object.

**AggregateException** overrides **Exception’s GetBaseException** method, it returns the innermost **AggregateException** that is the root cause of the problem. \
**AggregateException** also offers a **Flatten** method that creates a new **AggregateException**, whose **InnerExceptions** property contains a list of exceptions produced by walking the original **AggregateException**’s inner exception hierarchy. 

In addition to waiting for a single task, the **Task** class also offers two static methods (**WaitAny, WaitAll**) that allow a thread to wait an array of **Task** objects. \
**WaitAny** method blocks the calling thread until any **Task** objects in the array have completed. This method returns an **Int32** index into the array indicating which **Task** object completed. \
**WaitAll** method blocks the calling thread until all **Task** objects in the array have completed. The **WaitAll** method returns **true** if all the Task objects complete and **false** if a timeout occurs; an **OperationCanceledException** is thrown if **WaitAll** is canceled via a **CancellationToken**.

## Canceling a Task