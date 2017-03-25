---
layout      : post
title       : Lightweight Framework for Tiny Web Services
description : Introducing a small open source library for writing web services
headline    : AGE OF JAVA
category    : projects
modified    : 2015-05-28
tags        : [Framework, Java, Lightweight, REST, Service, Web]
---

Some while ago I was sitting on the train with my laptop, half-heartedly coding on some minor side project to make the time pass a little bit faster. I have honestly forgotten what the project was about, but I remember thinking that it would have been so much easier to get a prototype up if I didn't have to mess with all the server mumbo-jumbo.

I usually write most of my web projects on top of [NanoHttpd](https://github.com/NanoHttpd/nanohttpd) since it is lightweight and is easy to get started with, but after a while I started fantasize about how it could be _even easier_ to use. That was what led me to create [ServiceKit](https://github.com/Pyknic/ServiceKit).

Imagine you are creating a web app that users can communicate with using a REST api. You have two commands, "register" and "login". They both take an username and a password and should return a status of whether the operation succeeded. The code might look something like this:

###### LoginProject.java
```java
public final class LoginProject extends NanoHTTPD {

    @Override
    public Response serve(IHTTPSession session) {
        final Map<String, String> body = new HashMap<>();
        final NanoHTTPD.Method method = session.getMethod();
        final String uri = session.getUri();
        final String command = uri.substring(uri.indexOf("/") + 1);

        if (Method.PUT.equals(method) || Method.POST.equals(method)) {
            try {
                // Make sure the body is downloaded.
                session.parseBody(body);
            } catch (IOException ex) {
                return new Response(Response.Status.INTERNAL_ERROR, MIME_PLAINTEXT,
                    "INTERNAL SERVER ERROR: IOException: " + ex.getMessage()
                );
            } catch (ResponseException ex) {
                return new Response(ex.getStatus(), MIME_PLAINTEXT, ex.getMessage());
            }
        }
        
        final Map<String, String> params = session.getParms();
        final String username = params.get("username");
        final String password = params.get("password");

        final String msg;
        switch (command) {
            case "register":
                if (tryRegister(username, password)) {
                    msg = "{'success':true}";
                } else {
                    msg = "{'success':false,'msg':'Username already taken.'}";
                }
                break;
            case "login":
                if (tryLogin(username, password)) {
                    msg = "{'success':true}";
                } else {
                    msg = "{'success':false,'msg':'Wrong username or password.'}";
                }
                break;
            default:
                msg = "{'success':false,'msg':'Unknown command.'}";
                break;
        }
        
        return new NanoHTTPD.Response(msg);
    }

    ...
    
    public static void main(String... params) {
        ServerRunner.run(LoginProject.class);
    }

    public LoginProject() {
        super (1337);
    }
}
```

With service kit, the code could be reduced significantly by using annotations to declare new services and automatically parsing in- and output to JSON.

###### LoginProject.java
```java
public final class LoginProject extends HttpServer {
    
    @Service({"username", "password"})
    public static Status login(String username, String password) {
        if (tryLogin(username, password)) {
            return Status.success();
        } else {
            return Status.failure("Wrong username or password.");
        }
    }
    
    @Service({"username", "password"})
    public static Status register(String username, String password) {
        if (tryRegister(username, password)) {
            return Status.success();
        } else {
            return Status.failure("Username already taken.");
        }
    }
    
    ...
    
    private static class Status {
        private final boolean success;
        private final String msg;

        private Status(boolean success, String msg) {
            this.success = success;
            this.msg     = msg;
        }

        public static Status success() {
            return new Status(true, "");
        }
        
        public static Status failure(String msg) {
            return new Status(false, msg);
        }
    }
    
    public static void main(String... param) {
        HttpServer.run(LoginProject.class);
    }
    
    public LoginProject() {
        super (1337);
    }
}
```

ServiceKit uses the [Googles GSON project](https://github.com/google/gson) for the parsing so no specific interface is required to serialize and deserialize the parameters. That makes the code more readable and prevents bugs related to malformed results etc.

The example above will give you two commands accessible over HTTP like this:

> http://example.com/login?username=YourName&password=YourPassword

> http://example.com/register?username=YourName&password=YourPassword

The source code for ServiceKit [is available here on GitHub](https://github.com/Pyknic/ServiceKit)! 

Let me know what you think about this approach!
