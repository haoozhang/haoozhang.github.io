---
layout:     post
title:      Chapter 23. Assembly Loading and Reflection
subtitle:   
date:       2022-09-22
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

This chapter is all about discovering types information, creating instances of them, and accessing their members when you don't know about anything at compile time. It is typically used to create a dynamically extensible application. 

For example, one company builds a host application and other companies create add-ins to extend the host application. The host can’t be built or tested against the add-ins because the add-ins are created by different companies and are likely to be created after the host application has already shipped. This is why the host needs to discover the add-ins at run time.

A dynamically extensible application could take advantage of common language runtime (CLR) hosting and AppDomains as discussed earlier.

## Assembly Loading

As you know, when the just-in-time (JIT) compiler compile the IL code into native code for a method, it sees what types are referenced in the IL code. Then at run time, the JIT compiler uses the assembly's **TypeRef** and **AssemblyRef** metadata tables to determine which assembly defines the referenced type. \
The **AssemblyRef** table entry containes all of the parts that make up the strong name of the assembly. JIT compiler grabs - name, version, culture, public key token, and then attempts to load an matching assembly into the AppDomain. \
If the assembly being loaded is weakly named, JIT compiler just uses the name of the assembly to load.

### Assembly.Load

CLR attempts to load the assembly by using the **System.Reflection.Assembly#Load** static method, which is to explicitly load an assembly into your AppDomain. \
Internally, **Load** makes the CLR to apply a version-binding redirection policy to the assembly and looks for the assembly in the global assembly cache (GAC), application's base directory, private path subdirectories, and codebase locations. If **Load** reveives a weakly named assembly, it doesn't apply a version-binding policy. \
If **Load** finds the specified assembly, it returns a **Assembly** object. If fails, it throws FileNotFound exception.

### Assembly.LoadFrom

**Assembly#LoadFrom** static method loads an assembly specifying a path name.

Internally, **LoadFrom** firstly calls **AssemblyName#GetAssemblyName** method to open specified file and find **AssemblyDef** metadata table, then searches the assembly identity information from the **AssemblyDef** metadata table, and returns it in **AssemblyName** object. \
Then, **LoadFrom** internally calls **Assembly’s Load** method. If **Load** finds the assembly, it will load it. \
If **Load** fails to find an assembly, **LoadFrom** loads the assembly specified in the path argument.

By the way, LoadFrom method allows to pass a URL as the argument. CLR will downloads the file, installs it into the user’s download cache, and loads the file from there.

### Assembly.ReflectionOnlyLoadFrom or ReflectionOnlyLoad

If you are building a tool that simply analyzes an assembly’s metadata via reflection, and ensure that none of the code contained inside the assembly executes. The best way for you to load an assembly is to use **Assembly’s ReflectionOnlyLoadFrom** method, or in some rarer cases, **Assembly’s ReflectionOnlyLoad** method.

**ReflectionOnlyLoadFrom** method will load the file specified by the path, the strong name identity is not obtained, and the file is not searched in the GAC or elsewhere. \
**ReflectionOnlyLoad** method will search for the specified assembly looking in the GAC, application base directory, private paths, and codebases. However, unlike the **Load** method, the **ReflectionOnlyLoad** method does not apply versioning policies, so you will get the exact version that you specify.

