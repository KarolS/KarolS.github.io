---
layout: post
title: "Random ideas for Scala 3.0"
date: 2014-04-29 01:22
comments: true
categories: [scala, programming]
---

Here comes a list of things that I would love to see in Scala 3.0. Some of them are breaking changes, hence 3.0 not 2.13 or anything like that. Some of them are about the compiler, some of them are about the library, some of them are about the external tools. Some of those ideas are different solutions for the same problem.

<!-- more -->

### Syntax

* Treating number literals with leading zeroes as decimal (with a warning for a version or two).

* Binary and octal literals in forms of `0b01010101` and `0o037`.

* Underscores in numeric literals: `1_000_000`.

* Some kind of byte array literal.

* `BigInt` and `BigDecimal` literals: `100000000000000000000000000000000000N`, `0.01m`

* `Short` and `Byte` literals, both accepting both signed and unsigned values: `20000s`, `255y`

* Removal of XML literals and an `xml` macro string context as a replacement: `xml"<p>Hello $world</p>"`

* Introducing true `break`, `continue` and `goto`.

* A special syntax for monads, like Haskell's `do`.

    * `do` is already a keyword, it can cause problems with `do...while` loops.

    * `for` is clunky, especially when it comes to `if...else`. Maybe fix `for`?

### Standard library

* Clean-up of the collections. Currently, the standard collections are a little mess. I think [Paul Phillips sums it up nicely](http://www.slideshare.net/extempore/a-scala-corrections-library).

* Standard `scala-time` library, being a wrapper for both Joda Time and Java 8 Time API.

    * There would be two functionally identical implementations.

* HLists.

* Vectors with length known at compile-time.

* A macro that includes C header files on compile time and generates corresponding JNA interfaces.

* Removal of `/:` and `:\` methods.

* A clone of .NET's `dynamic` type.

* `String.toInt` and friends accepting a base.

* A `^` operator for sets, returning the symmetric difference.

### Value classes

* Specialization for custom value classes.

* Generating specialized classes on the fly by the compiler (so the compiler would take an existing class, either ours or from an external library, and specialize it).

    * This would allow to specialize collections.

* Unboxed arrays for custom value classes.

* Different name mangling for methods taking value classes.

    * Currently, if you have `class A(x: Int) extends AnyVal`, `class B(x: Int) extends AnyVal` and try to write both `def method(a: A)` and `def method(b: B)`, you get an error.

* Multi-field value classes.

    * This could work as following: if the value is a field, a local variable, or a parameter, use several variables. If it's a return value, box it.

    * This of course wouldn't improve performance much, unless there would be an alternative way to return such values.

### Type system

* Almost full type inference for private and local functions and variables.

* Built-in type-level integers, booleans and strings, with some operations on them.

* Compile-time null safety -- detection of provable paths that could potentially lead to NPE's.

* Multimethods.

### Fixing minor annoyances

* Adding Scalaz's `some` and `none`.

* Adding Scalaz's `Equal` and `Monoid` typeclasses.

* `@adt` annotation on a trait would make the generated `apply` method of companion objects of case classes that implement that trait return that trait.

    * For example, annotating `Option` with `@adt` and writing `var x = Some(1)` would make `x` of type `Option[Int]`, not `Some[Int]`.

    * It would be nice to make it somehow work with case objects, especially in polymorphic cases, like with `Option` and `None`, or `Either` and either `Left` or `Right` (either of these fixes only one type parameter, not two).

* Ignoring missing annotation definitions in external libraries.

* Allowing for creating annotations with runtime retention in Scala.

### Tooling

* A decent, officially supported Findbugs plugin.

* A configurable style checker, preferably including ["powerlevels"](http://www.scala-lang.org/old/node/8610).

* A Java-to-idiomatic-Scala converter.

* A Scala-to-idiomatic-Java converter.

* Easier developing for Android.

    * Built-in Proguard-like optimizer?

    * Dalvik backend?