---
layout:     post
title:      Chapter 18. Custom Attributes
subtitle:   
date:       2022-09-04
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

One of the most innovative features the .NET Framework offers: **Custom Attributes**.

Custom attributes allow you to declaratively annotate your code, thereby enabling special features.

## Using Custom Attributes

The first thing you should realize about custom attributes is that, they’re just a way to associate additional information with a target. \
Most attributes have no meaning for the compiler; the compiler simply detects the attributes in the source code and emits the corresponding metadata.

The .NET FCL defines hundreds of custom attributes that can be applied. For example,
+ **Serializable** attribute to a type, informs the serialization formatters that an instance’s fields may be serialized and deserialized.
+ **AssemblyVersion** attribute to an assembly sets the version number of the assembly.

A custom attribute is simply an instance of a type, must be derived from the public abstract **System.Attribute** class.

```c#
[DllImport("Kernel32", CharSet = CharSet.Auto, SetLastError = true)]
```

In this example, *Kernel32* is passed for the instance constructor (mandatory). The other parameters are used to set the fields and properties of *DllImportAttribute* (optional).

## Defining Your Own Attribute Class

Say you’re responsible for adding the bit flag support to enumerated types. The first thing you have to do is define a FlagsAttribute class.
```c#
namespace System {
    public class FlagsAttribute : System.Attribute {
        public FlagsAttribute() {
        } 
    }
}
```
Notice that the *FlagsAttribute* class inherits from *Attribute*, and has a simple public constructor. \
Then, to make this attribute be applied to enumerated types only, we should apply an instance of the *System.AttributeUsageSttribute* class to the attribute class.

```c#
namespace System {
    [AttributeUsage(AttributeTargets.Enum, Inherited = false)]
    public class FlagsAttribute : System.Attribute {
        public FlagsAttribute() {
        } 
    }
}
```

The *AttributeUsageAttribute* class offers two additional public properties: **AllowMultiple** and **Inherited**.

For most attributes, it makes no sense to apply them to a single target more than once. However, for a few attributes, it does make sense to apply the attribute multiple times to a single target. For example, *ConditionalAttribute*. 

The *Inherited* property indicates if the attribute should be applied to derived classes and overriding methods when applied on the base class.

```c#
[AttributeUsage(AttributeTargets.Class | AttributeTargetsMethod, Inherited=true)]
internal class TastyAttribute : Attribute {
}

[Tasty][Serializable]
internal class BaseType {
    [Tasty] 
    protected virtual void DoSomething() { }
}
  
internal class DerivedType : BaseType {
    protected override void DoSomething() { }
}
```

In this code, **DerivedType** and its **DoSomething** method are both considered **Tasty** because the **TastyAttribute** class is marked as inherited. However, **DerivedType** is not serializable because the FCL’s **SerializableAttribute** class is marked as a non-inherited attribute.

## Detecting the Use of a Custom Attribute

Defining an attribute class is useless by itself, we still need to check if the enumerated type has the Flags attribute metadata associated, to change the behavior of application.

If you were the Microsoft employee responsible for implementing Enum’s Format method, you would implement it like the following.
```c#
public override String ToString() {
    // Does the enumerated type have an instance of the FlagsAttribute type applied to it?
    if (this.GetType().IsDefined(typeof(FlagsAttribute), false)) {
        // Yes; execute code treating value as a bit flag enumerated type.
        ...
    } else {
        // No; execute code treating value as a normal enumerated type.
        ... 
    }
    ... 
}
```

To check for an attribute on a target, **System.Reflection.CustomAttributeExtensions** class defines three static methods for retrieving the attributes associated with a target: 
+ IsDefined
+ GetCustom­ Attributes
+ GetCustomAttribute

## Detecting the Use of a Custom Attribute Without Creating Attribute-Derived Objects

When you call **Attribute’s GetCustom­ Attribute(s)** methods, internally, these methods call the attribute class’s constructor and can also call property set accessor methods. 

In some security-conscious scenarios, we use the **System.Reflection.CustomAttributeData** class to discover attributes without allowing attribute class code to execute.

## Conditional Attribute Classes

see https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.conditionalattribute?view=net-6.0