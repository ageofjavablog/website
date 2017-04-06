---
layout      : post
title       : Building Reactive Systems with JavaFX
description : A step-for-step guide on how to use the bindings class and other event componenets in the JavaFX library to build reactive systems.
headline    : AGE OF JAVA
category    : java
modified    : 2016-01-19
tags        : [Bindings, Events, Graphics, Java, JavaFX, Programming, Reactive, Speedment, UI, User Interface]
featured    : true
---

JavaFX is the new standard library for building graphical applications in Java, but many programmers out there is still stuck with Swing or even (tremble) AWT.

A lot has happened in the 20 years java has been around. When I began looking into the JavaFX libraries two years ago [for the Speedment UI](https://github.com/speedment/speedment) I found many things fascinating! Here are a few tips on how you can use many of the new awesome features in the JavaFX toolkit to build reactive and fast applications!

### 1. Property Values
If you have snooped around in the JavaFX components you must have come across the term Property. Almost every value in the FX library can be observed, the width of a divider, the size of an image, the text in a label, the children of a list as well as the status of a checkbox. Properties come in two categories; Writables and Readables. A writable value can be changed either using a setter or by directly modifying the property. JavaFX will handle the event processing and make sure every component that depends on the property will be notified. A readable value has methods that allow you to receive notifications when the value changes.

##### Example:

```java
// Read- and writable
StringProperty name = new SimpleStringProperty("Emil");
// Only readable
ObservableBooleanValue nameIsEmpty = name.isEmpty();
```

### 2. Binding values
When you have a writable and a readable value, you can start defining rules for how these values relate. A writable property can be bound to a readable property so that its value will always match the readable one. Bindings are not immediate, but they will be resolved before the values are observed (see what I did there). Bindings can be unidirectional or bidirectional. Of course, if they are bidirectional, both properties will need to be writable.

##### Example:

```java
TextField fieldA = new TextField();
TextField fieldB = new TextField();
fieldA.prefWidthProperty().bind(fieldB.widthProperty());
```

### 3. Observable Lists
Properties are not the only thing that can be observed. The members of a list can also be observed if the list is wrapped in an [`ObservableList`](https://docs.oracle.com/javase/8/javafx/api/javafx/collections/ObservableList.html). The reaction model of the `ObservableList` is quite advanced. Not only can you receive a notification when the list is modified, you can also see exactly how the list was changed.

##### Example:
```java
List<String> otherList = Arrays.asList("foo", "bar", "bar");
ObservableList<String> list = FXCollections.observableList(otherList);

list.addListener((ListChangeListener.Change<? extends String> change) -> {
    System.out.println("Received event.");
    while (change.next()) {
        if (change.wasAdded()) {
            System.out.println(
                "Items " + change.getAddedSubList() + " was added.");
        }

        if (change.wasRemoved()) {
            System.out.println(
                "Items " + change.getRemoved() + " was removed.");
        }
    }
});

System.out.println("Old list: " + list);
list.set(1, "foo");
System.out.println("New list: " + list);
```

The output from the above is:
```
Old list: [foo, bar, bar]
Received event.
Items [foo] was added.
Items [bar] was removed.
New list: [foo, foo, bar]
```

As you can see, the set operation only created one event.

### 4. StringConverter
Sometimes, you will find that you don’t have the exact value in a component as you need to create a binding. A typical example of this is that you have a [`StringProperty`](https://docs.oracle.com/javase/8/javafx/api/javafx/util/StringConverter.html) with the path that you have gotten from a `TextField`. If you want an observable property with this value expressed as a `Path`, you will need to create a `StringConverter` for that.

##### Example:
```java
TextField fileLocation = new TextField();
StringProperty location = fileLocation.textProperty();
Property<Path> path = new SimpleObjectProperty<>();

Bindings.bindBidirectional(location, path, new StringConverter<Path>() {
    @Override
    public String toString(Path path) {
        return path.toString();
    }

    @Override
    public Path fromString(String string) {
        return Paths.get(string);
    }
});
```

The object property is not bound bidirectionally to the textfield value.

### 5. Expressions
Using the [`Bindings`-class](https://docs.oracle.com/javase/8/javafx/api/javafx/beans/binding/Bindings.html) shown before you can create all kinds of expressions. Say that you have two textfields that the user can enter information into. You now want to define a read-only field that always contains a string that if the two string lengths are equal, expresses the character by character mix between the two. If the lengths are not equal, a helping message should be shown instead.

##### Example:
```java
TextField first  = new TextField();
TextField second = new TextField();
TextField mix    = new TextField();

mix.textProperty().bind(
    Bindings.when(
        first.lengthProperty().isEqualTo(second.lengthProperty())
    ).then(Bindings.createStringBinding(
        () -> {
            int length        = first.lengthProperty().get();
            String firstText  = first.textProperty().get();
            String secondText = second.textProperty().get();
            char[] result     = new char[length * 2];

            for (int i = 0; i < length; i++) {
                result[i * 2]     = firstText.charAt(i);
                result[i * 2 + 1] = secondText.charAt(i);
            }

            return new String(result);
        },
        first.textProperty(),
        second.textProperty()
    )).otherwise("Please enter two strings of exactly the same length.")
);
```

## Conclusion
These were only a handful of the many features of JavaFX. Hopefully you can find many more creative ways of utilizing the event system!
