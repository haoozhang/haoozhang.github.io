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
    for (; n > 0; n--­­)
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
t.Wait();  // FYI: Overloads exist accepting timeout/CancellationToken

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

### Canceling a Task

Of course, you can use **CancellationTokenSource** to cancel a **Task**. \
We must revise our **Sum** method so that it accepts a **CancellationToken**.

```c#
private static Int32 Sum(CancellationToken ct, Int32 n) {
    Int32 sum = 0;
    for (; n > 0; n­--­) {
       // The following line throws OperationCanceledException when Cancel
       // is called on the CancellationTokenSource referred to by the token
       ct.ThrowIfCancellationRequested();

       checked { sum += n; }   // if n is large, this will throw System.OverflowException
    }
    return sum; 
}
```

In this code, the compute-bound operation's loop periodically checks if the operation has been canceled, by calling **CancellationToken’s ThrowIfCancellationRequested** method. **ThrowIfCancellationRequested** throws an **OperationCanceledException** if the **CancellationTokenSource** has been canceled.

Now, we will create the **CancellationTokenSource** and **Task** objects as follows.
```c#
CancellationTokenSource cts = new CancellationTokenSource();
Task<Int32> t = Task.Run(() => Sum(cts.Token, 1000000000), cts.Token);

// Sometime later, cancel the CancellationTokenSource to cancel the Task
cts.Cancel(); // This is an asynchronous request, the Task may have completed already

try {
    // If the task got canceled, Result will throw an AggregateException
    Console.WriteLine("The sum is: " + t.Result);   // An Int32 value
}
catch (AggregateException x) {
    // Consider OperationCanceledException objects as handled.
    // Any other exceptions cause a new AggregateException containing
    // only the unhandled exceptions to be thrown
    x.Handle(e => e is OperationCanceledException);

    // If all the exceptions were handled, the following executes
    Console.WriteLine("Sum was canceled");
}
```

### Starting a New Task Automatically When Another Task Completes

Calling **Wait** or querying a task’s **Result** property when the task has not yet finished running will most likely cause the thread pool to create a new thread, which increases resource usage and hurts performance. \
Fortunately, there is a better way to find out when a task has completed running. When a task completes, it can start another task.

```c#
// Create and start a Task, continue with another task
Task<Int32> t = Task.Run(() => Sum(CancellationToken.None, 10000));

// ContinueWith returns a Task but you usually don't care
Task cwt = t.ContinueWith(task => Console.WriteLine("The sum is: " + task.Result));
```

Now, when the task executing **Sum** completes, this task will start another task (also on some thread pool thread) that displays the result. 

Also, **Task** objects internally contain a collection of **ContinueWith** tasks. So you can actually specify multiple **ContinueWith** tasks for a single **Task** object. In addition, when calling **ContinueWith**, you can specify a bitwise OR **TaskContinuationOptions**.

+ TaskContinuationOptions.OnlyOnCanceled : new task executes only if the first task is canceled
+ Task­ContinuationOptions.OnlyOnFaulted : new task executes only if the first task throws an unhandled exception
+ Task­ContinuationOptions.OnlyOnRanToCompletion : new task executes only if the first task runs completion without being canceled or throwing an unhandled exception.

```c#
// Create and start a Task, continue with multiple other tasks
Task<Int32> t = Task.Run(() => Sum(10000));

t.ContinueWith(task => Console.WriteLine("The sum is: " + task.Result),
   TaskContinuationOptions.OnlyOnRanToCompletion);

t.ContinueWith(task => Console.WriteLine("Sum threw: " + task.Exception.InnerException),
   TaskContinuationOptions.OnlyOnFaulted);

t.ContinueWith(task => Console.WriteLine("Sum was canceled"),
   TaskContinuationOptions.OnlyOnCanceled);
```

### A Task May Start Child Tasks

Tasks support parent/child relationships, as demonstrated by the following code.

```c#
Task<Int32[]> parent = new Task<Int32[]>(() => {
    var results = new Int32[3];   // Create an array for the results

    // This tasks creates and starts 3 child tasks
    new Task(() => results[0] = Sum(10000), TaskCreationOptions.AttachedToParent).Start();
    new Task(() => results[1] = Sum(20000), TaskCreationOptions.AttachedToParent).Start();
    new Task(() => results[2] = Sum(30000), TaskCreationOptions.AttachedToParent).Start();
    
    // Returns a reference to the array (even though the elements may not be initialized yet)
    return results;
});

// When the parent and its children have run to completion, display the results
var cwt = parent.ContinueWith(
   parentTask => Array.ForEach(parentTask.Result, Console.WriteLine));

// Start the parent Task so it can start its children
parent.Start();
```

