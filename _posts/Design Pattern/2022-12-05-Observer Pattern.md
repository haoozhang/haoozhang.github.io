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

### Motivation

Imagine that you have two types of objects: a **Customer** and a **Store**. The customer is very interested in a particular brand of product (say, iPhone 14), which should become available in the store very soon.

The customer could visit the store every day and check product availability. But it will cost their time very much. On the other hand, the store could send the notification to all consumers when the iPhone 14 comes, but it will disturb the consumers who are not interested.

![img](/img/DesignPattern/observer_motivation.png)

Now in this case, we need the Observer Pattern. The consumers who interested the particular brand of product can subscribe it, instead of querying it frequently. Then the store will notify only those subscribed consumers (not all) when the product comes.

### Definition

The **Observer Pattern** defines a **one-to-many** dependency between objects so that when one object changes state, all of its dependents are notified and updated automatically.

### Applicability

When state changes of one object may require changing other objects. You can often experience this problem when working with graphical user interface. For example, you created custom button classes, and you want to let the clients hook some custom code to your buttons so that it fires whenever a user presses a button.

### Structure

![img](/img/DesignPattern/observer.png)

### Participants

**Subject** is an interface, it has only three methods: register/remove/notify observer.

**ConcreteSubject** is the implementation of **Subject** interface. In addition to the three methods, it has many observers and methods for getting or setting state, so that it can notify observers when state reaches certain threshold.

**Observer** is also an interface, it has only one method: update. **ConcreteObject** is the implementation, which is the specific observer. It customize own logic in *update* method.

### Consequence

Decoupled the subject with observers. You can introduce new observer without having to change the subject (and vice versa).

Note that here the observers are **notified in random order**.

### Implementation

Here we take the weather station as an example. Once the weather data produced, the different dashboards will be notified and updated.

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
            MeasurementsChanged();
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

    // this method is to notify observer when state data hit the threshold
    private void MeasurementsChanged()
    {
        NotifyObserver();
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

Java has built-in support for observer pattern with **Observable** and **Observer**.
+ For an **Object** to become an observer. As usual, implement the **Observer** interface, and call add/remove observer methods.
+ For the **Observable** to send notifications. There it is a two-step process: first call the *setChanged()* method to signify that the state has changed, then call *notifyObservers()* or *notifyObservers(Object arg)* method.
+ For an **Observer** to receive notifications. If you want to "push" data to the observers, you can pass the data as a data object to the *notifyObservers(arg)* method. Else, you call *notifyObservers()* method and the **Observer** has to "pull" data from **Observable** object via calling some getter methods.

Both JavaBeans and Swing also provide their own implementations of this pattern, e.g., **PropertyChangeListener** interface in JavaBeans.

### Related Patterns

