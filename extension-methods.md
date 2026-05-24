# Extension Methods in Scala 3  

An extension method in Scala 3 lets you add new methods to an existing  
type without modifying its source code or creating a wrapper class. The  
`extension` keyword introduces a block that binds methods to a receiver  
type, letting callers invoke those methods using familiar dot notation.  
Extension methods are resolved statically and carry no runtime overhead  
beyond a normal method call.  

In Scala 2, the idiomatic way to enrich a type was to write an implicit  
conversion class that wrapped the original value and exposed additional  
methods. This approach was verbose, relied on the `implicit` keyword,  
and could conflict with other implicit conversions already in scope.  
Scala 3 replaces implicit classes entirely with first-class extension  
methods that are explicit, composable, and more predictable during  
resolution.  

Extension methods improve API readability by allowing behaviour to live  
alongside the concept it serves, even when the type originates in a  
library you cannot change. A well-designed API can expose fluent,  
chained calls on standard types such as `String`, `Int`, or `List`  
without sacrificing type safety. This keeps call sites clean and  
expressive while the implementation remains in a clearly defined  
location.  

Type inference interacts naturally with extension methods. When the  
compiler resolves a call `x.foo`, it first checks the static type of  
`x`, then searches the implicit scope for extension methods that match  
the receiver type. Generic type parameters in extension definitions are  
inferred from the argument at the call site, so callers rarely need to  
supply type arguments explicitly.  

Extension methods compose seamlessly with other Scala 3 features.  
Generic extensions can carry type bounds and context bounds, letting  
you require typeclass evidence inline. Inside typeclass traits, extension  
methods become the standard mechanism for providing syntax to any type  
that has a given instance. Opaque types gain safe, zero-cost behaviour  
exclusively through extension methods, because the underlying  
representation is hidden everywhere outside the definition scope.  

Extension methods are appropriate when you want to add operations to a  
type you do not own, when you want to provide typeclass-driven syntax,  
or when you are modeling a domain with opaque type wrappers. Avoid them  
when ordinary method definitions on your own class would be clearer.  
Used thoughtfully, extension methods make Scala 3 code significantly  
more readable and maintainable.  

---  

## A simple extension method  

A minimal extension adds one new method to an existing type.  

```scala
extension (s: String)
    def greet: String = s"Hello, $s!"

@main def main() =
    val name = "Scala"
    println(name.greet)
    println("World".greet)
end main
```

The `extension` keyword takes a parameter that acts as the receiver  
value. The method `greet` behaves exactly like an instance method on  
`String`, even though `String` is a library type we cannot modify. At  
the call site `name.greet`, the compiler desugars the call into the  
equivalent free-function form `greet(name)`. No wrapper object is  
allocated at runtime.  

## Extension methods with parameters  

Extension methods accept additional parameters alongside the receiver.  

```scala
extension (s: String)
    def repeatStr(n: Int): String = s * n
    def padLeft(width: Int, ch: Char = ' '): String =
        s.reverse.padTo(width, ch).reverse

@main def main() =
    println("ha".repeatStr(3))
    println("42".padLeft(6, '0'))
    println("hi".padLeft(8))
end main
```

The receiver `s` appears in the `extension` clause, while extra  
parameters follow in the method signature. Multiple methods sharing  
the same receiver type can live in one `extension` block, avoiding  
repetition of the receiver declaration. Default parameter values  
behave identically to those on ordinary methods.  

## Extension methods returning values  

An extension method can compute and return any Scala expression.  

```scala
extension (n: Int)
    def digits: List[Int] =
        n.abs.toString.map(_.asDigit).toList

    def isPrime: Boolean =
        if n < 2 then false
        else (2 to math.sqrt(n.toDouble).toInt).forall(n % _ != 0)

@main def main() =
    println(12345.digits)
    println(7.isPrime)
    println(10.isPrime)
end main
```

`digits` converts the integer to a string, maps each character to its  
digit value, and returns a `List[Int]`. The return type is inferred  
from the body, so no explicit annotation is needed. `isPrime` returns  
a `Boolean` computed by trial division up to the square root of `n`.  
Both methods integrate naturally with dot-call syntax on `Int` literals.  

## Extension methods on built-in types  

