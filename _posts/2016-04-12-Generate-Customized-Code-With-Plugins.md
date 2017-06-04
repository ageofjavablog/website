---
layout      : post
title       : How to Generate Customized Java 8 Code with Plugins
description : Tutorial on how to create custom plugins for Speedment to control how the Java 8 code is generated.
headline    : AGE OF JAVA
category    : java
modified    : 2016-04-12
tags        : [Automatic, Code Generation, Component, Custom, Enum, Java, Java 8, Modular, Plugin, Programming, Speedment]
featured    : false
---

One thing most programmers hate is to write boilerplate code. Endless hours are spent setting up entity classes and configuring database connections. To avoid this you can let a program like [Speedment Open Source](https://github.com/speedment/speedment) generate all this code for you. This makes it easy to get a database project up and running with minimal manual labour, but how do you maintain control of the written code when large parts of it is handed over to a machine?

Say that you have a database with a table "user" which has a column "gender", and you want that implemented as an `enum` in java. If you run Speedment and use it to generate code, the "gender" field will be represented as a `String`. The reason for this is that there isn’t any built-in mappers for converting between database `ENUM`s and custom java classes. This is one of those cases when you might feel that the generator is taking away control for you. Well, fear not, for since the [2.3 Hamilton release](https://github.com/speedment/speedment/releases/tag/2.3.0), you can get this same control by creating your own plugin for Speedment!

### The Goal of this Article

![Spire and Duke are plugging Components into Speedment](https://3.bp.blogspot.com/-RW17_QWuHJ4/Vww7_A44imI/AAAAAAAAEwA/p-wUFmNjNeYv2XmxpMD03DHtP22eEullQCLcB/s640/PLUGIN2.png)

In this example we have a database schema with a table called "Person". A person has an id, a name and a gender. The gender is declared as an `ENUM` with three possible values: "Male", "Female" and "Other". If we use the default settings in Speedment to generate this class, Speedment will consider the `ENUM` a `String`. There are some issues with this however. For an example, if you want to persist a new person into the database, there is nothing that prevents you from spelling a gender wrong and getting an exception when you do the insert. Instead, we want to define a java `enum` with the specified alternatives as constants. What would make the generated code more secure and easier to use.

We can achieve this using a plugin for Speedment!

### Creating the Plugin Project

To do any custom modifications to the Speedment platform we will need to define a plugin. A plugin is a piece of software that can be plugged into the Speedment runtime from the `pom.xml`-file. The plugin resides in its own maven project and can be shared between projects.

Begin by creating a new Maven Project and declare Speedment as a dependency. You will not need the `speedment-maven-plugin` in this project.

```xml
<dependency>
    <groupId>com.speedment</groupId>
    <artifactId>speedment</artifactId>
    <version>${speedment.version}</version>
</dependency>
```

The plugin system revolves around two interfaces; `Component` and `ComponentConstructor`. A `Component` is a pluggable piece of software that can be executed as part of the Speedment lifecycle. Every component has a number of stages in which it is allowed to execute. These are "initialize", "load", "resolve" and "start".

The `ComponentConstructor` is a lightweight type that has a default constructor and a method for initializing new instances of the custom component. This is used by the maven plugin to setup the new code.

Here is how our two implementations will look:

###### CustomMappingComponent.java

```java
public final class CustomMappingComponent 
extends AbstractComponent {

    CustomMappingComponent(Speedment speedment) {
        super(speedment);
    }

    @Override
    public void onResolve() {
        // Resolve logic here...
    }

    @Override
    public Class<CustomMappingComponent> getComponentClass() {
        return CustomMappingComponent.class;
    }

    @Override
    public Software asSoftware() {
        return AbstractSoftware.with(
            "Custom Mapping Component", 
            "1.0", 
            APACHE_2
        );
    }

    @Override
    public Component defaultCopy(Speedment speedment) {
        return new CustomMappingComponent(speedment);
    }
}
```

###### CustomMappingComponentInstaller.java

```java
public final class CustomMappingComponentInstaller 
implements ComponentConstructor<CustomMappingComponent> {

    @Override
    public Component create(Speedment speedment) {
        return new CustomMappingComponent(speedment);
    }
}
```

We now have a bare-bone plugin that can be added to a Speedment project. The next step is to define the logic that maps between strings and genders. To this this, first we need the `Gender` enum.

###### Gender.java

```java
public enum Gender {
    MALE   ("Male"), 
    FEMALE ("Female"),
    OTHER  ("Other");

    private final String databaseName;

    Gender(String databaseName) {
        this.databaseName = databaseName;
    }

    public String getDatabaseName() {
        return databaseName;
    }
}
```

If you store the enum values in upper-case in the database, this class could be much shorter since you could simply use the `Enum.name()`-method to get the database name, but this approach is better if you want flexibility in naming the constants.

Now for the final piece. We need to declare a type that implements the `TypeMapper`-interface in Speedment. A type mapper is really simple. It contains two methods for mapping to and from the database type as well as methods for retrieving the java class of both types.

###### StringToGenderMapper.java

```java
public final class StringToGenderMapper implements TypeMapper<String, Gender> {

    @Override
    public Class<Gender> getJavaType() {
        return Gender.class;
    }

    @Override
    public Class<String> getDatabaseType() {
        return String.class;
    }

    @Override
    public Gender toJavaType(String value) {
        if (value == null) {
            return null;
        } else {
            return Stream.of(Gender.values())
                .filter(g -> g.getDatabaseName().equals(value))
                .findAny()
                .orElseThrow(() -> 
                    new UnsupportedOperationException(
                        "Unknown gender '" + value + "'."
                    )
                );
        }
    }

    @Override
    public String toDatabaseType(Gender value) {
        if (value == null) return null;
        else return value.getDatabaseName();
    }

    @Override
    public boolean isIdentityMapper() {
        return false;
    }
}
```

This new mapper also need to be installed in the Speedment platform. We can do that from the component we created earlier by modifying the `onResolve()`-method:

###### CustomMappingComponent.java

```java
@Override
public void onResolve() {
    // Resolve logic here...
    getSpeedment().getTypeMapperComponent()
        .install(StringToGenderMapper::new);
}
```

Our new plugin is now done! Build the project and you are set to go!

### Using the Plugin

To use a plugin in a project, you only need to modify the `pom.xml`-file of that project. Open up an existing Speedment project and locate the `pom.xml`-file. In it, you should be able to find the `speedment-maven-plugin`. To make your own plugin accessible for the maven plugin, you need to add it as a dependency inside the `<plugin>`-tag and add the `ComponentInstaller` to the configuration. Here is an example of how it can look:

###### pom.xml

```xml
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

        <!-- Our plugin project -->
        <dependency>
            <groupId>com.github.pyknic</groupId>
            <artifactId>custom-mapping-component</artifactId>
            <version>1.0.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <configuration>
        <components>
            <!-- Path to the component installer -->
            <component implementation="
com.github.pyknic.salesinfo.plugin.CustomMappingComponentInstaller
            " />
        </components>
    </configuration>
</plugin>
```

You also need to add the project as a runtime dependency since the new `Gender`-enum must be accessible from the generated code.

```xml
<dependencies>
    ...
    <dependency>
        <groupId>com.github.pyknic</groupId>
        <artifactId>custom-mapping-component</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </dependency>
    ...
</dependencies>
```

### Trying it out

That’s it! The plugin is installed! If you want to a particular column to be mapped to a `Gender` instead of a `String`, you can go into the User Interface, navigate to the particular column in the "Project Tree" and select your new Type Mapper in the dropdown list.

![screenshot of the Speedment user interface](https://lh6.googleusercontent.com/V3X1vuUHpUsLMuGvmhqkQ8QA6ya5lj5x2PTTvZldRw_Hk6p2YdfWK9fVcgVgz_MNC1ls4VtOXou38QLivNdIHqVeiLBHP9-VpLuuZQT_TVQHVB5uYvZAa-u1qY2uQBF4GmP_05Q7 "Speedment User Interface - Main Window")

If you want to see a list of all the components and/or type mappers loaded into the platform, you can also go to "About" → "Components..." in the UI. There you should see the new component.

![screenshot of the components dialog in the Speedment user interface](https://lh4.googleusercontent.com/p_QK6vO4AdBLAp2Gvhde_YZP8fBqEzXr5xM5DMQ19g9hUo5YOHNMUIruvlXS5ZI4_ZlGE6GsV6tYRee0dvQDoP07Ix_6OhEAqg_6dwaBEAWwRWWgzb6LE42vFiXSRIexaPgYrvzm "Speedment User Interface - Components Dialog")

### Summary

In this article you have learned how to create a custom plugin for [Speedment](https://github.com/speedment/speedment/) that integrates a new Type Mapper from a String to a Gender enum. You have also learned how you can see which components are loaded into the platform and select which type mapper you want to use for each column.

**PS:** If you create some cool new mappers for your Speedment project, consider sharing them with the community in [our Gitter chat](https://gitter.im/speedment/speedment)!
