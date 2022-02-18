binder:https://mybinder.org/v2/gh/freechipsproject/chisel-bootcamp/b6bff4acf4cec99e78e60b9be6cbc14eb6bb3c38

# Module 1: Introduction to Scala

## Variables and Constants

`val` is constant, and `var` is mutable(also can be reassigned).

## Conditionals

We should add a pair of parentheses around the condition.

> Some punctuations:
>
> () -> parentheses
>
> [] -> brackets
>
> {} -> braces
>
> , -> comma
>
> : -> colon
>
> ; -> semicolon
>
> . -> period
>
> ? -> question mark
>
> ! -> exclamation point

Sometimes we can omit the braces, for example:

```scala
if (condition)
	println("hello, world")
else
	a += 1
```

Additionally, the `else if` should start at a new line.

Like Rust, every `if` condition returns a value.

## Methods(Functions)

We use keyword `def` to define a function. It has arguments and return value, like:

```scala
def double(x: Int): Int = 2 * x
```

If the function does not require any arguments, then no side effects => no parentheses.

If the function does not have return values, then no colon.

Overloading functions: It is okay if two functions have the same names but different signatures.

Here is an example of recursive and nested functions:

```scala
/** Prints a triangle made of "X"s
  * This is another style of comment
  */
def asciiTriangle(rows: Int) {
    
    // This is cute: multiplying "X" makes a string with many copies of "X"
    def printRow(columns: Int): Unit = println("X" * columns)
    
    if(rows > 0) {
        printRow(rows)
        asciiTriangle(rows - 1) // Here is the recursive call
    }
}

// printRow(1) // This would not work, since we're calling printRow outside its scope
asciiTriangle(6)
```

## List

```scala
val x = 7
val y = 14
// Two ways to assemble a list
val list1 = List(1, 2, 3)
val list2 = x :: y :: y :: Nil
// append list2 to list1
val list3 = list1 ++ list2
// get length & size
val m = list2.length
val s = list2.size
// get first element
val head = list1.head
// get rest except the first element
val tail = list2.tail
// access by index
val v = list1(2)
```

## `for` statement

```scala
for (i <- 0 to 7) {...} // include 7, step = 1
for (i <- 0 until 7) {...} // exclude 7, step = 1
for (i <- 0 to 10 by 2) {...} // include 10, step = 2
for (v <- randomList) {...} // iterate over a list
```

## packages and imports

Package names are lower case and do not contain separators.

Common imports from `chisel3`:

```scala
// A wildcard include all classes and methods.
import chisel3._
import chisel3.iotesters.{ChiselFlatSpec, Driver, PeekPokeTester}
```

## Scala is Object Oriented 

Variables, constants, literals, even functions are all objects.
Objects are instances of classes.
Programmers specify immutable(`val`) and mutable(`var`) fields and methods when defining a class.
Subclasses can be extended from their superclasses and inherit their fields and methods.
Subclasses can also inherit from traits. They are a kind of lightweighted class, and a class can inherit from multiple of them.

## Class Example

```scala
// WrapCounter counts up to a max value based on a bit size
class WrapCounter(counterBits: Int) {

  val max: Long = (1 << counterBits) - 1
  var counter = 0L
    
  def inc(): Long = {
    counter = counter + 1
    if (counter > max) {
        counter = 0
    }
    counter
  }
  println(s"counter created with max value $max")
}
```

defining code block: run when an instance is initialized(or constructed)

class methods are wrapped in the defining code block

s before `""` means interpolated string, and by using `$` you can print the value(formatting-like)

## Instantiation

```scala
val x = new WrapCounter(2)

x.inc()
// or x inc()

// member variables/values are public by default
if (x.counter == x.max) {
    println("counter is about to wrap")
}
```

## Code Blocks

code blocks are also expressions(like rust)

type of an empty code block `{}` is `Unit`

code blocks with parameters:

```scala
// style 1
def add1(c: Int): Int = c + 1
class RepeatString(s: String): String {
    val repeatedString = s + s
}

// style 2
val intList = List(1, 2, 3)
// closure(anonymous function)
val stringList = intList.map { i => 
    i.toString
}
```

## Passing parameters

```scala
// prototype
def myMethod(count: Int, wrap: Boolean, wrapValue: Int = 24): Unit = { ... }
// normal order
myMethod(count = 10, wrap = false, wrapValue = 23)
// different order
myMethod(wrapValue = 23, wrap = false, count = 10)
// we have to provide value for the arguments which does not have default value
```

