# Advanced Scala Types & Type-Level Programming

Scala 3 offers one of the most expressive type systems available on the JVM.  
It draws from decades of research in programming language theory and delivers  
features that were previously confined to theorem provers or academic  
languages. The result is a practical system where powerful abstractions  
compile to efficient bytecode and integrate naturally with everyday code.  

Advanced types in Scala 3 serve three primary roles. They improve safety by  
encoding invariants directly in the type system so the compiler rejects  
programs that violate constraints before they run. They enable abstraction  
by expressing common patterns such as serialisation or ordering without  
inheritance, using typeclasses and given instances instead. They also guide  
API design by making illegal states unrepresentable and by encoding  
state-machine logic in types rather than documentation.  

Scala 3 unifies and simplifies several complex type features from earlier  
versions. Union types replace the historical need for Either chains or  
sealed trait hierarchies when modelling disjunctions. Intersection types  
provide a clean way to combine constraints. Opaque types replace the  
value-class pattern with cleaner syntax and stronger guarantees. Match types  
replace some macro use-cases with readable type-level logic. Together they  
reduce boilerplate and improve compile-time feedback.  

The Scala type system connects deeply with functional programming. Typeclasses  
express generic behaviour without inheritance. Given instances and context  
bounds make type-directed dispatch the default style. Dependent types connect  
types to values, enabling libraries that were previously impossible without  
external code generation. The entire edifice supports purely functional code  
while remaining pragmatic enough for mixed-style projects.  

Compile-time and runtime behaviour are carefully separated. Match types,  
`inline` definitions, and `compiletime` operations run entirely during  
compilation and produce zero runtime overhead. Opaque types are erased to  
their underlying representation. Phantom types carry no runtime data. This  
discipline gives Scala programmers a rich toolkit for safety and  
expressiveness without sacrificing performance.  

---

## Basic union types

A union type `A | B` describes a value that is either of type `A` or `B`.  

```scala
def describe(value: Int | String | Boolean): String =
  value match
    case n: Int     => s"integer $n"
    case s: String  => s"string '$s'"
    case b: Boolean => s"boolean $b"

@main def main() =
  val values: List[Int | String | Boolean] =
    List(42, "hello", true, 0, "world", false)
  for v <- values do
    println(describe(v))
end main
```

`Int | String | Boolean` is a union type. The compiler tracks that every  
possible case is handled in the `match` expression and reports a warning  
if a branch is missing. Inside each branch, the type is narrowed to the  
concrete alternative so `n`, `s`, and `b` carry precise static types.  
Union types are commutative: `Int | String` is identical to `String | Int`.  

## Basic intersection types

An intersection type `A & B` describes a value that satisfies both `A`  
and `B` simultaneously.  

```scala
trait Flyable:
  def fly(): String

trait Swimmable:
  def swim(): String

class Duck extends Flyable, Swimmable:
  def fly(): String  = "Duck is flying"
  def swim(): String = "Duck is swimming"

def demonstrate(creature: Flyable & Swimmable): Unit =
  println(creature.fly())
  println(creature.swim())

@main def main() =
  val duck = new Duck
  demonstrate(duck)
end main
```

The parameter `creature: Flyable & Swimmable` demands that the argument  
implement both traits. `Duck` satisfies the constraint, so the call  
compiles cleanly. Inside `demonstrate`, both `fly` and `swim` are  
available without any cast. Intersection types are the dual of union  
types and represent the logical AND of two constraints, ensuring that  
capabilities are combined at the type level rather than documented  
in prose.  

## Narrowing unions

Pattern matching narrows a union type to one of its specific alternatives  
inside each branch.  

```scala
type Input = Int | Double | String

def process(input: Input): String =
  input match
    case n: Int    => s"whole number: $n"
    case d: Double => f"decimal: $d%.2f"
    case s: String =>
      if s.isEmpty then "empty string"
      else s"text of length ${s.length}"

@main def main() =
  val inputs: List[Input] = List(7, 3.14, "Scala", "", -1, 2.718)
  inputs.foreach(i => println(process(i)))
end main
```

Each `case` branch narrows the type from `Input` to the concrete  
alternative. Inside `case s: String`, the compiler knows `s` is a  
`String` and allows calling `.isEmpty` and `.length` without any cast.  
Exhaustiveness is checked at compile time: removing any branch produces  
a compiler warning. Narrowing is the primary technique for safely  
working with union values and is fully integrated with pattern matching.  

## Combining unions and intersections

Union and intersection types compose freely to express nuanced constraints.  

```scala
trait Readable:
  def read(): String

trait Writable:
  def write(data: String): Unit

trait Closeable:
  def close(): Unit

type ReadWrite = Readable & Writable
type Resource  = ReadWrite | Closeable

def handle(r: Resource): Unit =
  r match
    case rw: (Readable & Writable) =>
      println(s"Read: ${rw.read()}")
      rw.write("response")
    case c: Closeable =>
      c.close()
      println("resource closed")

@main def main() =
  val rw = new ReadWrite:
    def read(): String       = "payload"
    def write(d: String): Unit = println(s"wrote: $d")
  handle(rw)
end main
```

`ReadWrite` is an intersection requiring both capabilities. `Resource` is  
a union that accepts either an intersection or a closeable. Composing the  
two operators expresses fine-grained requirements without introducing an  
inheritance hierarchy. The `match` on `Resource` is exhaustive because  
both alternatives are covered, and the compiler verifies this statically.  

## Unions in APIs

Union types simplify API design by replacing ad-hoc error hierarchies  
with concise, self-documenting return types.  

```scala
case class Success(value: String)
case class NotFound(key: String)
case class Forbidden(user: String)

type LookupResult = Success | NotFound | Forbidden

def lookup(user: String, key: String): LookupResult =
  if user == "guest"       then Forbidden(user)
  else if key.startsWith("_") then NotFound(key)
  else Success(s"value-of-$key")

@main def main() =
  val pairs = List(
    ("alice", "config"),
    ("guest", "config"),
    ("bob",   "_secret")
  )
  for (user, key) <- pairs do
    lookup(user, key) match
      case Success(v)   => println(s"OK: $v")
      case NotFound(k)  => println(s"Key not found: $k")
      case Forbidden(u) => println(s"Access denied: $u")
end main
```

The return type `LookupResult` documents all possible outcomes in one  
place. Callers must handle every alternative because the compiler enforces  
exhaustiveness. Unlike a sealed trait hierarchy, the three case classes  
are independent and can be reused across different return types without  
inheritance coupling. Adding a new outcome simply extends the union and  
triggers compile-time warnings at every incomplete match site.  

## Intersections for constraints

Intersection types enforce that a value meets multiple constraints  
simultaneously at the call site.  

```scala
trait Named:
  def name: String

trait Aged:
  def age: Int

def sortAndShow[A <: Named & Aged](items: List[A]): Unit =
  val sorted = items.sortBy(_.age)
  sorted.foreach(p => println(s"${p.name} (${p.age})"))

case class Person(name: String, age: Int) extends Named, Aged

@main def main() =
  val people = List(
    Person("Charlie", 35),
    Person("Alice",   28),
    Person("Bob",     31)
  )
  sortAndShow(people)
end main
```

The bound `A <: Named & Aged` guarantees that any type passed to  
`sortAndShow` provides both `name` and `age`. The compiler rejects types  
that satisfy only one constraint. Intersection bounds are a clean  
alternative to introducing a new combined trait just for a single  
method signature and keep each trait focused on a single concern.  

## Pattern matching on unions

Pattern matching on union types provides exhaustive and type-safe dispatch  
across all alternatives.  

```scala
sealed trait Shape
case class Circle(radius: Double)       extends Shape
case class Rectangle(w: Double, h: Double) extends Shape
case class Triangle(base: Double, h: Double) extends Shape

type Primitive = Int | String | Boolean

def describeShape(s: Shape): String =
  s match
    case Circle(r)       => f"Circle  r=$r%.1f"
    case Rectangle(w, h) => f"Rect    ${w}x$h%.1f"
    case Triangle(b, h)  => f"Triangle b=$b%.1f h=$h%.1f"

def describePrimitive(p: Primitive): String =
  p match
    case n: Int     => s"Int($n)"
    case s: String  => s"Str($s)"
    case b: Boolean => s"Bool($b)"

@main def main() =
  val shapes: List[Shape] =
    List(Circle(5), Rectangle(4, 6), Triangle(3, 4))
  shapes.foreach(s => println(describeShape(s)))
  val prims: List[Primitive] = List(1, "hello", true)
  prims.foreach(p => println(describePrimitive(p)))
end main
```

Both sealed trait hierarchies and explicit union types support exhaustive  
matching. Sealed traits match by constructor patterns while union  
alternatives match by `case v: Type`. The compiler verifies completeness  
in both cases, so adding a new variant triggers a compile-time warning on  
every incomplete match site. The two styles are fully complementary and  
can be mixed in the same codebase.  

## Intersections with givens

A given instance can be provided for an intersection type, combining  
behaviour from multiple typeclasses in a single declaration.  

