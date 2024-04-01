---
layout:     post
title:      Adapter Pattern
subtitle:   
date:       2022-12-14
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

![img](/img/DesignPattern/adapter_motivation.png)

### Motivation

Remember our ducks mentioned in Strategy Pattern? Duck has **Fly** and **Quack** behaviors. Here is a subclass of Duck, the MallardDuck.
```c#
public interface Duck 
{
    public void Quack();
    public void Fly();
}

public class MallardDuck : Duck
{
    public void Fly()
    {
        Console.WriteLine("I'm flying!");
    }

    public void Quack()
    {
        Console.WriteLine("Quack!");
    }
}
```
Now, let’s say you’re short on Duck objects and you’d like to use some Turkey objects in their place. Obviously we can’t use the turkeys directly because they have a different interface. So we need an adapter to convert the turkeys into something like ducks.
```c#
public interface ITurkey
{
    public void Gobble();
    public void Fly();
}
```

### Definition

The **Adapter Pattern** converts the interface of a class into another interface the clients expect. Adapter lets classes that have incompatible interfaces work together.

### Applicability

When you want to use some existing class, but its interface isn’t compatible with the rest of your code. The adapter pattern lets you create a middle-layer class that serves as a translator between your code and a legacy class.

### Structure

![img](/img/DesignPattern/adapter.png)

In some languages that support multiple inheritance, we can also use following implementation.

![img](/img/DesignPattern/adapter2.png)

### Participants

In the first structure, the adapter class must implement the target interface, and reference to the adaptee object. 

In the second implementation, the adapter class is the subclass of the target interface and adaptee.

### Consequence

**Single Responsibility Principle.** You can separate the interface or data conversion code from the primary business logic.

**Open Closed Principle.** You can introduce new adapters without breaking the existing client code, as long as they work with the adapters through the client interface.

### Implementation

```c#
public class TurkeyAdapter : IDuck
{
    public ITurkey Turkey { get; set; }

    public TurkeyAdapter(Turkeys.ITurkey turkey)
    {
        Turkey = turkey;
    }
    
    public void Fly()
    {
        Turkey.Fly();
    }

    public void Quack()
    {
        for (int i = 0; i < 5; i++)
        {
            Turkey.Gobble();
        }
    }
}
```

See [here](https://github.com/haoozhang/Head-First-Design-Pattern/tree/main/AdapterPattern) for complete code sample.

### Known Uses


### Related Patterns

**Decorator Pattern** enhances an object without changing its interface. In addition, it supports recursive composition. **Adapter Pattern** changes the interface of an existing object.

