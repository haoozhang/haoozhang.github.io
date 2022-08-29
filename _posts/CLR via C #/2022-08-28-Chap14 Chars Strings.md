---
layout:     post
title:      Chapter 14. Charsm Strings, and Working with Text
subtitle:   
date:       2022-08-28
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

+ System.Char 
+ System.Stirng 
+ System.Text.StringBuilder 
+  format objects into strings
+ System.Security.SecureString

## Characters

char are always represented in 16-bit Unicode code values (2 bytes).

static **GetUnicodeCategory** method return the **UnicodeCategory** enumerated type of a char, which indicates whether the character is a control character, a lowercase letter, an uppercase letter, a math symbol, or another characters.

## System.String

A String represents an **immutable** sequence of characters. \
String type derives from Object, is **reference type**, so always **lives in the heap**.

In C#, you can’t use the *new* operator to construct a String object from a literal string. Instead, you must use the following simplified syntax.
```c#
String s = new String("Hi there.");  // <­­- Error

String s = "Hi there.";  // <- Correct
```

When using '+' operator to concatenate several strings, if all strings are literal strings, compiler concat the at compile time. \
No literal strings cause the concatenation to be performed at run time.

Using **'@'** character to specify directory path or regular expressions.
```
String file = @"C:\Windows\System32\Notepad.exe";
```

A String object is **immutable**. For the following code, the *ToUpperInvariant* and *Substring* operations return new strings, and GC will reclaim their memory.
```c#
if (s.ToUpperInvariant().Substring(10, 21).EndsWith("EXE")) {
    ...
}
```
Having immutable strings also means that there are **no thread synchronization issues**. 

String type is **sealed**, and can't be used as base class.

### Comparing Strings

It is highly recommended that you call one of these methods defined by the String class.

```
Boolean Equals(String value, StringComparison comparisonType)
static Boolean Equals(String a, String b, StringComparison comparisonType)
static Int32 Compare(...)
```

When using strings for internal programmatic purposes, should always use **StringComparison.Ordinal** or **StringComparison.OrdinalIgnoreCase**. \
When comparing strings in a linguistically correct manner (usually for display to an end user), you should use **StringComparison.CurrentCulture** or **String­ Comparison.CurrentCultureIgnoreCase**.

## Constructing a String Efficiently

Because the String type represents an immutable string, the FCL provides another type, System. Text.StringBuilder, which allows you to perform dynamic operations efficiently with strings.

```
StringBuilder sb = new StringBuilder();
```

+ Maximum capacity : The default is Int32.MaxValue. You might specify a smaller maximum capacity. Once constructed, a StringBuilder’s maximum capacity value can’t be changed.
+ Capacity : An Int32 value indicating the size of the character array being maintained. The default is 16. \
When appending characters to the character array, the StringBuilder detects if the array is grow beyond the array’s capacity. If it is, the StringBuilder automatically **doubles the capacity field, allocates a new array (the size of the new capacity), and copies the characters from the original array into the new array**. The original array will be garbage collected.
+ Character array : An array of Char structures that maintains the set of characters. 

Notice that the String and StringBuilder classes don’t have full method parity; that is, String has *ToLower, ToUpper, EndsWith, Trim,* and so on. StringBuilder doesn't offer those. StringBuild offers a Replace method, but String doesn't offer it. For example, to build up a string, convert to uppercase, and then insert a string requires code like the following.

```c#
StringBuilder sb = new StringBuilder();
sb.AppendFormat("{0} {1}", "Jeffrey", "Richter").Replace(" ", "­");
String s = sb.ToString().ToUpper();
sb.Length = 0;
sb.Append(s).Insert(8, "Marc­");
s = sb.ToString();
Console.WriteLine(s);
```

## ToString and Parse

ToString : Obtain a String representation of an Object \
You can customize the **ToString** method defined by Object.

Parse: Parse a String to obtain an Object \
Any type that can parse a string offers a public, static method called **Parse**, which takes a String and returns an instance of the type

## Encodings: Converting Between Characters and Bytes

Here’s an example that encodes and decodes characters by using **UTF-8**.
```c#
using System;
using System.Text;

public static class Program {
    public static void Main() {
        // This is the string we're going to encode.
        String s = "Hi there.";
        // Obtain an Encoding ­derived object that knows how
        // to encode/decode using UTF8
        Encoding encodingUTF8 = Encoding.UTF8;
        // Encode a string into an array of bytes.
        Byte[] encodedBytes = encodingUTF8.GetBytes(s);
        // Show the encoded byte values.
        Console.WriteLine("Encoded bytes: " +
           BitConverter.ToString(encodedBytes));
        // Decode the byte array back to a string.
        String decodedString = encodingUTF8.GetString(encodedBytes);
        // Show the decoded string.
        Console.WriteLine("Decoded string: " + decodedString);
    }
}
```

The UTF-16 and UTF-8 encodings are quite popular. It is also quite popular to encode a sequence of bytes to **a base-64 string**.

```c#
using System;

public static class Program {
    public static void Main() {
        // Get a set of 10 randomly generated bytes
        Byte[] bytes = new Byte[10];
        new Random().NextBytes(bytes);
        // Display the bytes
        Console.WriteLine(BitConverter.ToString(bytes));
        // Decode the bytes into a base­64 string and show the string
        String s = Convert.ToBase64String(bytes);
        Console.WriteLine(s);
        // Encode the base­64 string back to bytes and show the bytes
        bytes = Convert.FromBase64String(s);
        Console.WriteLine(BitConverter.ToString(bytes));
    } 
}
```

## Secure Strings

Microsoft added a more secure string class to the FCL: System.Security.**SecureString**. 

When constructing a SecureString object, it allocates a block of unmanaged memory that contains an array of characters. \
**Unmanaged memory** is used so that the garbage collector isn’t aware of it.

You can AppendChar, InsertAt, RemoveAt, and SetAt for a secure string. \
Whenever you call these methods, internally, the method decrypts characters, performs operation in place, and then re-encrypts the characters.

The SecureString class implements the **IDisposable** interface to to easily destroy the string’s secured contents. \
You simply call SecureString’s **Dispose** method, or use a Secure­ String instance in a **using** construct, when you on longer need the sensitive string information.

Don't recommend that you use the SecureString class for new development, because it just makes the window getting the plain text shorter, but it doesn't fully prevent it as .NET still has to convert the string to a plain text. \
For more information, see [SecureString shouldn't be used](https://github.com/dotnet/platform-compat/blob/master/docs/DE0001.md) on GitHub.
