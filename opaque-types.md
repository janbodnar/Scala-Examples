# Opaque Types in Scala 3

Scala 3 introduces opaque types as a first-class language feature that lets  
you define a new named type whose underlying representation is completely  
hidden from external code. The `opaque type` keyword creates a restricted  
alias: the real type is visible only inside the defining scope — the  
companion object or the file where it is declared — while all other code  
sees a fully abstract, distinct type. This boundary gives you compile-time  
type safety without any runtime overhead.  

A plain type alias (`type Alias = String`) is transparent everywhere: the  
compiler treats the alias and its underlying type as interchangeable in all  
call sites. An opaque type breaks this transparency. Outside its defining  
scope `Alias` and `String` remain separate types, and the compiler enforces  
that distinction at every use. You may only create or inspect values of an  
opaque type through the constructors and accessors that the author explicitly  
provides.  

Value classes (`class Dollars(val amount: Double) extends AnyVal`) were the  
pre-Scala-3 way of adding a type wrapper with reduced overhead. They carry  
several restrictions: they cannot be nested, they are subject to boxing when  
used with generics or collections, and they inherit from `AnyVal`. Opaque  
types have none of those limitations. A value of an opaque type and a value  
of its underlying type occupy exactly the same memory and are accessed with  
identical bytecode — no allocation, no boxing, no virtual dispatch.  

The transparency rule is precise. Inside the companion object (or the  
top-level file scope) that defines an opaque type the alias is fully  
transparent: you can freely convert to and from the underlying type. Outside  
that boundary the type is opaque, and no implicit coercion is available  
unless you provide one deliberately through a `given Conversion`.  

Because the wrapper erases completely, opaque types carry zero runtime cost.  
A `Dollars` value and a `Double` value occupy identical heap space and are  
loaded by the same JVM instruction. The performance characteristics of an  
opaque type are therefore identical to those of the underlying type,  
regardless of whether the opaque type appears in a generic class, a  
collection, or a higher-order function.  

Opaque types are the right tool when you want to prevent mixing up values of  
the same underlying type (metres vs. seconds), encode domain invariants at  
compile time (only positive numbers, non-empty strings), hide an  
implementation type from library consumers, or build a safe, ergonomic API  
without sacrificing performance. They scale from single-file utilities to  
large library boundaries.  

## A simple opaque type

A minimal opaque type wraps a primitive and hides it from external code.  

```scala
object Length:
  opaque type Metre = Double
  def apply(d: Double): Metre = d
  extension (m: Metre)
    def value: Double = m

@main def main() =
  val dist = Length(5.0)
  println(s"Distance: ${dist.value} m")
  // val d: Double = dist  // compile error outside Length
end main
```

The `Metre` type is defined inside the `Length` object.  Outside that object  
callers can only create a `Metre` through `Length.apply` and read it back  
through the `value` extension.  There is no way to assign a raw `Double` to  
a `Metre` variable from outside, preventing silent unit confusion.  The  
runtime representation is a plain `Double` — no heap allocation occurs.  

## Opaque type vs type alias

An opaque type enforces a boundary that a plain type alias cannot.  

```scala
type Alias = String

object Opaque:
  opaque type Tag = String
  def apply(s: String): Tag = s
  extension (t: Tag)
    def unwrap: String = t

@main def main() =
  val a: Alias  = "hello"
  val s: String = a           // OK — alias is fully transparent
  val t = Opaque("world")
  // val s2: String = t       // compile error: Tag is not String
  println(s"alias: $a, opaque: ${t.unwrap}")
end main
```

The type alias `Alias` is completely transparent: the compiler considers  
`Alias` and `String` identical everywhere, so `val s: String = a` compiles  
without issue.  By contrast, `Opaque.Tag` is opaque: attempting to assign a  
`Tag` to a `String` variable from outside the `Opaque` object is a  
compilation error.  The only legal way to access the wrapped `String` is  
through the explicitly provided `unwrap` extension method.  This gives you  
real type safety, not just documentation.  

## Opaque type inside an object

Grouping related opaque types inside one object keeps them isolated.  

