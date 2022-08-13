---
layout:     post
title:      Chapter 7. Constants and Fields
subtitle:   
date:       2022-08-08
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

# Constants

Constant value never changes, so it is always considered to be **static member**, not instance member.

When code refers to a constant, compiler looks up the constant in metadata table of the assmebly, extract the constant’s value, and embed the value in the emitted Intermediate Language (IL) code. \
**Because a constant’s value is embedded directly in code, constants don’t require any memory to be allocated at run time.**

**Attention:** If you want to refer a constant defined in another DLL assembly, when the constant changes and you only rebuild the DLL assembly, your application won't pick up new value of constant at runtime unless you re-compile it. \
In this time, you should use **readonly** fields.

# Fields

CLR supports *readonly* fields and *read/write* fields. *readonly* fields can **be written to only within a constructor method**.

Note that *reflection* can be used to modify a *readonly* field.

C# treats initializing a field inline as shorthand syntax for initializing the field in a constructor. 
```c#
public readonly String Pathname = "Untitled";
```

**Attention:** When a field of reference type is marked as *readonly*, it denotes that the reference is immutable, not the object that the field refers to. For example,
```c#
public sealed class AType {
    // InvalidChars must always refer to the same array object
    public static readonly Char[] InvalidChars = new Char[] { 'A', 'B', 'C' };
}

public sealed class AnotherType {
    public static void M() {
    // The lines below are legal, compile, and successfully
    // change the characters in the InvalidChars array
    AType.InvalidChars[0] = 'X';
    AType.InvalidChars[1] = 'Y';
    AType.InvalidChars[2] = 'Z';

    // The line below is illegal and will not compile because
    // what InvalidChars refers to cannot be changed
    AType.InvalidChars = new Char[] { 'X', 'Y', 'Z' };
    }
}
```