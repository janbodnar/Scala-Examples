# Lambdas

A lambda, also called an anonymous function or function literal, is a  
function defined without a name. In Scala 3 a lambda is a first-class  
value: it can be stored in a variable, passed to another function, or  
returned from a function, just like an `Int` or a `String`. The syntax  
for a lambda is a parameter list followed by `=>` and a body expression:  
`(x: Int) => x * 2`. When the type can be inferred from context, the  
annotation may be omitted: `x => x * 2`. For single-parameter lambdas  
Scala also provides the underscore shorthand: `_ * 2`.  

**Lambdas versus named functions and methods**  

A named `def` function is compiled to a JVM method and is resolved  
statically at the call site. A lambda is compiled to an instance of a  
`FunctionN` trait (`Function0`, `Function1`, …, `Function22`) and is  
treated as a heap-allocated object. Because of this, passing a `def` to  
a higher-order function that expects a `Function1[A, B]` automatically  
triggers *eta-expansion*: the compiler wraps the method in a function  
object. In most cases the difference is transparent, but it is useful to  
know that lambdas carry a small allocation cost that `def` calls do not.  

**Syntax variations**  

Scala 3 supports several equivalent ways to write a lambda:  

- Full form with type: `(x: Int, y: Int) => x + y`  
- Inferred types: `(x, y) => x + y` (inside a typed context)  
- Underscore syntax: `_ + _` (each underscore is a distinct parameter)  
- Block body: `x => { val doubled = x * 2; doubled + 1 }`  
- Indentation body (Scala 3): a lambda body may span indented lines  

**Higher-order functions**  

A higher-order function is one that accepts a function as a parameter or  
returns a function as its result. The standard library methods `map`,  
`filter`, `foldLeft`, `flatMap`, and many others are higher-order  
functions that accept lambdas. Lambdas make it possible to pass custom  
behaviour to a generic algorithm without defining a named class or  
object.  

**Closures**  

A lambda that references a variable defined in the enclosing scope is  
called a closure. The lambda *closes over* that variable: it carries a  
reference to it and can read (or, if the variable is a `var`, write) its  
value even after the enclosing scope has exited.  

**Performance considerations**  

Each lambda that closes over mutable state creates a new object on each  
invocation of the enclosing scope. For hot paths, consider caching  
lambdas in `val` definitions or replacing them with named methods.  
Scala's `@inline` and `inline def` features can eliminate the lambda  
allocation entirely in some cases. For most application code, however,  
the convenience of lambdas far outweighs any micro-optimisation concern.  

---

## A simple lambda

The most basic lambda takes one argument and returns an expression.  

```scala
@main def main() =

  val double = (x: Int) => x * 2

  println(double(3))
  println(double(7))
end main
```

The lambda `(x: Int) => x * 2` is assigned to the value `double`. The  
parameter `x` is declared with an explicit `Int` type annotation. The  
body `x * 2` is a single expression; no `return` keyword is needed. When  
`double` is called with `(3)`, the body evaluates to `6` and `println`  
prints it.  

## Lambdas with multiple parameters

A lambda may declare any number of comma-separated parameters.  

```scala
@main def main() =

  val add = (x: Int, y: Int) => x + y
  val greet = (name: String, greeting: String) => s"$greeting, $name!"

  println(add(3, 4))
  println(greet("Alice", "Hello"))
end main
```

Multiple parameters are listed inside parentheses, separated by commas.  
The lambda `(x: Int, y: Int) => x + y` stores the sum of two integers.  
A second lambda builds a greeting string with string interpolation.  
Both lambdas are invoked just like named functions by appending an  
argument list.  

## Explicitly typed lambdas

Providing the full type signature on the variable makes the lambda  
contract visible at the definition site.  

```scala
@main def main() =

  val multiply: (Int, Int) => Int = (x, y) => x * y
  val shout: String => String = s => s.toUpperCase + "!"

  println(multiply(6, 7))
  println(shout("hello"))
end main
```

