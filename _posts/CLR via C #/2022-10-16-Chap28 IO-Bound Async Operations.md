---
layout:     post
title:      Chapter 28. IO-Bound Asynchoronous Operations
subtitle:   
date:       2022-10-16
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

The previous chapter focused on ways to perform compute-bound async operations, allowing the thread pool to schedule the tasks to multiple cores, so that multiple threads can work concurrently and increase throughput. In this chapter, we’ll focus on performing IO-bound operations asynchronously, allowing hardware devices to handle the tasks while threads and the CPU are not used all the time.

## How Windows Performs I/O Operations

Let’s begin by discussing how Windows performs synchronous I/O operations. In your program, \
1) you open a disk file by constructing a **FileStream** object. 
2) you call the **Read** method to read data from the file. 
    + when calling **FileStream’s Read** method, thread transitions from managed code to native/user-mode code
    + **Read** internally calls the Win32 **ReadFile** function, **ReadFile** function allocates a small data structure called IO Request Packet (IRP).
    + **ReadFile** function transitions from native/user mode to native/kernel mode, and call Windows kernel, passing IRP to kernel.
    + based on the IRP, Windows kernel delivers the IRP to appropriate device driver.
    + the hardware device now performs the requested I/O operation.
    + **note that**: when the hardware is performing I/O request, your thread has nothing to do, so Windows puts your thread to sleep. But it still wastes memory (user-mode stack, kernel-mode stack, etc).
3) The hardware device completes the I/O operation, and then Windows will wake up your thread, schedule it to a CPU, and let it return from kernel mode to user mode, and then back to managed code.
4) **FileStream’s Read** method now returns an Int32, indicating the actual number of bytes read from the file.

Let’s imagine that you are implementing a web application. Each client request comes into your server, you need to make a database request. Then a thread pool thread will call into your code, if you call synchronously, the thread will block to wait for the response. As more and more client requests come in, more and more threads are created, and all these threads block waiting for the database to respond.

Now, let’s discuss how Windows performs asynchronous I/O operations.
1) open the disk file by constructing a **FileStream** object, passing **FileOptions.Asynchronous** flag.
2) call **ReadAsync** (instead of Read) to allocate a **Task\<Int32>** object, which represents the read operation.
    + **ReadAsync** calls Win32’s **ReadFile** function.
    + **ReadFile** allocates its IRP.
    + **ReadFile** function passes IRP to kernel.
    + Windows adds the IRP to the hard disk driver.
    + **note that:**: now, instead of blocking your thread, your thread returns from its call to **ReadAsync**.
3) after hardware completes processing the IRP, it will queue the completed IRP into the CLR’s thread pool. Sometimes in the future, a thread pool thread will extract the completed IRP and execute code to complete the task.

Now, a client request comes in, and our server makes an asynchronous database request. Then our thread won't block, and it returns to the thread pool to handle more incoming client requests. So we have just one thread handling all incoming client requests. \
When the database server responds, its response is also queued into the thread pool, so our thread pool thread will just process it at some point and ultimately send the necessary data back to the client. So we have just one thread processing all client requests and all database responses.

## C#’s Asynchronous Functions

Performing async operations is the key to build responsive applications that allow to use few threads to execute lots of operations. And plus the thread pool, async operations allow you to take advantage of all of the CPUs. \
Microsoft designed a programming model that would make it easy for developers to take advantage of this capability, which leverages **Tasks** and **C# async functions**.

Here is an example that uses an async function.

```c#
private static async Task<String> IssueClientRequestAsync(String serverName, String message) {
    using (var pipe = new NamedPipeClientStream(serverName, "PipeName", PipeDirection.InOut, PipeOptions.Asynchronous | PipeOptions.WriteThrough)) {
        pipe.Connect(); // Must Connect before setting ReadMode
        pipe.ReadMode = PipeTransmissionMode.Message;

        // Asynchronously send data to the server
        Byte[] request = Encoding.UTF8.GetBytes(message);
        await pipe.WriteAsync(request, 0, request.Length);

        // Asynchronously read the server's response
        Byte[] response = new Byte[1000];
        Int32 bytesRead = await pipe.ReadAsync(response, 0, response.Length);

        return Encoding.UTF8.GetString(response, 0, bytesRead);
   }  // Close the pipe
}
```

When the thread calls **IssueClientRequestAsync** method, it constructs a **NamedPipeClientStream** object, then call **Connect()** and set its property. \
Then the thread calls **WriteAsync**, **WriteAsync** internally allocates a **Task** object and returns it back to **IssueClientRequestAsync**. \
At this point, C# **await** operator calls **ContinueWith** on the **Task** object, then returns from **IssueClientRequestAsync** method.

Sometime in the future, the network device completes writing data to the pipe. \
Then a thread pool thread will notify the **Task** object, this Task object will activate the **ContinueWith** callback method, and re-enter the **IssueClientRequestAsync** method (start from **await**), the await operator returns the result. \
In this case, WriteAsync returns a Task instead of a Task<TResult>, so there is no return value.

Now, our method continues executing **ReadAsync** async method. Internally, **ReadAsync** creates a **Task\<Int32>** object and returns it. the **await** operator effectively calls **ContinueWith** on the **Task\<Int32>** object. Then, the thread returns from Issue­ ClientRequestAsync again.

Sometime in the future, the server will send a response back to the client. A thread pool will notify the **Task\<Int32>** object, the **await** operator will query the **Task** object’s **Result** property, and assigns the resuslt to the **bytesRead** local variable. Then, the thread runs the rest of the code.

Some restrictions related to async functions:
+ Can't make Main, constructors, property/event accessor methods an async function.
+ Can't have any **out** or **ref** parameters on an async function.
+ Can't use the **await** operator inside a **catch**, **finally**, or **unsafe** block.

## Canceling IO Operations

In general, Windows doesn’t give you a way to cancel an outstanding I/O operation. To assist with this, I recommend you implement a **WithCancellation** extension method that extends **Task\<TResult>** (you need a similar overload that extends **Task** too) as follows.

```c#
private struct Void { } // Because there isn't a non­generic TaskCompletionSource class.
      
private static async Task<TResult> WithCancellation<TResult>(this Task<TResult> originalTask, CancellationToken ct) {
    // Create a Task that completes when the CancellationToken is canceled
    var cancelTask = new TaskCompletionSource<Void>();
    
    // When the CancellationToken is canceled, complete the Task
    using (ct.Register(
       t => ((TaskCompletionSource<Void>)t).TrySetResult(new Void()), cancelTask)) {
       // Create a Task that completes when either the original or
       // CancellationToken Task completes
       Task any = await Task.WhenAny(originalTask, cancelTask.Task);

       // If any Task completes due to CancellationToken, throw OperationCanceledException
       if (any == cancelTask.Task) ct.ThrowIfCancellationRequested();
    }

    // await original task (synchronously); if it failed, awaiting it
    // throws 1st inner exception instead of AggregateException
    return await originalTask;
}
```

Now, you can call this extension method as follows.

```c#
public static async Task Go() {
   // Create a CancellationTokenSource that cancels itself after # milliseconds
   var cts = new CancellationTokenSource(5000); // To cancel sooner, call cts.Cancel()
   var ct = cts.Token;

    try {
        // I used Task.Delay for testing; replace this with another method that returns a Task
        await Task.Delay(10000).WithCancellation(ct);
        Console.WriteLine("Task completed");
    }
    catch (OperationCanceledException) {
        Console.WriteLine("Task cancelled");
    }
}
```
