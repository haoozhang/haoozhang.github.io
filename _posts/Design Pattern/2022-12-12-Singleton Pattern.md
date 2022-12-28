---
layout:     post
title:      Singletion Pattern
subtitle:   
date:       2022-12-12
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

There are many objects that we need only one: thread pool, recycle bin, task manager and etc. In fact, for many of these types of objects, if we were to initialize more than one instance we’d run into all sorts of problems.

A global variable can solve this problem, but it has some drawbacks. For example, if we assign on object to a global variable, then the object will be created when application starts up. But if we don't use it in whole lifetime and this object occupies much resources, these resources will be wasted.

### Definition

The **Singleton Pattern** ensures a class has only one instance, and provides a global point of access to it.

### Applicability

When a class should have just a single instance available to all clients; for example, a single database object shared by different parts of the program.

### Structure

![img](/img/DesignPattern/singleton.png)

### Participants

All implementations of the singleton have these steps in common:
+ Define a static filed, which saves the singleton instance.
+ Make the default constructor as **private**, to prevent other objects from using the **new** operator to construct instance.
+ Create a static creation method that acts as a constructor, where this method calls the private constructor to create an object and saves it in the above static field. All following calls to this method return the cached object.

### Consequence

You can be sure that a class has only a single instance.

You gain a global access point to that instance.

### Implementation

The first implementation shows as follows. It uses early-init way to initialize static variable when start up. As mentioned earilier, the object will be wasted if it not used in whole lifetime.

```c#
// initialize static variable when initializing the class
private static readonly Singleton Singleton = new Singleton();

private Singleton() { }

public Singleton GetSingleton()
{
    // return directly
    return Singleton;
}
```

So we introduced the second implementation, which uses lazy-init way. The instance won't be constructed until it is accessed for the first time. But it has the drawback that the **synchronized** keyword works well for the first time access, but all following calls to this method don't have to be **synchronized** and it will hurt the performance.

```java
private static Singleton Singleton;

private Singleton() { }

public static synchronized Singleton GetSingleton()
{
    if (Singleton == null)
    {
        Singleton = new Singleton();
    }        
    return Singleton;
}
```

To improve the performance in multiple threads environment, we introduce the **double-check locking** technique as follows. 

```java
// volatile keyword ensures that the Singleton variable can be constructed completely.
// JVM first allocate the memory for Singleton variable, init the instance using construcor, and assign the reference to Singleton.
// But in actual rare case, JVM could assign the reference before initializing the instance. So at this time, other threads may get the incomplete instance.
private volatile static Singleton Singleton;

private Singleton() { }

public static Singleton GetSingleton()
{
    if (Singleton == null)
    {
        // multiple threads acquire this lock, the one acquired will create singleton instance
        synchronized (Singleton.class)
        {
            if (Singleton == null) 
            {
                Singleton = new Singleton();
            }
        }
    }        
    return Singleton;
}
```
By the way, in C# language, we can implement as follows using **Monitor**. The **Monitor** keyword here takes the same effect as **synchronized** before.

```c#
private static readonly Object s_lock = new Object();

private static Singleton s_value = null;

private Singleton() { }

public static Singleton GetSingleton() {
    if (s_value == null) {
        Monitor.Enter(s_lock);

        if (s_value == null) {
            Singleton temp = new Singleton();
            Volatile.Write(ref s_value, temp);
        }

        Monitor.Exit(s_lock);
    }
    return s_value;
}
```

And we can also use the **Interlocked** to implement as follows. Note that even if multiple threads can construct more than one **temp** variable, the **Interlocked** allows only one can be assigned to **Singleton** variable. So, we just need to check **Singleton** once.

```c#
private static SingletonLazyInit Singleton;
    
private SingletonLazyInit() { }

public static  SingletonLazyInit GetSingleton()
{
    if (Singleton == null)
    {
        SingletonLazyInit temp = new SingletonLazyInit();
        // assign with atomic way
        Interlocked.CompareExchange(ref Singleton, temp, null);
    }        
    return Singleton;
}
```

### Known Uses



### Related Patterns

