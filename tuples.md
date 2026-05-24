# Tuples

A tuple in Scala 3 is a fixed-size, immutable, heterogeneous container  
that groups a known number of values, each with its own static type.  
Unlike a `List` or `Array`, a tuple does not require all elements to  
share the same type, and unlike a `Map`, it associates no keys with  
values. Tuples are value-aggregate types: `(Int, String, Boolean)`  
describes exactly three values of three different static types, and  
the compiler tracks every element type individually.  

**Tuples versus case classes and collections**  

A case class also groups heterogeneous values, but it attaches a named  
type and field names to each slot. Tuples are lighter, require no  
declaration, and are ideal for ad-hoc grouping where field names would  
add no clarity. A collection such as `List[Int]` is homogeneous and  
can grow or shrink at runtime; a tuple is always heterogeneous and  
fixed in its arity. When you find yourself adding field names and  
documentation to a tuple, the code is signalling that a case class  
would be clearer.  

**Immutability and fixed arity**  

Every tuple in Scala 3 is immutable. You cannot update a slot in  
place; any transformation produces a new tuple. The arity, meaning  
the number of elements, is part of the type: `(Int, String)` and  
`(Int, String, Boolean)` are entirely different types with no common  
supertype beyond `Tuple`. This makes tuples safe to share across  
threads and function boundaries without defensive copying.  

**Tuple1 through Tuple22 and new Scala 3 operations**  

Scala 2 represented tuples as distinct classes `Tuple1` through  
`Tuple22`, limiting arity to 22 elements. Scala 3 retains those  
classes for JVM compatibility but also introduces a first-class  
`Tuple` type hierarchy built from a type-level cons operator `*:`  
and an `EmptyTuple` base case. This makes it possible to write  
generic code that works over tuples of arbitrary arity using match  
types, inline methods, and compile-time operations. Scala 3 also  
adds `++` for concatenation, `splitAt` for splitting, and a  
polymorphic `map` method on the `Tuple` trait itself.  

**Performance characteristics**  

Tuples are allocated on the heap as ordinary objects. Accessing an  
element with `._1` or `._2` is a single field read and runs in O(1).  
Constructing a tuple allocates one object whose size is proportional  
to its arity. For the common cases of 2- to 5-element tuples the  
JIT compiler can often eliminate the allocation entirely through  
escape analysis. Very large tuples (more than 22 elements) are  
stored as `Array`-backed structures internally, which is transparent  
to the programmer but affects GC pressure.  

**When tuples are appropriate**  

Prefer tuples for short-lived, anonymous groupings: multiple return  
values from a private helper, intermediate results in a pipeline,  
pairs in a `Map`, or data that is immediately destructured by the  
caller. Prefer case classes when the grouped data travels far  
through the codebase, carries domain significance, or benefits from  
named fields, default values, or type class derivation.  

---

## Creating a tuple

A tuple literal groups comma-separated values inside parentheses.  

```scala
@main def main() =
  val pair   = (1, "hello")
  val triple = (42, true, 3.14)
  val single = Tuple1("only")
  val empty  = EmptyTuple

  println(pair)
  println(triple)
  println(single)
  println(empty)
end main
```

The parenthesised syntax `(a, b, c)` is syntactic sugar for  
`Tuple3(a, b, c)`. The compiler infers the type of each slot  
independently, so `pair` has type `(Int, String)` without any  
annotation. `Tuple1` wraps a single value and prevents the  
parentheses from being interpreted as ordinary grouping. `EmptyTuple`  
is the zero-element tuple and serves as the base case in  
type-level tuple recursion.  

## Accessing tuple elements

Individual elements of a tuple are read with the `._N` projection  
methods, where `N` is the one-based index.  

```scala
@main def main() =
  val t = (10, "Scala", true, 3.14)

  println(t._1)
  println(t._2)
  println(t._3)
  println(t._4)

  val first = t(0)
  println(first)
end main
```