```scala
object Canvas:
  opaque type Width  = Int
  opaque type Height = Int
  def width(n: Int):  Width  = n
  def height(n: Int): Height = n
  def area(w: Width, h: Height): Int = w * h

@main def main() =
  val w = Canvas.width(800)
  val h = Canvas.height(600)
  println(s"Area: ${Canvas.area(w, h)} px²")
  // Canvas.area(h, w)  // compile error: arguments are swapped
end main
```

Both `Width` and `Height` are backed by `Int`, but the compiler treats them  
as distinct types.  The function `area` requires its arguments in a specific  
order, and any accidental swap is caught at compile time rather than  
producing a silently wrong result at runtime.  Inside `Canvas` the two types  
are both transparent as `Int`, so arithmetic such as `w * h` compiles  
without explicit conversion.  

## Opaque type at the top level

Scala 3 allows opaque types to be declared at the package (file) level.  

```scala
opaque type PackageId = Long

object PackageId:
  def apply(n: Long): PackageId = n
  extension (id: PackageId)
    def value: Long = id

@main def main() =
  val pid = PackageId(1001L)
  println(s"Package ID: ${pid.value}")
end main
```

A top-level opaque type is transparent throughout the entire file that  
declares it.  The companion object `PackageId` provides the constructor and  
the extension method that exposes the wrapped `Long`.  Code in other files  
sees only the abstract `PackageId` type, with no knowledge of the underlying  
`Long`.  This pattern is the Scala 3 idiomatic replacement for package  
objects when you need file-scoped type definitions.  

## Companion methods for construction

The companion object is the natural place to put smart constructors.  

```scala
object Username:
  opaque type Name = String
  def apply(s: String): Name = s.trim.toLowerCase
  extension (n: Name)
    def display: String = n.capitalize
    def raw: String     = n

@main def main() =
  val u = Username("  Alice  ")
  println(s"raw:     ${u.raw}")
  println(s"display: ${u.display}")
end main
```

The `apply` method is the only public path to a `Name` value.  It trims  
whitespace and lowercases the input before storing it, so all `Name`  
instances are guaranteed to be in a canonical form.  External code cannot  
create a `Name` from a raw `String` without going through this constructor,  
ensuring the invariant is always maintained.  Extension methods then expose  
safe read-only operations on the opaque value.  

## Companion methods for validation

Smart constructors can return `Either` to report validation failures.  

```scala
object Score:
  opaque type Points = Int

  def from(n: Int): Either[String, Points] =
    if n >= 0 && n <= 100 then Right(n)
    else Left(s"Score must be 0–100, got $n")

  extension (p: Points)
    def value: Int = p
    def grade: String =
      if p >= 90 then "A"
      else if p >= 80 then "B"
      else if p >= 70 then "C"
      else "F"

@main def main() =
  Score.from(85) match
    case Right(s) => println(s"Grade: ${s.grade}")
    case Left(e)  => println(s"Error: $e")
  Score.from(110) match
    case Right(s) => println(s"Grade: ${s.grade}")
    case Left(e)  => println(s"Error: $e")
end main
```

The factory method `from` checks whether the integer lies in the valid range  
and wraps a success in `Right` or an error message in `Left`.  Callers are  
forced to handle both branches, making invalid scores unrepresentable at  
runtime.  The `grade` extension method operates only on `Points` that have  
already passed validation, so its logic never needs a bounds check.  

## Runtime representation equality

An opaque type and its underlying type are identical at runtime.  

```scala
object Wrapper:
  opaque type Wrapped = Int
  def apply(n: Int): Wrapped = n
  extension (w: Wrapped)
    def toInt: Int = w

@main def main() =
  val w = Wrapper(42)
  val n = 42
  println(w.toInt == n)               // true
  println(w.toInt.getClass.getName)   // int
  println(n.getClass.getName)         // int
end main
```

The comparison `w.toInt == n` returns `true` because both values are plain  
`Int` at the JVM level.  There is no boxing, no wrapper object, and no  
extra indirection.  Calling `getClass` on both confirms they share the same  
runtime class.  The opaque type boundary exists entirely in the type checker  
and is erased before bytecode is generated.  

## Compile-time abstraction boundaries

Opaque types from different objects are mutually incompatible.  