Here, the parent task creates and starts three **Task** objects. By default, **Task** objects created by another task are top-level tasks that have no relationship to the task that creates them. However, the **TaskCreationOptions.AttachedToParent** flag associates a **Task** with the **Task** that creates it, so that the creating task is not considered finished until all its children (and grandchildren) have finished running.

### Inside a Task

Each **Task** object has a set of fields: 
+ Int32 Id (readonly).
+ Int32 execution state of **Task** : 
+ a reference of parent task.
+ a reference to the **TaskScheduler**.
+ a reference to the callback method.
+ a reference to the object that is to be passed to the callback
+ a reference to an **ExecutionContext**.

So, although tasks provide a lot of features, there is some cost to tasks because memory must be allocated for all this state.

Each Task can be identified by a unique value. **Task.CurrentId** returns the id of currently running task.

You can query **Task’s** read-only **Status** property to get where it is in lifecycle. This property returns a **TaskStatus** value that is defined as follows.
```c#
 public enum TaskStatus {
   // These flags indicate the state of a Task during its lifetime:
   Created,             // Task created explicitly; you can manually Start() this task
   WaitingForActivation,// Task created implicitly; it starts automatically
   WaitingToRun,  // The task was scheduled but isn’t running yet
   Running,       // The task is actually running
   
   // The task is waiting for children to complete before it considers itself complete
   WaitingForChildrenToComplete,

   // A task's final state is one of these:
   RanToCompletion,
   Canceled,
   Faulted
}
```

When you first construct a **Task** object, its status is **Created**. \
When the **Task** is started, its status changes to **WaitingToRun**. \
When the **Task** is actually running on a thread, its status changes to **Running**. \
When the **Task** stops running and is waiting for any child tasks, the status changes to **WaitingForChildrenToComplete**. \
When a task is completely finished, it enters one of three final states: **RanToCompletion, Canceled, or Faulted**. \
When a **Task\<TResult>** runs to completion, you can query the task’s result via **Task\<TResult>’s Result** property. \

The easiest way to determine if a **Task** completed successfully is to use code like the following.
```c#
if (task.Status == TaskStatus.RanToCompletion) ...
```

If the Task is created by calling **ContinueWith, ContinueWhenAll, ContinueWhenAny,** or **FromAsync**, it is in the **WaitingForActivation** state, which means that the **Task’s** scheduling is controlled by the task infrastructure.

### Task Factory

Occasionally, you might want to create a bunch of Task objects that share the same configuration. The **System.Threading.Tasks** namespace defines a **TaskFactory** type as well as a **TaskFactory\<TResult>** type.

If you want to create a bunch of tasks that return **void**, you will construct a **TaskFactory**. \
If you want to create a bunch of tasks that have return type, you will construct a **TaskFactory\<TResult>**, where you pass the generic **TResult** argument. \
When you create task factory class, you should pass **CancellationToken, TaskScheduler, TaskCreationOptions,** and **TaskContinuationOptions** settings.

```c#
Task parent = new Task(() => {
    var cts = new CancellationTokenSource();
    var tf  = new TaskFactory<Int32>(
        cts.Token, 
        TaskCreationOptions.AttachedToParent,
        TaskContinuationOptions.ExecuteSynchronously, 
        TaskScheduler.Default
    );

    // This task creates and starts 3 child tasks
    var childTasks = new[] {
        tf.StartNew(() => Sum(cts.Token, 10000)),
        tf.StartNew(() => Sum(cts.Token, 20000)),
        tf.StartNew(() => Sum(cts.Token, Int32.MaxValue))  // Too big, throws OverflowException
    };

    // If any of the child tasks throw, cancel the rest of them
    for (Int32 task = 0; task < childTasks.Length; task++)
        childTasks[task].ContinueWith(t => cts.Cancel(), TaskContinuationOptions.OnlyOnFaulted);
    
    // When all children are done, get the maximum value returned from the
    // non­faulting/canceled tasks. Then pass the maximum value to another
    // task that displays the maximum result
    tf.ContinueWhenAll(
        childTasks,
        completedTasks => completedTasks.Where(t => t.Status == TaskStatus.RanToCompletion).Max(t => t.Result),
        CancellationToken.None  // Even if other tasks are canceled, still run this task
    ).ContinueWith(
        t =>Console.WriteLine("The maximum is: " + t.Result),
            TaskContinuationOptions.ExecuteSynchronously
    );
});

// When the children are done, show any unhandled exceptions too
parent.ContinueWith(p => {
    StringBuilder sb = new StringBuilder("The following exception(s) occurred:" + Environment.NewLine);
    foreach (var e in p.Exception.Flatten().InnerExceptions)
        sb.AppendLine("   "+ e.GetType().ToString());
    Console.WriteLine(sb.ToString());
}, TaskContinuationOptions.OnlyOnFaulted);

// Start the parent Task so it can start its children
parent.Start();
```

