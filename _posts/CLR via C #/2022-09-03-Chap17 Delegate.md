---
layout:     post
title:      Chapter 17. Delegates
subtitle:   
date:       2022-09-03
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

.NET Framework exposes a **callback function** mechanism by using **delegates**.

Unlike unmanaged C++, delegated in C# ensure that callback method is type-safe. \
Delegates also can call multiple methods sequentially, and support to call static methods as well as instance methods.

The following code demonstrates how to declare, create, and use delegates.

```c#
using System;
using System.Windows.Forms;
using System.IO;

// Declare a delegate type; instances refer to a method that
// takes an Int32 parameter and returns void.
internal delegate void Feedback(Int32 value);

public sealed class Program {
    public static void Main() {
        StaticDelegateDemo();
        InstanceDelegateDemo();
        ChainDelegateDemo1(new Program());
        ChainDelegateDemo2(new Program());
    }

    private static void StaticDelegateDemo() {
        Console.WriteLine("­­­­­ Static Delegate Demo ­­­­­");
        Counter(1, 3, null);
        Counter(1, 3, new Feedback(Program.FeedbackToMsgBox));
        Counter(1, 3, new Feedback(FeedbackToMsgBox));
        Console.WriteLine();
    }

    private static void InstanceDelegateDemo() {
        Console.WriteLine("­­­­­ Instance Delegate Demo ­­­­­");
        Program p = new Program();
        Counter(1, 3, new Feedback(p.FeedbackToFile));
        Console.WriteLine();
    }

    private static void ChainDelegateDemo1(Program p) {
        Console.WriteLine("­­­­­ Chain Delegate Demo 1 ­­­­­");
        Feedback fb1 = new Feedback(FeedbackToConsole);
        Feedback fb2 = new Feedback(FeedbackToMsgBox);
        Feedback fb3 = new Feedback(p.FeedbackToFile);

        // combine delegate with .Combine
        Feedback fbChain = null;
        fbChain = (Feedback) Delegate.Combine(fbChain, fb1);
        fbChain = (Feedback) Delegate.Combine(fbChain, fb2);
        fbChain = (Feedback) Delegate.Combine(fbChain, fb3);
        Counter(1, 2, fbChain);

        Console.WriteLine();
        fbChain = (Feedback)
           Delegate.Remove(fbChain, new Feedback(FeedbackToMsgBox));
        Counter(1, 2, fbChain);
    }

    private static void ChainDelegateDemo2(Program p) {
        Console.WriteLine("­­­­­ Chain Delegate Demo 2 ­­­­­");
        Feedback fb1 = new Feedback(FeedbackToConsole);
        Feedback fb2 = new Feedback(FeedbackToMsgBox);
        Feedback fb3 = new Feedback(p.FeedbackToFile);

        // combine delegate with +=
        Feedback fbChain = null;
        fbChain += fb1;
        fbChain += fb2;
        fbChain += fb3;
        Counter(1, 2, fbChain);

        Console.WriteLine();
        fbChain ­= new Feedback(FeedbackToMsgBox);
        Counter(1, 2, fbChain);
    }

    private static void Counter(Int32 from, Int32 to, Feedback fb) {
        for (Int32 val = from; val <= to; val++) {
            // If any callbacks are specified, call them
            if (fb != null)
                fb(val); 
        }
    }

    private static void FeedbackToConsole(Int32 value) {
        Console.WriteLine("Item=" + value);
    }

    private static void FeedbackToMsgBox(Int32 value) {
        MessageBox.Show("Item=" + value);
    }

    private void FeedbackToFile(Int32 value) {
        using (StreamWriter sw = new StreamWriter("Status", true)) {
            sw.WriteLine("Item=" + value);
        }
    }
}
```

## Using Delegates to Call Back Static Methods

As aboved **StaticDelegateDemo** method.

Both C# and the CLR allow for **covariance and contra-variance** of **reference types** (not value types) when binding a method to a delegate.

For example, given a delegate defined like this:
```c#
delegate Object MyCallback(FileStream s);
```

it is possible to construct an instance of this delegate type bound to a method that is prototyped like this.

```c#
String SomeMethod(Stream s);
```

Here, *SomeMethod*’s return type (*String*) is a type that is derived from the delegate’s return type (*Object*); this covariance is allowed. *SomeMethod*’s parameter type (*Stream*) is a type that is a base class of the delegate’s parameter type (*FileStream*); this contra-variance is allowed.

## Using Delegates to Call Back Instance Methods

As aboved **InstanceDelegateDemo** method.

## Behind Delegates

How the compiler and the CLR work together to implement delegates.

For this line of code,
```c#
internal delegate void Feedback(Int32 value);
```

Actually, compiler defines a complete class that looks something like this.
```c#
internal class Feedback : System.MulticastDelegate {
    // Constructor
    public Feedback(Object @object, IntPtr method);

    // Method with same prototype as specified by the sourcecode
    public virtual void Invoke(Int32 value);

    // Methods allowing the callback to be called asynchronously
    public virtual IAsyncResult BeginInvoke(Int32 value, AsyncCallback callback, Object @object);
    public virtual void EndInvoke(IAsyncResult result);
}
```

All delegate types are derived from **MulticastDelegate**. They inherit Multicast­ Delegate’s fields, properties, and methods. Of all of these members, three non-public fields are probably most significant.

