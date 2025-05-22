---
layout: post
title:  "Why is my code hard to test?"
date:   2024-10-20 05:49:29 +1200
category: software
tags:
  - java
  - testing
---

A frequent excuse to not write unit tests for a particular section of the code
is that the code is hard to test. It is clearly a weak excuse with a
straightforward reply: "well, in that case you should refactor the code
to make it more testable!"

However, it may not be obvious how to achieve that. In order to understand how to
make the code more testable, we first need to understand what makes the code
hard to test in the first place.

This article enumerates some of the common problems that contribute to the
difficulties encountered in testing the code. Once we are familiar with those
problems, it becomes easier to refactor the code to avoid them, consequently
making the code more testable. This is, however, a chicken-and-egg problem,
because refactoring code without the tests is a risky business. An obvious
solution is to practice Test Driven Development (TDD), but that is not a
practice loved by all programmers, and is definitely not an option when we have
to deal with a legacy codebase that has already been written without much
consideration about testability.

Even for developers and teams who do not practice TDD, or which would like to
practice but they do not find the appropriate support from the business, it is
worth knowing some of the pitfalls that make it harder to (eventually, one hopes)
add the tests.

## Static calls

...

## Object instantiation

...

## Side-effects (including void-returning methods)

...

## Multiple responsibilities

...

## Deep object graph traversal

...