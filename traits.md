# Scala 3 Traits  

A trait in Scala 3 is a reusable unit of behaviour that may contain both  
abstract and concrete members. Traits are the primary abstraction mechanism  
for building flexible, composable type hierarchies. Unlike a concrete class,  
a trait cannot be instantiated on its own. Unlike an abstract class, a  
single class or object may mix in any number of traits at once, enabling  
structured multiple inheritance without the fragility of traditional  
diamond-inheritance approaches.  

Compared to abstract classes, traits historically could not accept  
constructor parameters. Scala 3 removes that restriction: a trait may now  
declare a parameter list, letting callers supply values at mixin time  
without requiring an intermediate abstract class. Abstract classes remain  
useful when you need JVM-level single inheritance, controlled instantiation  
through a primary constructor, or interoperability with Java code that  
expects a concrete class type.  

Multiple inheritance in traits is resolved through **linearisation**. The  
compiler arranges all traits in a deterministic sequence based on  
declaration order and routes every `super` call through that chain. This  
guarantees that each trait in the hierarchy is visited exactly once and  
that method dispatch remains predictable regardless of how many traits are  
composed.  

Traits fit naturally into Scala's object-oriented and functional hybrid  
model:  

- As **interface contracts** they declare abstract members that implementors  
  must fulfil.  
- As **mixin modules** they carry default implementations that classes  
  inherit without duplication.  
- As **type-class carriers** they group `given` instances and extension  
  methods participating in implicit resolution.  
- As **capability tokens** they restrict which operations a type is allowed  
  to perform at compile time.  
- As **dependency injection wires** they express component dependencies  
  through self-type annotations.  

Scala 3 extends traits with `given`/`using` clauses, extension method  
blocks, `inline` optimisations, opaque type aliases, and context function  
types, making traits the central building block for both object-oriented  
hierarchies and purely functional library design.  

## Simple trait  

A trait that acts as an interface contract implemented by two classes.  

```scala
trait Greeter:
  def greet(name: String): String

class FormalGreeter extends Greeter:
  def greet(name: String): String = s"Good day, $name."

class CasualGreeter extends Greeter:
  def greet(name: String): String = s"Hey, $name!"

@main def run() =
  val formal: Greeter = FormalGreeter()
  val casual: Greeter = CasualGreeter()
  println(formal.greet("Alice"))
  println(casual.greet("Bob"))
```

The `Greeter` trait declares one abstract method `greet` with no default  
implementation. Both `FormalGreeter` and `CasualGreeter` extend the trait  
and provide their own concrete version. Because both instances are typed  
as `Greeter`, the code at the call site is polymorphic and decoupled from  
any particular implementation. This pattern mirrors a Java interface but  
is expressed with lighter syntax.  

## Abstract members  

A trait with both abstract methods and an abstract field.  

```scala
trait Shape:
  val name: String
  def area: Double
  def perimeter: Double

class Circle(radius: Double) extends Shape:
  val name: String   = "Circle"
  def area: Double   = math.Pi * radius * radius
  def perimeter: Double = 2 * math.Pi * radius

@main def run() =
  val c = Circle(5.0)
  println(f"${c.name}: area=${c.area}%.2f, perimeter=${c.perimeter}%.2f")
```

The `Shape` trait declares three abstract members: a stable `val name`  
and two abstract `def` members. The concrete `Circle` class is obligated  
to provide all three before it can be instantiated. Abstract fields  
declared with `val` must be overridden by a stable `val` in the concrete  
class, not by a `def`. The `f""` interpolator formats the output to two  
decimal places.  

## Concrete members  

A trait that supplies a default method implementation shared across  
subclasses.  

```scala
trait Printable:
  def label: String
  def print(): Unit = println(s"[Item] $label")

class Document(val label: String) extends Printable

class Invoice(val label: String) extends Printable:
  override def print(): Unit = println(s"[Invoice] $label")

@main def run() =
  val doc = Document("Annual Report")
  val inv = Invoice("INV-2024-001")
  doc.print()
  inv.print()
```

The `Printable` trait provides a concrete implementation of `print` that  
any subclass may use without change. `Document` inherits the default  
version directly. `Invoice` overrides `print` to customise the prefix  
while leaving `label` as the sole abstract requirement. Concrete members  
in traits enable code reuse across unrelated class hierarchies without  
duplicating logic.  

## Mixing traits into classes  

A class that gains behaviour from two traits at definition time.  

