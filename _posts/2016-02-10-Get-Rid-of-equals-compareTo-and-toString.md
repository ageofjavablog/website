---
layout      : post
title       : Get rid of equals, compareTo and toString
description : An approach to object-oriented design in Java that makes code more maintainable by not overriding equals and compareTo.
headline    : AGE OF JAVA
category    : java
modified    : 2016-02-10
tags        : [Architecture, CompareTo, Design, Equals, Functional References, Inheritance, Java, Java 8, Object, Stream, ToString]
featured    : false
---

Have you ever looked at the javadoc of [the Object-class](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html) in Java?

Probably. You tend to end up there every now and then when digging your way down the inheritance tree. One thing you might have noticed is that it has quite a few methods that every class must inherit. The favorite methods to implement yourself rather than stick with the original ones are probably `.toString()`, `.equals()` and `.hashCode()` (why you should always implement both of the latter is described well [by Per-Åke Minborg in this post](https://minborgsjavapot.blogspot.com/2014/10/new-java-8-object-support-mixin-pattern.html)).

But these methods are apparently not enough. Many people mix in additional interfaces from the standard libraries like [Comparable](https://docs.oracle.com/javase/8/docs/api/java/lang/Comparable.html) and [Serializable](https://docs.oracle.com/javase/8/docs/api/java/io/Serializable.html). But is that really wise? Why do everyone want to implement these methods on their own so badly? Well, implementing your own `.equals()` and `.hashCode()` methods will probably make sense if you are planning on storing them in something like a `HashMap` and want to control hash collisions, but what about `compareTo()` and `toString()`?

In this article I will present an approach to software design that we use on [the Speedment open source project](https://github.com/speedment/speedment) where methods that operate on objects are implemented as functional references stored in variables rather than overriding Javas built in methods. There are several advantages to this. Your POJOs will be shorter and more concise, common operations can be reused without inheritance and you can switch between different configurations in a flexible matter.

<img src="/website/images/2016-02-10/cleaner.png" alt="Spire and Duke cleaning the house" />

### Original Code
Let us begin by looking at the following example. We have a typical Java class named `Person`. In our application we want to print out every person from a `Set` in the order of their firstname followed by lastname (in case two persons share the same firstname).

##### Person.java
```java
public class Person implements Comparable<Person> {

    private final String firstname;
    private final String lastname;

    public Person(String firstname, String lastname) {
        this.firstname = firstname;
        this.lastname  = lastname;
    }

    public String getFirstname() {
        return firstname;
    }

    public String getLastname() {
        return lastname;
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
        }
        return Objects.equals(this.lastname, other.lastname);
    }

    @Override
    public int compareTo(Person that) {
        if (this == that) return 0;
        else if (that == null) return 1;

        int comparison = this.firstname.compareTo(that.firstname);
        if (comparison != 0) return comparison;

        comparison = this.lastname.compareTo(that.lastname);
        return comparison;
    }

    @Override
    public String toString() {
        return firstname + " " + lastname;
    }
}
```

##### Main.java
```java
public class Main {
    public static void main(String... args) {
        final Set people = new HashSet<>();

        people.add(new Person("Adam", "Johnsson"));
        people.add(new Person("Adam", "Samuelsson"));
        people.add(new Person("Ben", "Carlsson"));
        people.add(new Person("Ben", "Carlsson"));
        people.add(new Person("Cecilia", "Adams"));

        people.stream()
            .sorted()
            .forEachOrdered(System.out::println);
    }
}
```

##### Output
```
run:
Adam Johnsson
Adam Samuelsson
Ben Carlsson
Cecilia Adams
BUILD SUCCESSFUL (total time: 0 seconds)
```

Person implements several methods here to control the output of the stream. The `hashCode()` and `equals()` method make sure that duplicate persons can't be added to the set. The `compareTo()` method is used by the sorted action to produce the desired order. The overridden `toString()`-method is finally controlling how each `Person` should be printed when `System.out.println()` is called. Do you recognize this structure? You can find it in almost every java project out there.

### Alternative Code
Instead of putting all functionality into the `Person` class, we can try and keep it as clean as possible and use functional references to handle these decorations. We remove all the boilerplate with `equals`, `hashCode`, `compareTo` and `toString` and instead we introduce two static variables, `COMPARATOR` and `TO_STRING`.

##### Person.java
```java
public class Person {

    private final String firstname;
    private final String lastname;

    public Person(String firstname, String lastname) {
        this.firstname = firstname;
        this.lastname  = lastname;
    }

    public String getFirstname() {
        return firstname;
    }

    public String getLastname() {
        return lastname;
    }

    public final static Comparator<Person> COMPARATOR =
        Comparator.comparing(Person::getFirstname)
            .thenComparing(Person::getLastname);

    public final static Function<Person, String> TO_STRING =
        p -> p.getFirstname() + " " + p.getLastname();
}
```

##### Main.java
```java
public class Main {
    public static void main(String... args) {
        final Set people = new TreeSet<>(Person.COMPARATOR);

        people.add(new Person("Adam", "Johnsson"));
        people.add(new Person("Adam", "Samuelsson"));
        people.add(new Person("Ben", "Carlsson"));
        people.add(new Person("Ben", "Carlsson"));
        people.add(new Person("Cecilia", "Adams"));

        people.stream()
            .map(Person.TO_STRING)
            .forEachOrdered(System.out::println);
    }
}
```

##### Output
```
run:
Adam Johnsson
Adam Samuelsson
Ben Carlsson
Cecilia Adams
BUILD SUCCESSFUL (total time: 0 seconds)
```

The nice thing with this approach is that we can now replace the order and the formatting of the print without changing our `Person` class. This will make the code more maintainable and easier to reuse, not to say faster to write.
