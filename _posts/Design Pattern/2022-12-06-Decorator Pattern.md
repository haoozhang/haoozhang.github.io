---
layout:     post
title:      Decorator Pattern
subtitle:   
date:       2022-12-06
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

Imagine that you design an order system for the coffee shop. They provide several kinds of coffee for sales, also the customer can ask for several condiments like milk, soy, and mocha. The order system charges a bit for each of these. Maybe here is your first attempt.

![img](/img/DesignPattern/decorator_motivation_1.png)

This design has too many similar classes, which causes many problems. \
First, we can't add a new brand of beverage conveniently (lack of extendability). Once a new beverage needs to be added, we have to add all kinds of subclasses associated with condiments. \
Second, we can't update the existing classes quickly. Suppose that we need to update the price of certain beverage or condiment, we need to update all involving classes. 

Why we need all these classes? Can we use inheritance to define beverage and use some bool methods to describe each kind of condiment? just like this.

![img](/img/DesignPattern/decorator_motivation_2.png)

The drawback of this design is that it's not align with the **Open/Closed Design Principle** (open for extension, and closed for modification). The goal is to extend the behavior without modifying existing code. But now we have to update the super class if we add more condiment.
Besides, what if the customer want to buy a cup of coffee with double milk? It can't be achieved conveniently for current design.

### Definition

The **Decorator Pattern** attaches **additional responsibilities to an object** dynamically.
Decorators provide a more flexible alternative than subclasses for extending functionality.

### Applicability

When you need to be able to **assign extra behaviors to objects** at runtime without breaking the code that uses these objects.

### Structure

![img](/img/DesignPattern/decorator.png)

### Participants

**ConcreteComponent** and **Decorator** have the same super type **Component**.

You can use one or more decorators to wrap an object.

The decorator adds its own behavior either before or after decorated component's behavior to do the rest of the job.

### Consequence

Let’s work our beverages order system into this framework.

![img](/img/DesignPattern/decorator_consequence.png)

### Implementation

```c#
public interface IBeverage
{
    public string Description { get; }

    public double Cost();
}

public class Coffee : IBeverage
{
    public string Description { get; set; } = "Coffee";

    public double Cost()
    {
        return 6.66;
    }
}
```

```c#
public interface ICondimentDecorator : IBeverage
{
}

public class Milk : ICondimentDecorator
{
    public string Description
    {
        get => "Milk, " + Beverage.Description;
    }

    public IBeverage Beverage { get; set; }

    public Milk(IBeverage beverage)
    {
        Beverage = beverage;
    }
    
    public double Cost()
    {
        return Beverage.Cost() + 1.01;
    }
}

public class Mocha : ICondimentDecorator
{
    public string Description
    {
        get => "Mocha, " + Beverage.Description;
    }

    public IBeverage Beverage { get; set; }

    public Mocha(IBeverage beverage)
    {
        Beverage = beverage;
    }
    
    public double Cost()
    {
        return Beverage.Cost() + 2.01;
    }
}
```

Then, use the following code to test it.

```c#
IBeverage beverage = new Tea();
Console.WriteLine($"{beverage.Description}'s cost:{beverage.Cost()}");

beverage = new Milk(beverage);
beverage = new Milk(beverage);
beverage = new Soy(beverage);
Console.WriteLine($"{beverage.Description}'s cost:{beverage.Cost()}");
```

See [here](https://github.com/haozhangms/Head-First-Design-Pattern/tree/main/Starbuzz) for complete code sample.

### Known Uses

Decorating implementation in Java I/O.

![img](/img/DesignPattern/decorator_knownuse.png)

### Related Patterns

