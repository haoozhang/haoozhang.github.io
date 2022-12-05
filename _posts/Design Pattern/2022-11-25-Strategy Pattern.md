---
layout:     post
title:      Strategy Pattern
subtitle:   
date:       2022-11-25
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Definition

The **Strategy Pattern** defines a family of algorithms (interfaces), encapsulates each one (multiple implementations of interface), and makes them inter-changeable (set interface implementation). Strategy lets the algorithm dependent from clients that use it (clients face to interface type only).

### Motivation

Using *inheritance* can't solve the problem very well, because some subclasses should not implement all methods defined in base class, e.g., fake duck is duck, but it can't fly, so it should not have the fly behavior defined in base class. 

Using *interface* also can't works well, because no code reuse, i.e., for every implementation change, we have to change all involving sub classes.

### Applicability

A behavior has many implementations, and we only need one implementation to configure our client.  Also, we may need to change the implementation later.

### Structure

![img](/img/DesignPattern/strategy.png)

### Participants

Duck is the base class, and all xxDuck are subclasses, which act as client role.
Encapsulated fly/quack behavior are two family of algorithms, they have multiple inter-changeable implementations for each interface.

For clients, they interact with interface type (FlyBehavior/QuackBehavior), so those implementations are transparent for client side.

### Consequence

Separated the changing part from non-changing part. Now, the subclass can reference the specific implemntation on demand, e.g., *FakeDuck* reference the *FlyNoWay*. Also, if certain behavior needs to be updated, we only change the implementation at one place.

### Implementation

```c#
public abstract class Duck
{
    public IFlyBehavior FlyBehavior { get; set; }

    public abstract void Display();

    public void PerformFly()
    {
        FlyBehavior.Fly();
    }
}

public class MallardDuck : Duck
{
    public MallardDuck()
    {
        FlyBehavior = new FlyWithWings();
    }
    
    public override void Display()
    {
        Console.WriteLine("I'm mallard duck!");
    }
}
```

```c#
public interface IFlyBehavior
{
    public void Fly();
}

public class FlyWithWings : IFlyBehavior
{
    public void Fly()
    {
        Console.WriteLine("fly with wings!");
    }
}
```

See [here](https://github.com/haozhangms/Head-First-Design-Pattern/tree/main/SimUDuck) for complete code sample.

### Known Uses



### Related Patterns

