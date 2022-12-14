---
layout:     post
title:      Template Method Pattern
subtitle:   
date:       2022-12-15
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

Some people can't live without coffee; some people can't live without tea. Tea and coffee are made in very similar ways: 1) Boil some water; 2) Brew coffee / tea in boiling water; 3) Pour it in cup; 4) Add some condiments (milk / lemon).

Let's create the coffee and tea classes.

```c#
public class Coffee
{
    public void PrepareRecipe()
    {
        BoilWater();
        BrewCoffee();
        PourInCup();
        AddSugarAndMilk();
    }

    public void BoilWater()
    {
        Console.WriteLine("Boil Water....");
    }

    public void BrewCoffee()
    {
        Console.WriteLine("Dripping Coffee through filter....");
    }

    public void PourInCup()
    {
        Console.WriteLine("Pour In Cup....");
    }

    public void AddSugarAndMilk()
    {
        Console.WriteLine("Adding Sugar and Milk....");
    }
}

public class Tea
{
    public void PrepareRecipe()
    {
        BoilWater();
        SteepTea();
        PourInCup();
        AddLemon();
    }

    public void BoilWater()
    {
        Console.WriteLine("Boil Water....");
    }

    public void SteepTea()
    {
        Console.WriteLine("Steeping the tea....");
    }

    public void PourInCup()
    {
        Console.WriteLine("Pour In Cup....");
    }

    public void AddLemon()
    {
        Console.WriteLine("Adding Lemon....");
    }
}
```

Considering that there are some similar code between two classes, you can redesign it like this.

![img](/img/DesignPattern/template_motivation.png)

So what else do coffee and tea have in common? let's start with the recipes. Notice that both recipes follow the same algorithm. So, can we find a way to abstract *prepareRecipe()* too? Yes! We can leverage template method to achieve it.

### Definition

The **Template Method Pattern** defines the skeleton of an algorithm in a method, *deferring some steps* to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm’s structure.

### Applicability

When you have several classes that contain almost identical algorithms with some minor differences, you can also pull up the steps with similar implementations into a superclass, eliminating code duplication. 

### Structure

![img](/img/DesignPattern/template.png)

### Participants

**Abstract Class** contains template method and abstract primitive operations that are invoked in template method.

**Concrete Class** implements the primitive operations required by the template method.

### Consequence

Let the subclass customize only particular part of a large algorithm, make them less affected by changes that happen to other parts of the algorithm.

Pull the duplicate code into a superclass.

### Implementation

```c#
public abstract class CaffeineBeverage
{
    public void PrepareRecipe()
    {
        BoilWater();

        Brew();

        PourInCup();

        // Here is a hook, which is declared in the abstract class, but only given an empty or default implementation. 
        // This gives subclasses the ability to “hook into” the algorithm at various points
        if (CustomerWantCondiments())
        {
            AddCondiments();
        }
    }

    public void BoilWater()
    {
        Console.WriteLine("Boil Water....");
    }

    protected abstract void Brew();

    public void PourInCup()
    {
        Console.WriteLine("Pour In Cup....");
    }

    protected abstract void AddCondiments();

    public virtual bool CustomerWantCondiments()
    {
        return true;
    }
}
```

```c#
public class Coffee : CaffeineBeverage
{
    protected override void Brew()
    {
        Console.WriteLine("Dripping Coffee through filter....");
    }

    protected override void AddCondiments()
    {
        Console.WriteLine("Adding Sugar and Milk....");
    }

    public override bool CustomerWantCondiments()
    {
        return false;
    }
}

public class Tea : CaffeineBeverage
{
    protected override void Brew()
    {
        Console.WriteLine("Steeping the tea....");
    }

    protected override void AddCondiments()
    {
        Console.WriteLine("Adding Lemon....");
    }

    public override bool CustomerWantCondiments()
    {
        return true;
    }
}
```

See [here](https://github.com/haozhangms/Head-First-Design-Pattern/tree/main/TemplatePattern) for complete code sample.

### Known Uses


### Related Patterns

**Factory Method Pattern** defer instantiation to subclasses. \
**Template Method Pattern** deferring some steps of a large algorithm to subclasses.