When the variable carries a type annotation such as `(Int, Int) => Int`,  
the compiler already knows the parameter types, so the lambda on the  
right-hand side can omit them: `(x, y) => x * y`. The `String => String`  
annotation similarly lets `s => s.toUpperCase + "!"` be written without  
annotating `s`. Explicit type annotations improve readability and serve  
as self-contained documentation.  

## Underscore syntax

The underscore `_` is a placeholder for a single positional parameter.  

```scala
@main def main() =

  val square: Int => Int = _ * _  // each _ is a separate parameter
  val isEven: Int => Boolean = _ % 2 == 0
  val negate: Int => Int = -_

  val nums = List(1, 2, 3, 4, 5)

  println(nums.map(_ * 3))
  println(nums.filter(isEven))
  println(nums.map(negate))
end main
```

Each underscore stands for a distinct, positionally ordered parameter.  
`_ * _` is equivalent to `(a, b) => a * b` and requires a two-parameter  
context. `_ % 2 == 0` is a one-parameter predicate. The placeholder  
syntax is concise but should be avoided when the intent would be unclear  
to the reader. Assigning the lambda to a named `val` with a type  
annotation, as shown here, keeps the meaning explicit.  

## Lambdas with map

`map` transforms every element of a collection by applying a lambda.  

```scala
@main def main() =

  val words = List("scala", "is", "fun")
  val upper = words.map(w => w.toUpperCase)
  val lengths = words.map(_.length)

  println(upper)
  println(lengths)
end main
```

`map` is a higher-order method that accepts a `A => B` lambda and  
produces a new list where each element is the result of applying that  
lambda. The first lambda `w => w.toUpperCase` converts every word to  
upper case. The second uses the underscore shorthand `_.length` to  
extract each string's length. Neither call modifies the original list;  
both return freshly allocated lists.  

## Lambdas with filter

`filter` keeps only the elements for which a predicate lambda returns  
`true`.  

```scala
@main def main() =

  val nums = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
  val evens = nums.filter(_ % 2 == 0)
  val largeOdds = nums.filter(n => n % 2 != 0 && n > 5)

  println(evens)
  println(largeOdds)
end main
```

The predicate `_ % 2 == 0` is a lambda of type `Int => Boolean`. Every  
element for which it returns `true` is included in the result list;  
others are discarded. The second predicate combines two conditions with  
`&&`. Because `filter` returns a new list, the original `nums` is  
unchanged and can be filtered again with a different predicate.  

## Lambdas with fold

`foldLeft` reduces a collection to a single value by repeatedly  
applying a combining lambda.  

```scala
@main def main() =

  val nums = List(1, 2, 3, 4, 5)
  val sum = nums.foldLeft(0)((acc, n) => acc + n)
  val product = nums.foldLeft(1)(_ * _)
  val largest = nums.foldLeft(Int.MinValue)((acc, n) => acc max n)

  println(s"sum     = $sum")
  println(s"product = $product")
  println(s"largest = $largest")
end main
```

`foldLeft(seed)(f)` starts with `seed` and folds the list from left to  
right: at each step the accumulator `acc` and the current element `n`  
are combined by the lambda `f`, producing the new accumulator. The sum  
example accumulates addition; the product example uses `_ * _`; the  
third finds the maximum element by picking the larger of the accumulator  
and each element. `foldLeft` is the fundamental building block from  
which many other collection operations can be derived.  

## Storing lambdas in variables

Lambdas are values and can be freely stored and passed around.  

```scala
@main def main() =

  val ops: List[(Int, Int) => Int] = List(
    _ + _,
    _ - _,
    _ * _
  )

  for op <- ops do
    println(op(10, 3))
end main
```

A `List` can hold lambdas just as it holds integers or strings. Here  
each lambda has the same type `(Int, Int) => Int`, so they can all live  
in one list. The `for` loop iterates over the list and invokes each  
operation with the arguments `10` and `3`, printing `13`, `7`, and `30`  
respectively. Storing lambdas in collections is useful for building  
configurable pipelines or dispatch tables.  

## Lambdas returning lambdas

A lambda can return another lambda, creating a higher-order function  
factory.  

