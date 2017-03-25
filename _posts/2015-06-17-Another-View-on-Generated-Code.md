---
layout      : post
title       : Another View on Generated Code
description : How to generate code in multiple languages automatically using CodeGen
headline    : AGE OF JAVA
category    : java
modified    : 2015-06-17
tags        : [Automatic Programming, Generator, Java, Languages, Speedment, XML]
---

This is a follow-up on [my last article](https://ageofjavablog.github.io/website/Object-Oriented-Approach-to-Code-Generation) regarding CodeGen, the [MVC-oriented code generator for java](https://github.com/Pyknic/CodeGen).

One of the advantages of the modular approach demonstrated there is that multiple views can be attached to the same generator. By using different "factories", the rendering can go through the same pipeline and still be divided into different contexts.

In this example, we will render the same "concat-method" model using two different installers. One is the standard Java-installer and one is an example XML-installer.

The `XMLInstaller` will have two additional views, `MethodXMLView` and `FieldXMLView`. Those views will use some of the Java-views (like `TypeView`) since a type looks the same in both contexts.


###### Main.java
```java
public class Main {
    private final static TransformFactory 
        XML = new DefaultTransformFactory("XMLTransformFactory")
            .install(Method.class, MethodXMLView.class)
            .install(Field.class, FieldXMLView.class),

        JAVA = new JavaTransformFactory();

    public static void main(String... args) {
        final Generator gen = new DefaultGenerator(JAVA, XML);
        Formatting.tab("    ");

        gen.on(
            Method.of("concat", DefaultType.STRING).public_()
                .add(Field.of("str1", DefaultType.STRING))
                .add(Field.of("str2", DefaultType.STRING))
                .add("return str1 + str2;")
        ).forEach(code -> {
            System.out.println("-------------------------------------");
            System.out.println("  " + code.getInstaller().getName() + ":");
            System.out.println("-------------------------------------");
            System.out.println(code.getText());
        });
    }
}
```

###### MethodXMLView.java
```java
public class MethodXMLView implements Transform<Method, String> {
    @Override
    public Optional<String> render(Generator gen, Method model) {
        return Optional.of(
            "<method name=\"" + model.getName() + "\" type=\"" + 
            gen.on(model.getType()).get() + "\">" + nl() + indent(
                "<params>" + nl() + indent(
                    gen.metaOn(model.getFields())
                        .filter(c -> XML.equals(c.getFactory()))
                        .map(c -> c.getText())
                        .collect(Collectors.joining(nl()))
                ) + nl() + "</params>" + nl() +
                "<code>" + nl() + indent(
                    model.getCode().stream().collect(Collectors.joining(nl()))
                ) + nl() + "</code>"
            ) + nl() + "</methods>"
        );
    }
}
```

###### FieldXMLView.java
```java
public class FieldXMLView implements Transform<Field, String> {
    @Override
    public Optional<String> render(Generator gen, Field model) {
        return Optional.of("<field name=\"" + model.getName() + "\" />");
    }
}
```

### Results
When the application above is executed, the following will be outputed:

```
-------------------------------------
  JavaTransformFactory:
-------------------------------------
public String concat(String str1, String str2) {
    return str1 + str2;
}
-------------------------------------
  XMLTransformFactory:
-------------------------------------
<method name="concat" type="java.lang.String">
    <params>
        <field name="str1" />
        <field name="str2" />
    </params>
    <code>
        return str1 + str2;
    </code>
</methods>
```

The same model is rendered in two different languages using the same rendering pipeline.