Multiple utility methods can enrich the standard `Int` type in one block.  

```scala
extension (n: Int)
    def isEven: Boolean             = n % 2 == 0
    def isOdd: Boolean              = n % 2 != 0
    def squared: Int                = n * n
    def clamp(lo: Int, hi: Int): Int =
        if n < lo then lo else if n > hi then hi else n

@main def main() =
    println(4.isEven)
    println(7.isOdd)
    println(9.squared)
    println(15.clamp(0, 10))
    println((-3).clamp(0, 10))
end main
```

All four methods share the single `n: Int` receiver declared at the  
top of the block. The `isEven` and `isOdd` predicates read naturally  
at call sites, replacing repetitive inline expressions. `clamp` takes  
two extra parameters and restricts the value to a closed interval, a  
common need in graphics, animation, and game programming.  

## Extension methods on custom types  

Extension methods work equally well on case classes and other user-defined types.  

```scala
case class Circle(radius: Double)

extension (c: Circle)
    def area: Double             = math.Pi * c.radius * c.radius
    def perimeter: Double        = 2 * math.Pi * c.radius
    def scale(f: Double): Circle = Circle(c.radius * f)

@main def main() =
    val circle = Circle(5.0)
    println(f"Area:      ${circle.area}%.2f")
    println(f"Perimeter: ${circle.perimeter}%.2f")
    println(f"Scaled:    ${circle.scale(2.0).radius}%.1f")
end main
```

The `Circle` case class holds only data, while all derived behaviour  
lives in the `extension` block. This separation of data and behaviour  
mirrors the functional style common in Scala 3. `scale` returns a new  
`Circle`, preserving the immutability of the original value. The  
approach works for any case class, trait implementation, or opaque type.  

## Multiple extensions in one block  

Grouping related methods in one block clarifies that they all concern the same receiver.  

```scala
case class Temperature(celsius: Double)

extension (t: Temperature)
    def toFahrenheit: Double = t.celsius * 9.0 / 5.0 + 32.0
    def toKelvin: Double     = t.celsius + 273.15
    def isFreezing: Boolean  = t.celsius <= 0.0
    def isBoiling: Boolean   = t.celsius >= 100.0
    def +(other: Temperature): Temperature =
        Temperature(t.celsius + other.celsius)

@main def main() =
    val boiling = Temperature(100.0)
    val body    = Temperature(37.0)
    println(f"Fahrenheit: ${boiling.toFahrenheit}%.1f")
    println(f"Kelvin:     ${boiling.toKelvin}%.2f")
    println(s"Freezing:   ${Temperature(-10.0).isFreezing}")
    println(s"Sum:        ${(boiling + body).celsius}")
end main
```

Placing multiple methods in one `extension` block communicates that  
they all belong to the same concept. The `+` operator is defined as  
an extension, which is idiomatic in Scala 3 for domain types that  
need arithmetic semantics. Every method in the block can freely read  
`t.celsius` because they all share the same receiver declaration.  

## Extension methods with default parameters  

Default values make extension methods flexible without requiring many overloads.  

```scala
extension (s: String)
    def truncate(maxLen: Int = 10, ellipsis: String = "..."): String =
        if s.length <= maxLen then s
        else s.take(maxLen) + ellipsis
    def wrap(prefix: String = "[", suffix: String = "]"): String =
        prefix + s + suffix

@main def main() =
    val text = "Hello, Scala 3 world!"
    println(text.truncate())
    println(text.truncate(5))
    println(text.truncate(5, "…"))
    println("hi".wrap())
    println("hi".wrap("<", ">"))
end main
```

Default parameters in extension methods work identically to those in  
ordinary methods. Callers may invoke `truncate()` without arguments  
or supply only the first one, leaving the rest at their defaults. This  
reduces the number of overloads the API author must write while keeping  
the call site concise and readable.  

## Generic extension methods  

A type parameter on the extension makes the method work for any element type.  

```scala
extension [A](xs: List[A])
    def second: Option[A] =
        xs match
            case _ :: x :: _ => Some(x)
            case _           => None

    def penultimate: Option[A] =
        if xs.length >= 2 then Some(xs(xs.length - 2)) else None

@main def main() =
    println(List(10, 20, 30).second)
    println(List(1).second)
    println(List("a", "b", "c", "d").penultimate)
    println(List("only").penultimate)
end main
```

