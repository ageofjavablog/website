---
layout      : post
title       : What’s New in Speedment 3.0
description : Description of the new module system inspired by Jigsaw used in Speedment 3.0.
headline    : AGE OF JAVA
category    : java
modified    : 2016-10-31
tags        : [Commons, Java, Java 8, Java 9, Jigsaw, Maven, Module, Release, Speedment]
featured    : false
---

If you have followed my blog you know that I have been involved in the [open-source project Speedment](https://github.com/speedment/speedment) for a while. During the summer and fall I have worked a lot with finishing up the next big [3.0.0 release](https://github.com/speedment/speedment/releases/tag/3.0.0) of the toolkit. In this post I will showcase some of the cool new features we have built into the platform and also explain how you can get started!

![Spire standing in front of a sign that says Forest](https://lh3.googleusercontent.com/Qm93wI3PLGiIxuXXFUcO6CdPe9oy_81XpNlwnKORv1M8BNDEv6Am7KvpCCh4qSjurbVrSAoDdaEyRbcdgByU3TLzIDnbg2QDQ3GjNOcc9EI4cDK5bP4SoJCaez_J9z_5yOHDC4tc)

### New Module System

The biggest change from the previous version of Speedment, and the one that took us most time to get right, is the new module system. If you have been following the progress of the new [JDK 9 project Jigsaw](http://openjdk.java.net/projects/jigsaw/), you will recognize this subject. Previously, Speedment consisted of one big artifact called **com.speedment:speedment**. In addition to this, we had a few minor projects like the **speedment-maven-plugin** and **speedment-archetypes** that made the tool easier to use. There was several issues with this design. First, it was very tedious to develop in it since we often needed to rebuild the entire thing multiple times each day and every build could take minutes. It was also not very plugin-friendly since a plugin had to depend on the entire code base, even if it only modified a small group of classes.

In 3.0 however, **com.speedment** is actually a multi-module pom-project with a clear build order. Inside it are groups of artifacts, also realized as multi-module projects, that separate artifacts based on when they are needed. We now have the following artifact groups:

1.  **common-parent** contains artifacts that are mature, reusable in a number of situations and that doesn’t have any dependencies (except on our own [lightweight logging framework](https://github.com/speedment/speedment/tree/master/common-parent/logger)). Here you will find some of the core utilities of Speedment like [MapStream](https://github.com/speedment/speedment/tree/master/common-parent/mapstream) and [CodeGen](https://github.com/speedment/speedment/tree/master/common-parent/codegen).
2.  **runtime-parent** contains artifacts that are required by the end-user during the runtime of their application. We wanted to separate these into their own group to make sure that the final jar of the user’s app has as small footprint as possible.
3.  **generator-parent** contains artifacts related to the code generation and database analyzation parts of Speedment. These classes doesn’t require a graphical environment which is useful if you want to use Speedment as a general purpose code generator in a non-graphical environment.
4.  **tool-parent** contains all the artifacts used by the graphical Speedment tool. Here we put all our home-brewed JavaFX-components as well as resources like icons used by the UI.
5.  **build-parent** is a meta group that contains various artifacts that we build simply to make Speedment easier to use for the end user. Here we for an example have a number of shaded artifacts that you can use when you deploy your application on a server and the Maven Plugin that users use to launch Speedment as a Maven goal.
6.  **plugins-parent** is a whole new group where we put official plugins for Speedment that doesn’t quite fit into the general framework but that many users request. This makes it possible for us to automatically rebuild them in the general build cycle, making sure they are always up-to-date with the latest changes in the platform.
7.  **archetypes-parent** is a group of all the official Maven Archetypes for Speedment. This was previously a separate project but has now been lifted into the main project so that they can be automatically reinstalled every time Speedment is built.

All these groups are built in the same order as specified above. This makes it much easier to keep dependencies single-directional and the overall design of the system more comprehensive.

### So how do I use it?

The beautiful thing is that you barely have to change a thing! We automatically build an artifact that is called **com.speedment:runtime** that you can depend on in your project. It contains transitive dependencies to the exact set of artifacts that are required to run Speedment.

```xml
<dependency>
    <groupId>com.speedment</groupId>
    <artifactId>runtime</artifactId>
    <version>3.0.1</version>
    <type>pom</type>
</dependency>
```

When it is time for deployment, you simply replace this dependency with **com.speedment:runtime-deploy** and you will get a shaded jar with all the Speedment-stuff bundled together and ready to ship!

```xml
<dependency>
    <groupId>com.speedment</groupId>
    <artifactId>runtime-deploy</artifactId>
    <version>3.0.1</version>
</dependency>
```

For more details about the new release, go to [this official GitHub page](https://github.com/speedment/speedment) and fork it!
