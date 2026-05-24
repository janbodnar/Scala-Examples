# Scala 3 Enums

Enums in Scala 3 are a first-class language feature introduced to replace the  
verbose `sealed trait` / `case object` pattern that Scala 2 developers relied  
on. A Scala 3 enum declares a named type together with a fixed set of named  
values or parameterised cases in a single, concise block. The compiler  
automatically generates an `ordinal` field, a `values` array, and a `fromOrdinal`  
lookup method for every enum, giving it parity with Java's `enum` without  
sacrificing type safety.  

Compared with Java enums, Scala 3 enums go further in two key directions.  
First, individual cases may carry parameters, turning each case into a  
lightweight case-class-like construct with its own fields. Second, enum types  
participate fully in the type system: they can have type parameters, implement  
traits, define methods, carry `given` instances, and appear in generic  
abstractions. This makes them suitable for modelling algebraic data types  
(ADTs), domain values, state machines, and capability sets without any  
ceremony.  

Compared with sealed traits, enums are more compact. A sealed trait hierarchy  
requires a trait declaration, one object or case-class per variant, and a  
companion object for utilities. An equivalent enum collapses all of that into  
a single `enum` block. Pattern matching works identically for both, and the  
compiler still performs exhaustiveness checking, so migrating from sealed  
traits to enums is largely mechanical.  

Scala 3 enums support two case styles. Simple cases written as `case Red`  
define singleton values, similar to Java enum constants. Parameterised cases  
written as `case Some(value: A)` define distinct types with their own  
constructor, similar to case classes nested inside the enum. Both styles can  
coexist in the same enum, which is the canonical way to express ADTs such as  
`Option`, `Either`, or custom domain hierarchies.  

Enums also integrate naturally with advanced features: `given` instances can  
be declared inside an enum companion, `inline` methods allow compile-time  
dispatch, and opaque types can be layered on top to restrict construction.  
The sections below progress from the simplest enum declaration to real-world  
design patterns, covering each feature incrementally.  

## Simple enum

A basic enum declares a set of named singleton cases under a shared type.  

```scala
enum Direction:
  case North, South, East, West

@main def run() =
  val d: Direction = Direction.North
  println(d)
  println(d.ordinal)
```

The `Direction` enum defines four cases in a single declaration. Each case  
is a singleton value of type `Direction`, so `Direction.North` and  
`Direction.South` are distinct objects sharing a common supertype. The  
`ordinal` field is generated automatically by the compiler and reflects  
the zero-based position of the case in the declaration order.  

## Enum values array

The compiler generates a `values` array that lists all cases in declaration  
order.  

```scala
enum Planet:
  case Mercury, Venus, Earth, Mars, Jupiter, Saturn, Uranus, Neptune

@main def run() =
  for p <- Planet.values do
    println(s"${p.ordinal}: $p")
```

`Planet.values` returns an `Array[Planet]` containing every case in the  
order they were declared. Iterating over it with a `for` expression is  
the standard way to enumerate all members of an enum at runtime. The  
`ordinal` field lets you convert any case back to its integer index, and  
`Planet.fromOrdinal(n)` performs the reverse lookup.  

## Enum with parameters

Cases can carry immutable fields by listing parameters in the case header.  

```scala
enum Color(val hex: String):
  case Red   extends Color("#FF0000")
  case Green extends Color("#00FF00")
  case Blue  extends Color("#0000FF")

@main def run() =
  for c <- Color.values do
    println(s"$c -> ${c.hex}")
```

Declaring `val hex: String` on the enum header makes the field available  
on every case. Each case then provides its concrete value via the `extends`  
clause that calls the enum constructor. Marking the parameter with `val`  
exposes it as a public read-only field on every `Color` instance. The  
`Color.values` array lets you iterate all colours and inspect their  
associated hex strings.  

## Enum with methods

An enum body may contain method definitions shared by all cases.  

