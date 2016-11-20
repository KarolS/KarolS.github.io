---
layout: post
title: "Java 8 Functional Interfaces Cheatsheet"
date: 2015-08-08 22:37
categories: [java, programming]
---

Java 8 introduced basic support for first-class functions. The functions, unlike in other languages, aren't represented by only handful of types. Instead, Java 8 uses dozens of various types depending on arity, parameter types and the return type. The standard documentation is pretty unwieldy, so for my and your convenience, I prepared a list of all functional interfaces in a more useful order.

Few things to have in mind:

* All interfaces are in `java.util.function` package unless otherwise noted.

* Interfaces are specialized only for `boolean`, `int`, `long` and `double`, and only sometimes. Other primitive types, and all primitive types in certain situations, will have to be boxed.

### Nullary functions

function type | Java type
--- | ---
() → **void** | `java.lang.Runnable`
() → **boolean** | `BooleanSupplier`
() → **int** | `IntSupplier`
() → **long** | `LongSupplier`
() → **double** | `DoubleSupplier`
() → *A* | `Supplier<A>` or `java.lang.Callable<A>`

<!-- more -->


Whether to use `Supplier` or `Callable`, it's mostly a matter of taste and semantics. `Supplier` has a method called `get`, `Callable` has `call`. This suggests that functions with larger side-effects should be `Callable`, and functions with small side-effects (mostly lazily-initialized values) should be `Supplier`s. Of course, this is only a suggestion.

---

### Unary functions

function type | Java type
--- | ---
**int** → **void** | `IntConsumer`
**long** → **void** | `LongConsumer`
**double** → **void** | `DoubleConsumer`
*A* → **void** | `Consumer<A>`
**int** → **boolean** | `IntPredicate`
**long** → **boolean** | `LongPredicate`
**double** → **boolean** | `DoublePredicate`
*A* → **boolean** | `Predicate<A>`
**int** → **int** | `IntUnaryOperator`
**long** → **int** | `LongToIntFunction`
**double** → **int** | `DoubleToIntFunction`
*A* → **int** | `ToIntFunction<A>`
**int** → **long** | `IntToLongFunction`
**long** → **long** | `LongUnaryOperator`
**double** → **long** | `DoubleToLongFunction`
*A* → **long** | `ToLongFunction<A>`
**int** → **double** | `IntToDoubleFunction`
**long** → **double** | `LongToDoubleFunction`
**double** → **double** | `DoubleUnaryOperator`
*A* → **double** | `ToDoubleFunction<A>`
*A* → *A* | `UnaryOperator<A>`
*A* → *B* | `Function<A, B>`


Note the following:

* The primitive type of the argument is attached directly, the primitive type of the result uses the prefix `To`.
* If the result is `boolean`, then the function is called a predicate. It' it's `void`, then a consumer.
* If the parameter type and the result type are the same, the function is called a unary operator.
* There are no specialized interfaces from a primitive to a reference.

---

### Binary functions

function type | Java type
--- | ---
(*A*, **int**) → **void** | `ObjIntConsumer<A>`
(*A*, **long**) → **void** | `ObjLongConsumer<A>`
(*A*, **double**) → **void** | `ObjDoubleConsumer<A>`
(*A*, *B*) → **void** | `BiConsumer<A, B>`
(*A*, *B*) → **boolean** | `BiPredicate<A, B>`
(**int**, **int**) → **int** | `IntBinaryOperator`
(**long**, **long**) → **long** | `LongBinaryOperator`
(**double**, **double**) → **double** | `DoubleBinaryOperator`
(*A*, *A*) → *A* | `BinaryOperator<A>`
(*A*, *B*) → **int** | `ToIntFunction<A, B>`
(*A*, *B*) → **long** | `ToLongBiFunction<A, B>`
(*A*, *B*) → **double** | `ToDoubleBiFunction<A, B>`
(*A*, *B*) → *C* | `BiFunction<A, B, C>`

---

### Ternary functions and above

None.

