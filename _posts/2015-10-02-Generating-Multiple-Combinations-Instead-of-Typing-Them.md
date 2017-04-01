---
layout      : post
title       : Generating Multiple Combinations Instead of Typing Them
description : Generate multiple combinations of for an example the functional interfaces of Java 8 to minimize human errors in code.
headline    : AGE OF JAVA
category    : java
modified    : 2015-10-02
tags        : [Automatic Programming, Code, CodeGen, Generator, Java, Speedment, Tool]
---

There is only one thing most programmers hate more than to repetitively type the same code over and over again and that is to try to fix all the bugs that come with that kind of design. A really good principle is [DRY ("Don't Repeat Yourself")](http://programmer.97things.oreilly.com/wiki/index.php/Don't_Repeat_Yourself). But still, with the new functions, optionals, streams and so on that came with Java 8 it is almost impossible to not do it if you want to avoid expensive boxing and unboxing of primitive data types into objects. Hopefully this will all be fixed in [project Valhalla](http://www.infoq.com/news/2014/07/Project-Valhalla), but until that is accepted into OpenJDK, we will have to manage in some other way.

In an earlier article I demonstrated an [object oriented code generator](https://github.com/Pyknic/CodeGen) that we used in the [Speedment project](https://github.com/speedment/speedment). To solve this issue with code repetition regarding primitive and object types, [I wrote a small utility class](https://gist.github.com/Pyknic/247a50f9d8a35e684aa6) that can be used to get all the different combinations that exist in streams, functions and so on. Here is how it looks when I generate all the 11 possible combinations of a `MapAction` class:

```java
final Generator gen = new JavaGenerator();
        Formatting.tab("    ");

        System.out.println(Variants.allByAll()
            .filter(Variants::checkCompability)
            .map(two -> {
                final File file = File.of(
                    two.getName("com/speedment/steam/action/", "To", "MapAction.java")
                );

                final Type in  = Type.of("IN");
                final Type out = Type.of("OUT");

                file.add(Class.of(two.getName("To", "MapAction"))
                    .final_()
                    .call(clazz ->
                        two.getGenerics("IN", "OUT").forEachOrdered(clazz::add))
                            .add(Type.of("com.speedment.stream.StreamAction")
                            .add(Generic.of().add(two.first().streamType(in)))
                            .add(Generic.of().add(two.second().streamType(out)))
                    )

                    .add(Field.of("mapper", two.asFunction(in, out))
                        .private_().final_()
                    )

                    .call(() -> file
                        .add(Import.of(Type.of(Objects.class))
                            .static_()
                            .setStaticMember("requireNonNull")
                        )
                    )

                    .add(Constructor.of().public_()
                        .add(Field.of("mapper", two.asFunction(in, out)))
                        .add("this.mapper = requireNonNull(mapper);")
                    )

                    .add(Method.of("apply", two.second().streamType(out))
                        .add(OVERRIDE).public_()
                        .add(Field.of("stream", two.first().streamType(in)))
                        .add("return requireNonNull(stream).map(mapper);")
                    )

                ).call(new AutoJavadoc<>());

                return file;
            })
            .map(gen::on)
            .filter(Optional::isPresent)
            .map(Optional::get)
            .collect(joining("\n-------------------------------------------\n"))
        );
```

When the code is executed it will generate eleven Java files and output them to the console. [The output of the program can be found here](https://gist.github.com/Pyknic/71eef9a25acb184bc14c).
