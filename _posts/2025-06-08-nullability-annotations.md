---
layout: post
title:  "Nullability annotations in Java"
date:   2025-06-08 05:49:29 +1200
category: software
tags:
  - java
  - nullability
  - types
  - annotations
---

In 1965, Tony Hoare was designing ALGOL W, a structured programming language that was a significant step forward
over low level assembly languages, and that would eventually evolve in to Pascal in the 1970s. Its influence
can be felt in many languages through a lineage that includes C, C++, Java, Python and many others.
But a fateful decision was made in the design of ALGOL W that would haunt programmers for decades to come:
the invention of `null` references. That was 60 years ago. Since then, `null` references have caused
countless headaches, bugs and security vulnerabilities.

Thirty years later, when Java was being created in the mid 1990s, the designers of the language continued
the tradition of `null` references. And once they were baked into the language, it was too late to change
them. It took another 20 years for the Java language to take baby steps towards a solution in the form of
the `Optional` class, which was introduced in Java 8. However, `Optional` is not a magic bullet, and to this
day, Java programmers still have to deal with `null` references. This is a consequence of the strict backwards
compatibility of the Java language. However, `null` references are not a necessary evil and can be
managed, as many other languages
have shown. Modern mainstream languages like Rust, Swift and Kotlin have shown that is it possible to
design a language that still feels like C/C++ or Java, but has a safe approach to nullabiliy. Of those, Kotlin
is particularly notable because it is a JVM language that is interoperable with Java, showing a remarkable
balance between safety and interoperability.

Back to Java, there have been attempts to introduce some safety around `null` references in the form of
static analysis on top of the base language. A crucial step in that direction was the introduction of annotations,
which happened in Java 5 in 2004, opening the door to adding metadata to the types. The next step should have
been the introduction of nullability annotations, but that's when things started to get messy.

# FindBugs and the never finished JSR 305