```scala
trait Flyable:
  def fly(): String = "I can fly!"

trait Swimmable:
  def swim(): String = "I can swim!"

class Duck extends Flyable with Swimmable:
  def describe(): String = s"${fly()} And ${swim()}"

@main def run() =
  val duck = Duck()
  println(duck.describe())
  println(duck.fly())
  println(duck.swim())
```

The `Duck` class extends `Flyable` and mixes in `Swimmable` using the  
`with` keyword. The class gains both `fly` and `swim` methods without  
repeating any code. The instance can be used wherever either `Flyable`  
or `Swimmable` is expected, making `Duck` a subtype of both traits  
simultaneously. Multiple traits are listed after the first `extends`  
keyword, each separated by `with`.  

## Mixing multiple traits  

A class that incorporates three independent traits to compose distinct  
capabilities.  

```scala
trait Loggable:
  def log(msg: String): Unit = println(s"[LOG]   $msg")

trait Auditable:
  def audit(action: String): Unit = println(s"[AUDIT] $action")

trait Exportable:
  def export: String

class UserService extends Loggable with Auditable with Exportable:
  def export: String = "UserService{}"
  def create(name: String): Unit =
    log(s"Creating user: $name")
    audit(s"user.create($name)")

@main def run() =
  val svc = UserService()
  svc.create("Alice")
  println(svc.export)
```

`UserService` mixes three separate traits, each contributing a distinct  
cross-cutting concern. The class must only provide the one remaining  
abstract member, `export`, while the other traits supply default  
implementations for logging and auditing. Composing behaviour through  
multiple traits avoids deep, rigid inheritance chains and keeps each  
concern isolated in its own trait definition.  

## Trait linearization  

Stackable modifications that chain calls through the mixin order.  

```scala
trait TextTransform:
  def process(s: String): String = s

trait Trimmed extends TextTransform:
  override def process(s: String): String = super.process(s.trim)

trait UpperCased extends TextTransform:
  override def process(s: String): String = super.process(s).toUpperCase

trait Exclaimed extends TextTransform:
  override def process(s: String): String = super.process(s) + "!"

class Pipeline extends Trimmed with UpperCased with Exclaimed

@main def run() =
  val p = Pipeline()
  println(p.process("  hello world  "))
```

Scala linearises `Pipeline` as: `Pipeline → Exclaimed → UpperCased →  
Trimmed → TextTransform`. When `process` is called, `Exclaimed` runs  
first, delegating to `UpperCased` via `super`, which calls `Trimmed`,  
which finally calls `TextTransform`. The whitespace is trimmed first,  
the result is uppercased, and then the exclamation mark is appended,  
producing `HELLO WORLD!`. This stackable-modifications pattern lets  
pipeline stages be assembled declaratively at the class definition site.  

## Traits with parameters  

A Scala 3 trait that carries a parameter list, removing the need for an  
intermediate abstract class.  

```scala
trait Logger(val prefix: String):
  def log(msg: String): Unit = println(s"[$prefix] $msg")

trait Metrics(val serviceName: String):
  def record(event: String): Unit =
    println(s"metric.$serviceName.$event")

class PaymentService(name: String)
    extends Logger(s"Service:$name")
    with Metrics(name):
  def pay(amount: Double): Unit =
    log(s"processing $$${amount}")
    record("payment")

@main def run() =
  val svc = PaymentService("payments")
  svc.pay(99.99)
```

Scala 3 introduces parameterised traits, a feature absent from Scala 2.  
`Logger` accepts a `prefix` value used in every `log` call, and `Metrics`  
accepts a service name. `PaymentService` supplies both arguments in its  
own `extends` clause, exactly as a class calls a superclass constructor.  
The `val` modifier on the parameter makes it a publicly readable field  
accessible from outside the trait.  

## Traits extending other traits  

A trait that inherits abstract and concrete members from another trait.  

```scala
trait Animal:
  def name: String
  def sound: String

trait Pet extends Animal:
  def owner: String
  def introduce(): String =
    s"I am $name, owned by $owner. I say: $sound."

class Cat(val name: String, val owner: String) extends Pet:
  val sound: String = "meow"

class Dog(val name: String, val owner: String) extends Pet:
  val sound: String = "woof"

@main def run() =
  val cat = Cat("Whiskers", "Alice")
  val dog = Dog("Rex", "Bob")
  println(cat.introduce())
  println(dog.introduce())
```

`Pet` extends `Animal` and adds the abstract member `owner`, while  
providing a default `introduce` implementation that uses all three  
abstract members from the combined interface. `Cat` and `Dog` only need  
to satisfy the full abstract surface: `name`, `owner`, and `sound`. Trait  
inheritance allows incremental refinement of behaviour without forcing  
implementors to know about every ancestor in the hierarchy.  

