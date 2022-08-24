---
layout:     post
title:      Chapter 12. Generics
subtitle:   
date:       2022-08-23
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - CLR via C#
---

One of the big benefits from object-oriented programming is code reuse, the ability to derive a class. The derived class can inherits all of a base class, override virtual methods, and add some new methods to customize behaviors.

**Generics** is another benefit provided by CLR and programming language, to provide one more form of code reuse: algorithm reuse. For example, generic list **List\<T>**.

Generics provide the following big benefits to developers:
+ **Source code protection** : The developer using a generic algorithm doesn’t need to have access to the algorithm’s source code.
+ **Type safety** : When using generic algorithm with specific type, compiler and CLR will ensure type compatibility.
+ **Cleaner code** : Because the compiler enforces type safety, fewer casts are required, code is easier to write and maintain.
+ **Better performance** : If no generics, we will have to work with *Object* data type, and CLR has to box and unbox, which causes additional memory allocateions and performance hurt.

Framework Class Library (FCL) provides some generic collections. Most of these classes can be found in the **System.Collections.Generic** namespace.

## Generic Infrastructure

CLR create type object for each type in an application. A type with generic type parameters is still considered a type. 

However, a type with generic type parameters is called an **open type**, and CLR does not allow any instance of an open type. \
If we passed actual type for type arguments, the type is called a **closed type**, and CLR allow instances of a closed type.

CLR has some optimizations to reduce code explosion (generate native code for every method/type combination). 
+ If a method is called for a particular type argument, and later, the method is called again using the same type argument, CLR will compile this method/type combination just once.
+ CLR considers all reference type arguments to be identical, because all reference type arguments or variables are really just pointers. 
+ But if any type argument is a value type, CLR must produce native code specifically for that value type.




