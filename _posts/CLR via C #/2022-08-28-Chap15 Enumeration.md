---
layout:     post
title:      Chapter 15. Enumerated Types and Bit Flags
subtitle:   
date:       2022-08-29
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

## Enumerated Types

In .NET framework, enumerated types are treated as **first-class citizens** in the type system, which is different from other environments (such as unmanaged c++).

enumerated type is derived directly from **System.Enum**, which is derived from **System.ValueType**, which in turn is derived from **System.Object**. So enumerated types are **value types**. \
However, unlike other value types, an enumerated type can’t define any methods, properties, or events. But you can use *extension method* feature to add methods.

When an enumerated type is compiled, the C# compiler turns each symbol into a **constant** field of the type.


```c#
// These two methods return a array thar contains each element.
public static Array GetValues(Type enumType);      // Defined in System.Enum
public        Array GetEnumValues();               // Defined in System.Type
```
However, they always return an Array, which then I have to cast to the specific array type. So I always define my own generic method.

```c#
public static TEnum[] GetEnumValues<TEnum>() where TEnum : struct {
    return (TEnum[])Enum.GetValues(typeof(TEnum));
}
```

```c#
// These two methods convert a symbol to an instance of an enumerated type
public static Object Parse(Type enumType, String value);
public static Boolean TryParse<TEnum>(String value, out TEnum result) where TEnum: struct;
```

```c#
// These two methods determine whether a numeric value is legal for an enumerated type
public static Boolean IsDefined(Type enumType, Object value); // Defined in System.Enum
public        Boolean IsEnumDefined(Object value);                // Defined in System.Type
```

Enumerated types are always used in conjunction with some other type. Typically, they’re used for the type’s method parameters or return type, properties, and fields. \
**You should define your enumerated type at the same level as the class.**

## Bit Flags



## Adding Methods to Enumerated Types


