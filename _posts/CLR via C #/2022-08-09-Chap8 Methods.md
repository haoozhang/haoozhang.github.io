---
layout:     post
title:      Chapter 8. Methods
subtitle:   
date:       2022-08-09
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

This chapter focuses on the various kinds of methods, including instance constructor and type constructor, operator overload methods and type conversion, extension methods that allow you to add your own instance methods to existing types, and partial methods that spread a type's implementation into multiple parts.

## Instance Constructor and Reference Types

Instance constructors are specific methods that **initialize an instance of a type** to be initial state, are always called **.ctor** in method definition metadata table.

When creating an instance of a reference type, firstly **allocate memory** for instance's data fields, and the object's overhead fields (type object pointer and sync block index). \
Then the memory is always **zeroed**, to gurantee all fields have a value of 0 or null even if they are not explicitly initialized. \
And then the **instance constructor is called** to set the initial state of object.

Instance constructors are **never inherited**, so can not apply modifiers: virtual, new, override, sealed, or abstract.

C# compiler provides a **default parameterless constructor** if the class does not define any constructors, except static class.

Instance constructor will call the constructor of base class automatically, until called System.Object's constructor.

In a few cases, we can create instance without instance constructor. For example, calling **Object’s MemberwiseClone** method allocates memory, initializes object’s overhead fields, and then copies the source object’s bytes to the new object. Deserializing an object with the runtime serializer allocates memory for object by using **GetUninitializedObject** method.

For those convenient sytax that initialize the instance fields inline, compiler will **translate inline initialization to code in constructor** and perform initialization. For following code, every constructor initialize *m_x* and *m_s*, then perform actual constructor method. 

```c#
internal sealed class SomeType {
   private Int32  m_x = 5;
   private String m_s = "Hi there";
   private Byte   m_b;
   // Here are some constructors.
   public SomeType()         { ... }
   public SomeType(Int32 x)  { ... }
}
```

## Instance Constructor and Structures (Value Types)

C# compiler doesn't emit default parameterless constructors for value types. The fields of the value types are initialized to 0/null.

CLR does allow you to define constructors on value types. But it is executed **only when explicitly called**. 
```c#
internal struct Point {
    public Int32 m_x, m_y;
    public Point() {
        m_x = m_y = 5;
    }
}
internal sealed class Rectangle {
    public Point m_topLeft, m_bottomRight;
    public Rectangle() {
    }
}
```
In this code, due to complier will never call value type's default construtor, even if it offers a parameterless constructor. The *m_x* and *m_y* are still 0.

C# doesn’t allow a value type to define a parameterless constructor.

## Type Constructor

also known as *static constructor, class constructor, type intializer*.

Type construcotr are also used to set initial state of a type. Types **don't have type constructor default**. If has, it can **have no more than one**. And, type constructors **never have parameters**.

How to define the type constructor for reference type and value type.
```c#
internal sealed class SomeRefType {
    static SomeRefType() {
        // This executes the first time a SomeRefType is accessed.
    }
}
internal struct SomeValType {
    // C# does allow value types to define parameterless type constructors.
    static SomeValType() {
        // This executes the first time a SomeValType is accessed.
    }
}
```

For calling type constructor, 
1. when JIT (just-in-time) compiler is compiling a method, it sees what types are referenced. If any type defines type constructor, it will check if it has already been executed for this AppDomain.
2. If never executed, JIT compiler will emit the call to type constructor into native code. If already executed, JIT will not emit the call.
3. When thread executes the native code of the method, it may that multiple threads execute the same methods concurrently. CLR ensure that a type’s constructor executes **only once per AppDomain**.
4. Calling thread acquires a mutually exclusive thread synchronization lock when calling type constructor. Other threads will be blocked until the calling thread leaves. Then they will check if it has been executed. If yes, they will simply return.

Because that type constructor executes only once per App- Domain and is thread-safe, it's a great palce to **initialize any singleton objects**.

Type constructor is to **initialize static fields**. C# also offers the convenient syntax inline. Similarly, compiler automatically generates a type constructor to perform intialization.

## Operator Overload Methods

String, Decimal and DatetTime types overload the equlity (==) and inequality (!=) operators. 

CLR doesn’t know anything about operator overloading because it doesn’t even know what an operator is. This is the feature provided by programming language.

Operator overload methods must be **public and static** methods. And, at least one parameter must be same as the type that method is defined within, which can help C# to search possible operator method quickly.
```c#
public sealed class Complex {
   public static Complex operator+(Complex c1, Complex c2) { ... }
}
```

## Conversion Operator Methods

Occasionally, we intend to convert an object from one type to another different type. If the source type and target type are all primitive types, compiler knows how to convert the object. \
If the source type or target type is not a primitive, the compiler will let CLR to perform the conversion (cast). CLR just checks if the source type is same as target type, or derived from target type. \
Sometimes, we want to convert source type to a completely different type, we can define public constructor that take the parameter: an instance of the source type. Also, we can define public instance method (ToXXX) to convert this type to XXX type.