The type parameter `[A]` appears in square brackets before the receiver  
clause. The compiler infers `A` from the concrete element type of the  
list at each call site. Both methods return `Option[A]` to safely  
handle the case where the list is too short, avoiding exceptions.  
Pattern matching on the list structure keeps the implementation concise.  

## Extension methods with type parameters  

Type parameters may also appear on individual methods within the extension block.  

```scala
extension [A](value: A)
    def pipe[B](f: A => B): B =
        f(value)
    def tap(f: A => Unit): A =
        f(value)
        value

@main def main() =
    val result =
        "  hello world  "
            .pipe(_.trim)
            .pipe(_.split(" ").toList)
            .tap(ws => println(s"Words: ${ws.length}"))
            .pipe(_.map(_.capitalize))
            .pipe(_.mkString(", "))
    println(result)
end main
```

`pipe` transforms a value through a function and returns the result,  
while `tap` applies a side-effecting function and returns the original  
value unchanged. Both methods carry their own type parameter in addition  
to the receiver's `[A]`. Chaining them produces a top-to-bottom pipeline  
that makes the data flow easy to trace without intermediate variables.  

## Extension methods with bounds  

Type bounds restrict which types the extension applies to.  

```scala
extension [A: Ordering](xs: List[A])
    def minMax: Option[(A, A)] =
        if xs.isEmpty then None
        else Some((xs.min, xs.max))

    def isSorted: Boolean =
        xs.zip(xs.tail).forall(summon[Ordering[A]].lteq)

@main def main() =
    val nums = List(3, 1, 4, 1, 5, 9, 2, 6)
    println(nums.minMax)
    println(List(1, 2, 3).isSorted)
    println(List(3, 1, 2).isSorted)
    println(List("apple", "cherry", "mango").minMax)
end main
```

The context bound `[A: Ordering]` requires the compiler to locate a  
`given Ordering[A]` instance for the element type. `xs.min` and  
`xs.max` consume this evidence implicitly, while `isSorted` retrieves  
it explicitly via `summon`. Standard types such as `Int` and `String`  
already carry `Ordering` instances, so no extra `given` is needed at  
the call site.  

## Extension methods on collections  

Extensions can add aggregate operations that are absent from the standard library.  

```scala
extension [A](xs: List[A])
    def frequencies: Map[A, Int] =
        xs.groupBy(identity).view.mapValues(_.length).toMap

    def sliding2: List[(A, A)] =
        xs.zip(xs.tail)

    def intersperse(sep: A): List[A] =
        xs match
            case Nil | _ :: Nil => xs
            case head :: tail   =>
                head :: tail.flatMap(x => List(sep, x))

@main def main() =
    val letters = List('a', 'b', 'a', 'c', 'b', 'a')
    println(letters.frequencies)
    println(List(1, 2, 3, 4).sliding2)
    println(List(1, 2, 3).intersperse(0))
end main
```

`frequencies` groups equal elements and maps each group to its count,  
building a histogram in one expression. `sliding2` produces consecutive  
pairs from the list, useful for computing deltas or detecting adjacent  
duplicates. `intersperse` inserts a separator element between every  
pair of adjacent elements, a standard operation when constructing  
formatted output from a list of values.  

## Extension methods on tuples  

Extension methods work on product types such as two-element tuples.  

```scala
extension [A, B](pair: (A, B))
    def swap: (B, A)                    = (pair._2, pair._1)
    def mapFirst[C](f: A => C): (C, B)  = (f(pair._1), pair._2)
    def mapSecond[C](f: B => C): (A, C) = (pair._1, f(pair._2))

@main def main() =
    val p = ("hello", 42)
    println(p.swap)
    println(p.mapFirst(_.toUpperCase))
    println(p.mapSecond(_ * 2))
    val q = (3, 7)
    println(q.mapFirst(_ + 1).mapSecond(_ - 1))
end main
```

Two independent type parameters `[A, B]` let the compiler track each  
element type separately. `mapFirst` and `mapSecond` each introduce an  
additional type parameter `[C]` for the transformed element. The  
extension enables a clean, functional style for manipulating pairs  
without unpacking and repacking them manually at every call site.  

