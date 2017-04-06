---
layout      : post
title       : Definition of the Trait Pattern in Java
description : A smart way of casting an element in Java 8 streams to multiple interfaces at once using traits.
headline    : AGE OF JAVA
category    : java
modified    : 2016-02-03
tags        : [Architecture, Definition, Design, Java, Java 8, Languages, Mixins, Multiple Inheritance, Object Oriented, Pattern, Scientific, Traits]
featured    : true
---

In this article I will present [the concept of traits](/website/java/Using-Traits-in-Java-8) and give you a concrete example of how they can be used in Java to achieve less redundancy in your object design.

<img src="/website/images/2016-02-03/traitlike.png" alt="Spire and Duke trying on each others traits" />

I will begin by presenting a fictional case where traits could be used to reduce repetition and then finish with an example implementation of the trait pattern using Java 8.

Suppose you are developing a message board software and you have identified the following as your data models: “topics”, “comments” and “attachments”. A topic has a title, a content and an author. A comment has a content and an author. An attachment has a title and a blob. A topic can have multiple comments and attachments. A comment can also have multiple comments, but no attachments.

<img src="/website/images/2016-02-03/system.png" alt="UML diagram of a simple comment system in Java" />

Soon you realise that no matter how you implement the three models, there will be code repetition in the program. If you for an example want to write a method that adds a new comment to a post, you will need to write one method for commenting topics and one for commenting comments. Writing a method that summarizes a discussion by printing out the discussion tree will have to take into consideration that a node can be either a topic, a comment or an attachment.

Since the inception of Java over 20 years ago, object-oriented programming has been the flesh and soul of the language, but during this time, other languages has experimented with other tools for organizing the structure of a program. One such tool that we use in [Speedment Open Source](https://github.com/speedment/speedment) is something called “Traits”. A trait is kind of a “micro interface” that describes some characteristic of a class design that can be found in many different components throughout the system. By referring to the traits instead of the implementing class itself you can keep the system decoupled and modular.

Let’s look at how this would change our example with the message board.

<img src="/website/images/2016-02-03/complex-with-traits.png" alt="UML diagram of a comment system with Java 8 Traits" />

Now the different _traits_ of each entity has been separated into different interfaces. This is good. Since Java allows us to have multiple interfaces per class, we can reference the interfaces directly when writing our business logic. In fact, the classes will not have to be exposed at all!

Traits have existed for many years in other programming languages such as Scala, PHP, Groovy, and many more. To my knowledge there is no consensus regarding what is considered a trait between different languages. On [the Wikipedia page](https://en.wikipedia.org/wiki/Trait_(computer_programming)#Characteristics) regarding traits it says that:

> “Traits both provide a set of methods that implement behaviour to a class and require that the class implement a set of methods that parameterize the provided behaviour”

The following properties are named as distinctive for traits:

* traits can be combined (symmetric sum)
* traits can be overriden (asymmetric sum)
* traits can be expanded (alias)
* traits can be excluded (exclusion)

Since Java 8 you can actually fulfill most of these criteria using interfaces. You can for an example cast an implementing class of an unknown type to a union of traits using the and (`&`) operator, which satisfies the symmetric sum criteria. A good example of this [is described here](/website/java/Using-Traits-in-Java-8). By creating a new interface and [using default implementations](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html) you can override some methods to fulfill the asymmetric sum criteria. Aliases can be created in a similar way. The only problem is exclusion. Currently java has no way of removing a method from inheritance so there is no way to prevent a child class from accessing a method defined in a trait.

If we return to the message board example, we could for an example require the implementing class to have a method `getComments()`, but all additional logic regarding adding, removing and streaming over comments could be put in the interface.

```java
public interface HasComments<R extends HasComments<R>> {

    // one method that parameterize the provided behaviour
    List<Comment> getComments();

    // two methods that implement the behaviour
    default R add(Comment comment) {
        getComments().add(comment);
        return (R) this;
    }

    default R remove(Comment comment) {
        getComments().remove(comment);
        return (R) this;
    }
}
```

If we have an object and we want to cast it to a symmetric sum of `HasComments` and `HasContent`, we could do it using the and (`&`) operator:

```java
final Object obj = ...;
Optional.of(obj)
    .map(o -> (HasComments<?> & HasContent<?>) o)
    .ifPresent(sum -> {/* do something */});
```

That was all for this time!

**PS:** If you want to read more about traits as a concept, I really suggest you to read the <cite>[Traits: Composable Units of Behaviour](http://scg.unibe.ch/archive/papers/Scha03aTraits.pdf)</cite> paper from 2003 by N. Schärli et al.