```scala
trait Show[A]:
  def show(a: A): String

trait Eq[A]:
  def eqv(a: A, b: A): Boolean

type ShowEq[A] = Show[A] & Eq[A]

case class Color(r: Int, g: Int, b: Int)

object Color:
  given ShowEq[Color] with
    def show(c: Color): String =
      s"rgb(${c.r},${c.g},${c.b})"
    def eqv(a: Color, b: Color): Boolean =
      a.r == b.r && a.g == b.g && a.b == b.b

def compare[A](a: A, b: A)(using se: ShowEq[A]): Unit =
  println(se.show(a))
  println(if se.eqv(a, b) then "equal" else "different")

@main def main() =
  import Color.given
  compare(Color(255, 0, 0), Color(255, 0, 0))
  compare(Color(0, 255, 0), Color(0,   0, 255))
end main
```

The `given` instance for `Color` implements both `Show[Color]` and  
`Eq[Color]` simultaneously. The function `compare` requires a single  
context parameter of the intersection type, so callers provide one  
instance instead of two. This pattern keeps the given scope tidy when  
two capabilities are always used together and avoids the need for a  
dedicated combined typeclass trait.  

## Union types with null

Scala 3's explicit-null mode expresses nullable values directly through  
union types rather than relying on implicit null references.  

```scala
type MaybeString = String | Null

def greet(name: MaybeString): String =
  name match
    case null    => "Hello, stranger!"
    case s: String => s"Hello, $s!"

def findUser(id: Int): String | Null =
  if id == 1 then "Alice" else null

@main def main() =
  val names: List[MaybeString] = List("Alice", null, "Bob", null)
  names.foreach(n => println(greet(n)))
  val user = findUser(1)
  println(greet(user))
  val missing = findUser(99)
  println(greet(missing))
end main
```

Using `String | Null` makes nullability visible and explicit in the type  
signature. The compiler forces every call site to handle both alternatives  
before using the value as a `String`, eliminating an entire class of null  
pointer exceptions. In projects compiled with `-Yexplicit-nulls`, standard  
library references become non-nullable by default and null must be  
introduced deliberately through union types.  

## Union flattening and widening

The compiler automatically flattens nested unions and widens the type of  
a collection that holds several union alternatives.  

```scala
type AB = Int | String
type ABC = AB | Boolean   // flattened to Int | String | Boolean

def widen(x: Int | String): Int | String | Boolean = x

@main def main() =
  val a: Int | String          = 42
  val b: Int | String | Boolean = widen(a)

  // Compiler widens element type of a mixed list
  val mixed = List(1, "two", true, 3, "four")
  // mixed: List[Int | String | Boolean]

  mixed.foreach:
    case n: Int     => println(s"number $n")
    case s: String  => println(s"text   $s")
    case b: Boolean => println(s"flag   $b")
end main
```

Nesting `A | (B | C)` and `(A | B) | C` are the same type as `A | B | C`.  
The compiler normalises union types so there is no difference between a  
deeply nested and a flat form. When the compiler infers the type of a  
`List` whose elements span several union alternatives it widens  
automatically to the smallest enclosing union type.  

---

## Simple type aliases

A type alias introduces a new name for an existing type without creating  
a new type at the value level.  

```scala
type Callback    = String => Unit
type ResultMap   = Map[String, Int]
type Predicate[A] = A => Boolean

def runWith(data: String, cb: Callback): Unit = cb(data)

def filter[A](items: List[A], p: Predicate[A]): List[A] =
  items.filter(p)

@main def main() =
  val log: Callback = msg => println(s"[LOG] $msg")
  runWith("event fired", log)

  val scores: ResultMap = Map("alice" -> 90, "bob" -> 75)
  println(scores)

  val isEven: Predicate[Int] = _ % 2 == 0
  println(filter(List(1, 2, 3, 4, 5, 6), isEven))
end main
```

Type aliases are transparent to the compiler: `Callback` and  
`String => Unit` are completely interchangeable and there is no  
runtime difference. Aliases improve readability by giving  
domain-meaningful names to common function or collection types  
without introducing wrappers. They are the right choice when the goal  
is documentation rather than type-safety enforcement.  

## Parameterized type aliases

Type aliases can be parameterized and act as small type-level functions.  

```scala
type Pair[A]         = (A, A)
type Transform[A, B]  = A => B
type Validated[E, A] = Either[List[E], A]

def applyTwice[A](f: Transform[A, A], pair: Pair[A]): Pair[A] =
  (f(pair._1), f(pair._2))

def validate[A](
    value: A,
    checks: List[A => Option[String]]
): Validated[String, A] =
  val errors = checks.flatMap(_(value))
  if errors.isEmpty then Right(value) else Left(errors)

@main def main() =
  val doubled: Pair[Int] = applyTwice(_ * 2, (3, 7))
  println(doubled)

  val ageChecks: List[Int => Option[String]] = List(
    age => if age < 0   then Some("negative age") else None,
    age => if age > 150 then Some("unrealistic")  else None
  )
  println(validate(25, ageChecks))
  println(validate(-5, ageChecks))
end main
```

Parameterized aliases make complex generic signatures readable.  
`Validated[String, A]` reads as a domain concept rather than  
`Either[List[String], A]` scattered across the codebase. Because  
aliases are transparent, the compiler checks correctness against  
the underlying `Either` type without any extra wrapper overhead.  
Higher-kinded aliases such as `Pair` also improve the signal-to-noise  
ratio of method signatures in utility libraries.  

## Opaque types

An opaque type creates a new, distinct type backed by an existing type  
but indistinguishable from it outside the defining scope.  

```scala
opaque type Meters    = Double
opaque type Kilograms = Double

object Meters:
  def apply(d: Double): Meters = d
  extension (m: Meters) def value: Double = m

object Kilograms:
  def apply(d: Double): Kilograms = d
  extension (k: Kilograms) def value: Double = k

def speed(dist: Meters, time: Double): Double =
  dist.value / time

@main def main() =
  val distance = Meters(100.0)
  val mass     = Kilograms(75.0)
  println(f"speed = ${speed(distance, 9.58)}%.2f m/s")
  // speed(mass, 9.58)  // Compile error: Kilograms ≠ Meters
end main
```

Outside the companion objects `Meters` and `Kilograms` are fully  
opaque. You cannot pass a `Kilograms` value where `Meters` is expected,  
even though both are `Double` at runtime. Inside the companion the  
opaque type is transparent, so arithmetic can be performed directly.  
This delivers unit-safety at zero runtime cost, a goal that previously  
required value classes with their limitations.  

## Opaque types with companion methods

Companion objects extend opaque types with smart constructors,  
validation logic, and arithmetic operations.  

```scala
opaque type PositiveInt = Int

object PositiveInt:
  def apply(n: Int): Option[PositiveInt] =
    if n > 0 then Some(n) else None
  def unsafe(n: Int): PositiveInt =
    require(n > 0, s"Expected positive, got $n")
    n
  val one: PositiveInt = 1
  extension (p: PositiveInt)
    def value: Int                       = p
    def +(q: PositiveInt): PositiveInt   = (p: Int) + (q: Int)
    def doubled: PositiveInt             = (p: Int) * 2

@main def main() =
  val a = PositiveInt.unsafe(5)
  val b = PositiveInt.unsafe(3)
  println((a + b).value)
  println(a.doubled.value)
  println(PositiveInt(10))
  println(PositiveInt(-1))
end main
```

Inside `object PositiveInt` the alias is transparent, so `(p: Int) + (q: Int)`  
calls `Int.+` directly. Outside the object, users can only construct  
`PositiveInt` through the smart constructors, maintaining the invariant  
that the wrapped integer is always positive. Extension methods add the  
arithmetic operations needed to work with the type naturally.  

## Opaque types for domain modeling

Opaque types prevent primitive obsession by giving each domain concept  
its own distinct type backed by a primitive.  

```scala
opaque type UserId   = String
opaque type Email    = String
opaque type Username = String

object UserId:
  def apply(s: String): UserId = s
  extension (id: UserId) def value: String = id

object Email:
  def apply(s: String): Option[Email] =
    if s.contains("@") then Some(s) else None
  extension (e: Email) def value: String = e

object Username:
  def apply(s: String): Username = s
  extension (u: Username) def value: String = u

case class User(id: UserId, email: Email, name: Username)

def greet(user: User): String =
  s"Hi ${user.name.value} <${user.email.value}>"

@main def main() =
  val user = User(
    UserId("u-001"),
    Email("alice@example.com").get,
    Username("alice")
  )
  println(greet(user))
  // greet(user.id, user.name, user.email)  // Compile error: wrong order
end main
```

Without opaque types, `UserId`, `Email`, and `Username` are all `String`  
and can be silently swapped. With opaque types the compiler rejects every  
accidental transposition. `Email.apply` enforces the invariant that an  
`@` sign is present. The domain model becomes self-documenting and  
machine-verified at no runtime cost.  

## Zero-cost abstractions

Opaque types are erased at compile time, producing identical bytecode to  
direct use of the underlying primitive.  

