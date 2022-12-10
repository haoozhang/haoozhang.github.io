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

The **Strategy Pattern** defines a family of algorithms (interfaces), encapsulates each one (multiple implementations of an interface), and makes them inter-changeable (set as interface implementation). Strategy lets the algorithm dependent from clients that use it (clients face to interface type only).

### Motivation

Let's say that we have different kinds of ducks (redhead duck, mallard duck, fake duck and etc), and they have fly behavior and quack behavior. The intuitive solution is to let different kinds of ducks extend and override the super duck class, where contains fly and quack behaviors, just like this.

![img](/img/DesignPattern/strategy_inheritance.png)

However, using *inheritance* can't solve the problem very well, because some sub-classes should not implement all methods defined in super class, e.g., decoy duck is also a kind of duck, but it can't fly, so it should not override the fly behavior defined in super class.

Then we may think out the solution using interface, that is, let those ducks implement the fly and quack interface to customize their behavior. For decoy duck, it doesn't implement fly behavior due to that's not applicable.

![img](/img/DesignPattern/strategy_interface.png)

But using *interface* also can't works well, because no code reuse, i.e., for every potenial interface implementation change later, we have to change all involving sub-classes.

### Applicability

When you want to use different variants of an algorithm within an object and be able to switch from one algorithm to another during runtime.

When you have a lot of similar classes that only differ in the way they execute some behavior.

### Structure

![img](/img/DesignPattern/strategy.png)

### Participants

**Duck** is the base class, and all **xxDuck** are sub-classes, which act as client role.
Encapsulated fly/quack behavior are two family of algorithms, they have multiple inter-changeable implementations for each interface.

For clients, they interact with interface type (**FlyBehavior**/**QuackBehavior**), so those implementations are transparent from client side.

### Consequence

**Separate the changing part from non-changing part.** Now, the sub-class can reference the specific behavior on demand, e.g., **DecoyDuck** reference the **FlyNoWay**. 

**Open closed principle**. if we want to add new strategy or update certain strategy, we just change the implementation at only one place.

Isolate the implementation details of an algorithm from the code that uses it. Redhead duck face to interface only and don't have to know how the fly behavior implements.

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