The projection `t._1` returns the first element and is compiled to  
a direct field access, making it O(1). The `apply(n)` method accepts  
a zero-based `Int` index and returns an `Any`, which is useful when  
the index is computed dynamically but loses the precise element type.  
Attempting to access an out-of-range index with `apply` throws an  
`IndexOutOfBoundsException` at runtime.  

## Destructuring a tuple

Assigning a tuple to a pattern of variables binds each slot to a  
named value in one step.  

```scala
@main def main() =
  val point = (3, 7)
  val (x, y) = point

  println(s"x = $x, y = $y")

  val (name, age, active) = ("Alice", 30, true)
  println(s"$name is $age years old, active: $active")

  val (head, tail*) = (1, 2, 3, 4, 5)
  println(head)
  println(tail)
end main
```

The `val (x, y) = point` pattern is a structural match that the  
compiler rewrites to `val x = point._1; val y = point._2`. Using  
`tail*` on the right side of a pattern captures all remaining  
elements as a `Tuple`. Destructuring eliminates verbose `._N` chains  
and makes the intent of each binding clear at the call site.  

## Pattern matching on tuples

Tuple patterns inside `match` expressions let you dispatch on  
combinations of values in a single, readable construct.  

```scala
@main def main() =
  val pairs = List((0, 0), (1, 0), (0, 1), (3, 4))

  for p <- pairs do
    p match
      case (0, 0)       => println("origin")
      case (x, 0)       => println(s"on x-axis at $x")
      case (0, y)       => println(s"on y-axis at $y")
      case (x, y)       => println(s"general point ($x, $y)")
end main
```

Each `case` clause destructures the tuple and binds its slots to  
fresh variables scoped to that branch. Constants like `0` act as  
equality guards: the branch is chosen only when the corresponding  
slot equals that constant. The final catch-all `case (x, y)`  
matches any pair not handled by an earlier branch and is required  
to make the match exhaustive.  

## Nested tuples

Tuples may contain other tuples, creating a hierarchical structure  
that can be destructured in layers.  

```scala
@main def main() =
  val rgb   = ((255, 0, 0), (0, 255, 0), (0, 0, 255))
  val ((r1, g1, b1), _, _) = rgb

  println(s"red channel: $r1, $g1, $b1")

  val matrix = ((1, 2), (3, 4))
  val ((a, b), (c, d)) = matrix

  println(a + b + c + d)
end main
```

The outer destructuring `((r1, g1, b1), _, _)` reaches into the  
first nested tuple in a single binding expression. The underscore  
pattern `_` discards elements that are not needed, which prevents  
compiler warnings about unused variables. Deeply nested tuples can  
become hard to read; when the nesting exceeds two levels, a  
hierarchy of case classes usually communicates intent more clearly.  

## Tuples in for-comprehensions

A `for` expression can destructure tuple elements directly in its  
generator pattern, eliminating manual projections inside the body.  

```scala
@main def main() =
  val coords = List((1, 2), (3, 4), (5, 6))

  val sums =
    for (x, y) <- coords
    yield x + y

  println(sums)

  for
    (x, y) <- coords
    if x + y > 5
  do println(s"($x, $y) sums to ${x + y}")
end main
```

The generator `(x, y) <- coords` is a pattern that the compiler  
translates into a `map` call whose lambda destructures each tuple.  
The guard `if x + y > 5` is a `withFilter` call appended to the  
comprehension; only pairs satisfying the predicate reach the body.  
`for` / `yield` produces a new `List`, while `for` / `do` is used  
for its side effects.  

## Mapping over a tuple

The polymorphic `map` method on `Tuple` applies a type-preserving  
function to every element and returns a new tuple of the same arity.  

```scala
@main def main() =
  val t = (1, "hello", true)

  val asStrings = t.map([X] => (x: X) => x.toString)
  println(asStrings)

  val lengths = ("one", "two", "three").map(
    [X] => (x: X) => x.asInstanceOf[String].length
  )
  println(lengths)
end main
```

