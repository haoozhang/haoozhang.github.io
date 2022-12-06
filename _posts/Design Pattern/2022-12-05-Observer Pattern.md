---
layout:     post
title:      Observer Pattern
subtitle:   
date:       2022-12-05
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Definition

The **Observer Pattern** defines a *one-to-many* dependency between objects so that when one object changes state, all of its dependents are notified and updated automatically.

### Motivation

Imagine that you have two types of objects: a **Customer** and a **Store**. The customer is very interested in a particular brand of product (say, iPhone 14), which should become available in the store very soon.

The customer could visit the store every day and check product availability. But it will cost their time very much. On the other hand, the store could send the notification to all consumers when the iPhone 14 comes, but it will disturb the consumers who are not interested.

Now, we need the observer pattern. The consumers who interested the particular brand of product can subscribe it, instead of querying it every day. The store will notify only those subscribed consumers (not all) when the product comes.

### Applicability

When changes to the state of one object may require changing other objects.

### Structure

![img](/img/DesignPattern/observer.png)

### Participants

**Subject** is an interface, it has only three methods: register/remove/notify observer. **ConcreteSubject** is the implementation of **Subject** interface. In addition to the three methods, it has many observers and methods for getting or setting state, so that it can notify observers when state reaches certain threshold.

**Observer** is also an interface, it has only one method: update. **ConcreteObject** is the implementation, which is the specific observer. It customize own logic in update method.

### Consequence

Decouple subject with observers. You can introduce new observer without having to change the subject (and vice versa).

Note that the observers are notified in random order.

### Implementation

```c#
public interface ISubject
{
    public void RegisterObserver(IObserver observer);

    public void RemoveObserver(IObserver observer);

    public void NotifyObserver();
}

public class WeatherStation : ISubject
{
    private readonly List<IObserver> _observers;

    private WeatherData _weatherData;
    
    public WeatherData WeatherData
    {
        get => _weatherData;
        set
        {
            _weatherData = value;
            NotifyObserver();
        }
    }

    public WeatherStation()
    {
        _observers = new List<IObserver>();
    }

    public void RegisterObserver(IObserver observer)
    {
        _observers.Add(observer);
    }

    public void RemoveObserver(IObserver observer)
    {
        _observers.Remove(observer);
    }

    public void NotifyObserver()
    {
        foreach (var observer in _observers)
        {
            observer.Update(WeatherData);
        }
    }
}
```

```c#
public interface IObserver
{
    public void Update(WeatherData weatherData);
}

public class CurrentConditionDisplay : IObserver
{
    private WeatherData _weatherData;

    private ISubject WeatherStation;

    public CurrentConditionDisplay(ISubject subject)
    {
        _weatherData = new WeatherData();
        
        WeatherStation = subject;
        WeatherStation.RegisterObserver(this);
    }
    
    public void UnregisterObserver()
    {
        WeatherStation.RemoveObserver(this);
    }
    
    public void Update(WeatherData weatherData)
    {
        _weatherData = weatherData;
        Display();
    }

    public void Display()
    {
        Console.WriteLine($"Current condition: {_weatherData.Temperature}, {_weatherData.Humidity}, {_weatherData.Pressure} ");
    }
}
```

See [here](https://github.com/haozhangms/Head-First-Design-Pattern/tree/main/WeatherObserver) for complete code sample.

### Known Uses



### Related Patterns

