---
layout:     post
title:      Command Pattern
subtitle:   
date:       2022-12-13
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

After a long walk through the city, you get to a nice restaurant and sit at the table by the window. A friendly waiter approaches you and quickly takes your order, writing it down on a piece of paper. The waiter goes to the kitchen and sticks the order on the wall. After a while, the order gets to the chef, who reads it and cooks the meal accordingly. The cook places the meal on a tray along with the order. The waiter discovers the tray, checks the order to make sure everything is as you wanted it, and brings everything to your table.

In this case, the paper order serves as a command. It remains in a queue until the chef is ready to serve it. The order contains all the relevant information required to cook the meal. It allows the chef to start cooking right away instead of running around clarifying the order details from you directly.

### Definition

The **Command Pattern** encapsulates a request as an object that contains all information about the request. This transformation lets you pass requests as a method arguments, delay or queue a request’s execution, and support undoable operations.

### Applicability

When you want to parametrize objects with operations. The command pattern can turn a specific method call into a stand-alone object. You can pass commands as method arguments, store them inside other objects, etc.

When you want to queue operations, schedule their execution, or execute them remotely. As with any other object, a command can be serialized, which means converting it to a string that can be written to a database. Later, the string can be restored as the initial command object. Thus, you can delay and schedule command execution. 

### Structure

![img](/img/DesignPattern/command.png)

### Participants

The **Invoker** must have a field for storing a reference to a command object. It triggers that command instead of sending the request directly to the receiver. Note that the invoker isn’t responsible for creating the command object. Usually, it gets a pre-created command from the client.

The **Command** interface usually declares just a single method for executing the command.

**Concrete Commands** implement various kinds of requests. A concrete command isn’t supposed to perform the work on its own, but rather to pass the call to one of the business logic objects. \
Parameters required to execute a method on a receiving object can be declared as fields in the concrete command. You can make command objects immutable by only allowing the initialization of these fields via the constructor.

The **Receiver** class contains some business logic and knows how to response the request.

The **Client** creates and configures concrete command objects. The client must pass all of the request parameters, including a receiver instance, into the command’s constructor. 

### Consequence

**Single Responsibility Principle.** You can decouple classes that invoke operations from classes that perform these operations.

**Open Closed Principle.** You can introduce new commands into the app without breaking existing client code.

You can implement undo/redo operation.

### Implementation

Let's implement a simple remote control. There are several on and off buttons in the remote controller, we can put defferent devices in each slot and control via the button. Also, you can click the undo button that undos the last operation.

```c#
// Command: holds execute() method 
public interface ICommand
{
    public void Execute();

    public void Undo();
}
```
```c#
// Concrete commands: call the receiver to perform the request
public class DoorOff : ICommand
{
    public Door Door { get; set; }

    public DoorOff(Door door)
    {
        Door = door;
    }
    
    public void Execute()
    {
        Door.Off();
    }

    public void Undo()
    {
        Door.On();
    }
}

public class DoorOn : ICommand
{
    public Door Door { get; set; }

    public DoorOn(Door door)
    {
        Door = door;
    }
    
    public void Execute()
    {
        Door.On();
    }

    public void Undo()
    {
        Door.Off();
    }
}

// Receiver: know how to perform the request
public class Door
{
    public void On()
    {
        Console.WriteLine("Door is on!");
    }

    public void Off()
    {
        Console.WriteLine("Door is off!");
    }
}
```

```c#
// Invoker: call the command, don't product command
public class RemoteControl
{
    public ICommand[] OnCommands { get; set; }
    
    public ICommand[] OffCommands { get; set; }
    
    public ICommand UndoCommand { get; set; }

    public RemoteControl()
    {
        OnCommands = new ICommand[4];
        OffCommands = new ICommand[4];

        ICommand noCommand = new NoOp();

        for (int i = 0; i < 4; i++)
        {
            OnCommands[i] = noCommand;
            OffCommands[i] = noCommand;
        }

        UndoCommand = noCommand;
    }

    public void OnButtonPressed(int slot)
    {
        OnCommands[slot].Execute();
        UndoCommand = OnCommands[slot];
    }

    public void OffButtonPressed(int slot)
    {
        OffCommands[slot].Execute();
        UndoCommand = OffCommands[slot];
    }

    public void UndoButtonPressed()
    {
        UndoCommand.Undo();
    }
}
```

```c#
// create command (contain receiver)
RemoteControl remoteControl = new RemoteControl();

Door door = new Door();
DoorOn doorOn = new DoorOn(door);
DoorOff doorOff = new DoorOff(door);

remoteControl.OnCommands[1] = doorOn;
remoteControl.OffCommands[1] = doorOff;

Console.WriteLine(remoteControl.ToString());
remoteControl.OnButtonPressed(0);
remoteControl.OffButtonPressed(0);
remoteControl.UndoButtonPressed();
```

See [here](https://github.com/haozhangms/Head-First-Design-Pattern/tree/main/CommandPattern) for complete code sample, in which also contains macro commands.

### Known Uses

Message Queue

Commands give us a way to package a piece of computation (a receiver and a set of actions) and pass it as a first-class object. Then it may be invoked long after, even be invoked by a different thread. We can take this scenario and apply it to many useful applications such as schedulers, thread pools, and job queues. \
Imagine a job queue: you add commands to the queue on one end, and on the other end sits a group of threads. Threads run the following script: they remove a command from the queue, call its *execute()* method, wait for the call to finish, then discard the command object and retrieve a new one.

### Related Patterns

