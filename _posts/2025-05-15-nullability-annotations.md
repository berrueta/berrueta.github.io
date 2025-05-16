---
layout: post
title:  "Nullability annotations in Java"
date:   2025-04-05 05:49:29 +1200
category: software
tags:
  - java
  - nullability
  - types
  - annotations
---

In 1965, Tony Hoare was designing ALGOL W, a structured programming language that was a significant step forward
over low level assembly languages, and that would eventually evolve in to Pascal in the 1970s. Its influence
can be felt in many languages through a lineage of languages that includes C, C++, Java, Python and many others.
But a fateful decision was made in the design of ALGOL W that would haunt programmers for decades to come:
the invention of `null` references. That was 60 years ago. Since then, `null` references have caused
countless headaches, bugs and security vulnerabilities.

Thirty years later, when Java was being created in the mid 1990s, the designers of the language continued
the tradition of `null` references. And once they were baked into the language, it was too late to change
them. It took another 20 years for the Java language to take baby steps towards a solution in the form of
the `Optional` class, which was introduced in Java 8. However, `Optional` is not a magic bullet, and to this
day, Java programmers still have to deal with `null` references. This is a consequence of the strict backwards
compatibility of the Java language. However, `null` references are not a necessary evil, as many other languages
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
a dormant state for 20 years. Therefore, the despite their official-looking name, those annotations are not part
of the Java specifications, and therefore they lack any official standing. The only way to get them is to
include a dependency. However, to make things more complicated, FindBugs had a complicated history. For a while
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

...

# The vendor-specific annotations

...

# JSpecify

...

# The future of Java

...