```scala
@main def main() =

  val multiplier: Int => (Int => Int) = factor => (x => x * factor)

  val triple = multiplier(3)
  val times10 = multiplier(10)

  println(triple(5))
  println(times10(7))
end main
```

`multiplier` is a lambda that takes an `Int` called `factor` and returns  
a new lambda `x => x * factor`. This inner lambda closes over `factor`,  
capturing its value at the time `multiplier` is called. Calling  
`multiplier(3)` returns a lambda that always multiplies its argument by  
`3`. This pattern is the basis of currying and partial application.  

## Closures

A lambda that reads a variable from the enclosing scope forms a closure.  

```scala
@main def main() =

  var callCount = 0

  val increment = () =>
    callCount += 1
    callCount

  println(increment())
  println(increment())
  println(increment())
  println(s"Total calls: $callCount")
end main
```

The lambda `increment` captures the `var` `callCount` from the  
enclosing scope. Each invocation increments the shared variable and  
returns its current value. Because `callCount` is a `var`, the lambda  
can both read and write it. After three calls the variable holds `3`,  
which is confirmed by the final `println`. Closures over mutable state  
must be used carefully in concurrent programs to avoid data races.  

## Lambdas in for-comprehensions

A for-comprehension desugars into calls to `map` and `flatMap`, each  
of which accepts a lambda.  

```scala
@main def main() =

  val pairs =
    for
      x <- List(1, 2, 3)
      y <- List(10, 20)
    yield (x, y)

  println(pairs)

  val doubled =
    for x <- List(1, 2, 3, 4, 5) yield x * 2

  println(doubled)
end main
```

The Scala compiler rewrites `for x <- xs yield f(x)` as `xs.map(x =>  
f(x))`. A nested generator, as in the `pairs` example, desugars into a  
`flatMap` followed by a `map`. The lambdas are written using the  
comprehension syntax rather than explicitly, which often reads more  
naturally when several generators or guards are involved.  

## Lambdas with pattern matching

A lambda body can contain a `match` expression for structural  
decomposition.  

```scala
@main def main() =

  val describe: Int => String = n =>
    n match
      case 0 => "zero"
      case n if n < 0 => "negative"
      case n if n % 2 == 0 => "positive even"
      case _ => "positive odd"

  List(-3, 0, 4, 7).map(describe).foreach(println)
end main
```

The lambda assigned to `describe` receives an `Int` and dispatches on  
it with a `match` expression. Guards (`if n < 0`) add conditions beyond  
the structural pattern. The wildcard `_` catches all remaining cases.  
The lambda is then passed to `map`, producing a list of strings that  
`foreach` prints one per line. Combining lambdas with `match` avoids  
the need to define a separate named function for simple dispatch logic.  

## Lambdas with tuples

Lambdas can destructure tuples directly in the parameter list.  

```scala
@main def main() =

  val pairs = List((1, "one"), (2, "two"), (3, "three"))

  val formatted = pairs.map((n, word) => s"$n -> $word")
  val sums = List((1, 2), (3, 4), (5, 6)).map(_ + _)

  println(formatted)
  println(sums)
end main
```

Scala 3 allows a lambda parameter list to destructure a tuple directly:  
`(n, word) => …` works when the expected type is a two-element tuple.  
This avoids the need to write `t => s"${t._1} -> ${t._2}"`. The second  
example uses `_ + _` which adds the two components of each `(Int, Int)`  
tuple. Tuple destructuring in lambdas keeps code readable when working  
with paired or structured data.  

## Lambdas with case classes

Case class fields can be accessed inside a lambda, or the instance can  
be destructured using a partial-function style.  

```scala
@main def main() =

  case class Point(x: Double, y: Double)

  val points = List(Point(1, 2), Point(3, 4), Point(-1, 5))
  val distances = points.map(p => math.sqrt(p.x * p.x + p.y * p.y))
  val positiveX = points.filter(_.x > 0)

  distances.foreach(d => println(f"$d%.2f"))
  println(positiveX)
end main
```