The syntax `[X] => (x: X) => x.toString` is a polymorphic function  
literal introduced in Scala 3. It is required here because each slot  
of the tuple has a different type, so the mapping function must be  
generic over the element type `X`. The result has type  
`Tuple.Map[T, F]`, where `F[X]` is `String` in the first example,  
giving the compiler enough information to type-check the output as  
`(String, String, String)` without losing static typing.  

## Swapping tuple elements

Exchanging the two elements of a pair is a common operation that can  
be made ergonomic with an extension method.  

```scala
extension [A, B](t: (A, B))
  def swap: (B, A) = (t._2, t._1)

@main def main() =
  val p = (42, "hello")
  println(p.swap)

  val q = ("Scala", 3)
  println(q.swap)

  val pairs = List((1, 'a'), (2, 'b'), (3, 'c'))
  println(pairs.map(_.swap))
end main
```

The extension method `swap` is generic in both `A` and `B`, so it  
works for any pair regardless of element types. Calling `p.swap`  
constructs a fresh tuple `(t._2, t._1)`; no mutation occurs.  
Applying `_.swap` inside `map` demonstrates that extension methods  
integrate seamlessly with higher-order collection operations.  

## Concatenating tuples

The `++` operator appends one tuple to another and produces a new  
tuple whose type reflects the combined element sequence.  

```scala
@main def main() =
  val t1 = (1, 2, 3)
  val t2 = ("a", "b")
  val t3 = t1 ++ t2

  println(t3)
  println(t3._4)
  println(t3._5)

  val empty = EmptyTuple
  val t4    = (10, 20) ++ empty
  println(t4)
end main
```

`t1 ++ t2` has type `(Int, Int, Int, String, String)`, which the  
compiler computes at compile time using `Tuple.Concat[T1, T2]`.  
All projections, including `._4` and `._5`, are therefore  
type-checked at compile time; accessing a non-existent projection  
would be a compile error. Concatenating with `EmptyTuple` on either  
side returns a tuple equal to the other operand.  

## Splitting a tuple

`splitAt` divides a tuple at a given index, returning a pair of  
tuples that together contain all original elements.  

```scala
@main def main() =
  val t = (1, 2, 3, 4, 5)

  val (left, right) = t.splitAt(2)
  println(left)
  println(right)

  val (head, rest) = t.splitAt(1)
  println(head)
  println(rest)
end main
```

`t.splitAt(2)` returns `((Int, Int), (Int, Int, Int))`: the first  
two elements on the left and the remaining three on the right. The  
split index must be a literal integer because the compiler uses it  
to compute the precise output types via `Tuple.Take` and  
`Tuple.Drop` at compile time. Providing a non-literal index forces  
the result type to `(Tuple, Tuple)`, losing individual element types.  

## Tupled and untupled functions

`Function.tupled` converts a multi-parameter function into one that  
accepts a single tuple argument, and `Function.untupled` reverses  
the transformation.  

```scala
@main def main() =
  val add = (x: Int, y: Int) => x + y
  val tAdd = add.tupled

  println(tAdd((3, 4)))

  val pairs = List((1, 2), (3, 4), (5, 6))
  println(pairs.map(tAdd))

  val f: ((Int, Int)) => Int = { case (x, y) => x + y }
  val g = Function.untupled(f)
  println(g(10, 20))
end main
```

`add.tupled` is equivalent to `(p: (Int, Int)) => add(p._1, p._2)`.  
Passing `tAdd` to `map` is idiomatic when a `List[(A, B)]` must be  
transformed by a function `(A, B) => C`, because `map` expects a  
function of one argument. `Function.untupled` goes in the opposite  
direction, splitting the single tuple argument back into separate  
parameters, which is useful when an API requires a curried style.  

## Zipping collections into tuples

`zip` pairs corresponding elements from two collections into a  
list of 2-tuples.  

```scala
@main def main() =
  val names  = List("Alice", "Bob", "Carol")
  val scores = List(85, 92, 78)

  val zipped = names.zip(scores)
  println(zipped)

  val indexed = names.zipWithIndex
  println(indexed)

  val zipped3 = (names lazyZip scores lazyZip List(1, 2, 3)).toList
  println(zipped3)
end main
```