```scala
enum Weekday:
  case Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday

  def isWeekend: Boolean =
    this == Saturday || this == Sunday

  def next: Weekday =
    Weekday.fromOrdinal((ordinal + 1) % 7)

@main def run() =
  println(Weekday.Friday.isWeekend)
  println(Weekday.Saturday.isWeekend)
  println(Weekday.Sunday.next)
```

Methods defined inside an enum body are available on every case, just as  
instance methods work on case classes. The `isWeekend` method compares  
`this` against the two weekend cases using reference equality. The `next`  
method wraps around to `Monday` after `Sunday` by using `fromOrdinal` with  
modular arithmetic on the `ordinal` value.  

## Enum with overridden methods per case

Individual cases may override methods to provide case-specific behaviour.  

```scala
enum Shape:
  case Circle(radius: Double)
  case Rectangle(width: Double, height: Double)
  case Triangle(base: Double, height: Double)

  def area: Double = this match
    case Circle(r)        => math.Pi * r * r
    case Rectangle(w, h)  => w * h
    case Triangle(b, h)   => 0.5 * b * h

@main def run() =
  val shapes = List(
    Shape.Circle(5),
    Shape.Rectangle(4, 6),
    Shape.Triangle(3, 8)
  )
  shapes.foreach(s => println(f"${s}: area = ${s.area}%.2f"))
```

Here the cases carry parameters, making each one a distinct parameterised  
variant of `Shape`. The `area` method is defined once on the enum using a  
`match` expression that dispatches on the concrete case. This keeps related  
logic together without scattering it across separate classes. The compiler  
enforces exhaustiveness, so adding a new case without updating `area` causes  
a compile-time warning.  

## Enum extending a trait

An enum type can extend a trait to satisfy an interface contract.  

```scala
trait Printable:
  def display(): Unit

enum Season extends Printable:
  case Spring, Summer, Autumn, Winter

  def display(): Unit =
    println(s"Current season: $this")

@main def run() =
  val s: Printable = Season.Autumn
  s.display()
  Season.values.foreach(_.display())
```

Using `extends Printable` on the enum header makes every case a subtype of  
`Printable`. The `display` method fulfils the trait contract for all cases  
uniformly. Because the enum type itself extends the trait, instances can be  
passed wherever a `Printable` is expected, enabling polymorphic usage without  
any additional boilerplate.  

## Enum mixing in multiple traits

An enum may mix in several traits simultaneously using `with`.  

```scala
trait Named:
  def name: String

trait Ranked:
  def rank: Int

enum Medal(val name: String, val rank: Int) extends Named with Ranked:
  case Gold   extends Medal("Gold",   1)
  case Silver extends Medal("Silver", 2)
  case Bronze extends Medal("Bronze", 3)

@main def run() =
  val medals: List[Named & Ranked] = Medal.values.toList
  medals.foreach(m => println(s"${m.rank}. ${m.name}"))
```

Mixing in `Named` and `Ranked` means every `Medal` case satisfies both  
interfaces. The intersection type `Named & Ranked` on the list demonstrates  
that the compiler tracks both supertype constraints simultaneously. Adding  
further traits simply appends another `with` clause without restructuring  
the existing hierarchy.  

## Enum with a companion object

A companion object placed outside the enum body adds factory methods and  
utility logic.  

```scala
enum Priority:
  case Low, Normal, High, Critical

object Priority:
  def fromString(s: String): Option[Priority] =
    values.find(_.toString.equalsIgnoreCase(s))

  def highest: Priority = Critical

@main def run() =
  println(Priority.fromString("high"))
  println(Priority.fromString("unknown"))
  println(Priority.highest)
```

The companion object has the same name as the enum and lives in the same  
scope. `fromString` searches the `values` array case-insensitively and  
returns an `Option[Priority]` to signal failure without throwing an  
exception. The `highest` convenience member avoids hard-coding `Critical`  
at call sites, centralising that domain knowledge in one place.  

## Enum with apply and unapply

Custom `apply` and `unapply` in the companion enable flexible construction  
and pattern matching.  