## Traits extending classes  

A trait constrained to be mixed only into subclasses of a specific class.  

```scala
abstract class Entity(val id: Int):
  def describe: String = s"Entity($id)"

trait Auditable extends Entity:
  private var changeLog: List[String] = Nil
  def recordChange(msg: String): Unit =
    changeLog = s"[$id] $msg" :: changeLog
  def history: List[String] = changeLog.reverse

class Record(id: Int, val data: String) extends Entity(id) with Auditable:
  override def describe: String = s"Record($id, $data)"

@main def run() =
  val r = Record(1, "inventory-2024")
  r.recordChange("created")
  r.recordChange("updated quantity")
  println(r.describe)
  r.history.foreach(println)
```

When a trait extends a class, it can only be mixed into classes that also  
extend that class, ensuring the trait's requirements are always met.  
`Auditable` extends `Entity` and therefore has access to `id` from the  
parent class. `Record` satisfies both the `Entity(id)` constructor call  
and the `Auditable` mixin in a single `extends` clause. The private  
`changeLog` field is allocated per instance of `Record`, not shared  
across instances.  

## Self-types  

A trait that declares a dependency on another type without extending it.  

```scala
trait Repository:
  def findById(id: Int): Option[String]

trait UserService:
  self: Repository =>
  def getUser(id: Int): String =
    findById(id).getOrElse(s"<user $id not found>")

class UserStore extends Repository with UserService:
  private val store = Map(1 -> "Alice", 2 -> "Bob")
  def findById(id: Int): Option[String] = store.get(id)

@main def run() =
  val store = UserStore()
  println(store.getUser(1))
  println(store.getUser(42))
```

The `self: Repository =>` annotation inside `UserService` is a self-type  
declaration. It does not create an inheritance relationship; instead, it  
requires that any class mixing in `UserService` must also be a subtype of  
`Repository`. Within `UserService`, members of `Repository` are accessible  
through the alias `self` or simply as `this`. This separates the concerns  
of service logic from storage while enforcing the dependency at compile  
time without coupling the two traits through inheritance.  

## Dependency injection  

Using self-types to wire collaborating components into a single assembled  
service object.  

```scala
trait Database:
  def query(sql: String): List[String]

trait EmailSender:
  def send(to: String, body: String): Unit =
    println(s"Email -> $to: $body")

trait OrderService:
  self: Database with EmailSender =>
  def placeOrder(userId: String): Unit =
    val orders = query(s"SELECT * FROM orders WHERE user='$userId'")
    send(userId, s"You have ${orders.size} active order(s).")

class AppModule extends Database with EmailSender with OrderService:
  def query(sql: String): List[String] =
    List("order-101", "order-102")

@main def run() =
  AppModule().placeOrder("alice@example.com")
```

`OrderService` declares a self-type of `Database with EmailSender`,  
meaning it can use both collaborators as if they were mixed in, yet the  
three traits remain independently testable. `AppModule` assembles them  
all in one place. Swapping the `Database` implementation requires only  
changing that one class, leaving `OrderService` untouched. This pattern  
is the Scala equivalent of constructor injection and is known informally  
as the Cake pattern.  

## Traits with type parameters  

A trait parameterised by the type of element it operates on.  

```scala
trait Container[A]:
  def value: A
  def map[B](f: A => B): Container[B]
  def get: A = value

class Box[A](val value: A) extends Container[A]:
  def map[B](f: A => B): Container[B] = Box(f(value))
  override def toString: String = s"Box($value)"

@main def run() =
  val intBox: Container[Int]    = Box(42)
  val strBox: Container[String] = intBox.map(n => s"value=$n")
  println(intBox.get)
  println(strBox.get)
  println(strBox.map(_.length).get)
```

The `Container[A]` trait is generic over any element type `A`. The `map`  
method introduces a second type parameter `B`, allowing the container to  
change its element type through a pure transformation. `Box` provides a  
minimal concrete implementation and inherits `get` for free. Because  
`intBox` is typed as `Container[Int]`, the compiler statically verifies  
that only functions from `Int` are passed to `map`. Parameterised traits  
are the foundation for building reusable collection and wrapper  
abstractions.  

## Traits with givens  

A trait that exposes `given` instances to any class that mixes it in.  

```scala
trait Orderings:
  given Ordering[Int] with
    def compare(a: Int, b: Int): Int = a.compareTo(b)

  given byLength: Ordering[String] with
    def compare(a: String, b: String): Int = a.length - b.length

class Sorter extends Orderings:
  def sortInts(xs: List[Int]): List[Int] = xs.sorted
  def sortByLength(xs: List[String]): List[String] =
    xs.sorted(using byLength)

@main def run() =
  val s = Sorter()
  println(s.sortInts(List(5, 2, 8, 1, 9, 3)))
  println(s.sortByLength(List("banana", "fig", "cherry", "plum")))
```

