---
layout: post
title: "Compiling Brainfuck with optimisations"
date: 2013-05-21
description:
categories: [brainfuck, scala, "code-generation", programming]
published: false
---

Brainfuck is the epitome of Turing tarpit, one of the simplest programming languages ever conceived. I assume everyone here has heard about it; if not, I strongly recommend to check it out.

Brainfuck can, as most programming languages, be either interpreted or compiled. While interpretation is of no interest for me (writing a Brainfuck interpreter is a freshman's exercise), compiling poses several interesting challenges.

I'll assume from here that what we're doing is taking Brainfuck source and generating equivalent C source. Some may refer to such process as “transpiling”, but since you can always compile the resulting source yourself, it makes it synonymous with compiling. Many other languages have historically been transpiled to C, for example C++ and Haskell.

The simplest Brainfuck compiler can be written with sed and 10 or so regular expressions. The result is not that good. The resulting code is slow, even when compiled with heavily optimising C compiler.

<!-- more -->

For testing, I'm going to use [Lost Kingdom Brainfuck port by Jon Ripley](http://jonripley.com/i-fiction/games/LostKingdomBF.html) and [a Mandelbrot set generator](http://esoteric.sange.fi/brainfuck/bf-source/prog/mandelbrot.b).

Let's start with a simple test: how long does it take for the default Brainfuck interpreter that's available in many Linux distros repositories to run the Mandelbrot program?

    $ time bf mandelbrot.bf > /dev/null
    
    real    0m7.818s
    user    0m7.788s
    sys     0m0.012s

7.8 seconds.

Pretty slow, huh. Let's see how can we compile Brainfuck program. Assume `char * p` is the main pointer used in the program:

* `+` can be compiled to `*p++;`

* `-` can be compiled to `*p--;`

* `<` can be compiled to `p--;`

* `>` can be compiled to `p++;`

* `[···]` can be compiled to `while(*p){···}`

* `,` and `.` can be compiled to correct I/O operations

Note that:

* Multiple `<` can be compiled to `p-=N;`

* Multiple `>` can be compiled to `p+=N;`

* Multiple `-` can be compiled to `*p-=N;`

* Multiple `+` can be compiled to `*p+=N;`

* `[-]` can be compiled to `*p=0;`

Converting code with such rules to C and compiling it with `gcc -O1 --std=c99` yiels the result of 1.243 seconds.

Let's generalise it a bit. As you may have noticed, we abstracted common Brainfuck patterns into more idiomatic C statements. If we could isolate more patterns, even more abstract ones, we could optimise the code even more.

Let us assume that for now we'll be satisfied if our code got transformed into an abstract syntax tree built from the following elements:

* Increase(i, x) `p[i] += x;`

* Shift(i) `p += i;`

* Write(i) `fputc(p[i], stdout);`

* Read(i) `p[i] = fgetc(stdin);`

* While(i, List(···)) `while(p[i]) {···}`

* Assign(i, x) `p[i] = x;`

* Move(f, t, a) `p[t] += a*p[f];`

We can model it in Scala quite easily:

```scala
sealed trait BfExpr
case class Increase(i: Int, x: Byte) extends BfExpr
case class Shift(i: Int) extends BfExpr
case class Write(i: Int) extends BfExpr
case class Read(i: Int) extends BfExpr
case class While(i :Int, l: List[BfExpr]) extends BfExpr
case class Assign(i: Int, x: Byte) extends BfExpr
case class Move(f: Int, t: Int, a: Byte) extends BfExpr
```

And we can parse Brainfuck source also easily:

```scala
import scala.util.parsing.combinator._

object BfParser extends RegexParsers {

  override val whiteSpace = """[^\.,<>\+\-\[\]]+""".r
      
  def op: Parser[BfExpr] = (
    "+" ^^^ Increase(0,1) |
    "-" ^^^ Increase(0,-1) |
    ">" ^^^ Shift(1) |
    "<" ^^^ Shift(-1) |
    "." ^^^ Write(0) |
    "," ^^^ Read(0) |
    "[" ~> rep(op) <~ "]" ^^ (xs => While(0, xs))
    )
  
  def parseBf(program: String) = parseAll(rep(op), program).get
}
```

And then we can emit C code from the parsed code:

``` scala
import java.io._

object CEmitter {

  def emit(l: List[BfExpr], os: PrintStream){
    os.println("""#include <stdio.h>
#include <stdint.h>
static uint8_t m[30000];
int main(){
register uint8_t *p = m;""")
    l foreach { emit2(_, os, 1) }
    os.println("return 0;\n}");
  }

  def emit2(e: BfExpr, os: PrintStream, indent: Int){
    val tabs = " " * (4*indent)
    os.print(tabs)
    e match {
      case Shift(1) => os.println("++p;")
      case Shift(-1) => os.println("--p;")
      case Shift(x) if x<0 => os.println(s"p -= ${-x};")
      case Shift(x) => os.println(s"p += $x;")
      case Increase(i,x) => os.println("p[$i] += $x;")
      case Assign(i, x) => os.println(s"p[$i] = $x;")
      case Move(f, t, a) => os.println(s"p[$t] += $a * p[$f];")
      case Read(i) => os.println(s"p[$i] = fgetc(stdin);")
      case Write(i) => os.println(s"fputc(p[$i], stdout);")
      case While(i, es) => 
        os.println(s"while(p[$i]){")
        es foreach { emit2(_, os, indent+1) }
        os.println(tabs+"}")
    }
  }
}
```

Of course the above code does not do any optimisations yet. For example, the following Brainfuck code:


    +++[-]<>,.

is compiled into:

``` c
#include <stdio.h>
#include <stdint.h>
static uint8_t m[30000];
int main(){
  register uint8_t *p = m;
  p[0] += 1;
  p[0] += 1;
  p[0] += 1;
  while(p[0]){
    p[0] += -1;
  }
  --p;
  ++p;
  p[0] = fgetc(stdin);
  fputc(p[0], stdout);
  return 0;
}
```

As you can see, there are several places where this can be optimised:

* triple `p[0] += 1;` could be replaced with `p[0] += 3;`

* the loop should be replaced with assignment of 0

* `--p; ++p;` should be optimised away.

Let's design a simple optimising algorithm:

* analyse the code as a list of instructions

* if the beginning of the list can be optimised, optimise it

* if not, toss one instruction to the list of optimised instructions and try to optimise the shorter list

For example (assume the vertical bar is the cursor), if we could only apply rules from the old compiler:

``` c
| p[0]+=1; p[0]+=1; p[0]+=1; while(p[0]){ p[0]+=-1; } --p; ++p; p[0]=fgetc(stdin); fputc(p[0],stdout);
| p[0]+=2; p[0]+=1; while(p[0]){ p[0]+=-1; } --p; ++p; p[0]=fgetc(stdin); fputc(p[0],stdout);
| p[0]+=3; while(p[0]){ p[0]+=-1; } --p; ++p; p[0]=fgetc(stdin); fputc(p[0],stdout);
p[0]+=3; | while(p[0]){ p[0]+=-1; } --p; ++p; p[0]=fgetc(stdin); fputc(p[0],stdout);
p[0]+=3; | p[0]=0; --p; ++p; p[0]=fgetc(stdin); fputc(p[0],stdout);
p[0]+=3; p[0]=0; | --p; ++p; p[0]=fgetc(stdin); fputc(p[0],stdout);
p[0]+=3; p[0]=0; | p+=0; p[0]=fgetc(stdin); fputc(p[0],stdout);
p[0]+=3; p[0]=0; p+=0; | p[0]=fgetc(stdin); fputc(p[0],stdout);
p[0]+=3; p[0]=0; p+=0; p[0]=fgetc(stdin); | fputc(p[0],stdout);
p[0]+=3; p[0]=0; p+=0; p[0]=fgetc(stdin); fputc(p[0],stdout); |
```

As you can see, it's still not perfect. In order to improve it, we need more rules:

* removal of obvious no-ops like `p+=0;`

* removal of memory modifications that are going to be overwritten anyway

* and so on...

So let's do it. Since we're doing matching on patterns, it may be obvious that we should use Scala's pattern matching:

``` scala
object Optimizer{
  def optimize(exprs: List[BfExpr], level: Int): List[BfExpr] = {
    if (level<=0) exprs
    else optimize(opt(exprs),level-1)
  }
  def opt(exprs: List[BfExpr]):List[BfExpr]={
    var optimized = List[BfExpr]() // reversed list of processed elements
    var others = exprs // list of unoptimised elements
    while (others != Nil){
      others match {

        // let's merge identical increases and shifts
        case Increase(i, x)::Increase(j, y)::xs if i==j =>
          others = Increase(i, (x+y)toByte)::xs
        case Increase(_, 0)::xs => others = xs
        case Shift(i)::Shift(j)::xs => others = Shift(i+j)::xs
        case Shift(0)::xs => others = xs

        // let's find assignments to zero
        case While(i, Increase(j, x)::Nil)::xs if i==j && (x%2)!=0 =>
          others = Assign(i,0)::xs

        // let's consolidate all pointer shifts in one place
        case Shift(d)::Increase(i, x)::xs => others = Increase(i+d, x)::Shift(d)::xs
        case Shift(d)::Assign(i, x)::xs => others = Assign(i+d, x)::Shift(d)::xs
        case Shift(d)::Read(i)::xs => others = Read(i+d)::Shift(d)::xs
        case Shift(d)::Write(i)::xs => others = Write(i+d)::Shift(d)::xs
        case Shift(d)::Move(f, t, a)::xs => others = Move(f+d, t+d, a)::Shift(d)::xs
        case Shift(d)::While(i, es)::xs =>
          others = While(i+d, Shift(d)::es:::Shift(-d)::Nil)::Shift(d)::xs

        // let's clean up some assignments
        case Assign(i, x)::Increase(j, y)::xs if i==j =>
          others = Assign(i, (x+y)toByte)::xs
        case Assign(i, x)::Assign(j, y)::xs if i==j =>
          others = Assign(i, y)::xs
        case Increase(i, x)::Assign(j, y)::xs if i==j =>
          others = Assign(i, y)::xs
        case Move(f, t, a)::Assign(j, y)::xs if t==j =>
          others = Assign(j, y)::xs
        case While(i, es)::Increase(j, x)::xs if i==j =>
          others = While(i, es)::Assign(j, x)::xs

        // let's finally create some move instructions
        case While(i, Increase(j,-1)::Increase(d,a)::Nil)::xs if i==j && d!=i =>
          others = Move(i, d, a)::Assign(i, 0)::xs
        case While(i, Increase(d,a)::Increase(j,-1)::Nil)::xs if i==j && d!=i =>
          others = Move(i, d, a)::Assign(i, 0)::xs

        // optimising move operations
        case Move(f1,t1,a1)::Move(f2,t2,a2)::xs if f1==f2 && t1==t2 =>
          others = Move(f1, t1, (a1+a2)toByte)::xs

        // simplifying everything else
        case While(i, es)::xs => 
          optimized = While(i, opt(es))::optimized
          others = xs
        case x::xs => 
          optimized = x::optimized
          others = xs
        case Nil => ()
      }
    }
    optimized reverse
  }
}
```

I've posted here this quite big chunk of code, but I've actually gone way further. I've added more optimisation rules, and a randomised optimisation verificator, resulting in [Screw](https://github.com/KarolS/Screw), one of the most optimising Brainfuck compilers ever.

Just to compare, Screw's results (at a not-too-high optimisation level) are as follows:

    real    0m0.767s
    user    0m0.760s
    sys     0m0.004s
