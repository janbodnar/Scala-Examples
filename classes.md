# Classes

Classes are the primary blueprint for creating objects in Scala 3.  
A class definition describes the state and behaviour that every instance  
of that type will possess. Scala 3 classes occupy a central place in the  
language's type system, which unifies object-oriented and functional  
programming into one coherent model.  

Every class definition in Scala 3 may carry a primary constructor written  
directly in the class header. Parameter modifiers such as `val` and `var`  
promote constructor parameters to persistent fields automatically. When  
`val` is used the field is immutable; when `var` is used the field can  
be reassigned. Parameters without a modifier become constructor-only  
values that are not visible outside the class body.  

Auxiliary constructors are written as `def this(...)` inside the class  
body and must call another constructor as their first statement. This  
delegation rule ensures that the primary constructor is always executed,  
keeping initialisation logic centralised. Default parameter values often  
eliminate the need for auxiliary constructors entirely.  

Case classes extend ordinary classes with structural equality, pattern  
matching support, and an automatically generated companion object. Traits  
define reusable interfaces and partial implementations that can be mixed  
into any class. Abstract classes resemble traits but may carry constructor  
parameters and are better suited to Java interoperability. Together these  
mechanisms let Scala model rich domain hierarchies with minimal ceremony.  

Scala 3 also introduces opaque type aliases, which wrap primitive values  
in a zero-overhead abstraction at compile time. Extension methods, context  
parameters, and given instances work naturally alongside classes, giving  
them a functional flavour without sacrificing the familiar OOP structure.  
This blend makes Scala 3 classes suitable for everything from simple data  
containers to complex algebraic data types and typeclass hierarchies.  

## Simple class

A minimal class with a primary constructor and a single method.  

```scala
class Greeter(name: String):
  def greet(): Unit = println(s"Hello, $name!")

@main def run() =
  val g = Greeter("Alice")
  g.greet()
```

The `Greeter` class declares one constructor parameter, `name`, which is  
accessible throughout the class body but is not promoted to a public  
field because no `val` or `var` modifier is used. The `greet` method  
builds a message with string interpolation and prints it. Instantiation  
uses the familiar `new`-free syntax introduced in Scala 3; writing  
`Greeter("Alice")` is identical to `new Greeter("Alice")`.  

## Primary constructor with fields

Using `val` in the constructor header promotes parameters to immutable  
public fields.  

```scala
class Point(val x: Double, val y: Double):
  def distanceTo(other: Point): Double =
    val dx = x - other.x
    val dy = y - other.y
    math.sqrt(dx * dx + dy * dy)

  override def toString: String = s"Point($x, $y)"

@main def run() =
  val p1 = Point(0.0, 0.0)
  val p2 = Point(3.0, 4.0)
  println(p1)
  println(p2)
  println(p1.distanceTo(p2))
```

Declaring `val x` and `val y` in the header means callers can read  
`p.x` and `p.y` directly. The `distanceTo` method uses both fields  
to compute the Euclidean distance with the standard `math.sqrt` function.  
Overriding `toString` gives every `Point` instance a human-readable  
representation that the `println` function picks up automatically.  

## Auxiliary constructor

An auxiliary constructor provides an alternate way to construct an object.  

```scala
class Rectangle(val width: Double, val height: Double):
  def this(side: Double) = this(side, side)

  def area: Double      = width * height
  def perimeter: Double = 2 * (width + height)

  override def toString: String =
    s"Rectangle($width x $height)"

@main def run() =
  val r1 = Rectangle(4.0, 6.0)
  val r2 = Rectangle(5.0)  // square
  println(r1)
  println(r2)
  println(r1.area)
  println(r2.perimeter)
```

The auxiliary constructor `def this(side: Double)` delegates to the  
primary constructor by calling `this(side, side)`, which satisfies the  
Scala rule that every auxiliary constructor must invoke another constructor  
as its very first action. The `area` and `perimeter` properties are  
parameterless methods that behave like computed fields at the call site.  
Default parameters could achieve the same result, but explicit auxiliary  
constructors make distinct construction paths immediately visible.  