The `Orderings` trait declares two `given` instances. When `Sorter`  
extends `Orderings`, those instances enter the implicit scope of every  
method in the class. The call to `xs.sorted` inside `sortInts` resolves  
against the `given Ordering[Int]` provided by the trait, while  
`sortByLength` passes `byLength` explicitly to make the choice clear.  
Traits are a convenient way to bundle related `given` definitions and  
share them across a module.  

## Traits with extension methods  

A trait that adds syntactic extension methods to an existing type.  

```scala
trait StringOps:
  extension (s: String)
    def shout: String    = s.toUpperCase + "!!!"
    def whisper: String  = s.toLowerCase + "..."
    def times(n: Int): String = s * n

object Syntax extends StringOps

@main def run() =
  import Syntax.*
  println("hello".shout)
  println("QUIET".whisper)
  println("go! ".times(3))
  println("ha".times(4).shout)
```

Extension methods defined inside a trait are inherited by any object or  
class that mixes it in. Importing `Syntax.*` brings the extensions into  
scope and makes them available on every `String` value without modifying  
the `String` class itself. `Syntax` is a plain singleton object that acts  
as a namespace for the extension package. Grouping extension methods in a  
trait lets you compose multiple extension sets from separate sources  
cleanly.  

## Traits with inline methods  

A trait that uses `inline def` to eliminate method call overhead at  
compile time.  

```scala
trait Validator[A]:
  inline def validate(a: A): Boolean
  inline def require(a: A, msg: String): Unit =
    if !validate(a) then throw IllegalArgumentException(msg)

object PositiveInt extends Validator[Int]:
  inline def validate(n: Int): Boolean = n > 0

object NonEmpty extends Validator[String]:
  inline def validate(s: String): Boolean = s.nonEmpty

@main def run() =
  PositiveInt.require(5, "must be positive")
  println("5 is valid")
  NonEmpty.require("Scala", "must not be empty")
  println("'Scala' is valid")
  try
    PositiveInt.require(-3, "must be positive")
  catch
    case e: IllegalArgumentException => println(e.getMessage)
```

Declaring `validate` and `require` as `inline` directs the compiler to  
substitute their bodies at every call site rather than emitting a normal  
virtual dispatch. This eliminates overhead for small validation predicates  
while preserving the abstraction. The abstract `inline def validate` in  
the trait forces implementors to also mark their version as `inline`,  
ensuring consistent inlining throughout the hierarchy. The concrete  
`require` in the trait is a non-abstract inline method that builds on  
the abstract one.  

## Private and protected members  

A trait that hides implementation details while exposing a controlled  
public interface.  

```scala
trait Counter:
  private var count = 0
  protected def increment(): Unit = count += 1
  protected def decrement(): Unit = count -= 1
  def current: Int   = count
  def reset(): Unit  = count = 0

class UpCounter extends Counter:
  def tick(): Unit = increment()

class BidirectionalCounter extends Counter:
  def up(): Unit   = increment()
  def down(): Unit = decrement()

@main def run() =
  val up = UpCounter()
  up.tick(); up.tick(); up.tick()
  println(s"Up counter: ${up.current}")
  val bi = BidirectionalCounter()
  bi.up(); bi.up(); bi.down()
  println(s"Bi counter: ${bi.current}")
  bi.reset()
  println(s"After reset: ${bi.current}")
```

The `private var count` is encapsulated inside the trait; nothing outside  
the trait can read or write it directly. The `protected` modifier on  
`increment` and `decrement` limits those methods to subclasses. `current`  
and `reset` remain public, forming the intended API surface. Each class  
that mixes in `Counter` receives its own independent `count` field stored  
in the instance, not shared across instances of different classes.  

## Overriding members  

A trait that overrides inherited behaviour from another trait and calls  
`super` to extend it.  

```scala
trait Notification:
  def message: String = "Default notification"
  def send(): Unit    = println(s"Sending: $message")

trait Urgent extends Notification:
  override def message: String = s"URGENT: ${super.message}"
  override def send(): Unit =
    println("*** HIGH PRIORITY ***")
    super.send()

class BasicNotification extends Notification
class AlertSystem        extends Urgent

@main def run() =
  val basic = BasicNotification()
  basic.send()
  println()
  val alert = AlertSystem()
  alert.send()
```