The lambda `p => math.sqrt(p.x * p.x + p.y * p.y)` accesses the fields  
of the `Point` case class through dot notation. The filter predicate  
`_.x > 0` uses the underscore shorthand: the single implicit parameter  
is the `Point` being tested, and `.x` extracts its `x` field. Case  
classes are ideal companions to lambdas because their fields are public  
and immutable by default.  

## Lambdas with generics

Lambdas can work with generic type parameters, enabling reusable  
higher-order utilities.  

```scala
@main def main() =

  def applyTwice[A](f: A => A, value: A): A = f(f(value))

  val doubleIt: Int => Int = _ * 2
  val shout: String => String = s => s.toUpperCase + "!"

  println(applyTwice(doubleIt, 3))
  println(applyTwice(shout, "hello"))
end main
```

`applyTwice` is parameterised over `A`. It accepts a lambda `f: A => A`  
and applies it to `value` twice. When called with `doubleIt` and `3`,  
the type parameter `A` is inferred as `Int`, yielding `3 * 2 * 2 = 12`.  
When called with `shout` and `"hello"`, `A` is `String`. Generic  
higher-order functions decouple the algorithm from the element type and  
work uniformly for any type that satisfies the constraint.  

## Lambdas with Option

`Option` has `map`, `flatMap`, and `filter` that accept lambdas, making  
it easy to transform optional values without null checks.  

```scala
@main def main() =

  val maybeNum: Option[Int] = Some(42)
  val noNum: Option[Int] = None

  val doubled = maybeNum.map(_ * 2)
  val filtered = maybeNum.filter(_ > 100)
  val chained = maybeNum
    .map(_ + 8)
    .filter(_ % 2 == 0)
    .map(n => s"Result: $n")

  println(doubled)
  println(filtered)
  println(chained)
  println(noNum.map(_ * 2))
end main
```

`Option.map` applies a lambda only when the option is `Some`; on `None`  
it returns `None` without invoking the lambda. `filter` turns a `Some`  
into `None` if the predicate returns `false`. Chaining these operations  
creates a safe pipeline over values that may be absent. The final line  
demonstrates that applying `map` to `None` is always a no-op.  

## Partial application with lambdas

Partial application fixes some arguments of a function, producing a  
specialised lambda.  

```scala
@main def main() =

  def power(base: Int, exp: Int): Int =
    math.pow(base, exp).toInt

  val square = (n: Int) => power(n, 2)
  val cube   = (n: Int) => power(n, 3)

  val nums = List(1, 2, 3, 4, 5)

  println(nums.map(square))
  println(nums.map(cube))
end main
```

`square` and `cube` are lambdas that call `power` with the second  
argument (`exp`) already fixed to `2` or `3`. This is manual partial  
application: a lambda wraps the original function and supplies the  
known argument. The resulting lambdas have type `Int => Int` and can be  
passed directly to `map`. Partial application is a key technique for  
adapting multi-parameter functions to higher-order contexts.  

## Currying

A curried function takes its arguments one at a time and returns a  
lambda after each step.  

```scala
@main def main() =

  val add: Int => Int => Int = x => y => x + y

  val add5 = add(5)
  val add10 = add(10)

  println(add5(3))
  println(add10(7))
  println(List(1, 2, 3).map(add(100)))
end main
```

`add` is a lambda whose body is itself a lambda. The type  
`Int => Int => Int` reads as "a function from `Int` to a function from  
`Int` to `Int`". Calling `add(5)` returns the inner lambda  
`y => 5 + y`, which is stored in `add5`. This technique is called  
currying, after the mathematician Haskell Curry. Curried functions  
integrate naturally with `map` and other higher-order methods, as shown  
in the last `println`.  

## Function composition

`andThen` and `compose` combine lambdas into a new lambda without  
explicit nesting.  

```scala
@main def main() =

  val trim: String => String = _.trim
  val lower: String => String = _.toLowerCase
  val addBang: String => String = _ + "!"

  val process = trim andThen lower andThen addBang

  val inputs = List("  Hello  ", " WORLD ", "  Scala 3 ")
  println(inputs.map(process))
end main
```