```scala
opaque type Millis = Long
opaque type Nanos  = Long

object Millis:
  def apply(n: Long): Millis   = n
  def now: Millis              = System.currentTimeMillis()
  extension (m: Millis)
    def value: Long            = m
    def toNanos: Nanos         = (m: Long) * 1_000_000L
    def +(other: Millis): Millis = (m: Long) + (other: Long)

object Nanos:
  def apply(n: Long): Nanos  = n
  extension (n: Nanos)
    def value: Long          = n
    def toMillis: Millis     = (n: Long) / 1_000_000L

@main def main() =
  val start = Millis.now
  Thread.sleep(10)
  val elapsed = Millis.now + Millis(-start.value)
  println(s"Elapsed: ${elapsed.value} ms")
  println(s"In nanos: ${elapsed.toNanos.value}")
end main
```

The JVM sees only `long` values throughout. No boxing, no wrapper objects,  
and no virtual dispatch occur at runtime. The zero-cost property makes  
opaque types superior to value classes, which still allocate on the heap  
in some circumstances. Libraries can expose rich typed APIs without  
sacrificing the performance characteristics of raw primitives.  

## Opaque types vs type aliases

Type aliases and opaque types have the same syntax but radically different  
guarantees about type identity across module boundaries.  

```scala
// Transparent alias: String and Name are the same type everywhere
type Name = String

// Opaque type: UserName is distinct outside its companion
opaque type UserName = String
object UserName:
  def apply(s: String): UserName = s.trim
  extension (u: UserName) def value: String = u

def acceptString(s: String): Unit = println(s"String: $s")

@main def main() =
  val alias: Name     = "Alice"
  val opaque: UserName = UserName("  Bob  ")

  acceptString(alias)       // OK: Name =:= String
  // acceptString(opaque)   // Compile error: UserName ≠ String
  acceptString(opaque.value) // OK: explicit unwrap

  // Type alias allows confusion:
  val rawName: Name = "  not trimmed  "
  println(rawName)   // no sanitisation enforced
  println(opaque.value) // trimmed by smart constructor
end main
```

Type aliases are useful when you want documentation without enforcement.  
Opaque types are the right choice when you need to enforce invariants  
and prevent accidental mixing of structurally identical but semantically  
different values. Prefer opaque types for domain primitives and reserve  
aliases for readability improvements in generic signatures.  

## Newtype pattern with opaque types

Opaque types can replicate the Haskell newtype pattern to wrap any  
underlying type with a distinct identity.  

```scala
opaque type Latitude  = Double
opaque type Longitude = Double

object Latitude:
  def apply(d: Double): Either[String, Latitude] =
    if d >= -90.0 && d <= 90.0 then Right(d)
    else Left(s"Invalid latitude: $d")
  extension (lat: Latitude) def value: Double = lat

object Longitude:
  def apply(d: Double): Either[String, Longitude] =
    if d >= -180.0 && d <= 180.0 then Right(d)
    else Left(s"Invalid longitude: $d")
  extension (lon: Longitude) def value: Double = lon

case class Coordinate(lat: Latitude, lon: Longitude)

@main def main() =
  val result =
    for
      lat <- Latitude(48.8566)
      lon <- Longitude(2.3522)
    yield Coordinate(lat, lon)

  result match
    case Right(c) =>
      println(f"Paris: ${c.lat.value}%.4f, ${c.lon.value}%.4f")
    case Left(err) => println(s"Error: $err")
end main
```

The newtype pattern enforces domain validity at construction and prevents  
mixing latitudes and longitudes even though both are `Double` at runtime.  
Using `Either` in the smart constructor lets callers handle invalid input  
with the standard monadic for-comprehension. Adding extension methods to  
the companion provides a natural API without any runtime overhead.  

---

## Basic match types

A match type computes a result type by inspecting a type argument,  
mirroring how pattern matching inspects values.  

```scala
type Elem[C] = C match
  case List[t]  => t
  case Array[t] => t
  case String   => Char

@main def main() =
  val nums: List[Int]      = List(1, 2, 3)
  val strs: Array[String]  = Array("a", "b")
  val text: String         = "Scala"

  val n: Elem[List[Int]]     = nums.head    // Elem reduces to Int
  val s: Elem[Array[String]] = strs(0)      // Elem reduces to String
  val c: Elem[String]        = text(0)      // Elem reduces to Char

  println(n)
  println(s)
  println(c)
end main
```

`Elem[List[Int]]` reduces to `Int` at compile time without any runtime  
evaluation. The compiler applies the match type like a function from  
types to types when the argument is concrete. This eliminates the need  
for implicit evidence chains that were previously required to express  
the same relationship. Match types are the bridge between value-level  
pattern matching and type-level computation.  

## Recursive match types

Match types can recurse on themselves to compute type-level quantities  
from structural information.  

```scala
import scala.compiletime.ops.int.S

// Compute the length of a tuple type at compile time
type Length[T <: Tuple] = T match
  case EmptyTuple => 0
  case _ *: t     => S[Length[t]]

@main def main() =
  val t1: (Int, String, Boolean) = (1, "two", true)
  val t2: (Int, String)          = (1, "two")

  // summon witnesses that the type-level lengths are correct
  val _: Length[(Int, String, Boolean)] =:= 3 = summon
  val _: Length[(Int, String)]          =:= 2 = summon
  val _: Length[EmptyTuple]             =:= 0 = summon

  println(s"Tuple sizes verified at compile time")
  println(t1)
  println(t2)
end main
```

`S[N]` is the type-level successor from `scala.compiletime.ops.int`.  
The recursion unfolds entirely during type-checking: `Length[(Int, String)]`  
reduces to `S[S[0]]` which equals `2`. `summon` checks that the equality  
holds without any runtime computation. Recursive match types enable  
type-level arithmetic that would otherwise require a macro.  

## Match types with tuples

Match types can deconstruct tuple types to extract or transform their  
element types.  

```scala
// Extract the head type of a non-empty tuple
type Head[T <: NonEmptyTuple] = T match
  case h *: ? => h

// Extract the tail type
type Tail[T <: NonEmptyTuple] = T match
  case ? *: t => t

// Map a function over all element types in a tuple
type Mapped[T <: Tuple, F[_]] = T match
  case EmptyTuple => EmptyTuple
  case h *: t     => F[h] *: Mapped[t, F]

@main def main() =
  type T = (Int, String, Boolean)

  val headEvidence:   Head[T] =:= Int     = summon
  val tailEvidence:   Tail[T] =:= (String, Boolean) = summon

  println("Head and Tail match types verified")

  type OptionT = Mapped[(Int, String), Option]
  val _: OptionT =:= (Option[Int], Option[String]) = summon
  println("Mapped match type verified")
end main
```

The wildcard `?` inside a match type case matches any type without  
binding it. `h *: t` binds the head and tail of the tuple type. These  
primitives compose to build rich type-level tuple operations. `Mapped`  
shows how to lift a type constructor over every element in a tuple,  
a capability that underpins generic tuple processing libraries.  

## Match types with unions

Match types can inspect union type arguments and compute different  
result types for each alternative.  

```scala
type Unwrap[T] = T match
  case Option[a] => a
  case List[a]   => a
  case Either[?, b] => b
  case a         => a

@main def main() =
  val _: Unwrap[Option[Int]]          =:= Int    = summon
  val _: Unwrap[List[String]]         =:= String = summon
  val _: Unwrap[Either[String, Long]] =:= Long   = summon
  val _: Unwrap[Double]               =:= Double = summon

  println("Unwrap match type verified for all cases")

  // Using Unwrap to annotate values
  def unwrapOption[A](o: Option[A]): Unwrap[Option[A]] =
    o.get
  println(unwrapOption(Some(42)))
end main
```

Each arm of the match type handles one structural form. The final  
catch-all `case a => a` acts as an identity for types that do not match  
any wrapper. `summon` constructs evidence of the equality relations that  
the compiler derives from the match type reduction, confirming that each  
branch produces the expected result type without any runtime operation.  

## Match types for compile-time computation

Match types combined with `compiletime.ops` perform arithmetic at the  
type level and embed the result in value-level code.  

```scala
import scala.compiletime.ops.int.*

type Max[A <: Int, B <: Int] = (A >= B) match
  case true  => A
  case false => B

type Abs[A <: Int] = (A >= 0) match
  case true  => A
  case false => Negate[A]

@main def main() =
  val _: Max[3, 7]  =:= 7   = summon
  val _: Max[10, 4] =:= 10  = summon
  val _: Abs[-5]    =:= 5   = summon
  val _: Abs[3]     =:= 3   = summon

  println("Compile-time arithmetic verified")

  // The widths are known at compile time
  type Width  = 1920
  type Height = 1080
  val _: Max[Width, Height] =:= 1920 = summon
  println("Max dimension verified")
end main
```

`compiletime.ops.int` exposes arithmetic and comparison operators that  
produce singleton `Int` literal types. Match types can branch on the  
result of comparisons because `>=` produces a `Boolean` literal type.  
The entire computation happens at compile time and produces a concrete  
literal type that subsequent code can use as a constraint.  

## Match types for type-level programming

Match types enable type-level programming patterns analogous to  
value-level functional programming.  