## Mutable fields

Using `var` in the constructor creates fields that can be reassigned.  

```scala
class Counter(var value: Int = 0):
  def increment(): Unit = value += 1
  def decrement(): Unit = value -= 1
  def reset(): Unit     = value = 0

  override def toString: String = s"Counter($value)"

@main def run() =
  val c = Counter()
  c.increment()
  c.increment()
  c.increment()
  println(c)
  c.decrement()
  println(c)
  c.reset()
  println(c)
```

The `var value` parameter becomes a mutable public field on every  
`Counter` instance. The default value `0` makes it optional at the  
call site; `Counter()` creates a counter starting at zero. Each method  
mutates the field in place and returns `Unit`. Mutable state is  
appropriate here because a counter is inherently stateful, but immutable  
alternatives using pure functions and return values are generally  
preferred in functional Scala code.  

## Private and protected members

Access modifiers control the visibility of fields and methods.  

```scala
class BankAccount(owner: String, private var balance: Double):
  protected def validate(amount: Double): Boolean = amount > 0

  def deposit(amount: Double): Unit =
    if validate(amount) then balance += amount

  def withdraw(amount: Double): Unit =
    if validate(amount) && amount <= balance then
      balance -= amount

  def getBalance: Double = balance

  override def toString: String =
    s"$owner: $$${balance}"

@main def run() =
  val acc = BankAccount("Alice", 100.0)
  acc.deposit(50.0)
  acc.withdraw(30.0)
  println(acc)
  println(acc.getBalance)
```

`balance` is declared `private var`, so it can only be read or modified  
inside `BankAccount` and its companion object. The `validate` helper is  
`protected`, meaning it is also visible in subclasses. External code  
must use the `deposit`, `withdraw`, and `getBalance` methods, which  
enforce the business invariants. This encapsulation pattern prevents  
accidental corruption of internal state from outside the class.  

## Default parameters

Default values reduce boilerplate by making certain arguments optional.  

```scala
class HttpRequest(
    val method: String  = "GET",
    val path: String    = "/",
    val timeout: Int    = 30
):
  override def toString: String =
    s"$method $path (timeout: ${timeout}s)"

@main def run() =
  val r1 = HttpRequest()
  val r2 = HttpRequest("POST", "/api/users")
  val r3 = HttpRequest(path = "/health", timeout = 5)
  println(r1)
  println(r2)
  println(r3)
```

Every parameter in `HttpRequest` carries a default value, so the class  
can be instantiated with zero, one, two, or all three arguments. Named  
arguments, shown in the `r3` construction, let callers skip earlier  
parameters or supply them out of order without ambiguity. This pattern  
is common when modelling configuration objects where sensible defaults  
exist for most fields.  

## Getters and setters

Scala supports custom property accessors following the uniform access  
principle.  

```scala
class Temperature(private var _celsius: Double):
  def celsius: Double = _celsius
  def celsius_=(value: Double): Unit =
    if value < -273.15 then
      throw IllegalArgumentException("Below absolute zero")
    _celsius = value

  def fahrenheit: Double = _celsius * 9.0 / 5.0 + 32.0

  override def toString: String =
    f"${_celsius}%.2f°C / ${fahrenheit}%.2f°F"

@main def run() =
  val t = Temperature(20.0)
  println(t)
  t.celsius = 100.0
  println(t)
```

The private field `_celsius` is exposed through a getter `celsius` and  
a setter `celsius_=`. Scala recognises the `_=` suffix as the setter  
convention, allowing call sites to write `t.celsius = 100.0` as if it  
were a simple field assignment. The setter validates the input before  
persisting it, which keeps the object in a legal state at all times. The  
`fahrenheit` property is a computed, read-only value derived from the  
stored Celsius temperature.  

## Companion object

A companion object shares the class name and has access to all private  
members.  

