---
layout:     post
title:      Iterator Pattern
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

Let's say that you have two menus that differ from implementations, one is using the Array to represent menu items, another is using ArrayList. What’s the problem with having two different menus? 
Let’s try implementing a client that uses the two menus. If we want to print all menu items provided by these two menus, we have to loop through the Array to print all, then loop through the ArrayList to print all, just look like this.

```java
for (int i = 0; i < array.length; i++) {
    MenuItem menuItem = array[i];
    System.out.println(menuItem);
}
for (int i = 0; i < arraylist.size(); i++) {
    MenuItem menuItem = arraylist.get(i);
    System.out.println(menuItem);
}
```

These two menus don’t want to change their implementations because it would mean rewriting a lot of code that is in each respective menu class. But if one of them doesn’t give in, the client that uses these two menus is going to be hard to maintain and extend.

It would really be nice if we could find a way to allow them to implement the same interface for their menus, so that we can get rid of the multiple loops required to iterate over both menus, just like this.

```c#
Iterator iterator = array.createIterator();
while(iterator.hasNext()) {
    MenuItem menuItem = iterator.next();
}

Iterator iterator = arraylist.createIterator();
while(iterator.hasNext()) {
    MenuItem menuItem = iterator.next();
}
```

Well, it looks like encapsulating iteration, as you may know, it is a Design Pattern called Iterator Pattern.

### Definition

The **Iterator Pattern** provides a way to access the elements of an aggregate object
sequentially without exposing its underlying representation.

### Applicability

When your collection has a complex data structure, but you want to hide its complexity from clients.

### Structure

![img](/img/DesignPattern/iterator.png)

### Participants

Declare the Iterator interface, which contains the methods for checking the next position and fetching next element, or other methods like remove current element.

Implement concrete iterator classes to manage certain single collection instance. Usually, this collection instance is established via the iterator’s constructor.

Declare the collection interface and describe a method for fetching iterators. The return type should be the type of iterator interface.

Implement the collection interface. The main idea is to provide the client with a shortcut for creating iterators.

Go over the client code to replace all of the collection traversal code with the use of iterators.

### Consequence

Single Responsibility Principle. You can clean up the client code and the collections by extracting bulky traversal algorithms into separate classes.

Open Closed Principle. You can implement new types of collections and iterators and pass them to existing code without breaking anything.

### Implementation

```c#
public interface IIterator
{
    public bool HasNext();

    public object Next();
}

public class DinerMenuIterator : IIterator
{
    private readonly MenuItem[] MenuItems;

    private int _position = 0;

    public DinerMenuIterator(MenuItem[] menuItems)
    {
        MenuItems = menuItems;
    }
    
    public bool HasNext()
    {
        if (_position >= MenuItems.Length || MenuItems[_position] == null)
        {
            return false;
        }
        else
        {
            return true;
        }
    }

    public object Next()
    {
        return MenuItems[_position++];
    }
}
```

```c#
public class DinerMenu : IMenu
{
    private static readonly int MAX_ITEMS = 6;

    private int NumberOfItems = 0;

    public MenuItem[] MenuItems;

    public DinerMenu()
    {
        MenuItems = new MenuItem[MAX_ITEMS];

        AddItem("Soup of day", "potato salard", true, 3.29);
        AddItem("Hot dog", "hot dog with cheese", false, 4.99);
        AddItem("Vegetarian BLT", "tomato with wheat", true, 2.49);
    }

    public void AddItem(string name, string description, bool vegetarian, double price)
    {
        var menuItem = new MenuItem(name, description, vegetarian, price);
        if (NumberOfItems >= MAX_ITEMS)
        {
            Console.WriteLine("Menu is full! Can't add more items!");
        }
        else
        {
            MenuItems[NumberOfItems++] = menuItem;
        }
    }

    public IIterator CreateIterator()
    {
        return new DinerMenuIterator(MenuItems);
    }
}
```

```c#
// client
public class Waitress
{
    public IMenu PancakeHouseMenu { get; set; }

    public IMenu DinerMenu { get; set; }

    public Waitress(IMenu pancakeHouseMenu, IMenu dinerMenu)
    {
        PancakeHouseMenu = pancakeHouseMenu;
        DinerMenu = dinerMenu;
    }

    public void PrintMenu()
    {
        IIterator pancakeHouseIterator = PancakeHouseMenu.CreateIterator();
        IIterator dinerIterator = DinerMenu.CreateIterator();

        Console.WriteLine("==> Breakfast");
        PrintMenu(pancakeHouseIterator);
        Console.WriteLine("==> Lunch");
        PrintMenu(dinerIterator);
    }

    private void PrintMenu(IIterator iterator)
    {
        while (iterator.HasNext())
        {
            var menuItem = iterator.Next() as MenuItem;
            Console.WriteLine("Name: " + menuItem?.Name);
            Console.WriteLine("Description: " + menuItem?.Description);
            Console.WriteLine("Price: " + menuItem?.Price);
        }
    }
}
```

See [here](https://github.com/haoozhang/Head-First-Design-Pattern/tree/main/IteratorPattern) for complete code sample.

### Known Uses

Using **java.util.Iterator** to simplify above code. Due to *ArrayList* has its own *iterator()* method, we use this existing one, instead of creating by ourselves.

See [here](https://github.com/haoozhang/Head-First-Design-Pattern/tree/main/IteratorPattern_Java) for complete java implementation.

```java
public class PancakeHouseMenu implements Menu
{
    public List<MenuItem> MenuItems;

    public PancakeHouseMenu()
    {
        MenuItems = new ArrayList<>();

        AddItem("Regular Pancake Breakfast", "fired eggs, sausage", false, 2.99);
        AddItem("Blueberry Pancakes", "fresh blueberry", true, 1.99);
        AddItem("K&B Breakfast", "scrambled eggs, toast", true, 3.99);
    }

    public void AddItem(String name, String description, boolean vegetarian, double price)
    {
        var menuItem = new MenuItem(name, description, vegetarian, price);
        MenuItems.add(menuItem);
    }

    public Iterator CreateIterator()
    {
        return MenuItems.iterator();
    }
}
```

### Why not merge concrete iterator into collection?

One point needs to be called out that perviously I have a question that why not let the collection instance implement the iterator interafce directly, instead, we assign a concrete iterator for the collection instance.

The answer is **Single Responsibility Principle**. If we let the collection instance manage two responsibilities (manage collection, and loop through elements), we will give the clss two reasons to change. When the change does, it's going to affect two aspects of your design.

Every responsibility of a class is an area of potential change. More than one responsibility means more than one area of change. This principle guides us to keep each class to a single responsibility.