```scala
enum Status:
  case Active, Inactive, Pending

object Status:
  def apply(code: Int): Status = code match
    case 1 => Active
    case 2 => Inactive
    case _ => Pending

  def unapply(s: Status): Option[Int] = s match
    case Active   => Some(1)
    case Inactive => Some(2)
    case Pending  => Some(0)

@main def run() =
  val s = Status(1)
  println(s)

  s match
    case Status(code) => println(s"Code is $code")
```

The `apply` method converts an integer code into the corresponding `Status`  
case, defaulting to `Pending` for unrecognised values. The `unapply` extractor  
enables the `Status(code)` pattern in match expressions, which recovers the  
integer representation at the call site. Together they create a symmetric  
encoding and decoding API without coupling callers to the raw enum ordinal.  

## Enum with pattern matching

Pattern matching on enum cases is concise and exhaustively checked.  

```scala
enum HttpMethod:
  case GET, POST, PUT, DELETE, PATCH

def describe(m: HttpMethod): String = m match
  case HttpMethod.GET    => "Read a resource"
  case HttpMethod.POST   => "Create a resource"
  case HttpMethod.PUT    => "Replace a resource"
  case HttpMethod.DELETE => "Remove a resource"
  case HttpMethod.PATCH  => "Partially update a resource"

@main def run() =
  HttpMethod.values.foreach(m => println(s"$m: ${describe(m)}"))
```

Each arm of the match corresponds to a single enum case. The compiler  
verifies at compile time that every case of `HttpMethod` is covered; removing  
any arm produces a warning about a non-exhaustive match. Importing  
`HttpMethod.*` at the top of a file removes the need for the `HttpMethod.`  
prefix in each arm, making the match expression even more readable.  

## Exhaustive match checking

The compiler emits a warning when a match expression does not cover all  
cases.  

```scala
enum Light:
  case Red, Yellow, Green

def action(l: Light): String = l match
  case Light.Red    => "Stop"
  case Light.Yellow => "Caution"
  case Light.Green  => "Go"

@main def run() =
  println(action(Light.Red))
  println(action(Light.Yellow))
  println(action(Light.Green))
```

Because `Light` is a sealed enum, the compiler knows the complete set of  
possible cases at compile time. Commenting out the `Light.Green` arm causes  
the compiler to report `match may not be exhaustive`, turning a runtime  
`MatchError` into a compile-time diagnostic. This exhaustiveness guarantee  
is one of the principal safety benefits of modelling domains with enums  
rather than plain integers or strings.  

## Enum with parameterised cases (ADT style)

Combining simple cases and parameterised cases in one enum models algebraic  
data types directly.  

```scala
enum Expr:
  case Num(value: Double)
  case Add(left: Expr, right: Expr)
  case Mul(left: Expr, right: Expr)
  case Neg(expr: Expr)

def eval(e: Expr): Double = e match
  case Expr.Num(v)    => v
  case Expr.Add(l, r) => eval(l) + eval(r)
  case Expr.Mul(l, r) => eval(l) * eval(r)
  case Expr.Neg(x)    => -eval(x)

@main def run() =
  // (2 + 3) * -4
  val expr = Expr.Mul(
    Expr.Add(Expr.Num(2), Expr.Num(3)),
    Expr.Neg(Expr.Num(4))
  )
  println(eval(expr))
```

`Expr` is a recursive ADT: each parameterised case references `Expr` in its  
own fields, allowing arbitrarily deep expression trees to be built. The  
`eval` function deconstructs the tree recursively using pattern matching.  
Parameterised cases behave exactly like case classes: they have structural  
equality, `copy` methods, and automatically generated `toString` output. This  
pattern replaces `sealed trait Expr` / `case class Add` hierarchies with a  
single, compact block.  

## Enum as Option-like ADT

An enum with one empty case and one parameterised case models optional  
values.  

