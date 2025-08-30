---
layout: post
title:  "Testing and time in Java"
date:   2025-06-15 05:49:29 +1200
category: software
tags:
  - java
  - time
  - instant
  - clock
  - precision
---

The other day I was reviewing a couple of pull request in a Java project, and I noticed that
both contained a problem with the way time was being handled in the tests. Normally I would not
have thought much of it, but the fact that these two problems appeared almost simultaneously in two pull requests submitted
by two different developers, and that each problem was essentially the opposite of the other, prompted me
to write this post.

The common theme of the two pull requests was that they both contained unit tests that used the `Instant`
class, and that they made assumptions about the speed at which the code runs compared to the internal
clock of the system, resulting in flaky tests.

## Problem 1: Assuming that the code runs fast enough

In the first pull request, the developer had written a complex test, but in order to illustrate the
problem, let's consider a massively simplified version of the test:

```java
public class TimeTest {
    @Test
    void testTimeMovesForward() {
        Instant present = Instant.now();
        Instant past = Instant.now().minusSeconds(60);

        assertThat(past).isBefore(present);
    }
}
```

Normally this test would pass, because the first call to `Instant.now()` returns the current time,
and the second call returns a time that is 60 seconds in the past, so the assertion that `past` is before `present`
holds true. However, this test is flaky because it assumes that the code runs fast enough so that
the second call to `Instant.now()` is executed no more than 60 seconds after the first call.
That is a reasonable assumption in most cases, but it is not guaranteed to be true. For example, if we
set up a breakpoint in the line that starts with `Instant past =`, and we wait for 60 seconds before
continuing the execution, the test will fail because the second call to `Instant.now()` will return a time
that is actually in the future compared with `present`. Another way to trigger the failure is to
add a `Thread.sleep(61_000)` before the second call to `Instant.now()`:

```bash
$ ./gradlew test
> Task :test FAILED

TimeTest > testTimeMovesForward() FAILED
    java.lang.AssertionError at TimeTest.java:16
```

Breakpoints and `Thread.sleep()` may seem like an extreme way to trigger the failure, but in practice there
are many other ways in which the code can sometimes run slower than expected, like garbage collection,
high CPU load, CPU trashing, or simply a slow computer. This kind of flakiness can be very hard to track down.

There are many ways to avoid this problem. One may think that the simplest solution is to
use a wider gap between `present` and `past`, like 10 minutes instead of 60 seconds, but that is just
a band-aid. Another solution is to avoid time subtraction altogether, and instead only use
time addition, like this:

```java
Instant present = Instant.now();
Instant future = Instant.now().plusSeconds(60);
```

This may look like a reasonable solution, but it is making another assumption, namely that the code runs
in a forward direction, which may not hold in every programming language or environment. For example, some
languages and compilers may optimise the code in such a way that it seems to run backwards or in parallel.
That is a complex topic that is beyond the scope of this post, and fortunately we do not have to worry about
it in this case, because there is a much simpler solution that works in general: avoid sampling the system clock
more than once! For example, we can rewrite the test like this:

```java
public class TimeTest {
    @Test
    void testTimeMovesForward() {
        Instant present = Instant.now(); // sampling the system clock once
        Instant past = present.minusSeconds(60);

        assertThat(past).isBefore(present);
    }
}
```

That eliminates any assumption about the speed at which the code runs,
because we are only sampling the system clock once. But can we do better than that? Yes, we can!
There is no reason `present` has to be the current time. We can use any time we want, completely
removing the sampling of the system clock and using only constants:

```java
public class TimeTest {
    @Test
    void testTimeMovesForward() {
        Instant present = Instant.EPOCH; // using a constant time
        Instant past = present.minusSeconds(60);

        assertThat(past).isBefore(present);
    }
}
```

Almost any constant will do here. In the example above, I used `Instant.EPOCH`, but we could have used
an arbitrary time like `Instant.parse("2000-01-01T00:00:00Z")`. Why is using a constant better than
sampling the system clock even once? Because it makes the test deterministic, meaning that it will always
pass or fail in the same way, regardless of when it is executed. For example, with the previous version of the test
that sampled the system clock once, the test would pass if executed in our lifetime, but it would fail due to
overflow if executed
near the `Instant.MIN`, which is the lower bound of the `Instant` class range. In case you are wondering when
that was, it was a billion years ago, well before humans invented computers. But if some single-celled
organism living in the ocean of the primitive Earth or the supercontinent Rodinia were to run the test, it would fail for them.
That may seem like an extreme case, but a more complex and realistic test may have a much narrower range of
validity.

For those reasons, every time I see a call to `Instant.now()` in a test, I immediately start suspecting that
the test may be flaky, and I start thinking about how to eliminate that call and use a constant time instead.

## Problem 2: Assuming that the code runs slowly enough

The second pull request was representative of the opposite problem to the one described above. In order
to understand this second problem, let's assume we have a class of `Widget`s, and a `WidgetRepository` that stores them. For the sake
of simplicity, the widgets are stored in a `Map` in memory:

```java
public class Widget {
    private final Long id;
    private Instant creationTime;
    private Instant lastModifiedTime;

    public Widget(Long id) {
        this.id = id;
    }

    // Getters and setters omitted for brevity
}
```

