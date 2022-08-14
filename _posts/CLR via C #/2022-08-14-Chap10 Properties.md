---
layout:     post
title:      Chapter 10. Properties
subtitle:   
date:       2022-08-14
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

# Parameterless Properties

Many types define state information that can be retrieved or altered. Frequently, this state information is implemented as **fields** of the type. For example,
```c#
public sealed class Employee {
    public String Name;
    public Int32  Age;
}
```
Then we can easily get or set any of this state information.
```c#
Employee e = new Employee();
e.Name = "Jeffrey Richter";
e.Age  = 48;
Console.WriteLine(e.Name);
```

However, I would argue that the preceding code should never be implemented as shown, because of **data encapsulation**. Data encapsulation means that your type’s fields should never be publicly exposed. A developer could easily corrupt an Employee object with code like the following.
```c#
e.Age = ­5;
```

There are additional reasons for encapsulating access to a type’s data field. For example, you might want access to a field to execute some side effect, cache some value, or lazily create some internal object.

So, when designing a type, I strongly suggest that all of fields be private. Then, expose methods to allow to get or set state information. For example,
```c#
public sealed class Employee {
    private String m_Name;    // Field is now private
    private Int32  m_Age;     // Field is now private
    public String GetName() {
        return(m_Name);
    }
    public void SetName(String value) {
        m_Name = value;
    }
    public Int32 GetAge() {
        return(m_Age);
    }
    public void SetAge(Int32 value) {
        if (value < 0)
            throw new ArgumentOutOfRangeException("value", value.ToString(),
            "The value must be greater than or equal to 0");
        m_Age = value;
    }
}
```

Although we can get the enormous benefit from above code, encapsulating the data as shown earlier has two disadvantages. First, you have to write more code to implement additional methods. Second, users must call methods rather than simply refer to a single field.

CLR offer a mechanism called **properties**. You can think of **properties as smart fields: fields with additional logic**. For example,

```c#
public sealed class Employee {
    private String m_Name;
    private Int32  m_Age;
    public String Name {
        get { return(m_Name); }
        set { m_Name = value; } // The 'value' keyword always identifies the new value.
    }   
    public Int32 Age {
        get { return(m_Age); }
        set {
            if (value < 0)    // The 'value' keyword always identifies the new value.
                throw new ArgumentOutOfRangeException("value", value.ToString(),
               "The value must be greater than or equal to 0");
            m_Age = value;
        } 
    }
}
```

When defining a property, depending on its definition, the compiler will emit either two or three of items into the resulting managed assembly:
+ A method representing the property’s get accessor method. Only if exists *get* accessor method.
+ A method representing the property’s set accessor method. Only if exists *set* accessor method.
+ A property definition in the managed assembly’s metadata. This is always emitted.

When C# compiler sees code that get or set
a property, the compiler actually emits a call to one of these methods. In addition, compilers also emit a property definition entry into the managed assembly’s metadata for each property. This entry contains some flags and the type of the property, and it refers to the get and set accessor methods. Compilers and other tools can use this metadata, which can be obtained by using the **System.Reflection.PropertyInfo** class.

## Automatically Implemented Properties

If you are creating a property to simply encapsulate a backing field, then C# offers a simplified syntax known as **automatically implemented properties**. For example,
```c#
public String Name { get; set; }
```
Compiler will automatically declare a private field with the type of property, implement the get_Name and set_Name methods.

## Object and Collection Initializers

C# language supports a special object initialization syntax.
```c#
Employee e = new Employee() { Name = "Jeff", Age = 45 };
```

Also, if a property’s type implements the IEnumerable or IEnumerable\<T> interface, we can construct as follow.
```c#
public sealed class Classroom {
    private List<String> m_students = new List<String>();
    public List<String> Students { get { return m_students; } }
    public Classroom() {}
}

public static void M() {
    Classroom classroom = new Classroom {
        Students = { "Jeff", "Kristin", "Aidan", "Grant" }
    };
}
```

For *Dictionary* type property, 
```c#
var table = new Dictionary<String, Int32> {
   { "Jeffrey", 1 }, { "Kristin", 2 }, { "Aidan", 3 }, { "Grant", 4 }
};
```

## Anonymous Types

```c#
var o1 = new { Name = "Jeff", Year = 1964 };
```
The compiler infers the type of each expression, creates private fields of these inferred types, creates public read-only properties for each of the fields, and creates a constructor that accepts all these expressions. \
In addition, the compiler overrides **Object’s Equals, GetHashCode, and ToString** methods and generates code inside all these methods.

The compiler is very intelligent about defining anonymous types. If you are defining multiple anonymous types that have the identical structure, the compiler will create just one definition for the anonymous type and create multiple instances.

So, we can create an implicitly typed array of anonymous types.
```c#
var people = new[] {
   o1,  // From earlier in this section
   new { Name = "Kristin", Year = 1970 },
   new { Name = "Aidan", Year = 2003 },
   new { Name = "Grant", Year = 2008 }
};
```

Anonymous types are most commonly used with the Language Integrated Query (LINQ) tech- nology, where you perform a query that results in a collection of objects that are all of the same anonymous type.

