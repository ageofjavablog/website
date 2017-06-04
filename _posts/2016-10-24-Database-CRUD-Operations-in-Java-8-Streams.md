---
layout      : post
title       : Database CRUD Operations in Java 8 Streams
description : Tutorial on how to query and update a database using Java 8 Streams with the Speedment tool.
headline    : AGE OF JAVA
category    : java
modified    : 2016-10-24
tags        : [Create, CRUD, Database, Delete, Functional, Java, Java 8, JDBC, Lambda, Library, Modern, Query, Read, Speedment, Statement, Stream, Update]
featured    : true
---

The biggest obstacle to overcome when starting out with a new tool is to get your head around how to do the little things. By now you might feel confident in how the new Java 8 Stream API works, but you might not have used it for database querying yet. To help you get started creating, modifying and reading from your SQL database using the Stream API, I have put together this quick start. Hopefully it can help you take your streams to the next level!

![Spire and Duke in front of the letters CRUD](https://lh3.googleusercontent.com/17K5cRtmbFf1dmtiD1m-jeYvnsZYRQpEm3V7d7x1qAu_su-TuoAF5tpiCOYsLMRNDXrMk_mu5QU_175TdUqH0j-ug8G05hyVBjngSx470Jr_UD3k8Acb73tqmTRz9RAAg37WxL-v)

### Background

[Speedment is an Open Source toolkit](https://github.com/speedment/speedment) that can be used to generate java entities and managers for communicating with a database. Using a graphical tool you connect to your database and generate a complete ORM tailored to represent your domain model. But Speedment is not only a code generator but also a runtime that plugs into your application and makes it possible to translate your Java 8 streams into optimized SQL queries. That is the part that I will focus on in this article.

### Generate Code

To begin using Speedment in a Maven project, add the following lines to your pom.xml-file. In this example I am using MySQL, but you can use PostgreSQL or MariaDB as well. Connectors to proprietary databases like Oracle are available for enterprise customers.

###### pom.xml

```xml
<properties>
  <speedment.version>3.0.1</speedment.version>
  <db.groupId>mysql</db.groupId>
  <db.artifactId>mysql-connector-java</db.artifactId>
  <db.version>5.1.39</db.version>
</properties>

<dependencies>
  <dependency>
    <groupId>com.speedment</groupId>
    <artifactId>runtime</artifactId>
    <version>${speedment.version}</version>
    <type>pom</type>
  </dependency>

  <dependency>
    <groupId>${db.groupId}</groupId>
    <artifactId>${db.artifactId}</artifactId>
    <version>${db.version}</version>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <groupId>com.speedment</groupId>
      <artifactId>speedment-maven-plugin</artifactId>
      <version>${speedment.version}</version>

      <dependencies>
        <dependency>
          <groupId>${db.groupId}</groupId>
          <artifactId>${db.artifactId}</artifactId>
          <version>${db.version}</version>
        </dependency>
      </dependencies>
    </plugin>
  </plugins>
</build>
```

You now have access to a number of new Maven Goals that make it easier to use the toolkit. The launch the Speedment UI, execute:

```bash
mvn speedment:tool
```

This will guide you through the process of connecting to the database and configure the code generation. The simplest way in the beginning is you just run along with the default settings. Once you press “Generate”, Speedment will analyze your database metadata and fill your project with new sources like entity and manager classes.

### Initialize Speedment

Once you have generated your domain model, setting up Speedment is easy. Create a new `Main.java`-file and add the following lines. All the classes you see are generated so their names will depend on the names of your database schemas, tables and columns.

###### Main.java

```java
public class Main {
  public static void main(String... param) {
    final HaresApplication app = new HaresApplicationBuilder()
      .withPassword("password")
      .build();
  }
}
```

The code above creates a new application instance using a generated builder pattern. The builder makes it possible to set any runtime configuration details like database passwords.

Once we have an app instance, we can use it to get access to the generated managers. In this case, I have four tables in the database; “hare”, “carrot”, “human” and “friend”. (You can see [the entire database definition here](https://github.com/speedment/speedment-code-samples/tree/master/hares/src/main/resources)).

```java
final CarrotManager carrots = app.getOrThrow(CarrotManager.class);
final HareManager hares     = app.getOrThrow(HareManager.class);
final HumanManager humans   = app.getOrThrow(HumanManager.class);
final FriendManager friends = app.getOrThrow(FriendManager.class);
```

These managers can now be used to perform all our CRUD operations.

### Create Entities

Creating entities is very straight forward. We use the generated implementation of our entities, set the values we want for columns and then persist it to the data source.

```java
hares.persist(
  new HareImpl()
    .setName("Harry")
    .setColor("Gray")
    .setAge(8)
);
```

The persist method returns a (potentially) new instance of Hare where auto-generated keys like “id” has been set. If we want to use Harry after persisting him we should therefore use the instance returned by persist.

```java
final Hare harry = hares.persist(
  new HareImpl()
    .setName("Harry")
    .setColor("Gray")
    .setAge(8)
);
```

If the persistence fails, for an example if a foreign key or a unique constraint fails, a `SpeedmentException` is thrown. We should check for this and show an error if something prevented us from persisting the hare.

```java
try {
  final Hare harry = hares.persist(
    new HareImpl()
      .setName("Harry")
      .setColor("Gray")
      .setAge(8)
  );
} catch (final SpeedmentException ex) {
  System.err.println(ex.getMessage());
  return;
}
```

### Read Entities

The coolest feature in the Speedment runtime is the ability to stream over data in your database using Java 8 Streams. _“Why is that so cool?”_ you might ask yourself. _“Even [Hibernate have support for streaming](http://javadeau.lawesson.se/2016/10/java-8-streams-in-hibernate-and-beyond.html) nowadays!”_

The beautiful thing with Speedment streams is that they take intermediary and terminating actions into consideration when constructing the stream. This means that if you add a filter to the stream after it has been created, it will still be taken into consideration when building the SQL statement.

Here is an example. We want to count the total number of hares in the database.

```java
final long haresTotal = hares.stream().count();
System.out.format("There are %d hares in total.%n", haresTotal);
```

The SQL query that will be generated is the following:

```sql
SELECT COUNT(*) FROM hares.hare;
```

The terminating operation was a `.count()` so Speedment knows that it is a `SELECT COUNT(...)`-statement that is to be created. It also knows that the primary key for the “hare” table is the column “id”, which makes it possible to reduce the entire statement sent to the database down into this.

A more complex example might be to find the number of hares that has a name that ends with the letters “rry” and an age greater or equal to 5\. That can be written as this:

```java
final long complexTotal = hares.stream()
  .filter(Hare.NAME.endsWith("rry"))
  .filter(Hare.AGE.greaterOrEqual(5))
  .count();
```

We use the predicate builders generated to us by Speedment to define the filters. This make it possible for us to analyze the stream programmatically and reduce it down to the following SQL statement:

```sql
SELECT COUNT(id) FROM hares.hare
WHERE hare.name LIKE CONCAT("%", ?)
AND hare.age >= 5;
```

If we add an operation that Speedment can’t optimize to the stream, it will be resolved just like any Java 8 stream. We are never limited to the use of the generated predicate builders, it just makes the stream more efficient.

```java
final long inefficientTotal = hares.stream()
  .filter(h -> h.getName().hashCode() == 52)
  .count();
```

This would produce the following extremely inefficient statement, but it will still work.

```sql
SELECT id,name,color,age FROM hares.hare;
```

### Update Entities

Updating existing entities are done very similar to how we read and persist entities. Changes made to a local copy of an entity will not affect the database until we call the `update()`-method in the manager.

In this case we take the hare Harry created earlier and we want to change his color to brown:

```java
harry.setColor("brown");
final Hare updatedHarry = hares.update(harry);
```

The manager returns a new copy of the hare if the update is accepted, so we should continue using that instance after this point. Just like in the “create”-example, the update might fail. Maybe color was defined as a “unique” column and a “brown” hare already existed. In that case, a `SpeedmentException` is thrown.

We can also update multiple entities at the same time by combining it with a stream. Say that we want to make all hares named “Harry” brown. In that case, we do this:

```java
hares.stream()
  .filter(Hare.NAME.equal("Harry"))
  .map(Hare.COLOR.setTo("Brown"))
  .forEach(hares.updater()); // Updates remaining elements in the Stream
```

We should also wrap it in a try-catch to make sure we warn the user if a constraint failed.

```java
try {
  hares.stream()
    .filter(Hare.NAME.equal("Harry"))
    .map(Hare.COLOR.setTo("Brown"))
    .forEach(hares.updater());
} catch (final SpeedmentException ex) {
  System.err.println(ex.getMessage());
  return;
}
```

### Removing Entities

The last CRUD operation we need to know is how to remove entities from the database. This is almost identical to the “update”. Say that we want to remove all hares older than 10 years. We then do this:

```java
try {
  hares.stream()
    .filter(Hare.AGE.greaterThan(10))
    .forEach(hares.remover()); // Removes remaining hares
} catch (final SpeedmentException ex) {
  System.err.println(ex.getMessage());
  return;
}
```

### Summary

In this article you have learned how to set up Speedment in a Maven project and how to create, update, read and delete entities from a database using Java 8 Streams. This is only a small subset of all the things you can do with Speedment, but it serves as a good introduction to start getting your hands dirty. More examples and more advanced use-cases can be [found on the GitHub-page](https://github.com/speedment/speedment/wiki).

Until next time!