```scala
class Circle(val radius: Double):
  def area: Double         = Circle.Pi * radius * radius
  def circumference: Double = 2.0 * Circle.Pi * radius
  override def toString: String = s"Circle(r=$radius)"

object Circle:
  private val Pi  = math.Pi
  val UnitCircle  = Circle(1.0)

  def fromDiameter(d: Double): Circle = Circle(d / 2.0)

@main def run() =
  val c1 = Circle(5.0)
  val c2 = Circle.fromDiameter(10.0)
  val c3 = Circle.UnitCircle
  println(c1)
  println(f"Area: ${c1.area}%.2f")
  println(c2)
  println(c3)
```

The companion object `Circle` lives in the same source file as the class  
and may access its private members. The factory method `fromDiameter`  
gives callers an expressive alternative to the raw constructor. The  
constant `UnitCircle` is a ready-made singleton instance stored in the  
companion. Classes and their companions together form a module-like unit  
that bundles both instance-level and object-level concerns cleanly.  

## apply and unapply in companion

The `apply` method enables call syntax; `unapply` enables pattern  
matching.  

```scala
class Colour(val red: Int, val green: Int, val blue: Int):
  override def toString: String = f"#$red%02X$green%02X$blue%02X"

object Colour:
  def apply(hex: String): Colour =
    val v = Integer.parseInt(hex.stripPrefix("#"), 16)
    Colour((v >> 16) & 0xff, (v >> 8) & 0xff, v & 0xff)

  def unapply(c: Colour): Option[(Int, Int, Int)] =
    Some((c.red, c.green, c.blue))

@main def run() =
  val red   = Colour(255, 0, 0)
  val coral = Colour("#FF6B6B")
  println(red)
  println(coral)

  coral match
    case Colour(r, g, b) => println(s"r=$r g=$g b=$b")
```

`apply` is called whenever the companion is invoked as a function, so  
`Colour("#FF6B6B")` transparently delegates to the factory. `unapply`  
is the extractor used by the pattern matcher; it receives an instance  
and returns an `Option` wrapping the decomposed values. These two methods  
together give `Colour` the same ergonomics as a case class while letting  
the class definition remain fully hand-crafted.  

## Inheritance

A subclass extends a superclass to inherit and specialise behaviour.  

```scala
open class Animal(val name: String):
  def speak(): String = "..."
  override def toString: String = s"$name says: ${speak()}"

class Dog(name: String) extends Animal(name):
  override def speak(): String = "Woof"

class Cat(name: String) extends Animal(name):
  override def speak(): String = "Meow"

@main def run() =
  val animals: List[Animal] = List(Dog("Rex"), Cat("Whiskers"))
  animals.foreach(println)
```

The `open` modifier on `Animal` signals that the class is designed for  
subclassing; in Scala 3 classes without `open` emit a warning when  
extended from a different compilation unit. `Dog` and `Cat` both call  
the parent constructor by passing `name` to `extends Animal(name)`.  
Overriding `speak` demonstrates runtime polymorphism: iterating over  
the `List[Animal]` calls the correct `speak` implementation for each  
actual type at runtime.  

## Overriding methods

Method overriding allows subclasses to replace superclass behaviour.  

```scala
open class Shape:
  def area: Double     = 0.0
  def describe: String = s"Shape with area $area"

class Triangle(val base: Double, val height: Double) extends Shape:
  override def area: Double = 0.5 * base * height

class Square(val side: Double) extends Shape:
  override def area: Double   = side * side
  override def describe: String =
    s"Square(side=$side) with area $area"

@main def run() =
  val shapes: List[Shape] = List(
    Triangle(3.0, 4.0),
    Square(5.0)
  )
  shapes.foreach(s => println(s.describe))
```

The `override` keyword is mandatory in Scala and acts as a safety net:  
the compiler rejects it if no matching method exists in any superclass,  
preventing accidental method hiding. `Triangle` overrides only `area`;  
it inherits the default `describe` implementation from `Shape`. `Square`  
overrides both methods, supplying a more specific description. Calling  
`describe` on a `List[Shape]` dispatches dynamically to each concrete  
implementation.  

## Abstract class

