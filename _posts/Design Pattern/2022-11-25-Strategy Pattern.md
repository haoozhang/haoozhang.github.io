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

### Motivation

Let's say that we have different kinds of ducks (redhead duck, mallard duck, decoy duck and etc), and they have fly behavior and quack behavior. We want to design a duck simulation game with above elements. The intuitive solution is to define a base duck class and then let different kinds of ducks extend and override the super duck class, where contains fly and quack behaviors, just like this.

![img](/img/DesignPattern/strategy_inheritance.png)

However, this solution with **inheritance** can't solve the problem very well, because some subclasses should not implement all methods defined in super class, e.g., decoy duck is also a kind of duck in our game, but it can't fly, so should not override the fly behavior defined in super class.

Then we may think out the solution using interface, that is, let those ducks implement the fly and quack interface to customize their behavior. For decoy duck, it doesn't implement fly behavior due to that's not applicable.

![img](/img/DesignPattern/strategy_interface.png)

But using **interface** also can't works very well, because of no code reuse, i.e., for every potenial change of any interface implementation later, we have to change in the involving subclasses.

> Design Principle: Find those possibly changing part, and separate them from non-chaning part. By doing this, we can encapsulate this changing part and modify it easily later, without breaking the non-changing part.

So let's separate the possibly changing duck's behaviors from the duck itself, one is fly behavior, the other is quack behavior. How to implement these two kinds of behaviors?

> Design Principle: Interface oriented programming, instead of implementation oriented programming.

With this goal, we use interface to implement these kinds of behaviors, and each specific behavior implements the specific interface. For example, `FlyWithWings` implements `FlyBehavior`, `Quack` implements `QuackBehavior`.

Similarly, due to the interface oriented programming, the duck base class should refer to the interface, instead of certain specific behavior. Using which one specific behavior is determined by the specific duck.

To sum up the above words, we have the complete structure as follows.

![img](/img/DesignPattern/strategy.png)

### Definition

The **Strategy Pattern** defines a family of algorithms that encapsulates each one (multiple implementations of one interafce), and makes them inter-changeable. Strategy lets the algorithm dependent from clients that use it, clients face to interface only.

### Participants

**Duck** is the base class, and all **xxDuck** are subclasses, which act as client role.

Encapsulated fly and quack behaviors are two family of algorithms, they have multiple inter-changeable implementations for each interface.

For clients, they interact with interface type (**FlyBehavior / QuackBehavior**), so those implementations are transparent from client side.

### Applicability

When you want to **use different variants of an algorithm within an object** and be able to switch from one algorithm to another during runtime.

When you have a lot of similar classes that only differ in the way they execute some behavior.

### Consequence

**Separate the changing part from non-changing part.** Now, the subclass can reference the specific behavior on demand, e.g., **DecoyDuck** reference the **FlyNoWay**. 

**Open Closed Principle**. If we want to add new duck or update certain behavior, we just change the implementation at only one place, without breaking existing code.

**Isolate the implementation details of an algorithm from the code that uses it**. Redhead duck face to interface only and don't have to know how the fly behavior implements.

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

See [here](https://github.com/haoozhang/Head-First-Design-Pattern/tree/main/SimUDuck) for complete code sample.