```scala
object Celsius:
  opaque type Temp = Double
  def apply(d: Double): Temp = d
  extension (t: Temp)
    def value: Double = t

object Fahrenheit:
  opaque type Temp = Double
  def apply(d: Double): Temp = d
  extension (t: Temp)
    def value: Double = t

@main def main() =
  val c = Celsius(100.0)
  val f = Fahrenheit(212.0)
  // c + f  // compile error: Celsius.Temp and Fahrenheit.Temp differ
  println(s"${c.value} °C  /  ${f.value} °F")
end main
```

Both `Celsius.Temp` and `Fahrenheit.Temp` are backed by `Double`, but they  
live in separate companion scopes.  From outside those scopes they are  
completely unrelated types, and any expression that tries to mix them is  
rejected by the compiler.  This statically prevents classic temperature-unit  
bugs — the kind that have caused real-world engineering failures — at zero  
runtime cost.  

## Preventing accidental misuse

Giving distinct opaque types to physically similar quantities eliminates a  
whole class of argument-order bugs.  

```scala
object Physics:
  opaque type Metres  = Double
  opaque type Seconds = Double
  opaque type Speed   = Double

  def metres(d: Double): Metres   = d
  def seconds(d: Double): Seconds = d
  def speed(m: Metres, s: Seconds): Speed = m / s

  extension (v: Speed)
    def value: Double = v

@main def main() =
  val dist = Physics.metres(100.0)
  val time = Physics.seconds(9.58)
  val spd  = Physics.speed(dist, time)
  println(f"Speed: ${spd.value}%.2f m/s")
  // Physics.speed(time, dist)  // compile error: arguments swapped
end main
```

All three types (`Metres`, `Seconds`, `Speed`) are `Double` underneath, so  
they have identical performance.  Yet the compiler treats them as entirely  
separate types.  Passing `time` where `dist` is expected is a type error,  
not a silent runtime bug.  This pattern is especially valuable in scientific  
or financial code where quantities of the same primitive type carry different  
semantic meaning.  

## Hiding implementation details

An opaque type can conceal sensitive or unstable implementation choices.  

```scala
object TokenStore:
  opaque type Token = String

  private def generate(): String =
    java.util.UUID.randomUUID().toString

  def create(): Token = generate()

  extension (t: Token)
    def header: String = s"Bearer $t"

@main def main() =
  val tok = TokenStore.create()
  println(tok.header)
end main
```

Callers receive a `Token` value but have no way to inspect, forge, or  
manipulate the underlying UUID string.  The only operation they can perform  
is reading the `header` representation.  If the implementation later  
switches from UUID strings to signed JWTs, the external API remains  
unchanged and existing call sites continue to compile.  Hiding the  
representation through an opaque type enforces encapsulation without  
performance cost.  

## Opaque types for IDs

Different ID types for different entities prevent accidental cross-use.  

```scala
object Domain:
  opaque type UserId    = Long
  opaque type ProductId = Long

  def userId(n: Long): UserId       = n
  def productId(n: Long): ProductId = n

  extension (id: UserId)
    def raw: Long = id

  extension (id: ProductId)
    def raw: Long = id

@main def main() =
  val uid = Domain.userId(1L)
  val pid = Domain.productId(1L)
  println(s"User: ${uid.raw}, Product: ${pid.raw}")
  // val wrong: Domain.UserId = pid  // compile error
end main
```

Both `UserId` and `ProductId` wrap a `Long`, and both even hold the value  
`1L` in this example.  Nevertheless the compiler refuses to treat one as the  
other, so passing a product ID to a function that expects a user ID is a  
compile error.  This pattern is ubiquitous in domain-driven design where  
multiple entity types share the same primitive representation but must never  
be mixed.  

## Opaque types for units of measure

Separating physical units at the type level prevents dimensional errors.  

```scala
object Units:
  opaque type Kilogram = Double
  opaque type Litre    = Double

  def kg(d: Double): Kilogram = d
  def L(d: Double):  Litre    = d

  def sumKg(a: Kilogram, b: Kilogram): Kilogram = a + b
  def sumL(a: Litre, b: Litre):        Litre    = a + b

  extension (k: Kilogram)
    def value: Double = k

  extension (l: Litre)
    def value: Double = l

@main def main() =
  val flour = Units.kg(0.5)
  val sugar = Units.kg(0.25)
  val water = Units.L(1.0)
  val total = Units.sumKg(flour, sugar)
  println(s"Flour + sugar: ${total.value} kg")
  println(s"Water: ${water.value} L")
  // Units.sumKg(flour, water)  // compile error: Kilogram ≠ Litre
end main
```