An abstract class defines an incomplete blueprint that subclasses must  
complete.  

```scala
abstract class Logger(val prefix: String):
  def log(message: String): Unit
  def warn(message: String): Unit  = log(s"[WARN] $message")
  def error(message: String): Unit = log(s"[ERROR] $message")

class ConsoleLogger(prefix: String) extends Logger(prefix):
  def log(message: String): Unit =
    println(s"[$prefix] $message")

class SilentLogger extends Logger("SILENT"):
  def log(message: String): Unit = ()

@main def run() =
  val cl: Logger = ConsoleLogger("APP")
  cl.log("Server started")
  cl.warn("Low memory")
  cl.error("Connection refused")

  val sl: Logger = SilentLogger()
  sl.error("This will not be printed")
```

An abstract class may contain both abstract and concrete members.  
`log` is abstract; every subclass must provide a concrete definition  
or itself remain abstract. `warn` and `error` are concrete helpers that  
delegate to `log`, so subclasses inherit useful behaviour for free.  
Unlike traits, abstract classes support constructor parameters, making  
them the right choice when a base initialisation parameter like `prefix`  
must be forwarded from every concrete subclass.  

## Traits mixed into classes

Traits express capabilities that can be mixed into multiple unrelated  
classes.  

```scala
trait JsonSerializable:
  def toJson: String

trait Printable:
  def print(): Unit = println(toString)

class Product(val id: Int, val name: String)
    extends JsonSerializable, Printable:
  def toJson: String = s"""{"id":$id,"name":"$name"}"""
  override def toString: String = s"Product($id, $name)"

class User(val username: String, val email: String)
    extends JsonSerializable, Printable:
  def toJson: String = s"""{"user":"$username","email":"$email"}"""
  override def toString: String = s"User($username)"

@main def run() =
  val items: List[JsonSerializable & Printable] =
    List(Product(1, "Lamp"), User("alice", "a@b.com"))

  items.foreach: item =>
    item.print()
    println(item.toJson)
```

A class may extend multiple traits by listing them after `extends`  
separated by commas. `Printable` provides a default `print`  
implementation for free, while `JsonSerializable` declares an abstract  
contract that each class must fulfil. The compound type  
`JsonSerializable & Printable` expresses that every element in the list  
satisfies both constraints simultaneously. This mixin composition  
replaces multiple inheritance without its ambiguity problems.  

## Anonymous class

An anonymous class is an unnamed on-the-spot implementation of a trait  
or abstract class.  

```scala
trait Transformer:
  def transform(s: String): String

@main def run() =
  val upper: Transformer = new Transformer:
    def transform(s: String): String = s.toUpperCase

  val reverse: Transformer = new Transformer:
    def transform(s: String): String = s.reverse

  val pipeline = List(upper, reverse)
  val input    = "hello"
  val result   = pipeline.foldLeft(input)((s, t) => t.transform(s))
  println(result)
```

Anonymous classes are created with the `new Trait:` syntax followed by  
method implementations in an indented block. No name is needed because  
the instance is assigned directly to a `val`. They are useful for quick,  
localised adaptors or callbacks where defining a full named class would  
add unnecessary ceremony. The example builds a transformation pipeline  
by folding a list of anonymous transformers over an initial string.  

## Nested class

A class defined inside another class creates a separate type per outer  
instance.  

```scala
class Department(val name: String):
  class Employee(val empName: String):
    def info: String = s"$empName in $name"

  private var employees: List[Employee] = Nil

  def hire(empName: String): Employee =
    val e = Employee(empName)
    employees = e :: employees
    e

  def roster: List[String] = employees.map(_.info)

@main def run() =
  val eng = Department("Engineering")
  val mkt = Department("Marketing")

  eng.hire("Alice")
  eng.hire("Bob")
  mkt.hire("Carol")

  println(eng.roster)
  println(mkt.roster)
```

In Scala, an inner class is path-dependent: the type of an employee  
hired from `eng` is `eng.Employee`, distinct from `mkt.Employee`. This  
means the compiler prevents mixing employees from different departments  
by accident. The outer instance's `name` field is accessible inside the  
inner class without any qualification, because the inner class captures  
the reference to its enclosing instance.  

