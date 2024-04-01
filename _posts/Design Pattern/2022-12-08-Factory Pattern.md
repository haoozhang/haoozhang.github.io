---
layout:     post
title:      Factory Pattern
subtitle:   
date:       2022-12-08
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

Imagine that you have two pizza stores, you want them sale different styles of pizza based on their location (New York, Chicago), while you want the same producing process (to ensure the quality of product). So you may implement it like this.

![img](/img/DesignPattern/factory_motivation.png)

This design doesn't meet the **Open Closed Principle**. Suppose that we want to add one pizza type or pizza store, we have to update the method implementation.

### Definition

The **Factory Method Pattern** defines an interface for creating an object in super class, but lets subclasses decide which class to instantiate. **Factory Method** lets a class defer instantiation to subclasses.

### Applicability

When you don't need to know the exact types, and just need an object to work with the rest of procedures.

### Structure

![img](/img/DesignPattern/factory.png)

### Participants

**Creator** is the factory, where the return type of the **factoryMethod** is the abstract **Product** type, not **ConcreteProduct**.

**ConcreteCreator** is the specific subclass that is responsible for creating specific type of product.

### Consequence

The **Factory Method** separates product construction code from the code that actually uses the product. Therefore it’s easier to extend the product construction code independently from the rest of the code.

![img](/img/DesignPattern/factory_consequence.png)

### Implementation

```c#
public abstract class PizzaStoreFactory
{
    protected abstract Pizza CreatePizza(string type);

    public Pizza OrderPizza(string type)
    {
        Pizza pizza = CreatePizza(type);

        pizza.Prepare();
        pizza.Bake();
        pizza.Cut();
        pizza.Box();

        return pizza;
    }
}

public class ChicagoPizzaStore : PizzaStoreFactory
{
    protected override Pizza CreatePizza(string type)
    {
        switch (type)
        {
            case "cheese":
                return new ChicagoStyleCheesePizza();
            case "clam":
                return new ChicagoStyleClamPizza();
            case "veggie":
                return new ChicagoStyleVeggiePizza();
            default:
                return new ChicagoStyleCheesePizza();
        }
    }
}
```

```c#
public abstract class Pizza
{
    public string Name { get; set; }
    
    public string Dough { get; set; }
    
    public string Sauce { get; set; }

    public List<string> Toppings = new List<string>();

    public void Prepare() 
    {
        Console.WriteLine("Preparing...");
    }

    public void Bake()
    {
        Console.WriteLine("Baking...");
    }

    public void Cut()
    {
        Console.WriteLine("Cutting...");
    }

    public void Box()
    {
        Console.WriteLine("Boxing...");
    }
}

public class ChicagoStyleCheesePizza : Pizza
{
    public ChicagoStyleCheesePizza()
    {
        Name = "Chicago Style Cheese Pizza";
        Toppings.Add("cheese");
    }
}
```

Further more, if we add a ingredient factory to create different ingredients, we will introduce the **Abstract Factory Pattern**, which lets a family of classes defer instantiation to subclasses. The result just look like.

![img](/img/DesignPattern/factory_abstract_consequence.png)

See [here](https://github.com/haoozhang/Head-First-Design-Pattern/tree/main/PizzaFactory) for complete code sample.

### Abstract Factory

The **Abstract Factory Pattern** provides an interface for creating families of related or dependent objects without specifying their concrete classes.

The only difference between these two patterns is that the abstract factory defines a family of classes to be initialized by subclasses.

![img](/img/DesignPattern/factory_abstract.png)


