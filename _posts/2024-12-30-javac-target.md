---
layout: post
title:  "Cross-version Java compilation"
date:   2024-12-30 05:49:29 +1200
category: software
tags:
  - java
  - version
  - javac
---

In its 30 years of life, the Java language has earned a reputation for being
forward compatible. This means that code written for an older version of Java can
be generally compiled and run on a newer version of the Java runtime. This characteristic
of the language makes it very appropriate for enterprise software, where applications
tend to have long lifetimes and are often maintained for many years. It is also appreciated
by library developers, who can target a wide range of versions of Java without
having to change the code, contrary to the dramas that other languages have
suffered. This stability comes at a price, namely the slow pace of language evolution.
The Java designers have been very cautious when introducing new features to the
language to ensure that new features do not break existing code.

But what about the other way around? Can code written for a newer version of Java run
on an older version of the Java runtime? And why would you want to do that? The answer
to the second question is that developers of libraries and frameworks often want to
use the latest language features, but they also want their code to be usable by as
many developers as possible.

For many years, the answer to the first question was that it is possible to compile Java code into bytecode
that can be run with an older version of the JVM using the `-target` option of the
`javac` compiler. By default, `javac` compiles code into bytecode that is compatible with the
version of the JVM with the same version number as the compiler. For example, if we compile
a program with `javac` version 21, the default target version is also 21, and the bytecode
will be rejected by older versions of the JVM:

```bash
# When the shell is configured to use Java 21
$ javac -version
javac 21

$ javac HelloWorld.java

$ java -version
openjdk version "21.0.5" 2024-10-15 LTS

$ java HelloWorld
Hello World!

# Now we switch to Java 11
$ java -version
openjdk version "11.0.25" 2024-10-15 LTS

$ java HelloWorld
Error: LinkageError occurred while loading main class HelloWorld
  java.lang.UnsupportedClassVersionError: HelloWorld has been compiled by a more recent version
  of the Java Runtime (class file version 65.0), this version of the Java Runtime only
  recognizes class file versions up to 55.0
```

We can instruct the `javac` compiler to produce bytecode compatible with the JVM 11 by specifying the `-target` option:

```bash
# When the shell is configured to use Java 21
$ javac -target 11 HelloWorld.java
warning: target release 11 conflicts with default source release 21
```

That "warning" is actually a show-stopper. The `-target` option is not enough to make the
bytecode compatible with an older version of the JVM. We also need to downgrade the source
version to 11:

```bash
# When the shell is configured to use Java 21
$ javac -source 11 -target 11 HelloWorld.java

# Now we switch to Java 11
$ java HelloWorld
Hello World!
```

That works, because the Hello World program is very simple and does not use any new features
introduced since Java 11. Let's see now what happens when we use some modern Java.
We will use the following file:

```java
public class CoordinatesMain {
    record Coordinates(int x, int y) {
    }
    public static void main(String[] args) {
        System.out.println(new Coordinates(1, 2));
    }
}
```

This code uses `record`, a feature introduced in Java 14 (preview) and
Java 16 (final version). Therefore, the program will not compile with `javac` version 21
if we provide the `-source 11` parameter, since `record`s were not available in Java 11:

```bash
# When the shell is configured to use Java 21
$ javac -source 11 -target 11 CoordinatesMain.java
warning: [options] system modules path not set in conjunction with -source 11
CoordinatesMain.java:2: error: records are not supported in -source 11
    record Coordinates(int x, int y) {
    ^
  (use -source 16 or higher to enable records)
CoordinatesMain.java:2: warning: 'record' may become a restricted type name in a future release and may be unusable for type declarations or as the element type of an array
    record Coordinates(int x, int y) {
    ^
1 error
2 warnings
```

That makes sense. Let's see another, more subtle example:

```java
import java.util.List;

public class LastOfList {
    public static void main(String[] args) {
        System.out.println("The last one is " + List.of("a", "b", "c").getLast());
    }
}
```

It may not be obvious at a first glance, but the `List::getLast()` method is not available in Java 11.
In fact, it was not introduced until Java 21. So what does `javac` do?

```bash
# When the shell is configured to use Java 21
$ javac -source 11 -target 11 LastOfList.java
warning: [options] system modules path not set in conjunction with -source 11
1 warning
```

It does compile. However, when we try to run it with Java 11, we get a runtime error:

```bash
# Now we switch to Java 11
$ java LastOfList
Exception in thread "main" java.lang.NoSuchMethodError: 'java.lang.Object java.util.List.getLast()'
  at LastOfList.main(LastOfList.java:5)
```

This is disappointing. Nobody likes runtime errors! The problem is that code was
(successfully) compiled against the Java 21 standard library, which includes `List::getLast()`.
But at runtime, when the JVM tries to execute the code and loads the `List` class from the
Java 11 standard library, it cannot find the `getLast()` method, and throws an exception.

For years, it was a common misconception that the combination of the `-source` and
`-target` options of `javac` was enough to
make the bytecode compatible with older versions of the JVM. For example, library developers
would use recent versions of Java and compile their code with `-target 8` to make it
compatible with Java 8, and then distribute the JAR file under the assumption that it would work
with those older versions. But as we have seen, that is not the case. The `-target` option only
affects the bytecode generated by the compiler, but it does not affect the standard library
that the compiler uses to resolve symbols.

To address that, [JEP 247](https://openjdk.org/jeps/247) introduced the `--release` option to `javac`, starting with Java 9.
The `--release` option is a more robust way to compile code so the resulting bytecode can be
understood and executed by older versions of the JVM using older versions of the standard library.
It replaces `-source` and `-target` options.

Back to the first example, `--release` does not change the outcome for `CoordinatesMain.java`, since
the compiler cannot produce bytecode for the `record` feature. We get the same compile time
error as before:

```bash
# When the shell is configured to use Java 21
$ javac --release 11 CoordinatesMain.java
CoordinatesMain.java:2: error: records are not supported in -source 11
    record Coordinates(int x, int y) {
    ^
  (use -source 16 or higher to enable records)
CoordinatesMain.java:2: warning: 'record' may become a restricted type name in a future release and may be unusable for type declarations or as the element type of an array
    record Coordinates(int x, int y) {
    ^
1 error
1 warning
```

But for the other example, `LastOfList.java`, the `--release` option does make a difference.
When we compile the code with `--release 11`, the compiler uses the Java 11 standard library,
and the code fails at compile time, which is much better than failing at runtime:

```bash
# When the shell is configured to use Java 21
$ javac --release 11 LastOfList.java
LastOfList.java:5: error: cannot find symbol
        System.out.println("The last one is " + List.of("a", "b", "c").getLast());
                                                                      ^
  symbol:   method getLast()
  location: interface List<String>
1 error
```

In summary, the `--release` option of `javac` is a better way to compile code that is intended
to be run on older versions of the JVM. It ensures that the code is compatible with the standard
library of the target version, and it catches errors at compile time instead of at runtime.

For Maven users, remove the old `<maven.compiler.source>` and `<maven.compiler.target>`
properties from the `pom.xml` file and replace them with a single `<maven.compiler.release>`
property, as described in the
[Maven documentation](https://maven.apache.org/plugins/maven-compiler-plugin/examples/set-compiler-release.html).
Similarly, there is `release` property in the `build.gradle` file for Gradle projects, as described
in the [Gradle documentation](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.compile.CompileOptions.html#org.gradle.api.tasks.compile.CompileOptions:release).
