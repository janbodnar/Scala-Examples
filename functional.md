# Foundations of Functional Programming in Scala 3

Functional programming is a paradigm that treats computation as the  
evaluation of mathematical functions. It emphasises expressions over  
statements, immutability over in-place mutation, and function composition  
over imperative loops. Programs written in this style are easier to reason  
about, test, and parallelise because they avoid shared mutable state and  
hidden side effects. Scala 3 supports functional programming natively  
while remaining fully interoperable with the Java Virtual Machine.  

**Pure functions and referential transparency**  

A pure function always produces the same result for the same input and  
performs no observable side effects such as writing to disk, mutating  
shared state, or throwing exceptions. Referential transparency is the  
property that any expression in a program may be replaced by the value  
it evaluates to without changing the overall behaviour. Together, purity  
and referential transparency make programs easy to test and refactor  
because each function behaves like an isolated mathematical relation.  

**Immutability and persistent data structures**  

Immutability means that once a value is created it cannot be modified in  
place. Instead of changing existing objects, functional programs build new  
values derived from the old ones. Persistent data structures implement  
this efficiently by sharing unchanged parts of the old structure with the  
new one. Scala's standard-library collections — `List`, `Vector`, `Set`,  
and `Map` — are immutable and persistent by default. Structural sharing  
keeps memory usage low even when many derived collections coexist.  

**Higher-order functions**  

A higher-order function either accepts a function as an argument or returns  
a function as its result. Higher-order functions are the primary abstraction  
tool in functional programming. They eliminate duplicated loop patterns by  
expressing common operations — transformation, selection, and aggregation  
— as reusable building blocks. Scala's `map`, `filter`, `flatMap`, and  
`fold` are all higher-order functions that operate uniformly over any  
collection.  

**Algebraic data types**  

An algebraic data type (ADT) is a composite type formed by combining  
simpler types in one of two ways. A *product type* bundles several fields  
together; in Scala 3 this is typically a case class. A *sum type*  
represents a choice among several distinct alternatives; Scala 3 expresses  
these as sealed trait hierarchies or `enum` definitions. ADTs capture the  
precise shape of data and integrate seamlessly with pattern matching.  

**Pattern matching**  

Pattern matching is a control-flow construct that tests the shape and  
content of a value and branches accordingly. It is far more expressive  
than `if`-`else` chains because it can simultaneously inspect structure,  
bind inner values to names, apply guard predicates, and provide  
compile-time exhaustiveness checking. Pattern matching integrates directly  
with ADTs, making it the natural companion to case classes and enums.  

**Functional error handling**  

Rather than relying on exceptions, functional code represents failure as  
an ordinary value. Scala provides three standard wrapper types for this  
purpose. `Option[A]` models the possible absence of a value: it is either  
`Some(a)` or `None`. `Either[E, A]` models a result that may fail with a  
typed error: it is either `Left(error)` or `Right(value)`. `Try[A]` wraps  
a computation that may throw: it is either `Success(value)` or  
`Failure(throwable)`. All three types support `map`, `flatMap`, and  
for-comprehensions.  

**Why functional programming fits Scala 3**  

Scala 3 was designed with functional programming as a first-class concern.  
The language provides expressive ADT syntax through `enum` and sealed  
hierarchies, exhaustive pattern matching with compile-time checks,  
immutable collections in the standard library, and a powerful type system  
supporting type classes, given/using, and extension methods. The  
indentation-based syntax reduces visual noise and makes functional  
pipelines easy to read. All these features make Scala 3 an exceptionally  
capable host for the functional style.  

---

## A pure function

A pure function derives its result solely from its inputs and produces  
no observable side effects.  

```scala
@main def main() =

    def square(n: Int): Int = n * n

    def add(x: Int, y: Int): Int = x + y

    println(square(5))
    println(add(3, 7))
    println(square(add(2, 3)))
end main
```

The `square` and `add` functions always return the same value for the  
same arguments and touch no external state. Calling `square(5)` always  
yields `25`, making the function entirely predictable. Composing pure  
functions — as shown in `square(add(2, 3))` — is safe and produces no  
surprises because each function is an isolated transformation.  

## Referential transparency

Referential transparency means a pure expression can be substituted by  
its value anywhere without altering the program's meaning.  

```scala
@main def main() =

    def double(n: Int): Int = n * 2

    val a = double(4)       // a == 8
    val b = double(4) + 1   // equivalent to 8 + 1

    println(a)
    println(b)
    println(double(4) == a) // true: always interchangeable
end main
```

Because `double` is pure, `double(4)` can be replaced by `8` everywhere  
in the source without changing behaviour. This property enables equational  
reasoning: you can mentally substitute any call with its result to  
simplify and verify program logic. Referential transparency is the formal  
criterion that distinguishes a pure function from one with hidden side  
effects.  

## Immutable values

Using `val` creates a binding that cannot be reassigned after its initial  
definition.  

```scala
@main def main() =

    val basePrice = 19.99
    val taxRate   = 0.08
    val total     = basePrice * (1 + taxRate)

    println(s"Base price : $basePrice")
    println(s"Tax rate   : $taxRate")
    println(s"Total      : $total")
end main
```

Each `val` is bound exactly once and the compiler rejects any attempt to  
reassign it. This eliminates an entire class of bugs caused by accidental  
mutation and makes every binding carry its original value throughout its  
entire scope. Reasoning about the program becomes straightforward because  
names are stable aliases for values, not containers that can change at  
any moment.  

## Immutable collections

Transforming an immutable collection always returns a new collection  
while the original remains unchanged.  

```scala
@main def main() =

    val nums    = List(1, 2, 3, 4, 5)
    val doubled = nums.map(_ * 2)
    val evens   = nums.filter(_ % 2 == 0)

    println(s"Original : $nums")
    println(s"Doubled  : $doubled")
    println(s"Evens    : $evens")
end main
```

`map` and `filter` produce brand-new lists; `nums` is never modified. All  
three bindings can coexist in memory simultaneously because the underlying  
cons cells are shared where possible. Immutable collections make  
concurrent code safe: multiple threads can read the same list without  
synchronisation or defensive copying.  

## Avoiding side effects

