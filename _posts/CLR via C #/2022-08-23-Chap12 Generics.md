---
layout:     post
title:      Chapter 12. Generics
subtitle:   
date:       2022-08-23
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

One of the big benefits from object-oriented programming is code reuse, the ability to derive a class. The derived class can inherits all of a base class, override virtual methods, and add some new methods to customize behaviors.

**Generics** is another benefit provided by CLR and programming language, to provide one more form of code reuse: algorithm reuse. For example, generic list **List\<T>**.

Generics provide the following big benefits to developers:
+ **Source code protection** : The developer using a generic algorithm doesn’t need to have access to the algorithm’s source code.
+ **Type safety** : When using generic algorithm with specific type, compiler and CLR will ensure type compatibility.
+ **Cleaner code** : Because the compiler enforces type safety, fewer casts are required, code is easier to write and maintain.
+ **Better performance** : If no generics, we will have to work with *Object* data type, and CLR has to box and unbox, which causes additional memory allocateions and performance hurt.

Framework Class Library (FCL) provides some generic collections. Most of these classes can be found in the **System.Collections.Generic** namespace.

## Generic Infrastructure

CLR create type object for each type in an application. A type with generic type parameters is still considered a type. 

However, a type with generic type parameters is called an **open type**, and CLR does not allow any instance of an open type. \
If we passed actual type for type arguments, the type is called a **closed type**, and CLR allow instances of a closed type.

CLR has some optimizations to reduce code explosion (generate native code for every method/type combination). 
+ If a method is called for a particular type argument, and later, the method is called again using the same type argument, CLR will compile this method/type combination just once.
+ CLR considers all reference type arguments to be identical, because all reference type arguments or variables are really just pointers. 
+ But if any type argument is a value type, CLR must produce native code specifically for that value type.

## Generic Interfaces

A reference or value type can implement generic interface by specifying type arguments, or by leaving type arguments unspecified.

Here is the definition of a generic interface of **IEnumerator**.

```C#
public interface IEnumerator<T> : IDisposable, IEnumerator {
    T Current { get; }
}
```

### Case 1: implement generic interface and specify type arguments

```c#
internal sealed class Triangle : IEnumerator<Point> {
    
    private Point[] m_vertices;

    public Point Current { get { ... } }
    ... 
}
```

### Case 2: implements the same generic interface but with the type arguments left unspecified.

```c#
internal sealed class ArrayEnumerator<T> : IEnumerator<T> {

    private T[] m_array;

    public T Current { get { ... } }
    ... 
}
```

## Generic Delegates

A delegate is really a class definition with four methods: a constructor, an **Invoke** method, a **BeginInvoke** method, and an **EndInvoke** method.

When you define a delegate type that specifies type arguments, the compiler defines the delegate class' methods with type argument.

For example, if you define a generic delegate like this:
```c#
public delegate TReturn CallMe<TReturn, TKey, TValue>(TKey key, TValue value);
```

the compiler turns that into a class that logically looks like the following.

```c#
public sealed class CallMe<TReturn, TKey, TValue> : MulticastDelegate {
    public CallMe(Object object, IntPtr method);

    public virtual TReturn Invoke(TKey key, TValue value);

    public virtual IAsyncResult BeginInvoke(TKey key, TValue value,
       AsyncCallback callback, Object object);

    public virtual TReturn EndInvoke(IAsyncResult result);
}
```

## Contra-variant and Covariant Generic Type Arguments

This feature allows you to cast a variable of a generic delegate type to the same delegate type where the generic parameter types differ.

+ **Invariant** (不变量) generic type parameter cannot be changed.
+ **Contra-variant** (逆变量) generic type parameter can change from a class to a derived class. C# defines contra-variant generic type argument with **in** keyword.
+ **Covariant** (协变量) generic type argument can change from a class to one of base classes. C3 defines it with **out** keyword.

For example, for the following delegate type definition,
```c#
public delegate TResult Func<in T, out TResult>(T arg);
```
if I have a variable declared as follows,
```c#
Func<Object, ArgumentException> fn1 = null;
```
I can case into another **Func** type, where the feneric type argument are different.
```c#
Func<String, Exception>fn2 = fn1; // No explicit cast is required here
Exception e = fn2("");
```

Because *String* is the derived class from *Object*, and *Exception* is the base class of *ArgumentException*.

## Generic Methods

When you define a generic class or interface, any methods defined in these types can refer to the type argument. Also, CLR supports the ability for a method to specify its owen type parameters. For example,

```c#
internal sealed class GenericType<T> {
    private T m_value;

    public GenericType(T value) { m_value = value; }

    public TOutput Converter<TOutput>() {
       TOutput result = (TOutput) Convert.ChangeType(m_value, typeof(TOutput));
       return result;
    } 
}
```

## Constraints

+ Primary Constraints : A type parameter can specify zero or one primary constraint, which means that the specified type argument will either be of the same type or of a type derived from the constraint type.
```c#
internal sealed class PrimaryConstraintOfStream<T> where T : Stream {
    public void M(T stream) {
        stream.Close();// OK
    }
}
```

+ Secondary Constraints : A type parameter can specify zero or more secondary constraints, where a secondary constraint represents an interface type.
```c#
private static List<TBase> ConvertIList<T, TBase>(IList<T> list)
   where T : TBase {
    List<TBase> baseList = new List<TBase>(list.Count);
    for (Int32 index = 0; index < list.Count; index++) {
        baseList.Add(list[index]);
    }
    return baseList;
}
```

+ Constructor Constraints : A type parameter can specify zero or one constructor constraint. When specifying a constructor constraint, you are promising the compiler that a specified type argument will be a non-abstract type that implements a public, parameterless constructor.
```c#
 internal sealed class ConstructorConstraint<T> where T : new() {
    public static T Factory() {
       // Allowed because all value types implicitly
       // have a public, parameterless constructor and because
       // the constraint requires that any specified reference
       // type also have a public, parameterless constructor
       return new T();
} }
```
