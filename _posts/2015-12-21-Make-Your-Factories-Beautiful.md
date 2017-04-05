---
layout      : post
title       : Make Your Factories Beautiful
description : An interesting way to declare constructors to make the Factory Pattern more readable in Java code.
headline    : AGE OF JAVA
category    : java
modified    : 2015-12-21
tags        : [API, Design, Java, Java 8, Lambda, Speedment, Stream]
featured    : false
---

Every java programmer worth the name knows about the [Factory Pattern](https://en.wikipedia.org/wiki/Factory_method_pattern). It is a convenient and standardized way to reduce coupling by teaching a component how to fish rather than giving it to them.

When working with large systems the pattern does however add a lot of boilerplate code to the system. For every entity you need a number of different factories for producing different implementations of that entity, which is both tiresome and unnecessary to write. This is only one of many new patterns that we [have come to use at Speedment](https://github.com/speedment/speedment).

Here is a typical example where you want a car trader to be able to create instances of the `Car` interface without knowing the exact implementation.

###### Car.java
```java
public abstract class Car {
    private final Color color;

    public interface Factory {
        Car make(Color color);
    }

    protected Car(Color color) {
        this.color = color;
    }

    public abstract String getModel();
    public abstract int getPrice();
}
```

###### Volvo.java
```java
public final class Volvo extends Car {
    public Volvo(Color color) {
        super(color);
    }

    public String getModel() { return "Volvo"; }
    public int getPrice() { return 10_000; } // USD
}
```

###### Tesla.java
```java
public final class Tesla extends Car {
    public Tesla(Color color) {
        super(color);
    }

    public String getModel() { return "Tesla"; }
    public int getPrice() { return 86_000; } // USD
}
```

###### VolvoFactory.java
```java
public final class VolvoFactory implements Car.Factory {
    public Car make(Color color) { return new Volvo(color); }
}
```

###### TeslaFactory.java
```java
public final class TeslaFactory implements Car.Factory {
    public Car make(Color color) { return new Tesla(color); }
}
```

###### CarTrader.java
```java
public final class CarTrader {

    private Car.Factory factory;
    private int cash;

    public void setSupplier(Car.Factory factory) {
        this.factory = factory;
    }

    public Car buyCar(Color color) {
        final Car car = factory.make(color);
        cash += car.getPrice();
        return car;
    }
}
```

###### Main.java
```java
    ...
        final CarTrader trader = new CarTrader();
        trader.setSupplier(new VolvoFactory());
        final Car a = trader.buyCar(Color.BLACK);
        final Car b = trader.buyCar(Color.RED);
        trader.setSupplier(new TeslaFactory());
        final Car c = trader.buyCar(Color.WHITE);
    ...
```

One thing you might not have noticed yet is that most of these components are redundant from Java 8 and up. Since the factory interface might be considered a `@FunctionalInterface` we don’t need the factories, we can simply specify the constructor of the implementing classes as a method reference!

###### Car.java
```java
public abstract class Car {
    private final Color color;

    @FunctionalInterface
    public interface Factory {
        Car make(Color color);
    }
}
```

###### Main.java
```java
    ...
        trader.setSupplier(Volvo::new);
        trader.setSupplier(Tesla::new);
    ...
```

Notice that no changes are needed to the implementing classes `Volvo` and `Tesla`. Both the factories can now be removed and you are left with a much more concrete system!

(For simple examples  such as this, the `Factory`-interface is not needed at all. You could just as well make `CarTrader` take a `Function<Color, Car>`. The advantage of specifying an interface for the factory is that it is both easier to understand and it allows you to change the parameters of the constructor without changing the code that uses the factory.)