The helper functions `sumKg` and `sumL` accept only matching units.  Trying  
to add `Kilogram` and `Litre` is a type error, not a runtime exception.  
Inside the `Units` object both types are transparent as `Double`, so the  
addition `a + b` calls the ordinary `Double.+` method with no overhead.  
The pattern scales naturally to full unit-of-measure libraries.  

## Opaque types for constrained strings

An opaque type can enforce a string invariant at construction time.  

```scala
object NonEmpty:
  opaque type NEString = String

  def from(s: String): Option[NEString] =
    if s.nonEmpty then Some(s) else None

  extension (n: NEString)
    def value: String = n
    def length: Int   = n.length

@main def main() =
  NonEmpty.from("hello") match
    case Some(s) => println(s"'${s.value}' len=${s.length}")
    case None    => println("Empty string rejected")
  NonEmpty.from("") match
    case Some(s) => println(s"'${s.value}'")
    case None    => println("Empty string rejected")
end main
```

The factory `from` returns `None` for empty strings and `Some` for valid  
ones, wrapping the string in the opaque `NEString` type on success.  Any  
code holding a `NEString` value is statically guaranteed that the string is  
non-empty — no guard clause is needed at the use site.  This pattern  
eliminates an entire category of null-or-empty defensive checks that would  
otherwise be scattered throughout the codebase.  

## Opaque types for constrained numbers

Numeric invariants can be encoded directly in the type.  

```scala
object Positive:
  opaque type PosInt = Int

  def from(n: Int): Option[PosInt] =
    if n > 0 then Some(n) else None

  extension (p: PosInt)
    def value: Int    = p
    def doubled: PosInt = p * 2

@main def main() =
  Positive.from(5) match
    case Some(p) =>
      println(s"Positive: ${p.value}")
      println(s"Doubled:  ${p.doubled.value}")
    case None =>
      println("Not positive")
  Positive.from(-3) match
    case Some(p) => println(s"Got: ${p.value}")
    case None    => println("Negative rejected")
end main
```

The type `PosInt` can only be created through `from`, which rejects  
non-positive integers at the boundary.  The `doubled` extension multiplies  
the value by two; inside the `Positive` object the multiplication `p * 2`  
is plain `Int` arithmetic, so it compiles and runs at full speed.  Any  
function that accepts a `PosInt` argument can safely assume the value is  
positive without performing its own range check.  

## Opaque types for enum-like domains

A set of named constants can form a safe domain type.  

```scala
object Direction:
  opaque type Dir = String

  val North: Dir = "North"
  val South: Dir = "South"
  val East:  Dir = "East"
  val West:  Dir = "West"

  val all: List[Dir] = List(North, South, East, West)

  extension (d: Dir)
    def opposite: Dir =
      d match
        case "North" => South
        case "South" => North
        case "East"  => West
        case _       => East
    def label: String = d

@main def main() =
  println(Direction.North.label)
  println(Direction.North.opposite.label)
  println(Direction.all.map(_.label).mkString(", "))
end main
```

The four constants are the only `Dir` values that external code can obtain;  
there is no public constructor that would accept an arbitrary string.  
Inside `Direction` the underlying `String` is transparent, which lets the  
`opposite` method use a direct string match without an explicit cast.  This  
technique works well for small, closed domains where a full sealed enum  
would be heavier than necessary.  

## Extension methods for opaque types

Extension methods define a rich API on an opaque type without boxing.  

```scala
object Circle:
  opaque type Radius = Double

  def apply(d: Double): Radius = d

  extension (r: Radius)
    def value: Double    = r
    def diameter: Double = r * 2
    def area: Double     = math.Pi * r * r
    def >(other: Radius): Boolean = r > other

@main def main() =
  val r1 = Circle(5.0)
  val r2 = Circle(3.0)
  println(f"area:     ${r1.area}%.2f")
  println(f"diameter: ${r1.diameter}%.2f")
  println(s"r1 > r2:  ${r1 > r2}")
end main
```

