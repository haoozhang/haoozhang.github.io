---
layout:     post
title:      State Pattern
subtitle:   
date:       2022-12-17
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

The State Pattern is closely realted to the concept of a **Finite-State Machine** (有限状态机). 
The main idea is that at any given moment, there is a finite number of states which a program can be in. Within any unique state, the program behaves differnetly, and can be switched from one state to another. These switching rules, called transitions, are also finite and predetermined.

Imagine that we need to control the state changes for a gumball machine, make it work normally as follows.

![img](/img/DesignPattern/state_motivation1.png)

Our first implementation may be look like follows. We create an instance variable to hold the current state and define each state. Then we gather up all the actions, and for each action, we create a method that use conditional statements to determine appropriate behavior.

```c#
public class GumballMachine
{
    static int SOLD_OUT = 0;
    static int NO_QUARTER = 1;
    static int HAS_QUARTER = 2;
    static int SOLD = 3;

    int state = SOLD_OUT;

    public void InsertQuarter() 
    {
        if (state == HAS_QUARTER) 
        {
            Console.WriteLine("You have inserted one quarter, can't insert another one!");
        }
        else if (state == NO_QUARTER) 
        {
            Console.WriteLine("You insert a quarter!");
            state = HAS_QUARTER;
        }
        else if (state == SOLD_OUT) 
        {
            Console.WriteLine("Ball sold out, you can't insert quarter!");
        }
        else if (state == SOLD) 
        {
            Console.WriteLine("You have inserted one quarter, can't insert another one!");
        }
    }

    // other methods that use conditional statements
}
```

This feels like a nice solid design using a well-thought-out methodology. But you know it was coming... a change request! 

Now the manufacturer decide that the machine can release a free gumball one in ten. So we have to update the state machine and implementation to meet this requirement. Specifically, we have to add one more conditional check for new state in each method, which is not align with **Open Closed Principle**. 

![img](/img/DesignPattern/state_motivation2.png)

The well-thought-out solution is not extendable, we have to redesign this implementation (aims to encapsulate charging part). We had better place the behaviors of each state into separate classes, so that each state just manage its own actions. Therefore, the State Pattern comes up.

### Definition

The **State Pattern** allows an object to alter its behavior when its internal state changes. It appears as if the object changed its class.

### Applicability

When you have an object that behaves differently depending on its current state. The pattern suggests that you extract all state-specific code into a set of distinct classes. As a result, you can add new states or change existing ones independently of each other, reducing the maintenance cost.

### Structure

![img](/img/DesignPattern/state.png)

### Participants

**State** interface defines a common interface for all concrete state, which contains all actions.

**ConcreteState** implements the **State** interface, and customize its own state's behavior for each action.

**Context** hold the reference to current state, and declares all kinds of states. To switch the state of the context, just set the current state as target state. This is what said "object changed its class" in definition.

### Consequence

**Open Closed Principle.** Introduce new states without changing existing state classes or the context.

**Single Responsibility Principle.** Organize the code related to particular states into separate classes.

### Implementation

```c#
public interface IState
{
    public void InsertQuarter();

    public void EjectQuarter();

    public void TurnCrank();

    public void Dispense();
}

public class NoQuarterState : IState
{
    private Context Context { get; }

    public NoQuarterState(Context context)
    {
        Context = context;
    }
    
    public void InsertQuarter()
    {
        Console.WriteLine("You insert a quarter!");
        Context.CurrentState = Context.HasQuarterState;
    }

    public void EjectQuarter()
    {
        Console.WriteLine("You haven't insert a quarter yet!");
    }

    public void TurnCrank()
    {
        Console.WriteLine("You should insert a quarter first and then turn crank!");
    }

    public void Dispense()
    {
        Console.WriteLine("You need to insert a quarter first!");
    }
}
```

```c#
public class Context
{
    public NoQuarterState NoQuarterState { get; }
    
    public HasQuarterState HasQuarterState { get; }
    
    public SoldState SoldState { get; }
    
    public SoldOutState SoldOutState { get; }
    
    public WinnerState WinnerState { get; }

    public IState CurrentState { get; set; }

    public int Count;

    public Context(int count)
    {
        NoQuarterState = new NoQuarterState(this);
        HasQuarterState = new HasQuarterState(this);
        SoldState = new SoldState(this);
        SoldOutState = new SoldOutState(this);
        WinnerState = new WinnerState(this);
        
        CurrentState = NoQuarterState;
        Count = count;
    }

    public void InsertQuarter()
    {
        CurrentState.InsertQuarter();
    }

    public void EjectQuarter()
    {
        CurrentState.EjectQuarter();
    }

    public void TurnCrank()
    {
        CurrentState.TurnCrank();
        CurrentState.Dispense();
    }

    public void ReleaseBall()
    {
        Console.WriteLine("Released a ball!");
        Count--;
    }
}
```

See [here](https://github.com/haozhangms/Head-First-Design-Pattern/tree/main/StatePattern) for complete code sample.

### Known Uses


### Related Patterns

As you may remember that the structures of **Strategy Pattern** and **State Pattern** are exactly same, the difference between them is about their intention.

With the State Pattern, we have a set of behaviors encapsulated in state objects; the current state changes across the set of state objects to reflect the internal state of the context. The client usually knows very little about the state objects.

But with Strategy Pattern, the client usually specifies the strategy object that the context is composed with.