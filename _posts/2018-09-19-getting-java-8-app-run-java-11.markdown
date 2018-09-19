---
layout: post
title: "Getting your Java 8 application to run on Java 11 – in 9 easy steps"
date: 2018-09-19 00:40
categories: [programming, Java]
---

Recently, I had an opportunity to fix two applications to get them running on Java 10 while preserving compatibility with Java 8.
Migrating to the module system was not on the table, the goal was just to run everything on a newer OpenJDK.
It was not a trivial task: neither of programs worked on Java 10 out of the box.
Here's how I managed to fix the applications so the same binary code could run on Java 8, 10 and the release candidate version of Java 11.

<!-- more -->

## Step 1 – try building/running your program with Java 10/11

Huge chances that it will fail to work. Huge chances it even fail to _compile_. Don't worry, we'll fix it later.
Start by trying to run your automated tests on new Java (you do have them, right?) and see what's failing.
Keep analysing the errors after every fix you make.

## Step 2 – remove unused or unimportant imports

Remove all unused imports, this includes imports in the `.gradle` files.
There's nothing more annoying than failing to compile a file that imports a nonexistent class which it doesn't even use.

Also, if you're just importing a random class from `com.sun.*` or `javax.*` to use just one static method,
consider replacing it with an alternative.
In our case, they were `javax.xml.bind.DatatypeConverter#printHexBinary` and `javax.xml.bind.DatatypeConverter#parseHexBinary`,
which was used purely for debugging, so they were replaced with own implementations.

## Step 3 – update your build tools to run against the new Java

We were using Gradle 3.x, which doesn't work on new Java at all.
To update it, you need to change the `distribution-url` property in your `gradle-wrapper.properties`,
remove `gradle-wrapper.jar` and run `./gradlew --version` to make it fetch the new version.

_Optional: You may also want to remove old Gradle caches in your home directory
(most likely at `~/.cache/gradle`) to reclaim some disk space._

After your tools are updated, run the `clean` command of your build tool using the new Java.
Keep running it from time to time whenever you suspect something weird is going on with the compiler.

## Step 4 – add missing dependencies

Java 11 removes several packages from the standard Java distribution. In short, everything that starts with `javax.` is suspect.
Here is what dependencies I had to add: 

* `javax.annotation:javax.annotation-api`

* `javax.validation:validation-api`

* `javax.xml.bind:jaxb-api`

* `com.sun.xml.bind:jaxb-core`

* `com.sun.xml.bind:jaxb-impl`

* `javax.xml.ws:jaxws-api`

* `com.sun.xml.ws:jaxws-rt`

* `com.sun.xml.ws:rt`

* `javax.activation:activation`

You might need a totally different set. For example, JavaFX is no longer a part of Java SE, so you might need to add that.

Don't worry, adding those dependencies doesn't cause trouble on Java 8.

## Step 5 – check for name clashes between new Java classes and your classes

In our case, it was `java.lang.Module` vs `com.fasterxml.jackson.databind.Module`.
The simplest solution was to use the fully qualified version of the latter.

## Step 6 – fix all the other random compilation errors that might pop up

For some unexplained reason, a piece of code using a method handle as a parameter to `Optional#map` stopped wanting to compile.
I just replaced it with a lambda:

```java
     // worked on Java 8, but for some reason didn't on Java 11
     optional.flatMap(b -> foo(x, b)).map(Bar::getBarValue)

     // works on both Java 8 and 11
     optional.flatMap(b -> foo(x, b)).map(b -> b.getBarValue())
```

After this step, your program should build, but it might probably still work incorrectly.

## Step 7 – update your dependencies

Anything that uses code generation, annotation processing, native code, reflection, and so on is the most likely suspect.

In our case, the main problem cause was Eclipselink.
We were using the 2.6.x branch, but it does not support Java above 9.
The solution was simply to replace `org.eclipse.persistence:eclipselink:2.6.5`
with `org.eclipse.persistence:org.eclipse.persistence.jpa:2.7.3` and fix all the issues that popped up because of that.

## Step 8 – run all automatic tests on both Java 8 and Java 11

I hope I didn't surprise you with this one. Make it green for both.

## Step 9 – deploy to one of your test environments and test it there

After doing all the fixes, I ran the application first locally, with some instances on Java 8 and some on 11,
and when I verified it's working, I pushed it to our test cluster which was a mix of Java 8 and Java 10. Everything worked fine.

**Done.** At least, that's what it took me to do it, you might encounter some extra issues I luckily managed to dodge.

If you remember how easy was it to migrate from Java 7 to 8, this process felt much worse.
I can really hope that most of it is just growing pains related to the module system and the new versioning scheme introduced in Java 9 and that it won't be as problematic in the future.
But that's Oracle we're talking about, who knows.

