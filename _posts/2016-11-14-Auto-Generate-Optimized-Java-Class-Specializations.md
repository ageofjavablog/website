---
layout      : post
title       : Auto-Generate Optimized Java Class Specializations
description : How to automatically generate java class specializations to increase the JVM performance using code generation.
headline    : AGE OF JAVA
category    : java
modified    : 2016-04-08
tags        : [Automated, Class, Code Generation, CodeGen, Java, Java 8, Model Based, Performance, Programming, Specialization, ToString]
featured    : true
---

If you visited JavaOne this year you might have attended my presentation on “How to Generate Customized Java 8 Code from your Database”. In that talk I showcased how the [Speedment Open Source toolkit](https://github.com/speedment/speedment) is used to generate all kinds of Java code using a database as the domain model. One thing we didn’t have time to go into though is the fact that Speedment is not only making code generation easier, it is also itself made up by generated code. In this article I will show you have we set up Speedment to generate specialized versions of many classes to reduce the memory footprint of performance critical parts of the system.

### Background

![Spire and Duke are playing with specialized geometric shapes](https://lh5.googleusercontent.com/oCF8wcMf4KdPMSw77jaIx7g07ci3U8OU2aB-sMQHVL8rbGJUstfAfkxFHAUbwgUW5zIryuHJjBR_xmT1Vywu7X9BuBs4cuaqUShjTzky2VClHEGXpFn8IP29nXQ2NQTbw1Dxj0X_)

As you might know, Java has a number of built-in value types. These are bytes, shorts, integers, longs, floats, doubles, booleans and characters. Primitive value types are different from ordinary objects primarily in that they can be allocated directly on the memory stack, reducing the burden on the Garbage Collector. A problem with not inheriting from Object is that they can’t be put into collections or passed as parameters to methods that take object parameters without being _wrapped_. Typical wrapper classes are therefore “Integer”, “Double”, “Boolean” etc.

Wrapping a value type is not always a bad thing. The JIT (Just-In-Time) compiler is very good at optimizing away wrapper types if they can safely be replaced with primitive values, but that is not always possible. If this occur in a performance critical section of your code like in an inner loop, this can heavily impact the performance of the entire application.

That’s what happened to us when working on Speedment. We had special predicates and functions that contained metadata about their purpose. That metadata needed to be analyzed very rapidly inside an inner loop, but we was slowed down by the fact that most of this metadata was wrapped inside generic types so that they could be instantiated dynamically.

A common solution to this problem is to create a number of “specializations” of the classes that contain the wrapper types. The specializations are identical to the original class except that they use one of the primitive value types instead of a generic (object only) type. A good example of specializations are the various Stream interfaces that exist in Java 8\. In addition to “Stream” we also have an “IntStream”, a “DoubleStream” and a “LongStream”. These specializations are more efficient for their particular value type since they don’t have to rely on wrapping types in objects.

The problem with specialization classes is that they add a lot of boilerplate to the system. Say that the parts that need to be optimized consist of 20 components. If you want to support all the 8 primitive variations that Java has you suddenly have 160 components. That is a lot of code to maintain. A better solution would be to generate all the extra classes.

### Template Based Code Generation

The most common form of code generation in higher languages is template based. This means that you write a template file and then do keyword replacement to modify the text depending on what you are generating. Good examples of these are [Maven Archetypes](https://maven.apache.org/guides/introduction/introduction-to-archetypes.html) or [Thymeleaf](http://www.thymeleaf.org/). A good template engine will have support for more advanced syntax like repeating sections, expressing conditions etc. If you want to generate specialization classes using a template engine you would replace all occurrences of “int”, “Integer”, “IntStream” with a particular keyword like “${primitive}”, “${wrapper}”, “${stream}” and then specify the dictionary of words to associate with each new value type.

Advantages of template based code generation is that it is easy to setup and easy to maintain. I think most programmers that read this could probably figure out how to write a template engine fairly easy. A disadvantage is that a templates are difficult to reuse. Say that you have a basic template for a specialized, but you want floating types to also have an additional method. You could solve this with a conditional statement, but if you want that extra method to also exist in other places, you will need to duplicate code. A typical example of code that often need to be duplicated is hashCode()-methods or toString(). This is where model based code generation is stronger.

### Model Based Code Generation

In model based code generation, you build an abstract syntax tree over the code you want generated and then render that tree using a suitable renderer. The syntax tree can be mutated depending on the context that it is being used in, for an example by adding or removing methods to implement a certain interface. The main advantage of this is higher flexibility. You can dynamically take an existing model and manipulate which methods and fields to include. The downside is that model based code generation generally take a bit longer to set up.

### Case Study: Speedment Field Generator

At Speedment, we developed a code generator called [CodeGen](https://github.com/speedment/speedment/tree/master/common-parent/codegen) that uses the model based approach to automatically generate field specializations for all the primitive value types. In total around 300 classes are generated on each build.

Speedment CodeGen uses an abstract syntax tree built around the basic concepts of object oriented design. You have classes, interfaces, fields, methods, constructors, etc that you use to build the domain model. Below the method level you still need to write templated code. To define a new main class, you would write:

```java
import com.speedment.common.codegen.model.Class; // Not java.lang.Class

...

Class createMainClass() {
  return Class.of("Main")
    .public_().final_()
    .set(Javadoc.of("The main entry point of the application")
      .add(AUTHOR.setValue("Emil Forslund"))
      .add(SINCE.setValue("1.0.0"))
    )
    .add(Method.of("main", void.class)
      .public_().static_()
      .add(Field.of("args", String[].class))
      .add(
        "if (args.length == 0) " + block(
          "System.out.println(\"Hello, World!\");"
        ) + " else " + block(
          "System.out.format(\"Hi, %s!%n\", args[0]);"
        )
      )
    );
}
```

This would generate the following code:

```java
/**
 * The main entry point of the application.
 * 
 * @author Emil Forslund
 * @since  1.0.0
 */
public final class Main {
  public static void main(String[] args) {
    if (args.length == 0) {
      System.out.println("Hello, World!");
    } else {
      System.out.format("Hi, %s!%n", args[0]);
    }
  }
}
```

The entire model doesn’t have to be generated at once. If we for an example want to generate a `toString()`-method automatically, we can define that as an individual method.

```java
public void generateToString(File file) {
  file.add(Import.of(StringBuilder.class));
  file.getClasses().stream()
    .filter(HasFields.class::isInstance)
    .filter(HasMethods.class::isInstance)
    .map(c -> (HasFields & HasMethods) c)
    .forEach(clazz -> 
      clazz.add(Method.of("toString", void.class)
        .add(OVERRIDE)
        .public_()
        .add("return new StringBuilder()")
        .add(clazz.getFields().stream()
          .map(f -> ".append(\"" + f.getName() + "\")")
          .map(Formatting::indent)
          .toArray(String[]::new)
        )
        .add(indent(".toString();"))
      )
    );
}
```

Here you can see how the [Trait Pattern](http://www.ageofjava.com/2016/02/definition-of-trait-pattern-in-java.html) is used to abstract away the underlying implementation from the logic. The code will work for `Enum` as well as `Class` since both implement both the traits “HasFields” and “HasMethods”.

### Summary

In this article I have explained what specialization classes are and why they are sometimes necessary to improve performance in critical sections of an application. I have also showed you how Speedment use model based code generation to automatically produce specialization classes. If you are interested in generating code with these tools yourself, go ahead and check out the [latest version of the generator at GitHub](https://github.com/speedment/speedment)!