```scala
import scala.compiletime.ops.int.*

// Flatten a nested tuple type one level deep
type Flatten[T <: Tuple] = T match
  case EmptyTuple   => EmptyTuple
  case Tuple *: t   => Tuple Concat Flatten[t]
  case h *: t       => h *: Flatten[t]

// Reverse a tuple type
type Reverse[T <: Tuple] = T match
  case EmptyTuple => EmptyTuple
  case h *: t     => Tuple.Concat[Reverse[t], h *: EmptyTuple]

@main def main() =
  type T = (Int, String, Boolean)
  val _: Reverse[T] =:= (Boolean, String, Int) = summon
  println("Reverse match type verified")

  type Pair = (Int, String)
  val _: Reverse[Pair] =:= (String, Int) = summon
  println("Reverse of pair verified")
end main
```

`Tuple.Concat` is the standard library type-level concatenation operator  
for tuples. Combined with match type recursion it becomes a building block  
for type-level list processing. These patterns are used in shapeless-style  
generic programming and in macroless serialisation libraries that need to  
enumerate and transform the element types of a product at compile time.  

## Match types with inline

`inline` methods use match types so the compiler resolves the return type  
and inlines the logic at every call site.  

```scala
import scala.compiletime.{erasedValue, error}

inline def defaultValue[T]: T =
  inline erasedValue[T] match
    case _: Int     => 0.asInstanceOf[T]
    case _: Long    => 0L.asInstanceOf[T]
    case _: Double  => 0.0.asInstanceOf[T]
    case _: Boolean => false.asInstanceOf[T]
    case _: String  => "".asInstanceOf[T]
    case _          => error("No default value for this type")

@main def main() =
  val i: Int     = defaultValue[Int]
  val d: Double  = defaultValue[Double]
  val b: Boolean = defaultValue[Boolean]
  val s: String  = defaultValue[String]

  println(s"Int:     $i")
  println(s"Double:  $d")
  println(s"Boolean: $b")
  println(s"String:  '$s'")
end main
```

`erasedValue[T]` produces a phantom value of type `T` used only for  
`inline match` dispatch. The actual case bodies never execute the  
match at runtime; instead the compiler inlines the matching arm that  
corresponds to the concrete type argument. `error` terminates  
compilation with a message if no arm matches, making the constraint  
explicit at compile time rather than at runtime.  

## Match types for safe APIs

Match types make API return types depend on their input types, removing  
the need for unsafe casts at call sites.  

```scala
sealed trait Codec
case object IntCodec    extends Codec
case object StringCodec extends Codec
case object BoolCodec   extends Codec

type Decoded[C <: Codec] = C match
  case IntCodec.type    => Int
  case StringCodec.type => String
  case BoolCodec.type   => Boolean

def decode[C <: Codec](codec: C, raw: String): Decoded[C] =
  (codec match
    case IntCodec    => raw.toInt
    case StringCodec => raw
    case BoolCodec   => raw.toBoolean
  ).asInstanceOf[Decoded[C]]

@main def main() =
  val n: Int     = decode(IntCodec,    "42")
  val s: String  = decode(StringCodec, "hello")
  val b: Boolean = decode(BoolCodec,   "true")

  println(n + 1)
  println(s.toUpperCase)
  println(!b)
end main
```

The return type `Decoded[C]` reduces to the exact type that corresponds  
to the given codec singleton. Callers receive an `Int`, `String`, or  
`Boolean` directly without any cast and without a wrapper type. The  
single `asInstanceOf` inside the implementation is the only unsafe point  
and it is hidden behind a safe interface. This is one of the most  
practical applications of match types in everyday API design.  

## Match types and bounds

Match type arms can constrain type variables with bounds to express  
relationships between the input and output types.  

```scala
type NumericResult[T] = T match
  case Int    => Long
  case Float  => Double
  case Long   => Long
  case Double => Double

def widen[T](x: T): NumericResult[T] =
  (x match
    case n: Int    => n.toLong
    case f: Float  => f.toDouble
    case n: Long   => n
    case d: Double => d
  ).asInstanceOf[NumericResult[T]]

@main def main() =
  val i: Long   = widen(42)
  val f: Double = widen(3.14f)
  val l: Long   = widen(100L)
  val d: Double = widen(2.718)

  println(s"Int    -> Long:   $i")
  println(s"Float  -> Double: $f")
  println(s"Long   -> Long:   $l")
  println(s"Double -> Double: $d")
end main
```

`NumericResult` encodes a promotion table in the type system. Each arm  
maps a narrow numeric type to its wider counterpart. Call sites are  
guaranteed to receive the correct wide type without any explicit  
annotation. This approach centralises widening logic in one match type  
rather than spreading overloads across the codebase.  

## Match types for heterogeneous collections

Match types enable type-safe access to heterogeneous tuples by selecting  
the element type at a given index.  

```scala
import scala.compiletime.ops.int.*

type At[T <: Tuple, I <: Int] = T match
  case h *: ?  => I match
    case 0 => h
    case S[i] => At[? *: Tuple, i]

// Concrete version for small tuples
type Fst[T <: NonEmptyTuple] = T match { case h *: ? => h }
type Snd[T <: Tuple]         = T match { case ? *: h *: ? => h }
type Thd[T <: Tuple]         = T match { case ? *: ? *: h *: ? => h }

@main def main() =
  type Row = (Int, String, Boolean)

  val _: Fst[Row] =:= Int     = summon
  val _: Snd[Row] =:= String  = summon
  val _: Thd[Row] =:= Boolean = summon
  println("Heterogeneous tuple element types verified")

  val row: Row = (42, "scala", true)
  val fst: Fst[Row] = row._1
  val snd: Snd[Row] = row._2
  val thd: Thd[Row] = row._3

  println(fst * 2)
  println(snd.toUpperCase)
  println(!thd)
end main
```

`Fst`, `Snd`, and `Thd` are match types that project specific element  
types from a tuple type. Because they reduce at compile time, the result  
variables `fst`, `snd`, and `thd` carry the precise types `Int`, `String`,  
and `Boolean` without any cast. This technique underpins generic tuple  
processing libraries that need to map functions over heterogeneous  
structures while preserving full type information at every step.  

---

## Dependent method types

A dependent method type allows the return type of a method to reference  
one of its value parameters.  

```scala
trait Container:
  type Item
  def get: Item

class IntBox(private val n: Int) extends Container:
  type Item = Int
  def get: Int = n

class StrBox(private val s: String) extends Container:
  type Item = String
  def get: String = s

def extract(c: Container): c.Item = c.get

@main def main() =
  val ib = IntBox(42)
  val sb = StrBox("hello")

  val n: Int    = extract(ib)  // return type is ib.Item = Int
  val s: String = extract(sb)  // return type is sb.Item = String

  println(n)
  println(s)
end main
```

The return type `c.Item` depends on the specific value `c` passed to  
`extract`. When called with `ib`, the compiler resolves `ib.Item` to  
`Int` and gives the result that precise type. No cast is required.  
Dependent method types are the foundation of type-safe heterogeneous  
data structures and plugin systems where each plugin defines its own  
associated output type.  

## Dependent function types

Scala 3 introduces dependent function types that capture the  
type-dependency relationship in a first-class function value.  

```scala
trait Key:
  type Value
  def get: Value

val intKey = new Key:
  type Value = Int
  def get: Int = 99

val strKey = new Key:
  type Value = String
  def get: String = "dependent"

// Dependent function type: return type depends on the argument
type Extractor = (k: Key) => k.Value

val extract: Extractor = k => k.get

@main def main() =
  val n: Int    = extract(intKey)
  val s: String = extract(strKey)
  println(n)
  println(s)

  // Store extractors in a list
  val extractors: List[Extractor] = List(extract, extract)
  println(extractors.head(intKey))
end main
```

`(k: Key) => k.Value` is a first-class dependent function type. It can  
be stored in a variable, passed as an argument, or placed in a data  
structure like any other function. This is a Scala 3 innovation; earlier  
versions could only express the dependency in method signatures. Dependent  
function types make it possible to build higher-order abstractions that  
preserve type-level information across function boundaries.  

## Path-dependent types

Types defined inside an object depend on the specific outer instance,  
producing distinct types for each container even though they share the  
same structure.  

```scala
class Registry:
  class Entry(val label: String)
  private var entries: List[Entry] = Nil

  def register(label: String): Entry =
    val e = Entry(label)
    entries = e :: entries
    e

  def list: List[Entry] = entries.reverse

@main def main() =
  val r1 = new Registry
  val r2 = new Registry

  val a: r1.Entry = r1.register("alpha")
  val b: r2.Entry = r2.register("beta")

  // r1.Entry and r2.Entry are different types
  // val wrong: r1.Entry = b   // Compile error

  r1.register("gamma")
  println(r1.list.map(_.label))
  println(r2.list.map(_.label))
end main
```

`r1.Entry` and `r2.Entry` are path-dependent types. They share the  
same structure but the compiler treats them as incompatible, preventing  
entries from one registry from being inserted into another. This  
technique is used to enforce scope safety in DSLs, effect systems, and  
resource managers where mixing artefacts from different contexts must  
be a compile-time error.  

## Dependent pairs

A dependent pair (or sigma type) bundles a value with a type that depends  
on it, approximated in Scala 3 through type members.  

