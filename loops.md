# Scala Loops  

This document covers looping constructs in Scala 3. It includes the `for`  
expression, `for` comprehension, `while` loop, `do-while` loop, and  
functional iteration methods. Examples progress from simple range iteration  
to multi-generator comprehensions, guards, and break support.  

## For loop with a range  

This example demonstrates the most basic `for` loop iterating over an  
inclusive integer range.  

```scala
@main def main() =

    for n <- 1 to 5 do
        println(n)
end main
```  

The `1 to 5` expression creates a `Range` that includes both endpoints, so  
the loop body executes for `n` equal to 1, 2, 3, 4, and 5. Scala 3 uses  
the `for … do` syntax without curly braces. No explicit counter variable  
or increment expression is needed.  

## Half-open range  

This example uses `until` to create a range that excludes the upper bound.  

```scala
@main def main() =

    for n <- 0 until 5 do
        println(n)
end main
```  

`0 until 5` yields the values 0, 1, 2, 3, and 4 — the value 5 is not  
included. Half-open ranges align naturally with zero-based indexing: the  
range `0 until list.length` covers every valid index of a sequence.  

## Range with step  

This example specifies a custom step value to skip elements in a range.  

```scala
@main def main() =

    for n <- 0 to 30 by 5 do
        println(n)
end main
```  

The `by` method appends a step to any range. Here the loop produces 0, 5,  
10, 15, 20, 25, and 30. The step can be any non-zero integer. Using `by`  
is equivalent to multiplying the loop variable by the step in other  
languages.  

## Descending range  

This example iterates in reverse order using a negative step.  

```scala
@main def main() =

    for n <- 10 to 1 by -1 do
        println(n)
end main
```  

To count down, the start must exceed the end and the step must be  
negative. Without `by -1`, the range `10 to 1` would be empty because  
the default step is `+1`. Descending ranges are useful for reverse  
printing or countdown timers.  

## Iterating over a list  

This example shows how to loop over every element of a `List`.  

```scala
@main def main() =

    val fruits = List("apple", "banana", "cherry", "date")

    for fruit <- fruits do
        println(fruit)
end main
```  

The generator `fruit <- fruits` binds each list element to the variable  
`fruit` in turn. The loop body runs once per element in order. Scala's  
`for` expression works uniformly across all iterable collections, so  
switching between `List`, `Array`, or `Vector` requires no change to  
the loop syntax.  

## Iterating over an array  

This example demonstrates iterating over the elements of an `Array`.  

```scala
@main def main() =

    val nums = Array(10, 20, 30, 40, 50)

    for n <- nums do
        println(n)
end main
```  

An `Array` is treated exactly like any other sequence by the `for`  
expression. The compiler generates efficient bytecode for array loops,  
making this form suitable for performance-sensitive code.  

## Iterating over a map  

This example destructures key-value pairs while looping over a `Map`.  

```scala
@main def main() =

    val capitals = Map(
        "France"  -> "Paris",
        "Germany" -> "Berlin",
        "Spain"   -> "Madrid",
        "Italy"   -> "Rome"
    )

    for (country, capital) <- capitals do
        println(s"$country -> $capital")
end main
```  

A `Map` yields `(key, value)` tuples. The tuple pattern  
`(country, capital)` in the generator destructures each entry,  
binding the key and value to named variables. This avoids explicit  
calls to `._1` and `._2` and makes the intent immediately clear.  

## Iterating with index  

This example accesses both element and position during iteration using  
`zipWithIndex`.  

```scala
@main def main() =

    val colors = List("red", "green", "blue", "yellow")

    for (color, idx) <- colors.zipWithIndex do
        println(s"$idx: $color")
end main
```  

`zipWithIndex` pairs each element with its zero-based index, returning a  
sequence of `(element, index)` tuples. The tuple pattern in the generator  
unpacks both values at once. This technique is useful for building  
numbered lists or when the element's position is needed alongside its  
value.  

## Iterating over a string  

This example iterates over the individual characters of a `String`.  

```scala
@main def main() =

    val word = "Scala"

    for ch <- word do
        println(ch)
end main
```  

A `String` in Scala is implicitly converted to `IndexedSeq[Char]`, so  
it can be used as a generator directly. Each iteration binds one  
character. This makes character-level processing concise without  
calling `toCharArray` or using an index variable.  