Here is an example that returns all the files in my document directory that have been modified within the past seven days.
```c#
String myDocuments = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments);
var query =
    from pathname in Directory.GetFiles(myDocuments)
    let LastWriteTime = File.GetLastWriteTime(pathname)
    where LastWriteTime > (DateTime.Now ­ TimeSpan.FromDays(7))
    orderby LastWriteTime
    select new { Path = pathname, LastWriteTime }; // Set of anonymous type objects
foreach (var file in query)
    Console.WriteLine("LastWriteTime={0}, Path={1}", file.LastWriteTime, file.Path);
```

Instances of anonymous types are not supposed to leak outside of a method. A method cannot accept a parameter of an anonymous type because it cannot specify the anonymous type. 

Similarly, a method cannot return a reference to an anonymous type. Although the instance of an anonymous type can be cast as Object, there is no way to cast back into an anonymous type, because you don’t know the name of the anonymous type at compile time.

If you want to pass a tuple around, then you should consider using the **System.Tuple** type

## System.Tuple Type

System namespace has several generic Tuple types that differ by the number of parameters.

Here is an example of a method that uses a Tuple type to return two pieces of information to a caller.
```c#
// Returns minimum in Item1 & maximum in Item2
private static Tuple<Int32, Int32> MinMax(Int32 a, Int32 b) {
    return new Tuple<Int32, Int32>(Math.Min(a, b), Math.Max(a, b));
}
// This shows how to call the method and how to use the returned Tuple
private static void TupleTypes() {
    var minmax = MinMax(6, 2);
    Console.WriteLine("Min={0}, Max={1}", minmax.Item1, minmax.Item2); // Min=2, Max=6
}
```

With anonymous types, the properties are given actual names. With Tuple types, the properties are assigned their Item# names by Microsoft and you cannot change it. This reduces code readability and maintainability so you should add comments to your code.

# Parameterful Properties

**Parameterless Properties:** get accessor methods for the property accept no parameters.
**Parameterful Properties:** get and set accessor methods accept parameters.

Different languages use different terms to refer to parameterful properties: C# calls *indexers*, VB calls *default properties*.

In C#, parameterful properties (indexers) are exposed using an array-like syntax. In other words, you can think of an indexer as a way to overload the [] operator.

Here’s an example of a BitArray class that allows array-like syntax to index into the set of bits.
```c#
public sealed class BitArray {
    // Private array of bytes that hold the bits
    private Byte[] m_byteArray;
    private Int32  m_numBits;

    // Constructor that allocates the byte array and sets all bits to 0
    public BitArray(Int32 numBits) {
        // Validate arguments first.
        if (numBits <= 0)
            throw new ArgumentOutOfRangeException("numBits must be > 0");
        // Save the number of bits.
        m_numBits = numBits;
        // Allocate the bytes for the bit array.
        m_byteArray = new Byte[(numBits + 7) / 8];
    }
    
    // This is the indexer (parameterful property).
    public Boolean this[Int32 bitPos] {
        // This is the indexer's get accessor method.
        get {
           // Validate arguments first
           if ((bitPos < 0) || (bitPos >= m_numBits))
              throw new ArgumentOutOfRangeException("bitPos");
           // Return the state of the indexed bit.
           return (m_byteArray[bitPos / 8] & (1 << (bitPos % 8))) != 0;
        }

        // This is the indexer's set accessor method.
        set {
           if ((bitPos < 0) || (bitPos >= m_numBits))
              throw new ArgumentOutOfRangeException("bitPos", bitPos. TString());
           if (value) {
              // Turn the indexed bit on.
              m_byteArray[bitPos / 8] = (Byte)
                 (m_byteArray[bitPos / 8] | (1 << (bitPos % 8)));
           } else {
              // Turn the indexed bit off.
              m_byteArray[bitPos / 8] = (Byte)
                 (m_byteArray[bitPos / 8] & ~(1 << (bitPos % 8)));
           }
        } 
    }
}
```

Using the BitArray class’s indexer is incredibly simple.
```c#
// Allocate a BitArray that can hold 14 bits.
BitArray ba = new BitArray(14);
// Turn all the even­numbered bits on by calling the set accessor.
for (Int32 x = 0; x < 14; x++) {
    ba[x] = (x % 2 == 0);
}
// Show the state of all the bits by calling the get accessor.
for (Int32 x = 0; x < 14; x++) {
    Console.WriteLine("Bit " + x + " is " + (ba[x] ? "On" : "Off"));
}
```

In the BitArray example, the indexer takes one Int32 parameter. \
Dictionary type offers an indexer that takes a key and returns the value associated with the key. \
ColorMatrix class has an indexer that takes more than one parameter.

The compiler automatically generates names for these methods, they choose *Item*, i.e., *get_Item* and *set_Item*.

**System.Runtime.CompilerServices.IndexerNameAttribute** allows to change the default name. For example, the name of **String**’s indexer is *Chars* instead of *Item*.

# About the Performance of Calling Property Accessor Methods

For simple get and set accessor methods, the just-in-time (JIT) compiler inlines the code, so that there’s no run-time performance hit.

**Inlining** is when the code for a method (or accessor method, in this case) is compiled directly in the method that is making the call. \
This removes the overhead associated with making a call at run time, at the expense of making the compiled method’s code bigger. Because property accessor methods typically contain very little code, inlining them can make the native code smaller and can make it execute faster.