## Enums as special classes

Scala 3 enums define closed sets of named values and may carry  
parameters and methods.  

```scala
enum Direction(val degrees: Int):
  case North extends Direction(0)
  case East  extends Direction(90)
  case South extends Direction(180)
  case West  extends Direction(270)

  def opposite: Direction =
    Direction.fromOrdinal((ordinal + 2) % 4)

@main def run() =
  for d <- Direction.values do
    println(s"$d (${d.degrees}°) -> opposite: ${d.opposite}")
```

Scala 3 `enum` is syntactic sugar for a sealed class hierarchy. Each  
case is a singleton that extends the base enum with its own constructor  
argument. Methods like `opposite` are shared by all cases. The `values`  
array and `fromOrdinal` are generated automatically by the compiler.  
Enums are a concise alternative to a sealed trait plus companion object  
and case objects, and they integrate naturally with pattern matching.  

## Case class

A case class is an immutable data type with structural equality and  
built-in pattern-matching support.  

```scala
case class Address(street: String, city: String, zip: String)
case class Person(name: String, age: Int, address: Address)

@main def run() =
  val p1 = Person(
    "Alice", 30, Address("1 Main St", "Springfield", "12345"))
  val p2 = p1.copy(age = 31)
  val p3 = p1.copy(address = p1.address.copy(city = "Shelbyville"))

  println(p1)
  println(p2)
  println(p1 == p2)

  p1 match
    case Person(name, age, Address(_, city, _)) =>
      println(s"$name, age $age, lives in $city")
```

Case classes automatically generate `equals`, `hashCode`, `toString`,  
and a `copy` method. `copy` returns a new instance with selected fields  
replaced, which is the idiomatic way to simulate mutation in immutable  
data. Nested case classes can be destructured in a single pattern, as  
shown in the `match` expression where `Address` is unpacked inline.  
Case classes form the backbone of algebraic data types in Scala.  

## Opaque type with class wrapper

Opaque types provide zero-cost compile-time wrappers for domain values.  

```scala
object Domain:
  opaque type UserId = Int
  opaque type Email  = String

  object UserId:
    def apply(n: Int): UserId = n
    extension (id: UserId) def value: Int = id

  object Email:
    def apply(s: String): Email =
      require(s.contains("@"), "invalid email")
      s
    extension (e: Email) def value: String = e

import Domain.*

@main def run() =
  val uid   = UserId(42)
  val email = Email("alice@example.com")
  println(uid.value)
  println(email.value)
  // val bad: UserId = 99  // compile error: Int is not UserId
```

Opaque types exist only at compile time; at runtime `UserId` is a plain  
`Int` with no boxing overhead. Outside the defining object, `UserId` and  
`Int` are distinct types and cannot be substituted for each other, which  
prevents mistakes like passing a raw integer where a typed identifier is  
expected. Extension methods defined in the companion object restore  
convenient access to the underlying value when needed.  

## Inline methods in class

Inline methods are expanded at every call site by the compiler, allowing  
zero-overhead abstractions.  

```scala
class Vec2(val x: Double, val y: Double):
  inline def +(other: Vec2): Vec2 =
    Vec2(x + other.x, y + other.y)

  inline def scale(factor: Double): Vec2 =
    Vec2(x * factor, y * factor)

  inline def dot(other: Vec2): Double =
    x * other.x + y * other.y

  override def toString: String = s"Vec2($x, $y)"

@main def run() =
  val v1 = Vec2(1.0, 2.0)
  val v2 = Vec2(3.0, 4.0)
  println(v1 + v2)
  println(v1.scale(2.0))
  println(v1.dot(v2))
```

The `inline` modifier instructs the Scala 3 compiler to substitute the  
method body directly at the call site during compilation. This removes  
the overhead of a virtual method dispatch for small mathematical  
operations. `Vec2` uses inline methods for its core arithmetic, making  
it suitable for performance-sensitive numeric code. Unlike macros,  
inline methods are straightforward to write and reason about.  

