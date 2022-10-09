---
layout:     post
title:      Chapter 26. Thread Basics
subtitle:   
date:       2022-10-06
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

## Why Does Windows Support Threads?

In the early days of computers, operating systems didn’t offer the concept of a thread. There was just one executing thread throughout the entire system, which included both operating system code and application code. 
The problem is that a long-running task would prevent other tasks from executing, e.g., in the days of 16-bit Windows, a task of printing a document occupies current executing thread, causing the operating system and all other applications to stop responding.

When Microsoft was designing this operating system kernel, they decided to run each instance of an application in what is called a **process**. A process is just a collection of resources that is used by a single instance of an application. Each process is given a virtual address space, ensuring that the code and data used by one process is not accessible to another process. So now, application code cannot corrupt other applications or the operating system.

But what about the CPU itself? What if an application enters an infinite loop? If there is only one CPU in the machine, then it executes the infinite loop and cannot execute anything else, the system could still stop responding to the end user. **Threads** were the answer. A thread is a Windows concept whose job is to virtualize the CPU. Each process is given its own thread, and if application code enters an infinite loop, the process associated freezes up, but other processes (which have their own threads) are not frozen; they keep running!

## Thread Overhead

Threads are awesome because they enable Windows to be responsive even when applications are executing long-running tasks. But as with every virtualization mechanism, threads have space (memory consumption) and time (runtime execution performance) overhead.

Every thread has one of each of the following:
+ Thread kernel object. The data structure contains a bunch of properties that describe the thread, and thread's context.
+ User-mode stack. The user-mode stack is used for local variables and arguments passed to methods.
+ Kernel-mode stack. The kernel-mode stack is also used when application code passes arguments to a kernel-mode function in the operating system.

In addition, we’re going to start talking about context switching. A computer with only one CPU in it can do only one thing at a time. Therefore, Windows has to share the actual CPU hardware among all the threads.

At any given moment in time, Windows assigns one thread to a CPU. That thread is allowed to run for a time-slice. When the time-slice expires, Windows context switches to another thread. Every context switch requires that Windows performs the following actions:
1. Save the values in the CPU’s registers to the currently running thread’s context structure inside the thread’s kernel object.
2. Select one thread from the set of existing threads to schedule next. If this thread is owned by another process, then Windows must also switch the virtual address space seen by the CPU before it starts executing any code or touching any data.
3. Load the values in the selected thread’s context structure into the CPU’s registers.

After the context switch is complete, the CPU executes the selected thread until its time-slice expires, and then another context switch happens again.

In addition, when performing a garbage collection, the CLR must suspend all the threads, walk their stacks to find the roots to mark objects in the heap, walk their stacks again (updating roots to objects that moved during compaction), and then resume all the threads.

From this discussion, you should avoid using threads as much as possible because they consume a lot of memory and they require time to create, destroy, and manage. 

I should also point out that a computer with multiple CPUs can actually run multiple threads simultaneously, increasing scalability. Windows will assign one thread to each CPU core, and each core will perform its own context switching to other threads.

## About Thread Count

If we cared about performance only, then the optimum number of threads is identical to the number of CPUs. So a machine with one CPU would have only one thread, a machine with two CPUs would have two threads, and so on. The reason is obvious: if you have more threads than CPUs, then context switching is introduced and performance drops. If each CPU has just one thread, then no context switching exists and the threads run at full speed.

However, Microsoft designed Windows to favor reliability and responsiveness as opposed to favoring raw speed and performance, because applications may stop the operating system and other applications.

## CPU Trends

In the past, CPU manufactures tends to increase CPU speeds. However, when CPU runs at high speeds, they produce more heat and consume more power. So CPU manufacturers can’t continuously produce higher-speed CPUs, they turned their attention to make smaller size. So that, a chip can contain more CPU cores.

Computers use three kinds of multi-CPU technologies today:
+ Multiple CPUs. Some Computers just have multiple CPUs.
+ Hyperthreaded chips. This technology (owned by Intel) allows a single chip to look like two chips. The chip contains two sets of architectural states, but has only one set of execution resources. To Windows, this looks like there are two CPUs in the machine, so Windows schedules two threads concurrently. When one thread pauses due to a cache miss, branch misprediction, or data dependency, the chip switches to the other thread. This all happens in hardware, and Windows doesn’t know it.
+ Multi-core chips. single chip contains multiple CPU cores.

## Use a Dedicated Thread to Perform an Async Operation

how to create a thread and have it perform an asynchronous compute-bound operation.

```c#
public sealed class Thread : CriticalFinalizerObject, ... {
    public Thread(ParameterizedThreadStart start);
    ....
}
```

The **start** parameter identifies the method that the dedicated thread will execute, and this method must match the signature of the **ParameterizedThreadStart** delegate.

```c#
delegate void ParameterizedThreadStart(Object obj);
```

