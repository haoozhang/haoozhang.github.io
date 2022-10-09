---
layout:     post
title:      Chapter 24. Runtime Serialization
subtitle:   
date:       2022-09-25
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

**Serialization** is the process of converting an object into a stream of bytes. **Deserialization** is the process of converting a stream of bytes back into the object.

+ An application’s state can easily be saved in a disk file or database and then restored.
+ A set of objects can easily be sent over the network to a process running on another machine.

.NET framework has built support for serialization and deserialization. As a developer, you can work with you object before serialization and after deserialization, and have .NET framework handle the middle.

## Serialization/Deserialization Quick Start

```c#
using System;
using System.Collections.Generic;
using System.IO;
using System.Runtime.Serialization.Formatters.Binary;

internal static class QuickStart {
    public static void Main() {
        // Create a graph of objects to serialize them to the stream
        var objectGraph = new List<String> { "Jeff", "Kristin", "Aidan", "Grant" };

        Stream stream = SerializeToMemory(objectGraph);

        // Reset everything for this demo
        stream.Position = 0;
        objectGraph = null;

        // Deserialize the objects and prove it worked
        objectGraph = (List<String>) DeserializeFromMemory(stream);
        
        foreach (var s in objectGraph) Console.WriteLine(s);
    }

    private static MemoryStream SerializeToMemory(Object objectGraph) {
       // Construct a stream that is to hold the serialized objects
       MemoryStream stream = new MemoryStream();

       // Construct a serialization formatter that does all the hard work
       BinaryFormatter formatter = new BinaryFormatter();

       // Tell the formatter to serialize the objects into the stream
       formatter.Serialize(stream, objectGraph);

       // Return the stream of serialized objects back to the caller
       return stream;
    }

    private static Object DeserializeFromMemory(Stream stream) {
       // Construct a serialization formatter that does all the hard work
       BinaryFormatter formatter = new BinaryFormatter();

       // Tell the formatter to deserialize the objects from the stream
       return formatter.Deserialize(stream);
    }
}
```

Note that: 
+ Ensure to use the same formatter for both serialization and deserialization. For example, don’t serialize by using the **SoapFormatter** and then deserialize by using the **BinaryFormatter**.
+ Can serialize multiple objects out to a single stream. For example,
```c#
// class definition
[Serializable] internal sealed class Customer { /* ... */ }
[Serializable] internal sealed class Order    { /* ... */ }

// define static fields
private static List<Customer> s_customers       = new List<Customer>();
private static List<Order>    s_pendingOrders   = new List<Order>();
private static List<Order>    s_processedOrders = new List<Order>();

// Serialize
BinaryFormatter formatter = new BinaryFormatter();
// Serialize our application's entire state
formatter.Serialize(stream, s_customers);
formatter.Serialize(stream, s_pendingOrders);
formatter.Serialize(stream, s_processedOrders);

// Deserialize
BinaryFormatter formatter = new BinaryFormatter();
// Deserialize our application's entire state (same order as serialized)
s_customers       = (List<Customer>) formatter.Deserialize(stream);
s_pendingOrders   = (List<Order>)    formatter.Deserialize(stream);
s_processedOrders = (List<Order>)    formatter.Deserialize(stream);
```
+ When deserializing an object, the full name of the type and the name of type's defining assembly are wrritten to the stream. When deserializing an object, the formatter first load the assembly, then look up the type matching the object being deserialized.

## Making a Type Serializable

When a type is designed, the developer must decide whether or not the type can be serializable. **By default, types are not serializable.** For example, 

```c#
internal struct Point { public Int32 x, y; }

private static void OptInSerialization() {
    Point pt = new Point { x = 1, y = 2 };
    using (var stream = new MemoryStream()) {
        new BinaryFormatter().Serialize(stream, pt); // throws SerializationException
    }
}
```

To solve this problem, the developer must apply **System.SerializableAttribute** custom attribute to this type.

```c#
[Serializable]
internal struct Point { public Int32 x, y; }
```

When serializing an object, the formatter checks that every object’s member type is serializable. If any member is not serializable, it throws **SerializationException**.

The SerializeAttribute custom attribute can be applied to **class, struct, enum, delegate**. Note that enumerated and delegate types are always serializable, so there is no need to explicitly apply the **SerializableAttribute** attribute.

The **SerializableAttribute** custom attribute **not inherited** by derived types. So a Person object can be serialized, but an Employee object cannot.

```c#
[Serializable]
internal class Person { ... }

internal class Employee : Person { ... }
```

To fix this, just apply the **SerializableAttribute** attribute to the Employee type.

```c#
[Serializable]
internal class Person { ... }

[Serializable]
internal class Employee : Person { ... }
```

However, if the base type is not allowed to be serialized, To make its derived type serializable is not easy. Because the fields of base type are also th part of derived type. This is why System.Object has the **SerializableAttribute** attribute applied to it.

## Controlling Serialization and Deserialization

When applying **SerializableAttribute** custom attribute to a type, all instance fields (public, private, protected, and so on) are serialized. However, sometimes we want some fields should not be serialized. We can use **System.NonSerializedAttribute** attribute.

```c#
[Serializable]
internal class Circle {
    private Double m_radius;

    [NonSerialized]
    private Double m_area;

    public Circle(Double radius) {
        m_radius = radius;
        m_area = Math.PI * m_radius * m_radius;
    }
    ...
}
```