When using reflection to analyse an assembly loaded with one of these two methods, the code will have to frequently register a callback method with AppDomain’s **ReflectionOnlyAssembly­Resolve** event to manually load any referenced assemblies. (CLR doesn't do it for you automatically)

Many applications consists of an EXE file that dependes on many DLL files. When deploying the application, all od files must be deployed. However, there is a technique that you can use to deplot a single EXE file. \
You need find all referenced DLL files, then add them to Visual Studio project. For each DLL file, you change its **BuildAction** to **Embedded**. This causes the C# compiler to embed the DLL files into EXE file, and you can deploy this one EXE file.

## Using Reflection to Build a Dynamically Extensible Application

As you know, metadata is stored in a bunch of tables. When building an assembly or module, the compiler creates a type definition table, a field definition table, a method definition table, and so on. The **System.Reflection** namespace contains several types that allow to write code to parse these metadata tables.

You can easily enumerate all of the types in a type definition metadata table. Then for each type, you can obtain its base type, the interfaces it implements. You can query the type’s fields, methods, properties, and events. You can also discover any custom attributes that applied to metadata entities.

In reality, very few applications will have the need to use the reflection types. 

## Reflection Performance

Reflection is an extremely powerful mechanism because it allows you to discover and use types and members at run time that you did not know about at compile time. This power does come with two main drawbacks:
+ Reflection prevents type safety at compile time. Reflection uses strings heavily, for example, if you call **Type.GetType("int");** to ask reflection to find a type called "int", the code can be compiled but returns null at run time, because CLR knows "System.Int32" only.
+ Reflection is slow. You discover the types and members by using a string name at run time, which means that it involves too many string searches.

### Discovering Types Defined in an Assembly

```c#
public static class Program {
    public static void Main() {
        String dataAssembly = "System.Data, version=4.0.0.0, " +
            "culture=neutral, PublicKeyToken=b77a5c561934e089";
        LoadAssemAndShowPublicTypes(dataAssembly);
    }

    private static void LoadAssemAndShowPublicTypes(String assemId) {
        // Explicitly load an assembly in to this AppDomain
        Assembly a = Assembly.Load(assemId);
        // Execute this loop once for each Type
        // publicly­exported from the loaded assembly
        foreach (Type t in a.ExportedTypes) {
            // Display the full name of the type
            Console.WriteLine(t.FullName);
        }
    } 
}
```

### Type Object

Notice that the previous code iterates over a sequence of **System.Type**, which represents a type reference. 
Recall that System.Object defines a public, nonvirtual instance method named **GetType**. When calling this method, CLR determines the object's type and returns a **Type** object. 
Because there is only one **Type** object per type in an AppDomain, you can use equality and inequality operators to see if they are same type.

```c#
private static Boolean AreObjectsTheSameType(Object o1, Object o2) {
    return o1.GetType() == o2.GetType();
}
```

FCL offers several more ways to obtain **Type** object:
+ **System.Type#GetType** static method. The primitive type names supported by compiler are NOT allowed because these names mean nothing to CLR.
+ **System.Type#ReflectionOnlyGetType** static method.
+ System.TypeInfo : **DeclaredNestedTypes** and **GetDeclaredNestedType**.
+ System.Reflection.Assembly: **GetType**, **DefinedTypes**, and **ExportedTypes**.

C# offers an operator **typeof** to obtain a Type object from a type name.

### Constructing an Instance of a Type

After you have a reference to a **Type**-derived object, you might want to construct an instance of this type. 

+ System.Activator’s CreateInstance methods
+ System.Activator’s CreateInstanceFrom methods
+ System.AppDomain : CreateInstance, Create­ InstanceAndUnwrap, CreateInstanceFrom, and CreateInstanceFromAndUnwrap.
+ System.Reflection.ConstructorInfo’s Invoke instance method

To create an array, you should call Array’s static **CreateInstance** method. \
To create a delegate, you should call MethodInfo’s **CreateDelegate** method. \
To construct an instance of generic type, first get a reference to the open type, then call **Type#MakeGenericType** method, passing in type arguments. Then, take the returned **Type** object to construct instance.

```c#
internal sealed class Dictionary<TKey, TValue> { }

public static class Program {
    public static void Main() {
        // Get a reference to the generic type's type object
        Type openType = typeof(Dictionary<,>);
        // Close the generic type by using TKey=String, TValue=Int32
        Type closedType = openType.MakeGenericType(typeof(String), typeof(Int32));
        // Construct an instance of the closed type
        Object o = Activator.CreateInstance(closedType);
        // Prove it worked
        Console.WriteLine(o.GetType());
    }
}
```

## Using Reflection to Discover a Type’s Members

So far, this chapter has focused on the parts of reflection—assembly loading, type discovery, and object construction. After an instance is constructed, you can use reflection to discover and then invoke it's members.

### Discovering a Type's Members

Fields, constructors, methods, properties, events, and nested types can all be defined as members within a type. FCL defines a type called **System.Reflection.MemberInfo**, to encapsulate a bunch of properties.

![img](/img/CLR/MemberInfo.png)

```c#
using System;
using System.Reflection;

public static class Program {
    public static void Main() {
        // Loop through all assemblies loaded in this AppDomain
        Assembly[] assemblies = AppDomain.CurrentDomain.GetAssemblies();
        foreach (Assembly a in assemblies) {
            Show(0, "Assembly: {0}", a);
            // Find Types in the assembly
            foreach (Type t in a.ExportedTypes) {
                Show(1, "Type: {0}", t);
                // Discover the type's members
                foreach (MemberInfo mi in t.GetTypeInfo().DeclaredMembers) {
                    String typeName = String.Empty;
                    if (mi is Type) typeName = "(Nested) Type";
                    if (mi is FieldInfo) typeName = "FieldInfo";
                    if (mi is MethodInfo) typeName = "MethodInfo";
                    if (mi is ConstructorInfo) typeName = "ConstructoInfo";
                    if (mi is PropertyInfo) typeName = "PropertyInfo";
                    if (mi is EventInfo) typeName = "EventInfo";
                    Show(2, "{0}: {1}", typeName, mi);
                } 
            }
        }
    }

    private static void Show(Int32 indent, String format, params Object[] args) {
        Console.WriteLine(new String(' ', 3 * indent) + format, args);
    }
}
```

### Invoking a Type’s Members

The following sample application demonstrates the various ways to use reflection to access a type’s members. Each method uses reflection in a different way to accomplish the same thing.

+ **BindToMemberThenInvokeTheMember** method demonstrates how to bind to a member and invoke it later.
+ **BindToMemberCreateDelegateToMemberThenInvokeTheMember** method demonstrates how to bind to an object or member, and then it creates a delegate that refers to that object or member. Calling through the delegate is very fast, and this technique yields faster performance if you intend to invoke the same member on the same object multiple times.
+ **UseDynamicToBindAndInvokeTheMember** method demonstrates how to use C# dynamic primitive type to simplify the syntax for accessing members. This technique can give good performance if you intend to invoke the same member on different objects that are all of the same type.

```c#
// This class is used to demonstrate reflection
// It has a field, constructor, method, property, and an event
internal sealed class SomeType {
    private Int32 m_someField;
    public SomeType(ref Int32 x) { x *= 2; }
    public override String ToString() { return m_someField.ToString(); }
    public Int32 SomeProp {
        get { return m_someField; }
        set {
            if (value < 1)
                throw new ArgumentOutOfRangeException("value");
            m_someField = value;
        }
    }
    public event EventHandler SomeEvent;
    private void NoCompilerWarnings() { SomeEvent.ToString();}
}

public static class Program {
    public static void Main() {
        Type t = typeof(SomeType);
        BindToMemberThenInvokeTheMember(t);
        Console.WriteLine();
        BindToMemberCreateDelegateToMemberThenInvokeTheMember(t);
        Console.WriteLine();
        UseDynamicToBindAndInvokeTheMember(t);
        Console.WriteLine();
    }

    private static void BindToMemberThenInvokeTheMember(Type t) {
        Console.WriteLine("BindToMemberThenInvokeTheMember");

        // Construct an instance
        Type ctorArgument = Type.GetType("System.Int32&"); // or typeof(Int32).MakeByRefType();
        ConstructorInfo ctor = t.GetTypeInfo().DeclaredConstructors.First(
           c => c.GetParameters()[0].ParameterType == ctorArgument);
        Object[] args = new Object[] { 12 };  // Constructor arguments

        Console.WriteLine("x before constructor called: " + args[0]);
        Object obj = ctor.Invoke(args);
        Console.WriteLine("Type: " + obj.GetType());
        Console.WriteLine("x after constructor returns: " + args[0]);

        // Read and write to a field
        FieldInfo fi = obj.GetType().GetTypeInfo().GetDeclaredField("m_someField");
        fi.SetValue(obj, 33);
        Console.WriteLine("someField: " + fi.GetValue(obj));

        // Call a method
        MethodInfo mi = obj.GetType().GetTypeInfo().GetDeclaredMethod("ToString");
        String s = (String)mi.Invoke(obj, null);
        Console.WriteLine("ToString: " + s);

        // Read and write a property
        PropertyInfo pi = obj.GetType().GetTypeInfo().GetDeclaredProperty("SomeProp");
        try {
           pi.SetValue(obj, 0, null);
        }
        catch (TargetInvocationException e) {
           if (e.InnerException.GetType() != typeof(ArgumentOutOfRangeException)) throw;
           Console.WriteLine("Property set catch.");
        }    
        pi.SetValue(obj, 2, null);
        Console.WriteLine("SomeProp: " + pi.GetValue(obj, null));
   
        // Add and remove a delegate from the event
        EventInfo ei = obj.GetType().GetTypeInfo().GetDeclaredEvent("SomeEvent");
        EventHandler eh = new EventHandler(EventCallback); // See ei.EventHandlerType
        ei.AddEventHandler(obj, eh);
        ei.RemoveEventHandler(obj, eh);
    }

    // Callback method added to the event
    private static void EventCallback(Object sender, EventArgs e) { }

    private static void BindToMemberCreateDelegateToMemberThenInvokeTheMember(Type t) {
        Console.WriteLine("BindToMemberCreateDelegateToMemberThenInvokeTheMember");
   
        // Construct an instance (You can't create a delegate to a constructor)
        Object[] args = new Object[] { 12 };  // Constructor arguments
        Console.WriteLine("x before constructor called: " + args[0]);
        Object obj = Activator.CreateInstance(t, args);
        Console.WriteLine("Type: " + obj.GetType().ToString());
        Console.WriteLine("x after constructor returns: " + args[0]);

        // NOTE: You can't create a delegate to a field
        
        // Call a method
        MethodInfo mi = obj.GetType().GetTypeInfo().GetDeclaredMethod("ToString");
        var toString = mi.CreateDelegate<Func<String>>(obj);
        String s = toString();
        Console.WriteLine("ToString: " + s);

        // Read and write a property
        PropertyInfo pi = obj.GetType().GetTypeInfo().GetDeclaredProperty("SomeProp");
        var setSomeProp = pi.SetMethod.CreateDelegate<Action<Int32>>(obj);
        try {
           setSomeProp(0);
        }
        catch (ArgumentOutOfRangeException) {
           Console.WriteLine("Property set catch.");
        }
        setSomeProp(2);
        var getSomeProp = pi.GetMethod.CreateDelegate<Func<Int32>>(obj);
        Console.WriteLine("SomeProp: " + getSomeProp());

        // Add and remove a delegate from the event
        EventInfo ei = obj.GetType().GetTypeInfo().GetDeclaredEvent("SomeEvent");
        var addSomeEvent = ei.AddMethod.CreateDelegate<Action<EventHandler>>(obj);
        addSomeEvent(EventCallback);
        var removeSomeEvent = ei.RemoveMethod.CreateDelegate<Action<EventHandler>>(obj);
        removeSomeEvent(EventCallback);
    }

    private static void UseDynamicToBindAndInvokeTheMember(Type t) {
        Console.WriteLine("UseDynamicToBindAndInvokeTheMember");

        // Construct an instance (You can't use dynamic to call a constructor)
        Object[] args = new Object[] { 12 };  // Constructor arguments
        Console.WriteLine("x before constructor called: " + args[0]);
        dynamic obj = Activator.CreateInstance(t, args);
        Console.WriteLine("Type: " + obj.GetType().ToString());
        Console.WriteLine("x after constructor returns: " + args[0]);

        // Read and write to a field
        try {
           obj.m_someField = 5;
           Int32 v = (Int32)obj.m_someField;
           Console.WriteLine("someField: " + v);
        }
        catch (RuntimeBinderException e) {
           // We get here because the field is private
           Console.WriteLine("Failed to access field: " + e.Message);
        }

        // Call a method
        String s = (String)obj.ToString();
        Console.WriteLine("ToString: " + s);
        // Read and write a property
        try {
           obj.SomeProp = 0;
        }
        catch (ArgumentOutOfRangeException) {
           Console.WriteLine("Property set catch.");
        }
        obj.SomeProp = 2;
        Int32 val = (Int32)obj.SomeProp;
        Console.WriteLine("SomeProp: " + val);
        
        // Add and remove a delegate from the event
        obj.SomeEvent += new EventHandler(EventCallback);
        obj.SomeEvent ­= new EventHandler(EventCallback);
    } 
}
internal static class ReflectionExtensions {
    // Helper extension method to simplify syntax to create a delegate
    public static TDelegate CreateDelegate<TDelegate>(this MethodInfo mi, Object target = null) {
        return (TDelegate)(Object)mi.CreateDelegate(typeof(TDelegate), target);
    }
}
```