A function free of side effects does not write to the console, modify  
external state, or perform I/O during its computation.  

```scala
@main def main() =

    def celsius(f: Double): Double = (f - 32) * 5.0 / 9.0

    val temperatures = List(32.0, 77.0, 212.0)
    val converted    = temperatures.map(celsius)

    converted.foreach(c => println(f"$c%.2f °C"))
end main
```

The `celsius` function reads only its parameter and performs no I/O.  
Side effects such as `println` are deferred to the program boundary  
inside `main`, while the computation itself stays clean and testable.  
Separating pure logic from I/O is one of the most practical benefits of  
functional programming: the transformation layer can be unit-tested  
without mocking file systems, databases, or streams.  

## Persistent data structures

Prepending to a Scala list creates a new list that shares its tail with  
the original, consuming minimal additional memory.  

```scala
@main def main() =

    val original = List(2, 3, 4)
    val extended = 1 :: original

    println(s"Original   : $original")
    println(s"Extended   : $extended")
    println(s"Tail shared: ${extended.tail eq original}")
end main
```

The `::` operator creates a single new cons cell whose `tail` points  
directly to `original`. No elements are copied. Persistent data  
structures rely on this structural sharing to keep immutability  
affordable: adding a new head is an O(1) operation regardless of the  
list's length. The `eq` check confirms that both lists share the exact  
same in-memory tail object.  

## Recursion

Recursion expresses repetitive computation by having a function call  
itself with a reduced sub-problem until a base case is reached.  

```scala
@main def main() =

    def factorial(n: Int): BigInt =
        if n <= 1 then 1
        else n * factorial(n - 1)

    println(factorial(0))
    println(factorial(5))
    println(factorial(10))
end main
```

`factorial` delegates to itself with `n - 1` on every recursive call.  
The base case `n <= 1` terminates the descent and returns `1`. The call  
stack then unwinds, multiplying each intermediate result. `BigInt`  
prevents integer overflow for large inputs. Recursion replaces mutable  
loop counters with immutable function parameters, which is idiomatic in  
functional code.  

## Tail recursion

A tail-recursive function calls itself only as the very last action,  
allowing the compiler to replace the call stack with a loop.  

```scala
@main def main() =

    import scala.annotation.tailrec

    @tailrec
    def factorial(n: Int, acc: BigInt = 1): BigInt =
        if n <= 1 then acc
        else factorial(n - 1, acc * n)

    println(factorial(10))
    println(factorial(20))
end main
```

The `@tailrec` annotation instructs the compiler to verify that the  
recursive call is in tail position. When it is, the compiler emits a JVM  
loop instead of a recursive call, keeping stack depth constant regardless  
of input size. The accumulator parameter `acc` carries the partial result  
forward, eliminating the post-call multiplication that prevented the  
naive version from being tail-recursive.  

---

## Functions as values

In Scala 3 a function defined with `def` can be converted to a first-class  
value and stored in a `val`.  

```scala
@main def main() =

    def increment(n: Int): Int = n + 1

    val inc: Int => Int    = increment  // eta-expansion
    val double: Int => Int = _ * 2

    println(inc(5))
    println(double(5))
    println(List(1, 2, 3).map(inc))
end main
```

Assigning `increment` to `inc` triggers *eta-expansion*: the compiler  
wraps the method in a function object automatically. The `_` shorthand  
creates an anonymous function value in a single character. Both `inc` and  
`double` can be passed to `map` just like any other value, demonstrating  
that functions are ordinary first-class citizens in Scala.  

## Passing functions as parameters

Higher-order functions accept function values as arguments, enabling  
behaviour to be injected at the call site.  

```scala
@main def main() =

    def applyTwice(f: Int => Int, x: Int): Int = f(f(x))

    println(applyTwice(_ + 3, 7))     // 13
    println(applyTwice(_ * 2, 4))     // 16
    println(applyTwice(Math.abs, -5)) // 5
end main
```

`applyTwice` does not care what `f` does; it simply applies it twice to  
`x`. Callers supply any compatible `Int => Int` function — an anonymous  
lambda, a partially applied method, or a named function value. This  
technique abstracts over behaviour, replacing multiple specialised  
functions with a single, reusable higher-order one.  

## Returning functions

A function can return another function, capturing its argument in a  
closure.  

```scala
@main def main() =

    def multiplier(factor: Int): Int => Int =
        (n: Int) => n * factor

    val triple   = multiplier(3)
    val tenTimes = multiplier(10)

    println(triple(7))
    println(tenTimes(5))
    println(List(1, 2, 3).map(triple))
end main
```

`multiplier` returns a lambda that closes over `factor`. Each call to  
`multiplier` creates a distinct closure bound to a different value of  
`factor`. The returned functions `triple` and `tenTimes` behave like any  
other `Int => Int` and can be passed directly to `map`, confirming that  
returned functions are fully first-class values.  

## The map function

`map` applies a function to every element of a collection and returns a  
new collection of the same shape with the transformed values.  

```scala
@main def main() =

    val words   = List("hello", "scala", "world")
    val upper   = words.map(_.toUpperCase)
    val lengths = words.map(_.length)
    val squares = Vector(1, 2, 3, 4, 5).map(n => n * n)

    println(upper)
    println(lengths)
    println(squares)
end main
```

`map` transforms each element independently and always produces a new  
collection without mutating the original. The anonymous function  
`_.toUpperCase` is shorthand for `(s: String) => s.toUpperCase`. Because  
`map` is defined on virtually every collection type in Scala, learning  
it once unlocks a universal transformation pattern.  

## The flatMap function

`flatMap` applies a function that returns a collection to each element  
and flattens the nested results into a single collection.  

```scala
@main def main() =

    val sentences = List("hello world", "scala is fun")
    val words     = sentences.flatMap(_.split(" ").toList)

    val numbers = List(1, 2, 3)
    val pairs   = numbers.flatMap(n => List(n, n * 10))

    println(words)
    println(pairs)
end main
```

If plain `map` were used, the result would be a `List[List[String]]`.  
`flatMap` collapses the extra nesting by concatenating all inner lists.  
This makes it essential for chaining dependent computations, including  
for-comprehensions, which are syntactic sugar built directly on `flatMap`  
and `map`.  