The extension block attached to the companion object exposes `diameter`,  
`area`, and a comparison operator.  From the call site the syntax  
`r1.diameter` looks like a normal method call on a concrete type.  Inside  
the extension body `r` is a plain `Double`, so `r * 2` and `math.Pi * r * r`  
compile to ordinary floating-point multiplication — no virtual dispatch  
and no object wrapping.  

## Operators on opaque types

Standard arithmetic operators can be defined for opaque types.  

```scala
object Money:
  opaque type USD = BigDecimal

  def apply(amount: BigDecimal): USD = amount

  extension (a: USD)
    def +(b: USD): USD         = (a: BigDecimal) + b
    def -(b: USD): USD         = (a: BigDecimal) - b
    def *(factor: Double): USD = (a: BigDecimal) * factor
    def show: String           = f"$$${(a: BigDecimal).toDouble}%.2f"

@main def main() =
  val price      = Money(BigDecimal("19.99"))
  val tax        = Money(BigDecimal("1.60"))
  val total      = price + tax
  val discounted = total * 0.9
  println(total.show)
  println(discounted.show)
end main
```

The explicit type ascription `(a: BigDecimal)` inside each extension body  
makes it unambiguous that the operation is delegated to `BigDecimal`'s own  
arithmetic, not called recursively on the extension.  Outside the `Money`  
object callers use natural infix syntax: `price + tax`, `total * 0.9`.  The  
`show` method formats the value as a dollar amount.  Only `USD` operands are  
accepted, so mixing currencies or raw `BigDecimal` values is a type error.  

## Opaque types with givens

A given instance can describe how an opaque type participates in abstractions.  

```scala
object Priority:
  opaque type Level = Int

  def apply(n: Int): Level = n

  given Ordering[Level] with
    def compare(a: Level, b: Level): Int = a.compareTo(b)

  extension (l: Level)
    def value: Int = l

@main def main() =
  val levels = List(3, 1, 4, 1, 5, 9, 2, 6).map(Priority(_))
  val sorted  = levels.sorted
  println(sorted.map(_.value).mkString(", "))
end main
```

The `given Ordering[Level]` instance lives inside the `Priority` companion  
and is therefore in the implicit scope whenever `Level` appears in a type  
position.  Calling `levels.sorted` triggers implicit lookup, finds the  
ordering, and sorts the list without any explicit import.  Inside  
`compare`, `a` and `b` are transparent as `Int`, so `a.compareTo(b)` calls  
the standard `Int` comparison method.  

## Opaque types with typeclasses

A typeclass instance for an opaque type integrates it with generic code.  

```scala
trait Printable[A]:
  def print(a: A): String

object Celsius:
  opaque type Temp = Double

  def apply(d: Double): Temp = d

  given Printable[Temp] with
    def print(t: Temp): String = f"$t%.1f °C"

  extension (t: Temp)
    def value: Double = t

def show[A](a: A)(using p: Printable[A]): String = p.print(a)

@main def main() =
  val t = Celsius(37.0)
  println(show(t))
end main
```

The `Printable[Temp]` instance is defined inside `Celsius`, placing it in  
the implicit scope for `Celsius.Temp` without any manual import.  The  
generic `show` function works with any `Printable[A]`, and the compiler  
automatically selects the right instance when called with a `Temp` argument.  
Inside the `print` body, `t` is a transparent `Double`, so the `f` string  
interpolation formats it as a floating-point number.  

## Opaque types with inline methods

Inline methods on opaque types allow the compiler to specialise call sites.  

```scala
object Bits:
  opaque type Flags = Int

  val empty: Flags = 0
  def apply(bits: Int): Flags = bits

  inline def set(f: Flags, bit: Int): Flags    = f | (1 << bit)
  inline def clear(f: Flags, bit: Int): Flags  = f & ~(1 << bit)
  inline def test(f: Flags, bit: Int): Boolean = (f & (1 << bit)) != 0

  extension (f: Flags)
    def value: Int = f

@main def main() =
  var flags = Bits.empty
  flags = Bits.set(flags, 0)
  flags = Bits.set(flags, 2)
  println(s"flags = ${flags.value}")          // 5
  println(s"bit 0: ${Bits.test(flags, 0)}")  // true
  println(s"bit 1: ${Bits.test(flags, 1)}")  // false
  flags = Bits.clear(flags, 0)
  println(s"after clear: ${flags.value}")     // 4
end main
```