```scala
trait Dep:
  type T
  val value: T

def pack[A](a: A): Dep { type T = A } =
  new Dep:
    type T = A
    val value: T = a

def useFirst(d: Dep): d.T = d.value

@main def main() =
  val d1: Dep { type T = Int }    = pack(42)
  val d2: Dep { type T = String } = pack("hello")

  val n: d1.T = useFirst(d1)   // d1.T = Int
  val s: d2.T = useFirst(d2)   // d2.T = String

  println(n * 2)
  println(s.toUpperCase)

  // Type information flows through the pair
  val pairs: List[Dep] = List(pack(1), pack("two"), pack(true))
  pairs.foreach(p => println(p.value))
end main
```

`pack` returns a refinement `Dep { type T = A }` that records the exact  
type of the packed value. When held through the refined type, the  
companion value `n` has precise type `Int`. When held through the  
existential `Dep`, the type member is abstract and the value can only  
be used as `Any`. This approximates the sigma type of dependent type  
theory using the tools available in Scala 3.  

## Dependent return types

Methods can compute a return type by examining the runtime value of an  
argument whose type carries sufficient information.  

```scala
sealed trait Format
case object JsonFormat extends Format
case object CsvFormat  extends Format

trait Serialised[F <: Format]:
  type Output
  def result: Output

def serialise(fmt: Format)(data: Map[String, Int])
    : Serialised[fmt.type] =
  fmt match
    case JsonFormat =>
      new Serialised[JsonFormat.type]:
        type Output = String
        def result: String =
          data.map((k, v) => s""""$k":$v""").mkString("{", ",", "}")
    case CsvFormat =>
      new Serialised[CsvFormat.type]:
        type Output = String
        def result: String =
          data.map((k, v) => s"$k,$v").mkString("\n")

@main def main() =
  val data = Map("a" -> 1, "b" -> 2)
  val json = serialise(JsonFormat)(data)
  val csv  = serialise(CsvFormat)(data)
  println(json.result)
  println(csv.result)
end main
```

The return type `Serialised[fmt.type]` depends on the singleton type of  
the `fmt` argument. Each concrete `fmt` value produces a different  
`Serialised` subtype with its own `Output` member. The pattern extends  
naturally to plugin systems and codec registries where the exact output  
type depends on a runtime-selected strategy.  

## Dependent types in APIs

Practical APIs combine dependent types with type members to create  
type-safe heterogeneous maps and request-response channels.  

```scala
trait TypedKey[V]:
  def name: String

object Keys:
  val userId: TypedKey[Int]    = new TypedKey[Int]    { def name = "userId"    }
  val userName: TypedKey[String] = new TypedKey[String] { def name = "userName"  }
  val active: TypedKey[Boolean]= new TypedKey[Boolean] { def name = "active"    }

class TypedMap(private val store: Map[String, Any] = Map.empty):
  def put[V](key: TypedKey[V], value: V): TypedMap =
    TypedMap(store + (key.name -> value))
  def get[V](key: TypedKey[V]): Option[V] =
    store.get(key.name).map(_.asInstanceOf[V])

@main def main() =
  val m = TypedMap()
    .put(Keys.userId,   1001)
    .put(Keys.userName, "alice")
    .put(Keys.active,   true)

  println(m.get(Keys.userId))
  println(m.get(Keys.userName))
  println(m.get(Keys.active))
end main
```

`TypedKey[V]` binds a string key to the type of its associated value.  
`get` returns `Option[V]` where `V` is determined by the key, so the  
caller never needs to cast. The single internal `asInstanceOf` is  
confined to the map implementation and hidden from users. This technique  
is the foundation of type-safe heterogeneous maps used in effect systems,  
configuration libraries, and dependency injection frameworks.  

## Dependent types with generics

Dependent types interact with type parameters to produce refined generic  
abstractions.  

```scala
trait Algebra:
  type Expr
  def num(n: Int): Expr
  def add(a: Expr, b: Expr): Expr
  def eval(e: Expr): Int

val intAlgebra: Algebra =
  new Algebra:
    type Expr = Int
    def num(n: Int): Int           = n
    def add(a: Int, b: Int): Int   = a + b
    def eval(e: Int): Int          = e

val strAlgebra: Algebra =
  new Algebra:
    type Expr = String
    def num(n: Int): String           = n.toString
    def add(a: String, b: String): String = s"($a + $b)"
    def eval(e: String): Int             = e.count(_ == '+')

def compute(alg: Algebra): alg.Expr =
  alg.add(alg.num(1), alg.add(alg.num(2), alg.num(3)))

@main def main() =
  val iResult = compute(intAlgebra)
  val sResult = compute(strAlgebra)
  println(intAlgebra.eval(iResult))
  println(strAlgebra.eval(sResult))
end main
```

`compute` returns `alg.Expr`, a type that varies with the chosen algebra.  
With `intAlgebra` the result is `Int`; with `strAlgebra` it is `String`.  
This is the tagless final encoding in Scala 3: programmes are written  
against an abstract `Algebra` and evaluated by substituting concrete  
interpreters. Dependent types make the encoding type-safe without  
requiring a type parameter on every method.  

## Dependent types with givens

Given instances can be provided for path-dependent or refined types,  
combining dependent type safety with typeclass-driven dispatch.  

```scala
trait Scope:
  type Item
  given itemOrdering: Ordering[Item]

val intScope: Scope =
  new Scope:
    type Item = Int
    given itemOrdering: Ordering[Int] = Ordering.Int

val strScope: Scope =
  new Scope:
    type Item = String
    given itemOrdering: Ordering[String] = Ordering.String

def sort(scope: Scope)(items: List[scope.Item]): List[scope.Item] =
  given Ordering[scope.Item] = scope.itemOrdering
  items.sorted

@main def main() =
  val nums = sort(intScope)(List(3, 1, 4, 1, 5, 9, 2))
  val strs = sort(strScope)(List("banana", "apple", "cherry"))
  println(nums)
  println(strs)
end main
```

The `Scope` trait bundles a type member with a `given` instance that  
knows how to order values of that type. `sort` retrieves the ordering  
from the scope and brings it into the implicit scope with a local  
`given` alias. This pattern creates self-contained capability bundles  
where the typeclass instance is guaranteed to match the associated type.  

---

## Typeclass pattern

The typeclass pattern separates behaviour from data by defining an  
interface as a parameterised trait and providing instances separately.  

```scala
trait Show[A]:
  def show(a: A): String

object Show:
  given Show[Int] with
    def show(n: Int): String = n.toString
  given Show[Boolean] with
    def show(b: Boolean): String = if b then "yes" else "no"
  given [A](using s: Show[A]): Show[List[A]] with
    def show(xs: List[A]): String =
      xs.map(s.show).mkString("[", ", ", "]")

def display[A](a: A)(using s: Show[A]): Unit =
  println(s.show(a))

@main def main() =
  import Show.given
  display(42)
  display(true)
  display(List(1, 2, 3))
  display(List(false, true))
end main
```

`Show[A]` is the typeclass. Given instances for `Int`, `Boolean`, and  
`List[A]` are the evidence values that tell the compiler how to render  
each type. The `List` instance derives from any existing `Show` instance  
for the element type, demonstrating how typeclass instances compose  
inductively. This pattern is the backbone of every major Scala library.  

## Given instances

Given instances are the Scala 3 replacement for implicit values,  
providing context parameters automatically based on their type.  

```scala
trait Formatter[A]:
  def format(a: A): String

case class Temperature(celsius: Double)
case class Distance(metres: Double)

object Temperature:
  given Formatter[Temperature] with
    def format(t: Temperature): String =
      f"${t.celsius}%.1f°C"

object Distance:
  given Formatter[Distance] with
    def format(d: Distance): String =
      f"${d.metres}%.2f m"

def print[A](value: A)(using fmt: Formatter[A]): Unit =
  println(fmt.format(value))

@main def main() =
  import Temperature.given, Distance.given
  print(Temperature(36.6))
  print(Distance(1609.34))
  print(Temperature(-40.0))
end main
```

Given instances are resolved by the compiler based on their type and  
scope. Placing them in the companion object of the type they serve means  
they are automatically visible without any explicit import. The `using`  
clause in `print` requests a `Formatter[A]` from the compiler, which  
searches the given scope and fills in the argument transparently.  

## Summon

`summon[T]` retrieves a given instance of type `T` from the implicit  
scope without requiring a method context parameter.  

```scala
trait Config[A]:
  def value: A

object Config:
  given Config[Int] with
    def value: Int = 8080
  given Config[String] with
    def value: String = "localhost"
  given Config[Boolean] with
    def value: Boolean = false

@main def main() =
  import Config.given

  val port: Int        = summon[Config[Int]].value
  val host: String     = summon[Config[String]].value
  val debug: Boolean   = summon[Config[Boolean]].value

  println(s"$host:$port (debug=$debug)")

  // Retrieve and use in one step
  def cfg[A](using c: Config[A]): A = c.value
  println(cfg[Int])
  println(cfg[String])
end main
```

`summon` is the Scala 3 equivalent of `implicitly`. It explicitly  
materialises a given instance, which is useful when you need the  
instance itself rather than just using it through a method parameter.  
It is particularly helpful in `inline` methods where the compiler  
must resolve the instance at compile time before code generation.  