### Task Scheduler

**TaskScheduler** object is responsible for executing scheduled tasks and also exposes task information to the Visual Studio debugger. FCL ships with two TaskScheduler-derived types: the thread pool task scheduler and a synchronization context task scheduler. 

By default, all applications use the thread pool task scheduler. This task scheduler schedules tasks to the thread pool’s worker threads. You can get a reference to the default task scheduler by querying **TaskScheduler’s** static **Default** property.

The synchronization context task scheduler is typically used for GUI applications, such as Windows Forms and WPF. This task scheduler schedules all tasks onto the application’s GUI thread. It does not use the thread pool. You can get a reference to a synchronization context task scheduler by querying **TaskScheduler’s** static **FromCurrent­SynchronizationContext** method.

Here is a simple Windows Forms application that demonstrates the use of the synchronization context task scheduler.

```c#
internal sealed class MyForm : Form {

    private readonly TaskScheduler m_syncContextTaskScheduler;
    
    public MyForm() {
        // Get a reference to a synchronization context task scheduler
        m_syncContextTaskScheduler = TaskScheduler.FromCurrentSynchronizationContext();
        Text = "Synchronization Context Task Scheduler Demo";
        Visible = true; Width = 600; Height = 100;
    }
   
    private CancellationTokenSource m_cts;
   
    protected override void OnMouseClick(MouseEventArgs e) {
        if (m_cts != null) { // An operation is in flight, cancel it
            m_cts.Cancel();
            m_cts = null;
        } else { // An operation is not in flight, start it
            Text = "Operation running";
            
            m_cts = new CancellationTokenSource();
            
            // This task uses the default task scheduler and executes on a thread pool thread
            Task<Int32> t = Task.Run(() => Sum(m_cts.Token, 20000), m_cts.Token);
         
            // These tasks use the sync context task scheduler and execute on the GUI thread
            t.ContinueWith(task => Text = "Result: " + task.Result,
               CancellationToken.None, TaskContinuationOptions.OnlyOnRanToCompletion,
               m_syncContextTaskScheduler);
            t.ContinueWith(task => Text = "Operation canceled",
               CancellationToken.None, TaskContinuationOptions.OnlyOnCanceled,
               m_syncContextTaskScheduler);
            t.ContinueWith(task => Text = "Operation faulted",
               CancellationToken.None, TaskContinuationOptions.OnlyOnFaulted,
               m_syncContextTaskScheduler);
        }
        base.OnMouseClick(e);
    }
}
```

When you click in the client area of this form, a compute-bound task will start executing on a thread pool thread. This is good because the GUI thread is not blocked and can respond to othrer UI operations. But the code executed by the thread pool thread should not attempt to update UI components, else an **InvalidOperationException** will be thrown.

When the compute-bound task is done, one of the three continue-with tasks will execute. These tasks are all scheduled by the synchronization context task scheduler corresponding to the GUI thread.

Because the compute-bound work (Sum) is running on a thread pool thread, the user can interact with the UI to cancel the operation. In this demo, I allow the user to cancel the operation by clicking in the form’s client area.

## Parallel’s Static For, ForEach, and Invoke Methods

There are some common programming scenarios that benefit from the improved performance with tasks. To simplify programming, the static **System.Threading.Tasks.Parallel** class encapsulates these common scenarios while using **Task** internally.

For example, instead of processing all the items in a collection like this.

```c#
// One thread performs all this work sequentially
for (Int32 i = 0; i < 1000; i++) DoWork(i);
```

you can get multiple thread pool threads to perform this work by using the **Parallel’s For** method.

