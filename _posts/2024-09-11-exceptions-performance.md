---
layout: post
title:  "Avoid exceptions for control flow - the performance argument"
date:   2024-09-11 05:49:29 +1200
category: software
tags:
  - java
  - exceptions
  - performance
---

It is widely accepted mantra that exceptions should not be used for control flow. The main
arguments against that practice have to do with code structuring, pointing out that exceptions
are not very different from "go-to" statements, which are considered harmful, and that even in
languages that support exceptions there are often better ways to handle errors. Exceptions
can make the code harder to read, harder to maintain/evolve, harder to test and harder to debug.

When should exceptions be used then? The general rule of thumb is that exceptions should be used
for truly exceptional situations, that is, conditions that are not part of the normal flow of
the program or that are difficult to predict, like `StackOverflowError` or `OutOfMemoryError`.
On the opposite side of the spectrum, exceptions must not be used for conditions that are part
of the normal flow of the program, like a user entering an invalid input.

One of the arguments against using exceptions for control flow is their poor performance
in languages like Java. But how bad is it really? Let's take a look at the following class:

```java
public class DeliveryService {
    public String deliver_withExceptions(String postalCodeOrTown) {
        try {
            return deliverToPostalCode(Integer.parseInt(postalCodeOrTown));
        } catch (NumberFormatException e) {
            return deliverToTown(postalCodeOrTown);
        }
    }

    private String deliverToPostalCode(int postalCode) {
        return "Delivering to postal code " + postalCode;
    }

    private String deliverToTown(String town) {
        return "Delivering to town " + town;
    }
}
```

For users' convenience, this `DeliveryService` class allows them to provide either a postal code
or a town name as input. It will try to parse the input as an integer (postal code) and fallback to
delivering to the town name if it fails. It does that by wrapping the call to `Integer.parseInt`
in a `try/catch` block, and recovering from the `NumberFormatException`.

Let's now write a very un-scientific performance test to get a sense of how
this code performs:

```java
 public static void main(String[] args) {
     var deliveryService = new DeliveryService();

     var start = System.nanoTime();

     for (long i = 0; i < 10_000_000; i++) {
         deliveryService.deliver_withExceptions("Springfield");
     }

     var finish = System.nanoTime();

     System.out.println("Time elapsed: " + (finish - start)/1_000_000 + " ms");
 }
```

This test will call the method 10 million times, and measure the time it takes to do so.
On my laptop, takes around 4.4 seconds. That does not sound too bad, does it?

We can refactor the code to avoid exceptions. One option is to use a very simple regular
expression to check if the input is a number before trying to parse it. Another option is
to use a library like Apache Commons Lang, which provides a `NumberUtils.isNumeric` method.
The code looks like this:

```java
    public String deliver_withRegex(String postalCodeOrTown) {
        if (postalCodeOrTown.matches("\\d+")) {
            return deliverToPostalCode(Integer.parseInt(postalCodeOrTown));
        } else {
            return deliverToTown(postalCodeOrTown);
        }
    }

    public String deliver_withIsNumeric(String postalCodeOrTown) {
        if (isNumeric(postalCodeOrTown)) {
            return deliverToPostalCode(Integer.parseInt(postalCodeOrTown));
        } else {
            return deliverToTown(postalCodeOrTown);
        }
    }
```

Let's run the same test with these two new methods, again with 10 million iterations.
The results are:

| Method            | Time elapsed (ms) |
|-------------------|-------------------|
| withExceptions    | 4,400             |
| withRegex         | 620               |
| withIsNumeric     | 30                |

As mentioned before, this is a very un-scientific test. Nevertheless, the performance difference
is staggering. The version with regular expressions is 7 times faster, despite regular
expressions being known for being slow. The version with `NumberUtils.isNumeric` is
146 times faster.

There are a number of reasons why the version with exceptions is so slow. One of them is that
exceptions are expensive to create because by default they include a stacktrace, which
basically requires an inspection of the contents of the stack. Another reason is that
exceptions disrupt the "predictable" flow of the program, which impedes some of the optimisations
that the JVM and the microprocessors can do.

Some clever readers may have noticed that the "improved" versions without exceptions
are not perfect. For example, both the implementation with a regular expression and
the `NumberUtils.isNumeric` one will fail if the input is `"12345678901"` because although
it matches the regular expression and is numeric, that number is too large to be parsed
as an integer, which is what we need in order to call `deliverToPostalCode`.
We may actually need to handle that case with a `try/catch`, after all. But
that case falls into the category of exceptional situations, where an exception may be the
right tool for the job. It can be argued that `"12345678901"` is neither a valid postal code
nor a valid town name, and therefore it is not a valid input. Therefore
in that case we would not be using exceptions for control flow as part of
processing valid inputs, but for actual error handling.