## Extension methods with overloading  

Multiple methods with different parameter types constitute overloads in the usual sense.  

```scala
case class Vec2(x: Double, y: Double):
    override def toString: String = f"Vec2($x%.2f, $y%.2f)"

extension (v: Vec2)
    def +(other: Vec2): Vec2     = Vec2(v.x + other.x, v.y + other.y)
    def -(other: Vec2): Vec2     = Vec2(v.x - other.x, v.y - other.y)
    def *(scalar: Double): Vec2  = Vec2(v.x * scalar, v.y * scalar)
    def magnitude: Double        = math.sqrt(v.x * v.x + v.y * v.y)
    def normalised: Vec2 =
        val mag = v.magnitude
        if mag == 0.0 then Vec2(0, 0) else Vec2(v.x / mag, v.y / mag)

@main def main() =
    val v1 = Vec2(3.0, 4.0)
    val v2 = Vec2(1.0, 2.0)
    println(v1 + v2)
    println(v1 - v2)
    println(v1 * 2.0)
    println(f"Magnitude: ${v1.magnitude}%.2f")
    println(v1.normalised)
end main
```

The extension block defines three operators and two query methods on  
`Vec2`. Operator names such as `+`, `-`, and `*` follow Scala's infix  
call convention, so `v1 + v2` desugars to `v1.+(v2)`. Multiple methods  
with different parameter types constitute overloads resolved at compile  
time. Defining operators through extensions keeps the `case class` body  
focused on data only.  

## Extension methods inside a typeclass  

In Scala 3, typeclass syntax is expressed by placing extensions directly inside the trait.  

```scala
trait Show[A]:
    extension (a: A)
        def show: String

given Show[Int] with
    extension (n: Int)
        def show: String = s"Int($n)"

given Show[Boolean] with
    extension (b: Boolean)
        def show: String = if b then "yes" else "no"

given Show[List[Int]] with
    extension (xs: List[Int])
        def show: String = xs.map(_.show).mkString("[", ", ", "]")

@main def main() =
    println(42.show)
    println(false.show)
    println(List(1, 2, 3).show)
end main
```

Placing the extension inside the typeclass trait means that any `given`  
instance of `Show[A]` automatically brings `show` into scope as a  
method on values of type `A`. This is the canonical Scala 3 pattern for  
typeclass-derived syntax. The `given Show[List[Int]]` instance delegates  
to `Show[Int]` by calling `.show` on each element, demonstrating how  
typeclass instances compose naturally.  

## Extension methods using givens  

A given instance can provide extension methods that depend on implicit context.  

```scala
trait Describable[A]:
    def describe(a: A): String

given Describable[String] with
    def describe(s: String): String =
        s"String(\"$s\", length=${s.length})"

given Describable[Int] with
    def describe(n: Int): String =
        s"Int($n, ${if n >= 0 then "non-negative" else "negative"})"

extension [A](a: A)(using d: Describable[A])
    def describe: String = d.describe(a)

@main def main() =
    println("hello".describe)
    println((-7).describe)
end main
```

The extension carries a `using` clause requiring a `Describable[A]`  
context parameter. The compiler locates the appropriate given instance  
based on the concrete type of `a` at each call site. This pattern keeps  
the typeclass interface separate from the extension syntax that makes it  
available as a dot-call, which clarifies the contract at both definition  
and use points.  

## Extension methods as typeclass syntax  

Context bounds provide concise shorthand for typeclass-driven extensions.  

```scala
trait Printable[A]:
    def format(a: A): String

given Printable[Double] with
    def format(d: Double): String = f"$d%.4f"

given Printable[List[String]] with
    def format(xs: List[String]): String = xs.mkString("{", ", ", "}")

extension [A: Printable](a: A)
    def formatted2: String = summon[Printable[A]].format(a)

@main def main() =
    println(3.14159265.formatted2)
    println(List("one", "two", "three").formatted2)
end main
```

The context bound `[A: Printable]` desugars to a `using` parameter  
`(using Printable[A])`, which the compiler resolves from the given  
scope. `summon[Printable[A]]` retrieves the instance so the extension  
can forward the call. At the call site, `3.14159265.formatted2` reads  
naturally without any explicit type arguments, which is the primary  
benefit of combining context bounds with extension methods.  

