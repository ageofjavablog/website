---
layout      : post
title       : My Thoughts on Utility Classes
description : An opinion on how to design good utility classes
headline    : AGE OF JAVA
category    : other
modified    : 2015-04-14
tags        : [Classes, Design, Java, Naming, Programming, Standard, Utility]
---

Okey, it is time we sat down and had a talk. Yes, I know it is difficult to find the time, but we really need to have this conversation. We need to agree on some rules regarding utility classes in java.

Quite often when I browse around Github for interesting java projects (yes, I do that, don't judge me..) I find code that looks like this:

```java
public class Foo {
    private final int x, y, z;
    
    public Foo(int x, int y, int z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }

    ...

    public static Foo doSomethingWith(Foo a, Foo b) {
        ...
    }

    public static Foo doSomethingElseWith(Foo a, Foo b) {
        ...
    }
}
```

You have a class with a specific purpose and then you add logic that operates on that class as static methods. Here is why this is bad practice:

* Your classes will turn out quite long
* It is confusing to see Foo.blablabla and foo.blablabla in the same context
* The static methods can't be used for anything else than `Foo`

That's where utility classes enters to picture. If we can agree to put static logic in a separate class and reserve the suffix "Util" for that purpose, we can separate the model from the logic.

```java
public class FooUtil {

    public static Foo doSomethingWith(Foo a, Foo b) {
        ...
    }

    public static Foo doSomethingElseWith(Foo a, Foo b) {
        ...
    }
}
```

### A Template for Utility Classes
The thing now is that we have nothing that prevents us from instantiating, extending or in other ways misuse this new class. Here is a template I usually go for when writing utility classes:

```java
public final class FooUtil {

    public static Foo doSomethingWith(Foo a, Foo b) {
        ...
    }

    public static Foo doSomethingElseWith(Foo a, Foo b) {
        ...
    }

    private FooUtil() {
        throw new RuntimeException(
            "This class is a utility class and should therefore never be instantiated."
        );
    }
}
```

The exception in the constructor prevents the class from being instantiated using reflection.

