---
layout: post
title: "Implementing catamorphisms in Java"
date: 2014-04-29 01:20
categories: [programming, java, maths]
---

*Warning: I'm now going to wake up the ghosts of the past. The past which includes your undergraduate abstract algebra lectures.*

This post is mostly a result of my boredom and my willingness to show that you can shoehorn almost every abstraction into almost every programming language, but it's not exactly the best idea to do so.

Also, I simply wanted to say a word or two about several abstract mathematical concepts. It can be a worthwhile intellectual exercise.

## Algebras

Before I talk about catamorphisms, mentioned in the title, I'd like to have a look at a very abstract and general mathematical structure: an algebra.

An **algebra** is a tuple that contains:

* some sets (called carrier sets), most often one

* usually some operations, each of them from some Cartesian product of the carrier sets to one of the carrier sets

* sometimes some distinguished elements from those sets (they're often superfluous, but they will be required later; you can also think about them as of nullary operations)

Examples: 

* a set of strings `Str` and the operation of concatenation `+: Str × Str → Str`

* a set of natural numbers `Nat` and the operation of addition: `+: Nat × Nat → Nat`

* a set of finite subsets of natural numbers `Nat` and a set of natural numbers `P(Nat)`, a distinguished empty set `Ø` and the operations of union and intersection `P(Nat) × P(Nat) → P(Nat)` and the largest element operation `max: P(Nat) → Nat` (with `max(Ø) = 0`)

*Note: I'm using the `+` operator for string concatenation because the article is supposed to end up with creating some Java code, and Java uses `+` for string concatenation.*

<!-- more -->

Usually when talking about an algebra, one specifies properties those operations have, for example concatenation is associative, addition is commutative and associative, and so on.

When you look at it, an algebra can be considered equivalent to a program in a statically typed functional language. 

## Homomorphisms

The second important thing we need are algebra homomorphisms. Let's assume we have two algebras `A₁` and `A₂` with the following properties:

* They have the same number of carrier sets, e.g. `X₁, Y₁, Z₁` and `X₂, Y₂, Z₂`.

* They have the same number of distinguished elements from the corresponding carriers sets, e.g. `ε₁ ∈ X₁, α₁ ∈ Y₁, β₁ ∈ Y₁` and `ε₂ ∈ X₂, α₂ ∈ Y₂, β₂ ∈ Y₂`.

* They have the same number of operations from the correspoding carriers sets to the corresponding carrier sets, e.g. `f₁: X₁×Y₁→Y₁, g₁: Y₁×Y₁→Y₁, h₁: X₁×X₁→X₁, k₁: Y₁→Z₁` and `f₂: X₂×Y₂→Y₂, g₂: Y₂×Y₂→Y₂, h₂: X₂×X₂→X₂, k₂: Y₂→Z₂`.

I'll be calling such pairs of algebras **algebras with matching signatures**.

Now if we have a function `H` from carrier sets of one algebra to carriers sets of another algebra, and that function preserves the algebraic structure, e.g. in the example from above:

* `H: (X₁ ∪ Y₁ ∪ Z₁) → (X₂ ∪ Y₂ ∪ Z₂)`

* `H(ε₁) = ε₂`, `H(α₁) = α₂`, `H(β₁) = β₂`

* `H(f₁(x,y)) = f₂(H(x), H(y))`, `H(g₁(y,y')) = g₂(H(y), H(y'))`, `H(h₁(x,x')) = h₂(H(x), H(x'))`, `H(k₁(y)) = k₂(H(y))`

then we call `H` a **homomorphism**.

You probably need an example. I'll start with a simple one: the two first algebras I mentioned have a homomorphism between them: the length. Indeed, `length(str₁ + str2) = length(str₁) + length(str₂)`. 

There's also a homomorphism back, let's call it `aaaa`, which for a given natural number returns a string made of that number of a's, e.g. `aaaa(3) = "aaa"`. You can easy see that `aaaa(n₁ + n₂) = aaaa(n₁) + aaaa(n₂)`.

*Note: You have probably noticed, that instead of performing calculations on some operands in one algebra and finally converting the result to another algebra using a homomorphism, we can first convert the result to the second algebra and then perform the calculations. Indeed that is correct and usually it's useful, especially if the operations in the second algebra are easier to perform. Haskell library [HLearn](https://github.com/mikeizbicki/HLearn) converts an algebra of data samples to an algebra of statistics, allowing for instantaneous recalculating of statistics after adding some data to an enormous sample.*

If the homomorphism is invertible, i.e. there is a homomorphism `K` from the second algebra back to the first, and `K(H(x)) = x` and `H(K(y)) = y`, then we call `H` an **isomorphism** and the two algebras **isomorphic**. Two isomorphic algebras can be considered equivalent for most purposes.

Since `length` and `aaaa` don't have that property (`aaaa(length("b")) = "a" ≠ "b"`), nor any other pair of homomorphisms does, the algebra of strings with concatenation and the algebra of natural numbers with addition are not isomorphic.

## Initial algebras

A special category of algebras are **initial algebras**. Their carrier sets are not given, but are defined from the given distinguished elements and operations. For example, the carrier set `X` for an initial algebra with distinguished element `Z ∈ X` and unary operation `S: X → X` is `{Z, S(Z), S(S(Z)), S(S(S(Z))), S(S(S(S(Z))))...}`. Operations in initial algebras are not commutative nor associative, but they are always injective. They're sometimes called **constructors**, and like the constructors in programming languages, they always construct new, distinct values. When initial algebras are implemented, they're usually called **algebraic data structures**.

Let's implement the above initial algebra in Java:

```java
public interface X {
  
}
```

```java
public class Z implements X {
    public boolean equals(Object o){
        return o instanceof Z;
    }
}
```

```java
public class S implements X {
  
  public final X field1;
  
  public S(X param1){
      this.field1 = param1;
  }
  
  public boolean equals(Object o){
      return o instanceof S && ((S)o).field1.equals(field1);
  }
}
```

*Note: Implementing `hashCode`, providing null-safety and adding getters left as an exercise for the reader.*

##Catamorphisms

The nice thing about the initial algebras is that for every algebra with a matching signature, there exists exactly one homomorphism from initial algebra to the given algebra. That unique homomorphism is called the **catamorphism**.

For example, given the algebra above, we can uniquely map it onto an algebra with natural numbers, number `0`, and function `x → x + 1`: `H(Z) → 0`, `H(S(x)) → H(x) + 1`. While it may look tautological, keep in mind that the elements of the carrier set of an initial algebra are uniquely defined by the distinguished elements and operations that were used to create them.

Let's implement it in Java!

```java
public int h(X x){
    if (x instanceof Z) {
        return 0;
    } else if (x instanceof S) {
        return h(((S)x).field1) + 1;
    else {
        throw new IllegalArgumentException();
    }
}
```

It works, but it's ugly.

## Pattern matching... sort of

Let's take another look at our homomorphism in Java. The code has several issues:

* it explicitly checks type and casts the parameter;

* it calls itself recursively;

* it has this ugly unreachable code that will suddenly start throwing exceptions when we add another constructor to our `X` algebra.

Let's address #1 and #3 first.

If we were using a functional language with first-class support for algebraic data types, we could write our homomorphism using pattern matching:

```fsharp
let h x = 
  match x with
  | Z -> 0
  | S field1 -> h field1 + 1
```

Java does not support pattern matching, that's obvious and furthermore, the alternative solution using explicit casts is ugly and prone to bugs. To solve it in Java, the officially preferred way is to use the visitor pattern.

```java
interface XVisitor<T> {
      T visit(Z z);
      T visit(S s);
}

public interface X {
      <T> T visit(XVisitor<T> visitor);
}

public class Z implements X {
    public boolean equals(Object o){
        return o instanceof Z;
    }
    public <T> T visit(XVisitor<T> visitor){
        return visitor.visit(this);
    }
}

public class S implements X {
  
    public final X field1;
    
    public S(X param1){
        this.field1 = param1;
    }
    
    public boolean equals(Object o){
        return o instanceof S && ((S)o).field1.equals(field1);
    }

    public <T> T visit(XVisitor<T> visitor){
        return visitor.visit(this);
    }
}

class H implements XVisitor<Integer> {

    public int visitZ(Z z){
        return 0;
    }

    public int visit(S s){
        return s.field1.visit(this) + 1;
    }
}
```

*Note: As you can see, it's not the typical Visitor pattern as described by the Gang of Four. First, here the `visit` methods actually return something instead of returning `void`, second, GoF suggests that the branching classes, like `S` here, always had the visitor visit their children and express that in the `S.visit` method.*

Since the only thing we need from the instances of our classes is values of their fields, we can simplify it further:


```java
interface XVisitor<T> {
    T visitZ();
    T visitS(X field1);
}

public interface X {
    <T> T visit(XVisitor<T> visitor);
}

public class Z implements X {
    public boolean equals(Object o){
        return o instanceof Z;
    }
    public <T> T visit(XVisitor<T> visitor){
        return visitor.visitZ();
    }
}

public class S implements X {
  
    public final X field1;
    
    public S(X param1){
        this.field1 = param1;
    }
    
    public boolean equals(Object o){
        return o instanceof S && ((S)o).field1.equals(field1);
    }

    public <T> T visit(XVisitor<T> visitor){
        return visitor.visitS(field1);
    }
}

class H implements XVisitor<Integer> {

    public int visitZ(){
        return 0;
    }

    public int visitS(X field1){
        return field1.visit(this) + 1;
    }
}
```

Looks way better. Too enterprisy for my taste, but hey, it's Java.

The upside of this new approach is that whenever we add a new class implementing our base interface, the compiler will kick and scream until we fix our visitors.

The downside is that we still have to call `visit` in `visitS`. The burden of converting `field1` to `T` should be on the `S` class. The visitor interface should be decoupled from the `X` interface. Let's try a bit harder.

## Catamorphisms and visitors, take two

Let's have a look at our simple algebra so far:

```java
interface X:
    Z()
    S(X param1)
```

As you can see, it's recursive. That means it's also infinite.

Let's get rid of the recursion somehow. What happens if we replace X with some type parameter T?

```java
interface XPrecursor<T>:
    ZPrecursor<T>()
    SPrecursor<T>(T param1)
```

(Sorry for weird syntax.)

As you can see, our algebra is no longer recursive. (It can still be infinite though, for example if some constructors use another infinite types.) Such family of non-recursive algebras is called the **precursor** of our initial algebra `X`. All algebras in this family are also initial.

*Note: `XPrecursor<X>` is isomorphic to `X`. Here are the homomorphisms both ways:*

```java
X fromPrecursor(XPrecursor<X> p){
    if (p instanceof ZPrecursor) {
        return new Z();
    } else if (p instanceof SPrecursor {
        return new S(((SPrecursor<X>)p).param1);
    } else {
        throw new IllegalArgumentException();
    }
}

XPrecursor toPrecursor(X x){
    if(x instanceof Z){
        return new ZPrecursor();
    } else if (x instanceof X) {
        return new SPrecursor<X>(((S)x).param1);
    } else {
        throw new IllegalArgumentException();
    }
}
```

How would a visitor for `XPrecursor<T>` look like?

```java
interface XPrecursorVisitor<T,U> {
    U visitZ();
    U visitS(T field1);
}
```

*Note: Since `XPrecursor<T>` is not recursive, the visitor does not have to know anything about `XPrecursor<T>`.*

*Note: Remember: every `XPrecursorVisitor<T,U>` defines a homomorphism `XPrecursor<T> → U`.*

And finally: can we convert an `XPrecursorVisitor<T,T>` into an `XVisitor<T>`? The answer is yes:

```java
class XCata<T> implements XVisitor<T> {
    private final XPrecursorVisitor<T,T> underlying;
    
    public XCata(XPrecursorVisitor<T,T> u){
        underlying = u;
    }
    
    public T visitZ(){
        return underlying.visitZ();
    }
    
    public T visitS(X x){
        return underlying.visitS(x.visit(this));
    }
}
```

Now, let's sum all that we know:

* `X` is an initial algebra

* for every `T`, `XPrecursor<T>` is an initial algebra

* for every homomorphism `f: XPrecursor<T> → T` (which means `XPrecursorVisitor<T,T>`) exist at least one homomorphism `g: X → T` (which means `XVisitor<T>`), and only one that doesn't incorporate any external constants into it

As some of you may have noticed, I called the converter `XCata`. Indeed, if we think of the `XPrecursorVisitor<T,T>` as of an algebra over `T` (where the methods represent the distinguished elements if zero parameters and the operations if not), `XCata` is the catamorphism from initial algebra `X` to the `T` algebra.

Armed with this knowledge, we can add `<T> T visit(XPrecursorVisitor<T> visitor)` to our X interface and inline the methods of `XCata` in the implementations of the new method:

```java
public interface X {
    <T> T visit(XPrecursorVisitor<T> visitor);
}
public class Z implements X {
    public boolean equals(Object o){
        return o instanceof Z;
    }
    public <T> T visit(XPrecursorVisitor<T> visitor){
        return visitor.visitZ();
    }
}

public class S implements X {
  
    public final X field1;
    
    public S(X param1){
        this.field1 = param1;
    }
    
    public boolean equals(Object o){
        return o instanceof S && ((S)o).field1.equals(field1);
    }

    public <T> T visit(XPrecursorVisitor<T> visitor){
        return visitor.visitS(field1.visit(visitor));
    }
}
```

## Some more initial algebra examples

Before we continue, we will need some more examples. Here are some unfinished Java snippets:

```java
// a binary tree with leaves containing values of type T
interface Tree<T> { 
...

class Leaf<T> implements Tree<T> {
    Leaf(T leaf){ 
...

class Branch<T> implements Tree<T> {
    Branch(Tree<T> tree, Tree<T> right) {
...

// a simple integer arithmetic expression tree with both variables and constants
interface Expression {
...

class Variable extends Expression {
    Variable(String name) {
...

class Constant extends Expression {
    Constant(int value){
...

class Add extends Expression {
    Add(Expression expr1, Expression expr2) {
...

class Multiply extends Expression {
    Multiply(Expression expr1, Expression expr2) {
...

class Negate extends Expression {
    Negate(Expression expr) {
...
```

In fact, trees and expressions are the most common examples discussed in articles on similar topics.

## Where the heck to use all that theoretical mumbo-jumbo?


[Wikipedia says](http://en.wikipedia.org/wiki/Catamorphism) that functional programmers refer to catamorphisms as *folds*. Every time you have some recursive datatype, usually tree-like or list-like, *folding* takes such large value and folds it into some nice, small, single value. It does that by looking only at the nearest neighbourhood: the already folded child nodes and the type and other fields of the current node.

From now on, I'll call things I called PrecursorVisitors before, Folders.

Given the tree defined above, how would a folder for a `Tree<T>` look like?

```java
interface TreeFolder<T,U> {

    U visitLeaf(T leaf);

    U visitBranch(U left, U right);
}
```

And it's used like this:

```java
interface Tree<T> {
    <U> fold(TreeFolder<T,U> folder);
}

class Leaf<T> implements Tree<T> {

    final T leaf;

    Leaf(T leaf) {
        this.leaf = leaf;
    } 

    public <U> U fold(TreeFolder<T,U> folder) {
        return folder.visitLeaf(leaf);
    }
}

class Branch<T> implements Tree<T> {

    final Tree<T> left;
    final Tree<T> right;

    Branch(Tree<T> left, Tree<T> right) {
        this.left = left;
        this.right = right;
    } 

    public <U> U fold(TreeFolder<T,U> folder) {
        return folder.visitBranch(left.fold(folder), right.fold(folder));
    }
}
```

What can you use it for? For example, to get the leftmost element of the tree:

```java
class Leftmost<T> extends TreeFolder<T,U> {

    public T visitLeaf(T leaf){
        return leaf;
    }

    public T visitBranch(T left, T right){
        return left;
    }
}
```

Or count leaves:

```java
class LeftMost<T> extends TreeFolder<T,Integer> {

    public Integer visitLeaf(T leaf){
        return 1;
    }

    public Integer visitBranch(Integer left, Integer right){
        return left + right;
    }
}
```

As for the expressions, evaluating them is one way of folding them:

```java
interface ExprFolder<T> {
    T visitConst(int value);
    T visitVariable(String name);
    T visitAdd(T expr1, T expr2);
    T visitMul(T expr1, T expr2);
    T visitNegate(T expr);
}
```

```java
class Evaluate implements ExprFolder<Integer> {
    private final Map<String, Integer> vars;
  
    Evaluate(Map<String, Integer> vars){
        this.vars = vars;
    }

    public Integer visitConst(int value){
        return value;
    }

    public Integer visitVariable(String name){
        return vars.get(name);
    }

    public Integer visitAdd(Integer expr1, Integer expr2){
        return expr1 + expr2;
    }

    public Integer visitMul(Integer expr1, Integer expr2){
        return expr1 * expr2;
    }

    public Integer visitNegate(Integer expr){
       return -expr1;
    }
}
```

or convert to a string:

```java
class Stringify implements ExprFolder<String> {

    public String visitConst(int value){
        return "" + value;
    }

    public String visitVariable(String name){
        return name;
    }

    public String visitAdd(String expr1, String expr2){
        return "(" + expr1 + " + " + expr2 + ")";
    }

    public String visitMul(String expr1, String expr2){
        return expr1 + "×" + expr2;
    }

    public String visitNegate(String expr){
        return "(-" + expr1 + ")";
    }
}
```

And so on.

## So, is everything a fold?

In short, not quite.

Some operations, like for example expression optimisation, are not a fold, unless you pick a weird result type. Folds always progress from leaves to the root and never descend back. Optimisation of expression trees, on the other hand, is usually implemented as a descent from the root to the leaves.

In theory though, everything can be implemented as a fold. Please don't do that though.

Furthermore, folding shown in this post is strict, which means it always descends to the end of a tree before returning back. If the data structure it was traversing was infinite, it would never yield an result.

## Final remarks

First of all, I'd like to thank Bartosz Milewski for a nice write-up on the same topic: 
https://www.fpcomplete.com/user/bartosz/understanding-algebras1111

Second of all, Java is not the best suited programming language for working with algebraic data types. Lack of pattern matching, verbose class declaration syntax, defaulting to equality by reference, defaulting to mutable fields, requiring explicit copying from constructor parameters to fields, and few other things make this pattern bothersome. It works better in other languages though, see for example Milewski's Haskell examples.