## Guard  

This example filters loop iterations using an `if` guard.  

```scala
@main def main() =

    for n <- 1 to 20 if n % 2 == 0 do
        println(n)
end main
```  

A guard is an `if` condition placed inside the `for` header. The loop  
body executes only for iterations where the condition holds. Here only  
even numbers are printed. Guards keep filtering logic close to the  
iteration, avoiding a separate `filter` call or a nested `if` inside  
the body.  

## Guards on a collection  

This example applies a guard to filter words from a list.  

```scala
@main def main() =

    val words = List("sky", "war", "water", "rain",
        "some", "cup", "train", "wrinkle", "worry")

    for
        word <- words
        if word.startsWith("w")
    do
        println(word)
end main
```  

The guard `if word.startsWith("w")` skips any word that does not begin  
with the letter `w`. Only the matching words — `war`, `water`,  
`wrinkle`, and `worry` — are passed to the loop body and printed.  

## Multiple guards  

This example combines two guards to apply an AND condition.  

```scala
@main def main() =

    val words = List("sky", "war", "water", "rain",
        "some", "cup", "train", "wrinkle", "worry")

    for
        word <- words
        if word.startsWith("w")
        if word.endsWith("r")
    do
        println(word)
end main
```  

Multiple `if` guards on separate lines act as a logical AND — a word  
must satisfy every guard to reach the loop body. Here only words that  
both start with `"w"` and end with `"r"` are printed, producing `war`  
and `wrinkle`. Placing each guard on its own line improves readability  
when conditions are complex.  

## Multiple generators  

This example uses two generators to produce every combination of two  
lists, equivalent to nested loops.  

```scala
@main def main() =

    val letters = List('A', 'B', 'C')
    val digits  = List(1, 2, 3)

    for
        letter <- letters
        digit  <- digits
    do
        print(s"$letter$digit ")

    println()
end main
```  

Each additional generator introduces a nested iteration. The rightmost  
generator varies fastest, so the output is `A1 A2 A3 B1 B2 B3 C1 C2  
C3`. Multiple generators are the `for`-expression equivalent of nested  
loops but without extra levels of indentation.  

## Nested loops  

This example uses explicit nested `for` loops to print a multiplication  
table.  

```scala
@main def main() =

    for i <- 1 to 5 do
        for j <- 1 to 5 do
            print(f"${i * j}%4d")
        println()
end main
```  

The outer loop controls the row and the inner loop controls the column.  
`f"${i * j}%4d"` formats each product into a four-character wide field  
so the columns align. Nested `for` loops are straightforward to write in  
Scala 3 using indentation instead of curly braces.  

## For comprehension  

This example collects loop results into a new list using `yield`.  

```scala
@main def main() =

    val nums = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    val squares = for
        n <- nums
        if n % 2 == 0
    yield n * n

    println(squares)
end main
```  

When `yield` is present the `for` expression becomes a comprehension  
that builds and returns a new collection. The type of the result matches  
the source collection — here `List[Int]`. The guard removes odd numbers  
and `yield n * n` squares each remaining element. The original list is  
not modified.  

## For comprehension with multiple generators  

This example produces a flat list of string pairs from two generators.  

```scala
@main def main() =

    val letters = List('A', 'B', 'C', 'D', 'E')
    val digits  = List(1, 2, 3, 4, 5)

    val pairs = for
        letter <- letters
        digit  <- digits
    yield s"$letter$digit"

    println(pairs)
end main
```  

The comprehension with two generators returns a flat `List[String]` of  
all combinations. For five letters and five digits the result contains  
25 elements. The compiler desugars multi-generator comprehensions into  
nested `flatMap` and `map` calls, so no intermediate collections are  
needed.  

## For comprehension with a case class  

This example filters a list of objects using a `for` comprehension and  
a guard.  

```scala
case class User(name: String, age: Int)

@main def main() =

    val users = List(
        User("John",     18),
        User("Brian",    31),
        User("Veronika", 23),
        User("Lucia",    48),
        User("Peter",    21)
    )

    val result = for
        user <- users
        if user.age >= 20 && user.age < 30
    yield user

    result.foreach(println)
end main
```  

The comprehension iterates over a `List[User]` and collects only those  
users whose age falls in the half-open range `[20, 30)`. The result is  
a new `List[User]` containing Veronika and Peter. Using a `for`  
comprehension instead of `filter` makes the intent explicit and allows  
additional guards or generators to be added without restructuring.  

