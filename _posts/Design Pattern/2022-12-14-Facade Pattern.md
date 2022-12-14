---
layout:     post
title:      Adapter Pattern and Facade Pattern
subtitle:   
date:       2022-12-14
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

Let's say that you have built your own home theater with a DVD player, a projection system and an automated screen. Now it's time to enjoy a movie...

You need to perform a few tasks: 1) put the screen down; 2) turn the projector on; 3) set the projector input to DVD; 4) turn the DVD player on; 5) start the DVD player playing.

But there's more... when the movie is over, how do you turn everything off? So let’s see how the Facade Pattern can get us out of this mess so we can enjoy the movie...

### Definition

The **Facade Pattern** provides a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

### Applicability

When you need to have a limited but straightforward interface to a complex subsystem.

### Structure

![img](/img/DesignPattern/facade.png)

### Participants

The **Facade** provides convenient access to a particular part of the subsystem’s functionality. 

The **Complex Subsystem** consists of dozens of various objects. To make them all do something meaningful, you have to dive deep into the subsystem’s implementation details.

### Consequence

isolate your code from the complexity of a subsystem.

### Implementation

```c#
public class HomeTheaterFacade
{
    DvdPlayer dvd;
    Projector projector;
    Screen screen;

    public HomeTheaterFacade(DvdPlayer dvd, Projector projector, Screen screen)
    {
        this.dvd = dvd;
        this.projector = projector;
        this.screen = screen;
    }

    public void WatchMovie(string movie) 
    {
        screen.Down();
        projector.On();
        projector.WideScreenMode();
        dvd.On();
        dvd.SetMovie(movie);
    }

    public void EndMovie()
    {
        screen.Up();
        projector.Off();
        dvd.Stop();
        dvd.Eject();
        dvd.Off();
    }
}
```

### Known Uses



### Related Patterns


