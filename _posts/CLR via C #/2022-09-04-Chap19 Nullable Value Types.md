---
layout:     post
title:      Chapter 19. Nullable Value Types
subtitle:   
date:       2022-09-04
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

A value type can never be **null**; it always contains the value type's value.

Unfortunately, there are some scenarios in which this is a problem. For example, a database's Int32 column allows **null** even if the C# FCL value types have no **null** value. 

To improve this situation, Microsoft introduces nullable value types **Nullable\<T>**. 

## C#' support for Nullable Value Types

C# offers a cleaner syntax to define nullable value types, i.e., Int32? is a synonym notation for Nullable\<Int32>.

```c#
Int32? x = 5;
Int32? y = null;
```

C# also allows you to apply operators to nullable instances.

```c#
Int32? a = 5;
Int32? b = null;

a++;    // a = 6
b = -­b; // b = null

a = a + 3; // a = 9
b = b * 3; // b = null;

if (a == null) { /* no  */ } else { /* yes */ }
if (b == null) { /* yes */ } else { /* no  */ }
if (a != b)    { /* yes */ } else { /* no  */ }

if (a < b)     { /* no  */ } else { /* yes */ }
```

you can also define own value types that overload the various operators.

```c#
internal struct Point {
    private Int32 m_x, m_y;
    public Point(Int32 x, Int32 y) { m_x = x; m_y = y; }

    public static Boolean operator==(Point p1, Point p2) {
      return (p1.m_x == p2.m_x) && (p1.m_y == p2.m_y);
    }

    public static Boolean operator!=(Point p1, Point p2) {
        return !(p1 == p2);
    } 
}
```

At this point, you can use nullable instance of Point and compiler will invoke your overloaded operators.

```c#
internal static class Program {
    public static void Main() {
        Point? p1 = new Point(1, 1);
        Point? p2 = new Point(2, 2);
        Console.WriteLine("Are points equal? " + (p1 == p2).ToString());
        Console.WriteLine("Are points not equal? " + (p1 != p2).ToString());
    }
}
```

C# has an operator **??**. If the operand on the left is not null, the operand’s value is returned. If the operand on the left is null, the value of the right operand is returned.

```c#
Int32? b = null;
// The following line is equivalent to:
// x = (b.HasValue) ? b.Value : 123
Int32 x = b ?? 123;
Console.WriteLine(x);  // "123"
```

## CLR has Special Support for Nullable Value Types

The CLR has built-in support for nullable value types.

### Boxing Nullable Value Types

Imagine you have a Nullable\<Int32> variable, and this variable is passed to a method that expects an **Object**. We need a reference to the boxed **Nullable\<Int32>**.

When CLR is boxing a **Nullable\<T>** variable, if it is null, CLR just returns **null**. If the nullable instance is not null, CLR takes the value and boxes it.

```c#
Int32? n = null;
Object o = n;  // o is null
Console.WriteLine("o is null={0}", o == null);  // "True"

n = 5;
o = n;   // o refers to a boxed Int32
Console.WriteLine("o's type={0}", o.GetType()); // "System.Int32"
```

### Unboxing Nullable Value Types

CLR allows a boxed value type **T** to be unboxed into a **T** or a **Nullable\<T>**. If the reference to the boxed value type is null, and you are unboxing it to a **Nullable\<T>**, the CLR sets **Nullable\<T>**’s value to null.

```c#
Object o = 5;

Int32? a = (Int32?) o;  // a = 5
Int32  b = (Int32)  o;  // b = 5

o = null;

a = (Int32?) o;       // a = null
b = (Int32)  o;       // NullReferenceException
```

### Calling GetType via a Nullable Value Type

When calling **GetType** on a **Nullable\<T>** object, the CLR actually returns the type **T** instead of the type **Nullable\<T>**.

```c#
Int32? x = 5;
// return "System.Int32"; not "System.Nullable<Int32>"
Console.WriteLine(x.GetType());
```

### Calling Interface Methods via a Nullable Value Type

In the following code, we cast a **Nullable\<Int32>**, to **IComparable\<Int32>**. However, the **Nullable\<T>** type does not implement the **IComparable\<Int32>** interface as **Int32** does. CLR’s verifier considers this code verifiable.

```c#
Int32? n = 5;
Int32 result = ((IComparable) n).CompareTo(5);  // Compiles & runs OK
Console.WriteLine(result);                      // 0
```

If CLE doesn't provide this feature, perhaps we need to write as follows.

```c#
Int32 result = ((IComparable) (Int32) n).CompareTo(5);
```