`zip` stops at the shorter collection, so extra elements from the  
longer one are silently discarded. `zipWithIndex` is a convenience  
method that pairs each element with its zero-based position without  
requiring a second collection. `lazyZip` is the Scala 3 way to zip  
three or more collections simultaneously; it returns a lazy view  
that materialises into a `List` only when `toList` is called,  
avoiding the creation of intermediate collections.  

## Unzipping collections of tuples

`unzip` splits a list of pairs into two separate lists in a single  
pass.  

```scala
@main def main() =
  val pairs = List(("Alice", 85), ("Bob", 92), ("Carol", 78))

  val (names, scores) = pairs.unzip
  println(names)
  println(scores)

  val triples = List((1, 'a', true), (2, 'b', false))
  val (ns, cs, bs) = triples.unzip3
  println(ns)
  println(cs)
  println(bs)
end main
```

`unzip` traverses the list once and fills two accumulators, one for  
each tuple slot. The result type is `(List[A], List[B])`, and the  
two output lists always have the same length as the input. `unzip3`  
handles lists of 3-tuples in the same way. When working with larger  
tuples, manually mapping each projection is the standard approach  
because no `unzip4` variant exists in the standard library.  

## Tuples with case classes

Scala 3 provides smooth interoperability between case classes and  
tuples through `Tuple.fromProductTyped` and `.tupled` on companion  
objects.  

```scala
case class Point(x: Int, y: Int)
case class RGB(r: Int, g: Int, b: Int)

@main def main() =
  val p  = Point(3, 4)
  val t  = Tuple.fromProductTyped(p)
  println(t)
  println(t._1 + t._2)

  val mkPoint = Point.apply.tupled
  val p2 = mkPoint((10, 20))
  println(p2)

  val colors = List((255, 0, 0), (0, 255, 0))
  val rgbs   = colors.map(RGB.apply.tupled)
  println(rgbs)
end main
```

`Tuple.fromProductTyped(p)` converts any case class instance into a  
typed tuple of its fields in declaration order; the result type is  
`(Int, Int)` for `Point`. `Point.apply.tupled` converts the companion  
object's two-argument `apply` into a one-argument function that  
accepts a pair, which composes cleanly with `map` over a list of  
raw tuples. This pattern is useful when reading data as tuples from  
external sources and converting into domain types.  

## Tuples with generics

Generic functions over tuples express operations that work for any  
element types without losing static type information.  

```scala
def swap[A, B](pair: (A, B)): (B, A) =
  (pair._2, pair._1)

def mapPair[A, B, C](pair: (A, B), f: A => C): (C, B) =
  (f(pair._1), pair._2)

def zip3[A, B, C](a: A, b: B, c: C): (A, B, C) =
  (a, b, c)

@main def main() =
  println(swap(1, "hello"))
  println(swap(true, 3.14))
  println(mapPair((5, "world"), _ * 2))
  println(zip3("x", 42, true))
end main
```

Each function is parameterised over its tuple element types, letting  
the compiler enforce that `swap` returns `(B, A)` and not `(A, B)`.  
`mapPair` transforms only the first element while preserving the  
type of the second; the type parameter `C` is inferred from the  
return type of `f`. Generic tuple utilities like these can be  
composed into expressive data transformation pipelines that remain  
fully type-safe.  

## Tuples with givens

A `given` instance can supply an `Ordering` for tuples that drives  
sorted operations on collections.  

```scala
given Ordering[(Int, String)] =
  Ordering.by(_._1)

def sortPairs[A, B](xs: List[(A, B)])(using ord: Ordering[(A, B)])
    : List[(A, B)] =
  xs.sorted

@main def main() =
  val pairs = List((3, "c"), (1, "a"), (2, "b"))
  println(pairs.sorted)

  val pairs2 = List((1, "z"), (1, "a"), (2, "m"))
  val byBoth = Ordering.Tuple2[Int, String]
  println(pairs2.sorted(using byBoth))
end main
```