## Extension methods with summon  

Calling `summon` explicitly gives full control over typeclass dispatch.  

```scala
trait Codec[A]:
    def encode(a: A): String
    def decode(s: String): Option[A]

given Codec[Int] with
    def encode(n: Int): String         = n.toString
    def decode(s: String): Option[Int] = s.toIntOption

given Codec[Boolean] with
    def encode(b: Boolean): String = if b then "1" else "0"
    def decode(s: String): Option[Boolean] =
        s match
            case "1" => Some(true)
            case "0" => Some(false)
            case _   => None

extension [A](a: A)(using c: Codec[A])
    def encode: String = c.encode(a)

extension (s: String)
    def decodeAs[A](using c: Codec[A]): Option[A] = c.decode(s)

@main def main() =
    println(42.encode)
    println(true.encode)
    println("42".decodeAs[Int])
    println("0".decodeAs[Boolean])
end main
```

Here the given instance is named `c` and called directly rather than  
through `summon`. The `decodeAs` extension lives on `String` rather  
than on `A`, so it takes the target type as a type argument at the call  
site. This shows that extensions can bridge different types by pairing  
a fixed receiver with a typeclass parameter from the given scope.  

## Extension methods with context bounds  

Multiple context bounds let an extension require several typeclass instances at once.  

```scala
extension [A: Ordering: Numeric](xs: List[A])
    def sortedSum: A =
        val num = summon[Numeric[A]]
        xs.sorted.foldLeft(num.zero)(num.plus)

    def normalised: List[Double] =
        val num   = summon[Numeric[A]]
        val total = xs.foldLeft(num.zero)(num.plus)
        val d     = num.toDouble(total)
        if d == 0.0 then xs.map(_ => 0.0)
        else xs.map(x => num.toDouble(x) / d)

@main def main() =
    println(List(3, 1, 4, 1, 5).sortedSum)
    println(List(1, 2, 3, 4).normalised.map(v => f"$v%.2f"))
end main
```

Two context bounds `[A: Ordering: Numeric]` desugar to two implicit  
parameters the compiler resolves automatically. `Ordering` enables  
`xs.sorted`, while `Numeric` provides `zero`, `plus`, and `toDouble`.  
Using `summon` to name the instances explicitly avoids ambiguity when  
both typeclasses need to be called within the same method body.  

## Adding behavior to opaque types  

Extension methods are the sole mechanism for adding operations to opaque types outside their file.  

```scala
opaque type Meters = Double

object Meters:
    def apply(d: Double): Meters = d

extension (m: Meters)
    def value: Double             = m
    def toKilometers: Double      = m / 1000.0
    def toMiles: Double           = m / 1609.344
    def +(other: Meters): Meters  = m + other
    def *(factor: Double): Meters = m * factor

@main def main() =
    val dist = Meters(1500.0)
    println(s"Value:      ${dist.value} m")
    println(f"Kilometers: ${dist.toKilometers}%.3f km")
    println(f"Miles:      ${dist.toMiles}%.4f mi")
    println(s"Total:      ${(dist + Meters(500.0)).value} m")
end main
```

Outside the file that defines `Meters`, the compiler treats it as an  
opaque type and hides its `Double` representation. Extensions defined  
in the same file can access the representation because they share the  
same lexical scope. The `value` accessor is the controlled way for  
external code to retrieve the underlying number. The `+` and `*`  
operators return `Meters`, preserving the opaque type boundary.  

## Extension methods inside opaque type companions  

Placing extensions in the companion object gives them automatic availability via implicit scope.  

```scala
opaque type Email = String

object Email:
    def apply(s: String): Option[Email] =
        if s.contains("@") && s.contains(".") then Some(s) else None

    def unsafe(s: String): Email = s

    extension (e: Email)
        def local: String  = e.split("@").head
        def domain: String = e.split("@").last
        def value: String  = e

@main def main() =
    Email("alice@example.com") match
        case Some(email) =>
            println(s"Local:  ${email.local}")
            println(s"Domain: ${email.domain}")
            println(s"Full:   ${email.value}")
        case None =>
            println("Invalid email address")
end main
```