```scala
enum Maybe[+A]:
  case Empty
  case Present(value: A)

  def getOrElse[B >: A](default: B): B = this match
    case Maybe.Empty       => default
    case Maybe.Present(v)  => v

  def map[B](f: A => B): Maybe[B] = this match
    case Maybe.Empty      => Maybe.Empty
    case Maybe.Present(v) => Maybe.Present(f(v))

@main def run() =
  val found: Maybe[Int]    = Maybe.Present(42)
  val missing: Maybe[Int]  = Maybe.Empty

  println(found.getOrElse(0))
  println(missing.getOrElse(0))
  println(found.map(_ * 2))
```

`Maybe[+A]` is covariant in `A`, so a `Maybe[Int]` is a subtype of  
`Maybe[Any]`. The `getOrElse` type parameter `B >: A` widens the return  
type when the default has a different, broader type. The `map` method  
demonstrates functor-style transformation without touching the `Empty` case.  
This mirrors the design of Scala's built-in `Option` type almost exactly,  
showing that standard library types are not special—they can be rebuilt  
with plain enum syntax.  

## Enum with generics

Enums accept one or more type parameters to create fully generic variants.  

```scala
enum Result[+A, +E]:
  case Ok(value: A)
  case Err(error: E)

  def fold[B](onOk: A => B, onErr: E => B): B = this match
    case Result.Ok(v)  => onOk(v)
    case Result.Err(e) => onErr(e)

@main def run() =
  val success: Result[Int, String]  = Result.Ok(100)
  val failure: Result[Int, String]  = Result.Err("not found")

  println(success.fold(v => s"Value: $v", e => s"Error: $e"))
  println(failure.fold(v => s"Value: $v", e => s"Error: $e"))
```

The `Result[+A, +E]` enum is covariant in both type parameters, making it  
safe to widen `Result[Int, Nothing]` to `Result[Int, String]`. The `fold`  
method accepts two functions, one for each case, and applies the appropriate  
one, eliminating the need for callers to write match expressions. This  
pattern is the foundation of error-handling libraries in functional Scala  
and mirrors `Either` from the standard library.  

## Enum with type bounds

Type parameters on enum cases can carry upper or lower bounds for precise  
type constraints.  

```scala
enum Container[+A <: AnyRef]:
  case Box(item: A)
  case EmptyBox

  def inspect: String = this match
    case Container.Box(i)  => s"Contains: $i"
    case Container.EmptyBox => "Empty"

@main def run() =
  val b1: Container[String] = Container.Box("hello")
  val b2: Container[String] = Container.EmptyBox

  println(b1.inspect)
  println(b2.inspect)
```

The upper bound `A <: AnyRef` restricts the type parameter to reference  
types, excluding primitive types such as `Int` or `Boolean`. This is  
occasionally necessary when the implementation relies on reference semantics  
or when the container will only ever hold objects. The covariance annotation  
`+A` still applies on top of the bound, combining both constraints cleanly.  

## Enum with givens

A `given` instance placed in the enum companion provides a typeclass  
implementation for the enum type.  

```scala
enum Suit:
  case Clubs, Diamonds, Hearts, Spades

object Suit:
  given Ordering[Suit] with
    def compare(x: Suit, y: Suit): Int =
      x.ordinal - y.ordinal

@main def run() =
  val hand = List(Suit.Hearts, Suit.Clubs, Suit.Spades, Suit.Diamonds)
  println(hand.sorted)
  println(hand.min)
  println(hand.max)
```

Placing the `given Ordering[Suit]` inside the companion object makes it  
available for automatic summoning whenever the compiler needs an `Ordering`  
for `Suit`. Methods such as `sorted`, `min`, and `max` on `List[Suit]`  
implicitly resolve this instance. The `ordinal`-based comparison imposes  
the natural declaration order on the suit ranking, and the whole mechanism  
requires no explicit `import` at the call site.  

## Enum with inline methods

An `inline` method on an enum forces compile-time evaluation or dispatch.  

```scala
enum LogLevel:
  case Trace, Debug, Info, Warn, Error

  inline def isEnabled(threshold: LogLevel): Boolean =
    ordinal >= threshold.ordinal

@main def run() =
  val level = LogLevel.Info

  if level.isEnabled(LogLevel.Debug) then
    println("Debug logging active")

  if level.isEnabled(LogLevel.Error) then
    println("Error logging active")
  else
    println("Error logging inactive")
```

