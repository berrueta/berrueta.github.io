---
layout: post
title:  "Think @Before you setUp it again: the frequently unnecessary method in JUnit tests"
date:   2025-04-05 05:49:29 +1200
category: software
tags:
  - java
  - testing
  - junit
---

JUnit is probably one of the most well-known Java libraries. It has been a staple
of Java development almost as long as Java itself. According to Martin Fowler, JUnit
was born in 1997, co-authored by Kent Beck and Erich Gamma during a long-distance
flight, almost 30 years ago.

JUnit has evolved over the years. The first version that I used was JUnit 3, which
predated the support for annotations in Java 5, and therefore required all test
classes to extend the `TestCase` class and all test methods to start with the prefix
`test`. There were also a couple of methods that could be overridden in the test
class, namely `setUp` and `tearDown`, which were called before and after each test
method.

JUnit 4 introduced annotations, which removed the need to extend the `TestCase`,
as well as the naming conventions. This made it possible to write test methods
with arbitrary names. Also, the `@Before` and `@After` annotations were introduced,
which allowed the `setUp` and `tearDown` methods to be named arbitrarily, although
in my experience many developers still use the `setUp` and `tearDown` names due
to inertia or convention. Similarly, the annotations `@BeforeClass` and `@AfterClass`
were introduced to allow the execution of code before and after all test methods.

JUnit 5 was released in 2017, although it took a while for it to be widely adopted
because it was a major rewrite with some important changes to the API and the
engine. One of the changes introduced in JUnit 5 was that the `@Before`, `@After`, `@BeforeClass`
and `@AfterClass` annotations were replaced by `@BeforeEach`, `@AfterEach`, `@BeforeAll`
and `@AfterAll`, respectively. Other than the name change, the semantics of these
annotations remained the same.

I have been writing and reviewing unit tests for many years, and I have noticed
that many developers feel a compulsion to write methods with the `@Before` (JUnit 4)
or `@BeforeEach` (JUnit 5) annotation, when in many cases they are unnecessary.
In the next sections I will explain how oftentimes these methods can be avoided,
often resulting in simpler, more concise and less error-prone test code.

## Initialising fields

