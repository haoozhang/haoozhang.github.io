---
layout:     post
title:      Compound Pattern 
subtitle:   
date:       2022-12-19
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Design Pattern
---

### Motivation

One of the best ways to use patterns is to let them interact with other patterns. The more you use patterns the more you're going to see them showing up together in your designs. We have a special name for a set of patterns that work together in a design: **Compound Pattern**, which can be applied over many problems.

### Definition

Patterns are often used together and combined within the same design solution. A compound pattern combines two or more patterns into a solution that solves a general problem.

### Applicability



### Structure


### Participants


### Consequence

### Implementation

We're going to rebuild our duck simulator from scratch and give it some interesting capabilities by using a bunch of patterns. Okay, let's get started...

**First, we'll create a Quackable interface.** \
Like we said, we’re starting from scratch. The Ducks are going to implement a Quackable interface.
```java
public interface Quackable {
    public void quack();
}
```

**Now, some Ducks that implement Quackable.** \
What goos is an interface without some classes to implement it? Time to create some concrete ducks.
```java
public class MallardDuck implements Quackable {
    @Override
    public void quack() {
        System.out.println("quack!");
    }
}

public class ReadheadDuck implements Quackable {
    @Override
    public void quack() {
        System.out.println("quack!");
    }
}

public class RubberDuck implements Quackable {
    @Override
    public void quack() {
        System.out.println("squeak!");
    }
}

public class DuckCall implements Quackable {
    @Override
    public void quack() {
        System.out.println("kayak!");
    }
}
```

**Okay, we've got our ducks; now all we need is a simulator.** \
Let's create a simulator that creates a few ducks and makes sure their quackers are working...
```java
public class DuckSimulator {

    public static void main(String[] args) {
        DuckSimulator simulator = new DuckSimulator();
        simulator.simulate();
    }
    
    public void simulate() {
        Quackable mallardDuck = new MallardDuck();
        Quackable redheadDuck = new RedheadDuck();
        Quackable duckCall = new DuckCall();
        Quackable rubberDuck = new RubberDuck();

        System.out.println("==== Duck Simulator ====");

        simulate(mallardDuck);
        simulate(redheadDuck);
        simulate(duckCall);
        simulate(rubberDuck);
    }

    public void simulate(Quackable duck) {
        duck.quack();
    }
}
```
It looks like everything is working; so far, so good. But we have not add patterns. \
**When ducks are around, geese can’t be far.** \
And geese can also make noise, so we need a goose adapter.

```java
public class Goose {
    public void honk() {
        System.out.println("honk!");
    }
}

public class GooseAdapter implements Quackable {
    private final Goose goose;

    public GooseAdapter(Goose goose) {
        this.goose = goose;
    }

    @Override
    public void quack() {
        goose.honk();
    }
}
```

**Now geese should be able to play in the simulator, too.**

```java
Quackable gooseAdapter = new GooseAdapter(new Goose());
simulate(gooseAdapter);
```

One thing Quackologists have always wanted to study is the total number of quacks made by a flock of ducks. \
**How can we add the ability to count duck quacks without having to change the duck classes?** \
Let’s create a decorator that gives the ducks some new behavior by wrapping them with a decorator object.
```java
public class QuackCounter implements Quackable {

    private final Quackable duck;

    private static int numberOfQuack;

    public QuackCounter(Quackable duck) {
        this.duck = duck;
        numberOfQuack = 0;
    }

    @Override
    public void quack() {
        duck.quack();
        numberOfQuack++;
    }

    public static int getNumberOfQuack() {
        return numberOfQuack;
    }
}
```

**We need to update the simulator to create decorated ducks.**
We must wrap each Quackable object in a QuackCounter decorator.
```java
Quackable mallardDuck = new QuackCounter(new MallardDuck());
Quackable redheadDuck = new QuackCounter(new RedheadDuck());
Quackable duckCall = new QuackCounter(new DuckCall());
Quackable rubberDuck = new QuackCounter(new RubberDuck());

System.out.println("The ducks quacked " + QuackCounter.getNumberOfQuack() + " times");
```

Here we have to decorate objects to get decorated behavior. Why not use a factory to produce ducks; in other words, let’s take the duck creation and decorating and encapsulate it.
```java
public abstract class AbstractDuckFactory {
    public abstract Quackable createMallardDuck();
    public abstract Quackable createRedheadDuck();
    public abstract Quackable createDuckCall();
    public abstract Quackable createRubberDuck();
}

public class DuckFactory extends AbstractDuckFactory {

    @Override
    public Quackable createMallardDuck() {
        return new MallardDuck();
    }

    @Override
    public Quackable createRedheadDuck() {
        return new ReadheadDuck();
    }

    @Override
    public Quackable createDuckCall() {
        return new DuckCall();
    }

    @Override
    public Quackable createRubberDuck() {
        return new RubberDuck();
    }
}

public class CountingDuckFactory extends AbstractDuckFactory {

    @Override
    public Quackable createMallardDuck() {
        return new QuackCounter(new MallardDuck());
    }

    @Override
    public Quackable createRedheadDuck() {
        return new QuackCounter(new ReadheadDuck());
    }

    @Override
    public Quackable createDuckCall() {
        return new QuackCounter(new DuckCall());
    }

    @Override
    public Quackable createRubberDuck() {
        return new QuackCounter(new RubberDuck());
    }
}
```

