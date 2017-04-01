---
layout      : post
title       : The Terrors of Automated JSON Encoding
description : An expressive way of declaring which fields to include in automatic Java to JSON encoding.
headline    : AGE OF JAVA
category    : java
modified    : 2015-07-21
tags        : [Encoding, Java, JSON, Parsing, Security]
---

Suddenly and out of nowhere are all your clients passwords out on the web IN CLEAR TEXT! Huh?! Oh, It was just a nightmare. Or was it? **~Spooky voice~**

One of the dangers with automation is that computers lack ethics. They will happily do whatever you tell them to, even if it is horrible stuff you didn't really think through. One such thing I encountered recently was a feature that would automatically parse your database entities into JSON. Well, that's nice, isn't it? Less boilerplate code, higher readability, what's not to like? Well, the thing is that one of the many database tables happened to be the Account table and in it was (for some god-damn reason) passwords in plain text! The nightmare was real, for when the corrupt dictator of a system got its hands on the entity it immediately parsed it into JSON and sent it out for the world to see. Well, it didn't really make it to production so no harm done, but it **could** have. That's the point.

Adding some pepper and salt to the database was quickly fixed, but even if the passwords are salted it would be really stupid to include even hashed passwords in the public API. We needed some way to control what information to include in the JSON parsing and what to put at the bottom of a very big rock.

The solution I came up with was this:

```java
JsonEncoder imageEncoder = JsonEncoder
    .allOf(images)
    .put(Image.UPLOADER,
        JsonEncoder.allOf(users)
            .remove(User.AVATAR)   // Too large...
            .remove(User.PASSWORD) // Too secret...
        );

String json = imageEncoder.apply(someImage);
```

A JSON encoder is instantiated using the entity manager. The encoder can then be configured by adding and removing entity fields, even including foreign tables to create a hierarchical view of the domain. When the encoder is setup, entities can be passed to it to produce JSON strings. The code above now produce the following JSON:

```json
{
    "id" : 512,
    "title" : "A beautiful waterfall",
    "description" : "Taken in the summer 2013",
    "uploader" : {
        "id" : 57,
        "mail" : "emil@speedment.com",
        "firstname" : "Emil",
        "lastname" : "Forslund"
    },
    "img_data" : "..."
}
```

The implementation of the `JsonEncoder`-class can be found in[the source code of the Speedment project here](https://github.com/speedment/speedment/blob/master/plugin-parent/json-stream/src/main/java/com/speedment/plugins/json/JsonEncoder.java).

Hopefully this post didn't scare you too much! :)
