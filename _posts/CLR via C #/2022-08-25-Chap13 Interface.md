---
layout:     post
title:      Chapter 13. Interfaces
subtitle:   
date:       2022-08-25
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

CLR does NOT support multiple inheritance, it offer this feature via **interface**.

## Class and Interface

In the CLR, a class is always derived from one and only one class (derived from *Object*). The base class provides **a set of method signatures and implementations for these methods**. The derived class can inherit all of the method signatures and implementations.

Interface just **give a set of methods signatures, and do not provide any implementation**. A class inherits an interface, and **MUST** explicitly provide implementations of the interface's methods.

## Defining an Interface

**An interface is a named set of method signatures.** \
It can define events, properties because these are just syntax shorthands that map to methods. \ 
cannot define any constructor methods, instance fields.

By convention, interface type names are prefixed with an uppercase **I**. \
CLR does support generic interface and generic methods in an interface.

## Inheriting an Interface

C# compiler requires that a method that implements an interface be marked as **public**. \
CLR requires that interface methods be marked as **virtual**.
+ If not explicitly mark the method as **virtual**, compiler marks the method as **virtual** and **sealed**.
+ If explicitly mark the method as **virtual**, compiler marks the method as **virtual** (and **unsealed**).

Here is an example that demonstrates this.

```c#
public static class Program {
    public static void Main() {
        // First Example
        Base b = new Base();
        // Calls Dispose by using b's type: "Base's Dispose"
        b.Dispose();
        // Calls Dispose by using b's object's type: "Base's Dispose"
        ((IDisposable)b).Dispose();

        // Second Example
        Derived d = new Derived();
        // Calls Dispose by using d's type: "Derived's Dispose"
        d.Dispose();
        // Calls Dispose by using d's object's type: "Derived's   Dispose"
        ((IDisposable)d).Dispose();

        // Third Example
        b = new Derived();
        // Calls Dispose by using b's type: "Base's Dispose"
        b.Dispose();
        // Calls Dispose by using b's object's type: "Derived'sDispose"
        ((IDisposable)b).Dispose();
    }
}

internal class Base : IDisposable {
    // This method is implicitly sealed and cannot be overridden
    public void Dispose() {
        Console.WriteLine("Base's Dispose");
    }
}

internal class Derived : Base, IDisposable {
   // This method cannot override Base's Dispose. 
   // 'new' is used to indicate that this method re­implements IDisposable's Dispose method
    new public void Dispose() {
        Console.WriteLine("Derived's Dispose");
        // NOTE: The next line shows how to call a base class's   implementation (if desired)
        // base.Dispose();
    }
}   
```

## About Calling Interface Methods

Using a variable of an interface type allows you to call methods defined by that interface. \
In addition, CLR allows you to call methods defined by Object because all classes inherit Object’s methods.

```c#
String s = "Jeffrey";

IComparable comparable = s;

IEnumerable enumerable = (IEnumerable) comparable;
```

The type of the variable indicates the action that I can perform on the object. For example, I can use **s** to call any members defined by **String** type, but can't use **comparable** to call public methods defined by **String**.

## Implicit and Explicit Interface Method Implementations (What’s Happening Behind the Scenes)

When a type in loaded into CLR, a method table is created and initialized for the type: 
+ one entry for every **new method** introduced by the type
+ entries for any virtual methods inherited by **base class**
+ entries for any virtual methods inherited by **interface**

If we have a simple type defined like this:
```c#
internal sealed class SimpleType : IDisposable {
    public void Dispose() { Console.WriteLine("Dispose"); }
}
```
the type’s method table contains entries for the following:
+ All the virtual instance methods defined by *Object*.
+ All the interface methods defined by IDisposable (only one method, Dispose).
+ The new method, **Dispose**, introduced by *SimpleType*.

Because the **SimpleType's Dispose** and **IDisposable’s Dispose** are identical (same parameter and return types), compiler matches a new method to an interface method. It emits metadata indicat- ing that both entries in SimpleType’s method table should refer to the same implementation.

```c#
public sealed class Program {
    public static void Main() {
        SimpleType st = new SimpleType();
        // This calls the public Dispose method implementation
        st.Dispose();
        // This calls IDisposable's Dispose method implementation
        IDisposable d = st;
        d.Dispose();
    } 
}

// output is as follows:
// Dispose
// Dispose
```

In the frist call, the **Dispose** method defined by **SimpleType** is called. In the second call, the **IDisposable** interface’s **Dispose** method is called. Because they are same implementation, the same code implementation will execute.

If I rewrite the **SimpleType** using *explicit interface method implementation*, the output will be different.
```c#
internal sealed class SimpleType : IDisposable {
    public void Dispose() { Console.WriteLine("public Dispose"); }
    void IDisposable.Dispose() { Console.WriteLine("IDisposable Dispose"); }
}

// output is as follows:
// public Dispose
// IDisposable Dispose
```

Here, **IDisposable.Dispose** is an explicit interface method implementation, it can't specify accessibility. \
When compiler generates metadata for the method, its accessibility is set to **private**, preventing any instance of the class from calling the interface method. \
**The only way to call the interface method is through a variable of the interface’s type.**

## Generic Interfaces

C#'s support of generic interfaces offers many great features for developers.

First, generic interfaces offer **compile-time type safety**.