Constructing a Thread object is a relatively lightweight operation because it does not actually create a physical operating system thread. To actually create the operating system thread and have it start executing the callback method, you must call **Thread’s Start** method, passing into the callback method's argument.
```c#
using System;
using System.Threading;

public static class Program {

    public static void Main() {
        Console.WriteLine("Main thread: starting a dedicated thread to do an asynchronous operation");
        Thread dedicatedThread = new Thread(ComputeBoundOp);
        dedicatedThread.Start(5);
        Console.WriteLine("Main thread: Doing other work here...");
        Thread.Sleep(10000);     // Simulating other work (10 seconds)
        dedicatedThread.Join();  // Wait for thread to terminate
        Console.WriteLine("Hit <Enter> to end this program...");
        Console.ReadLine();
    }

    // This method's signature must match the ParameterizedThreadStart delegate
    private static void ComputeBoundOp(Object state) {
       // This method is executed by a dedicated thread
       Console.WriteLine("In ComputeBoundOp: state={0}", state);
       Thread.Sleep(1000);  // Simulates other work (1 second)
       // When this method returns, the dedicated thread dies
    }
}
```
Notice that the **Join** method causes the calling thread to stop executing any code until the dedicated thread has destroyed or been terminated.

## Reasons to Use Threads

There are really two reasons to use threads:
+ Responsiveness (for GUI applications). Windows gives each process its own thread so that one application entering an infinite loop doesn’t prevent the user from working with other applications. Similarly, within your client-side GUI application, you could spawn some work off onto a thread so that your GUI thread remains responsive to user input.
+ Performance (for client and server side applications). Because Windows can schedule one thread per CPU and because the CPUs can execute these threads concurrently, your application can improve its performance by having multiple operations executing at the same time in parallel.

Every computer has an powerful resource: the CPU itself. So we has better consume it as soon as possible.
Here’s an example: When you stop typing in Visual Studio editor, Visual Studio automatically schedules the compiler and compiles your code. This makes developers more productive because they can see warnings and errors immediately. So, the traditional **Edit-Build-Debug** cycle will become **Edit-Debug** cycle, because building code happens all the time.

## Thread Scheduling and Priorities

Earlier in this chapter, every thread’s kernel object contains a context structure, which reflects the state of the thread’s CPU registers when the thread last executed. After a time-slice, Windows looks at all the thread kernel objects currently. Of these objects, only the threads that are not waiting for something are considered schedulable. Windows selects one of the schedulable thread kernel objects and context switches to it. Then, the thread is executing code and manipulating data in its process’s address space. After another time-slice, Windows performs another context switch.

Windows is called a preemptive multithreaded operating system because a thread can be stopped at any time and another thread can be scheduled. You cannot guarantee that your thread will always be running and also cannot guarantee that no other thread will be allowed to run.

Every thread is assigned a priority level ranging from 0 (the lowest) to 31 (the highest). When the system decides which thread to assign to a CPU, it examines the priority 31 threads first. If a priority 31 thread is schedulable, it is assigned to a CPU. At the end of this thread’s time-slice, the system checks to see whether there is another priority 31 thread. If so, it allows that thread to be assigned to a CPU.

As long as priority 31 threads are schedulable, the system never assigns any thread with a prior- ity of 0 through 30 to a CPU, which may incurs **starvation**. That is, higher-priority threads use too much CPU time and prevent lower-priority threads from executing. 

Higher-priority threads always preempt lower-priority threads, regardless of what the lower-priority threads are executing. For example, if a priority 5 thread is running and the system determines that
a higher-priority thread is ready to run, the system immediately suspends the lower-priority thread and assigns the CPU to the higher-priority thread.

By the way, when the system boots, it creates a special thread called the **zero page thread**. This thread is assigned priority 0 and is the only priority 0 thread in the entire system. The zero page thread is responsible for zeroing any free pages of RAM in the system when no other threads need to perform work.

## Foreground Threads vs. Background Threads

The CLR considers every thread to be either a **foreground thread** or a **background thread**. When all the foreground threads in a process stop running, the CLR forcibly ends any background threads. These background threads are ended immediately; no exception is thrown.

Therefore, you should use foreground threads to execute tasks that you really want to complete, like flushing data from a memory buffer out to disk. And you should use background threads for tasks that are not critical.

The following code demonstrates the difference between foreground and background threads.

```c#
using System;
using System.Threading;

public static class Program {

    public static void Main() {
        // Create a new thread (defaults to foreground)
        Thread t = new Thread(Worker);
        // Make the thread a background thread
        t.IsBackground = true;

        t.Start(); // Start the thread
        // If t is a foreground thread, the application won't die for about 10 seconds
        // If t is a background thread, the application dies immediately
        Console.WriteLine("Returning from Main");
    }

    private static void Worker() {
       Thread.Sleep(10000);  // Simulate doing 10 seconds of work
       // The following line only gets displayed if this code is executed by a foreground thread
       Console.WriteLine("Returning from Worker");
    }
}
```

An application’s primary thread and any threads explicitly created by constructing a **Thread** object default to being foreground threads. On the other hand, thread pool threads default to being background threads. Also, any threads created by native code are marked as background threads.

## Conclusion

In this chapter, We’ve explained the basics about threads, and I hope I’ve made it clear to you that **threads are very expensive resources** that should be used sparingly. The best way to accomplish this is by using the **thread pool**. The thread pool will manage thread creation and destruction for you automatically. The thread pool creates a set of threads that get reused for various tasks so your application requires just a few threads to accomplish all of work.

In Chapter 27, will focus on how to use the thread pool to perform compute-bound operations. \
In Chapter 28, will discuss how to use the thread pool to perform I/O-bound operations. \
In Chapter 29 and 30, will discuss the way that the thread synchronization constructs, and the difference between these various constructs.