+ **_target** (System.Object) : When the delegate object wraps a static method, this field is null. When the delegate objects wraps an instance method, this field refers to the object that should be operated on.
+ **_methodPtr** (System.IntPtr) : Used to identify the method that is to be called back.
+ **_invocationList** (System.Object) : This field is usually null. It can refer to an array of delegates when building a delegate chain.

So, when constructing a delegate instance with a method, compiler knows that a delegate is being constructed and parses the source code to determine which object and method are being referred to.

For the following delegate objects that are initialized,
```c#
Feedback fbStatic   = new Feedback(Program.FeedbackToConsole);
Feedback fbInstance = new Feedback(new Program().FeedbackToFile);
```
The *fbStatic* and *fbInstance* variables refer to two seperate delegate objects.
![img](/img/CLR/delegate_init.png)

Now that you know how delegate objects are constructed and what their internal structure looks like, let’s talk about how the callback method is invoked.

```c#
private static void Counter(Int32 from, Int32 to, Feedback fb) {
    for (Int32 val = from; val <= to; val++) {
        // If any callbacks are specified, call them
        if (fb != null)
            fb(val); 
    }
}
```

Look at the line of code just below the comment. Because *fb* is really just a variable that refers to a *Feedback* delegate object; it could also be null. We should check if it is null first.

Then, it looks like we call a function named *fb* and pass it one parameter (*val*). However, **there is no function called fb.** Compiler actually generates code to call the delegate object’s *Invoke* method.

When *Invoke* is called, it uses the private **_target** and **_methodPtr** fields to call the desired method on the specified object. 

Note that the signature of the *Invoke* method matches the signature of the delegate. Because the *Feedback* delegate takes one Int32 parameter and returns void, the *Invoke* method (produced by the compiler) takes one Int32 parameter and returns void.

## Using Delegates to Call Back Many Methods (Chaining)

As aboved **ChainDelegateDemo1** and **ChainDelegateDemo2**, the *fbChain* will be initialized as follows.

![img](/img/CLR/delegate_chain.png)

```c#
1) Feedback fbChain = null;
2) fbChain = (Feedback) Delegate.Combin(fbChain, fb1);
3) fbChain = (Feedback) Delegate.Combin(fbChain, fb2);
4) fbChain = (Feedback) Delegate.Combine(fbChain, fb3);
```
After line 1, *fbChain* points to null; \
After line 2, *fbChain* points to *fb1*; \
After line 3, *fbChain* constructs a new delegate object and is show as figure; \
After line 4, *fbChain* also constructs a new delegate object and is show as figure; \

## Generic Delegates

In fact, .NET Framework ships with **Action** and **Func** generic delegates. It is now recommended that these delegate types be used wherever possible instead of developers defining even more delegate types in their code. 

```c#
public delegate void Action();
public delegate void Action<T>(T obj);
public delegate void Action<T1, T2>(T1 arg1, T2 arg2);
...
public delegate void Action<T1, ..., T16>(T1 arg1, ..., T16arg16);

public delegate TResult Func<TResult>();
public delegate TResult Func<T, TResult>(Targ);
public delegate TResult Func<T1, T2, TResult>(T1 arg1, T2 arg2);
...
public delegate TResult Func<T1,..., T16, TResult>(T1 arg1, ..., T16 arg16);
```

## C#’s Syntactical Sugar for Delegates

Microsoft’s C# compiler offers programmers some syntax shortcuts when working with delegates.

### 1: No Need to Construct a Delegate Object

```c#
internal sealed class AClass {
    public static void CallbackWithoutNewingADelegateObject() {
        // Here, ThreadPool.QueueUserWorkItem method expects a reference to a Wait­ Callback delegate object
        ThreadPool.QueueUserWorkItem(SomeAsyncTask, 5);
    }
    private static void SomeAsyncTask(Object o) {
        Console.WriteLine(o);
    } 
}
```

### 2: No Need to Define a Callback Method (Lambda Expressions)

```c#
internal sealed class AClass {
    public static void CallbackWithoutNewingADelegateObject() {
        ThreadPool.QueueUserWorkItem( obj => Console.WriteLine(obj), 5); 
    }
}
```

### 3: No Need to Wrap Local Variables in a Class Manually to Pass Them to a Callback Method

```c#
internal sealed class AClass {
    public static void UsingLocalVariablesInTheCallbackCode(Int32 numToDo) {
        // Some local variables
        Int32[] squares = new Int32[numToDo];
        AutoResetEvent done = new AutoResetEvent(false);
      
        // Do a bunch of tasks on other threads
        for (Int32 n = 0; n < squares.Length; n++) {
            ThreadPool.QueueUserWorkItem(
                obj => {
                    Int32 num = (Int32) obj;
                    // This task would normally be more time consuming
                    squares[num] = num * num;
                    // If last task, let main thread continue running
                    if (Interlocked.Decrement(ref numToDo) == 0)
                        done.Set();
                }, n); 
        }

        // Wait for all the other threads to finish
        done.WaitOne();

        // Show the results
        for (Int32 n = 0; n < squares.Length; n++)
            Console.WriteLine("Index {0}, Square={1}", n, squares[n]);
    }
}
```

## Deledates and Reflection

In some rare cases, developer doesn’t know how many and what parameters the callback method requires at compile time. Fortunately, **System.Reflection.MethodInfo** offers a **CreateDelegate** method that allows you to create a delegate when you just don’t have all the necessary information about the delegate at compile time. Then you can call it by using Delegate’s **DynamicInvoke** method.