## The filter function

`filter` selects only those elements of a collection that satisfy a  
given predicate, discarding the rest.  

```scala
@main def main() =

    val nums  = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val evens = nums.filter(_ % 2 == 0)
    val odds  = nums.filterNot(_ % 2 == 0)
    val big   = nums.filter(_ > 6)

    println(evens)
    println(odds)
    println(big)
end main
```

`filter` iterates every element and retains those for which the predicate  
returns `true`. `filterNot` is the complement, keeping elements for which  
the predicate returns `false`. Both produce new immutable collections and  
leave the original list untouched. Predicates are ordinary `A => Boolean`  
function values and can be composed with `&&` and `||`.  

## foldLeft

`foldLeft` traverses a collection from left to right, combining each  
element with a running accumulator.  

```scala
@main def main() =

    val nums = List(1, 2, 3, 4, 5)

    val sum     = nums.foldLeft(0)(_ + _)
    val product = nums.foldLeft(1)(_ * _)
    val joined  = nums.foldLeft("")((acc, n) => acc + n.toString)

    println(s"Sum     : $sum")
    println(s"Product : $product")
    println(s"Joined  : $joined")
end main
```

`foldLeft` starts with the seed value and applies the binary function to  
the accumulator and each element in turn. The expression  
`foldLeft(0)(_ + _)` is equivalent to `((((0+1)+2)+3)+4)+5`. Almost all  
collection reductions — from `sum` to string concatenation — can be  
expressed as a fold with an appropriate seed and combinator.  

## foldRight

`foldRight` traverses a collection from right to left, which is natural  
for operations that build right-associative structures.  

```scala
@main def main() =

    val nums = List(1, 2, 3, 4, 5)

    val sumR    = nums.foldRight(0)(_ + _)
    val rebuilt = nums.foldRight(List.empty[Int])(_ :: _)

    println(s"Sum (right) : $sumR")
    println(s"Rebuilt     : $rebuilt")
end main
```

`foldRight(List.empty[Int])(_ :: _)` reconstructs the original list by  
prepending each element to the accumulator from the right. For commutative  
operations such as addition the result equals `foldLeft`, but the  
associativity differs for non-commutative ones. `foldRight` is the  
canonical way to build new lists from existing ones in a functional style.  

## Currying

A curried function takes its arguments one at a time in separate parameter  
lists rather than all at once in a single list.  

```scala
@main def main() =

    def add(x: Int)(y: Int): Int = x + y

    val add5 = add(5)

    println(add5(3))    // 8
    println(add5(10))   // 15
    println(add(2)(3))  // 5
end main
```

Declaring `add` with two parameter lists makes it a curried function.  
Applying only the first argument, `add(5)`, returns an `Int => Int`  
function that waits for `y`. This technique is the basis for partial  
application: locking in some arguments in advance to produce a  
specialised, reusable function from a general one.  

## Partial application

Partial application fixes some arguments of a multi-parameter function  
and returns a new function that accepts the remaining ones.  

```scala
@main def main() =

    def power(base: Int, exp: Int): Int =
        math.pow(base, exp).toInt

    val square = (n: Int) => power(n, 2)
    val cube   = (n: Int) => power(n, 3)

    println(List(1, 2, 3, 4, 5).map(square))
    println(List(1, 2, 3, 4, 5).map(cube))
end main
```

`square` and `cube` are specialisations of `power` with `exp` fixed. Each  
is an `Int => Int` function that can be passed directly to `map`. Partial  
application avoids repeating the fixed argument at every call site and  
promotes reuse of parameterised logic without introducing additional  
named functions.  

## Function composition with andThen

`andThen` chains two functions so that the output of the first becomes the  
input of the second, building a left-to-right pipeline.  

```scala
@main def main() =

    val trim:    String => String = _.trim
    val lower:   String => String = _.toLowerCase
    val exclaim: String => String = _ + "!"

    val process = trim.andThen(lower).andThen(exclaim)

    println(process("  Hello World  "))
    println(process("  SCALA 3  "))
end main
```

Each step transforms the string and passes the result to the next function  
in the chain. `andThen` creates a new function value without executing any  
computation until the composed function is called with an actual argument.  
Composition is the functional alternative to nested method calls and  
produces code that reads like a natural description of the data's journey  
through the pipeline.  

## Function composition with compose

`compose` chains two functions in right-to-left order: the second function  
runs first and its result flows into the first.  

```scala
@main def main() =

    val addOne: Int => Int = _ + 1
    val double: Int => Int = _ * 2

    val doubleThenAdd = addOne.compose(double) // double runs first
    val addThenDouble = double.compose(addOne) // addOne runs first

    println(doubleThenAdd(5)) // (5 * 2) + 1 = 11
    println(addThenDouble(5)) // (5 + 1) * 2 = 12
end main
```

`f.compose(g)` produces the mathematical composition f ∘ g, applying `g`  
first and then `f`. The order is the opposite of `andThen`, so  
`f.andThen(g)` equals `g.compose(f)`. Both combinators are available on  
any `Function1` value in Scala, making function composition a built-in  
capability rather than a library add-on.  

---

## Product types

A case class is Scala 3's idiomatic product type: it bundles a fixed set  
of named fields into a single immutable composite value.  

```scala
@main def main() =

    case class Point(x: Double, y: Double)
    case class Circle(center: Point, radius: Double)

    val origin = Point(0.0, 0.0)
    val p      = Point(3.0, 4.0)
    val c      = Circle(p, 5.0)

    println(origin)
    println(p)
    println(c)
    println(c.center.x)
end main
```

Case classes automatically generate `equals`, `hashCode`, `toString`, and  
a companion `apply` factory, making them concise to define and use.  
`Circle` is a product of a `Point` and a `Double`; `Point` is itself a  
product of two `Double`s. Nesting case classes models hierarchical  
entities precisely and readably without boilerplate.  

## Sum types

A Scala 3 `enum` defines a sum type: a value that can be exactly one of  
several named variants.  

```scala
@main def main() =

    enum Direction:
        case North, South, East, West

    enum Shape:
        case Circle(radius: Double)
        case Rectangle(width: Double, height: Double)

    val d = Direction.North
    val s = Shape.Circle(3.0)

    println(d)
    println(s)
end main
```