Marking `set`, `clear`, and `test` as `inline` instructs the compiler to  
expand their bodies at every call site, just like a macro.  The bitwise  
expressions become literal integer operations in the emitted bytecode  
rather than method calls, giving performance equivalent to hand-written  
low-level code.  The `Flags` type still provides full type safety: you  
cannot accidentally pass a raw `Int` where `Flags` is expected from outside  
the `Bits` object.  

## Parameterized opaque types

Opaque types accept type parameters, enabling generic wrappers.  

```scala
object Validated:
  opaque type Valid[A] = A

  def apply[A](a: A): Valid[A] = a

  extension [A](v: Valid[A])
    def get: A = v
    def map[B](f: A => B): Valid[B] = f(v)

@main def main() =
  val validInt = Validated(42)
  val validStr = Validated("hello")
  val doubled  = validInt.map(_ * 2)
  println(s"int:     ${validInt.get}")
  println(s"str:     ${validStr.get}")
  println(s"doubled: ${doubled.get}")
end main
```

`Valid[A]` is a generic opaque alias for `A`.  The type parameter `A` is  
preserved through operations: `map` transforms a `Valid[A]` into a  
`Valid[B]` by applying a function, and the result is still wrapped in the  
opaque type.  Extension methods on generic opaque types use the same  
bracket syntax `[A]` before the receiver parameter.  This pattern is a  
zero-cost version of the classic `Newtype[A]` pattern from functional  
programming.  

## Opaque types with type parameters and bounds

A bound on the type parameter of an opaque type constrains its use.  

```scala
object Sorted:
  opaque type SortedList[A] = List[A]

  def apply[A: Ordering](xs: List[A]): SortedList[A] = xs.sorted

  extension [A](sl: SortedList[A])
    def toList: List[A]     = sl
    def head: A             = sl.head
    def tail: SortedList[A] = sl.tail
    def size: Int           = sl.size

@main def main() =
  val nums = Sorted(List(3, 1, 4, 1, 5, 9))
  println(nums.toList)   // List(1, 1, 3, 4, 5, 9)
  println(nums.head)     // 1
  val strs = Sorted(List("banana", "apple", "cherry"))
  println(strs.toList)   // List(apple, banana, cherry)
end main
```

The `apply` constructor requires an `Ordering[A]` context bound to sort the  
list.  Once a `SortedList[A]` exists, that invariant — the list is sorted —  
is encoded in the type and cannot be broken from outside.  The extension  
methods expose read-only list operations.  The `tail` method returns  
`SortedList[A]` rather than `List[A]`, preserving the sorted guarantee for  
the remainder of the list.  

## Opaque types with match types

A match type can describe the encoding relationship of an opaque type.  

```scala
type Describe[A] = A match
  case Int     => String
  case Double  => String
  case Boolean => String
  case _       => String

object Labeled:
  opaque type Label[A] = String

  def from[A](a: A): Label[A] = a.toString

  extension [A](l: Label[A])
    def value: String           = l
    def prefixed(p: String): Label[A] = s"$p:$l"

@main def main() =
  val intLbl    = Labeled.from(42)
  val doubleLbl = Labeled.from(3.14)
  val n: Describe[Int] = intLbl.value   // Describe[Int] reduces to String
  println(intLbl.value)
  println(doubleLbl.prefixed("pi").value)
  println(n)
end main
```

The match type `Describe[A]` reduces to `String` for every branch — it  
documents that whatever input type is given, the label is stored as a  
`String`.  Inside `Labeled` the opaque alias `Label[A] = String` is  
transparent, so `a.toString` is stored directly.  The assignment  
`val n: Describe[Int] = intLbl.value` compiles because `Describe[Int]`  
reduces to `String`, confirming that the match type and the opaque  
type describe the same underlying representation.  

## Opaque types with union types

The underlying type of an opaque alias can be a union type.  