`Urgent` overrides both `message` and `send` and delegates to the parent  
trait via `super`. When `alert.send()` is called, the overriding version  
first prints the priority banner, then calls `super.send()`, which in turn  
uses the overridden `message`. The `override` keyword is mandatory in  
Scala when a concrete definition replaces another concrete one, making  
every override visible and preventing accidental shadowing of inherited  
behaviour.  

## Traits as capabilities  

Marker traits used as compile-time permission tokens on intersection  
types.  

```scala
trait CanRead
trait CanWrite
trait CanDelete

class FileHandle(val path: String)

def readFile(fh: FileHandle & CanRead): String =
  s"Reading ${fh.path}"

def writeFile(fh: FileHandle & CanWrite, data: String): Unit =
  println(s"Writing to ${fh.path}: $data")

def deleteFile(fh: FileHandle & CanDelete): Unit =
  println(s"Deleting ${fh.path}")

class ReadOnlyHandle(path: String) extends FileHandle(path) with CanRead
class ReadWriteHandle(path: String)
    extends FileHandle(path) with CanRead with CanWrite

@main def run() =
  val ro = ReadOnlyHandle("/etc/hosts")
  val rw = ReadWriteHandle("/tmp/output.txt")
  println(readFile(ro))
  println(readFile(rw))
  writeFile(rw, "hello!")
  // writeFile(ro, "x")   // compile error: CanWrite not satisfied
  // deleteFile(rw)        // compile error: CanDelete not satisfied
```

`CanRead`, `CanWrite`, and `CanDelete` carry no methods; they exist  
purely to express permission constraints in the type system. Functions  
that require a specific capability declare an intersection type such as  
`FileHandle & CanWrite`. The compiler statically rejects any call where  
the handle does not possess the required capability trait, preventing  
privilege-escalation errors at compile time with zero runtime overhead.  
`ReadWriteHandle` satisfies both `CanRead` and `CanWrite` by mixing in  
both traits.  

## Type-class style design  

A trait that forms the interface of a type class, with derived instances  
defined in the companion object.  

```scala
trait Show[A]:
  def show(a: A): String

object Show:
  given Show[Int] with
    def show(a: Int): String = a.toString

  given Show[String] with
    def show(a: String): String = s""""$a""""

  given Show[Boolean] with
    def show(a: Boolean): String = if a then "yes" else "no"

  given [A](using sa: Show[A]): Show[List[A]] with
    def show(xs: List[A]): String =
      xs.map(sa.show).mkString("[", ", ", "]")

def display[A: Show](a: A): String = summon[Show[A]].show(a)

@main def run() =
  println(display(42))
  println(display("hello"))
  println(display(true))
  println(display(List(1, 2, 3)))
  println(display(List("x", "y")))
```

The `Show[A]` trait is a classic type class: it abstracts over the notion  
of textual representation for any type `A`. Concrete instances are  
provided as `given` definitions in the companion object, which is the  
standard location for implicit search. The `display` function uses the  
`A: Show` context bound as a concise way to require a `Show[A]` instance.  
The `List[A]` instance is derived automatically from any existing  
`Show[A]`, demonstrating type-class composition without inheritance.  

## Traits with context functions  

A trait that introduces a context-function type alias to build a  
lightweight configuration DSL.  

```scala
case class Config(host: String, port: Int)

trait Configurable:
  type WithConfig[A] = Config ?=> A

  def configured[A](cfg: Config)(block: WithConfig[A]): A =
    given Config = cfg
    block

class App extends Configurable:
  def host(using cfg: Config): String = cfg.host
  def port(using cfg: Config): Int    = cfg.port
  def url(using cfg: Config): String  = s"https://${host}:${port}"

@main def run() =
  val app = App()
  val result = app.configured(Config("api.example.com", 443)):
    app.url
  println(result)
  val dev = app.configured(Config("localhost", 8080)):
    s"dev -> ${app.host}:${app.port}"
  println(dev)
```

The type alias `WithConfig[A] = Config ?=> A` describes a context  
function: a value that requires an implicit `Config` to produce an `A`.  
Inside `configured`, the `given Config = cfg` makes the provided config  
available for implicit resolution, and calling `block` evaluates the  
context function body. Methods such as `host` and `url` access the  
configuration through `using cfg: Config`, keeping call sites clean  
while requiring no explicit parameter threading at the use site.  

## Traits with match types  

A trait that uses a match type to select the output representation of  
a value at compile time.  

