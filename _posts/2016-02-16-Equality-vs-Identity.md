---
layout      : post
title       : Equality vs Identity
description : An explanation of the two concepts "equality" and "identity" when comparing objects in Java.
headline    : AGE OF JAVA
category    : java
modified    : 2016-02-16
tags        : [Comparison, definition, Equality, Equals, HashCode, Identity, Java]
featured    : false
---

When storing objects in a `Set`, it is important that the same object can never be added twice. That is the core definition of a `Set`. In java, two methods are used to determine whether two referenced objects are the same or if they can both exist in the same `Set`; `equals()` and `hashCode()`. In this article I will explain the difference between equality and identity and also take up some of the advantages they have over each other.

Java offers a standard implementation of both these methods. The standard `equals()`-method is defined as an "identity" comparing method. It means that it compares the two memory references to determine if they are the same. Two identical objects that are stored in different locations in the memory will therefore be deemed unequal. This comparison is done using the `==`-operator, as can be seen if you look at the source code of the `Object`-class.

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

The `hashCode()`-method is implemented by the virtual machine as a native operation so it is not visible in the code, but it is often realized as simply returning the memory reference (on 32-bit architectures) or a modulo 32 representation of the memory reference (on a 64-bit architecture).
One thing many programmers choose to do when designing classes is to override this method with a different equality definition where instead of comparing the memory reference, you look at the values of the two instances to see if they can be considered equal. Here is an example of that:

```java
import java.util.Objects;
import static java.util.Objects.requireNonNull;

public final class Person {

private final String firstname;
private final String lastname;

public Person(String firstname, String lastname) {
    this.firstname = requireNonNull(firstname);
    this.lastname = requireNonNull(lastname);
}

@Override
public int hashCode() {
    int hash = 7;
    hash = 83 * hash + Objects.hashCode(this.firstname);
    hash = 83 * hash + Objects.hashCode(this.lastname);
    return hash;
}

@Override
public boolean equals(Object obj) {
    if (this == obj) return true;
        if (obj == null) return false;
        if (getClass() != obj.getClass()) return false;
        final Person other = (Person) obj;
        if (!Objects.equals(this.firstname, other.firstname)) {
            return false;
        } else return Objects.equals(this.lastname, other.lastname);
    }
}

```

This comparison is called "equality" (compared to the previous "identity"). As long as two persons have the same first- and lastname, they will be considered equal. This can for an example be used to sort out duplicates from a stream of input. Remember that if you override the `equals()`-method, you should always override the `hashCode()`-method as well!

### Equality

Now, if you choose equality over identity, there are some things you will need to think about. The first thing you must ask yourself is: are two instances of this class with the same properties necessarily the same? In the case of Person above, I would say no. It is very likely that you will someday have two people in your system with the same first- and lastname. Even if you continue to add more personal information like birthday or favorite color, you will sooner or later have a collision. On the other hand, if your system are handling cars and each car contains a reference to a "model", it can be safely assumed that if two cars both are black Tesla model S, they are probably the same model even if the objects are stored in different places in the memory. That is an example of a case when equality can be good.

```java
import java.util.Objects;
import static java.util.Objects.requireNonNull;

public final class Car {

    public static final class Model {

        private final String name;
        private final String version;

        public Model(String name, String version) {
            this.name = requireNonNull(name);
            this.version = requireNonNull(version);
        }

        @Override
        public int hashCode() {
            int hash = 5;
            hash = 23 * hash + Objects.hashCode(this.name);
            hash = 23 * hash + Objects.hashCode(this.version);
            return hash;
        }

        @Override
        public boolean equals(Object obj) {
            if (this == obj) return true;
            if (obj == null) return false;
            if (getClass() != obj.getClass()) return false;
            final Model other = (Model) obj;
            if (!Objects.equals(this.name, other.name)) {
                return false;
            } else return Objects.equals(this.version, other.version);
        }
    }

    private final String color;
    private final Model model;

    public Car(String color, Model model) {
        this.color = requireNonNull(color);
        this.model = requireNonNull(model);
    }

    public Model getModel() {
        return model;
    }
}
```

Two cars are only considered the same if they have the same memory address. Their models on the other hand is considered the same as long as they have the same name and version. Here is an example of this:

```java
final Car a = new Car("black", new Car.Model("Tesla", "Model S"));
final Car b = new Car("black", new Car.Model("Tesla", "Model S"));

System.out.println("Is a and b the same car? " + a.equals(b));
System.out.println("Is a and b the same model? " + a.getModel().equals(b.getModel()));

// Prints the following:
// Is a and b the same car? false
// Is a and b the same model? true
```

### Identity

One risk of choosing equality over identity is that it can be an invitation to allocating more objects than necessarily on the heap. Just look at the car example above. For every car we create we also allocate space in memory for a model. Even if java generally optimizes string allocation to prevent duplicates, it is still a certain waste for objects that will always be the same. A short trick to turn the inner object into something that can be compared using identity comparing method and at the same time avoid unnecessary object allocation is to replace it with an enum:

```java
public final class Car {

    public enum Model {

        TESLA_MODEL_S ("Tesla", "Model S"),
        VOLVO_V70 ("Volvo", "V70");

        private final String name;
        private final String version;

        Model(String name, String version) {
            this.name    = name;
            this.version = version;
        }
    }

    private final String color;
    private final Model model;

    public Car(String color, Model model) {
        this.color = requireNonNull(color);
        this.model = requireNonNull(model);
    }

    public Model getModel() {
        return model;
    }
}

```

Now we can be sure that each model will only ever exist at one place in memory and can therefore safely be compared using identity comparison. An issue with this however is that is really limits our extendability. Before with could define new models on the fly without modifying the source code in the `Car.java`-file, but now we have locked ourselves into an enum that should generally be kept unmodified. If those properties are desired, an equals comparison is probably better for you.

A finishing note, if you have overridden the `equals()` and `hashCode()`-methods of a class and later want to store it in a Map based on identity, you can always use the [IdentityHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/IdentityHashMap.html) structure. It will use the memory address to reference its keys, even if the `equals()`- and `hashCode()`-methods have been overridden.
