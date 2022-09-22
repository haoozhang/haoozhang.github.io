---
layout:     post
title:      Chapter 21. Heap and Garbage Collection
subtitle:   
date:       2022-09-10
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

How managed applications construct new objects. \
How the managed heap controls the lifetime of these objects. \
How the memory for these objects get reclaimed.

## Managed Heap Basics

Every program use resources, files, memory buffers, network connection, and so on. The following steps are required to access a resource:
1. Allocate memory for the type that represents the resource (usually completed by using **new** operator).
2. Initialize the memory to set the initial state of the resource, the type's instance constructor is responsible for setting initial state.
3. Use the resource by accessing the type's field.
4. Tear down the state of a resource to cleanup
5. Free the memory. The garbage collector is responsible for this step.

If the developers manually manage the memory, they often forget to free memory, which causes a memory leak. Or they frequently use memory after having released it, causing the program to experience memory corruption.

As long as you write verifiable type-safe code, it is impossible for your application to experience memory corruption. But if you store objects in a collection and nevet remove them when no longer used, it is still possible to leak memory.

To simplify even more, most types don't require step 4. There is no need to clean up the resource and the garbage collector will free the memory. However, sometimes, you want to clean up a resource as soon as possible, rather than waiting for a GC. In these classes, you can call one additional method (called **Dispose**) in order to clean up the resource on your schedule.

### Allocating Resources from the Managed Heap

CLR requires that all objects be allocated from the **managed heap**. When a process is initialized, CLR allocates a region of address space, and maintains a pointer, **NectObjPtr**, which indicates where the next object is to be allocated.

C#’s new operator causes the CLR to perform the following steps:
1. Calculate the number of bytes required for the type’s fields (and all fields inherited from base types).
2. Add the bytes required for an object’s overhead. Each object has two overhead fields: a type object pointer and a sync block index.
3. The CLR then checks that the bytes required to allocate the object. If there is enough free space in the managed heap, the object will fit.

The managed heap allocates objects next to each other in memory, so you get excellent performance when accessing these objects due to locality of reference. Specifically, this means that your process’s working set is small, and the objects sccessed by your code can all reside in the CPU cache.

### Garbage Collection Algorithm

When an application calls the **new** operator to create an object, there might not be enough address space left to allocate the object. If insufficient space exists, then the CLR performs a GC.

For managing the lifetime of objects, some systems use a **reference counting algorithm**. Each object on the heap maintains an internal field indicating how many "parts" of the program are currently using that object. As each "part" no longer requires access to an object, it decrements that object’s count field. When the count field reaches 0, the object deletes itself from memory.

The big problem with many reference counting systems is that they **do not handle circular references** well. For example, A holds a reference to B, and B holds a reference to A, both objects will never be deleted.

Due to this problem with reference counting algorithm, CLR uses a **referencing tracking algorithm** instead. The reference tracking algorithm cares only about reference type variables, because only these variables can refer to an object on the heap. We refer to all reference type variable as **roots**.

When the CLR starts a GC, the CLR first suspends all threads in the process. Then, CLR performs **marking** phrase. It looks at all active roots to see which objects they refer to. When meeting an already-marked object, it does not loop it again and to next root, which prevents infinite loop for circular reference.

Now that the CLR knows that which objects must survive (marked objects) and which objects can be deleted (unmarked objects). It begins the GC’s **compacting** phase. The CLR shifts the memory consumed
by the marked objects in the heap, compacting all the surviving objects together so that they are contiguous in memory.

In order to make the application thread can still access those moved objects, the CLR subtracts from each root the number of shifted bytes. After the heap memory is compacted, the managed heap’s **NextObjPtr** pointer is set to the location just after the last surviving object. 

Notice that the two bugs (memory leak and memory corruption) no longer exist. First, any object not accessible will be collected, so it's impossible to leak memory. Second, references can only refer to living objects, so it's not possible to access an already-released object.

## Generations

CLR’s GC is a **generational garbage collector**, which makes the following assumptions:
+ The newer object has shorter lifetime.
+ The older object has longer lifetime.
+ Collecting a portion of heap is faster than collecting whole heap.

The managed heap supports three generations: generation 0, generation 1, generation 2. When the CLR initializes, it selects budgets for all generations. \
The managed heap contains no objects initially. Objects are firstly added to generation 0. When generation 0 has no more budget for adding object, a garbage collection must start. \
After the garbage collection, those survived objects are saied to be in generation 1, and generation 0 has no objects at this time. After collecting garbage in generation 0 for multiply times, the generation 1 may also contains enouth objects and has no more budgets, so it will also be collected. \
After collecting the garbage in generation 1, those survived objects will be saied to be in generation 2.

### Garbage Collection Triggers

+ Code explicitly calls **System.GC**
+ System reports low memory
+ CLR is unloading an AppDomain
+ CLR is shutting down

### Large Objects

CLR considers each object to be either a small object or a large object. For large objects,