## Context bounds

A context bound `[A: TC]` is syntactic sugar for a `using` parameter  
of type `TC[A]`, keeping method signatures concise.  

```scala
trait Summable[A]:
  def zero: A
  def add(a: A, b: A): A

object Summable:
  given Summable[Int] with
    def zero: Int             = 0
    def add(a: Int, b: Int): Int = a + b
  given Summable[String] with
    def zero: String                = ""
    def add(a: String, b: String): String = a + b
  given Summable[Double] with
    def zero: Double                 = 0.0
    def add(a: Double, b: Double): Double = a + b

def sum[A: Summable](xs: List[A]): A =
  val s = summon[Summable[A]]
  xs.foldLeft(s.zero)(s.add)

@main def main() =
  import Summable.given
  println(sum(List(1, 2, 3, 4, 5)))
  println(sum(List("a", "b", "c")))
  println(sum(List(1.1, 2.2, 3.3)))
end main
```

`def sum[A: Summable]` desugars to `def sum[A](using Summable[A])`. The  
`summon[Summable[A]]` call inside the method body retrieves the instance  
by name. Context bounds are the idiomatic style in Scala 3 because they  
minimise noise in generic method signatures while making the constraints  
clearly visible in the type parameter list.  

## Deriving typeclass instances

Scala 3 allows case classes and enums to automatically derive typeclass  
instances using the `derives` clause.  

```scala
import scala.deriving.Mirror

trait Describable[A]:
  def describe(a: A): String

object Describable:
  inline def derived[A](using m: Mirror.Of[A]): Describable[A] =
    new Describable[A]:
      def describe(a: A): String = a.toString

case class Point(x: Int, y: Int)          derives Describable
case class Person(name: String, age: Int) derives Describable

enum Direction derives Describable:
  case North, South, East, West

def show[A](a: A)(using d: Describable[A]): Unit =
  println(d.describe(a))

@main def main() =
  show(Point(1, 2))
  show(Person("Alice", 30))
  show(Direction.North)
end main
```

The `derives Describable` clause on a type instructs the compiler to  
call `Describable.derived` with a `Mirror` for the type and place the  
result in the companion object as a given instance. `Mirror` provides  
structural information about case classes and enums at compile time.  
Full derivation of complex typeclasses such as `Show`, `Eq`, and  
`Codec` is implemented this way in libraries like Circe and Tapir.  

## Typeclass-driven design

Typeclass-driven design centres the architecture around behaviours  
rather than inheritance, making components independently composable.  

```scala
trait Encoder[A]:
  def encode(a: A): String

trait Decoder[A]:
  def decode(s: String): Either[String, A]

trait Codec[A] extends Encoder[A], Decoder[A]

object Codec:
  given Codec[Int] with
    def encode(n: Int): String           = n.toString
    def decode(s: String): Either[String, Int] =
      s.toIntOption.toRight(s"Not an int: $s")
  given Codec[Boolean] with
    def encode(b: Boolean): String           = b.toString
    def decode(s: String): Either[String, Boolean] =
      s.toBooleanOption.toRight(s"Not a boolean: $s")

def roundTrip[A: Codec](value: A): Either[String, A] =
  val enc = summon[Codec[A]].encode(value)
  summon[Codec[A]].decode(enc)

@main def main() =
  import Codec.given
  println(roundTrip(42))
  println(roundTrip(true))
  println(summon[Codec[Int]].decode("bad"))
end main
```

`Codec` inherits both `Encoder` and `Decoder` so a single given  
instance satisfies all three constraints. `roundTrip` is written purely  
in terms of the typeclass interface, making it polymorphic over any type  
that has a codec. New types gain serialisation by providing a `given`  
without modifying existing code, honouring the open-closed principle.  

## Coherence and orphan rules

Organising typeclass instances in companion objects enforces coherence  
and avoids the pitfalls of orphan instances.  

```scala
trait Pretty[A]:
  def pretty(a: A): String

// Instance in the typeclass companion — always visible
object Pretty:
  given Pretty[Int] with
    def pretty(n: Int): String = s"Int($n)"

// Instance in the data type companion — also always visible
case class RGB(r: Int, g: Int, b: Int)
object RGB:
  given Pretty[RGB] with
    def pretty(c: RGB): String =
      f"#${c.r}%02x${c.g}%02x${c.b}%02x"

// Orphan instance — defined in an unrelated object, must be imported
object ExtraInstances:
  given Pretty[String] with
    def pretty(s: String): String = s""""$s""""

def pp[A](a: A)(using p: Pretty[A]): Unit = println(p.pretty(a))

@main def main() =
  import RGB.given, Pretty.given, ExtraInstances.given
  pp(42)
  pp(RGB(255, 128, 0))
  pp("hello")
end main
```

The Scala compiler ensures that at any given call site only one instance  
of `Pretty[A]` for a particular `A` is in scope. Instances in companion  
objects are visible everywhere without explicit imports, guaranteeing a  
consistent interpretation. Orphan instances, defined outside both the  
typeclass and the data type, must be imported explicitly, making their  
use visible and auditable at every call site.  

## Typeclasses with match types

Combining typeclasses and match types enables instances whose behaviour  
is derived directly from the structural shape of a type.  

```scala
import scala.compiletime.{summonInline, erasedValue}

trait Size[A]:
  def size(a: A): Int

object Size:
  given Size[Int]    with def size(n: Int): Int    = 1
  given Size[String] with def size(s: String): Int = s.length
  given Size[Boolean] with def size(b: Boolean): Int = 1
  given [A](using s: Size[A]): Size[List[A]] with
    def size(xs: List[A]): Int = xs.map(s.size).sum
  given [A](using s: Size[A]): Size[Option[A]] with
    def size(oa: Option[A]): Int = oa.map(s.size).getOrElse(0)

def totalSize[A: Size](a: A): Int = summon[Size[A]].size(a)

@main def main() =
  import Size.given
  println(totalSize(42))
  println(totalSize("hello"))
  println(totalSize(List(1, 2, 3)))
  println(totalSize(Some("world")))
  println(totalSize(Option.empty[Int]))
end main
```

The `Size` instances compose structurally: the list instance builds on  
any existing `Size[A]`, and the option instance does the same. This  
mirrors how match types recurse on the structure of a type. In more  
advanced scenarios, `inline` methods with `summonInline` and  
`erasedValue` can dispatch to the correct instance entirely at compile  
time, eliminating virtual dispatch and enabling aggressive inlining.  

## Given imports

Selective given imports give call sites fine-grained control over which  
typeclass instances are in scope.  

```scala
trait Render[A]:
  def render(a: A): String

object Render:
  given Render[Int] with
    def render(n: Int): String = s"int:$n"
  given Render[String] with
    def render(s: String): String = s"str:$s"

object AltInstances:
  given Render[Int] with
    def render(n: Int): String = s"0x${n.toHexString}"

def show[A](a: A)(using r: Render[A]): Unit = println(r.render(a))

@main def main() =
  // Import only the String instance from Render
  import Render.{given Render[String]}
  // Import the alternative Int instance
  import AltInstances.{given Render[Int]}

  show(255)      // uses AltInstances.Render[Int]
  show("hello")  // uses Render.Render[String]
end main
```

`import Render.{given Render[String]}` imports only the `String` instance  
while leaving the `Int` slot open. `import AltInstances.{given Render[Int]}`  
then fills that slot with a different implementation. Selective given  
imports make it possible to swap individual instances without affecting  
others, which is essential in test suites and in code that needs  
alternative representations in specific contexts.  

## Typeclass hierarchy

Typeclasses can form inheritance hierarchies where richer instances  
reuse and refine simpler ones.  

```scala
trait Eq[A]:
  def eqv(a: A, b: A): Boolean

trait Ord[A] extends Eq[A]:
  def compare(a: A, b: A): Int
  def eqv(a: A, b: A): Boolean = compare(a, b) == 0

trait Num[A] extends Ord[A]:
  def plus(a: A, b: A): A
  def negate(a: A): A
  def zero: A

given Num[Int] with
  def compare(a: Int, b: Int): Int  = a.compareTo(b)
  def plus(a: Int, b: Int): Int     = a + b
  def negate(a: Int): Int           = -a
  def zero: Int                     = 0

def sort[A: Ord](xs: List[A]): List[A] =
  val ord = summon[Ord[A]]
  xs.sortWith((a, b) => ord.compare(a, b) < 0)

def sumAll[A: Num](xs: List[A]): A =
  val num = summon[Num[A]]
  xs.foldLeft(num.zero)(num.plus)

@main def main() =
  println(sort(List(3, 1, 4, 1, 5, 9, 2)))
  println(sumAll(List(10, 20, 30)))
end main
```

A `Num[Int]` instance automatically satisfies `Ord[Int]` and `Eq[Int]`  
because `Num` extends `Ord` which extends `Eq`. Methods requiring only  
`Ord` can receive a `Num` given without any extra declarations. Typeclass  
hierarchies mirror mathematical structures and are the standard approach  
in functional library design for encoding algebraic laws at the type level.  

---

## Structural types

Structural types describe values by the methods they expose rather than  
by their declared class, enabling duck-typed interfaces.  

