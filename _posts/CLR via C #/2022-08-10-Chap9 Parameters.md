---
layout:     post
title:      Chapter 9. Parameters
subtitle:   
date:       2022-08-10
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

# Optional and Named Parameters

Optional parameters means that when you define a method's parameters, you can assign default values to some parameters. Then, you can call this method without specifing these parameters.

Named parameters means that when you call a method, you can specify arguments by using the name of their parameters.

## Rules and Guidelines

+ Parameters with default values must come after any parameters that do not have default values. In other words, after you define a parameter as having a default value, then all parameters to the right of it must also have default values.

+ Be careful not to rename parameter variables, because any callers who are passing arguments by parameter name will have to modify.

+ Cannot set default values for parameters marked with either the *ref* or *out* keywords.

In C#, when you give a parameter a default value, compiler internally assign the **OptionalAttribute** and **DefaultParameterValue** to the parameter, and this parameter is persisted in the metadata.

# Implicitly Typed Local Variables

C# support to infer the type of local variable from right expression, by using the **var** keyword.
```c#
private static void ImplicitlyTypedLocalVariables() {
    var name = "Jeff";
    // Displays: System.String
    ShowVariableType(name);  
 
    // Error: Cannot assign <null> to an implicitly­typed local  variable
    // var n = null;
 
    var x = (String)null;
    // Displays: System.String
    ShowVariableType(x);  
 
    var numbers = new Int32[] { 1, 2, 3, 4 };
    // Displays: System.Int32[]
    ShowVariableType(numbers); 
 
    var collection = new Dictionary<String, Single>() { { "Grant", 4.0f } };  
    // Displays: System.Collections.Generic.Dictionary`2[System. String,System.Single]
    ShowVariableType(collection);
 
    foreach (var item in collection) {
       // Displays: System.Collections.Generic.KeyValuePair`2[System. String,System.Single]
       ShowVariableType(item);
     } 
}
private static void ShowVariableType<T>(T t) {
    Console.WriteLine(typeof(T));
}
```

can NOT declare a method’s parameter type by using var.

can NOT declare a type’s field by using var.

Do not confuse *dynamic* and *var*. \
**var** is just a syntax shortcut that make compiler infer the specific data type from an expression. \
The **var** keyword can be used only for declaring local variables inside a method, whereas the **dynamic** keyword can be used for local variables, fields, and arguments. \
You must explicitly initialize a variable declared using **var**, whereas you do not have to initialize a variable de- clared with **dynamic**.

# Passing Parameters by Reference to a Method

When reference type objects are passed, the reference (or pointer) to the object is passed (by value) to the method. This means that the method can modify the object and the caller will see the change. 

For value type instances, a copy of the instance is passed to the method. This means that the method gets its own copy of the value type and the instance in the caller isn’t affected.

CLR allows you to pass parameters by reference instead of by value. In C#, you do this by using the **out** and **ref** keywords. 

From the CLR’s perspective, **out** and **ref** are identical. The same IL and metadata are produced, except for 1 bit, which is used to record whether you specified *out* or *ref* when declaring the method. 

Compiler treats the two keywords differently, if marked with out, the caller isn’t expected to have initialized the object, if marked with ref, the caller must initialize the parameter’s value prior to calling.

**Using out and ref with value types gives the same behavior that passing reference types by value.**

# Passing a Variable Number of Arguments to a Method

Declare a method that accepts a variable number of arguments.
```c#
static Int32 Add(params Int32[] values) {
    // NOTE: it is possible to pass the 'values'
    // array to other methods if you want to.
    Int32 sum = 0;
    if (values != null) {
        for (Int32 x = 0; x < values.Length; x++)
            sum += values[x];
    }
    return sum; 
}
```

Obviously, code can call this method
```c#
// Displays "15"
Console.WriteLine(Add(new Int32[] { 1, 2, 3, 4, 5 } ));
```

But, we can also call to Add as follows.
```c#
// Displays "15"
Console.WriteLine(Add(1, 2, 3, 4, 5));
```

**params** keyword tells the compiler to apply the System.ParamArrayAttribute custom attribute to the parameter.

When the C# compiler detects a call to a method, compiler first checks all of the methods with the specified name, where no parameter has **ParamArray** attribute applied. If a method exists, compiler generates the code to call it. If not exists, it looks for methods that have a ParamArray attribute. If the compiler finds a match, it constructs an array and populates its elements, then call the method.

Only the last parameter to a method can be marked with the params keyword.

It's legal to pass *null* or *a reference to an array of 0 entries** as the last parameter to the method.
```c#
// Both of these lines display "0"
Console.WriteLine(Add());     // passes new Int32[0] to Add
Console.WriteLine(Add(null)); // passes null to Add: more efficient(no array allocated)
```

If we want the parameters could be any type, just takes an Object[] instead of Int32[].

**Attention:** Be aware that calling a method that takes a variable number of arguments in- curs an additional performance hit unless you explicitly pass null. After all, an array object must be allocated on the heap. \
To help reduce the performance hit, consider defining a few overloaded methods that do not use the params keyword.

# Parameter and Return Type Guidelines

When declaring a method’s **parameter types**, you should **specify the weakest type** possible, preferring interfaces over base classes. For example, 
```c#
// Desired: This method uses a weak parameter type
public void ManipulateItems<T>(IEnumerable<T> collection) { ... }

// Undesired: This method uses a strong parameter type
public void ManipulateItems<T>(List<T> collection) { ... }
```

The reason is that someone can call the first method passing in an array object, a List\<T> object, a String object, and so on—any object whose type implements IEnumerable\<T>. The second method allows only List\<T> objects to be passed in; it will not accept an array or a String object. \
Obviously, the first method is better because it is much more flexible and can be used in a much wider range of scenarios.

It is usually best to declare a method’s **return type by using the strongest type** possible. For example, it is better to declare a method that returns a FileStream object as opposed to returning a Stream object.
```c#
// Desired: This method uses a strong return type
public FileStream OpenFile() { ... }

// Undesired: This method uses a weak return type
public Stream OpenFile() { ... }
```

Here, the first method is preferred because the returned object can be either a FileStream object or as a Stream object. The second method requires that the caller treat the returned object as a Stream object. 

