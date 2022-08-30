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

Enumerated type is derived directly from **System.Enum**, which is derived from **System.ValueType**, which in turn is derived from **System.Object**. So enumerated types are **value types**. \
However, unlike other value types, an enumerated type **can’t define any methods, properties, or events**. But you can use *extension method* feature to add methods.

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

When you call **System.IO.File** type’s Get­ Attributes method, it returns an instance of a **FileAttributes** type. \
**FileAttributes** type is an instance of an Int32-based enumerated type, in which each bit reflects a single attribute of the file.

```c#
[Flags, Serializable]
public enum FileAttributes {
    ReadOnly = 0x00001,
    Hidden = 0x00002,
    System = 0x00004,
    Directory = 0x00010,
    Archive = 0x00020,
    Device = 0x04000,
    Normal = 0x08000,
    ...
}
```

To determine whether a file is hidden, you would execute code like the following.

```c#
String file = Assembly.GetEntryAssembly().Location;
FileAttributes attributes = File.GetAttributes(file);
Console.WriteLine("Is {0} hidden? {1}", file, (attributes & FileAttributes.Hidden) != 0);
```

## Adding Methods to Enumerated Types

Use C#’s exten- sion method feature to simulate adding methods to an enumer- ated type.

```c#
internal static class FileAttributesExtensionMethods {
    public static FileAttributes Set(this FileAttributes flags, FileAttributes setFlags) {
            return flags | setFlags;
    }

    public static FileAttributes Clear(this FileAttributes flags,
      FileAttributes clearFlags) {
      return flags & ~clearFlags;
    }
}

// here is some code that call these methods.
FileAttributes fa = FileAttributes.System;
fa = fa.Set(FileAttributes.ReadOnly);
fa = fa.Clear(FileAttributes.System);
fa.ForEach(f => Console.WriteLine(f));
```