Marking `isEnabled` as `inline` causes the compiler to expand the call  
at the use site, potentially eliminating the method-call overhead. When both  
the receiver and the argument are known at compile time, the comparison  
reduces to a constant boolean, and the entire `if` branch may be erased  
by the optimiser. `inline` methods on enums are especially effective for  
logging guards and feature-flag checks that run in hot paths.  

## Enum with opaque types

An opaque type alias wraps an enum to hide its representation from callers.  

```scala
object Security:
  opaque type Permission = AccessLevel

  enum AccessLevel:
    case Read, Write, Admin

  object Permission:
    def read: Permission  = AccessLevel.Read
    def write: Permission = AccessLevel.Write
    def admin: Permission = AccessLevel.Admin

  def allows(p: Permission, required: AccessLevel): Boolean =
    p.ordinal >= required.ordinal

@main def run() =
  import Security.*

  val p = Permission.write
  println(allows(p, AccessLevel.Read))
  println(allows(p, AccessLevel.Admin))
```

Outside the `Security` object the type `Permission` is completely opaque:  
callers cannot access `ordinal`, cast to `AccessLevel`, or create arbitrary  
values. They can only use the smart constructors `read`, `write`, and  
`admin` provided by the companion. This design enforces an intentional  
construction discipline and prevents callers from bypassing access checks  
by constructing higher-privilege values directly.  

## Enum with context functions

A context function parameter threads implicit context automatically when  
dispatching on an enum case.  

```scala
trait Logger:
  def log(msg: String): Unit

enum Command:
  case Start, Stop, Restart

def execute(cmd: Command): Logger ?=> Unit = cmd match
  case Command.Start   => summon[Logger].log("Starting service")
  case Command.Stop    => summon[Logger].log("Stopping service")
  case Command.Restart =>
    summon[Logger].log("Stopping service")
    summon[Logger].log("Starting service")

@main def run() =
  given Logger with
    def log(msg: String): Unit = println(s"[LOG] $msg")

  execute(Command.Start)
  execute(Command.Restart)
```

The return type `Logger ?=> Unit` is a context function type. The compiler  
automatically passes the ambient `given Logger` when `execute` is called,  
removing the need to thread the logger manually through every call. Inside  
the function, `summon[Logger]` retrieves the instance. This pattern is  
useful for commands or events that need contextual services such as logging,  
metrics, or database connections.  

## Enum used as ADT for domain modeling

Enums naturally express domain concepts that have a closed set of variants.  

```scala
enum PaymentMethod:
  case Cash
  case Card(last4: String, network: String)
  case Crypto(wallet: String, coin: String)
  case BankTransfer(iban: String)

def fee(method: PaymentMethod): Double = method match
  case PaymentMethod.Cash                  => 0.0
  case PaymentMethod.Card(_, "Visa")       => 0.015
  case PaymentMethod.Card(_, _)            => 0.02
  case PaymentMethod.Crypto(_, _)          => 0.005
  case PaymentMethod.BankTransfer(_)       => 0.001

@main def run() =
  val methods = List(
    PaymentMethod.Cash,
    PaymentMethod.Card("4242", "Visa"),
    PaymentMethod.Card("1234", "Amex"),
    PaymentMethod.Crypto("0xABC", "ETH"),
    PaymentMethod.BankTransfer("DE89...")
  )
  methods.foreach(m => println(f"$m: fee = ${fee(m) * 100}%.1f%%"))
```

Each `PaymentMethod` case captures exactly the data relevant to its variant.  
`Cash` has no fields, while `Card` and `Crypto` carry two, and  
`BankTransfer` carries one. Pattern matching deconstructs each case and  
extracts its fields inline, and the `Card(_, "Visa")` pattern demonstrates  
partial matching on specific field values. This richness would otherwise  
require a sealed trait and separate case classes, all condensed into one  
`enum` block.  

