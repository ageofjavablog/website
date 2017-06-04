---
layout      : post
title       : Quick Tip To Prevent Your Caches From Exploding
description : Prevent java caches from running out of memory by overriding a method in LinkedHashMap
headline    : AGE OF JAVA
category    : java
modified    : 2016-12-20
tags        : [BigInteger, Cache, Collections, HashMap, Java, Java 8, LinkedHashMap, Primes, Speedment, Webservice]
featured    : false
---

There are many scenarios when you can benefit from caching commonly used objects in your application, especially in web and micro-service oriented environments. The most simple type of caching you can do in Java is probably to introduce a private `HashMap` that you query before calculating an object to make sure you don't do the job twice.

Here is an example of that:

```java
public class PrimeService {

    private Map<Long, BigInteger> cache = new HashMap<>();

    public BigInteger getPrime(long minimum) {
        return cache.computeIfAbsent(minimum, 
            m -> BigInteger.valueOf(m).nextProbablePrime()
        );
    }
}
```

This is a quick solution to the problem, but sadly not very efficient. After a few months of people entering all kinds of crazy numbers into the service, we will have a very large `HashMap` that might cause us to run out of memory.

Here is a quick trick to solve that. Instead of using a `HashMap`, you can use a `LinkedHashMap` and simply override the method `removeEldestEntry`. That allows you to configure your own limit function to prevent the map from exploding.

```java
public class PrimeService {

    public final static int CACHE_MAX_SIZE = 1_000;

    private Map<Long, BigInteger> cache = new LinkedHashMap<>() {
        @Override
        protected boolean removeEldestEntry(Map.Entry<ID, Boolean> eldest) {
            return size() > PrimeService.CACHE_MAX_SIZE;
        }
    };

    public BigInteger getPrime(long minimum) {
        return cache.computeIfAbsent(minimum, 
            m -> BigInteger.valueOf(m).nextProbablePrime()
        );
    }
}
```

We have now successfully limited the cache, preventing it from explosion. Old entries will be removed as new ones are added. It should be noted that this soluton works well in small applications, but for more advanced scenarious you might want to use an external caching solution instead. If you are interested in caching entries in a SQL database without writing tons of SQL code, I would recommend [Speedment](https://github.com/speedment/speedment) since it is lightweight and has a very fluent API.

Until next time!