## Using givens inside a class

A class can require context parameters implicitly via `using` clauses.  

```scala
trait Formatter[A]:
  def format(value: A): String

given Formatter[Int]    = v => s"Int($v)"
given Formatter[String] = v => s"Str($v)"

class Printer[A](val label: String)(using fmt: Formatter[A]):
  def print(value: A): Unit =
    println(s"$label: ${fmt.format(value)}")

@main def run() =
  val intPrinter    = Printer[Int]("number")
  val stringPrinter = Printer[String]("text")
  intPrinter.print(42)
  stringPrinter.print("hello")
```

The `using fmt: Formatter[A]` clause declares a context parameter that  
the compiler resolves automatically from the implicit scope when the  
class is instantiated. Callers do not need to pass the formatter  
explicitly; the compiler injects the correct given instance based on the  
type argument `A`. This pattern decouples the class from a concrete  
formatting strategy without resorting to subclassing.  

## Extension methods in companion

Extension methods defined in a companion extend the class with new  
operations without modifying its body.  

```scala
class WordBuffer(val content: String):
  override def toString: String = content

object WordBuffer:
  extension (wb: WordBuffer)
    def words: List[String]      = wb.content.split("\\s+").toList
    def wordCount: Int           = wb.words.length
    def reversed: WordBuffer     = WordBuffer(wb.content.reverse)

@main def run() =
  val buf = WordBuffer("the quick brown fox")
  println(buf.words)
  println(buf.wordCount)
  println(buf.reversed)
```

Extension methods in a companion object are automatically in scope  
wherever the class is used, without any explicit import. This makes  
companion objects a convenient place to add utility operations that  
logically belong to the type but are not part of its core contract.  
The extension block groups multiple methods together under one receiver  
type, keeping the code tidy.  

## Type parameters on classes

Generic classes accept one or more type parameters for reusable  
containers.  

```scala
class Pair[A, B](val first: A, val second: B):
  def swap: Pair[B, A]            = Pair(second, first)
  def mapFirst[C](f: A => C): Pair[C, B] = Pair(f(first), second)
  override def toString: String   = s"($first, $second)"

@main def run() =
  val p1 = Pair(1, "one")
  val p2 = p1.swap
  val p3 = p1.mapFirst(_ * 10)
  println(p1)
  println(p2)
  println(p3)
```

`Pair[A, B]` is parameterised over two independent types, so it can  
hold any combination of values without runtime casts. The `swap` method  
returns a `Pair[B, A]`, demonstrating that generic methods can produce  
types derived from the class parameters. `mapFirst` introduces a third  
type parameter `C` local to that method, transforming the first element  
while leaving the second unchanged. The compiler infers all type  
arguments at the call sites.  

## Variance annotations

Variance controls how a generic type relates to its type parameter  
hierarchy.  

```scala
class Producer[+A](val value: A):
  def get: A = value

class Consumer[-A]:
  def consume(value: A): Unit = println(s"Consuming: $value")

class Transformer[A](f: A => A):
  def apply(value: A): A = f(value)

@main def run() =
  val intProducer: Producer[Int] = Producer(42)
  val anyProducer: Producer[Any] = intProducer  // covariant

  val anyConsumer: Consumer[Any] = Consumer[Any]()
  val intConsumer: Consumer[Int] = anyConsumer   // contravariant

  val doubler = Transformer[Int](_ * 2)
  println(intProducer.get)
  println(doubler(5))
```

`+A` on `Producer` means covariance: a `Producer[Int]` is a subtype  
of `Producer[Any]`, matching our intuition that a source of integers  
is also a source of any values. `-A` on `Consumer` means  
contravariance: a `Consumer[Any]` is a subtype of `Consumer[Int]`  
because something that can consume any value can certainly consume  
integers. Invariant `A` on `Transformer` means no subtyping in either  
direction, which is correct for types that both read and write their  
parameter.  

## Self-type