+ They are not allocated within the same address space as small objects;
+ GC doesn’t compact large objects because of high expense for moving it;
+ Large objects are the part of generation 2; they are never in generation 0 or 1.

### Garbage Collection Modes

When the CLR starts, it selects a GC mode, and this mode cannot change during the lifetime of the process.
+ **Workstation** for client-side applications. It is optimized to provide low-latency GCs in order to minimize the time application’s threads are suspended.

+ **Server** for server-side applications. It is optimized for throughput and resource utilization. A special thread for garbage collection is running per CPU, and parallel collections work well for server applications to attain a performance improvement.

By default, applications run with the Workstation GC mode. A server application that hosts the CLR can request the CLR to load the Server GC.

Create a configuration file that contains a **gcServer** element to tell CLR to use Server GC mode.

```c#
<configuration>
    <runtime>
        <gcServer enabled="true"/>
    </runtime>
</configuration>
```

Ask the CLR if it is running in the Server GC mode.

```c#
using System;
using System.Runtime; // GCSettings is in this namespace
public static class Program {
    public static void Main() {
        Console.WriteLine("Application is running with server GC=" + GCSettings.IsServerGC);
    }
}
```

GC also supports two sub-modes: concurrent (the default) or non- concurrent. You can tell the CLR not to use the concurrent collector by **gcConcurrent** element.

```c#
<configuration>
   <runtime>
      <gcConcurrent enabled="false"/>
   </runtime>
</configuration>
```

### Forcing Collection

The **System.GC** type allows to directly control the garbage collector, you can query the maximum generations supported by the managed heap by **GC.MaxGeneration** property; this property always return 2.

You can also force collection by calling **GC.Collect** method, but under most circumstances, you should **avoid** calling any of the Collect methods, it’s best just to let the garbage collector run on its own accord.

```C#
void Collect(Int32 generation, GCCollectionMode mode, Boolean blocking);
```

### Monitoring Application's Memory Usage

There are a few methods that you can call to monitor the garbage collector. For instance, you can see how many collections have occurred of a specific generation or how much memory is currently being used by objects in the managed heap.

```c#
Int32 CollectionCount(Int32 generation);
Int64 GetTotalMemory(Boolean forceFullCollection);
```

## Requiring Specific Cleanup

most types need only memory to operate. However, some types require more than just memory, for example, the **System.IO.FileStream** type needs to open a file (a native resource) and store the file’s handle. Then the type’s Read and Write methods use this handle to manipulate the file.

Let's say that you want to create a temp file, write some bytes to the file, and then delete the file. You might start writing the code like this.

```c#
using System;
using System.IO;
public static class Program {
    public static void Main() {
       // Create the bytes to write to the temporary file.
       Byte[] bytesToWrite = new Byte[] { 1, 2, 3, 4, 5 };
       // Create the temporary file.
       FileStream fs = new FileStream("Temp.dat", FileMode.Create);
       // Write the bytes to the temporary file.
       fs.Write(bytesToWrite, 0, bytesToWrite.Length);
       // Delete the temporary file.
       File.Delete("Temp.dat");  // Throws an IOException
    }
}
```

Unfortunately, it mostly won't work. Because the delete operation is against the opening file, so it will throw IOException.

In some cases, the file might actually be deleted! If another thread somehow caused a garbage collection to start after **Write** method and before **Delete** method, **fs** will close the file, and the delete operation will work. But the previous code will fail more than 99 percent of the time.

If the calss allows the consumer to control the lifetime of native resource, it should implement the **IDispose** interface. So here’s the corrected source code.

```c#
using System;
using System.IO;
public static class Program {
    public static void Main() {
       // Create the bytes to write to the temporary file.
       Byte[] bytesToWrite = new Byte[] { 1, 2, 3, 4, 5 };
       // Create the temporary file.
       FileStream fs = new FileStream("Temp.dat", FileMode.Create);
       try {
          // Write the bytes to the temporary file.
          fs.Write(bytesToWrite, 0, bytesToWrite.Length);
       }
       finally {
            // Explicitly close the file when finished writing to it.
            if (fs != null)  fs.Dispose();
       }
       // Delete the temporary file.
       File.Delete("Temp.dat");
    }
}
```

Recommend to place the call of **Dispose** explicitly into the **finally** block.

C# also provides a **using** statement, which offers a simplified syntax.

```c#
using System;
using System.IO;
public static class Program {
    public static void Main() {
       // Create the bytes to write to the temporary file.
       Byte[] bytesToWrite = new Byte[] { 1, 2, 3, 4, 5 };
       // Create the temporary file.
       using (FileStream fs = new FileStream("Temp.dat", FileMode.Create)) {
          // Write the bytes to the temporary file.
          fs.Write(bytesToWrite, 0, bytesToWrite.Length);
       }
       // Delete the temporary file.
       File.Delete("Temp.dat");
    }
}
```