The `given Ordering[(Int, String)]` is resolved implicitly whenever  
`sorted` is called on a `List[(Int, String)]` without an explicit  
ordering argument. `Ordering.by(_._1)` creates an ordering that  
compares pairs by their first element only; ties in the first slot  
are left in their original relative order. `Ordering.Tuple2[A, B]`  
is the standard library's lexicographic ordering that compares the  
first slot first and breaks ties by the second.  

## Tuples with match types

Match types compute a result type at compile time by switching on  
the structure of a tuple type, enabling precise type-level  
programming.  

```scala
type Head[T <: Tuple] = T match
  case h *: _ => h

type Tail[T <: Tuple] = T match
  case _ *: t => t
  case EmptyTuple => EmptyTuple

type Size[T <: Tuple] <: Int = T match
  case EmptyTuple => 0
  case _ *: tail  => compiletime.ops.int.S[Size[tail]]

@main def main() =
  val h: Head[(Int, String, Boolean)] = 42
  val t: Tail[(Int, String, Boolean)] = ("hello", true)

  println(h)
  println(t)
end main
```

`Head[T]` reduces to the type of the first element by matching the  
cons pattern `h *: _`; the compiler substitutes this type wherever  
`Head[T]` appears. `Size[T]` is a recursive match type whose base  
case is `0` and whose recursive case uses `S[N]` from  
`compiletime.ops.int` to represent `N + 1` as a type. These  
type-level definitions are evaluated entirely at compile time and  
add no runtime overhead.  

## Tuples with context functions

A context function type `A ?=> B` threads an implicit context  
through a computation; returning a tuple from such a function  
bundles multiple results with their shared context.  

```scala
type Config = Map[String, String]
type Configured[T] = Config ?=> T

def greeting: Configured[(String, String)] =
  val cfg = summon[Config]
  (cfg.getOrElse("prefix", "Hello"), cfg.getOrElse("name", "World"))

def render(g: (String, String)): String =
  val (prefix, name) = g
  s"$prefix, $name!"

@main def main() =
  given Config = Map("prefix" -> "Hi", "name" -> "Scala 3")
  val result = render(greeting)
  println(result)
end main
```

`Configured[T]` is a type alias for the context function type  
`Config ?=> T`. Calling `greeting` automatically passes the `given  
Config` in scope as the implicit argument; the function body  
retrieves it with `summon[Config]`. Returning a tuple from a context  
function is a concise way to produce several related values that  
are all computed from the same implicit environment.  

## Multiple return values

Returning a tuple from a function is the idiomatic Scala way to  
deliver several results without defining an extra class.  

```scala
def minMax(xs: List[Int]): (Int, Int) =
  (xs.min, xs.max)

def stats(xs: List[Double]): (Double, Double, Double) =
  val mean = xs.sum / xs.length
  val variance = xs.map(x => math.pow(x - mean, 2)).sum / xs.length
  (mean, math.sqrt(variance), xs.max - xs.min)

@main def main() =
  val (lo, hi) = minMax(List(4, 1, 7, 2, 9, 3))
  println(s"min = $lo, max = $hi")

  val (mean, stdDev, range) = stats(List(2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0))
  println(f"mean=$mean%.2f stdDev=$stdDev%.2f range=$range%.2f")
end main
```

Callers destructure the return value with `val (lo, hi) = minMax(…)`,  
giving each result a meaningful local name without repeating `._1`  
and `._2`. This pattern scales to three or four return values before  
a case class becomes clearer. Functions that return more than four  
values almost always benefit from a named result type because the  
index-based projections become difficult to keep track of.  

## Grouping with tuples

`groupBy` produces a `Map` whose values are grouped elements;  
representing the key alongside its group as a tuple makes the  
structure explicit.  

```scala
@main def main() =
  val words = List("cat", "dog", "cow", "ant", "bee", "elk")

  val byLength: Map[Int, List[String]] = words.groupBy(_.length)
  println(byLength)

  val asTable: List[(Int, List[String])] = byLength.toList.sortBy(_._1)
  for (len, ws) <- asTable do
    println(s"length $len: ${ws.mkString(", ")}")
end main
```

