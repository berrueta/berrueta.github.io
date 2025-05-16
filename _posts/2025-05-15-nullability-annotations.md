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

# The defunct JSR 305

...

# The JSR 303 annotations

...

# The vendor-specific annotations

...

# JSpecify

...

# The future of Java

...