```scala
object Input:
  opaque type Value = Int | String | Boolean

  def fromInt(n: Int):     Value = n
  def fromStr(s: String):  Value = s
  def fromBool(b: Boolean): Value = b

  extension (v: Value)
    def render: String =
      v match
        case n: Int     => s"int($n)"
        case s: String  => s"str($s)"
        case b: Boolean => s"bool($b)"

@main def main() =
  val values: List[Input.Value] = List(
    Input.fromInt(1),
    Input.fromStr("hello"),
    Input.fromBool(true)
  )
  values.foreach(v => println(v.render))
end main
```

The union type `Int | String | Boolean` groups three distinct runtime  
representations under a single opaque name.  From outside the `Input`  
object callers only see an abstract `Value` type with no knowledge of the  
union structure.  Inside the companion the pattern match can discriminate  
between branches because the underlying union is transparent there.  This  
technique creates a closed, named variant type without defining a full sealed  
hierarchy.  

## Opaque types with intersection types

An intersection type can be the underlying representation of an opaque alias.  

```scala
trait Named:
  def name: String

trait Scored:
  def score: Int

object Player:
  opaque type Entity = Named & Scored

  def apply(n: String, s: Int): Entity =
    new Named with Scored:
      def name: String = n
      def score: Int   = s

  extension (e: Entity)
    def name: String  = (e: Named).name
    def score: Int    = (e: Scored).score
    def describe: String = s"${(e: Named).name}: ${(e: Scored).score} pts"

@main def main() =
  val p = Player("Alice", 850)
  println(p.describe)
  println(p.name)
  println(p.score)
end main
```

The underlying type `Named & Scored` is an anonymous class that satisfies  
both traits simultaneously.  From outside `Player`, callers work only with  
the abstract `Entity` type, unaware that the intersection type exists.  The  
explicit ascriptions `(e: Named)` and `(e: Scored)` inside the extension  
bodies resolve the intersection members without ambiguity.  Intersection  
types as underlying representations are useful when you need to guarantee  
that a value satisfies multiple independent contracts.  

## Opaque types with supertype bounds

A supertype bound on an opaque type allows limited coercion outside the scope.  

```scala
object Bounds:
  opaque type PositiveDouble <: Double = Double

  def apply(d: Double): Option[PositiveDouble] =
    if d > 0.0 then Some(d) else None

  def unsafe(d: Double): PositiveDouble = d

@main def main() =
  Bounds(3.14) match
    case Some(p) =>
      val d: Double = p   // OK: PositiveDouble <: Double
      println(f"Positive: $d%.2f")
    case None =>
      println("Non-positive rejected")
  Bounds(-1.0) match
    case Some(p) => println(s"Got: $p")
    case None    => println("Non-positive rejected")
  val x: Double = Bounds.unsafe(0.5)
  println(s"Via Double: $x")
end main
```

The declaration `opaque type PositiveDouble <: Double` makes `PositiveDouble`  
a subtype of `Double` everywhere, not just inside the companion.  This means  
a `PositiveDouble` value can be assigned to a `Double` variable or passed to  
any function expecting a `Double` without an explicit accessor call.  The  
opposite direction — assigning a raw `Double` to `PositiveDouble` — remains  
forbidden outside the companion.  The bound is useful when the opaque type  
is a refinement of the underlying type and you want callers to benefit from  
that hierarchy.  

## Opaque types in APIs

Opaque types make public API surfaces clear and resistant to misuse.  

```scala
object HttpApi:
  opaque type Status = Int
  opaque type Path   = String

  val Ok:        Status = 200
  val NotFound:  Status = 404
  val ServerErr: Status = 500

  def status(n: Int): Status = n
  def path(s: String): Path  = s

  extension (s: Status)
    def isSuccess: Boolean = s >= 200 && s < 300
    def code: Int          = s

  extension (p: Path)
    def value: String          = p
    def /(segment: String): Path = s"$p/$segment"

@main def main() =
  val apiPath = HttpApi.path("/api") / "users" / "42"
  val code    = HttpApi.Ok
  println(s"${apiPath.value} -> ${code.code}")
  println(s"success: ${code.isSuccess}")
  println(s"404 ok:  ${HttpApi.NotFound.isSuccess}")
end main
```

Consumers of `HttpApi` work with named constants (`Ok`, `NotFound`) and the  
path-building DSL, never with raw integers or strings.  The `/` operator  
on `Path` chains segments into a well-formed path string.  Importing a raw  
integer where a `Status` is expected is a compile error, so HTTP code  
arithmetic cannot silently produce nonsensical status values.  Adding new  
status constants or changing the internal path representation requires no  
changes to call sites.  

