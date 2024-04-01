---
layout:     post
title:      Proxy Pattern
subtitle:   
date:       2022-12-18
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

Imagine that you are a boss of gumball machine company, and you want to monitor the states of all deployed gumball machines remotely (these machines are placed other locations, instead of local). In this case, we can use remote proxy to control access to the real machines and help us to hide the newwork details.

Another case is that you are implementing an application that shows CD cover. When you click the specific CD, the viewer should load from network and show the corresponding CD cover. 
The only problem is, depending on the network load and the bandwidth of your connection, retrieving a CD cover might take a little time, so your application should display something while you are waiting for the image to load. 

An easy way to achieve this is through a virtual proxy. The virtual proxy can place the Icon, manage background loading, show "Loading, please wait ..." before loading. Once the image is loaded, the proxy delegates the display to the Icon.

### Definition

The **Proxy Pattern** provides a placeholder for another object to control access to it. A proxy allows you to **perform something either before or after the request** gets through to the original object.

### Applicability

**Remote Proxy**

Local execution of a remote service. This is when the service object is located on a remote server.

In this case, the proxy passes the client request over the network, handling all of the nasty details of working with the network.

**Virtual Proxy**

Lazy initialization. This is when you have a heavyweight service object that wastes system resources by being always up, even though you only need it from time to time.

Instead of creating the object when the app launches, you can delay the object’s initialization to a time when it’s really needed.

**Protection Proxy**

Access control. This is when you want only specific clients to be able to use the service object; for instance, when your objects are crucial parts of an operating system and clients are various launched applications.

The proxy can pass the request to the service object only if the client’s credentials match some criteria.

### Structure

![img](/img/DesignPattern/proxy.png)

### Participants

First we have a **Subject**, which provides an interface for the **RealSubject** and the **Proxy**. By implementing the same interface, the **Proxy** can be substituted for the **RealSubject** anywhere it occurs.

The **RealSubject** is the object that does the real work. It’s the object that the **Proxy** represents and controls access to.

The **Proxy** holds a reference to the **RealSubject**. In some cases, the **Proxy** may be responsible for creating and destroying the **RealSubject**. Clients interact with the **RealSubject** through the **Proxy**. Because the **Proxy** and **RealSubject** implement the same interface (**Subject**), the **Proxy** can be substituted anywhere the subject can be used. The **Proxy** also controls access to the **RealSubject**; this control may be needed if the **Subject** is running on a remote machine, if the **Subject** is expensive to create in some way or if access to the subject needs to be protected in some way.

### Consequence

You can control the service object without clients knowing about it.

You can manage the lifecycle of the service object when clients don’t care about it.

The proxy works even if the service object isn’t ready or is not available.

**Open Closed Principle.** You can introduce new proxies without changing the service or clients.

### Implementation

We have prepared two code samples to illustrate the Proxy Pattern. One is for remote proxy, another is for protection proxy.

For remote proxy, we take the gumball machine monitor as an example, use the **Java RMI** to achieve remote procedure call. See [here](https://github.com/haoozhang/Head-First-Design-Pattern/tree/main/RemoteProxy) for complete code sample for remote proxy.

```bash
#Run Steps 

$ cd target/classes
$ rmic Context  # use rmic to produce stub for remote service imple
$ rmiregistry 5001  # run rmiregistry
# (new terminal)
$ java GumballMachineTest  # register remote instance
# (new terminal)
$ java GumballMonitorTest  # run client application
```

```java
// Remote interface
public interface GumballMachineRemote extends Remote {

    public int getCount() throws RemoteException;

    public String getLocation() throws RemoteException;

    public State getCurrentState() throws RemoteException;
}

// Remote implementation
public class Context extends UnicastRemoteObject implements GumballMachineRemote {

    // other properties and methods
    // ....

    @Override
    public State getCurrentState() {
        return CurrentState;
    }

    @Override
    public int getCount() throws RemoteException {
        return Count;
    }

    @Override
    public String getLocation() throws RemoteException {
        return Location;
    }
}

// register remote service
public class GumballMachineTest {

    public static void main(String[] args) {
        try {
            GumballMachineRemote gumballMachine = new Context(10, "sane");
            Naming.rebind("rmi://localhost:5001/sane", gumballMachine);
            System.out.println("Registered sane");

            GumballMachineRemote gumballMachine2 = new Context(20, "boulder");
            Naming.rebind("rmi://localhost:5001/boulder", gumballMachine2);
            System.out.println("Registered boulder");
        } catch (RemoteException | MalformedURLException e) {
            e.printStackTrace();
        }
    }
}
```

```java
// client side: get instance and use it
public class GumballMonitorTest {

    public static void main(String[] args) {
        String[] locations = {
            "rmi://localhost:5001/sane",
            "rmi://localhost:5001/boulder",
        };

        GumballMonitor[] monitors = new GumballMonitor[locations.length];
        for (int i = 0; i < monitors.length; i++) {
            try {
                // look up instance from rmi
                GumballMachineRemote machine = (GumballMachineRemote) Naming.lookup(locations[i]);
                monitors[i] = new GumballMonitor(machine);
                System.out.println(monitors[i]);
            } catch (RemoteException e) {
                e.printStackTrace();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < monitors.length; i++) {
            monitors[i].Report();
        }
    }
}
```

Then for protection proxy, let's say that we need a matchmaking service to help boys and girls find their Mr./Ms. Right. You have an idea that the system should provide a "Hot or Not” feature where participants can rate each other.

This is a perfect example of where we might be able to use a Protection Proxy, because we should control access to the personal profile based on their ownship. For an owner, he/she can view and edit personal profile, but can't set the rating. But for the others, they can browse the others' profile and set the rating, but can't set others' profile.

We use the **Java Dynamic Proxy** to implement it. See [here](https://github.com/haoozhang/Head-First-Design-Pattern/tree/main/DynamicProxy) for complete code sample.

![img](/img/DesignPattern/proxy_dynamic.png)

### Related Patterns

**Decorator** and **Proxy** have similar structures, but very different intents. Both patterns are built on the composition principle, where one object is supposed to delegate some of the work to another. The difference is that a **Proxy** usually manages the lifecycle of its service object on its own, whereas the composition of **Decorators** is always controlled by the client.