This attribute can be applied only to a type’s fields, and it continues to apply to this field when inherited by another type.

It may causes some problems when deserializing, because **m_radius** can be deserialized correctly but **m_area** still be 0. The following code demonstrates how to fix this problem.

```c#
[Serializable]
internal class Circle {
    private Double m_radius;

    [NonSerialized]
    private Double m_area;

    public Circle(Double radius) {
        m_radius = radius;
        m_area = Math.PI * m_radius * m_radius;
    }
   
    [OnDeserialized]
    private void OnDeserialized(StreamingContext context) {
        m_area = Math.PI * m_radius * m_radius;
    }
}
```
In addition to the **OnDeserializedAttribute** custom attribute, the **System.Runtime.Seri­alization** namespace also defines **OnSerializingAttribute**, **OnSerializedAttribute**, and **OnDeserializingAttribute** custom attributes, which you can use to control serialization and deserialization more. Here is a sample.

```c#
[Serializable]
public class MyType {
    Int32 x, y; 

    [NonSerialized] 
    Int32 sum;
    
    public MyType(Int32 x, Int32 y) {
        this.x = x; this.y = y; 
        sum = x + y;
    }

    [OnDeserializing]
    private void OnDeserializing(StreamingContext context) {
        // Example: Set default values for fields in a new version of this type
    }

    [OnDeserialized]
    private void OnDeserialized(StreamingContext context) {
        // Example: Initialize transient state from fields
        sum = x + y; 
    }

    [OnSerializing]
    private void OnSerializing(StreamingContext context) {
        // Example: Modify any state before serializing
    }

    [OnSerialized]
    private void OnSerialized(StreamingContext context) {
        // Example: Restore any state after serializing
    }
}
```

When serializing an object, the formatter first calls all methods marked with **OnSerializing**, then serializes all the object's fileds, finally calls all methods marked with **OnSerialized** attribute. Similarly, when deserializing an object, the fromatter first calls all methods marked with **OnDeserializing** attribute, then deserializes all object's fields, and then calls all methods marked with **OnDeserialized** attribute.

If you serialize an instance of a type, add a new field to the type, and then try to deserialize the object that did not contain the new field, the formatter throws a **SerializationException**, indicating that the data in the stream being deserialized has the wrong number of members. You can apply **OptionalFieldAttribute** to each new field to fix this problem.

## How Formatters Serialize Type Instances

To make serialization / deserialization easier, the FCL offers a **FormatterService** type, which has only static methods and no instance.

### Serializes an object whose type has the SerializableAttribute attribute applied

1. The formatter calls **FormatterServices’s GetSerializableMembers** method.
```c#
public static MemberInfo[] GetSerializableMembers(Type type, StreamingContext context);
```
This method uses reflection to get the type's public and private fields (excluding any fields marked with **NonSerializedAttribute**). The method returns an array of **MemberInfo** objects.

2. The object being serialized and the **MemberInfo** array are passed to **FormatterServices’ GetObjectData** method.
```c#
public static Object[] GetObjectData(Object obj, MemberInfo[] members);
```
This method returns an array of **Objects** where each element identifies the value of a field in the object being serialized. This Object array and the MemberInfo array are parallel. That is, Object[0] id the value of MemberInfo[0].

3. The formatter writes the assembly’s identity and the type’s full name to the stream.

4. The formatter then enumerates over the elements in the two arrays, writing each member’s name and value to the stream.

### Deserializes an object whose type has the SerializableAttribute attribute applied

1. The formatter reads the assembly’s identity and full type name from the stream, and passed them to **FormatterServices’ GetTypeFromAssembly** method.
```c#
public static Type GetTypeFromAssembly(Assembly assem, String name);
```
This method returns a **System.Type** object indicating the type of object that is being deserialized.

2. The formatter calls **FormatterServices**’s static **GetUninitializedObject** method.
```c#
public static Object GetUninitializedObject(Type type);
```
This method allocates memory for a new object but does not call a constructor for the object. However, all the object’s bytes are initialized to null or 0 automatically.

3. The formatter now constructs and initializes a **MemberInfo** array by calling the **FormatterServices’s GetSerializableMembers** method.

4. The formatter creates and initializes an **Object** array from the data contained in the stream.

5. the newly allocated object, the MemberInfo array, and the parallel Object array are passed to **FormatterServices’** static **PopulateObjectMembers** method.
```c#
public static Object PopulateObjectMembers(Object obj, MemberInfo[] members, Object[] data);
```
This method enumerates over the arrays, initializing each field to its corresponding value. At this point, the object has been completely deserialized.

## Streaming Contexts

As mentioned earlier, there are many destinations for a serialized set of objects: same process, different process on the same machine, different process on a different machine. 

A number of the methods accept a **StreamingContext**, which just offers twp public read-only properties.

+ StreamingContextStates State. a set of bit flags indicating the source or destination of (de)serialization.
+ Object Context. contains any context information.

When you construct a formatter, the formatter initializes its **Context** property, that is, **StreamingContext­States** is set to **All** and state object is aet to **null**. \
After the formatter is constructed, you can construct a **StreamingContext** structure using any **StreamingContextStates** bit flags, and optionally an additional context information. \
Now, all you need to do is set the formatter’s **Context** property with this new **StreamingContext** object before calling the formatter’s **Serialize** or **Deserialize** methods.

