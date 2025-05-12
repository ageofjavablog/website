---
layout      : post
title       : DTO design with Jackson
description : Use composition to reuse code in Spring DTOs
headline    : AGE OF JAVA
category    : java
modified    : 2018-08-30
tags        : [Java, Spring, Spring Boot, DTO, Jackson, Lombok]
featured    : true
---
When designing a REST API, it's often necessary to vary the level of detail returned depending on the endpoint. For example, a "list all books" endpoint might only return a subset of each book’s fields, while a "get book by ID" endpoint could return the full details.

Consider a Spring REST controller like this:

```java
@RestController
@RequestMapping("/books")
@RequiredArgsConstructor
public class BookController {

    ...

    @GetMapping
    public PagedModel<BookBase> findAll(Pageable pageable) {
        ...
    }

    @GetMapping("/{id}")
    public ResponseEntity<BookDetailed> findById(@PathVariable int id) {
        ...
    }
}
```

A common approach is to let `BookDetailed` extend `BookBase`. While this may seem convenient at first, it introduces problems:

- **Poor API documentation:** Tools like SpringDoc/OpenAPI may struggle to generate accurate schemas.
- **Code duplication:** Converting from your domain model to multiple DTOs becomes repetitive and error-prone.

For instance:

```java
@Service
@RequiredArgsConstructor
public class BookService {

    private final BookRepository bookRepository;

    @Transactional(readOnly = true)
    public PagedModel<BookBase> getMany(Pageable pageable) {
        return new PagedModel<>(
            bookRepository.findAll(pageable)
                .map(this::toBookBase)
        );
    }

    @Transactional(readOnly = true)
    public Optional<BookDetailed> getOne(int id) {
        return bookRepository.findById(id)
            .map(this::toBookDetailed);
    }

    private BookBase toBookBase(Book book) {
        return BookBase.builder()
            .id(book.getId())
            .title(book.getTitle())
            .build();
    }

    private BookDetailed toBookDetailed(Book book) {
        return BookDetailed.builder()
            .id(book.getId())       // Code repetition
            .title(book.getTitle()) // Code repetition
            .description(book.getDescription())
            .build();
    }
}
```

There's no easy way to reuse the `BookBase` logic when building `BookDetailed`.

## A Better Alternative: Composition with `@JsonUnwrapped`
Instead of inheritance, you can use composition to structure your DTOs more cleanly. With Jackson’s `@JsonUnwrapped`, you can embed a `BookBase` instance directly into `BookDetailed` and have its fields serialized as if they were declared directly.

```java
@Data
@Builder
@FieldDefaults(level = AccessLevel.PRIVATE)
public class BookDetailed {
    @JsonUnwrapped BookBase book;
    String description;
}
```

Now you can build the DTO like this:

```java
private BookDetailed toBookDetailed(Book book) {
    return BookDetailed.builder()
        .book(toBookBase(book)) // Reuse previous logic
        .description(book.getDescription())
        .build();
}
```

This way, you keep the DTOs decoupled, avoid boilerplate, and preserve flexibility for future changes. It's also much easier to test and reuse your projection logic across services and layers.