`Direction` has four simple variants with no data attached. `Shape` carries  
different data depending on the variant: `Circle` holds a radius while  
`Rectangle` holds width and height. This is the essence of a sum type: the  
set of possible values is the union of the values representable by each  
variant. The compiler knows all variants and can check pattern matches  
exhaustively.  

## Nested ADTs

Algebraic data types can be freely nested to describe complex data  
structures with precise, composable types.  

```scala
@main def main() =

    enum Suit:
        case Clubs, Diamonds, Hearts, Spades

    enum Rank:
        case Ace, King, Queen, Jack
        case Number(n: Int)

    case class Card(rank: Rank, suit: Suit)

    val ace   = Card(Rank.Ace, Suit.Spades)
    val two   = Card(Rank.Number(2), Suit.Hearts)
    val queen = Card(Rank.Queen, Suit.Clubs)

    println(ace)
    println(two)
    println(queen)
end main
```

`Rank` mixes four simple variants with one parameterised variant in a  
single `enum`. `Card` is a product of `Rank` and `Suit`, both of which  
are sum types. Nesting enums inside case classes captures the precise set  
of valid deck cards directly in the type system, making invalid  
combinations impossible to construct.  

## Modeling domain logic with ADTs

ADTs let you encode business rules directly into the type system, making  
invalid states unrepresentable at compile time.  

```scala
@main def main() =

    enum OrderStatus:
        case Pending
        case Confirmed(orderId: String)
        case Shipped(trackingCode: String)
        case Cancelled(reason: String)

    case class Order(item: String, status: OrderStatus)

    def describe(o: Order): String =
        o.status match
            case OrderStatus.Pending       => s"${o.item}: awaiting"
            case OrderStatus.Confirmed(id) => s"${o.item}: ok ($id)"
            case OrderStatus.Shipped(t)    => s"${o.item}: sent ($t)"
            case OrderStatus.Cancelled(r)  => s"${o.item}: no ($r)"

    val orders = List(
        Order("Widget", OrderStatus.Pending),
        Order("Gadget", OrderStatus.Confirmed("ORD-42")),
        Order("Tool",   OrderStatus.Shipped("TRACK-99"))
    )

    orders.map(describe).foreach(println)
end main
```

Using an `enum` for `OrderStatus` instead of a plain `String` makes the  
set of possible states part of the type. The pattern match on `o.status`  
is exhaustive: the compiler reports a warning if a new variant is added  
to the enum without updating the match. This encodes state-machine  
transitions as data rather than implicit convention.  

## Copying case classes with copy

The `copy` method creates a modified clone of a case class with selected  
fields changed while all others remain the same.  

```scala
@main def main() =

    case class Config(host: String, port: Int, debug: Boolean)

    val defaults = Config("localhost", 8080, false)
    val devMode  = defaults.copy(debug = true)
    val staging  = defaults.copy(host = "staging.example.com", port = 443)

    println(defaults)
    println(devMode)
    println(staging)
end main
```

`copy` is generated automatically for every case class. It expresses  
immutable updates concisely: rather than constructing the entire value  
from scratch, you state only the fields that change. The original instance  
is never modified, so `defaults` and all derived copies can exist  
simultaneously without interference.  

## Structural equality of case classes

Case classes provide value-based equality: two instances with identical  
field values are considered equal regardless of their memory address.  

```scala
@main def main() =

    case class Money(amount: BigDecimal, currency: String)

    val a = Money(BigDecimal("9.99"), "USD")
    val b = Money(BigDecimal("9.99"), "USD")
    val c = Money(BigDecimal("9.99"), "EUR")

    println(a == b)  // true: same fields
    println(a == c)  // false: different currency
    println(a eq b)  // false: distinct heap objects
end main
```

The compiler generates an `equals` method that compares fields  
structurally rather than by reference identity (`eq`). Two `Money`  
instances with the same `amount` and `currency` are equal even though  
they are different heap objects. This makes case classes safe to use as  
keys in sets and maps and removes the need to override `equals` and  
`hashCode` manually.  

---

## Basic pattern matching

Pattern matching tests a value against a sequence of alternatives and  
executes the first branch whose pattern matches.  

```scala
@main def main() =

    def describe(n: Int): String =
        n match
            case 0          => "zero"
            case 1          => "one"
            case n if n < 0 => s"negative: $n"
            case n          => s"large: $n"

    println(describe(0))
    println(describe(1))
    println(describe(-5))
    println(describe(42))
end main
```

Each `case` clause is a pattern followed by `=>` and an expression.  
Patterns are tested in order; the first match wins. A bare identifier such  
as `n` in the last branch acts as a wildcard that binds the scrutinee to  
that name for use in the body. The guard `if n < 0` adds an extra boolean  
condition to the pattern.  

## Matching on case classes

Pattern matching deconstructs case classes by binding their fields to  
named variables in a single step.  

```scala
@main def main() =

    case class Point(x: Int, y: Int)

    def quadrant(p: Point): String =
        p match
            case Point(0, 0)                    => "origin"
            case Point(x, 0)                    => s"x-axis at $x"
            case Point(0, y)                    => s"y-axis at $y"
            case Point(x, y) if x > 0 && y > 0 => s"Q1 ($x, $y)"
            case Point(x, y)                    => s"other ($x, $y)"

    println(quadrant(Point(0, 0)))
    println(quadrant(Point(3, 0)))
    println(quadrant(Point(5, 7)))
    println(quadrant(Point(-1, 2)))
end main
```

Each case pattern names the constructor followed by sub-patterns for each  
field in order. The match simultaneously tests the structure and binds the  
inner values. Guard conditions can further restrict when a branch applies.  
The compiler also checks that all possible shapes of `Point` are covered.  

## Matching on enums

Pattern matching on a Scala 3 `enum` selects a branch based on the  
variant and extracts any data it carries.  