```scala
type Repr[A] = A match
  case Int    => String
  case Double => String
  case String => Int

trait Encoder:
  def encode[A](a: A): Repr[A]

object DefaultEncoder extends Encoder:
  def encode[A](a: A): Repr[A] =
    (a: @unchecked) match
      case n: Int    => n.toString.asInstanceOf[Repr[A]]
      case d: Double => f"$d%.4f".asInstanceOf[Repr[A]]
      case s: String => s.length.asInstanceOf[Repr[A]]

@main def run() =
  val enc = DefaultEncoder
  val s: String = enc.encode(42)
  val t: String = enc.encode(3.14159)
  val n: Int    = enc.encode("hello")
  println(s)
  println(t)
  println(n)
```

Match types are a Scala 3 feature that computes a result type from an  
input type using case-like rules evaluated at compile time. The `Repr[A]`  
alias maps `Int` and `Double` to `String`, and `String` to `Int`. The  
`Encoder` trait declares `encode` with the match type as its return type,  
giving the caller precise knowledge of the output type without overloaded  
signatures. The `asInstanceOf` casts in the implementation are safe  
because the runtime match mirrors the compile-time match type exactly.  

## Traits with opaque types  

A trait that defines opaque type aliases to create distinct, type-safe  
newtypes within a module boundary.  

```scala
trait NewtypeOps:
  opaque type UserId    = Int
  opaque type ProductId = Int

  object UserId:
    def apply(n: Int): UserId = n
    extension (id: UserId) def value: Int = id

  object ProductId:
    def apply(n: Int): ProductId = n
    extension (id: ProductId) def value: Int = id

object Domain extends NewtypeOps

@main def run() =
  import Domain.*
  val uid = UserId(42)
  val pid = ProductId(100)
  println(s"User: ${uid.value}, Product: ${pid.value}")
  // val bad: UserId = 42       // compile error outside the trait
  // val mixed: UserId = pid    // compile error - distinct types
```

Opaque types hide their underlying representation outside the scope in  
which they are declared. `UserId` and `ProductId` are both backed by  
`Int`, but from the caller's perspective they are completely distinct and  
non-interchangeable types. Placing the `opaque type` definitions inside a  
trait and sealing them into the `Domain` object allows multiple modules to  
reuse the same newtype pattern without sharing a single object scope. The  
companion objects and extension methods provide the only sanctioned way to  
construct values and access the underlying data.  

## Domain modeling  

Traits that capture a payment domain hierarchy with shared and specialised  
behaviour.  

```scala
trait PaymentMethod:
  def process(amount: BigDecimal): String

trait Refundable extends PaymentMethod:
  def refund(amount: BigDecimal): String

trait Recurring extends PaymentMethod:
  def schedule(amount: BigDecimal, intervalDays: Int): String

class CreditCard(number: String)
    extends Refundable with Recurring:
  private def last4 = number.takeRight(4)
  def process(amount: BigDecimal): String =
    f"Charged $$${amount}%.2f to card *$last4"
  def refund(amount: BigDecimal): String =
    f"Refunded $$${amount}%.2f to card *$last4"
  def schedule(amount: BigDecimal, days: Int): String =
    f"Scheduled $$${amount}%.2f every $days days"

class BankTransfer(iban: String) extends PaymentMethod:
  def process(amount: BigDecimal): String =
    f"Transferred $$${amount}%.2f to $iban"

@main def run() =
  val card = CreditCard("4111111111111234")
  println(card.process(BigDecimal("49.99")))
  println(card.refund(BigDecimal("9.99")))
  println(card.schedule(BigDecimal("9.99"), 30))
  val bank = BankTransfer("DE89370400440532013000")
  println(bank.process(BigDecimal("100.00")))
```

The domain is expressed as a hierarchy of traits rather than a flat  
enumeration. `PaymentMethod` defines the minimal capability; `Refundable`  
and `Recurring` add opt-in behaviours. `CreditCard` mixes in both,  
expressing that credit-card payments support all three capabilities.  
`BankTransfer` implements only the base payment, correctly excluding  
refund and recurring features. Callers can accept `Refundable` in a  
function signature, automatically rejecting bank transfers at compile  
time.  

## Functional programming patterns  

Traits that model `Functor` and `Foldable` abstractions for generic  
data structures.  

```scala
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

trait Foldable[F[_]]:
  def foldLeft[A, B](fa: F[A], z: B)(op: (B, A) => B): B
  def toList[A](fa: F[A]): List[A] =
    foldLeft(fa, List.empty[A])((acc, a) => acc :+ a)

given Functor[List] with
  def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)

given Foldable[List] with
  def foldLeft[A, B](fa: List[A], z: B)(op: (B, A) => B): B =
    fa.foldLeft(z)(op)

def sumSquares[F[_]: Functor: Foldable](fa: F[Int]): Int =
  val squared = summon[Functor[F]].map(fa)(x => x * x)
  summon[Foldable[F]].foldLeft(squared, 0)(_ + _)

@main def run() =
  println(sumSquares(List(1, 2, 3, 4)))
  println(summon[Foldable[List]].toList(List(10, 20, 30)))
```

