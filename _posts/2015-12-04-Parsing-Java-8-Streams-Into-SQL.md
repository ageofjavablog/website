---
layout      : post
title       : Parsing Java 8 Streams Into SQL
description : Stream over database entries just like you would a standard Java 8 collection and create SQL queries dynamically.
headline    : AGE OF JAVA
category    : java
modified    : 2015-12-04
tags        : [API, Database, Java, Java 8, Lambda, Queries, Speedment, SQL, Stream]
featured    : true
---

When Java 8 was released and people began streaming over all kinds of stuff, it didn’t take long before they started imagining how great it would be if you could work with your databases in the same way. Essentially relational databases are made up of huge chunks of data organized in table-like structures. These structures are ideal for filtering and mapping operations, as can be seen in the `SELECT`, `WHERE` and `AS` statements of the SQL language.

What people did at first (me included) was to ask the database for a large set of data and then process that data using the new cool Java 8-streams. The problem that quickly arose was that the latency alone of moving all the rows from the database to the memory took too much time. The result was that there was not much gain left from working with the data in-memory. Even if you could do really freaking advanced stuff with the new Java 8-tools, the greatness didn’t really apply to database applications because of the performance overhead.

When I began committing to the [Speedment Open Source](https://github.com/speedment/speedment) project, we soon realized the potential in using databases the Java 8-way, but we really needed a smart way of handling this performance issue. In this article I will show you how we solved this using a custom delegator for the Stream API to manipulate a stream in the background, optimizing the resulting SQL queries.

Imagine you have a table `User` in a database on a remote host and you want to print out the name of all users older than 70 years. The Java 8 way of doing this with Speedment would be:

```java
final UserManager users = speedment.managerOf(User.class);
users.stream()
    .filter(User.AGE.greaterThan(70))
    .map(User.NAME.get())
    .forEach(System.out::println);
```

Seeing this code might give you shivers at first. Will my program download the entire table from the database and filter it in the client? What if I have 100 000 000 users? The network latency would be enough to kill the application! Well, actually no because as I said previously, Speedment analyzes the stream before termination.

Let’s look at what happens behind the scenes. The `.stream()` method in `UserManager` returns a custom implementation of the `Stream` interface that contain all metadata about the stream until the stream is closed. That metadata can be used by the terminating action to optimize the stream. When `.forEach` is called, this is what the pipeline will look like:

<img src="/website/images/2015-12-04/step-1.png" alt="The Java 8 Pipeline" />

The terminating action (in this case `ForEach` will then begin to traverse the pipeline backwards to see if it can be optimized. First it comes across a map from a `User` to a `String`. Speedment recognizes this as a `Getter` function since the `User.NAME` field was used to generate it. A `Getter` can be parsed into SQL, so the terminating action is switched into a `Read` operation for the `NAME` column and the map action is removed.

<img src="/website/images/2015-12-04/step-2.png" alt="The first intermediate operation is removed" />

Next off is the `.filter` action. The filter is also recognized as a custom operation, in this case a predicate. Since it is a custom implementation, it can contain all the necessary metadata required to use it in a SQL query, so it can safely be removed from the stream and appended to the `Read` operation.

<img src="/website/images/2015-12-04/step-3.png" alt="The source of the Java 8 pipeline is reached" />

When the terminating action now looks up the pipeline, it will find the source of the stream. When the source is reached, the Read operation will be parsed into SQL and submitted to the SQL manager. The resulting `Stream<String>` will then be terminated using the original `.forEach` consumer. The generated SQL for the exact code displayed above is:

```sql
SELECT `name` FROM `User` WHERE `User`.`age` > 70;
```

No changes or special operations need to be used in the java code!

This was an simple example of how streams can be simplified before execution by using a custom implementation as done in [Speedment](https://github.com/speedment/speedment). You are welcome to look at the source code and find even better ways to utilize this technology. It really helped us improve the performance of our system and could probably work for any distributed Java-8 scenario.

Until next time!