## Opaque types in collections

An opaque type can wrap a collection, hiding its concrete type.  

```scala
object Tag:
  opaque type TagName = String
  opaque type TagSet  = Set[TagName]

  def apply(s: String): TagName = s.toLowerCase.trim

  val emptySet: TagSet = Set.empty
  def tagSet(ts: TagName*): TagSet = ts.toSet

  extension (t: TagName)
    def value: String = t

  extension (ts: TagSet)
    def add(t: TagName): TagSet        = ts + t
    def remove(t: TagName): TagSet     = ts - t
    def has(t: TagName): Boolean       = ts.contains(t)
    def toList: List[String]           = ts.toList.map(_.value)

@main def main() =
  val t1 = Tag("Scala")
  val t2 = Tag("  FP  ")
  val t3 = Tag("JVM")
  var ts = Tag.tagSet(t1, t2)
  ts = ts.add(t3)
  println(ts.toList.sorted.mkString(", "))
  println(ts.has(Tag("scala")))
  ts = ts.remove(Tag("jvm"))
  println(ts.toList.sorted.mkString(", "))
end main
```

Both `TagName` and `TagSet` are opaque types defined inside the same `Tag`  
object, so they share the same transparent scope.  This means the `TagSet`  
extension methods can use `+`, `-`, and `contains` directly on the  
underlying `Set[TagName]`.  External code interacts only with named  
constructors and extension methods, with no knowledge that a `Set` is  
involved.  The `TagName` normalization (lowercase, trimmed) is enforced at  
the `Tag.apply` boundary, so the set never holds inconsistently-cased tags.  

## Opaque types in DSLs

Opaque types give a DSL a typed vocabulary that hides its string backbone.  

```scala
object SqlDsl:
  opaque type Query = String

  def select(fields: String*): Query =
    s"SELECT ${fields.mkString(", ")}"

  def from(q: Query, table: String): Query =
    s"$q FROM $table"

  def where(q: Query, cond: String): Query =
    s"$q WHERE $cond"

  def limit(q: Query, n: Int): Query =
    s"$q LIMIT $n"

  extension (q: Query)
    def sql: String = q

@main def main() =
  val query =
    SqlDsl.limit(
      SqlDsl.where(
        SqlDsl.from(
          SqlDsl.select("id", "name", "email"),
          "users"
        ),
        "age > 18"
      ),
      10
    )
  println(query.sql)
end main
```

Each builder function takes a `Query` and returns a `Query`, so only  
values produced by the DSL itself can be composed — arbitrary strings are  
excluded from the outside.  The final `.sql` accessor exposes the assembled  
SQL string only when the caller explicitly requests it, preventing premature  
exposure of the underlying representation.  This technique is a lightweight  
alternative to a full algebraic expression tree when you trust the  
individual builder functions to produce valid fragments.  

## Opaque types with given conversions

A `given Conversion` enables opt-in implicit coercion between opaque types.  

```scala
object Distance:
  opaque type Kilometre = Double
  opaque type Mile      = Double

  def km(d: Double):    Kilometre = d
  def miles(d: Double): Mile      = d

  given Conversion[Mile, Kilometre] with
    def apply(m: Mile): Kilometre = m * 1.60934

  extension (k: Kilometre)
    def value: Double = k
    def show: String  = f"$k%.2f km"

  extension (m: Mile)
    def value: Double = m
    def show: String  = f"$m%.2f mi"

@main def main() =
  import Distance.given
  val marathon: Distance.Mile      = Distance.miles(26.2)
  val inKm:     Distance.Kilometre = marathon   // implicit conversion
  println(s"${marathon.show} = ${inKm.show}")
end main
```

The `given Conversion[Mile, Kilometre]` is defined inside `Distance` and  
brought into scope with `import Distance.given`.  With that import active  
the compiler will automatically apply the conversion whenever a `Mile` is  
used where a `Kilometre` is expected.  Without the import the two types  
remain completely incompatible, which means the conversion is opt-in at  
each call site.  This design gives the author full control over which  
coercions exist and where they apply, avoiding the pitfalls of always-on  
implicit conversions.  