`andThen` wires two `Function1` values so that the output of the first  
becomes the input of the second. `trim andThen lower andThen addBang`  
creates a pipeline that first strips whitespace, then lower-cases, then  
appends an exclamation mark. The resulting `process` lambda has the same  
type `String => String` as each individual step. `compose` works in  
reverse order: `f compose g` applies `g` first, then `f`.  

## Collection pipelines

Chaining several higher-order methods builds a declarative processing  
pipeline.  

```scala
@main def main() =

  val words = List(
    "apple", "banana", "cherry", "avocado",
    "blueberry", "apricot", "cranberry"
  )

  val result = words
    .filter(_.startsWith("a"))
    .map(_.capitalize)
    .sortBy(_.length)
    .take(3)

  println(result)
end main
```

Each step of the pipeline accepts a lambda and returns a new collection,  
leaving the original `words` unchanged. `filter` selects words starting  
with `"a"`, `map` capitalises them, `sortBy` orders them by length, and  
`take` keeps the three shortest. Reading the pipeline from top to bottom  
describes the transformation in natural language. This style of  
declarative data processing is idiomatic in Scala and avoids mutable  
accumulators entirely.  

## Lambdas in domain modeling

Lambdas can encode business rules as first-class values, making  
validation logic composable.  

```scala
@main def main() =

  case class User(name: String, age: Int, email: String)

  type Rule[A] = A => Option[String]

  def validate[A](value: A, rules: List[Rule[A]]): List[String] =
    rules.flatMap(_(value))

  val notEmpty: Rule[String] = s =>
    if s.trim.isEmpty then Some("must not be empty") else None

  val adultAge: Rule[User] = u =>
    if u.age < 18 then Some("must be at least 18") else None

  val validEmail: Rule[User] = u =>
    if !u.email.contains("@") then Some("invalid email") else None

  val user = User("", 16, "not-an-email")
  val errors = validate(user, List(adultAge, validEmail))

  println(errors)
end main
```

Each validation rule is a lambda of type `A => Option[String]` that  
returns `None` on success or `Some(message)` on failure. `validate`  
flattens the results of all rules into a list of error messages.  
Representing rules as lambdas makes them easy to compose, reorder, and  
test in isolation. New rules can be added without modifying existing  
code, following the open-closed principle.  

## Lambdas as event handlers

Lambdas provide a concise way to register callbacks and event handlers.  

```scala
@main def main() =

  type Handler[E] = E => Unit

  class EventBus[E]:
    private var handlers: List[Handler[E]] = List.empty

    def subscribe(h: Handler[E]): Unit =
      handlers = handlers :+ h

    def publish(event: E): Unit =
      handlers.foreach(_(event))

  val bus = EventBus[String]()

  bus.subscribe(e => println(s"Logger: $e"))
  bus.subscribe(e => println(s"Auditor: received '$e'"))

  bus.publish("UserLoggedIn")
  bus.publish("ItemPurchased")
end main
```

`EventBus` stores a list of lambdas, each of type `String => Unit`. The  
`subscribe` method appends a new handler lambda; `publish` invokes every  
registered handler with the event. Using lambdas as handlers eliminates  
the need for a separate listener interface or abstract class. The caller  
supplies the behaviour inline, keeping related logic together.  

## Lambdas in state machines

A state machine can be modelled as a map from states to transition  
lambdas.  

```scala
@main def main() =

  enum State:
    case Idle, Running, Paused, Stopped

  import State.*

  type Transition = String => State

  val transitions: Map[State, Transition] = Map(
    Idle    -> { case "start" => Running;  case _ => Idle    },
    Running -> { case "pause" => Paused;   case "stop" => Stopped; case _ => Running },
    Paused  -> { case "resume" => Running; case "stop" => Stopped; case _ => Paused  },
    Stopped -> { _ => Stopped }
  )

  def step(state: State, event: String): State =
    transitions.getOrElse(state, _ => state)(event)

  var state = Idle
  for event <- List("start", "pause", "resume", "stop") do
    state = step(state, event)
    println(s"$event -> $state")
end main
```

