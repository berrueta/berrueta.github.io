---
layout: post
title:  "Why I do not love Mockito annotations"
date:   2024-08-10 10:49:29 +1200
category: software
tags:
  - mockito
  - java
  - testing
  - annotations
---

Mockito is an open source mocking library for Java.
I have been using Mockito for many years, and I am very grateful to the maintainers of this
library for their hard work. Although the practice of mocking is a controversial topic
in itself, I have found Mockito to be a very useful tool in my testing toolbox.

When I first started using Mockito, I was very excited about the annotations provided
with it: `@Mock`, `@Captor`, `@Spy` and `@InjectMocks`.
I thought they were an elegant and concise way to create and inject mocks into
my test classes.  However, over time, I have come to realise that the annotations
are actually not great, and nowadays I tend to avoid them if I can.

Consider the following code:

```java
public class WidgetController {
    private final WidgetService widgetService;

    public WidgetController(WidgetService widgetService) {
        this.widgetService = widgetService;
    }

    public String getWidget() {
        return widgetService.getWidget();
    }
}
```

It is a very simple controller that simply delegates the call to a service class.
Now let's write a test for this controller using Mockito annotations:

```java
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.when;

class WidgetControllerTest {
    @Mock
    WidgetService widgetService;

    @InjectMocks
    WidgetController widgetController;

    @Test
    void testGetWidget() {
        when(widgetService.getWidget()).thenReturn("widget");
        assertEquals("widget", widgetController.getWidget());
    }
}
```

The test fails:

```console
Cannot invoke "WidgetService.getWidget()" because "this.widgetService" is null
```

Ok, ok, I know what you are thinking: I'm cheating in order to make my case against the
annotation. As we know, JUnit does not support the Mockito annotations out of the box.
We need to use the `MockitoExtension` to enable them. It is well documented, but still
it is very easy to forget about that. Note that the compiler does not help us here, and
instead we get a runtime error. Anyway, the fix is very simple. Just add the
`@ExtendWith(MockitoExtension.class)`. Now the test passes! We are done here, right?

Not quite. Contrary to dependency injection in Spring or Guice, `@InjectMocks` does a
"best effort" attempt to inject the dependencies, and may fail silently. For example,
let's say the next version of the `WidgetController` class adds a new dependency on
`FeatureFlagService`:

```java
public class WidgetController {
    private final WidgetService widgetService;
    private final FeatureFlagService featureFlagService;

    public WidgetController(WidgetService widgetService,
                            FeatureFlagService featureFlagService) {
        this.widgetService = widgetService;
        this.featureFlagService = featureFlagService;
    }

    public String getWidget() {
        return widgetService.getWidget();
    }
}
```

After refactoring `WidgetController` to add the new dependency on `FeatureFlagService`,
the unit test still passes. How is that possible? There is no mock of `FeatureFlagService`,
in the test, right? The answer is that when Mockito cannot find a suitable `@Mock` to inject,
it silently injects a `null` value. In this case, not only the compiler does not help us,
but the runtime does not help us either.

To be fair, once the `WidgetController` starts making use of the `FeatureFlagService`,
the test will fail with a `NullPointerException`, assuming that the unit test has 100%
coverage. But the point is that the test should have failed immediately after the
refactoring. If the application uses Spring or Guice to inject the dependencies
of `WidgetController`, the developer will probably assume that its fields can not
be `null`. But in reality, they can be `null` if the `@InjectMocks` annotation is used.
That is one of the reasons I recommend writing code to enforce the invariant, by
rewriting the constructor to use `Objects.requireNonNull`:

```java
    public WidgetController(WidgetService widgetService,
                            FeatureFlagService featureFlagService) {
        this.widgetService = requireNonNull(widgetService);
        this.featureFlagService = requireNonNull(featureFlagService);
    }
```

That way, the test will fail immediately after the refactoring, regardless of whether
the unit test has 100% coverage or not:

```console
org.mockito.exceptions.misusing.InjectMocksException: 
Cannot instantiate @InjectMocks field named 'widgetController' of type 'class WidgetController3'.
You haven't provided the instance at field declaration so I tried to construct the instance.
However the constructor or the initialization block threw an exception : null
```

That clear failure serves a reminder to update the test and add:

```java
    @Mock
    FeatureFlagService featureFlagService;

```

Still, this is not perfect. If later on the `FeatureFlagService` stops being useful
and it is removed from `WidgetController`, the test will still pass. Other than explicit
verifications of the number of calls to the mocks, which are often neglected,
nothing will remind us to remove the mock from the test and its associated programming.

Another problem with `@InjectMocks` is that it is limited regarding what it can inject.
Let's imagine that we change again the constructor of `WidgetController` to accept a
String parameter:

```java
    public WidgetController(WidgetService widgetService, String baseUrl) {
        this.widgetService = requireNonNull(widgetService);
        this.baseUrl = requireNonNull(baseUrl);
    }
```

Once again, the compiler does not complain. Thanks to the `requireNonNull` statements, at
least the test fails with a `NullPointerException`, otherwise it would have passed despite the
test not having been updated with the new parameter. How to fix it? Mockito cannot mock
`String`, therefore we cannot add a `@Mock String baseUrl` to the test, and anyway, it
would not make sense to mock a `String`. The `@InjectMock` annotation can only inject
fields declared with `@Mock` or `@Spy`, so it cannot inject the `baseUrl` parameter using it.
We have to revert to the old way of creating the object under test:

```java
WidgetController widgetController = new WidgetController(widgetService, "http://example.com/");
```

The question is: where should we put that line? We have several options:

* In the `@BeforeEach` method: this is the most common approach, but it is verbose. It
  requires a new method to be added to the test class.
* At the beginning of the `@Test` method. However, if we have multiple
  test methods, we have to repeat the line in each one.
* In the field declaration: this is the most concise, but when combined
  with the `@Mock` annotation, it leads to errors like this:

```java
@ExtendWith(MockitoExtension.class)
class WidgetControllerTest {
    @Mock
    WidgetService widgetService;

    WidgetController widgetController = new WidgetController(widgetService, "baseUrl");

    @Test
    void testGetWidget() {
        when(widgetService.getWidget()).thenReturn("widget");
        assertEquals("widget", widgetController.getWidget());
    }
}
```

At first glace, the code looks correct, but the test fails with a `NullPointerException`.
The problem is that the `widgetService` field is not initialized when the `widgetController`
field is initialised. The Mockito extension does not initialise the fields annotated with
`@Mock` until the test object is fully created.

Note that if we had not used the `requireNonNull` statement in the constructor, the
`NullPointerException` would have been thrown by the `widgetService.getWidget()` call,
making it harder to understand the cause of the problem because it would look like a
regular test failure rather than a failure to initialise the test object.

In conclusion, I recommend avoiding the Mockito annotations and instead creating the
mocks and the object under test explicitly. This does not add much boilerplate to the
test, and it makes the test easier to understand and more robust to refactorings.
It also avoids having to abandon the annotations at a later stage, when the
object under test becomes more complex. Finally, it also makes it possible to declare
the mocks and the object under test as `final`, entirely eliminating the class of
mutability problems that can arise when using Mockito annotations.

The resulting test looks as follows, using the convenient `mock()` method with
type inference. Note that the removal of the annotations results in fewer lines of
code and fewer characters, dispelling the myth that annotations are more concise.

```java
class WidgetControllerWithoutAnnotationsTest {
    final WidgetService widgetService = mock();

    final WidgetController widgetController = new WidgetController(widgetService, "baseUrl");

    @Test
    void testGetWidget() {
        when(widgetService.getWidget()).thenReturn("widget");
        assertEquals("widget", widgetController.getWidget());
    }
}
```

Finally, let me insist in the importance of adding `requireNonNull` statements to the
constructors. This is a good practice for any constructor that accepts an object reference
and saves it to a field without using it immediately. As we have seen, it helps to
detect problems early, leading to better error messages and more robust programs.
