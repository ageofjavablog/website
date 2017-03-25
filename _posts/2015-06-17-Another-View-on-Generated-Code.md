= Another View on Generated Code
:published_at: 2015-06-17
:hp-tags: Automatic Programming, Code, Generator, Java, Languages, Multiple, Speedment, XML
:source-highlighter: pygments

This is https://pyknic.github.io/2015/06/02/Object-Oriented-Approach-to-Code-Generation.html[a follow-up on my last article regarding CodeGen], the https://github.com/Pyknic/CodeGen[object-oriented code generator for java].

One of the advantages of the modular approach demonstrated there is that multiple views can be attached to the same generator. By using different "factories", the rendering can go through the same pipeline and still be divided into different contexts.

In this example, we will render the same "concat-method" model using two different installers. One is the standard java-installer and one is an example XML-installer.

The `XMLInstaller` will have two additional views, `MethodXMLView` and `FieldXMLView`. Those views will use some of the Java-views (like `TypeView`) since a type looks the same in both contexts.

[]
.Main.java
<pre name="code" class="java">public class Main {
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
}</pre>
<h4>MethodXMLView.java</h4>
<pre name="code" class="java">public class MethodXMLView implements Transform&lt;Method, String&gt; {
    @Override
    public Optional&lt;String&gt; render(Generator gen, Method model) {
        return Optional.of(
            "&lt;method name=\"" + model.getName() + "\" type=\"" + 
            gen.on(model.getType()).get() + "\"&gt;" + nl() + indent(
                "&lt;params&gt;" + nl() + indent(
                    gen.metaOn(model.getFields())
                        .filter(c -&gt; XML.equals(c.getFactory()))
                        .map(c -&gt; c.getText())
                        .collect(Collectors.joining(nl()))
                ) + nl() + "&lt;/params&gt;" + nl() +
                "&lt;code&gt;" + nl() + indent(
                    model.getCode().stream().collect(Collectors.joining(nl()))
                ) + nl() + "&lt;/code&gt;"
            ) + nl() + "&lt;/methods&gt;"
        );
    }
}</pre>
<h4>FieldXMLView.java</h4>
<pre name="code" class="java">public class FieldXMLView implements Transform&lt;Field, String&gt; {
    @Override
    public Optional&lt;String&gt; render(Generator gen, Field model) {
        return Optional.of("&lt;field name=\"" + model.getName() + "\" /&gt;");
    }
}</pre>
<h4>Results</h4>
When the application above is executed, the following will be outputed:<br />
<br />
<pre>-------------------------------------
  JavaTransformFactory:
-------------------------------------
public String concat(String str1, String str2) {
    return str1 + str2;
}
-------------------------------------
  XMLTransformFactory:
-------------------------------------
&lt;method name="concat" type="java.lang.String"&gt;
    &lt;params&gt;
        &lt;field name="str1" /&gt;
        &lt;field name="str2" /&gt;
    &lt;/params&gt;
    &lt;code&gt;
        return str1 + str2;
    &lt;/code&gt;
&lt;/methods&gt;</pre>
<br />
The same model is rendered in two different languages using the same rendering pipeline.