```scala
import scala.reflect.Selectable.reflectiveSelectable

type Named   = { def name: String }
type Sizable = { def size: Int    }

def greet(n: Named): Unit     = println(s"Hello, ${n.name}!")
def measure(s: Sizable): Unit = println(s"Size: ${s.size}")

class Person(val name: String)
class Team(val name: String, members: List[String]):
  def size: Int = members.size

@main def main() =
  val p = Person("Alice")
  val t = Team("Scala", List("Alice", "Bob", "Carol"))

  greet(p)
  greet(t)
  measure(t)

  // Anonymous structural type in-place
  val widget = new { val name = "Button"; val size = 12 }
  greet(widget)
  measure(widget)
end main
```

Structural types in Scala 3 use reflection for method dispatch and  
require `reflectiveSelectable` to be in scope. `Person` and `Team` both  
satisfy `Named` despite no inheritance relationship. This is useful when  
integrating with APIs from different libraries that define equivalent but  
unrelated interfaces. The cost is a runtime reflective call per structural  
method invocation, so structural types are not recommended in hot paths.  

## Refinement types

A refinement type extends a base type with additional members or  
constraints expressed inline without a new named type.  

```scala
trait Animal:
  def name: String
  def legs: Int

type Domesticated = Animal { def owner: String }
type QuadrupedPet = Animal { def owner: String; def legs: Int }

class Cat(val name: String, val owner: String) extends Animal:
  val legs: Int = 4

class GoldFish(val name: String, val owner: String) extends Animal:
  val legs: Int = 0

def introduce(a: Domesticated): Unit =
  println(s"${a.name} belongs to ${a.owner}")

@main def main() =
  val cat  = Cat("Whiskers", "Alice")
  val fish = GoldFish("Nemo", "Bob")
  introduce(cat)
  introduce(fish)

  def countLegs(a: Animal { def legs: Int }): Int = a.legs
  println(countLegs(cat))
  println(countLegs(fish))
end main
```

A refinement `Animal { def owner: String }` extends `Animal` with an  
additional `owner` member. Any class that extends `Animal` and provides  
`owner` satisfies the refinement without explicit declaration. Refinement  
types are ideal for ad-hoc constraints that are too specific to deserve a  
named trait but too important to leave unenforced.  

## Phantom types

Phantom types use type parameters that carry no runtime data to encode  
state or capability constraints in the type system.  

```scala
sealed trait Unverified
sealed trait Verified

opaque type Email[S] = String

object Email:
  def fromString(s: String): Email[Unverified] = s
  def verify(e: Email[Unverified]): Option[Email[Verified]] =
    if e.contains("@") && e.contains(".") then Some(e) else None
  extension [S](e: Email[S]) def address: String = e

def sendEmail(to: Email[Verified], body: String): Unit =
  println(s"Sending to ${to.address}: $body")

// def sendBad(to: Email[Unverified], body: String): Unit  // different type

@main def main() =
  val raw = Email.fromString("user@example.com")
  Email.verify(raw) match
    case Some(verified) => sendEmail(verified, "Hello!")
    case None           => println("Invalid email")

  val bad = Email.fromString("not-an-email")
  Email.verify(bad) match
    case Some(v) => sendEmail(v, "Hi!")
    case None    => println("Rejected: not-an-email")
end main
```

The type parameter `S` is a phantom: it appears in the type but carries  
no data at runtime since `Email` is opaque over `String`. The `Verified`  
and `Unverified` traits are sealed and uninhabited, existing only to  
distinguish types. `sendEmail` only accepts `Email[Verified]`, so an  
unverified address can never be passed to it without explicit verification.  

## Existential-style patterns in Scala 3

Scala 3 replaces existential types with wildcard arguments, type members,  
and abstract type abstractions.  

```scala
// Wildcard: replaces List[T] forSome { type T }
def printAny(lists: List[List[?]]): Unit =
  lists.foreach(l => println(s"length=${l.length}"))

// Type member: existential hidden in a trait
trait Wrapper:
  type Payload
  val value: Payload
  def describe: String

def makeWrapper[A](a: A, desc: String): Wrapper =
  new Wrapper:
    type Payload = A
    val value: Payload = a
    def describe: String = desc

@main def main() =
  val lists: List[List[?]] =
    List(List(1, 2, 3), List("a", "b"), List(true))
  printAny(lists)

  val w1 = makeWrapper(42,      "an integer")
  val w2 = makeWrapper("hello", "a string")
  val wrappers: List[Wrapper] = List(w1, w2)
  wrappers.foreach(w => println(w.describe))
end main
```

The wildcard `?` in `List[?]` is the Scala 3 spelling of an existential  
type argument. The length of such a list can be queried because `length`  
does not depend on the element type. The type-member approach packages a  
value with an abstract type in a trait, giving controlled access without  
exposing the concrete type to the outside world.  

## Compile-time validation

`inline` methods combined with `compiletime.error` reject invalid  
arguments during compilation rather than at runtime.  

```scala
import scala.compiletime.{error, codeOf}

inline def requireNonEmpty(inline s: String): String =
  inline if s.isEmpty then
    error("String must not be empty at compile time")
  else s

inline def assertPositive(inline n: Int): Int =
  inline if n <= 0 then
    error("Value must be positive")
  else n

@main def main() =
  val msg  = requireNonEmpty("Hello, Scala 3")
  val port = assertPositive(8080)

  println(msg)
  println(port)

  // The following lines would cause compile errors:
  // val bad1 = requireNonEmpty("")
  // val bad2 = assertPositive(-1)
end main
```

`inline if` is evaluated entirely by the compiler when all operands are  
compile-time constants. `error(...)` terminates compilation with a  
diagnostic message before any bytecode is generated. Because the checks  
are erased after validation, the runtime code contains only the valid  
values. This pattern is widely used for SQL query validation, route  
definition, and configuration schema enforcement in Scala 3 libraries.  

## Type-safe builders

Phantom types track which fields of a builder have been supplied,  
turning missing-field errors into compile-time failures.  

```scala
sealed trait WithName
sealed trait WithAge
sealed trait NoName
sealed trait NoAge

class PersonBuilder[N, A] private (
    private val name: Option[String],
    private val age:  Option[Int]
):
  def withName(n: String): PersonBuilder[WithName, A] =
    PersonBuilder(Some(n), age)
  def withAge(a: Int): PersonBuilder[N, WithAge] =
    PersonBuilder(name, Some(a))
  def build(using N =:= WithName, A =:= WithAge): String =
    s"Person(${name.get}, ${age.get})"

object PersonBuilder:
  def apply(): PersonBuilder[NoName, NoAge] =
    new PersonBuilder[NoName, NoAge](None, None)

@main def main() =
  val person = PersonBuilder()
    .withName("Alice")
    .withAge(30)
    .build
  println(person)
  // PersonBuilder().build            // Compile error: missing fields
  // PersonBuilder().withAge(25).build // Compile error: missing name
end main
```

The two phantom type parameters `N` and `A` track whether `withName`  
and `withAge` have been called. `build` requires evidence `N =:= WithName`  
and `A =:= WithAge`, which the compiler can only supply once both  
methods have been invoked. Attempting to call `build` with an incomplete  
builder fails at compile time with a clear missing-evidence error.  

## DSLs using types

Types encode the grammar of a domain-specific language, preventing  
structurally invalid expressions from compiling.  

```scala
sealed trait HtmlTag
sealed trait BlockTag extends HtmlTag
sealed trait InlineTag extends HtmlTag

case class Element[T <: HtmlTag](tag: String, content: String)

def div(content: String): Element[BlockTag]  = Element("div",  content)
def p(content: String): Element[BlockTag]    = Element("p",    content)
def span(content: String): Element[InlineTag] = Element("span", content)
def strong(content: String): Element[InlineTag] = Element("strong", content)

def page(blocks: Element[BlockTag]*): String =
  blocks.map(e => s"<${e.tag}>${e.content}</${e.tag}>").mkString("\n")

def inline_(inlines: Element[InlineTag]*): String =
  inlines.map(e => s"<${e.tag}>${e.content}</${e.tag}>").mkString

@main def main() =
  val doc = page(
    div("Welcome"),
    p(inline_(strong("Scala"), span(" 3 types")))
  )
  println(doc)
  // page(span("bad"))   // Compile error: InlineTag is not BlockTag
end main
```

`page` only accepts `Element[BlockTag]` values, so passing an inline  
element directly is a compile-time error. The type hierarchy mirrors  
the HTML content model, which normally exists only as documentation.  
Encoding it in the type system means the DSL is self-enforcing: an  
HTML document that violates nesting rules cannot be constructed.  

## State machines using types

Type parameters encode the current state of a resource, ensuring that  
operations are only invoked in valid states.  

```scala
sealed trait DoorState
sealed trait Open   extends DoorState
sealed trait Closed extends DoorState

class Door[S <: DoorState] private ():
  def open(using S =:= Closed): Door[Open]   = new Door[Open]
  def close(using S =:= Open): Door[Closed]  = new Door[Closed]
  def peek(using S =:= Open): String         = "You see inside."

object Door:
  def closed: Door[Closed] = new Door[Closed]

@main def main() =
  val d0: Door[Closed] = Door.closed
  val d1: Door[Open]   = d0.open
  println(d1.peek)
  val d2: Door[Closed] = d1.close
  // d0.close   // Compile error: Closed is not Open
  // d0.peek    // Compile error: Closed is not Open
  println("State machine transitions verified")
end main
```

