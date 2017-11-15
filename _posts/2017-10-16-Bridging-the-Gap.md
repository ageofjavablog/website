---
layout      : post
title       : Bridging the Gap Between Database and Stream (Webinar)
description : Webinar on how to query databases using Java Streams, using JShell to examplify how easy it is.
headline    : AGE OF JAVA
category    : java
modified    : 2017-10-16
tags        : [Automated, Automatic; Programming, Code Generation, CodeGen, Database, Generator, Java, Java 8, Java 9, JShell, JDBC, Query, Speedment, Stream, Tutorial, Video, Java One]
featured    : false
---
Do you also like the Stream Interface? What if you could use Streams to query databases without having to write any SQL?

Speedment is an open source implementation of the `Stream`-interface that lazily evaluates the operations performed on it to produce an optimal SQL query, fetching only the results needed for the terminating operation. Speedment also comes with a handy maven plugin that generates all the entity and manager classes needed to model the database using the database metadata as the domain. This means that you can get a database application up and running in no time.

In this video, me and Per Minborg demonstrates the database streams using JShell, the new REPL loop that comes with Java 9. We also explain how the Speedment implementation can optimize the streams before execution and why this is legal according to the Stream documentation. At the end of the video you have all the tools you need to quickly write Java 9-ready database applications using Streams instead of SQL.

Speedment is open-source and can be [downloaded from GitHub](https://github.com/speedment/speedment).

<iframe width="560" height="315" src="https://www.youtube.com/embed/89lVMmIr8MQ" frameborder="0" allowfullscreen></iframe>
