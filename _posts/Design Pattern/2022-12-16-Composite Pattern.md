---
layout:     post
title:      Composite Pattern
subtitle:   
date:       2022-12-16
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

Let's continue to take the menu as an example. In Iterator Pattern chapter, we discussed how to encapsulate the iteration over the menus with different implementations under the hood. 

Now, new requirement arrives. Imagine that we have to support not only multiple menus, but menus within menus. That is, it would be nice if we could make the dessert menu as an element of the certain menu, but that won't work as it is now implemented. It's time for change.

![img](/img/DesignPattern/composite_motivation.png)

### Definition

The **Composite Pattern** allows you to compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

Let's think about this in terms of our menus: this pattern give us a way to create a tree structure that can handle a nested group of menus and menu items in the same structure. By putting menus and items in the same structure we create a part-whole hierarchy; that is, a tree that is made of parts (menus and menu items).

### Applicability

When you have to implement a tree-like object structure.

When you want the client code to treat both simple and complex elements uniformly.

### Structure

![img](/img/DesignPattern/composite.png)

### Participants

**Component** declare an interface for all objects: composite and leaf node.

a **Leaf** class is to represent simple elements. A program may have multiple different leaf classes.

**Composite** is to represent complex elements. It provides an array field for storing references to sub-elements. The array must be able to store both leaves and composites, so it’s declared with the component interface type.

### Consequence

Work with complex tree structures more conveniently.

**Open/Closed Principle.** You can introduce new element types without breaking the existing code, which now works with the object tree.

### Implementation

```c#
// Component
public abstract class MenuComponent
{
    public abstract void Add(MenuComponent component);

    public abstract void Remove(MenuComponent component);

    public abstract MenuComponent GetChild(int i);

    public abstract void Print();

    public abstract IIterator CreateIterator();
}

// Composite
public class Menu : MenuComponent
{
    public List<MenuComponent> Components { get; set; }
    
    public string Name { get; set; }
    
    public string Description { get; set; }

    public Menu(string name, string description)
    {
        Components = new List<MenuComponent>();
        Name = name;
        Description = description;
    }
    
    public override void Add(MenuComponent component)
    {
        Components.Add(component);
    }

    public override void Remove(MenuComponent component)
    {
        Components.Remove(component);
    }

    public override MenuComponent GetChild(int i)
    {
        return Components[i];
    }

    public override void Print()
    {
        Console.WriteLine("Name: " + Name);
        Console.WriteLine("Description: " + Description);
        Console.WriteLine("============");

        IIterator iterator = CreateIterator();
        while (iterator.HasNext())
        {
            var component = iterator.Next() as MenuComponent;
            component?.Print();
        }
        
    }
    
    public override IIterator CreateIterator()
    {
        return new MenuComponentIterator(Components);
    } 
}

// Leaf
public class MenuItem : MenuComponent
{
    public string Name { get; set; }
    
    public string Description { get; set; }
    
    public bool Vegetarian { get; set; }
    
    public double Price { get; set; }

    public MenuItem(string name, string description, bool vegetarian, double price)
    {
        Name = name;
        Description = description;
        Vegetarian = vegetarian;
        Price = price;
    }
    
    public override void Add(MenuComponent component)
    {
        throw new InvalidOperationException("Can't add for menu item!");
    }

    public override void Remove(MenuComponent component)
    {
        throw new InvalidOperationException("Can't add for menu item!");
    }

    public override MenuComponent GetChild(int i)
    {
        throw new InvalidOperationException("Can't add for menu item!");
    }

    public override void Print()
    {
        Console.WriteLine("Name: " + Name);
        Console.WriteLine("Description: " + Description);
        Console.WriteLine("Price: " + Price);
    }

    public override IIterator CreateIterator()
    {
        return new NullIterator();
    }
}
```

See [here](https://github.com/haozhangms/Head-First-Design-Pattern/tree/main/CompositePattern) for complete code sample, where also uses the Iterator Pattern to loop through all nodes.

If you prefer java implementation (due to java built-in iterator), see [here](https://github.com/haozhangms/Head-First-Design-Pattern/tree/main/CompositePattern_Java) for complete java sample.

### Known Uses


### Related Patterns