In 2006 a PhD student at the University of Maryland and his PhD advisor released a static analysis tool called
FindBugs, which gained popularity in the Java community. One of its features was the ability to detect
potential null pointer dereferences. This required the use of annotations to mark the nullability of
parameters and return values, and so the `@Nullable` and `@NonNull` annotations were born. Initially they
resided in the [edu.umd.cs.findbugs.annotations](https://findbugs.sourceforge.net/api/edu/umd/cs/findbugs/annotations/package-summary.html) package.

It was clear that the Java language needed a standard way to express nullability, so in 2006 a Java Specification Request
([JSR 305](https://jcp.org/en/jsr/detail?id=305)) was created through the Java Community Process to standardize the
nullability annotations. Even before the JSR was completed, the FindBugs annotations were moved to the proposed
`javax.annotation` package, and that is the origin of `javax.annotation.Nullable` and `javax.annotation.Nonnull`.

However, the JSR was never completed and thus the annotations were never standardized. The JSR has been in
a dormant state for 20 years. Therefore, the despite their official-looking package name, those annotations are not part
of the Java specifications, and therefore they lack any official standing. The only way to get them is to
include a dependency. Furthermore, FindBugs had a complicated history. For a while
it was hosted in SourceForge, and its binary artifacts were published under the Maven `packageId` of
`net.sourceforge.findbugs` (note that Maven `packageId` did not match the Java package name). Later the source code
moved to Google Code (both SourceForge and Google Code were predecessors to GitHub), and the binary artifacts were
published under the Maven `packageId` of `com.google.code.findbugs`. In particular, the JSR 305 annotations implemented
by FindBugs can still be downloaded by adding the following dependency to your Maven project:

```xml
<dependency>
    <groupId>com.google.code.findbugs</groupId>
    <artifactId>jsr305</artifactId>
    <version>...</version>
</dependency>
```

Note that despite the `google` packageId, the annotations are not directly affiliated with Google. It just happens that
the project was hosted in Google Code for a while.

The story does not end there. FindBugs was declared abandoned 10 years later, and
a fork called SpotBugs was created and is still maintained in GitHub to this day. Recognising that JSR 305 is defunct,
the SpotBugs maintainers decided to stop publishing annotations under the
`javax.annotation` Java package which they do not own. Instead, they recommend using old FindBugs artifact described
above, or the original FindBugs annotations under `edu.umd.cs.findbugs.annotations`. This is an interesting plot
twist, basically a 180 turn, because those annotations were declared "deprecated" by FindBugs when they were (perhaps prematurely)
moved to `javax.annotation` 20 years ago, but now they have been reinstated by SpotBugs.

There is a common myth that the `javax.annotation.Nullable` and sibling `NotNull` annotations have a special
standing in Java, when in fact they do not. They are just annotations like any other that you and me can create.
They do not exist unless they are included via a dependency. And as explained above, they are basically abandonware.

# The JSR 303 annotations

In parallel to the JSR 305 annotations, another set of annotations was created in 2009, and unlike the JSR 305,
the JSR 303 annotations were successfully completed and standardized. The purpose of the JSR 303 annotations is to
provide a way to express validation constraints on Java beans, such as `@Positive`, `@Min` (for numeric
types), `@Size`, `@NotBlank` (for strings), and crucially for this article, `@NotNull`. Note that `@NotNull` is
not the same as JSR 305's `@Nonnull`.

In order to use the JSR 303 annotations, they must be included as a dependency. As other JavaEE dependencies, they
used to be published under the `javax.validation` package, but then they were moved to the `jakarta.validation`:

```xml
<dependency>
    <groupId>jakarta.validation</groupId>
    <artifactId>jakarta.validation-api</artifactId>
    <version>3.1.1</version> <!-- latest version as of 2025 -->
</dependency>
```

By itself, that dependency only provides the annotations, but it does not change the semantics of the Java language.
In order to actually use the annotations, a validation engine must be included in the classpath. The most
popular implementation is Hibernate Validator, which despite its name is not limited to Hibernate, and can be
used with other Java beans, like Spring MVC models. The validation happens at runtime, and only after the bean
has been created and populated with data. This means that the validation is not part of the type system, and
therefore it does not provide compile-time guarantees.

A crucial difference between the JSR 303 annotations and the JSR 305 annotations is the retention policy of
the annotations. The JSR 303 annotations have a retention policy of `RUNTIME`, which means that they are available
at runtime for reflection. This is in contrast to the JSR 305 annotations, which have a retention policy of
`CLASS`, which means that they are saved in the `.class` bytecode file but are not available at runtime,
and therefore can only be used by static analysis tools.

# The vendor-specific annotations

Due to the lack of a standard, over the years various vendors have created their own nullability annotations. To name a few,
there is `software.amazon.awssdk.annotations.NonNull` from the AWS SDK, `lombok.NonNull` from the Lombok library,
`org.eclipse.jdt.annotation.NonNull` from the Eclipse JDT and `org.springframework.lang.NonNull` from the Spring
Framework. But perhaps the most notable of the vendor-specific annotations are the ones from Jetbrains,
namely `org.jetbrains.annotations.NotNull` and `org.jetbrains.annotations.Nullable`. The dependency for those is:

```xml
<dependency>
    <groupId>org.jetbrains</groupId>
    <artifactId>annotations</artifactId>
    <version>26.0.2</version>
</dependency>
```

This dependency can be added to any Java project and has no transitive dependencies. The annotations have a retention
of `RUNTIME`. This means that they can be used by static analysis tools, but also at runtime for reflection. This is
crucial, because it allows to annotate Java code in a way that simplifies the interoperability with Kotlin. For example,
a Java class annotated with `@NotNull` can be used in Kotlin without having to use the `!!` operator because Kotlin
will automatically treat the Java type as non-nullable.

# JSpecify

After more than a decade of confusion, Google rallied many of the main players in the Java ecosystem
to converge in a standard set of nullability annotations. It took an additional 6 years of deliberation, but in 2024 the
release 1.0.0 of the JSpecify annotations was announced. The JSpecify annotations are a minimalistic set of nullability
annotations that have the support of Google (Android, Guava), Jetbrains (IntelliJ, Kotlin), Oracle (Java), Meta,
Microsoft, and Broadcom (Spring), among others. It also has backing of static analysis tools like SonarQube, NullAway,
ErrorProne and The Checker Framework. The JSpecify annotations are carefully specified to avoid some of the pitfalls of
JSR 305. Large projects like Spring Framework have already started to adopt the JSpecify annotations, and if everything
goes well, they will become the de facto standard for nullability annotations in Java, replacing the failed JSR 305 and
the many vendor-specific annotations. As of 2025, JSpecify are already the obvious choice for new projects.

# The future of Java

The JSpecify annotations are a step in the right direction, but they are not a complete solution. They do not
change the semantics of the Java language, and therefore they do not provide compile-time guarantees. Compared to
Kotlin null-safety, they are a bit unsatisfactory.

However, the Java language continues to evolve. When it was created in the mid 1990s, their designers made the
controversial decision to include `null` as a valid value of every object reference... but not primitive types like
`int`, `boolean` and `double`. It is too late to change that now, but there is hope that in the future Java will
introduce a new type system that is more in line with modern languages like Kotlin, Swift and Rust. There is a
[JEP proposal](https://openjdk.org/jeps/8316779) that is a stepping stone in the direction of Project Valhalla and
value types. The idea is to introduce type markers to indicate that some (value) types are null-restricted. If this
comes to fruition, it will not apply to any object reference or to existing code, but it will allow new code to
be written in a more null-safe way. However, at this point this is just a proposal, and it is not clear
when or if it will be implemented.