Each entry in the `transitions` map is a partial-function-style lambda  
written with `case` clauses. The `step` helper looks up the current  
state's transition lambda and applies it to the incoming event string.  
Unrecognised events fall through to the wildcard `case _ =>` clause,  
leaving the state unchanged. Encoding transitions as lambdas in a map  
makes it trivial to add or modify states without touching a large  
`match` expression.  

## Lambdas in DSLs

Lambdas with receiver-like semantics enable fluent builder DSLs.  

```scala
@main def main() =

  case class HtmlElement(tag: String, content: String)

  class HtmlBuilder:
    private val elements = collection.mutable.ListBuffer[HtmlElement]()

    def add(tag: String)(body: => String): HtmlBuilder =
      elements += HtmlElement(tag, body)
      this

    def build(): String =
      elements.map(e => s"<${e.tag}>${e.content}</${e.tag}>").mkString("\n")

  val html = HtmlBuilder()
    .add("h1")("Hello, Scala 3")
    .add("p")("Lambdas power DSLs.")
    .add("p")("Concise and expressive.")
    .build()

  println(html)
end main
```

`add` is a curried method whose second parameter is a by-name block  
`body: => String`. The caller provides the content as a lambda-like  
block using `("text")`, which evaluates lazily. Chaining `.add(…)(…)`  
calls produces a fluent builder interface reminiscent of HTML templating  
libraries. The final `build` call maps each stored element to an HTML  
tag string and joins them with newlines.  

## Lambdas with givens

Lambdas can be passed as `given` values to supply behaviour through  
Scala 3's contextual abstraction system.  

```scala
@main def main() =

  trait Comparator[A]:
    def compare(a: A, b: A): Int

  def sortWith[A](xs: List[A])(using cmp: Comparator[A]): List[A] =
    xs.sortWith((a, b) => cmp.compare(a, b) < 0)

  given Comparator[Int] with
    def compare(a: Int, b: Int): Int = a - b

  given Comparator[String] with
    def compare(a: String, b: String): Int = a.compareTo(b)

  println(sortWith(List(3, 1, 4, 1, 5, 9, 2, 6)))
  println(sortWith(List("banana", "apple", "cherry")))
end main
```

`sortWith` is generic in `A` and requires a `given Comparator[A]` in  
scope. Internally it passes a lambda `(a, b) => cmp.compare(a, b) < 0`  
to `List.sortWith`. The `given` instances supply the comparison logic  
for `Int` and `String` without the caller having to pass them explicitly.  
Lambdas and `given` instances complement each other: `given` carries  
reusable context, while lambdas supply one-off behaviour.  

## Context functions

A context function is a lambda that automatically receives an implicit  
argument, enabling scoped dependency injection.  

```scala
@main def main() =

  case class Config(prefix: String, separator: String)

  type Configured[A] = Config ?=> A

  def formatLine(value: String): Configured[String] =
    val cfg = summon[Config]
    s"${cfg.prefix}${cfg.separator}$value"

  def runWith(cfg: Config)(block: Config ?=> Unit): Unit =
    block(using cfg)

  runWith(Config("INFO", " | ")) {
    println(formatLine("Application started"))
    println(formatLine("Listening on port 8080"))
  }
end main
```

The type alias `Configured[A]` expands to `Config ?=> A`, a context  
function type. Any lambda of this type automatically receives a `Config`  
from the surrounding `given`/`using` scope. `formatLine` uses  
`summon[Config]` to access the injected configuration. `runWith`  
accepts such a lambda and provides the `Config` with `(using cfg)`.  
Context functions are a Scala 3 feature that enables zero-boilerplate  
dependency injection scoped to a block.  

## Lambdas in partial functions

`PartialFunction` restricts a lambda to a subset of its input domain  
and lets callers test applicability.  