The type parameter `S` tracks whether the door is `Open` or `Closed`.  
`open` requires evidence `S =:= Closed`, which only the compiler can  
provide when the door variable is statically known to be closed. Calling  
`close` on an already-closed door, or `peek` through a closed door, fails  
at compile time. This approach is used to model connections, sessions,  
file handles, and protocol states in type-safe resource APIs.  

## Domain modeling with types

A rich domain model combines opaque types, union types, and sealed  
hierarchies to make illegal business states unrepresentable.  

```scala
opaque type OrderId    = Int
opaque type CustomerId = String
opaque type Amount     = Double

object OrderId:
  def apply(n: Int): OrderId        = n
  extension (id: OrderId) def value: Int = id

object CustomerId:
  def apply(s: String): CustomerId       = s
  extension (id: CustomerId) def value: String = id

object Amount:
  def apply(d: Double): Option[Amount] =
    if d >= 0 then Some(d) else None
  extension (a: Amount) def value: Double = a

sealed trait OrderStatus
case object Draft     extends OrderStatus
case object Submitted extends OrderStatus
case object Shipped   extends OrderStatus

case class Order(id: OrderId, customer: CustomerId,
                 total: Amount, status: OrderStatus)

def submit(o: Order): Order =
  require(o.status == Draft, "Only draft orders can be submitted")
  o.copy(status = Submitted)

@main def main() =
  val order = for
    total <- Amount(99.99)
  yield Order(OrderId(1), CustomerId("C-01"), total, Draft)
  order.foreach: o =>
    val submitted = submit(o)
    println(s"Order ${submitted.id.value}: ${submitted.status}")
end main
```

`OrderId` and `CustomerId` are both distinct types despite being backed  
by primitive values, so they cannot be accidentally swapped. `Amount`  
validates non-negativity at construction time and returns `Option` to  
force callers to handle the invalid case. The sealed `OrderStatus`  
hierarchy makes exhaustive pattern matching on status changes mandatory.  
Together these techniques produce a domain model where the compiler  
enforces a significant portion of the business rules.  

## Idiomatic advanced type usage in Scala 3

Idiomatic Scala 3 combines multiple advanced type features into concise,  
expressive APIs that are safe, composable, and maintainable.  

```scala
opaque type Tag = String
object Tag:
  def apply(s: String): Tag            = s.trim.toLowerCase
  extension (t: Tag) def value: String = t

trait Taggable[A]:
  def tags(a: A): List[Tag]

case class Article(title: String, body: String, keywords: List[String])
object Article:
  given Taggable[Article] with
    def tags(a: Article): List[Tag] = a.keywords.map(Tag(_))

def search[A: Taggable](items: List[A], query: Tag): List[A] =
  items.filter(summon[Taggable[A]].tags(_).contains(query))

type SearchResult[A] = A match
  case Article => List[Article]
  case _       => List[Any]

@main def main() =
  import Article.given
  val articles = List(
    Article("Scala 3", "Dotty features", List("scala", "fp")),
    Article("Rust",    "Memory safety",  List("rust", "systems")),
    Article("FP Tour", "Monads etc",     List("fp", "haskell"))
  )
  val results = search(articles, Tag("fp"))
  results.foreach(a => println(a.title))
end main
```

`Tag` is opaque so raw strings and tags cannot be mixed at call sites.  
The `Taggable` typeclass separates the tagging concern from the data  
type. `search` is polymorphic over any type that has a `Taggable`  
instance, and the match type `SearchResult` documents the return type  
relationship inline. This example shows how union types, opaque types,  
typeclasses, context bounds, and match types combine to produce code that  
is simultaneously concise, safe, and open to extension.  

## Literal types

Literal types treat specific compile-time constant values as distinct  
types, enabling finer-grained constraints than their base types.  

```scala
import scala.compiletime.ops.int.*

def httpStatus(code: 200 | 201 | 400 | 404 | 500): String =
  code match
    case 200 => "OK"
    case 201 => "Created"
    case 400 => "Bad Request"
    case 404 => "Not Found"
    case 500 => "Internal Server Error"

def clamp[N <: Int](n: N)(using (N >= 0) =:= true,
                                 (N <= 100) =:= true): N = n

@main def main() =
  println(httpStatus(200))
  println(httpStatus(404))
  // println(httpStatus(418))  // Compile error: 418 is not in the union

  val pct: 75 = 75
  println(clamp(75))
  // clamp(200)  // Compile error: 200 >= 0 && 200 <= 100 is false

  val flag: true  = true
  val zero: 0     = 0
  val hello: "hi" = "hi"
  println(s"$flag $zero $hello")
end main
```

A literal type such as `200` is a subtype of `Int` that contains exactly  
one value. Using a union of literals as a parameter type restricts the  
accepted values at compile time without any runtime validation. The  
`compiletime.ops.int` operations produce `Boolean` literal types from  
comparisons, which can be used as `summon` constraints to verify  
numeric bounds statically.  

## Singleton types

Singleton types capture the exact type of a specific object instance,  
allowing the compiler to track identity through method calls.  

```scala
object Red
object Green
object Blue

type Color = Red.type | Green.type | Blue.type

def hex(c: Color): String =
  c match
    case Red   => "#FF0000"
    case Green => "#00FF00"
    case Blue  => "#0000FF"

def identity[A](a: A): a.type = a

@main def main() =
  val r: Red.type   = Red
  val g: Green.type = Green

  println(hex(r))
  println(hex(g))

  val s: "hello" = "hello"
  val s2: s.type = identity(s)     // s2 has type s.type = "hello"
  println(s2.length)

  // Singleton types preserve object identity
  val x = List(1, 2, 3)
  val y: x.type = identity(x)
  println(x eq y)
end main
```

`Red.type` is the singleton type of the `Red` object, inhabited by  
exactly one value. The union `Red.type | Green.type | Blue.type` is  
a closed set of colours checked exhaustively by the compiler. The  
method `identity` returns `a.type`, which preserves the exact type of  
its argument through the call boundary. Singleton types are foundational  
to type-level programming with literal values and to path-dependent types.  

## Polymorphic function types

Scala 3 introduces polymorphic function types that quantify over type  
parameters, making universal functions first-class values.  

```scala
// Polymorphic function: works for any type A
val identity: [A] => A => A =
  [A] => (x: A) => x

// Swap any pair
val swap: [A, B] => (A, B) => (B, A) =
  [A, B] => (a: A, b: B) => (b, a)

// Apply a function twice
val twice: [A] => (A => A) => A => A =
  [A] => (f: A => A) => (x: A) => f(f(x))

// Accept a polymorphic function as a parameter
def applyAll[A](f: [X] => X => X, values: List[A]): List[A] =
  values.map(f(_))

@main def main() =
  println(identity(42))
  println(identity("hello"))
  println(swap(1, "one"))
  println(swap(true, 3.14))
  println(twice[Int](_ + 1)(0))
  println(twice[String](_ + "!")("wow"))
  println(applyAll(identity, List(10, 20, 30)))
end main
```

`[A] => A => A` is a polymorphic function type. Values of this type carry  
their own type parameter, so `identity` can be applied to `Int` and  
`String` without separate overloads. Before Scala 3, this required a SAM  
trait or a method; now it is a proper first-class type that can be stored  
in variables, returned from methods, and placed in collections.  

## Type lambdas

Type lambdas express partially applied type constructors as anonymous  
type-level functions, replacing the `({type λ[X] = F[A, X]})#λ` pattern.  

```scala
// Type lambda: partially apply Either with String as the left type
type StringEither = [X] =>> Either[String, X]

// Partially apply Map with String keys
type StringMap = [V] =>> Map[String, V]

// Use type lambdas as higher-kinded type arguments
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

given Functor[Option] with
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)

given Functor[List] with
  def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)

given Functor[[X] =>> Either[String, X]] with
  def map[A, B](fa: Either[String, A])(f: A => B): Either[String, B] =
    fa.map(f)

def lift[F[_]: Functor, A, B](fa: F[A])(f: A => B): F[B] =
  summon[Functor[F]].map(fa)(f)

@main def main() =
  val e1: StringEither[Int]     = Right(42)
  val e2: StringEither[String]  = Left("error")
  val m1: StringMap[Int]        = Map("a" -> 1, "b" -> 2)

  println(e1)
  println(e2)
  println(m1)

  println(lift[Option](Some(10))(_ * 3))
  println(lift[[X] =>> Either[String, X]](Right(5): Either[String, Int])(_ + 1))
end main
```

`[X] =>> Either[String, X]` is a type lambda. It is equivalent to a  
type alias `type StringEither[X] = Either[String, X]` but can be written  
inline wherever a type constructor of kind `* -> *` is expected. Type  
lambdas are essential when working with higher-kinded abstractions such  
as functors, monads, and applicatives where the type constructor must be  
partially applied.  
