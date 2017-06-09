---
layout      : post
title       : Using Traits in Java 8
description : A smart way of casting an element in Java 8 streams to multiple interfaces at once using traits.
headline    : AGE OF JAVA
category    : java
modified    : 2016-01-27
tags        : [Java, Java 8, Programming, Speedment, Stream, Tips]
featured    : true
---

In [one of my earlier articles](/java/Type-Safe-Views-using-Abstract-Document-Pattern) I mentioned a programming component called "traits". These constructs have existed [for many years](https://en.wikipedia.org/wiki/Trait_(computer_programming)) in other programming languages like [Scala](http://docs.scala-lang.org/tutorials/tour/traits.html) and [PHP](https://secure.php.net/manual/en/language.oop5.traits.php), but have only recently been available through default methods in Java. I will not go into the possibilities with using traits in this article, but I will show you a neat trick [we use a lot at Speedment](https://github.com/speedment/speedment) that you can use if you ever need to stream over a collection of different objects and separate those that fulfill a number of traits.

<img src="/images/2016-01-27/traits.png" alt="Spire and Duke wearing masks" />

Say that you have two noun classes Person and Elephant. There is no reason really why persons and elephants should belong to the same super class; elephants are intelligent four-legged creatures and most humans are not. You might still find the two of them in the same computer system and sometimes you even need to store them in the same collection. One way of operating on this collection of various living beings without making them share a common ancestor, (which would totally be just a theory), you can give them similar _traits_.

Take a look at this interface:

```java
interface HasName extends Document {
    final String NAME = "name";

    default String getName() {
        return get(NAME);
    }

    default void setName(String name) {
        put(NAME, name);
    }
}
```

Using the [Abstract Document Pattern](/java/Type-Safe-Views-using-Abstract-Document-Pattern) presented earlier, the trait can set and get the attribute "name" from a map. If we now want to iterate over our collection of many living things that might or might not implement our specified traits, we can easily do it like this:

```java
final Set<Object> livingBeings = new HashSet<>();

livingBeings.add(new Person(...));
livingBeings.add(new Person(...));
livingBeings.add(new Elephant(...));

livingBeings.stream()
    .filter(HasName.class::isInstance)
    .filter(HasAge.class::isInstance)
    .filter(HasWeight.class::isInstance)
    .map(p -> (HasName & HasAge & HasWeight) p)
    .forEach(p ->
        System.out.println(
            p.getName() + " is " +
            p.getAge() + " years old and weighs " +
            p.getWeight() + " pounds."
        )
    );
```

Using the and-character (`&`) we can cast instances that implement all the required traits into a dynamic type, without them sharing an ancestor.

Was this interesting? In [the following article](/java/Using-Traits-in-Java-8) I present a more formal definition of the Trait Pattern in java.