```scala
@main def main() =

  val parseDigit: PartialFunction[Char, Int] =
    case '0' => 0
    case '1' => 1
    case '2' => 2
    case '3' => 3
    case '4' => 4
    case '5' => 5
    case '6' => 6
    case '7' => 7
    case '8' => 8
    case '9' => 9

  val input = "a1b2c3d"

  val digits = input.collect(parseDigit)
  println(digits)
  println(parseDigit.isDefinedAt('5'))
  println(parseDigit.isDefinedAt('z'))
end main
```

A `PartialFunction` is written as a sequence of `case` clauses without  
an explicit argument. The `collect` method on collections applies the  
partial function only to elements for which it is defined, silently  
skipping the rest. `isDefinedAt` tests whether a value falls within the  
function's domain without invoking it. Partial functions are useful  
wherever only a subset of the input type is valid input.  

## Lambdas with inline methods

An `inline` method can accept a lambda and inline its body at the call  
site, eliminating the function-object overhead.  

```scala
@main def main() =

  inline def measure(label: String)(inline block: => Long): Unit =
    val start = System.nanoTime()
    val result = block
    val elapsed = System.nanoTime() - start
    println(s"$label: result=$result, elapsed=${elapsed}ns")

  measure("sum 1..1000"):
    (1 to 1000).foldLeft(0L)(_ + _)

  measure("string concat"):
    (1 to 100).foldLeft("")((acc, n) => acc + n.toString).length.toLong
end main
```

`inline def` tells the compiler to substitute the method body at every  
call site. The `inline block: => Long` parameter is a by-name lambda  
that is also inlined, so there is no closure allocation at runtime.  
The `measure` utility times any block that returns a `Long` and prints  
the label alongside the result and elapsed nanoseconds. Inlining  
lambdas is particularly valuable in performance-sensitive utilities  
such as benchmarking helpers or assertion frameworks.  

## Y-combinator style recursion

The Y-combinator expresses anonymous recursion without a named function.  

```scala
@main def main() =

  def fix[A, B](f: (A => B) => (A => B)): A => B =
    lazy val self: A => B = f(self)
    self

  val factorial = fix[Int, Long](rec => n => if n <= 1 then 1L else n * rec(n - 1))
  val fibonacci = fix[Int, Long](rec => n => if n <= 1 then n.toLong else rec(n - 1) + rec(n - 2))

  println((1 to 10).map(factorial).toList)
  println((0 to 10).map(fibonacci).toList)
end main
```

`fix` is the fixed-point combinator. It takes a lambda `f` that receives  
a self-reference and returns a function. The `lazy val self` ties the  
knot: `self` calls `f(self)`, which supplies `self` as the recursive  
reference. The `factorial` and `fibonacci` lambdas are written without  
any `def` or named binding; the recursion is expressed purely through  
the argument `rec`. This is primarily a theoretical demonstration—in  
idiomatic Scala, named `def` is preferred for recursive functions.  

## Idiomatic lambda usage

Well-written Scala uses the simplest lambda form that keeps the meaning  
clear.  

```scala
@main def main() =

  case class Product(name: String, price: Double, inStock: Boolean)

  val catalog = List(
    Product("Keyboard", 49.99, true),
    Product("Monitor",  299.00, true),
    Product("Mouse",    19.99, false),
    Product("Headset",  79.00, true),
    Product("Webcam",   59.99, false)
  )

  val available = catalog
    .filter(_.inStock)
    .sortBy(_.price)
    .map(p => f"${p.name}%-12s £${p.price}%.2f")

  available.foreach(println)

  val total = catalog
    .filter(_.inStock)
    .map(_.price)
    .sum

  println(f"\nTotal available: £$total%.2f")
end main
```

The underscore shorthand `_.inStock` and `_.price` is ideal for simple  
field access where the intent is immediately obvious. A named parameter  
`p =>` is preferred when the body uses the value more than once, as in  
the formatted string `f"${p.name}%-12s £${p.price}%.2f"`. Keeping  
lambda forms consistent—shorthand for trivial projections, named  
parameters for anything more involved—makes collection pipelines easy  
to scan and maintain. The pipeline idiom also naturally separates  
concerns: filtering, sorting, and formatting are each expressed in one  
line.  