## Foreach method  

This example iterates over a collection using the `foreach` higher-order  
method.  

```scala
@main def main() =

    val nums = List(1, 2, 3, 4, 5)

    nums.foreach(n => println(n * n))
end main
```  

`foreach` accepts a function and applies it to every element in order.  
It is the functional counterpart to the `for … do` loop and is  
equivalent to calling `for n <- nums do f(n)`. The lambda  
`n => println(n * n)` squares and prints each element. When the  
function argument is a single method call, it can be shortened to  
`nums.foreach(println)`.  

## While loop  

This example demonstrates computing a sum using a `while` loop.  

```scala
@main def main() =

    val nums = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val n    = nums.length

    var i   = 0
    var sum = 0

    while i < n do
        sum += nums(i)
        i   += 1

    println(s"Sum: $sum")
end main
```  

The `while` loop repeats its body as long as the condition `i < n` is  
`true`. The variables `i` and `sum` must be declared with `var` because  
they are mutated on each iteration. The `while` loop is the right choice  
when the number of iterations is not known in advance or when index  
arithmetic is necessary.  

## Do-while loop  

This example shows a `do-while` loop that executes its body at least  
once before checking the condition.  

```scala
@main def main() =

    var n = 1

    do
        println(n)
        n += 1
    while n <= 5
end main
```  

A `do-while` loop guarantees that the body runs at least once, because  
the condition is evaluated after the first execution. Here `n` starts at  
1, the body prints it and increments it, and the loop continues while  
`n <= 5`. This construct is useful when an initial action must be  
performed before the exit condition can be meaningful.  

## While loop with accumulator  

This example processes elements using a `while` loop and an  
accumulator variable.  

```scala
@main def main() =

    val prices = Array(12.5, 8.0, 22.3, 5.75, 14.0)
    var total  = 0.0
    var i      = 0

    while i < prices.length do
        total += prices(i)
        i     += 1

    println(f"Total: $total%.2f")
end main
```  

Accumulator variables collect a running result across iterations. The  
pattern — initialise before the loop, update inside, consume after —  
mirrors imperative loops in other languages. In idiomatic Scala the  
same result is often achieved with `prices.sum` or  
`prices.foldLeft(0.0)(_ + _)`, but the `while` form makes the  
mechanics explicit.  

## Infinite loop with a condition check  

This example shows how to create an intentional infinite loop that  
terminates based on a runtime condition.  

```scala
import scala.util.Random

@main def main() =

    var attempts = 0

    while true do
        val n = Random.nextInt(10) + 1
        attempts += 1
        if n == 7 then
            println(s"Found 7 after $attempts attempt(s)")
            return

end main
```  

`while true do` creates an infinite loop. The only way to exit is via  
`return`, `throw`, or — with the Breaks library — a `break` call.  
`Random.nextInt(10) + 1` generates a random integer in `[1, 10]`. The  
loop counts attempts and exits as soon as the value 7 is produced.  

## Breaking out of a loop  

This example uses `scala.util.control.Breaks` to exit a loop early.  

```scala
import scala.util.control.Breaks.*

@main def main() =

    val nums = List(3, 7, 2, 9, 1, 5, 8, 4)

    breakable:
        for n <- nums do
            if n == 9 then
                println(s"Found 9, stopping")
                break()
            println(n)
end main
```  

Scala does not have a `break` keyword. The `Breaks` object provides  
`breakable` and `break()` as a library-level replacement. The `break()`  
call throws a special exception that is caught by the enclosing  
`breakable` block, ending the loop immediately. Import  
`scala.util.control.Breaks.*` to bring both names into scope.  

## Looping with a local function  

This example replaces a loop with a tail-recursive helper function.  

```scala
@main def main() =

    def sumTo(n: Int): Int =
        var acc = 0
        var i   = 1
        while i <= n do
            acc += i
            i   += 1
        acc

    println(sumTo(100))
end main
```  

Enclosing loop logic inside a local function keeps the surrounding code  
clean and makes the operation reusable. `sumTo` computes the sum of all  
integers from 1 to `n` using a `while` loop. The function returns `acc`  
after the loop completes. Defining helper functions locally is idiomatic  
in Scala and avoids polluting the outer scope.  


