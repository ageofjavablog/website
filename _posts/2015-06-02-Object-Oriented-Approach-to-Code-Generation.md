---
layout      : post
title       : Object-Oriented Approach to Code Generation
description : Introducing CodeGen, an MVC-oriented Code Generator for Java
headline    : AGE OF JAVA
category    : java
modified    : 2015-06-02
tags        : [Automatic Programming, CodeGen, Generation, Java, Speedment]
---

Code Generation is a common way to reduce the unhealthy load of boring tasks often put on us eager code monkeys. Many code generation frameworks I have seen use a template-replace-repeat approach where you write a template for how the generated code file should look and then replace certain keywords and repeat other sections to produce the specific file you want.

A problem with this approach that annoys me is that it is really difficult to know if the generated code will work or not until you compile it. You might have changed the name of one class and suddenly the generated code won't build. To handle this issue [I started a project called CodeGen](https://github.com/Pyknic/CodeGen) that aim to be completely object-oriented so that you can benefit from type-safety all the way from template to executable code. The main user case for the generator is the [Speedment software](https://github.com/speedment/speedment), but it can be used in a variety of projects.

Consider the following code:

```java
final Generator generator = new JavaGenerator();

final File file = File.of("org/example/Foo.java")
    .add(Class.of("Foo").public_()
        .add(Field.of("x", DOUBLE_PRIMITIVE).final_())
        .add(Field.of("y", DOUBLE_PRIMITIVE).final_())
        .add(Field.of("z", DOUBLE_PRIMITIVE).final_())
        .call(new AutoConstructor())
        .call(new AutoSetGetAdd())
        .call(new AutoEquals())
    )
    .call(new AutoJavadoc())
    .call(new AutoImports(generator))
;
```

The model tree of the application is built using beans. New methods and member variables can be added to the tree to create variants of the same class.

When the code is to be rendered it can easily be passed to a generator class.

```java
String code = generator.on(file).get();
```

The generated code will look like the following:

```java
/**
 * Write some documentation here.
 */
package org.example;

import java.util.Optional;

/**
 * @author You name here
 */
public class Foo {

    private final double x;
    private final double y;
    private final double z;

    /**
     * Initializes the Foo component.
     *
     * @param x  the x
     * @param y  the y
     * @param z  the z
     */
    public Foo(double x, double y, double z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }

    /**
     * Returns the value of x.
     *
     * @return  the value of x
     */
    public double getX() {
        return x;
    }

    /**
     * Sets a new value for x.
     *
     * @param x  the new value of x
     */
    public void setX(double x) {
        this.x = x;
    }

    /**
     * Returns the value of y.
     *
     * @return  the value of y
     */
    public double getY() {
        return y;
    }

    /**
     * Sets a new value for y.
     *
     * @param y  the new value of y
     */
    public void setY(double y) {
        this.y = y;
    }

    /**
     * Returns the value of z.
     *
     * @return  the value of z
     */
    public double getZ() {
        return z;
    }

    /**
     * Sets a new value for z.
     *
     * @param z  the new value of z
     */
    public void setZ(double z) {
        this.z = z;
    }

    /**
     * Generates a hashCode for this object. If any field is
     * changed to another value, the hashCode may be different.
     * Two objects with the same values are guaranteed to have
     * the same hashCode. Two objects with the same hashCode are
     * not guaranteed to have the same hashCode."
     *
     * @return  the hash code
     */
    @Override
    public int hashCode() {
        int hash = 7;
        hash = 31 * hash + (Double.hashCode(this.x));
        hash = 31 * hash + (Double.hashCode(this.y));
        hash = 31 * hash + (Double.hashCode(this.z));
        return hash;
    }

    /**
     * Compares this object with the specified one for equality.
     * The other object must be of the same type and not null for
     * the method to return true.
     *
     * @param other  the object to compare with
     * @return  {@code true} if the objects are equal
     */
    @Override
    public boolean equals(Object other) {
        return Optional.ofNullable(other)
            .filter(o -> getClass().equals(o.getClass()))
            .map(o -> (Foo) o)
            .filter(o -> this.x == o.x)
            .filter(o -> this.y == o.y)
            .filter(o -> this.z == o.z)
            .isPresent();
    }
} 
```

Every component is implemented as a `interface`-`class` pair so that you can change the implementation dynamically without rewriting other parts of the system.

Hopefully this will be helpful for other people!