```c#
// The thread pool’s threads process the work in parallel
Parallel.For(0, 1000, i => DoWork(i));
```

Similarly, if you have a collection, instead of doing this:
```c#
// One thread performs all this work sequentially
foreach (var item in collection) DoWork(item);
```

you can do this.
```c#
// The thread pool's threads process the work in parallel
Parallel.ForEach(collection, item => DoWork(item));
```

If you can use either **For** or **ForEach**, then it is recommended that you use **For** because it executes faster.

If you have several methods that need to execute, you could execute them all sequentially, like this:
```c#
// One thread executes all the methods sequentially
Method1();
Method2();
Method3();
```

or you could execute them in parallel, like this.
```c#
// The thread pool’s threads execute the methods in parallel
Parallel.Invoke(
   () => Method1(),
   () => Method2(),
   () => Method3()
);
```

Note that if any operation throws an unhandled exception, the **Parallel** method you called will ultimately throw an **AggregateException**.

When calling the **Parallel** method, there is an assumption that it is OK for the work items to be performed concurrently. So, do not use the **Parallel** methods if the work must be processed in sequential order. 

In addition, there are overloads of the For and ForEach methods that let you pass three delegates:
+ The task local initialization delegate (localInit) is invoked once for each task. This delegate is invoked before the task is asked to process a work item.
+ The body delegate (body) is invoked once for each item being processed.
+ The task local finally delegate (localFinally) is invoked once for each task. This delegate is invoked after the task has processed all the work items.

Here is some sample code that demonstrates the use of the three delegates by calculating the bytes for all files contained within a directory.

```c#
private static Int64 DirectoryBytes(String path, String searchPattern, SearchOption searchOption) {
   var files = Directory.EnumerateFiles(path, searchPattern, searchOption);
   Int64 masterTotal = 0;
   
   ParallelLoopResult result = Parallel.ForEach<String, Int64>(
        files,
        // localInit: Invoked once per task at start
        () => { 
            // Initialize that this task has seen 0 bytes
            return 0;   // Set taskLocalTotal initial value to 0
        },
        // body: Invoked once per work item
        (file, loopState, index, taskLocalTotal) => { 
            // Get this file's size and add it to this task's running total
            Int64 fileLength = 0;
            FileStream fs = null;
            try {
                fs = File.OpenRead(file);
                fileLength = fs.Length;
            }
            catch (IOException) { /* Ignore any files we can't access */ }
            finally { if (fs != null) fs.Dispose(); }
            
            return taskLocalTotal + fileLength;
        },
        // localFinally: Invoked once per task at end
        taskLocalTotal => { 
            // Atomically add this task's total to the "master" total
            Interlocked.Add(ref masterTotal, taskLocalTotal);
        }
    );
   
    return masterTotal;
}
```

Each task maintains its own running total using **taskLocalTotal** variable. After each task completes its work, the master total is updated in a thread-safe way by calling the **Interlocked.Add** method.

## Parallel Language Integrated Query

Microsoft's LINQ feature offers a convenient syntax for performing queries over collections. You can easily filter, sort, and return a projected set of items. When you use LINQ, only one thread process all the items sequentially, we call this **sequential query**. You can potentially improve the performance by using **Parallel LINQ**, which internally uses tasks to spread the processing across multiple CPUs.

The static **System.Linq.ParallelEnumerable** class implements all of the **Parallel LINQ** functionality, and so you must import the **System.Linq** namespace.

Here is an example of a sequential query that has been converted to a parallel query. This query returns all the obsolete methods defined within an assembly.

```c#
 private static void ObsoleteMethods(Assembly assembly) {
    var query =
        from type in assembly.GetExportedTypes().AsParallel()

        from method in type.GetMethods(BindingFlags.Public | BindingFlags.Instance | BindingFlags.Static)
            
        let obsoleteAttrType = typeof(ObsoleteAttribute)
        
        where Attribute.IsDefined(method, obsoleteAttrType)
      
        orderby type.FullName
        
        let obsoleteAttrObj = (ObsoleteAttribute)Attribute.GetCustomAttribute(method, obsoleteAttrType)
      
        select String.Format("Type={0}\nMethod={1}\nMessage={2}\n",
            type.FullName, method.ToString(), obsoleteAttrObj.Message);

    // Display the results
    foreach (var result in query) Console.WriteLine(result);
}
```