Placing the extension inside `object Email` means the methods become  
available automatically whenever an `Email` value is in scope, because  
the companion's implicit scope is always searched by the compiler. This  
is the idiomatic pattern for opaque type companions: validate and  
construct in `apply`, then expose all derived behaviour through companion  
extensions. The `unsafe` constructor is provided for cases where the  
value is known to be valid at compile time.  

## Extension methods for domain modeling  

Opaque types combined with extensions enforce domain invariants at compile time.  

```scala
opaque type Percentage = Double

object Percentage:
    def apply(v: Double): Percentage =
        require(v >= 0.0 && v <= 100.0, s"Invalid percentage: $v")
        v

    val Zero: Percentage    = 0.0
    val Hundred: Percentage = 100.0

    extension (p: Percentage)
        def value: Double              = p
        def asFraction: Double         = p / 100.0
        def complement: Percentage     = 100.0 - p
        def of(amount: Double): Double = amount * p / 100.0
        def +(q: Percentage): Percentage = (p + q).min(100.0)

@main def main() =
    val discount = Percentage(15.0)
    val price    = 200.0
    println(f"Discount:   ${discount.of(price)}%.2f")
    println(f"Fraction:   ${discount.asFraction}%.2f")
    println(f"Complement: ${discount.complement.value}%.1f%%")
    println(f"Combined:   ${(discount + Percentage(10.0)).value}%.1f%%")
end main
```

The `require` call in `apply` ensures that no `Percentage` outside  
0–100 can ever be constructed. Extensions provide all the arithmetic  
and derived views callers need. The `complement` and `+` methods return  
a `Percentage`, keeping the results within the valid range. This is a  
textbook example of making illegal states unrepresentable through the  
type system while keeping the API ergonomic.  

## Extension methods for zero-cost abstractions  

Distinct opaque types over the same primitive prevent mix-ups at zero runtime cost.  

```scala
opaque type UserId  = Long
opaque type OrderId = Long

object UserId:
    def apply(n: Long): UserId = n
    extension (u: UserId)
        def value: Long     = u
        def display: String = s"USR-$u"

object OrderId:
    def apply(n: Long): OrderId = n
    extension (o: OrderId)
        def value: Long     = o
        def display: String = s"ORD-$o"

@main def main() =
    val uid = UserId(1001L)
    val oid = OrderId(2001L)
    println(uid.display)
    println(oid.display)
    def processOrder(id: OrderId): Unit =
        println(s"Processing ${id.display}")
    processOrder(oid)
    // processOrder(uid)  // compile error: type mismatch
end main
```

At runtime, both `UserId` and `OrderId` are plain `Long` values on the  
JVM. No wrapper object is allocated and no boxing occurs. At compile  
time they are fully distinct types, so accidentally passing a `UserId`  
where an `OrderId` is expected produces a type error. Extension methods  
provide the `display` and `value` accessors at exactly zero overhead,  
achieving true zero-cost type safety through the type system alone.  

## Extension methods with inline  

The `inline` modifier asks the compiler to expand the method body at every call site.  

```scala
extension (n: Int)
    inline def isPositive: Boolean   = n > 0
    inline def abs: Int              = if n >= 0 then n else -n
    inline def clamp(lo: Int, hi: Int): Int =
        if n < lo then lo else if n > hi then hi else n

@main def main() =
    val x = -7
    println(x.abs)
    println(x.isPositive)
    println(15.clamp(0, 10))
    println((-3).clamp(0, 10))
    println(List(-5, 0, 3, 12).map(_.clamp(0, 10)))
end main
```

`inline` extension methods are expanded by the compiler before code  
generation, so the JVM sees only the inlined arithmetic expressions,  
not method dispatch. This matters for hot paths where even a single  
function call overhead is undesirable. `abs` and `clamp` are prime  
candidates because they contain trivially small bodies that are  
commonly called inside tight loops or in bulk over collections.  

## Extension methods with match types  

A match type expresses a return type that depends structurally on the input type.  

