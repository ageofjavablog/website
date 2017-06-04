---
layout      : post
title       : Event-Sourcing and CQRS in Practise
description : Tutorial on building Event Sourced Microservices in Java by using Speedment with a SQL Database
headline    : AGE OF JAVA
category    : java
modified    : 2016-10-11
tags        : [CQRS, Database, Distributed, Event, Event Sourcing, Java, Manager, Materialized Object View, Microservice, MOV, Speedment, SQL, Stream, Synchronization, Tutorial]
featured    : true
---

Anyone that has tried to implement a fully ACID compliant system knows that there are a lot of considerations you have to do. You need to make sure database entities can be freely created, modified and deleted without the risk of errors, and in most cases, the solution will be at the cost of performance. One methodology that can be used to get around this is to design the system based on a series of events rather than mutable states. This is generally called Event Sourcing.

In this article I will showcase a demo application that uses the Open Source toolkit [Speedment](https://github.com/speedment/speedment) to rapidly get a scalable event-sourced database application up and running. Full source code for the example [is available here](https://github.com/Pyknic/speedment-sauna-example).

![Sauna.png](https://lh5.googleusercontent.com/Vw_FXOwoRQf8FLi-TxjL5z4VYESLs1EsQ2V5k3fXAZGTXFGgwnYZjxsFvC8VUtQU-XDeuOO9x9hA1Wvaqoqop5iH-rBISIV8OxQG43j9pkA_MJ37c174mYpaSLSzIMhdq2Lj1FmL)

## What is Event Sourcing?

In a typical relational database system you store the _state_ of an entity as a row in a database. When the state changes, the application modifies the row using an UPDATE or a DELETE-statement. A problem with this method is that it adds a lot of requirements on the database when it comes to making sure that no row is changed in a way that puts the system in an illegal state. You don’t want anyone to withdraw more money than they have in their account or bid on an auction that has already been closed.

In an event-sourced system, we take a different approach to this. Instead of storing the _state_ of an entity in the database, you store the _series of changes_ that led to that state. An event is immutable once it is created, meaning that you only have to implement two operations, CREATE and READ. If an entity is updated or removed, that is realized using the creation of an “update” or “remove” event.

An event sourced system can easily be scaled up to improve performance, as any node can simply download the event log and replay the current state. You also get better performance due to the fact that writing and querying is handled by different machines. This is referred to as CQRS (Command-Query Responsibility Segregation). As you will see in the examples, we can get an eventually consistent materialized view up and running in a very little time using the Speedment toolkit.

## The Bookable Sauna

To showcase the workflow of building an event sourced system we will create a small application to handle the booking of a shared sauna in a housing complex. We have multiple tenants interested in booking the sauna, but we need to guarantee that the shy tenants never accidentally double-book it. We also want to support multiple saunas in the same system.

To simplify the communication with the database, we are going to use the [Speedment toolkit](https://github.com/speedment/speedment). Speedment is a java tool that allows us to generate a complete domain model from the database and also makes it easy to query the database using optimized Java 8 streams. Speedment is available under the Apache 2-license and there are a lot of great examples for different usages [on the Github page](https://github.com/speedment/speedment/wiki/Speedment-API-Quick-Start).

#### Step 1: Define the Database Schema

The first step is to define our (MySQL) database. We simply have one table called “booking” where we store the events related to booking the sauna. Note that a booking is an event and not an entity. If we want to cancel a booking or make changes to it, we will have to publish additional events with the changes as new rows. We are not allowed to modify or delete a published row.

```sql
CREATE DATABASE `sauna`;

CREATE TABLE `sauna`.`booking` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `booking_id` BIGINT NOT NULL,
  `event_type` ENUM('CREATE', 'UPDATE', 'DELETE') NOT NULL,
  `tenant` INT NULL,
  `sauna` INT NULL,
  `booked_from` DATE NULL,
  `booked_to` DATE NULL,
  PRIMARY KEY (`id`)
);
```

The “id” column is an increasing integer that is assigned automatically every time a new event is published to the log. The “booking_id” tells us which booking we are referring to. If two events share the same booking id, they refer to the same entity. We also have an enum called “event_type” that describes which kind of operation we were trying to do. After that comes the information that belongs to the booking. If a column is NULL, we will consider that as unmodified compared to any previous value.

#### Step 2: Generating Code using Speedment

The next step is to generate code for the project using Speedment. Simply create a new maven project and add the following code to the `pom.xml`-file.

###### pom.xml

```xml
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <maven.compiler.source>1.8</maven.compiler.source>
  <maven.compiler.target>1.8</maven.compiler.target>
  <speedment.version>3.0.0-EA2</speedment.version>
  <mysql.version>5.1.39</mysql.version>
</properties>

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

<dependencies>
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>${mysql.version}</version>
  </dependency>

  <dependency>
    <groupId>com.speedment</groupId>
    <artifactId>runtime</artifactId>
    <version>${speedment.version}</version>
    <type>pom</type>
  </dependency>
</dependencies>
```

If you build the project, a new maven goal called **speedment:tool** should appear in the IDE. Run it to launch the Speedment user interface. In there, connect to the Sauna database and generate code using the default settings. The project should now be populated with source files.

**Tip:** If you make changes to the database, you can download the new configuration using the **speedment:reload**-goal and regenerate sources using **speedment:generate**. No need to relaunch the tool!

#### Step 3: Creating the Materialized View

The materialized view is a component that regularly polls the database to see if any new rows have been added, and if so, downloads and merges them into the view in the correct order. Since the polling sometimes can take a lot of time, we want this process to run in a separate thread. We can accomplish that with a java `Timer` and `TimerTask`.

**Polling the database? Really?** Well, an important thing to take into consideration is that it is only the server that will poll the database, not the clients. This gives us very good scalability since we can have a handful of servers polling the database that in turn serve hundreds of thousands of tenants. Compare this with a regular system where every client would request a resource from the server, that in turn contacts the database.

###### BookingView.java

```java
public final class BookingView {

  ...

  public static BookingView create(BookingManager mgr) {
    final AtomicBoolean working = new AtomicBoolean(false);
    final AtomicLong last  = new AtomicLong();
    final AtomicLong total = new AtomicLong();

    final String table = mgr.getTableIdentifier().getTableName();
    final String field = Booking.ID.identifier().getColumnName();

    final Timer timer = new Timer();
    final BookingView view = new BookingView(timer);
    final TimerTask task = ...;

    timer.scheduleAtFixedRate(task, 0, UPDATE_EVERY);
    return view;
  }
}
```

The timer task is defined anonymously and that is where the polling logic will reside.

```java
final TimerTask task = new TimerTask() {
  @Override
  public void run() {
    boolean first = true;

    // Make sure no previous task is already inside this block.
    if (working.compareAndSet(false, true)) {
      try {

        // Loop until no events was merged 
        // (the database is up to date).
        while (true) {

          // Get a list of up to 25 events that has not yet 
          // been merged into the materialized object view.
          final List added = unmodifiableList(
            mgr.stream()
              .filter(Booking.ID.greaterThan(last.get()))
              .sorted(Booking.ID.comparator())
              .limit(MAX_BATCH_SIZE)
              .collect(toList())
            );

          if (added.isEmpty()) {
            if (!first) {
              System.out.format(
                "%s: View is up to date. A total of " + 
                "%d rows have been loaded.%n",
                System.identityHashCode(last),
                total.get()
              );
            }

            break;
          } else {
            final Booking lastEntity = 
              added.get(added.size() - 1);

            last.set(lastEntity.getId());
            added.forEach(view::accept);
            total.addAndGet(added.size());

            System.out.format(
              "%s: Downloaded %d row(s) from %s. " + 
              "Latest %s: %d.%n", 
              System.identityHashCode(last),
              added.size(),
              table,
              field,
              Long.parseLong("" + last.get())
            );
          }

          first = false;
        }

        // Release this resource once we exit this block.
      } finally {
        working.set(false);
      }
    }
  }
};
```

Sometimes the merging task can take more time to complete than the interval of the timer. To avoid this causing a problem, we use an `AtomicBoolean` to check and make sure that only one task can execute at the same time. This is similar to a `Semaphore`, except that we want tasks that we don’t have time for to be dropped instead of queued since we don’t really need every task to execute, a new one will come in just a second.

The constructor and basic member methods are fairly easy to implement. We store the timer passed to the class as a parameter in the constructor so that we can cancel that timer if we ever need to stop. We also store a map that keeps the current view of all the bookings in memory.

```java
private final static int MAX_BATCH_SIZE = 25;
private final static int UPDATE_EVERY   = 1_000; // Milliseconds

private final Timer timer;
private final Map<Long, Booking> bookings;

private BookingView(Timer timer) {
  this.timer    = requireNonNull(timer);
  this.bookings = new ConcurrentHashMap<>();
}

public Stream<Booking> stream() {
  return bookings.values().stream();
}

public void stop() {
  timer.cancel();
}
```

The last missing piece of the `BookingView` class is the `accept()`-method used above in the merging procedure. This is where new events are taken into consideration and merged into the view.

```java
private boolean accept(Booking ev) {
    final String type = ev.getEventType();

    // If this was a creation event
    switch (type) {
        case "CREATE" :
            // Creation events must contain all information.
            if (!ev.getSauna().isPresent()
            ||  !ev.getTenant().isPresent()
            ||  !ev.getBookedFrom().isPresent()
            ||  !ev.getBookedTo().isPresent()
            ||  !checkIfAllowed(ev)) {
                return false;
            }

            // If something is already mapped to that key, refuse the 
            // event.
            return bookings.putIfAbsent(ev.getBookingId(), ev) == null;

        case "UPDATE" :
            // Create a copy of the current state
            final Booking existing = bookings.get(ev.getBookingId());

            // If the specified key did not exist, refuse the event.
            if (existing != null) {
                final Booking proposed = new BookingImpl();
                proposed.setId(existing.getId());

                // Update non-null values
                proposed.setSauna(ev.getSauna().orElse(
                    unwrap(existing.getSauna())
                ));
                proposed.setTenant(ev.getTenant().orElse(
                    unwrap(existing.getTenant())
                ));
                proposed.setBookedFrom(ev.getBookedFrom().orElse(
                    unwrap(existing.getBookedFrom())
                ));
                proposed.setBookedTo(ev.getBookedTo().orElse(
                    unwrap(existing.getBookedTo())
                ));

                // Make sure these changes are allowed.
                if (checkIfAllowed(proposed)) {
                    bookings.put(ev.getBookingId(), proposed);
                    return true;
                }
            }

            return false;

        case "DELETE" :
            // Remove the event if it exists, else refuse the event.
            return bookings.remove(ev.getBookingId()) != null;

        default :
            System.out.format(
                "Unexpected type '%s' was refused.%n", type);
            return false;
    }
}
```

In an event sourced system, the rules are not enforced when events are received but when they are materialized. Basically anyone can insert new events into the system as long as they do it in the end of the table. It is in this method that we choose to discard events that doesn’t follow the rules setup.

#### Step 4: Example Usage

In this example, we will use the standard Speedment API to insert three new bookings into the database, two that are valid and a third that intersects one of the previous ones. We will then wait for the view to update and print out every booking made.

```java
public static void main(String... params) {
  final SaunaApplication app = new SaunaApplicationBuilder()
    .withPassword("password")
    .build();

  final BookingManager bookings = 
    app.getOrThrow(BookingManager.class);

  final SecureRandom rand = new SecureRandom();
  rand.setSeed(System.currentTimeMillis());

  // Insert three new bookings into the system.
  bookings.persist(
    new BookingImpl()
      .setBookingId(rand.nextLong())
      .setEventType("CREATE")
      .setSauna(1)
      .setTenant(1)
      .setBookedFrom(Date.valueOf(LocalDate.now().plus(3, DAYS)))
      .setBookedTo(Date.valueOf(LocalDate.now().plus(5, DAYS)))
  );

  bookings.persist(
    new BookingImpl()
      .setBookingId(rand.nextLong())
      .setEventType("CREATE")
      .setSauna(1)
      .setTenant(2)
      .setBookedFrom(Date.valueOf(LocalDate.now().plus(1, DAYS)))
      .setBookedTo(Date.valueOf(LocalDate.now().plus(2, DAYS)))
  );

  bookings.persist(
    new BookingImpl()
      .setBookingId(rand.nextLong())
      .setEventType("CREATE")
      .setSauna(1)
      .setTenant(3)
      .setBookedFrom(Date.valueOf(LocalDate.now().plus(2, DAYS)))
      .setBookedTo(Date.valueOf(LocalDate.now().plus(7, DAYS)))
  );

  final BookingView view = BookingView.create(bookings);

  // Wait until the view is up-to-date.
  try { Thread.sleep(5_000); }
  catch (final InterruptedException ex) {
    throw new RuntimeException(ex);
  }

  System.out.println("Current Bookings for Sauna 1:");
  final SimpleDateFormat dt = new SimpleDateFormat("yyyy-MM-dd");
  final Date now = Date.valueOf(LocalDate.now());
  view.stream()
    .filter(Booking.SAUNA.equal(1))
    .filter(Booking.BOOKED_TO.greaterOrEqual(now))
    .sorted(Booking.BOOKED_FROM.comparator())
    .map(b -> String.format(
      "Booked from %s to %s by Tenant %d.", 
      dt.format(b.getBookedFrom().get()),
      dt.format(b.getBookedTo().get()),
      b.getTenant().getAsInt()
    ))
    .forEachOrdered(System.out::println);

  System.out.println("No more bookings!");
  view.stop();
}
```

If we run it, we get the following output:

```
677772350: Downloaded 3 row(s) from booking. Latest id: 3.
677772350: View is up to date. A total of 3 rows have been loaded.
Current Bookings for Sauna 1:
Booked from 2016-10-11 to 2016-10-12 by Tenant 2.
Booked from 2016-10-13 to 2016-10-15 by Tenant 1.
No more bookings!
```

Full source code for this demo application is available [on my GitHub page](https://github.com/Pyknic/speedment-sauna-example). There you can also find many other examples on how to use Speedment in various scenarios to rapidly develop database applications.

## Summary

In this article we have developed a materialized view over a database table that evaluates events on materialization and not upon insertion. This makes it possible to spin up multiple instances of the application without having to worry about synchronizing them since they will be eventually consistent. We then finished by showing how the materialized view can be queried using the Speedment API to produce a list of current bookings.

Thank you for reading and please checkout [more Speedment examples](https://github.com/speedment/speedment) at the Github page!