some programming languages (such as C#) also offer conversion operator overloading, which mandates that must be **public and static**. In addition, C# requires that either the parameter or the return type must be the same as the type that the conversion method is defined within.
```c#
// Implicitly constructs and returns a Rational from a Single
public static implicit operator Rational(Single num) {
    return new Rational(num);
}
// Explicitly returns an Int32 from a Rational
public static explicit operator Int32(Rational r) {
    return r.ToInt32();
}
```

Recommend to learn about *System.Decimal* type to further understand operator overload methods and conversion operator methods.

## Extension Methods

We use an example to understand C#'s extension methods. *StringBuilder* class is the preferred way of manipulating a string because it is mutable. So, let’s say that we would like to define some missing methods to operate on a StringBuilder, e.g., *IndexOf* method as follows.
```c#
public static class StringBuilderExtensions {
    public static Int32 IndexOf(StringBuilder sb, Char value) {
        for (Int32 index = 0; index < sb.Length; index++) {
            if (sb[index] == value) return index;
        }
        return ­-1; 
    }
}
```

Then, we can use this method via following code.
```c#
StringBuilder sb = new StringBuilder("Hello. My name is Jeff.");
Int32 index = StringBuilderExtensions.IndexOf(sb.Replace('.', '!'), '!');
```
This code can work fine, but it's not ideal from a programmer'd perspective. First, we must know the existance of *StringBuilderExtensions* class and *IndexOf* method. Second, the code doesn't reflect the order of operations. We want to *Replace* first, then call *IndexOf*, but *IndexOf* appears first and *Replace* appears second.

Now, we need C#’s extension methods, which allows to define a static method that we can invoke using instance method syntax.
```c#
public static class StringBuilderExtensions {
    public static Int32 IndexOf(this StringBuilder sb, Char value) {
        for (Int32 index = 0; index < sb.Length; index++) {
            if (sb[index] == value) return index;
        }
        return ­-1; 
    }
}
```
We can just call it using following sytax.
```c#
Int32 index = sb.IndexOf('X');
```

We can also **define extension methods for interface types**, as the following code shows.
```c#
public static void ShowItems<T>(this IEnumerable<T> collection) {
    foreach (var item in collection)
        Console.WriteLine(item);
}
```

The extension methos can be invoked using any expression that implements the IEnumerable\<T\> interface.
```c#
public static void Main() {
    // Shows each Char on a separate line in the console
    "Grant".ShowItems();
    // Shows each String on a separate line in the console
    new[] { "Jeff", "Kristin" }.ShowItems();
    // Shows each Int32 value on a separate line in the console
    new List<Int32>() { 1, 2, 3 }.ShowItems();
}
```

### Rules and Guidelines

+ C# supports **extension methods only**, not extension properties / events / operators.
+ Extension methods must be declared in **non-generic, static classes**, must have **at least one parameter, and only the first parameter can be marked with *this* keyword**.
+ The static class that define extension methods **must have file scope**. In other words, you can't define extension methods in the static class nested within another class.

## Partial Methods

If we customize methods in one class, Normally, customization would be done by invoking virtual methods. base class have to contain definitions for virtual methods, and this methods would be implemented is to do nothing and simply return. Then, we define own class, derive it from the base class, and override any virtual methods.
```c#
internal class Base {
    private String m_name;

    // Called before changing the m_name field
    protected virtual void OnNameChanging(String value) {
    }
    
    public String Name {
        get { return m_name; }
        set {
            OnNameChanging(value.ToUpper());  // Inform class of potential change
            m_name = value;                   // Change the field
        }
    }   
}

internal class Derived : Base {
    protected override void OnNameChanging(string value) {
        if (String.IsNullOrEmpty(value))
            throw new ArgumentNullException("value");
    }
}
```

There are two problems for this way. First, the base class must be a mot-sealed class. Second, there are performance problem here. A type is defined just to override a method.

So, C#'s partial methods allows to overrides behavior while fixing above problems.
```c#
internal sealed partial class Base {
    private String m_name;
   
    // This defining­partial­method­declaration is called before changing the m_name field
    partial void OnNameChanging(String value);
   
    public String Name {
        get { return m_name; }
        set {
            OnNameChanging(value.ToUpper());  // Inform class of potential change
            m_name = value;                   // Change the field
        }
    } 
}

internal sealed partial class Base {
    // This implementing­partial­method­declaration is called before m_name is changed
    partial void OnNameChanging(String value) {
        if (String.IsNullOrEmpty(value))
            throw new ArgumentNullException("value");
    } 
}
```

+ The class is now **sealed**, In fact, it **could be static, even a value type**.
+ The partial method in original class is marked with *partial* keyword and has no body.
+ The partial method in new class is also marked with *partial* keyword and has a body.

**If there is no implementing partial method declaration, the compiler will not generate any metadata representing the partial method.** So, there is big improvement with partial methods. 

### Rules and Guidelines

+ Partial methods can only be declared within a partial class or struct.
+ Partial methods must have a **return type of void**, and cannot have any parameters marked with *out* modifier.
+ partial method declaration and implementation must have identical signature.
+ Partial methods are **always considered to be private**. 