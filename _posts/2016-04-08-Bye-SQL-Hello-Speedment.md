---
layout      : post
title       : Bye Manual SQL, Hello Speedment!
description : Tutorial on how to get started with Speedment, an open source framework for writing database applications in Java 8 without SQL.
headline    : AGE OF JAVA
category    : java
modified    : 2016-04-08
tags        : [Code Generation, Database, Java 8, Programming, Speedment, SQL, Stream, Tutorial, User Interface]
featured    : true
---

Most applications written in Java require some form of data storage. In small applications this is often realized using a primitive JDBC-connection that is queried using ordinary SQL. Larger systems on the other hand often use an Object Relational Mapping (ORM) frameworks to handle the database communication. There are pro’s and con’s with both of these approaches, but both tend to involve writing a lot of boilerplate code that looks more or less the same across every codebase. In this article I will showcase another approach to easy database communication using an open source project called [Speedment](https://github.com/speedment/speedment).

### What is Speedment?

![](https://1.bp.blogspot.com/-mE0IVu3Z7Dk/Vwg4lFJx4PI/AAAAAAAAEvw/x_aDWWzgknM6WA7AcPezNzi3lAYnenGpQ/s320/Hello-Speedment.png)](https://1.bp.blogspot.com/-mE0IVu3Z7Dk/Vwg4lFJx4PI/AAAAAAAAEvw/x_aDWWzgknM6WA7AcPezNzi3lAYnenGpQ/s1600/Hello-Speedment.png)

Speedment is a developer tool that generates java classes from your SQL metadata. The generated code handles everything from setting up a connection to data retrieval and persistence. The system is designed to integrate perfectly with the Java 8 Stream API so that you can query your database using lambdas without a single line of SQL. The created streams are [optimized in the background](https://dzone.com/articles/parsing-java-8-streams-into-sql) to reduce the network load.

### Setting Up a Project

In this article I will write a small application that asks for the user’s name and age and persist it in a MySQL database. First of, we will define the database schema. Open up your MySQL console and enter the following:

```sql
CREATE DATABASE hellospeedment;
USE hellospeedment;

CREATE TABLE IF NOT EXISTS `user` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `name` varchar(32) NOT NULL,
    `age` int(5) NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 AUTO_INCREMENT=1;

```

Next we will create our java project. Fire up your favorite IDE and create a new Maven Project from Archetype. Archetypes are template projects that can be used to quickly define new maven projects. Exactly how they are used differ between different IDEs. The following information will have to be entered:

| Repository | https://repo1.maven.org/maven2 |
| GroupId    | com.speedment                  |
| ArtifactId | speedment-archetype-mysql      |
| Version    | 2.3.0                          |

Similar archetypes are available for [PostgreSQL](https://search.maven.org/#search%7Cga%7C1%7Ca%3A%22speedment-archetype-postgresql%22) and [MariaDB](https://search.maven.org/#search%7Cga%7C1%7Ca%3A%22speedment-archetype-mariadb%22) as well.

On NetBeans, the archetype is usually found among the default ones indexed from the Maven Central Repository. When the project is created you should have something like this:

![](https://lh4.googleusercontent.com/pWHS2Fmd-BC7Wp_YN0Y0ReC8gpmTeXMEoULGgTa-21B05jt0GLO5NSHREw6LBvilRn-YzFQQmA9xKgx6R4gzmFOLsFxsIBzA301wKNSF1Shl7rXAYp4pQ6rlyZMM5Reer-l8Z7B5)

### Launching the Speedment UI

Now then the project has been created it is time to start up the Speedment User Interface. This is done by executing the `speedment:gui`-maven goal. In NetBeans and IntelliJ IDEA, a list of available maven goals can be found from within the IDE. In **Netbeans** this is found in the Navigator window (often located in the bottom-left of the screen). The project root node must be selected for the goals to appear. In **IntelliJ**, the goals can be found under the "Maven Projects"-tab in the far right of the screen. You might need to maximize the "Project Name", "Plugins" and "speedment-maven-plugin"-nodes to find it. In **Eclipse**, you don’t have a list of goals as far as I know. Instead you will have to define the goal manually. There is a tutorial for doing this [on the Speedment GitHub wiki](https://github.com/speedment/speedment/wiki/Tutorial:-Creating-a-Speedment-Project-in-Eclipse).

When the user interface starts the first time it will ask for your email address. After that you can connect to your database.

![](https://lh3.googleusercontent.com/_VNpvOblX1KdSZW1vE3MFxi_eF3ZtNmwAl5UJincVjG1Mi2-ATTqllTmHIExNueRERlo9pI34757KoUGcP3rVxjgksx_XivianXaL5HUEdvype58sBwKZizh8asM6JNixdaC9EQS)

The connection dialog will only allow you to choose between databases that you can connect to using the loaded JDBC-drivers. If you for example want to use a PostgreSQL-database, you should add the PostgreSQL-driver to the `<dependencies>`-tag of the `speedment-maven-plugin` section in the `pom.xml`-file and the re-run the UI.

Once you have connected to the database, the main window opens. On the left side you can see a tree-view of the database. In the middle is the workspace where things like database connection, code generation and entity naming can be configured. You can select what part of the project to configure by selecting other nodes in the tree.

![](https://lh6.googleusercontent.com/gHrVqbQlEEPbz0pY1RJ-uJCTr1JBUGlBNTJKw5CqDTlZLHdsWGV0GGWxkBn5YyJTp0_VczbizPQXlJdMdeSlIwrLnttg31duqNMixJ8zV2ccdSd55V7wCGGPob-U6W13CTTCMAUu)

In this case, we will simply press the "Generate"-button in the toolbar to generate a project using the default settings. We can now close the UI and return to our IDE.

### Write the Application

Now when Speedment has generated all the boilerplate code required to communicate with the `HelloSpeedment` database we can focus on writing the actual application. Let’s open the `Main.java`-file created by the maven archetype and modify the `main()` method.

```java
public class Main {
    public static void main(String... params) {
        Speedment speedment = new HellospeedmentApplication()
            .withPassword("secret").build();
        Manager<User> users = speedment.managerOf(User.class);
    }
}
```

In Speedment, an application is defined using a builder pattern. Runtime configuration can be done using different `withXXX()`-methods and the platform is finialized when the `build()`-method is called. In this case, we use this to set the MySQL password. Speedment will never store sensitive information like your database passwords in the configuration files so you will either have to have a unprotected database or set the password at runtime.

The next thing we want to do is to listen for user input. When a user starts the program, we should greet them and then ask for their name and age. We should then persist the user information in the database.

```java
final Scanner scn = new Scanner(System.in);

System.out.print("What is your name? ");
final String name = scn.nextLine();

System.out.print("What is your age? ");
final int age = scn.nextInt();

try {
    users.newEmptyEntity()
        .setName(name)
        .setAge(age)
        .persist();
} catch (SpeedmentException ex) {
    System.out.println("That name was already taken.");
}
```

If the persistence failed, a `SpeedmentException` is thrown. This could for example happen if a user with that name already exists since the `name` column in the schema is set to `UNIQUE`.

### Reading the Persisted Data

Remember I started out by telling you how Speedment fits in nicely with the Stream API in Java 8? Let’s try it out! If we run the application above a few times we can populate the database with some users. We can then query the database using the same `users` manager.

```java
System.out.println(
    users.stream()
        .filter(User.ID.lessThan(100))
        .map(User::toJson)
        .collect(joining(",\n    ", "[\n    ", "\n]"))
);
```

This will produce a result something like this:

```java
[
    {"id":1,"name":"Adam","age":24},
    {"id":2,"name":"Bert","age":20},
    {"id":3,"name":"Carl","age":35},
    {"id":4,"name":"Dave","age":41},
    {"id":5,"name":"Eric","age":18}
]
```

### Summary

This article has showcased how easy it is to write database applications using [Speedment](https://github.com/speedment/speedment). We have created a project using a maven archetype, launched the Speedment UI as a maven goal, established a connection with a local database and generated application code. We have then managed to do both data persistence and querying without a single row of SQL!

That was all for this time.

**PS:** [Speedment 2.3 Hamilton](https://github.com/speedment/speedment/releases/tag/2.3.0) was just released the other day and it contains a ton of really cool features for how you can manipulate the code generator to fit your every need. Check it out!