```scala
@main def main() =

    enum Shape:
        case Circle(radius: Double)
        case Rectangle(width: Double, height: Double)
        case Triangle(base: Double, height: Double)

    def area(s: Shape): Double =
        s match
            case Shape.Circle(r)       => math.Pi * r * r
            case Shape.Rectangle(w, h) => w * h
            case Shape.Triangle(b, h)  => 0.5 * b * h

    println(area(Shape.Circle(3.0)))
    println(area(Shape.Rectangle(4.0, 5.0)))
    println(area(Shape.Triangle(6.0, 8.0)))
end main
```

The compiler verifies that every variant is covered, reporting an error at  
compile time if a case is missing. Adding a new variant to the enum forces  
every match expression to be updated, preventing silent logic gaps. This  
exhaustiveness check is one of the most valuable guarantees that ADTs and  
pattern matching provide together.  

## Nested patterns

Patterns can be nested arbitrarily deep to match the full structure of a  
compound value in a single expression.  

```scala
@main def main() =

    case class Address(city: String, country: String)
    case class Person(name: String, address: Address)

    def greet(p: Person): String =
        p match
            case Person(n, Address(_, "US"))    =>
                s"Hi $n, welcome back!"
            case Person(n, Address(c, "UK"))    =>
                s"Hello $n from $c!"
            case Person(n, Address(c, country)) =>
                s"Greetings $n ($c, $country)"

    println(greet(Person("Alice", Address("NYC", "US"))))
    println(greet(Person("Bob",   Address("London", "UK"))))
    println(greet(Person("Carla", Address("Paris", "FR"))))
end main
```

The inner `Address(_, "US")` pattern is nested inside the outer `Person`  
pattern. The underscore `_` is a wildcard that matches any value without  
binding it. Nested patterns allow complex structural tests in a single  
readable expression without manual field access or chained `if` statements.  

## Match expressions as values

A `match` expression evaluates to a value and can be assigned to a `val`,  
returned from a function, or used inside a larger expression.  

```scala
@main def main() =

    def httpStatus(code: Int): String =
        code match
            case 200 => "OK"
            case 201 => "Created"
            case 400 => "Bad Request"
            case 404 => "Not Found"
            case 500 => "Internal Server Error"
            case _   => "Unknown"

    val codes        = List(200, 201, 404, 500, 418)
    val descriptions = codes.map(httpStatus)

    println(descriptions)
end main
```

`match` is an expression in Scala, not a statement. Its result can be  
mapped over a list, stored in a variable, or passed as a function argument.  
The wildcard `_` is the default branch that catches any value not matched  
by earlier patterns, ensuring the match is total and the compiler does not  
warn about missing cases.  

## Guards in patterns

A guard is a boolean condition attached to a pattern with the `if` keyword;  
the branch fires only when both the pattern and the guard hold.  

```scala
@main def main() =

    def classify(n: Int): String =
        n match
            case n if n < 0      => "negative"
            case 0               => "zero"
            case n if n % 2 == 0 => "positive even"
            case _               => "positive odd"

    List(-3, 0, 4, 7).foreach(n =>
        println(s"$n -> ${classify(n)}"))
end main
```

Guards attach arbitrary boolean logic to a pattern without nesting  
additional `if` expressions inside the branch body. They keep each case  
concise and maintain the linear, top-to-bottom readability that makes  
`match` expressions easy to follow. Pattern plus guard must both succeed  
for the branch to be selected.  

## Destructuring in for-expressions

Pattern matching works inside `for`-expression generators to destructure  
collection elements on the fly.  

```scala
@main def main() =

    case class Product(name: String, price: Double)

    val catalogue = List(
        Product("Widget", 9.99),
        Product("Gadget", 24.50),
        Product("Tool",   14.75)
    )

    val affordable =
        for
            Product(name, price) <- catalogue
            if price < 20.0
        yield s"$name: $$$price"

    affordable.foreach(println)
end main
```

The generator `Product(name, price) <- catalogue` both iterates the list  
and simultaneously destructures each element. The guard `if price < 20.0`  
filters out expensive products. This syntax is concise and avoids the  
explicit `.filter(...).map(...)` chain that a non-destructuring version  
would require.  

---

## The Option type

`Option[A]` represents a value that may or may not be present, avoiding  
`null` and making absence explicit in the type.  

```scala
@main def main() =

    val scores: Map[String, Int] =
        Map("Alice" -> 95, "Bob" -> 82)

    def score(name: String): Option[Int] = scores.get(name)

    println(score("Alice")) // Some(95)
    println(score("Bob"))   // Some(82)
    println(score("Eve"))   // None
end main
```

`Map.get` returns `Option[V]` rather than `V` or `null`, forcing the  
caller to handle both cases. `Some(95)` indicates that the value was  
found; `None` indicates it was absent. Making absence visible in the type  
signature prevents the `NullPointerException`s that plague code relying  
on `null` returns.  

## The Either type

`Either[E, A]` models a computation that may succeed with a value of  
type `A` or fail with a typed error of type `E`.  

```scala
@main def main() =

    def parseInt(s: String): Either[String, Int] =
        s.toIntOption match
            case Some(n) => Right(n)
            case None    => Left(s"'$s' is not a valid integer")

    println(parseInt("42"))
    println(parseInt("abc"))
    println(parseInt("-7"))
end main
```

`Right` conventionally holds the successful result and `Left` holds the  
error. `Either` differs from `Option` in that the error case carries a  
typed value, providing informative failure descriptions. The compiler  
enforces that callers handle both branches, eliminating silent failures.  

## The Try type

`Try[A]` wraps a computation that may throw an exception, producing either  
`Success(value)` or `Failure(throwable)`.  

```scala
@main def main() =

    import scala.util.Try

    def divide(a: Int, b: Int): Try[Int] = Try(a / b)

    println(divide(10, 2))
    println(divide(10, 0))
    println(divide(10, 0).isFailure)
end main
```

`Try(expr)` catches any exception thrown by `expr` and wraps it in  
`Failure`. If `expr` evaluates without throwing, the result is  
`Success(value)`. `Try` bridges the gap between exception-based Java APIs  
and functional Scala code, letting you handle errors as values without  
wrapping every call in a try-catch block.  

## Mapping over Option

`map` applies a function to the value inside a `Some`, leaving `None`  
unchanged, enabling transformations without explicit null checks.  

