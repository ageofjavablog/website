---
layout      : post
title       : Streaming over Maps with Java 8
description : A helper class in Java for streaming over maps with lambda expressions just like collections.
headline    : AGE OF JAVA
category    : java
modified    : 2015-12-08
tags        : [Beautiful, Collections, Java, Java 8, Lambda, Maps, Programming, Speedment, Stream]
featured    : true
---

In this article I will show you how [Speedment Open Source](http://speedment.org) streams efficiently over standard Java maps, expanding the `Stream` interface into something called a `MapStream`! This addition will make it easier to keep your streams concrete and readable even in complex scenarios. Hopefully this will allow you to keep streaming without prematurely collecting the result.

One of the largest features in Java 8 was the ability to stream over collections of objects. By adding the `.stream()`-method into the `Collection` interface, every collection in the java language was suddenly expanded with this new ability. Other data structures like the `Map`-interface, does not implement the method as they are not strictly speaking collections.

The `MapStream` will take two type parameters, a key and a value. It will also extends the standard `Stream` interface by specifying `Map.Entry<K, V>` as type parameter. This will allow us to construct a `MapStream` directly from any Java `Map`.

```java
public interface MapStream<K, V> extends Stream<Map.Entry<K, V>> {
    ...
}
```

The concept of polymorphism tells us that a child component may change the return type of an overidden method as long as the new return type is a more concrete implementation of the old return type. We will use this when defining the `MapStream` interface so that for each chaining operation, a `MapStream` is returned instead of a `Stream`.

```java
public interface MapStream<K, V> extends Stream<Map.Entry<K, V>> {

    @Override
    MapStream<K, V> filter(Predicate<? super Map.Entry<K, V>> predicate);

    @Override
    MapStream<K, V> distinct();

    @Override
    MapStream<K, V> sorted(Comparator<? super Map.Entry<K, V>> comparator);

    ...
}
```

Some operations will still need to return an ordinary `Stream`. If the operation change the type of the streamed element, we can’t ensure that the new type will be a `Map.Entry`. We can, however, add additional methods for mapping between types with Key-Value pairs.

```java
    @Override
    <R> Stream<R> map(Function<? super Map.Entry<K, V>, ? extends R> mapper);

    <R> Stream<R> map(BiFunction<? super K, ? super V, ? extends R> mapper);
```

In addition to the `Function` that let the user map from an `Entry` to something else, he or she can also map from a Key-Value-pair to something else. This is convenient, sure, but we can also add more specific mapping operations now that we are working with pairs of values.

```java
    <R> Stream<R> mapKey(BiFunction<? super K, ? super V, ? extends R> mapper);

    <R> Stream<R> mapValue(BiFunction<? super K, ? super V, ? extends R> mapper);
```

The difference doesn’t look like much, but the difference is apparent when using the API:

```java
// With MapsStream
final Map<String, List<Long>> map = ...;
MapStream.of(map)
    .mapKey((k, v) -> k + " (" + v.size() + ")")
    .flatMapValue((k, v) -> v.stream())
    .map((k, v) -> k + " >> " + v)
    .forEach(System.out::println);

// Without MapStream
final Map<String, List<Long>> map = ...;
map.entrySet().stream()
    .map(e -> new AbstractMap.SimpleEntry<>(
         e.getKey() + " (" + e.getValue().size() + ")"),
         e.getValue()
    )
    .flatMap(e -> e.getValue().stream()
        .map(v -> new AbstractMap.SimpleEntry<>(e.getKey(), v))
    )
    .map(e -> e.getKey() + " >> " + e.getValue())
    .forEach(System.out::println);
```

The full implementation of `MapStream` [can be found here](https://github.com/Pyknic/MapStream). If you are interested in more cool stuff, have a look at the [Speedment Github page](https://github.com/speedment/speedment). Have fun streaming!</p>