Because Parallel LINQ processes items by using multiple threads, the items are processed con- currently and the results are returned in an unordered fashion. If you reserve the orfer of items, you can call **ParallelEnumerable’s As­Ordered** method. \
The following operators produce unordered operations: Distinct, Except, Intersect, Union, Join, GroupBy, GroupJoin, and ToLookup. If you want to enforce ordering after these operations, just call **AsOrdered** method.

The following operators produce ordered operations: OrderBy, OrderByDescending. If you want to go back to unordered processing, just call the **AsUnordered** method.

Parallel LINQ offers some additional ParallelEnumerable methods that you can control how the query is processed.
```c#
// allow to cancel processing
public static ParallelQuery<TSource> WithCancellation<TSource>(
   this ParallelQuery<TSource> source, CancellationToken cancellationToken)
// specify the maximum number of threads
public static ParallelQuery<TSource> WithDegreeOfParallelism<TSource>(
   this ParallelQuery<TSource> source, Int32 degreeOfParallelism)
// ParallelExecutionMode.Default, or ParallelExecutionMode.ForceParallelism
// Let PLINQ decide how to best process, or force parallel process
public static ParallelQuery<TSource> WithExecutionMode<TSource>(
   this ParallelQuery<TSource> source, ParallelExecutionMode executionMode)
// control how the items are buffered and merged
public static ParallelQuery<TSource> WithMergeOptions<TSource>(
   this ParallelQuery<TSource> source, ParallelMergeOptions mergeOptions)
```

## Perform a Periodic Compute-Bound Operation

The **System.Threading** namespace defines a **Timer** class, which you can use to have a thread pool thread call a method periodically. 

The **Timer** class offers several constructors.
```c#
public Timer(TimerCallback callback, Object state, Int32    dueTime, Int32 period);
public Timer(TimerCallback callback, Object state, UInt32   dueTime, UInt32 period);
public Timer(TimerCallback callback, Object state, Int64    dueTime, Int64 period);
public Timer(TimerCallback callback, Object state, Timespan dueTime, TimeSpan period);
```

+ The **callback** parameter identifies the method that you want called back.
+ The **state** parameter allows you to pass state data to the callback method each time it is invoked.
+ The **dueTime** parameter tells how many millseconds to wait before running for the first time.
+ The **period** parameter specifies how long milliseconds to wait before running later. If you pass **Timeout.Infinite (-­1)**, a thread pool thread will call the callback method just once.

Internally, the thread pool has just one thread used for all **Timer** objects. When the Timer object is due, the thread wakes up and calls **ThreadPool’s QueueUserWorkItem** to enter an entry into the thread pool’s queue. \
If your callback method takes a long time to execute, the timer could triggle again. This could cause multiple thread pool threads to be execut- ing your callback method. To work around this, I recommend: construct the **Timer** specifying **Timeout.Infinite** for the **period** parameter. Now, the timer will fire only once. Then, in your callback method, call the **Change** method specifying a new due time and again specify **Timeout.Infinite** for the **period** parameter.

```c#
internal static class TimerDemo {
    private static Timer s_timer;
   
    public static void Main() {
        Console.WriteLine("Checking status every 2 seconds");

        s_timer = new Timer(Status, null, Timeout.Infinite, Timeout.Infinite);
        
        s_timer.Change(0, Timeout.Infinite);
        
        Console.ReadLine();   // Prevent the process from terminating
    }

    // This method's signature must match the TimerCallback delegate
    private static void Status(Object state) {
        // This method is executed by a thread pool thread
        Console.WriteLine("In Status at {0}", DateTime.Now);
        
        Thread.Sleep(1000);  // Simulates other work (1 second)
        
        // Just before returning, have the Timer fire again in 2 seconds
        s_timer.Change(2000, Timeout.Infinite);
        // When this method returns, the thread goes back
        // to the pool and waits for another work item
    }
}
```

There is another way to perform operation periodically: **Task.Delay** and async & await

```c#
internal static class DelayDemo {
    public static void Main() {
        Console.WriteLine("Checking status every 2 seconds");
        Status();
        Console.ReadLine();   // Prevent the process from terminating
    }

    // This method can take whatever parameters you desire
    private static async void Status() {
        while (true) {
            Console.WriteLine("Checking status at {0}", DateTime.Now);

            await Task.Delay(2000); // await allows thread to return

            // After 2 seconds, some thread will continue after await to loop around
        } 
    }
}
```