```scala
@main def main() =

    val maybeAge: Option[Int] = Some(25)
    val noAge: Option[Int]    = None

    val ageInDays = maybeAge.map(_ * 365)
    val noResult  = noAge.map(_ * 365)

    println(ageInDays)
    println(noResult)
    println(maybeAge.map(a => s"Age: $a").getOrElse("Unknown"))
end main
```

`map` transforms `Some(25)` into `Some(9125)` and passes `None` through  
untouched. This lets you chain transformations without ever writing  
`if (opt != null)` or pattern-matching manually. `getOrElse` at the end  
provides a fallback value when the option is `None`, completing safe  
handling in a single readable expression.  

## FlatMapping over Option

`flatMap` sequences two optional computations, short-circuiting to `None`  
as soon as any step returns `None`.  

```scala
@main def main() =

    val users:  Map[String, Int]    = Map("alice" -> 1, "bob" -> 2)
    val emails: Map[Int, String]    = Map(1 -> "alice@example.com")

    def userId(name: String): Option[Int]    = users.get(name)
    def email(id: Int):       Option[String] = emails.get(id)

    println(userId("alice").flatMap(email))
    println(userId("bob").flatMap(email))
    println(userId("eve").flatMap(email))
end main
```

Without `flatMap`, calling `map` with `email` would produce  
`Option[Option[String]]`. `flatMap` flattens this automatically, giving a  
single `Option[String]`. `userId("bob")` returns `Some(2)`, but there is  
no email for id 2, so the overall result is `None`. This same mechanic  
underlies for-comprehensions over `Option`.  

## Mapping over Either

`map` on `Either` transforms the `Right` value with a function, leaving a  
`Left` unchanged.  

```scala
@main def main() =

    def parseDouble(s: String): Either[String, Double] =
        s.toDoubleOption.toRight(s"Not a number: '$s'")

    val result = parseDouble("3.14").map(_ * 2)
    val failed = parseDouble("xyz").map(_ * 2)

    println(result) // Right(6.28)
    println(failed) // Left(Not a number: 'xyz')
end main
```

`map` on a `Right` value applies the function and wraps the result in a  
new `Right`. On a `Left` value, `map` is a no-op that preserves the error.  
`toRight` converts an `Option` to an `Either` by providing the error value  
for the `Left` case, which is a common pattern when bridging  
Option-returning APIs into an `Either`-based pipeline.  

## FlatMapping over Either

`flatMap` chains two `Either`-returning computations, stopping at the  
first `Left`.  

```scala
@main def main() =

    def nonEmpty(s: String): Either[String, String] =
        if s.nonEmpty then Right(s)
        else Left("String must not be empty")

    def parseInt(s: String): Either[String, Int] =
        s.toIntOption.toRight(s"Not an integer: '$s'")

    println(nonEmpty("42").flatMap(parseInt))
    println(nonEmpty("").flatMap(parseInt))
    println(nonEmpty("abc").flatMap(parseInt))
end main
```

`flatMap` passes the `Right` value from `nonEmpty` directly into  
`parseInt`, chaining the two validations. As soon as any step returns a  
`Left`, the rest of the chain is skipped. This makes error propagation  
automatic and keeps the happy path free of explicit error-checking  
boilerplate.  

## Mapping and recovering from Try

`map` transforms a successful `Try` result; `recover` converts a specific  
failure into a fallback success value.  

```scala
@main def main() =

    import scala.util.Try

    def parse(s: String): Try[Int] = Try(s.toInt)

    val good = parse("42").map(_ * 2)
    val bad  = parse("oops").recover:
        case _: NumberFormatException => -1

    println(good) // Success(84)
    println(bad)  // Success(-1)
end main
```

`map` on `Success(42)` produces `Success(84)`. On a `Failure`, `map` is  
skipped and the `Failure` propagates unchanged. `recover` intercepts  
specific exception types and converts a `Failure` back into a `Success`  
with a default value, providing a concise way to handle known failure  
modes without abandoning the functional pipeline.  

## For-comprehensions with Option

A for-comprehension over `Option` sequences multiple optional lookups,  
producing `None` as soon as any step yields `None`.  

```scala
@main def main() =

    val people:   Map[String, String] =
        Map("alice" -> "engineer", "bob" -> "designer")
    val salaries: Map[String, Int] =
        Map("engineer" -> 95000, "designer" -> 80000)

    def lookup(name: String): Option[Int] =
        for
            role   <- people.get(name)
            salary <- salaries.get(role)
        yield salary

    println(lookup("alice"))
    println(lookup("bob"))
    println(lookup("eve"))
end main
```

The for-comprehension desugars to nested `flatMap` and `map` calls. If  
`people.get(name)` is `None`, the whole expression short-circuits to  
`None` without evaluating the salary lookup. This pattern makes multi-step  
optional computations read like straightforward sequential code without  
manual null-check nesting.  

## For-comprehensions with Either

A for-comprehension over `Either` propagates the first `Left` error  
through the entire computation automatically.  

```scala
@main def main() =

    def nonBlank(s: String): Either[String, String] =
        if s.isBlank then Left("blank input") else Right(s.trim)

    def toInt(s: String): Either[String, Int] =
        s.toIntOption.toRight(s"not an integer: $s")

    def positive(n: Int): Either[String, Int] =
        if n > 0 then Right(n) else Left(s"must be positive: $n")

    def parse(input: String): Either[String, Int] =
        for
            trimmed <- nonBlank(input)
            n       <- toInt(trimmed)
            result  <- positive(n)
        yield result

    println(parse("  42  "))
    println(parse("  "))
    println(parse("abc"))
    println(parse("-5"))
end main
```

Each step in the for-comprehension can fail independently. The first `Left`  
result is returned immediately and subsequent steps are never evaluated.  
The final `yield` expression is automatically wrapped in `Right` when all  
steps succeed, making the error handling invisible in the happy path.  

## Chaining Try operations

Multiple `Try`-returning operations can be chained with a  
for-comprehension to build a safe computation pipeline.  

