---
layout      : post
title       : Hack Speedment into Your Own Personal Code Generator
description : Tutorial on how to hack the Speedment Code Generator to create any kind of code, not just database ORM entities.
headline    : AGE OF JAVA
category    : java
modified    : 2016-11-21
tags        : [Automated, Code, Code Generation, CodeGen, Custom, Dynamic, Generate, Hack, Java, Java 8, Plugin, Programming, Speedment]
featured    : false
---

[Speedment is an Open Source toolkit](https://github.com/speedment/speedment) that can be used to generate Java entities and managers for communicating with a database. This is great if you need an Object Relational Mapping of the domain model, but in some cases, you might want to generate something completely different using your database as the template. In this article I am going to show you a hack that you can use to take over that code generator in Speedment and use it for your own personal purposes. At the end of the article we will have a completely blank code generator that we can program to do our bidding!

![Spire and Duke dressed as Neo from Matrix](https://lh3.googleusercontent.com/nhxCiGJnv4ynMY7iiDwicFBDNtK3EMs9Pi4EQfw06wCgycN0i2e9sgAshaGUjrbCo3yMalpQdVJCfHwSF_hqV2IeYmidHHOL8AkgMjKpDRSIzHpla-wqKooHGnuQ6J2bbPU82ZIh)

### Background

Speedment is designed to operate as a plugin to Maven. By invoking various new Maven goals we can instruct Speedment to connect to a database, generate source code and also remove any generated files from our project. It also contains a graphical user interface that makes it easy to configure the generation job based on metadata collected from our database. Now, imagine all this information that we can collect from analyzing that metadata. We know what tables exist, we know of all the constraints they have and what types the individual columns have. There are probably millions of use-cases where we can benefit from automatically generating stuff from that metadata. Following the steps in this article, we can do all those things.

![Screenshot of the Speedment User Interface](https://lh4.googleusercontent.com/UUm_oO_opFhRMh8oalWhpLq9ErgAJTXrTBDPQjWMAJarY922Pl-yMseN4JhnLeAcmhAHT9ga001glER6FFBNKlWvfMbihE1T8X4hP3XfvAc2B-buScCEeHd9gYTZJWoOzMhCuAh4)

### Step 1: Set Up a Regular Speedment Project

Create a new Maven Project and add the following to the `pom.xml`-file:

###### pom.xml

```xml
<properties>
    <speedment.version>3.0.1</speedment.version>
    <mysql.version>5.1.39</mysql.version>
</properties>

<dependencies>
    <dependency>
        <groupId>com.speedment</groupId>
        <artifactId>runtime</artifactId>
        <version>${speedment.version}</version>
        <type>pom</type>
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
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>${mysql.version}</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

We have added Speedment as a runtime dependency and configured the Maven plugin to use the standard MySQL JDBC driver to connect to our database. Great! You now have access to a number of new Maven goals. For an example, if we wanted to launch the Speedment UI we could do that by running:

```bash
mvn speedment:tool
```

If we did that now, Speedment would launch in normal mode allowing us to connect to a database and from it generate entities and managers for communicating with that database using Java 8 streams. That is not what we want to do this time around. We want to hack it so that it does exactly what we need it to do. We therefore continue to modify the pom.

### Step 2: Modify the Plugin Declaration

Speedment is built up in a modular fashion with different artifacts responsible for different tasks. All the pre-existing generator tasks are located in an artifact called "com.speedment.generator:generator-standard". That is where we are going to strike! By removing that artifact from the classpath we can prevent Speedment from generating anything we don’t want it to.

We modify the `pom` as follows:

```xml
...
<plugin>
    <groupId>com.speedment</groupId>
    <artifactId>speedment-maven-plugin</artifactId>
    <version>${speedment.version}</version>
    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.version}</version>
        </dependency>

        <!-- Add this: -->
        <dependency>
            <groupId>com.speedment</groupId>
            <artifactId>tool</artifactId>
             <version>${speedment.version}</version>
             <type>pom</type>
             <exclusions>
                 <exclusion>
                     <groupId>com.speedment.generator</groupId>
                     <artifactId>generator-standard</artifactId>
                 </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
</plugin>
...
```

What is that? We exclude a dependency by adding one? How can that even work? Well, Speedment is designed to include as little code as possible unless it is explicitly needed by the application. The "com.speedment:tool-artifact" is a dependency to the maven plugin already, and by mentioning it in the `<dependencies>`-section of the maven plugin we can append settings to its configuration. In this case, we say that we want the plugin to have access to the tool except we don’t want the standard generator.

There is a problem here though. If we try and launch the speedment:tool goal, we will get an exception. The reason for this is that Speedment _expects_ the standard translators to be on the classpath.

Here is where the second ugly hack comes in. In our project, we create a new package called `com.speedment.generator.standard` and in it define a new java file called `StandardTranslatorBundle.java`. As it turns out, that is the only file that Speedment really needs to work. We give it the following content:

###### StandardTranslatorBundle.java

```java
package com.speedment.generator.standard;

import com.speedment.common.injector.InjectBundle;
import java.util.stream.Stream;

public final class StandardTranslatorBundle implements InjectBundle {
    @Override
    public Stream<Class<?>> injectables() {
        return Stream.empty();
    }
}
```

Next, we need to replace the excluded artifact with our own project so that the plugin never realizes that the files are missing. We return to the `pom.xml`-file and add our own project to the `<dependencies>`-section of the `speedment-maven-plugin`. The full pom file looks like this:

###### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" 
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
    http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <modelVersion>4.0.0</modelVersion>
  <groupId>com.github.pyknic</groupId>
  <artifactId>speedment-general-purpose</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <speedment.version>3.0.1</speedment.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>com.speedment</groupId>
      <artifactId>runtime</artifactId>
      <version>${speedment.version}</version>
      <type>pom</type>
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
            <groupId>com.speedment</groupId>
            <artifactId>tool</artifactId>
            <version>${speedment.version}</version>
            <type>pom</type>
            <exclusions>
              <exclusion>
                <groupId>com.speedment.generator</groupId>
                <artifactId>generator-standard</artifactId>
              </exclusion>
            </exclusions>
          </dependency>
          <dependency>
            <groupId>com.github.pyknic</groupId>
            <artifactId>speedment-general-purpose</artifactId>
            <version>1.0.0-SNAPSHOT</version>
          </dependency>   
          <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.39</version>
          </dependency>
        </dependencies>
      </plugin>
    </plugins>
  </build>
</project>
```

If we now build our project and then run the goal speedment:tool, we should be able to launch the graphical user interface without problem. If we connect to the database and then press "Generate", nothing will happen at all! We have successfully hacked Speedment into doing absolutely nothing!

### Step 3: Turn Speedment Into What You Want It To Be

Now when we have a fresh, clean Speedment, we can start turning it into the application we want it to be. We still have a powerful user interface where we can configure the code generation based on a database model. We have an expressive library of utilities and helper classes to work with generated code. And above all, we have a structure for analyzing the database metadata in an object-oriented fashion.

To learn more about how to write your own code generation templates and hook them into the platform, [check out this article](http://www.ageofjava.com/2016/04/how-to-generate-customized-java-8-code.html). You should also check out the [Speedment GitHub page](https://github.com/speedment/speedment) to learn how the existing generators work (the ones we just disabled) and maybe get some inspiration on how you can build your own.

Until next time, continue hacking!