```scala
type ElementType[C] = C match
    case List[a]  => a
    case Array[a] => a
    case String   => Char

extension [C](c: C)
    def firstElement: ElementType[C] =
        (c match
            case xs: List[?]  => xs.head
            case xs: Array[?] => xs.head
            case s: String    => s.head
        ).asInstanceOf[ElementType[C]]

@main def main() =
    val n: Int    = List(1, 2, 3).firstElement
    val ch: Char  = "Scala".firstElement
    val d: Double = Array(3.14, 2.71).firstElement
    println(n)
    println(ch)
    println(d)
end main
```

`ElementType[C]` is a match type that reduces to a different concrete  
type depending on `C`. The extension method's declared return type is  
`ElementType[C]`, which the compiler refines at each call site before  
type-checking the rest of the expression. The `asInstanceOf` cast  
bridges the runtime pattern match and the compile-time match type; the  
two reductions mirror each other exactly, so the cast is always safe.  

## Extension methods with union types  

A union receiver type allows a single extension to accept several concrete types.  

```scala
extension (v: Int | Double | String)
    def stringify: String =
        v match
            case n: Int    => s"int:$n"
            case d: Double => f"double:$d%.2f"
            case s: String => s"string:$s"

    def toDouble2: Double =
        v match
            case n: Int    => n.toDouble
            case d: Double => d
            case s: String => s.toDoubleOption.getOrElse(0.0)

@main def main() =
    val items: List[Int | Double | String] = List(42, 3.14, "2.71")
    items.foreach(item => println(item.stringify))
    println(items.map(_.toDouble2).sum)
end main
```

The receiver type `Int | Double | String` is a union type, meaning the  
extension applies to any value whose type is one of those three members.  
The pattern match inside the body is exhaustive because the compiler  
knows all possible cases from the union declaration. Union types avoid  
the need for a common interface while still providing shared operations  
over a bounded set of types.  

## Extension methods with dependent types  

A dependent method type lets the return type vary based on the receiver's type member.  

```scala
trait Container:
    type Item
    val value: Item

case class IntBox(value: Int) extends Container:
    type Item = Int

case class StrBox(value: String) extends Container:
    type Item = String

extension (c: Container)
    def transform(f: c.Item => String): String = f(c.value)
    def twice(f: c.Item => c.Item): c.Item     = f(f(c.value))

@main def main() =
    val ib = IntBox(21)
    val sb = StrBox("hello")
    println(ib.transform(n => s"number is $n"))
    println(ib.twice(_ * 2))
    println(sb.transform(_.toUpperCase))
    println(sb.twice(_ + "!"))
end main
```

The parameter `f: c.Item => String` is a dependent type: its signature  
depends on the path-dependent type `c.Item`, which the compiler resolves  
from the actual subtype of `c`. Only a function `Int => String` can be  
passed to an `IntBox`, and only `String => String` to a `StrBox`. This  
is more precise than a type parameter because the constraint is expressed  
through the structural type member rather than at the call site.  

## Chaining extension methods  

Extension methods chain naturally to build readable, reusable data-transformation pipelines.  

```scala
extension (s: String)
    def words: List[String]  = s.split("\\s+").toList.filter(_.nonEmpty)
    def wordCount: Int       = s.words.length
    def longestWord: String  = s.words.maxByOption(_.length).getOrElse("")
    def titleCase: String    = s.words.map(_.capitalize).mkString(" ")
    def reverseWords: String = s.words.reverse.mkString(" ")

@main def main() =
    val sentence = "the quick brown fox jumps over the lazy dog"
    println(s"Word count:   ${sentence.wordCount}")
    println(s"Longest word: ${sentence.longestWord}")
    println(s"Title case:   ${sentence.titleCase}")
    println(s"Reversed:     ${sentence.reverseWords}")
    val long = sentence.words.filter(_.length > 4)
    println(s"Long words:   $long")
end main
```

Each extension delegates to `s.words`, which splits the string once  
and returns a `List[String]`. Because all methods share the same base  
decomposition, the chain reads like a pipeline of transformations on  
a single concept. Real-world projects frequently combine several small,  
focused extensions this way to build a rich, cohesive API over a type  
they do not own.  

## Extension methods in DSL design  

Extension methods are the primary building block for fluent, internal domain-specific languages.  