## Enum for state machines

An enum models the states of a finite state machine with explicit  
transition logic.  

```scala
enum TrafficLight:
  case Red, Yellow, Green

  def next: TrafficLight = this match
    case Red    => Green
    case Green  => Yellow
    case Yellow => Red

  def canGo: Boolean = this == Green

@main def run() =
  var light = TrafficLight.Red
  for _ <- 1 to 6 do
    println(s"$light - go: ${light.canGo}")
    light = light.next
```

The `next` method encodes the transition table directly inside the enum,  
making each state responsible for knowing its successor. `canGo` expresses  
a guard predicate that external code can query before proceeding. Because  
the match in `next` is exhaustive, adding a new state without updating the  
transition table causes a compile-time warning, preventing forgotten  
transitions at the modelling stage rather than at runtime.  

## Enum for capability modeling

An enum can represent a set of permissions or capabilities, and bitwise-like  
operations can be composed with collections.  

```scala
enum Permission:
  case Read, Write, Execute, Delete

object Permission:
  type Set = scala.collection.immutable.Set[Permission]

  val readOnly: Set  = Set(Read)
  val standard: Set  = Set(Read, Write)
  val full: Set      = Set(Read, Write, Execute, Delete)

  def check(required: Permission, granted: Set): Boolean =
    granted.contains(required)

@main def run() =
  import Permission.*

  val userPerms = standard
  println(check(Read,    userPerms))
  println(check(Execute, userPerms))
  println(check(Delete,  userPerms))
```

Representing permissions as enum cases and grouping them into `Set[Permission]`  
collections gives a type-safe alternative to integer bitmasks. The  
`readOnly`, `standard`, and `full` constants define common permission  
profiles in one place. `check` evaluates whether a specific capability is  
present in a granted set without any casting or bitwise arithmetic, keeping  
the logic easy to audit.  

## Enum used with collections

Standard collection operations work directly on `List`, `Set`, and `Map`  
holding enum values.  

```scala
enum Category:
  case Food, Electronics, Clothing, Sports, Books

case class Product(name: String, category: Category, price: Double)

@main def run() =
  val catalog = List(
    Product("Shirt",    Category.Clothing,     29.99),
    Product("Phone",    Category.Electronics, 699.00),
    Product("Novel",    Category.Books,        12.50),
    Product("Sneakers", Category.Sports,       89.00),
    Product("Laptop",   Category.Electronics, 999.00),
    Product("Rice",     Category.Food,          3.50)
  )

  val byCategory = catalog.groupBy(_.category)
  byCategory.foreach { (cat, items) =>
    val total = items.map(_.price).sum
    println(f"$cat%-15s total: $$$total%.2f")
  }
```

`groupBy` accepts a function returning an enum value and produces a  
`Map[Category, List[Product]]`, with enum cases as keys. Standard map  
operations such as `foreach`, `filter`, and `map` work without any special  
treatment because enum cases support structural equality and a stable hash  
code. Using enum cases as map keys is idiomatic and far more readable than  
using string or integer identifiers.  

## Enum with match types

A match type selects a result type at compile time based on an enum-like  
discriminant.  

```scala
enum DataKind:
  case IntKind, StrKind, BoolKind

type Underlying[K <: DataKind] = K match
  case DataKind.IntKind.type  => Int
  case DataKind.StrKind.type  => String
  case DataKind.BoolKind.type => Boolean

def defaultValue[K <: DataKind](k: K): Underlying[K] =
  (k match
    case DataKind.IntKind  => 0
    case DataKind.StrKind  => ""
    case DataKind.BoolKind => false
  ).asInstanceOf[Underlying[K]]

@main def run() =
  println(defaultValue(DataKind.IntKind))
  println(defaultValue(DataKind.StrKind))
  println(defaultValue(DataKind.BoolKind))
```

