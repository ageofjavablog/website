---
layout      : post
title       : Auto-Generate a REST API from a Database With Spring
description : Use your database as the domain model when generating applications for Spring Boot.
headline    : AGE OF JAVA
category    : java
modified    : 2018-08-30
tags        : [Java, Java 8, Java 9, JDBC, Query, Speedment, Stream, Tutorial, In-Memory, Spring, Code-Generation, Generate]
featured    : true
---

If you have an existing database and you want to write a front-end to work with it, you often find yourself spending hours setting up the glue between the database and the frontend. It would be much more efficient use of your time if you could simply press a button and generate the entire REST API directly.

Speedment is a tool that uses code generation to produce a tailored domain model based on an existing database structure. Queries can either be sent directly to the database, or served from in-memory for better performance. In this article we will use the official plugin called “spring-generator” for Speedment Free to generate a complete Spring application to serve a simple REST API. We will add support for paging, remote filtering and sorting, without writing a single line of code.

### The Database
In the examples below, we are using the [Sakila database](https://dev.mysql.com/doc/sakila/en) for MySQL. Sakila is an example database that models a movie rental store. It has tables called Film, Actor, Category and so on.

### Step 1: Create the Project
The first thing we need is to configure our `pom.xml`-file to use the latest Speedment dependencies and Maven plugin. The fastest way to do this is to generate a `pom.xml`-file using the Speedment Initializer that you [can find here](https://speedment.com/initializer). If you select "Spring" in the plugin list and press "Download", you will get an entire project folder with a `Main.java`-file generated automatically.

![The Speedment Initializer](/images/2018-08-30/initializer.png)

![Structure of the downloaded folder](/images/2018-08-30/zip.png)

Next, open a command line where you downloaded the file and enter the following:

```shell
mvn speedment:tool
```

This will launch the Speedment tool and prompt you for a license key. Simply select “Start Free” and you will get a free license! Once you have registered, you can connect to the database and get started.

![Connecting to a Database with Speedment](/images/2018-08-30/connect.png)

### Step 2: Generate a Controller
By default, the only thing the `spring-generator` plugin adds in addition to the regular Speedment classes is a `@Configuration`-bean. To actually generate controllers for your tables, you can select the table in the tree to the left and then check "Generate @RestController" on the right side.

![Make sure the checkbox is checked](/images/2018-08-30/rest-enabled.png)

This will enable a number of options below it, but we will get to these in a minute. For now, this is all you need to do to enable the controller logic. Click "Generate" to generate the code.

![Click on the Generate-button in the Toolbar](/images/2018-08-30/generate.png)

In the IDE, you can now see a number of classes. If you downloaded the `.zip`-file from the Initializer then you should already have a `Main.java`-file. Make sure it is in the correct folder, then build and run the application.

```shell
mvn clean install && java -jar target/example-app-1.0.0-SNAPSHOT.jar
```

You can try out the new REST API by calling:

```shell
curl localhost:8080/film
```

Easy, huh?

### Step 3: Using Filters
A cool feature with the spring-generator plugin is that it supports remote filtering. It means that the frontend can send predicates encoded as JSON-objects to the server, and the server will respond with a filtered JSON response. Speedment automatically parses the JSON filters into a SQL `SELECT`-statement. If you want to be able to serve thousands of requests per second, you can also enable the Speedment Datastore to serve queries directly from memory, in which case the JSON filters will be parsed to find an appropriate in-memory index.

The syntax for the JSON filters is easy. To get films with a length less than 60 minutes, you simply call:

```shell
curl -G localhost:8080/film --data-urlencode \
  'filter={"property":"length","operator":"lt","value":60}'
```

(The -G argument makes sure that the command is sent as a GET request and not a POST request)

You can add multiple filters by wrapping the object in a list.

```shell
curl -G localhost:8080/film --data-urlencode \
  'filter=[{"property":"length","operator":"lt","value":60},
  {"property":"length","operator":"ge","value":30}]'
```

This will return all films with a length between 30 and 60 minutes. By default, all the operators in the list are assumed to be separated with AND-operators. All the conditions must apply for a row to pass the filter. To change this, you can use an explicit OR-statement.

```shell
curl -G localhost:8080/film --data-urlencode \
  'filter={"or":[{"property":"length","operator":"lt","value":30},
  {"property":"length","operator":"ge","value":60}]}'
```

This will return all films that are either shorter than 30 minutes or longer than one hour.

### Step 4: Using Sorters
By default, Speedment returns rows in the same order as the database returns them. If you add filters to the expression however, the returned order is undefined. To make sure the order is well defined in the frontend, you could therefore send a defined order to the generated backend to tell it how to sort elements. If you use Speedment Datastore to serve elements from in-memory, then it will use in-memory indexes to resolve the correct order.

Here is an example where the films are retrieved by their length.

```shell
curl -G localhost:8080/film --data-urlencode \
  'sort={"property":"length"}'
```

To get the films in the opposite order, you can specify a descending order like this:

```shell
curl -G localhost:8080/film --data-urlencode \
  'sort={"property":"length","direction":"DESC"}'
```

You can also send multiple sorters to the backend to define the primary order, the secondary order and so on.

```shell
curl -G localhost:8080/film --data-urlencode \
  'sort=[{"property":"length","direction":"DESC"},
  {"property":"title","direction":"ASC"}]'
```

### Step 5: Using Paging
The last feature of the `spring-generator` plugin is the ability to page results to avoid sending unnecessary large objects to the browser. This is enabled by default, which is why you won’t see more than 25 results when you query the backend. To skip a number of results (not pages), you can specify the `?start=` parameter.

```shell
curl localhost:8080/film?start=25
```

This will skip the first 25 elements and begin at the 26th. You can change the default page size by adding the `?limit=` parameter.

```shell
curl 'localhost:8080/film?start=25&limit=5'
```

This also begins at the 26th element, but only returns 5 elements instead of 25.

### Summary
In this article you have learned how easy it is to get a full REST API up and running using Speedment, Spring and the `generate-spring` plugin. There is [a free option of Speedment](https://speedment.com/initializer) so do try it out!
