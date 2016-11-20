---
layout: post
title: "Units 0.1 released"
date: 2014-01-02 20:26
comments: true
published: true
categories: [scala, "units-of-measurement", units, "personal-project", programming]
---

After several months of not-so-intensive work, I present the version 0.1 of the *Units* library: [https://github.com/KarolS/units](https://github.com/KarolS/units).

*Units* is a Scala library for providing type-level units of measurements checked on compile time. The goal of the library was to provide as seamless as possible way to check if the units used in arithmetic expressions are correct.

*If you have tried to compile it: Yes it does compile that long. A clean build takes 100 seconds on my i7.*

Compile-time unit checking has multiple applications:

* scientists will be able to distinguish values in metres per second, metres, metres per second squared [instead of crashing expensive space exploration equipment](http://en.wikipedia.org/wiki/Mars_Climate_Orbiter) or [drugging a patient](https://www.ismp.org/newsletters/acutecare/articles/A3Q99Action.asp)

* engineers will be able to distinguish values in metres, centimetres, feet, inches, litres, gallons [instead of running out of fuel in the middle of a flight](http://en.wikipedia.org/wiki/Air_Canada_Flight_143) or [thinking the distance to travel is many times shorter](http://spectrum.ieee.org/tech-talk/at-work/test-and-measurement/columbuss-geographical-miscalculations)

* designers will be able to distinguish values in millimetres, inches, pixels, points

* economists will be able to distinguish values in euros, dollars, dollars per hour, ounces of gold

* game developers will be able to distinguish values in pixels, tiles, damage points, minerals, barrels of vespen gas

* network software developers will be able to distinguish values in kilobytes, kibibytes, kilobits, kilobits per second

* and so on and on

There are not many languages with units of measurement support, the first that comes to mind is [F#](http://msdn.microsoft.com/en-us/library/dd233243.aspx). I must admit that it is great at this. There also other languages that support units as a first-class language feature, and many that support units with a library. Those libraries vary in their expressibility and versatility, some of them only allow SI units, some of them require you to explicitly express relations between multiplied values, and some of them only support a limited subset of units. There have been earlier Scala libraries with units of measurements, but they all had severe limitations. *Units* library tries to be both expressive and versatile. While it's not as powerful as F# built-in unit support, it definitely allows for quite a bit.

Enough of that, time for some examples that will showcase the main features.

<!-- more -->

Let's start with something simple:

```scala

import io.github.karols.units._
import io.github.karols.units.SI._

val length1 = 1.of[metre]
val length2 = 2.of[metre]
val area1 = 1.of[square[metre]]
val area2 = length1 * length2 

length1 < length2 // OK
area1 + area2 //OK
area1 + length2 // not OK, compile time error

```

The main difference you see is that all the values have a unit defined. The type of the variables in this example is `IntU[metre]` and `IntU[square[metre]]` respectively. It is an `AnyVal` wrapping a 64-bit `Long`. There is also `DoubleU`, which wraps 64-bit `Double`.

The library supports out-of-the-box most SI units, some Imperial and US Customary units, units of information and bandwidth, and many currencies. Most of the units can be semi-automatically converted, i.e. the following code:

```scala
def printMM(length: DoubleU[millimetre]) = {
  println(s"The length is $length mm.")
}

printMM(1.of[inch].convert)

printMM(1.of[metre] + 1.of[millimetre])
```

will correctly print 

```
The length is 25.4 mm.
The length is 1001.0 mm.
```

### Implementation

How does it work, you ask. It's simple. Down deep in the guts of the library, there lurks an implementation of a type-level map from strings to integers, with the following properties:

* strings are sorted lexicographically (so the equality can be structural)

* no integer value is equal to zero (so units to the zeroth power don't matter)

For example, the above units are represented as `{"m" -> 1}` and `{"m" -> 2}` respectively.

You are probably curious how type-level datatypes are defined. I admit that they are implemented pretty simply. Here are [booleans](https://github.com/KarolS/units/blob/master/units/src/main/scala/internal/Bools.scala), here [integers](https://github.com/KarolS/units/blob/master/units/src/main/scala/internal/Integers.scala), here [characters and strings](https://github.com/KarolS/units/blob/master/units/src/main/scala/internal/Strings.scala), here [unit names](https://github.com/KarolS/units/blob/master/units/src/main/scala/internal/SingleUnits.scala) and here [unit maps](https://github.com/KarolS/units/blob/master/units/src/main/scala/internal/UnitImpl.scala).

I'll focus here on integers, because they're simple enough to explain. Here's the code (slightly simplified):

```scala
sealed trait TInteger {
    type Succ <: TInteger
    type Pred <: TInteger
    type Negate <: TInteger
    type Add[X<:TInteger] <: TInteger
    type Mul[X<:TInteger] <: TInteger
    type ZeroNegPos[IfZero<:ResultType, IfNeg[N<:TInteger]<:ResultType, IfPos[N<:TInteger]<:ResultType, ResultType] <: ResultType
    type Equal[X<:TInteger] <: TBool
}
type LambdaNatFalse[N<:TInteger] = False
sealed trait _0 extends TInteger {
    type Succ = Inc[_0]
    type Pred = Dec[_0]
    type Negate = _0
    type Add[X<:TInteger] = X
    type Mul[X<:TInteger] = _0
    type ZeroNegPos[IfZero<:ResultType, IfNeg[N<:TInteger]<:ResultType, IfPos[N<:TInteger]<:ResultType, ResultType] = IfZero
    type Equal[X<:TInteger] <: X#ZeroNegPos[True, LambdaNatFalse, LambdaNatFalse, TBool]
}
sealed trait Inc[N <: TInteger] extends TInteger {
    type Succ = Inc[Inc[N]]
    type Pred = N
    type Negate = Dec[N#Negate]
    type Add[X<:TInteger] = N#Add[X#Succ]
    type Mul[X<:TInteger] = N#Mul[X]#Add[X]
    type ZeroNegPos[IfZero<:ResultType, IfNeg[N<:TInteger]<:ResultType, IfPos[N<:TInteger]<:ResultType, ResultType] = IfPos[N]
    type Equal[X<:TInteger] <: X#ZeroNegPos[False, LambdaNatFalse, N#Equal, TBool]
}
sealed trait Dec[N <: TInteger] extends TInteger {
    type Succ = N
    type Pred = Dec[Dec[N]]
    TInteger]
    type Negate = Inc[N#Negate]
    type Add[X<:TInteger] = N#Add[X#Pred]
    type Mul[X<:TInteger] = N#Mul[X]#Add[X#Negate]
    type ZeroNegPos[IfZero<:ResultType, IfNeg[N<:TInteger]<:ResultType, IfPos[N<:TInteger]<:ResultType, ResultType] = IfNeg[N]
    type Equal[X<:TInteger] <: X#ZeroNegPos[False, N#Equal, LambdaNatFalse, TBool]
}
```

As you can see:

* The code looks like normal code, but with `type` instead of `def` and with `Object#Method[Param]` instead of `object.method(param)`.

* `_0` is the type-level zero, `Dec` is a type-level negative number, and `Inc` is a type-level positive number.

* `Succ` and `Pred` are successor and predecessor respectively;

* `Negate`, `Add` and `Mul` are defined recursively in a pretty straightforward way.

* `ZeroNegPos` is integer pattern matching. It's explicitly polymorphic. It takes four parameters: the result for when the number is zero, the function for when the number is positive, the function for when the number is negative, and the result type. If the result is zero, the first parameter is returned. If it's not, the correct function is applied to the underlying value (the integer that is closer to zero), yielding the result. For example, let's have a look at `Inc#Equal`: `type Equal[X<:TInteger] <: X#ZeroNegPos[False, LambdaNatFalse, N#Equal, TBool]`

    * if `X` is `_0`, returns `False`

    * if `X` is `Dec[Y]`, returns `LambdaNatFalse[Y]`, i.e. `False`

    * if `X` is `Inc[Y]`, returns `N#Equal[Y]` (a recursive call)

(`True` and `False` are type-level booleans)

Type-level strings are defined using a custom type-level encoding that supports only ASCII letters and several symbols useful for defining units (including the degree sign `°` and capital omega `Ω`).

You can probably guess how the arithmetic works:

* adding and subtracting values requires both units to match

* multiplying causes the values with the same keys be added together

* dividing causes the values with the same keys be subtracted

* all missing keys in the unit map have value zero

* if after adding or subtracting you get zero, you remove the key



### Defining units

Defining your own units is quite easy. The most basic thing you have to do is to define a unit and its type-level string. The string will be used to display the name of the unit (so better pick something that makes sense), but also to distinguish units themselves (so better pick something unique).

```scala
import io.github.karols.units._
import io.github.karols.units.defining._

type fortnight = DefineUnit[_f ~: _o ~: _r ~: _t ~: _n ~: _i ~: _g ~: _h ~: _t]
```

That's it!

You can also define some conversions:

```scala
import io.github.karols.units.SI._

implicit val fortnight_to_day = one[fortnight].contains(14)[day]

println((1.of[fortnight] + 3.of[day]).mkString) // prints "17 d"
```

Sadly, you need to define conversions from fortnights to other units manually, and then you'll have to define conversions of compound units, like from *mile/fortnight* to *kilometre/day*. The ratios have several operators defined to help with this, consult the documentation and source for more info.

### Affine spaces

The library preserves also another distinction: between normal values with units and elements of affine spaces. Affine spaces (also known as torsor spaces) are spaces that contain elements that cannot be added or multiplied, because those operations make no sense. For example, temperature is such space: there's no reason to say "yesterday it was 3°C, today it's 8°C, the sum of these is 11°C". For these applications, the library provides `IntA` and `DoubleA` types. Subtracting two values of such type (using `--` operator) yields a value of `IntU` or `DoubleU` type, which we can call here the difference type.

The affine spaces often have an arbitrarily selected zero point. The zero point has no special properties, unlike the zero element of a vector space. It's simply chosen to provide people a frame of reference.

Some examples of affine spaces, their zero points, and their difference spaces:

* temperature is an affine space, its zero point is the temperature of zero degrees, and its difference space is the space of temperature differences

* timestamps form an affine space, its zero point is an arbitrary moment in time (usually first midnight of the 1st day of January 1900, 1904, 1970, or 2000), and its difference space is time

* positions form an affine space, its zero point is usually called an origin, and its difference space is the corresponding vector space

Adding or multiplying temperatures, timestamps or positions doesn't make sense. Adding to them a value from the corresponding difference space (this value is usually called an displacement) makes sense and yields another value from the affine space.

*Units* library suggests using `IntA` and `DoubleA` for values from affine spaces and `IntU` and `DoubleU` for values from difference spaces.

Out-of-the-box *Units* defines Celsius and Fahrenheit scales and Unix timestamps in second, millisecond and nanosecond precisions.

### Arrays

In Scala, the only unboxed collection types are arrays of primitive types. So all the other collections, and also arrays of custom value classes, are boxed. This leads to serious performance implications. You may think it would be easier to just use non-type safe code and revert back to raw doubles and longs.

To prevent this, the *Units* library provides array classes for `DoubleU`, `IntU`, `DoubleA` and `IntA`. According to few simple benchmarks that are available, the classes `DoubleUArray`, `IntUArray`, `DoubleAArray` and `IntAArray` are as fast as raw `Array[Double]` and `Array[Long]`.

Besides, you might have asked before:

> Hey, you **can** add temperatures, provided you divide them immediately afterwards to get an average!

All of these array classes provide an `avg` method, so you can write:

```scala
val t1 = 3.at[CelsiusScale]
val t2 = 8.at[CelsiusScale]

val average = IntAArray(t1,t2).avg

println(average.mkString) // prints 5.5 °C
```

### And there's more!

The *Units* library also supports 2D and 3D vectors (both normal and affine), unboxed efficient vector arrays, semi-automatic affine space conversions, various ways to express functions with unit polymorphism (sadly, none of them as clean as in F#) and interoperability layers for many libraries (Scalaz, Spire, Slick, JodaTime, Algebird, Scalacheck, more to come).

The next goal is to get it to Sonatype.
