A self-type annotation declares that a trait or class requires another  
type to be mixed in.  

```scala
trait Database:
  def query(sql: String): List[String]

trait Logging:
  def log(msg: String): Unit = println(s"[LOG] $msg")

trait UserRepository:
  self: Database & Logging =>

  def findUser(id: Int): Option[String] =
    log(s"Looking up user $id")
    query(s"SELECT name FROM users WHERE id = $id").headOption

class InMemoryDb extends Database, Logging, UserRepository:
  def query(sql: String): List[String] =
    if sql.contains("id = 1") then List("Alice") else Nil

@main def run() =
  val db = InMemoryDb()
  println(db.findUser(1))
  println(db.findUser(2))
```

The self-type `self: Database & Logging =>` inside `UserRepository`  
declares that any class mixing in `UserRepository` must also mix in  
`Database` and `Logging`. The compiler enforces this at the concrete  
class definition. Inside the trait body, `query` and `log` are  
available as if they were inherited, because the self-type annotation  
guarantees their presence. This pattern is a lightweight alternative  
to inheritance for expressing architectural dependencies.  

## Classes with collections

A class that encapsulates a collection and exposes higher-order  
operations.  

```scala
class Library(private val books: List[String]):
  def count: Int = books.length
  def search(keyword: String): List[String] =
    books.filter(_.toLowerCase.contains(keyword.toLowerCase))
  def add(book: String): Library = Library(book :: books)
  def sorted: Library            = Library(books.sorted)
  override def toString: String  = books.mkString("[", ", ", "]")

@main def run() =
  val lib = Library(List(
    "Scala Cookbook", "Clean Code",
    "Effective Java", "Scala with Cats"))
  println(lib)
  println(lib.count)
  println(lib.search("scala"))
  println(lib.add("Programming in Scala").sorted)
```

`Library` wraps a `List[String]` and exposes domain-specific operations  
rather than the raw collection API. Because `books` is a `val` and  
`add` returns a new `Library` instead of modifying the list in place,  
the class is fully immutable. The `sorted` method also returns a new  
instance. This pattern makes `Library` safe to share across threads  
without synchronisation.  

## Classes with pattern matching

Scala classes integrate naturally with `match` expressions for  
expressive control flow.  

```scala
sealed trait Expr
case class Num(value: Double)            extends Expr
case class Add(left: Expr, right: Expr)  extends Expr
case class Mul(left: Expr, right: Expr)  extends Expr
case class Neg(expr: Expr)               extends Expr

def eval(expr: Expr): Double =
  expr match
    case Num(v)    => v
    case Add(l, r) => eval(l) + eval(r)
    case Mul(l, r) => eval(l) * eval(r)
    case Neg(e)    => -eval(e)

@main def run() =
  // (2 + 3) * -(4)
  val tree = Mul(Add(Num(2), Num(3)), Neg(Num(4)))
  println(eval(tree))
```

The sealed trait plus case classes pattern forms an algebraic data type  
(ADT). The `sealed` modifier tells the compiler that all subtypes are  
defined in the same file, enabling exhaustiveness checking in `match`  
expressions: the compiler warns if a case is missing. The `eval`  
function recursively deconstructs the tree using pattern matching on  
the concrete case class types. This is the idiomatic functional Scala  
approach to interpreting expression trees.  

## Classes with context functions

A context function type `A ?=> B` captures an implicit context that  
classes can use for scoped configuration.  

```scala
case class Config(host: String, port: Int, debug: Boolean)

class AppContext(val config: Config):
  def run(task: Config ?=> Unit): Unit = task(using config)

def connect()(using cfg: Config): Unit =
  println(s"Connecting to ${cfg.host}:${cfg.port}")

def logLevel()(using cfg: Config): Unit =
  if cfg.debug then println("DEBUG mode on")
  else println("DEBUG mode off")

@main def run() =
  val ctx = AppContext(Config("localhost", 8080, debug = true))
  ctx.run:
    connect()
    logLevel()
```