```scala
case class Query(
    table:      String,
    conditions: List[String] = Nil,
    columns:    List[String] = Nil,
    limitRows:  Option[Int]  = None)

extension (q: Query)
    def select(cols: String*): Query =
        q.copy(columns = cols.toList)
    def where(cond: String): Query =
        q.copy(conditions = q.conditions :+ cond)
    def limitTo(n: Int): Query =
        q.copy(limitRows = Some(n))
    def toSQL: String =
        val cols  = if q.columns.isEmpty then "*"
                    else q.columns.mkString(", ")
        val conds = if q.conditions.isEmpty then ""
                    else " WHERE " + q.conditions.mkString(" AND ")
        val lim   = q.limitRows.fold("")(n => s" LIMIT $n")
        s"SELECT $cols FROM ${q.table}$conds$lim"

@main def main() =
    val sql = Query("users")
        .select("id", "name", "email")
        .where("age > 18")
        .where("active = true")
        .limitTo(25)
        .toSQL
    println(sql)
end main
```

The DSL chains method calls on an immutable `Query` value, producing a  
new `Query` at every step. This pattern is called a *fluent builder*  
and appears in many query, configuration, and testing libraries. Because  
each extension returns the same type, intermediate results can be stored,  
branched, or reused without losing type information.  

## Extension methods in API design  

A time-literals API uses extension methods on `Int` to create expressive, prose-like code.  

```scala
case class Duration(millis: Long):
    override def toString: String =
        val h = millis / 3_600_000
        val m = (millis % 3_600_000) / 60_000
        val s = (millis % 60_000) / 1000
        f"${h}h ${m}m ${s}s"

extension (n: Int)
    def seconds: Duration = Duration(n * 1_000L)
    def minutes: Duration = Duration(n * 60_000L)
    def hours: Duration   = Duration(n * 3_600_000L)
    def days: Duration    = Duration(n * 86_400_000L)

extension (d: Duration)
    def +(other: Duration): Duration = Duration(d.millis + other.millis)
    def *(factor: Int): Duration     = Duration(d.millis * factor)
    def toSeconds: Long              = d.millis / 1000L
    def toMinutes: Long              = d.millis / 60_000L

@main def main() =
    val timeout = 2.hours + 30.minutes + 15.seconds
    println(timeout)
    println(s"Seconds: ${timeout.toSeconds}")
    println(s"Minutes: ${timeout.toMinutes}")
    println(1.days * 7)
end main
```

Extensions on `Int` create a natural numeric-literal syntax: `2.hours`  
reads exactly like a prose specification. This pattern is used in  
well-known libraries such as Akka for timeouts and ScalaTest for  
durations. Because `Duration` is a plain case class, the extension  
methods carry no overhead beyond simple arithmetic, and the API feels  
native to the language.  

## Operator-style extension methods  

Extension methods can define symbolic operators, making domain types composable with infix syntax.  

```scala
case class Fraction(num: Int, den: Int):
    override def toString: String = s"$num/$den"

object Fraction:
    def apply(n: Int, d: Int): Fraction =
        val g = gcd(n.abs, d.abs)
        new Fraction(n / g, d / g)
    private def gcd(a: Int, b: Int): Int =
        if b == 0 then a else gcd(b, a % b)

extension (f: Fraction)
    def +(g: Fraction): Fraction =
        Fraction(f.num * g.den + g.num * f.den, f.den * g.den)
    def *(g: Fraction): Fraction =
        Fraction(f.num * g.num, f.den * g.den)
    def unary_- : Fraction   = Fraction(-f.num, f.den)
    def reciprocal: Fraction = Fraction(f.den, f.num)

@main def main() =
    val a = Fraction(1, 2)
    val b = Fraction(1, 3)
    println(a + b)
    println(a * b)
    println(-a)
    println(b.reciprocal)
    println(a + b + Fraction(1, 6))
end main
```

Symbolic operators such as `+`, `*`, and `unary_-` integrate with  
Scala's standard infix notation, so `a + b` desugars to `a.+(b)`. The  
`Fraction.apply` factory reduces fractions by dividing by the GCD,  
ensuring all results are in lowest terms. Defining arithmetic through  
extensions keeps the `case class` body minimal while providing a  
complete algebra. Any code with the extension in scope can write  
`a + b` without any explicit conversion or import.  