**Let’s set up the simulator to use the factory.** \
We're going to alter the *simulate()* method so that it takes a factory and uses it to create ducks.
```java
public class DuckSimulator {

    public static void main(String[] args) {
        DuckSimulator simulator = new DuckSimulator();
        AbstractDuckFactory duckFactory = new CountingDuckFactory();
        simulator.simulate();
    }
    
    public void simulate(AbstractDuckFactory factory) {
        Quackable mallardDuck = factory.createMallardDuck();
        Quackable redheadDuck = factory.createRedheadDuck();
        Quackable duckCall = factory.createDuckCall();
        Quackable rubberDuck = factory.createRubberDuck();
        ....
    }
}
```

Here’s another good question: Why are we managing ducks individually? What we need is a way to talk about collections of ducks and even sub- collections of ducks. It would also be nice if we could apply operations across the whole set of ducks. \
**Let’s create a flock of ducks (actually, a flock of Quackables).**

```java
public class Flock implements Quackable {

    private List<Quackable> quackers;

    public Flock() {
        quackers = new ArrayList<>();
    }

    public void add(Quackable quackable) {
        quackers.add(quackable);
    }

    @Override
    public void quack() {
        Iterator<Quackable> iterator = quackers.iterator();
        while (iterator.hasNext()) {
            Quackable quackable = iterator.next();
            quackable.quack();
        }
    }
}
```

**Now we need to alter the simulator.**
```java
Flock flock = new Flock();

flock.add(mallardDuck);
flock.add(redheadDuck);
flock.add(duckCall);
flock.add(rubberDuck);
flock.add(gooseAdapter);

Flock flockOfMallards = new Flock();

Quackable mallardOne = factory.createMallardDuc();
Quackable mallardTwo = factory.createMallardDuc();
Quackable mallardThree = factory.createMallardDuc();

flockOfMallards.add(mallardOne);
flockOfMallards.add(mallardTwo);
flockOfMallards.add(mallardThree);

flock.add(flockOfMallards);

simulate(flock);
simulate(flockOfMallards);
```

The Composite is working great! Now we have the opposite request: we also need to track individual ducks. Can we keep track of individual duck quacking in real time? \
**First we need an Observable interface.**
```java
public interface QuackObservable {
    public void registerObserver(Observer observer);
    public void notifyObservers();
}
```

Now we need to make sure all Quackables implement this interface...
```java
public interface Quackable extends QuackObservable {
    public void quack();
}
```

Now, we need to make sure all ducks that implement **Quackable** can handle being a **QuackObservable**. Instead of approaching this by implementing in each class, we do it a little differently: we encapsulate the code in another class, call it **Observable**, then we let all ducks reference to it.
```java
public class Observable implements QuackObservable {

    private List<Observer> observers;

    private QuackObservable duck;

    public Observable(QuackObservable duck) {
        observers = new ArrayList<Observer>();
        this.duck = duck;
    }

    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void notifyObservers() {
        Iterator<Observer> iterator = observers.iterator();
        while (iterator.hasNext()) {
            Observer observer = iterator.next();
            observer.update(duck);
        }
    }
}
```

**Integrate the helper Observable with the Quackable classes.**

```java
public class MallardDuck implements Quackable {

    // Each Quackable has an Observable instance variable.
    private Observable observable;

    public MallardDuck() {
        // pass the MallardDuck object into Observable
        observable = new Observable(this);
    }

    @Override
    public void quack() {
        System.out.println("quack!");
        notifyObservers();
    }

    // We just delegate to the helper.

    @Override
    public void registerObserver(Observer observer) {
        observable.registerObserver(observer);
    }

    @Override
    public void notifyObservers() {
        observable.notifyObservers();
    }
}
```

**We're almost there! We just need to work on the Observer side of the pattern.**

```java
public interface Observer {
    public void update(QuackObservable duck);
}

public class QuackOlogist implements Observer {

    @Override
    public void update(QuackObservable duck) {
        System.out.println("QuackOlogist: " + duck + "just quacked.");
    }
}
```

**We’re ready to observe. Let’s update the simulator and give it a try:**

```java
public void simulate(AbstractDuckFactory factory) {
    // create duck factories and ducks here

    // create flocks here

    // create a QuackOlogist and set as observer
    QuackOlogist quackOlogist = new QuackOlogist();
    flockOfMallards.registerObserver(quackOlogist);

    simulate(flockOfMallards);

    System.out.println("The ducks quacked " + QuackCounter.getNumberOfQuack() + " times");
}    
```

This is the big finale, six patterms have come togrther to create this amazing Duck Simulator. See [here](https://github.com/haozhangms/Head-First-Design-Pattern/tree/main/CompoundPattern) for complete code sample. 

### Known Uses

### Related Patterns