`groupBy` returns a `Map[K, List[V]]` whose iteration order is  
unspecified for the default `HashMap` implementation. Converting to  
`List[(K, List[V])]` with `toList` and then sorting by the first  
element gives a deterministic, human-readable output. The `for`  
generator destructures each pair into `len` and `ws`, making the  
body self-documenting without any projection syntax.  

## Pattern guards with tuples

A pattern guard `if condition` refines a tuple match by adding  
runtime constraints beyond structural equality.  

```scala
@main def main() =
  val results = List(
    ("Alice", 95, true),
    ("Bob",   72, false),
    ("Carol", 85, true),
    ("Dave",  60, false)
  )

  for r <- results do
    r match
      case (name, score, true) if score >= 90 =>
        println(s"$name: distinction")
      case (name, score, true) if score >= 80 =>
        println(s"$name: merit")
      case (name, _, true) =>
        println(s"$name: pass")
      case (name, score, false) =>
        println(s"$name: failed with $score")
end main
```

The third slot of each triple acts as a boolean flag; matching on  
the literal `true` or `false` narrows the branch further before  
the guard is evaluated. Pattern guards after `if` are not  
exhaustiveness-checked by the compiler, so it is the programmer's  
responsibility to ensure that all cases are covered. Combining  
structural patterns with guards is more readable than nested  
`if`/`else` chains for data-driven dispatch.  

## Tuples in domain modeling

Short, anonymous tuples serve as lightweight domain primitives for  
coordinates, ranges, intervals, and similar two- or three-value  
concepts.  

```scala
type Coord    = (Double, Double)
type LatLng   = (Double, Double)
type DateRange = (java.time.LocalDate, java.time.LocalDate)

def distance(a: Coord, b: Coord): Double =
  val (x1, y1) = a
  val (x2, y2) = b
  math.sqrt(math.pow(x2 - x1, 2) + math.pow(y2 - y1, 2))

@main def main() =
  val origin: Coord = (0.0, 0.0)
  val point:  Coord = (3.0, 4.0)
  println(distance(origin, point))

  import java.time.LocalDate
  val range: DateRange = (LocalDate.of(2024, 1, 1), LocalDate.of(2024, 12, 31))
  val (start, end) = range
  println(s"$start to $end")
end main
```

Type aliases like `type Coord = (Double, Double)` give a tuple a  
meaningful name without requiring a class declaration, improving  
readability at call sites. The alias is transparent to the compiler:  
`Coord` and `(Double, Double)` are interchangeable. This is  
particularly useful at module boundaries where the type appears  
frequently and deserves a name but does not yet warrant a full  
case class.  

## Tuples and type inference

Scala 3's bidirectional type inference fills in tuple type parameters  
from context, reducing the need for explicit annotations.  

```scala
@main def main() =
  val t1 = (1, "hello")
  val t2 = (true, 3.14, 'x')

  def showTypes[A, B](pair: (A, B)): String =
    s"(${pair._1.getClass.getSimpleName}, ${pair._2.getClass.getSimpleName})"

  println(showTypes(t1))
  println(showTypes(t2._1, t2._2))

  val pairs: List[(Int, String)] = List(1, 2, 3) zip List("a", "b", "c")
  println(pairs)

  def choose(flag: Boolean): (Int, String) =
    if flag then (1, "yes") else (0, "no")

  println(choose(true))
end main
```

The compiler infers `t1: (Int, String)` and `t2: (Boolean, Double,  
Char)` from the literal values on the right-hand side. In  
`choose`, both branches must produce a `(Int, String)` because the  
declared return type constrains inference; mixing incompatible  
types would be a compile error. Type inference over tuples extends  
to `zip`, which produces `List[(Int, String)]` without any  
annotation when the element types of both input collections are  
already known.  

## Tuples and type projections

`Tuple.Head`, `Tuple.Last`, `Tuple.Init`, and `Tuple.Tail` are  
built-in type-level projections that compute result types from a  
concrete tuple type.  