A context function type `Config ?=> Unit` is a function that implicitly  
receives a `Config` value when called. The `AppContext.run` method  
accepts such a function and provides the config via `using`, making it  
available to every call inside the block without manual threading. This  
pattern is useful for structured configuration scopes, database  
transactions, or any operation that requires ambient context without  
polluting every parameter list.  

## Classes for domain modelling

Case classes compose naturally into rich domain models with validation  
logic.  

```scala
case class Money(amount: BigDecimal, currency: String):
  require(amount >= 0, "Amount must be non-negative")

  def +(other: Money): Money =
    require(currency == other.currency, "Currency mismatch")
    Money(amount + other.amount, currency)

  def *(factor: BigDecimal): Money = Money(amount * factor, currency)

  override def toString: String = f"$amount%.2f $currency"

case class LineItem(description: String, price: Money, qty: Int):
  def total: Money = price * qty

case class Invoice(items: List[LineItem]):
  def subtotal: Money = items.map(_.total).reduce(_ + _)

@main def run() =
  val inv = Invoice(List(
    LineItem("Widget", Money(9.99, "USD"), 3),
    LineItem("Gadget", Money(24.50, "USD"), 1)
  ))
  println(inv.subtotal)
```

`Money` uses `require` to validate its constructor argument, throwing  
an `IllegalArgumentException` at construction time for negative amounts.  
Operator methods `+` and `*` return new `Money` instances, keeping the  
type immutable. `LineItem` and `Invoice` build on `Money` to form a  
small but complete domain model. This approach favours making illegal  
states unrepresentable: a `Money` with a negative amount simply cannot  
exist.  

## Classes for functional patterns

A class that implements `map` and `flatMap` becomes usable in `for`  
comprehensions.  

```scala
class Box[+A](val value: A):
  def map[B](f: A => B): Box[B]     = Box(f(value))
  def flatMap[B](f: A => Box[B]): Box[B] = f(value)
  def filter(pred: A => Boolean): Option[Box[A]] =
    if pred(value) then Some(this) else None
  override def toString: String = s"Box($value)"

@main def run() =
  val box = Box(3)

  val result =
    for
      x <- box
      y <- Box(x * 2)
    yield y + 1

  println(result)
  println(box.map(_.toString))
  println(box.filter(_ > 5))
  println(box.filter(_ > 1))
```

`Box` implements `map` and `flatMap`, which makes it compatible with  
Scala's `for` comprehension syntax without inheriting from any framework  
class. The `for` expression desugars into nested `flatMap` and `map`  
calls, producing a new `Box` with the transformed value. This pattern  
illustrates how any class with `map` and `flatMap` gains the full  
expressiveness of monadic composition in Scala's syntax.  

## Classes interacting with Java

Scala classes extend Java classes and implement Java interfaces  
seamlessly on the JVM.  

```scala
import java.util.ArrayList
import java.util.Collections

class Version(val major: Int, val minor: Int, val patch: Int)
    extends Comparable[Version]:

  def compareTo(other: Version): Int =
    val dm = major.compareTo(other.major)
    if dm != 0 then dm
    else
      val dn = minor.compareTo(other.minor)
      if dn != 0 then dn
      else patch.compareTo(other.patch)

  override def toString: String = s"$major.$minor.$patch"

@main def run() =
  val versions = ArrayList[Version]()
  versions.add(Version(2, 1, 0))
  versions.add(Version(1, 9, 3))
  versions.add(Version(2, 0, 5))
  versions.add(Version(1, 9, 10))

  Collections.sort(versions)
  versions.forEach(v => println(v))
```

Because Scala compiles to JVM bytecode, a Scala class can extend any  
Java class or implement any Java interface directly. Here `Version`  
implements the `Comparable[Version]` interface from `java.util`, which  
enables `Collections.sort` to order a `java.util.ArrayList` without a  
separate comparator. The natural ordering defined in `compareTo` follows  
semantic versioning rules, comparing major, minor, and patch components  
in turn. This interoperability lets Scala code integrate with the entire  
Java ecosystem without wrappers.  
