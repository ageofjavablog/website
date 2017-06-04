---
layout      : post
title       : How To - Insert and Read From a Database using Json
description : Use Speedment to generate Gson TypeAdapters for converting JSON objects to and from rows in a SQL database.
headline    : AGE OF JAVA
category    : java
modified    : 2016-08-10
tags        : [Automate, Automatic, Code Generation, CodeGen, Generate, Gson, Java, JSON, Modular, Plugins, Programming, Speedment, TypeAdapter]
featured    : false
---

In this article we will create a plugin for [Speedment](https://github.com/speedment/speedment) that generates serialization and deserialization logic using [Gson](https://github.com/google/gson) to make it super easy to map between database entities and JSON strings. This will help to showcase the extendability of the Speedment code generation while at the same time explore some of the cool features of the Gson library.

Speedment is a code generation tool for java that connects to a database and use it as a reference for generating entity and manager files for your project. The tool is very modular, allowing you to write your own plugins that modify the way the resulting code will look. One thing several people have mentioned on the Gitter chat is that Speedment entities are declared abstract which prevents them from being automatically deserialized. In this article we will look at how you can deserialize Speedment entities using Gson by automatically generating a custom TypeAdapter for each table in the database. This will not only give us better performance when working with JSON representations of database content, but might also serve as a general example on how you can extend the code generator to solve your problems.

![Spire and Duke downloading stuff from a database](https://lh5.googleusercontent.com/L62CgdVcpG0O0aC6u1BiY-HXy_9y08-oZ337B5ZjaWh4t2RJQwz6cEARAxlhN9ZC8hFwepJngXNg9EvNs3t6MHt8hR34LQy3B1EsbK4HfvuLnrdCON82Pm5H1De9t8U8DUUzj8SN)

### Step 1: Creating the Plugin Project
[In a previous article](/java/Generate-Customized-Code-With-Plugins) I went into detail on how to create a new plugin for Speedment, so here is the short version. Create a new maven project and set Speedment and Gson as dependencies.

###### pom.xml

```xml
<name>Speedment Gson Plugin</name>
<description>
    A plugin for Speedment that generates Gson Type Adapters for every 
    table in the database.
</description>

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <speedment.version>2.3.7</speedment.version>
</properties>

<dependencies>
    <dependency>
        <groupId>com.speedment</groupId>
        <artifactId>speedment</artifactId>
        <version>${speedment.version}</version>
    </dependency>

    <dependency>
        <artifactId>gson</artifactId>
        <groupId>com.google.code.gson</groupId>
        <version>2.6.2</version>
    </dependency>
</dependencies>
```

### Step 2: Create a Translator Class for the Type Adapter

Next we need to create the translator that will generate the new type adapter for us. A translator is a class that describes what name, path and content a generated file will have. To that it has a lot of convenience methods to make it easier to generate the code. The basic structure of the translator is shown below.

###### GeneratedTypeAdapterTranslator.java

```java
...
public GeneratedTypeAdapterTranslator(
       Speedment speedment, Generator gen, Table table) {
    super(speedment, gen, table, Class::of);
}

@Override
protected Class makeCodeGenModel(File file) {
    return newBuilder(file, getClassOrInterfaceName())
        .forEveryTable((clazz, table) -> {
            // Code generation goes here
        }).build();
}

@Override
protected String getClassOrInterfaceName() {
    return "Generated" + getSupport().typeName() + "TypeAdapter";
}

@Override
protected String getJavadocRepresentText() {
    return "A Gson Type Adapter";
}

@Override
public boolean isInGeneratedPackage() {
    return true;
}
...
```

Every translator is built up using a builder pattern that can be invoked using the `newBuilder()`-method. This becomes important later on when we want to modify an existing translator. The actual code is generated inside the builder’s `forEveryTable()`-method. It is a callback that will be executed for every table that is encountered in the scope of interest. In this case, the translator will only execute on one table at a time so the callback will only be executed once.

For complete sources for the `GeneratedTypeAdapterTranslator`-class, please go to [this github page](https://github.com/Pyknic/speedment-gson-plugin/blob/master/src/main/java/com/speedment/plugins/gson/GeneratedTypeAdapterTranslator.java).

### Step 3: Create Decorator for Modifying the Manager Interface

Generating a bunch of `TypeAdapters` is not enough though. We want to integrate the new code into the already existing managers. To do this, we need to define a decorator that will be applied to every generated manager after the default logic has been executed.

###### GeneratedManagerDecorator.java

```java
public final class GeneratedManagerDecorator 
implements TranslatorDecorator<Table, Interface> {

    @Override
    public void apply(JavaClassTranslator<Table, Interface> translator) {
        translator.onMake((file, builder) -> {
            builder.forEveryTable(Translator.Phase.POST_MAKE, 
            (clazz, table) -> {
                clazz.add(Method.of(
                    "fromJson", 
                    translator.getSupport().entityType()
                ).add(Field.of("json", STRING)));
            });
        });
    }
}
```

A decorator is similar to a translator, except it only defines the changes that should be done to an existing file. Every decorator executes in a specific phase. In our case we want to execute after the default code has been generated, so we select `POST_MAKE`. The logic we want to add is simple. In the interface, we want an additional method `fromJson(String)` to be required. We don’t need to define a `toJson` since every Speedment manager already has that from an inherited interface.

### Step 4: Create Decorator for Modifying the Manager Implementation

The manager implementation is a bit trickier to modify. We need to append it with a Gson instance as a member variable, a implementation for the new interface method we just added, an override for the `toJson`-method that uses Gson instead of the built-in serializer and we need to modify the manager constructor to instantiate Gson using our new `TypeAdapter`.

###### GeneratedManagerImplDecorator.java

```java
public final class GeneratedManagerImplDecorator 
implements TranslatorDecorator<Table, Class> {

    @Override
    public void apply(JavaClassTranslator<Table, Class> translator) {
        final String entityName = translator.getSupport().entityName();
        final String typeAdapterName = "Generated" + entityName + 
            "TypeAdapter";
        final String absoluteTypeAdapterName =
            translator.getSupport().basePackageName() + ".generated." + 
            typeAdapterName;

        Final Type entityType = translator.getSupport().entityType();

        translator.onMake((file, builder) -> {
            builder.forEveryTable(Translator.Phase.POST_MAKE, 
            (clazz, table) -> {

                // Make sure GsonBuilder and the generated type adapter 
                // are imported.
                file.add(Import.of(Type.of(GsonBuilder.class)));
                file.add(Import.of(Type.of(absoluteTypeAdapterName)));

                // Add a Gson instance as a private member
                clazz.add(Field.of("gson", Type.of(Gson.class))
                    .private_().final_()
                );

                // Find the constructor and define gson in it
                clazz.getConstructors().forEach(constr -> {
                    constr.add(
                        "this.gson = new GsonBuilder()",
                        indent(".setDateFormat(\"" + DATE_FORMAT + "\")"),
                        indent(".registerTypeAdapter(" + entityName + 
                            ".class, new " + typeAdapterName + "(this))"),
                        indent(".create();")
                    );
                });

                // Override the toJson()-method
                clazz.add(Method.of("toJson", STRING)
                    .public_().add(OVERRIDE)
                    .add(Field.of("entity", entityType))
                    .add("return gson.toJson(entity, " + entityName + 
                         ".class);"
                    )
                );

                // Override the fromJson()-method
                clazz.add(Method.of("fromJson", entityType)
                    .public_().add(OVERRIDE)
                    .add(Field.of("json", STRING))
                    .add("return gson.fromJson(json, " + entityName + 
                        ".class);"
                    )
                );
            });
        });
    }
}
```

### Step 5: Install All the New Classes into the Platform

Once we have created all the new classes, we need to create a component and a component installer that can be referenced from any project where we want to use the plugin.

###### GsonComponent.java

```java
public final class GsonComponent extends AbstractComponent {

    public GsonComponent(Speedment speedment) {
        super(speedment);
    }

    @Override
    public void onResolve() {
        final CodeGenerationComponent code = 
            getSpeedment().getCodeGenerationComponent();

        code.put(Table.class, 
            GeneratedTypeAdapterTranslator.KEY, 
            GeneratedTypeAdapterTranslator::new
        );
        code.add(Table.class, 
            StandardTranslatorKey.GENERATED_MANAGER, 
            new GeneratedManagerDecorator()
        );
        code.add(Table.class,
             StandardTranslatorKey.GENERATED_MANAGER_IMPL, 
             new GeneratedManagerImplDecorator()
        );
    }

    @Override
    public Class<GsonComponent> getComponentClass() {
        return GsonComponent.class;
    }

    @Override
    public Software asSoftware() {
        return AbstractSoftware.with("Gson Plugin", "1.0", APACHE_2);
    }

    @Override
    public Component defaultCopy(Speedment speedment) {
        return new GsonComponent(speedment);
    }
}
```

###### GsonComponentInstaller.java

```java
public final class GsonComponentInstaller 
implements ComponentConstructor<GsonComponent> {

    @Override
    public GsonComponent create(Speedment speedment) {
        return new GsonComponent(speedment);
    }
}
```

### Usage

When we want to use our new plugin in a project, we simply add it as a dependency both in the dependency section in the pom and as a dependency under the speedment maven plugin. We then add a configuration tag to the plugin like below:

```xml
<plugin>
    <groupId>com.speedment</groupId>
    <artifactId>speedment-maven-plugin</artifactId>
    <version>${speedment.version}</version>

    <dependencies>
        <dependency>
            <groupId>com.speedment.plugins</groupId>
            <artifactId>gson</artifactId>
            <version>1.0.0</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.39</version>
        </dependency>
    </dependencies>

    <configuration>
        <components>
            <component implementation="com.speedment.plugins.gson.GsonComponentInstaller" />
        </components>
    </configuration>
</plugin>
```

We can then regenerate our code and we should then have access to the new serialization and deserialization logic.

```java
final String pippi = "{" + 
    "\"id\":1," +
    "\"bookId\":-8043771945249889258," +
    "\"borrowedStatus\":\"AVAILABLE\"," + 
    "\"title\":\"Pippi Långström\"," + 
    "\"authors\":\"Astrid Lindgren\"," + 
    "\"published\":\"1945-11-26\"," + 
    "\"summary\":\"A story about the world's strongest little girl.\"" + 
    "}";

books.fromJson(pippi).persist();
```

### Summary

In this article we have created a new Speedment plugin that generated Gson `TypeAdapters` for every table in a database and integrates those adapters with the existing manager generation. If you want more examples on how you can use the Speedment code generator to increase your productivity, [check out the GitHub page](https://github.com/speedment/speedment)!

Until next time!