```scala
@main def main() =

    import scala.util.Try

    def readInt(s: String): Try[Int] = Try(s.toInt)

    def safeSqrt(n: Int): Try[Double] =
        if n < 0 then
            Try(throw IllegalArgumentException("negative input"))
        else Try(math.sqrt(n))

    val result = for
        n  <- readInt("16")
        sq <- safeSqrt(n)
    yield sq

    val failed = for
        n  <- readInt("abc")
        sq <- safeSqrt(n)
    yield sq

    println(result)
    println(failed)
end main
```

The for-comprehension desugars into `readInt("16").flatMap(safeSqrt)`. If  
`readInt` fails, `flatMap` skips `safeSqrt` entirely and propagates the  
`Failure`. This allows safe composition of exception-prone operations  
without try-catch blocks scattered through the code.  

## Modeling errors with ADTs

A sealed hierarchy expresses all known error variants precisely, providing  
richer domain error information than raw strings.  

```scala
@main def main() =

    enum ValidationError:
        case EmptyField(name: String)
        case TooShort(name: String, min: Int)
        case InvalidFormat(name: String, msg: String)

    def validateUsername(s: String): Either[ValidationError, String] =
        if s.isEmpty then
            Left(ValidationError.EmptyField("username"))
        else if s.length < 3 then
            Left(ValidationError.TooShort("username", 3))
        else
            Right(s)

    println(validateUsername(""))
    println(validateUsername("ab"))
    println(validateUsername("alice"))
end main
```

Using an ADT for errors makes the set of possible failures part of the  
type. Callers can pattern match on `ValidationError` exhaustively,  
receiving a compile-time warning if a new variant is added later. This is  
far more robust than comparing error message strings at runtime.  

---

## Immutable List

`List` is the fundamental immutable singly-linked sequence, ideal for  
head-first traversal and recursive algorithms.  

```scala
@main def main() =

    val fruits = List("apple", "banana", "cherry")
    val more   = "mango" :: fruits
    val nums   = List.range(1, 6)

    println(fruits)
    println(more)
    println(fruits.head)
    println(fruits.tail)
    println(nums.sum)
end main
```

`::` prepends an element in O(1) time by creating a single new cons cell.  
`List.range(1, 6)` produces `List(1, 2, 3, 4, 5)`. `head` and `tail`  
access the first element and the remainder in constant time. `sum` is a  
convenience method that folds the list with addition, demonstrating that  
common aggregations are built directly into the standard library.  

## Vector

`Vector` is an immutable indexed sequence backed by a shallow tree,  
offering effectively O(1) random access and functional updates.  

```scala
@main def main() =

    val v        = Vector(10, 20, 30, 40, 50)
    val updated  = v.updated(2, 99)
    val appended = v :+ 60

    println(v)
    println(updated)
    println(appended)
    println(v(3))
end main
```

`updated` returns a new `Vector` with one element changed without copying  
the whole structure; unchanged subtrees are shared. The `:+` operator  
appends an element in effectively O(1) time, making `Vector` a better  
choice than `List` when elements are frequently accessed by index or  
appended at the end.  

## LazyList

`LazyList` is a lazily evaluated sequence: elements are computed only when  
demanded, enabling potentially infinite streams.  

```scala
@main def main() =

    val naturals: LazyList[Int] = LazyList.from(1)
    val evens = naturals.filter(_ % 2 == 0)

    println(naturals.take(5).toList)
    println(evens.take(5).toList)

    val fibs: LazyList[BigInt] =
        def go(a: BigInt, b: BigInt): LazyList[BigInt] =
            a #:: go(b, a + b)
        go(0, 1)

    println(fibs.take(10).toList)
end main
```

`LazyList.from(1)` produces a conceptually infinite sequence starting at  
1; no element beyond what is actually requested is computed. `#::` is the  
lazy cons operator: the tail is a by-name parameter evaluated only when  
accessed. `take(10)` materialises the first ten Fibonacci numbers without  
any danger of an infinite loop.  

## Set

`Set` is an immutable collection of unique elements with efficient  
membership testing.  

```scala
@main def main() =

    val a = Set(1, 2, 3, 4)
    val b = Set(3, 4, 5, 6)

    println(a union b)
    println(a intersect b)
    println(a diff b)
    println(a.contains(3))
    println(a + 5)
    println(a - 2)
end main
```

Set operations — `union`, `intersect`, and `diff` — all return new  
immutable `Set` instances. `+` and `-` produce sets with one element added  
or removed without modifying the original. The default implementation uses  
a hash-array-mapped trie, giving O(log n) amortised complexity for most  
operations.  

## Map

`Map` is an immutable key-value store that associates each unique key with  
a single value.  

```scala
@main def main() =

    val scores  = Map("Alice" -> 90, "Bob" -> 75, "Carol" -> 88)
    val updated = scores + ("Dave" -> 95)
    val removed = scores - "Bob"

    println(scores.get("Alice"))
    println(scores.getOrElse("Eve", 0))
    println(updated)
    println(removed)
end main
```

`+` on a `Map` returns a new map with the additional entry; `-` returns  
one without the given key. `get` returns `Option[V]` making missing-key  
access explicit. `getOrElse` provides a safe default when the key is  
absent, avoiding any risk of a `NoSuchElementException`.  

## Functional transformations

`map` on a `Map` transforms values while producing a new immutable map  
without modifying the original.  

```scala
@main def main() =

    val prices = Map("apple" -> 1.5, "banana" -> 0.8, "cherry" -> 3.0)

    val doubled = prices.view.mapValues(_ * 2).toMap
    val labels  = prices.map((k, v) => s"$k: $$$v")

    println(doubled)
    println(labels)
end main
```

`mapValues` transforms each value while keeping the keys unchanged. `map`  
with a tuple lambda transforms both key and value, producing a  
`List[String]` here because the function no longer returns a pair. `view`  
defers evaluation and `toMap` materialises the result, avoiding  
intermediate collection allocations.  

## Filtering collections

`filter` and `filterNot` select elements from any collection type by  
applying a predicate, producing a new collection of the same type.  

```scala
@main def main() =

    val nums  = Vector(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val evens = nums.filter(_ % 2 == 0)
    val odds  = nums.filterNot(_ % 2 == 0)

    val words = Set("apple", "ant", "banana", "avocado", "cherry")
    val aWords = words.filter(_.startsWith("a"))

    println(evens)
    println(odds)
    println(aWords)
end main
```