Match types reduce a type-level computation to a concrete type based on the  
singleton type of an enum case. Each arm of the match type mirrors a  
corresponding arm in the term-level `match` expression inside `defaultValue`.  
The `asInstanceOf` cast is safe here because the match types and runtime  
types are aligned by construction. This technique powers zero-overhead  
heterogeneous containers and type-driven dispatch in advanced Scala libraries.  

## Enum with typeclass derivation

The `derives` clause instructs the compiler to generate typeclass instances  
automatically for an enum.  

```scala
import scala.deriving.Mirror

enum Compass derives CanEqual:
  case North, South, East, West

trait Describable[A]:
  def describe(a: A): String

object Describable:
  inline def derived[A](using m: Mirror.SumOf[A]): Describable[A] =
    a => a.toString.toLowerCase

  given Describable[Compass] = derived

@main def run() =
  val dir = Compass.North
  println(summon[Describable[Compass]].describe(dir))
  println(Compass.North == Compass.North)
  println(Compass.North == Compass.South)
```

The `derives CanEqual` clause generates a `CanEqual[Compass, Compass]`  
instance that enables the strict equality check `==` to compile without  
warnings under `-Ycheck-reachability`. The companion defines a minimal  
`Describable` typeclass and uses `inline def derived` with `Mirror.SumOf`  
to produce an instance automatically. Derivation eliminates hand-written  
boilerplate for typeclasses such as `Show`, `Encoder`, and `Decoder` in  
real applications.  

## Enum with custom constructors

A companion object can expose smart constructors that validate input before  
returning an enum case.  

```scala
enum Temperature:
  case Celsius(degrees: Double)
  case Fahrenheit(degrees: Double)
  case Kelvin(degrees: Double)

object Temperature:
  def celsius(d: Double): Either[String, Temperature] =
    if d < -273.15 then Left(s"$d °C is below absolute zero")
    else Right(Celsius(d))

  def kelvin(k: Double): Either[String, Temperature] =
    if k < 0 then Left(s"$k K is below absolute zero")
    else Right(Kelvin(k))

@main def run() =
  println(Temperature.celsius(100.0))
  println(Temperature.celsius(-300.0))
  println(Temperature.kelvin(0.0))
  println(Temperature.kelvin(-1.0))
```

The smart constructors `celsius` and `kelvin` perform domain validation  
before constructing a case, returning `Left` with a human-readable message  
on failure and `Right` with the valid case on success. Callers are forced  
to handle both outcomes because the return type is `Either[String, Temperature]`  
rather than `Temperature`. Raw case constructors remain accessible, but  
library documentation can steer users toward the safe factory methods.  

## Enum with private constructors

Making enum case constructors private restricts instantiation to controlled  
factory methods.  

```scala
enum Token private (val value: String):
  case Identifier extends Token("id")
  case Number     extends Token("num")
  case Operator   extends Token("op")

object Token:
  def parse(s: String): Option[Token] = s match
    case "id"  => Some(Identifier)
    case "num" => Some(Number)
    case "op"  => Some(Operator)
    case _     => None

@main def run() =
  println(Token.parse("id"))
  println(Token.parse("op"))
  println(Token.parse("unknown"))
```

The `private` modifier placed immediately after the enum name seals the  
primary constructor, preventing callers from writing `Token("something")`  
directly. Only the named cases declared inside the enum body, and the  
`parse` factory in the companion, can produce `Token` values. This enforces  
an invariant that every `Token` in the system originates from a known,  
controlled source.  

## Enum interacting with Java enums

Scala 3 enums can call methods on Java enum instances and be compared with  
them through shared interfaces.  

```scala
// Simulate a Java-style enum interaction via ordinal and name
enum JMonth:
  case Jan, Feb, Mar, Apr, May, Jun,
       Jul, Aug, Sep, Oct, Nov, Dec

  def quarter: Int = (ordinal / 3) + 1

  def daysInMonth(leapYear: Boolean = false): Int =
    this match
      case Feb                  => if leapYear then 29 else 28
      case Apr | Jun | Sep | Nov => 30
      case _                    => 31

@main def run() =
  for m <- JMonth.values do
    println(s"${m.name} Q${m.quarter} - days: ${m.daysInMonth()}")
```