```scala
@main def main() =
  type T = (Int, String, Boolean, Double)

  val h: Tuple.Head[T]      = 42
  val l: Tuple.Last[T]      = 3.14
  val i: Tuple.Init[T]      = (42, "hi", true)
  val t: Tuple.Tail[T]      = ("hi", true, 3.14)
  val r: Tuple.Reverse[T]   = (3.14, true, "hi", 42)

  println(h)
  println(l)
  println(i)
  println(t)
  println(r)
end main
```

These type aliases are computed entirely at compile time from the  
static type `T`. `Tuple.Head[(Int, String)]` reduces to `Int`; the  
compiler substitutes that type wherever `Tuple.Head[T]` appears.  
`Tuple.Reverse[T]` produces `(Double, Boolean, String, Int)` by  
reversing the element sequence at the type level. Using these  
projections in function signatures makes it possible to write  
generic functions whose return types depend on the input tuple's  
structure without losing element-level precision.  

## Tuples and inline methods

`inline` methods with `inline match` on erased tuple types perform  
computation and specialisation entirely at compile time.  

```scala
import scala.compiletime.erasedValue

inline def tupleArity[T <: Tuple]: Int =
  inline erasedValue[T] match
    case _: EmptyTuple => 0
    case _: (_ *: tail) => 1 + tupleArity[tail]

inline def tupleHead[T <: NonEmptyTuple]: Tuple.Head[T] =
  inline erasedValue[T] match
    case _: (h *: _) => (??? : h)

@main def main() =
  println(tupleArity[(Int, String, Boolean)])
  println(tupleArity[(Int, String)])
  println(tupleArity[EmptyTuple])
end main
```

`erasedValue[T]` produces a phantom value of type `T` that exists  
only during type checking and is erased before the program runs.  
The `inline match` branches are selected at compile time based on  
the static shape of `T`, and only the selected branch is emitted  
into the bytecode. `tupleArity` therefore computes the arity as a  
compile-time constant with no runtime loop or reflection.  

## Tuples and compile-time operations

`constValue` and `Tuple.Size` extract compile-time constants from  
tuple types, enabling assertions and specialisations checked by  
the compiler.  

```scala
import scala.compiletime.{constValue, summonAll}

@main def main() =
  val arity = constValue[Tuple.Size[(Int, String, Boolean)]]
  println(arity)

  type Row = (Int, String, Double, Boolean)
  val rowWidth = constValue[Tuple.Size[Row]]
  println(s"Row has $rowWidth columns")

  val mirrored = Tuple.fromArray(Array(1, "hello", true))
  println(mirrored)
end main
```

`constValue[Tuple.Size[T]]` evaluates the size of the tuple type  
`T` at compile time and inserts the literal integer directly into  
the bytecode; the result is a compile-time constant, not a runtime  
computation. `Tuple.fromArray` constructs a runtime tuple from an  
`Array[Any]`, which is useful when interoperating with reflective  
or dynamically typed data sources. The combination of compile-time  
arity checks and runtime construction covers both typed and  
untyped data scenarios.  

## Tuples and functional transformations

Higher-order collection methods accept tuple-destructuring lambdas  
in Scala 3, enabling concise, functional data pipelines.  

```scala
@main def main() =
  val pairs = List((1, 2), (3, 4), (5, 6))

  val sums     = pairs.map((a, b) => a + b)
  val products = pairs.map((a, b) => a * b)
  val filtered = pairs.filter((a, b) => a + b > 5)

  println(sums)
  println(products)
  println(filtered)

  val result = pairs
    .filter((a, _) => a % 2 != 0)
    .map((a, b) => a * b)
    .foldLeft(0)(_ + _)
  println(result)
end main
```

Scala 3 allows `(a, b) => a + b` as the argument to `map` on a  
`List[(Int, Int)]`; the compiler automatically adapts the  
two-parameter lambda to the expected single-argument form. The  
underscore wildcard `(a, _)` discards the second element when only  
the first is needed. Chaining `filter`, `map`, and `foldLeft` on  
a list of tuples is idiomatic for compact data aggregation without  
intermediate named variables.  