`filter` works uniformly across `Vector`, `List`, `Set`, `Map`, and other  
collection types, always returning a new instance of the same type. Using  
`filterNot` instead of `filter(! predicate)` makes the intent clearer at  
the call site. Both operations are purely functional: the original  
collections are never modified.  

## Grouping with groupBy

`groupBy` partitions a collection into a `Map` of groups keyed by the  
result of a classifier function.  

```scala
@main def main() =

    val words = List("ant", "bear", "cat", "alligator", "bee", "crane")
    val byFirstLetter = words.groupBy(_.head)

    byFirstLetter.toList.sorted.foreach: (k, vs) =>
        println(s"$k -> $vs")
end main
```

`groupBy` applies `_.head` to each word and collects words with the same  
first character under the same key. The resulting `Map[Char, List[String]]`  
is built in a single pass. Sorting the entries before printing them gives  
deterministic output regardless of the underlying hash-map ordering.  

## Partitioning with partition

`partition` splits a collection into two sub-collections based on a  
predicate, returning both halves simultaneously as a tuple.  

```scala
@main def main() =

    val nums = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val (evens, odds) = nums.partition(_ % 2 == 0)

    val words = List("hi", "hello", "hey", "world", "scala")
    val (short, rest) = words.partition(_.length <= 3)

    println(evens)
    println(odds)
    println(short)
    println(rest)
end main
```

`partition` traverses the collection once and returns both halves as a  
tuple, which is destructured with `val (a, b)` on the left-hand side.  
This is more efficient than calling `filter` twice with complementary  
predicates, which would traverse the collection twice.  

## Collecting with collect

`collect` applies a partial function to a collection, transforming and  
filtering simultaneously.  

```scala
@main def main() =

    val mixed: List[Any] = List(1, "hello", 2, true, 3, "world")

    val ints    = mixed.collect { case n: Int    => n * 10 }
    val strings = mixed.collect { case s: String => s.toUpperCase }

    println(ints)
    println(strings)
end main
```

`collect` applies the partial function only to elements for which it is  
defined and silently skips the rest. Combining a type-test pattern with a  
transformation inside a single `collect` call is cleaner than a `filter`  
followed by a `map`. The result preserves the relative order of elements  
for which the partial function was defined.  

## Sorting functionally

`sortBy` and `sortWith` produce sorted copies of a collection without  
modifying the original.  

```scala
@main def main() =

    case class Student(name: String, grade: Int)

    val students = List(
        Student("Charlie", 82),
        Student("Alice", 95),
        Student("Bob", 75)
    )

    val byGrade  = students.sortBy(_.grade)
    val byName   = students.sortBy(_.name)
    val topFirst = students.sortWith((a, b) => a.grade > b.grade)

    println(byGrade)
    println(byName)
    println(topFirst)
end main
```

`sortBy` extracts a key from each element and sorts by its natural  
ordering. `sortWith` accepts a custom binary comparator function. Both  
return a new `List` without altering the original. Case classes whose  
fields implement `Ordering` can be sorted without additional boilerplate.  

## Zipping collections

`zip` pairs up elements from two collections by position, producing a  
list of tuples.  

```scala
@main def main() =

    val names  = List("Alice", "Bob", "Carol")
    val scores = List(90, 85, 92)

    val paired = names.zip(scores)
    val ranked = names.zipWithIndex

    println(paired)
    println(ranked)

    paired.foreach: (name, score) =>
        println(s"$name scored $score")
end main
```

`zip` stops at the shorter of the two collections. `zipWithIndex` pairs  
each element with its zero-based index, which is useful for enumeration  
without a mutable counter. Destructuring the tuple in the `foreach` body  
keeps the code readable by giving meaningful names to each component.  

## FlatMapping collections

`flatMap` on a collection applies a function that returns a collection to  
each element and concatenates all the results.  

```scala
@main def main() =

    val sentences = List("the quick fox", "jumps over")
    val words     = sentences.flatMap(_.split(" ").toList)

    val table = List(1, 2, 3).flatMap: r =>
        List(1, 2, 3).map(c => r * c)

    println(words)
    println(table)
end main
```

`flatMap` is equivalent to calling `map` followed by `flatten`. Using it  
on `sentences` produces a flat `List[String]` of every word. The  
multiplication table nests two lists through `flatMap` and `map`,  
demonstrating that for-comprehensions with multiple generators are  
syntactic sugar for exactly this pattern.  

## Folding collections

`foldLeft` reduces a collection to a single derived value by repeatedly  
combining each element with an accumulator.  

```scala
@main def main() =

    val words = List("functional", "programming", "in", "scala")

    val longest = words.foldLeft(""): (acc, w) =>
        if w.length > acc.length then w else acc

    val histogram = words.foldLeft(Map.empty[Char, Int]): (acc, w) =>
        w.foldLeft(acc): (m, c) =>
            m + (c -> (m.getOrElse(c, 0) + 1))

    println(s"Longest word  : $longest")
    println(s"Letter counts : $histogram")
end main
```

The first fold finds the longest word by comparing each word against the  
running longest. The second builds a character-frequency map by folding  
over words and then over each word's characters. Both folds use immutable  
accumulators and produce a single derived value without any mutable  
variable.  

## A functional pipeline

A functional pipeline chains multiple higher-order operations on a  
collection, expressing a complex transformation as a readable sequence of  
steps.  

```scala
@main def main() =

    case class Sale(product: String, units: Int, price: Double)

    val sales = List(
        Sale("Widget", 10, 9.99),
        Sale("Gadget",  3, 24.50),
        Sale("Widget",  5, 9.99),
        Sale("Tool",    8, 14.75)
    )

    val report = sales
        .groupBy(_.product)
        .view
        .mapValues(ss => ss.map(s => s.units * s.price).sum)
        .toList
        .sortBy(-_._2)
        .map((product, total) => f"$product%-10s $$$total%.2f")

    report.foreach(println)
end main
```

The pipeline groups sales by product, sums the revenue per group, converts  
the map view to a list, sorts by descending revenue, and formats each row.  
Every step returns an immutable collection; no variable is mutated at any  
point. This style makes each transformation visible and independently  
testable.  