```java
public class WidgetRepository {
    private final Map<Long, Widget> widgets = new ConcurrentHashMap<>();

    public void addWidget(Widget widget) {
        Instant now = Instant.now();
        widget.setCreationTime(now);
        widget.setLastModifiedTime(now);
        widgets.put(widget.getId(), widget);
    }

    public void updateWidget(Widget widget) {
        widget.setLastModifiedTime(Instant.now());
        widgets.put(widget.getId(), widget);
    }

    public Widget getWidgetById(Long id) {
        return widgets.get(id);
    }
}
```

And here a test that checks that the `lastModifiedTime` is updated when a widget is updated:

```java
class WidgetRepositoryTest {
    private static final long ID = 1L;

    @RepeatedTest(3)
    void shouldUpdateWidget() {
        WidgetRepository repository = new WidgetRepository();

        repository.addWidget(new Widget(ID));
        Widget createdWidget = repository.getWidgetById(ID);
        assertThat(createdWidget).isNotNull();
        assertThat(createdWidget.getCreationTime()).isNotNull();
        assertThat(createdWidget.getLastModifiedTime()).isEqualTo(createdWidget.getCreationTime());

        // pretend the widget is modified, and now we update it

        repository.updateWidget(createdWidget);
        Widget updatedWidget = repository.getWidgetById(ID);
        assertThat(updatedWidget.getLastModifiedTime()).isAfter(createdWidget.getCreationTime());
    }
}
```

This seems straightforward. Note the last assertion crucially uses a strict comparison (`isAfter`) in order to
confirm that the `updateWidget` method has, indeed, updated the `lastModifiedTime`, so the two times are
no longer equal.

However, the test is flaky because it assumes that the computer is slow enough so that the system clock
has ticked at least once between the two calls to `Instant.now()`, namely the one in `addWidget` and the one
in `updateWidget`. This test will fail if the computer is fast enough that the two calls to `Instant.now()`
return the same value.

Java's `Instant` class has a precision of nanoseconds, which means that it can represent time with a resolution of
one nanosecond. As of 2025, even modest computers can execute billions of instructions per second, which means that
it is entirely possible for two calls to `Instant.now()` to return the same value if they are executed in quick
succession. And future computers will likely be even faster, making this problem more pronounced. Which means that
this test may pass reliably in a slow computer, but it will eventually fail when computers become fast enough.
It's a ticking time bomb (pun intended).

To make things worse, even if `Instant` representws time with nanosecond precision,
the actual precision of the system clock is likely to be lower than that. How low? It depends on the hardware,
the operating system and the JVM version. For example, versions of Windows prior to Windows 8 used to have a
precision of 15 milliseconds, which is ample time for a fast computer to execute many instructions.

Similarly, older versions of the JVM had a precision of milliseconds, which means the test may pass
on a new version of the JVM, but fail on an older version. This is exactly what happens in my computer.
When I run the test using Java 21, it seems to pass every time (although it cannot be guaranteed):

```bash
$ jenv local 21
$ ./gradlew test
...
BUILD SUCCESSFUL in 1s
```

but if I run it using Java 8, it seems to fail every time:

```bash
$ jenv local 1.8
$ ./gradlew test

> Task :test FAILED

WidgetRepositoryTest > shouldUpdateWidget() > repetition 2 of 3 FAILED
    java.lang.AssertionError at WidgetRepositoryTest.java:24

WidgetRepositoryTest > shouldUpdateWidget() > repetition 3 of 3 FAILED
    java.lang.AssertionError at WidgetRepositoryTest.java:24

3 tests completed, 2 failed
```

Note the use of `@RepeatedTest(3)` to run the test three times, which is a good practice to
catch flaky tests. Looks like the first repetition passes, but the second and third ones fail,
because they run "too fast" for the system clock to tick between the two calls to `Instant.now()`.

Fixing this test to make it not flakey but at the same time verify that `updateWidget` does indeed
update the `lastModifiedTime` is not trivial. One bad way to fix it would be to add a `Thread.sleep(...)`
before the call to `updateWidget`, to ensure the system clock has ticked at least once, but for
a number of reasons that is not a good idea. Another idea would be to add a retry loop using
a library like Awaitility to wait for the `lastModifiedTime` to be different from the
`creationTime`, but again that comes with its own set of problems.

The proper fix is to decouple the code from the real time, replacing the static calls to `Instant.now()` with a
`Clock` that can be controlled by the test. This way, we can simulate the passage of time without
making assumptions about the speed of the code or the precision of the system clock.
The details of how to do that are enough for a separate post.

## Conclusion

Writing tests that use time can be tricky, because they can easily become flaky if we make assumptions
about the speed at which the code runs or the precision of the system clock. The main takeaways from this post are:

* Avoid sampling the system clock more than once in a test, and ideally use constants instead of `Instant.now()`.

* Do not assume that the clock will tick between two calls to `Instant.now()`
  because it may not do so if the code runs fast enough compared to the precision of the clock.

* Avoid static calls to `Instant.now()` in production code, because it makes the code harder to test.
  Instead, use an injectable `Clock` that can be controlled by the tests.
  