In some cases, the `@Before` methods (I'll use the JUnit 4 name for brevity) is
used to initialise fields, like this:

```java
public class MyTest {
    private MyClass myClass;

    @Before
    public void setUp() {
        myClass = new MyClass();
    }

    @Test
    public void testMethod1() {
        // ... some test code about MyClass
    }
}
```

This is a common practice in unit tests, particularly to initialise the class
under test or a mock/stub object. 

However, this is unnecessarily verbose. The `myClass` field can be initialised directly at
the point of declaration, with the added benefit that the field can be declared
`final`, which is a good practice to avoid accidental reassignments. The
test can be written as:

```java
public class MyTest {
    private final MyClass myClass = new MyClass();

    @Test
    public void testMethod1() {
        // ... some test code about MyClass
    }
}
```

At this point, some developers raise their eyebrows and point out that initialising the
field at the point of declaration changes the semantics because now all test methods
share the same instance of `myClass`. This is a very common misconception which
I also had at some point. The reality is that the JUnit framework creates a new
instance of the test class for each test method, so each test method gets its own
instance of `myClass`. Perplexed, the same developers keep their eyebrows up and
ask if that is a new feature of JUnit 5. The answer is no, this has been the case
forever. It is one of the tenets of the design of JUnit, a very deliberate design
choice to isolate test methods dating back 30 years and
[explained by Martin Fowler](https://martinfowler.com/bliki/JunitNewInstance.html).

## Initialising mocks

Another common use of the `@Before` method is to initialise mocks, like this:

```java
public class MyTest {
    private MyClass myClass;
    private MyDependency myDependency;

    @Before
    public void setUp() {
        myDependency = mock(MyDependency.class);
        when(myDependency.getName()).thenReturn("foo");
        
        myClass = new MyClass(myDependency);
    }

    @Test
    public void testMethod1() {
        // ... some test code about MyClass
    }
}
```

Let's first move the object instantiation to the field declaration, which
I prefer over the use of the Mockito annotations (`@Mock`, `@InjectMocks`, etc.),
which [I do not love]({% post_url 2024-08-10-mockito-annotations %}):

```java
public class MyTest {
    private final MyDependency myDependency = mock(MyDependency.class);
    private final MyClass myClass = new MyClass(myDependency);

    @Before
    public void setUp() {
        when(myDependency.getName()).thenReturn("foo");
    }
    
    @Test
    public void testMethod1() {
        // ... some test code about MyClass
    }
}
```

We still have the `@Before` method to set up the mock's behaviour, but we can
take this one step further. There is no reason the mock programming has to happen
in the `@Before` method. It can be done in the test method itself:

```java
public class MyTest {
    private final MyDependency myDependency = mock(MyDependency.class);
    private final MyClass myClass = new MyClass(myDependency);

    @Test
    public void testMethod1() {
        // program the mock
        when(myDependency.getName()).thenReturn("foo");
        
        // ... some test code about MyClass
    }
}
```

That is uncomplicated and easy to read. But what if there are many test methods,
or if programming the mocks is complex and requires multiple lines of code which
we do not want to repeat? Easy enough: we can extract a regular method with a
descriptive name:

```java
public class MyTest {
    private final MyDependency myDependency = mock(MyDependency.class);
    private final MyClass myClass = new MyClass(myDependency);

    @Test
    public void testMethod1() {
        givenDependencyNameIs("foo");
        
        // ... some test code about MyClass
    }

    @Test
    public void testMethod2() {
        givenDependencyNameIs("bar");
        
        // ... some test code about MyClass
    }

    private void programMock(String name) {
        when(myDependency.getName()).thenReturn(name);
    }
}
```

I argue that explicitly calling a regular method leads to more readable code.
The method name is part of the test code, making it easier to follow
BDD (Behaviour Driven Development) principles. There is no behind-the-scenes
magic happening in the `@Before` method, potentially many lines of code away.
In fact, the IDE can help us to navigate to the method definition, resulting
in a WYSIWYG test method.

Of course extracting a method is just one of the many options we have to
program the mock. We can also use a builder-like pattern to define the
test data fixture, and get creative inventing our own fluent API with a DSL
specifically designed for the test class.

## Acquiring and releasing resources

In integration tests, the `@Before` and `@After` methods are also used to acquire
and release resources, like network or database connections, WireMock servers,
temporary files or directories, etc. This is a use case that often justifies the
use of `@Before` and `@After`, but there may be a better option: to use rules
(in JUnit 4) or extensions (in JUnit 5). There are a number of advantages to
that. For example, the resource acquisition and release code is separated from
the test class, which helps with keeping the test class as succinct as possible.
Also, the resource acquisition and release code can be reused across multiple
test classes.

A general rule of thumb, if you find yourself writing a `@Before` method without
a companion `@After` method, the lack of symmetry is a good indication that
the `@Before` method is unnecessary, because a typical pattern for expensive
resources is that they need to be released after the test method is executed.
Releasing resources in unit tests is often overlooked because, contrary to
server code, it is assumed that the test code runs for a short period of time
and therefore it can afford to be wasteful with resources, since the execution
will finish soon anyway.

However, that assumption is wrong at scale.
In fact, releasing scarce resources after the test has run may be essential.
Let's imagine a large Java project with a thousand test methods (the number
of test classes is irrelevant because, as explained above, JUnit creates a new
instance of the test class for each test method). If each test method allocates
an array of 1MB and saves it as a field, then to run that whole test suite would require 1GB.
Because the test runner (Maven Surefire or similar) retains every single instance
of the test classes in order to produce a final report at the end of the test run,
that memory cannot be garbage collected, and can lead to an `OutOfMemoryError` and
even before that, to sluggish performance due to excessive garbage collection.
The solution is easy: in the case of object references, they can be released in
the `@After` method, for example by setting the reference to `null`,
so object can be released from the heap. For example:

```java
public class MyTest {
    // ...
    private int[] largeArray;

    @Before
    public void acquireResources() {
        largeArray = new int[/*large*/];
    }

    @After
    public void releaseResources() {
        largeArray = null; // allow the array to be garbage-collected
    }
}
```

In this case, since we need to have the `@After` method to release the resource and the
field cannot be `final`, we can keep the `@Before` method as well for symmetry and to make it
obvious that the resource is acquired and released. Note that the test methods are descriptive
of what they do, rather than following the old and obsolete JUnit 3 convention of
`setUp` and `tearDown`.

## Conclusion

Although writing `@Before` and `@After` methods is a common practice and
by no means it is a bug, it is often unnecessary. There are some cases where those
methods can fit well, like when acquiring and releasing resources, but in the most
common cases they can be substituted.