`Functor` and `Foldable` are higher-kinded traits that abstract over type  
constructors `F[_]` rather than simple types. The `given` blocks wire  
`List` as an instance of both abstractions. `sumSquares` is parameterised  
over any `F` that has both a `Functor` and a `Foldable` instance,  
demonstrating how multiple context bounds compose cleanly. This approach  
mirrors patterns found in functional Scala libraries such as Cats but  
uses only the standard library.  

## Modular architecture  

Traits used as self-contained module layers assembled into a single  
application object.  

```scala
trait LoggingModule:
  trait Logger:
    def info(msg: String): Unit
  def logger: Logger

trait ConsoleLogging extends LoggingModule:
  val logger: Logger = new Logger:
    def info(msg: String): Unit = println(s"[INFO] $msg")

trait UserModule:
  self: LoggingModule =>
  def createUser(name: String): Unit =
    logger.info(s"Creating user: $name")
  def deleteUser(name: String): Unit =
    logger.info(s"Deleting user: $name")

object Application extends ConsoleLogging with UserModule

@main def run() =
  Application.createUser("Alice")
  Application.createUser("Bob")
  Application.deleteUser("Alice")
```

Each module trait defines an inner interface (`Logger`) and either an  
abstract or a concrete member that provides it. `ConsoleLogging` supplies  
the concrete logger; `UserModule` declares a self-type that requires a  
`LoggingModule`, ensuring that any wiring of `UserModule` must also  
provide a logger. `Application` resolves all dependencies in one place.  
Replacing `ConsoleLogging` with a file-based or structured logging  
module requires changing only the `Application` object.  

## Mixin composition  

Composing reusable cross-cutting concerns through independent traits  
applied at the class definition site.  

```scala
trait Logging:
  def log(msg: String): Unit = println(s"[LOG] $msg")

trait Caching:
  private val cache = scala.collection.mutable.Map.empty[String, String]
  def cached(key: String)(compute: => String): String =
    cache.getOrElseUpdate(key, compute)

trait Retrying:
  def retry[A](times: Int)(op: => A): A =
    var attempt = 0
    var last: Throwable = RuntimeException("no attempt")
    while attempt < times do
      try return op
      catch
        case e: Throwable =>
          last = e
          attempt += 1
    throw last

class DataService extends Logging with Caching with Retrying:
  def fetch(key: String): String =
    cached(key):
      retry(3):
        log(s"Fetching $key")
        s"data:$key"

@main def run() =
  val svc = DataService()
  println(svc.fetch("user:1"))
  println(svc.fetch("user:1"))
  println(svc.fetch("product:5"))
```

`DataService` accumulates three independent behaviours without inheriting  
from a common base class. `Logging`, `Caching`, and `Retrying` are each  
self-contained and can be unit-tested in isolation. The `cached` helper  
memoises results using a `Map` stored per instance, while `retry`  
reattempts a failing operation and rethrows the last exception on full  
exhaustion. Composing these traits is declarative: the class body focuses  
entirely on the domain logic.  

## API constraints  

Traits used as fine-grained permission types checked by the compiler at  
call sites.  

```scala
trait ReadAccess
trait WriteAccess
trait AdminAccess extends ReadAccess with WriteAccess

def readData()(using ReadAccess): String     = "payload"
def writeData(data: String)(using WriteAccess): Unit =
  println(s"Persisting: $data")
def purgeAll()(using AdminAccess): Unit =
  println("All data purged.")

object GuestToken extends ReadAccess
object AdminToken  extends AdminAccess

@main def run() =
  locally:
    given GuestToken.type = GuestToken
    println(readData())
    // writeData("x")  // compile error – WriteAccess not in scope
  locally:
    given AdminToken.type = AdminToken
    println(readData())
    writeData("record-1")
    purgeAll()
```

Each capability is a marker trait with no methods. A function that  
requires a capability declares it in a `using` clause; the compiler  
rejects the call if no matching `given` instance is in scope. `AdminAccess`  
extends both `ReadAccess` and `WriteAccess`, so a single `AdminToken`  
satisfies all three capability functions. The `locally` blocks create  
separate implicit scopes to demonstrate how capability availability  
changes with lexical scope boundaries.  

## Java interface interoperability  