While Scala 3 enums and Java enums are separate constructs, both expose  
`ordinal` and `name` members with identical semantics. `ordinal` is a  
zero-based integer index, and `name` returns the declared identifier as a  
string. The `daysInMonth` method uses a multi-case pattern `Apr | Jun | Sep  
| Nov` to handle the four 30-day months in one arm. Interoperability with  
Java code that consumes `java.lang.Enum` can be achieved by implementing a  
matching Java interface on the Scala enum.  

## Advanced enum hierarchies

Enums can model multi-level hierarchies by extending parameterised base  
traits.  

```scala
trait Event:
  def timestamp: Long

enum UserEvent(val timestamp: Long) extends Event:
  case Login(userId: String,  override val timestamp: Long)
      extends UserEvent(timestamp)
  case Logout(userId: String, override val timestamp: Long)
      extends UserEvent(timestamp)
  case Purchase(
    userId: String,
    amount: Double,
    override val timestamp: Long
  ) extends UserEvent(timestamp)

def summarise(e: Event): String = e match
  case UserEvent.Login(u, t)      => s"Login  $u at $t"
  case UserEvent.Logout(u, t)     => s"Logout $u at $t"
  case UserEvent.Purchase(u, a, t) =>
    f"Purchase by $u: $$$a%.2f at $t"

@main def run() =
  val events: List[Event] = List(
    UserEvent.Login("alice", 1000L),
    UserEvent.Purchase("alice", 49.99, 1020L),
    UserEvent.Logout("alice", 1080L)
  )
  events.foreach(e => println(summarise(e)))
```

Every case extends the `UserEvent` base, passing its own `timestamp`  
argument up to the enum primary constructor. The `Event` trait is satisfied  
because `UserEvent` extends it, making every case a valid `Event`. Pattern  
matching in `summarise` deconstructs each variant and extracts its specific  
fields. This multilevel design lets heterogeneous event streams be typed as  
`List[Event]` while preserving full case-level detail in match expressions.  

## Real-world enum design patterns

A complete domain model uses enums to capture states, commands, and events  
in a unified, type-safe vocabulary.  

```scala
enum OrderStatus:
  case Draft
  case Submitted(orderId: String)
  case Confirmed(orderId: String, eta: String)
  case Shipped(orderId: String, trackingCode: String)
  case Delivered(orderId: String)
  case Cancelled(reason: String)

object OrderStatus:
  def transition(
    current: OrderStatus,
    event: String
  ): Either[String, OrderStatus] =
    (current, event) match
      case (Draft, "submit")        =>
        Right(Submitted("ORD-001"))
      case (Submitted(id), "confirm") =>
        Right(Confirmed(id, "2026-06-01"))
      case (Confirmed(id, _), "ship") =>
        Right(Shipped(id, "TRK-999"))
      case (Shipped(id, _), "deliver") =>
        Right(Delivered(id))
      case (_, "cancel")            =>
        Right(Cancelled("User requested"))
      case (s, e)                   =>
        Left(s"Cannot apply '$e' to state $s")

@main def run() =
  var status: OrderStatus = OrderStatus.Draft
  val events = List("submit", "confirm", "ship", "deliver")

  for event <- events do
    OrderStatus.transition(status, event) match
      case Right(next) =>
        status = next
        println(s"[$event] -> $status")
      case Left(err) =>
        println(s"Error: $err")
```

`OrderStatus` captures every lifecycle state of an order as a distinct enum  
case, each carrying only the data that is meaningful at that stage. The  
`transition` companion method encodes the allowed state machine edges: a  
`Draft` order can be submitted, a `Submitted` order can be confirmed, and so  
on. Invalid transitions return `Left` with a diagnostic, which the caller  
handles explicitly. This pattern—enum as state, companion as transition  
table, `Either` as result—is a practical, production-ready design that  
scales from simple workflows to complex business processes.  