Some non-generic interfaces define methods that have *Object* parameters or retuen types. When passing parameters, a reference to an instance of any type can be passed. But this is usually not desired.

```c#
private void SomeMethod1() {
    Int32 x = 1, y = 2;

    IComparable c = x;

    // CompareTo expects an Object; passing y (an Int32) is OK
    c.CompareTo(y);     // y is boxed here

    // CompareTo expects an Object; passing "2" (a String) compiles
    // but an ArgumentException is thrown at runtime
    c.CompareTo("2");
}
```

Obviously, it is preferable to have the interface method strongly typed.

```C#
private void SomeMethod2() {
   Int32 x = 1, y = 2;

   IComparable<Int32> c = x;

   // CompareTo expects an Int32; passing y (an Int32) is OK
   c.CompareTo(y);     // y is not boxed here

   // CompareTo expects an Int32; passing "2" (a String) results
   // in a compiler error indicating that String cannot be cast to an Int32
   c.CompareTo("2");   // Error
}
```

The second benefit is that much less boxing will occur. 

In *SomeMethod1*, non-generic *IComparable* interface’s CompareTo method expects an *Object*; passing y (an Int32 value type) causes the value in y to be boxed. \
In *SomeMethod2*, the generic *IComparable\<in T>* interface’s CompareTo method expects an Int32; passing y causes it to be passed by value, without boxed.

The third benefit is that a class can implement the same interface multiple times as long as different type parameters are used.

```c#
public sealed class Number: IComparable<Int32>, IComparable<String> {
    private Int32 m_val = 5;

    // This method implements IComparable<Int32>'s CompareTo
    public Int32 CompareTo(Int32 n) {
       return m_val.CompareTo(n);
    }

    // This method implements IComparable<String>'s CompareTo
    public Int32 CompareTo(String s) {
       return m_val.CompareTo(Int32.Parse(s));
    }
}
public static class Program {
   public static void Main() {
      Number n = new Number();

      // Here, I compare the value in n with an Int32 (5)
      IComparable<Int32> cInt32 = n;
      Int32 result = cInt32.CompareTo(5);

      // Here, I compare the value in n with a String ("5")
      IComparable<String> cString = n;
      result = cString.CompareTo("5");
    } 
}
```

An interface’s generic type parameters can also be marked as *contra-variant* and *covariant*, As described in previous section.

## Generics and Interface Constraints

The benefits of constraining generic type parameters to interfaces.

First, you can constrain a single generic type parameter to multiple interfaces.

When you define a method’s parameters, each parameter’s type must be any class type that implements the interface.

```c#
public static class SomeType {
    private static void Test() {
        Int32 x = 5;
        Guid g = new Guid();

        // This call to M compiles fine because
        // Int32 implements IComparable AND IConvertible
        M(x);

        // This call to M causes a compiler error because
        // Guid implements IComparable but it does not implement  IConvertible
        M(g);
    }

    // M's type parameter, T, is constrained to work only with types that
    // implement both the IComparable AND IConvertible interfaces
    private static Int32 M<T>(T t) where T : IComparable, IConvertible {
        ... 
    }
}
```

The second benefit of interface constraints is **reduced boxing** when passing instances of value types.

## Implementing Multiple Interfaces That Have the Same Method Name and Signature

Perhaps, sometimes we define a type that implements multiple interface that define methods with the same name and signature. For example,

```c#
public interface IWindow {
    Object GetMenu();
}

public interface IRestaurant {
    Object GetMenu();
}
```

```c#
// This type is derived from System.Object and
// implements the IWindow and IRestaurant interfaces.
public sealed class MarioPizzeria : IWindow, IRestaurant {
   // This is the implementation for IWindow's GetMenu method.
   Object IWindow.GetMenu() { ... }

   // This is the implementation for IRestaurant's GetMenu method.
   Object IRestaurant.GetMenu() { ... }

   // This (optional method) is a GetMenu method that has nothing
   // to do with an interface.
   public Object GetMenu() { ... }
}
```

When using a **MarioPizzeria** object, we must cast to the specific interface to call the desired method.

```c#
MarioPizzeria mp = new MarioPizzeria();

// This line calls MarioPizzeria's public GetMenu method
mp.GetMenu();

// These lines call MarioPizzeria's IWindow.GetMenu method
IWindow window = mp;
window.GetMenu();

// These lines call MarioPizzeria's IRestaurant.GetMenu method
IRestaurant restaurant = mp;
restaurant.GetMenu();
```

## Explicit Interface Method Implementations

refer to : https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/interfaces/explicit-interface-implementation

## Design: Base Class or Interface?

About the question : Should I design a base type or an interface? Here are some guidelines that might help you.

+ **IS-A vs. CAN_DO** : Base class inply a IS-A relationship. Interfaces imply a CAN-DO relationship.

+ Ease of use : It’s easier for a developer to define a new type derived from a base type than to implement all of the methods of an interface. 

+ Consistent implementation : No matter how well an interface is designed initially, it's impossible that everyone will implement the contract 100% correctly. However, with a base class with a good initial implementation, we can use a well-initialized base type, and update its implementation on demand.

+ Versioning : If adding a method into a base class, the derived type inherits the new method and can use it derectly. If adding a new member to an interface, those classes that implement the interface have to change source code and recompile.