## Tuples in real-world patterns

Tuples appear naturally when parsing records, building lookup  
tables, and returning structured diagnostic information from  
pure functions.  

```scala
case class ParseError(line: Int, message: String)

def parseLine(raw: String): Either[ParseError, (String, Int)] =
  raw.split(",").toList match
    case name :: age :: Nil =>
      age.toIntOption match
        case Some(n) => Right((name.trim, n))
        case None    => Left(ParseError(0, s"invalid age: $age"))
    case _ => Left(ParseError(0, s"bad format: $raw"))

@main def main() =
  val lines = List("Alice, 30", "Bob, bad", "Carol, 25")

  for raw <- lines do
    parseLine(raw) match
      case Right((name, age)) => println(s"$name is $age")
      case Left(err)          => println(s"Error: ${err.message}")
end main
```

`Either[ParseError, (String, Int)]` expresses that the function  
either fails with a structured error or succeeds with a  
name-and-age pair. The `Right` branch destructures the tuple  
directly in the pattern, keeping the success path readable without  
extra variable bindings. This approach avoids defining a dedicated  
result class for a private parsing helper while still preserving  
full type safety at every branch.  

## Idiomatic tuple usage

Knowing when to use tuples and when to reach for case classes is  
the key to writing clear, maintainable Scala 3.  

```scala
case class User(name: String, age: Int)
case class Stats(mean: Double, stdDev: Double, count: Int)

def summarise(users: List[User]): Stats =
  val ages  = users.map(_.age.toDouble)
  val mean  = ages.sum / ages.length
  val variance = ages.map(x => math.pow(x - mean, 2)).sum / ages.length
  Stats(mean, math.sqrt(variance), ages.length)

def initials(name: String): (Char, Char) =
  val parts = name.trim.split(" ")
  (parts.head.head, parts.last.head)

@main def main() =
  val users = List(User("Alice Smith", 30), User("Bob Jones", 25))
  val stats = summarise(users)
  println(f"mean=${stats.mean}%.1f sd=${stats.stdDev}%.1f n=${stats.count}")

  users.map(u => initials(u.name)).foreach { (f, l) =>
    println(s"Initials: $f.$l.")
  }
end main
```

`Stats` is a case class because it travels beyond the function  
that produces it and benefits from named fields. `initials` returns  
`(Char, Char)` because the pair is consumed immediately at the  
call site and the field names would add no information. The lambda  
`{ (f, l) => … }` in `foreach` destructures the pair inline,  
making the code as readable as named fields without the overhead  
of a class definition. Applying these guidelines consistently  
keeps domain types explicit where it matters while keeping  
plumbing code lightweight.  

## Tuple and Mirror

`Mirror.ProductOf` exposes the internal tuple representation of  
any case class, enabling generic, type-safe conversions between  
product types and tuples.  

```scala
import scala.deriving.Mirror

def toTuple[A](a: A)(using m: Mirror.ProductOf[A]): m.MirroredElemTypes =
  Tuple.fromProductTyped(a)

def fromTuple[A](t: Tuple)(using m: Mirror.ProductOf[A]): A =
  m.fromProduct(t)

case class Point(x: Int, y: Int)
case class Color(r: Int, g: Int, b: Int)

@main def main() =
  val p  = Point(3, 4)
  val t  = toTuple(p)
  println(t)

  val p2 = fromTuple[Point]((10, 20))
  println(p2)

  val c  = Color(255, 128, 0)
  val ct = toTuple(c)
  println(ct)
end main
```

`Mirror.ProductOf[A]` provides the type `MirroredElemTypes`, which  
is the tuple type corresponding to the fields of `A`. For `Point`  
that is `(Int, Int)` and for `Color` it is `(Int, Int, Int)`.  
`Tuple.fromProductTyped` converts any `Product` instance into the  
typed tuple without reflection, and `m.fromProduct` converts it  
back. This mechanism is the foundation of type class derivation in  
Scala 3: libraries such as `circe` and `upickle` use `Mirror` and  
tuples to automatically generate codecs for any case class.  