A Scala trait that wraps `java.lang.Comparable` to provide a  
Scala-friendly comparison contract.  

```scala
import java.util.TreeSet

trait NaturalOrder extends java.lang.Comparable[NaturalOrder]:
  def rank: Int
  override def compareTo(other: NaturalOrder): Int = rank - other.rank

class Priority(val name: String, val rank: Int) extends NaturalOrder:
  override def toString: String = s"$name(rank=$rank)"

class Severity(val label: String, val rank: Int) extends NaturalOrder:
  override def toString: String = s"$label(severity=$rank)"

@main def run() =
  val set = TreeSet[NaturalOrder]()
  set.add(Priority("low",    3))
  set.add(Priority("high",   1))
  set.add(Priority("medium", 2))
  set.add(Severity("info",   4))
  set.add(Severity("error",  0))
  val it = set.iterator()
  while it.hasNext() do println(it.next())
```

`NaturalOrder` extends `java.lang.Comparable[NaturalOrder]` and  
implements `compareTo` in terms of the abstract `rank` member. Any class  
that extends `NaturalOrder` automatically satisfies the Java `Comparable`  
contract. `TreeSet` from the Java standard library sorts its elements  
using `compareTo`, so the mixed collection of `Priority` and `Severity`  
instances is sorted purely by rank without any extra configuration. This  
shows how a Scala trait can implement a Java interface transparently.  

## Observer design pattern  

Traits that model the Observer pattern for a typed domain event bus.  

```scala
enum OrderEvent:
  case Placed(orderId: String)
  case Shipped(orderId: String)

trait Observer[A]:
  def update(event: A): Unit

trait Observable[A]:
  private var observers: List[Observer[A]] = Nil
  def subscribe(o: Observer[A]): Unit =
    observers = o :: observers
  protected def notify(event: A): Unit =
    observers.foreach(_.update(event))

class OrderService extends Observable[OrderEvent]:
  def place(id: String): Unit =
    println(s"Placing order $id")
    notify(OrderEvent.Placed(id))
  def ship(id: String): Unit =
    println(s"Shipping order $id")
    notify(OrderEvent.Shipped(id))

class EmailNotifier extends Observer[OrderEvent]:
  def update(e: OrderEvent): Unit = println(s"[Email] $e")

class AuditLogger extends Observer[OrderEvent]:
  def update(e: OrderEvent): Unit = println(s"[Audit] $e")

@main def run() =
  val svc = OrderService()
  svc.subscribe(EmailNotifier())
  svc.subscribe(AuditLogger())
  svc.place("ORD-001")
  svc.ship("ORD-001")
```

The pattern is split across two traits. `Observer[A]` defines the single  
`update` callback, while `Observable[A]` manages the subscriber list and  
dispatches events. `OrderService` mixes in `Observable` and calls  
`notify` at appropriate points in its own methods. `EmailNotifier` and  
`AuditLogger` are completely independent and can be added or removed  
without touching `OrderService`. Using `enum` for `OrderEvent` ensures  
the set of events is closed and exhaustive.  

## Advanced trait hierarchies  

A layered trait hierarchy combining inheritance, overriding, and  
linearised `super` calls in a realistic domain model.  

```scala
trait Base:
  def name: String
  override def toString: String = name

trait Printable extends Base:
  def print(): Unit = println(s"[Printable] $name")

trait Saveable extends Base:
  private var saved = false
  def save(): Unit =
    saved = true
    println(s"[Saved] $name")
  def isSaved: Boolean = saved

trait Audited extends Printable with Saveable:
  private var log: List[String] = Nil
  override def save(): Unit =
    log = s"saved:${System.currentTimeMillis()}" :: log
    super.save()
  def history: List[String] = log.reverse

trait Versioned extends Base:
  val version: Int
  override def toString: String = s"${super.toString} v$version"

class Document(val name: String, val version: Int)
    extends Audited with Versioned

@main def run() =
  val doc = Document("specification", 3)
  doc.print()
  doc.save()
  doc.save()
  println(doc.isSaved)
  println(doc.history.mkString(", "))
  println(doc)
```

`Document` extends `Audited` and `Versioned`, which themselves form a  
deep hierarchy. The linearisation order is: `Document → Versioned →  
Audited → Saveable → Printable → Base`. When `doc.save()` is called,  
`Audited.save` runs first, appends to the log, then calls `super.save()`,  
which resolves to `Saveable.save`. The `toString` in `Versioned` chains  
to `super.toString` (which reaches `Base.toString`) and appends the  
version number. This example shows how cooperative trait design, where  
every override calls `super`, enables powerful stacking across a  
multi-level hierarchy.  
