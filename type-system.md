# Foundations of the Scala Type System

A type is a compile-time label that the Scala compiler attaches to every  
expression, value, and definition. It describes the set of values an  
expression may produce together with the operations those values support.  
When the compiler sees `val x: Int = 42`, the type `Int` tells it that  
`x` holds a 32-bit signed integer and that arithmetic operators such as  
`+`, `-`, and `*` are available on it. Types are verified strictly before  
any bytecode runs, so passing a `String` where an `Int` is expected  
triggers a precise compile-time error rather than a runtime crash.  

Scala operates on two distinct layers of type information. At compile  
time the type checker knows the complete declared type of every  
expression, including generic arguments, singleton types, union types,  
and intersection types. At runtime the JVM retains only the erased class  
of each object. A `List[Int]` and a `List[String]` are both represented  
as plain `scala.collection.immutable.List` at the bytecode level. This  
phenomenon, known as type erasure, limits certain runtime type tests.  
Scala 3 provides `TypeTest` and `ClassTag` to recover runtime type  
information when it is genuinely necessary.  

A type system is a formal proof system embedded in the language whose  
job is to reject ill-typed programs before they execute. Scala's type  
system prevents calling methods that do not exist on a value, passing  
arguments in the wrong order, returning incompatible values, and  
confusing semantically distinct concepts that share the same underlying  
representation. Beyond safety, the type system is the foundation for  
powerful abstractions: generics, type classes, and capability-based  
effect systems all rely on type-level machinery to describe  
relationships that would otherwise require fragile runtime encoding.  

Scala 3 uses a bidirectional type-checking algorithm to infer most  
types automatically. When you write `val xs = List(1, 2, 3)` the  
compiler determines that `xs` has type `List[Int]` by examining the  
element literals. Expected types flow inward from call sites: if a  
function expects a `String => Boolean`, the parameter type of an  
anonymous function supplied at that call site can be omitted entirely.  
Explicit type annotations remain important at public API boundaries,  
for recursive definitions, and wherever the inferred type is broader  
than intended.  

Scala's type hierarchy is rooted at `Any`, the supertype of every  
value. `Any` has two direct subtypes: `AnyVal`, which covers the eight  
JVM primitive types (`Int`, `Long`, `Double`, `Float`, `Char`,  
`Boolean`, `Byte`, `Short`) plus `Unit`; and `AnyRef`, an alias for  
`java.lang.Object` that is the parent of all reference types. `Nothing`  
sits at the very bottom and is a subtype of every other type in the  
hierarchy. `Null` is a subtype of every reference type and represents  
the explicit absence of an object.  

Value types extend `AnyVal` and map to JVM primitive types. They are  
represented without heap allocation in most contexts, cannot be `null`,  
and are compared by value rather than identity. Reference types extend  
`AnyRef`, are heap-allocated objects with identity pointers, and can  
in principle hold `null`. The Scala compiler automatically boxes value  
types into their Java wrapper counterparts (`java.lang.Integer`, and  
so on) when a reference is needed, and unboxes them back to primitives  
when the context demands it. User-defined value classes wrap a single  
field without heap allocation in most usage contexts.  

Scala 3 addresses several limitations of Scala 2. Opaque type aliases  
replace the `newtype` pattern with a zero-overhead built-in abstraction  
that hides the underlying representation from callers. Union types  
(`A | B`) and intersection types (`A & B`) replace cumbersome sealed  
hierarchies and implicit-based encodings. Polymorphic function types  
make generic lambdas first-class without extra boilerplate. Match types  
bring computation to the type level using familiar pattern-matching  
syntax. Context functions replace old implicit function types with a  
cleaner, more disciplined design. The improved type inference based on  
the DOT calculus delivers stronger guarantees and fewer surprising  
inference failures compared to Scala 2.  

---

## Type annotations

Explicit type annotations declare the expected type of a binding.  

```scala
@main def main() =
    val name: String    = "Alice"
    val age: Int        = 30
    val height: Double  = 1.75
    val active: Boolean = true
    val ratio: Float    = 0.5f

    println(s"$name, age $age, height $height m")
    println(s"Active: $active, ratio: $ratio")
end main
```

A type annotation appears after a colon that follows the identifier.  
The compiler checks that the right-hand expression conforms to the  
declared type and rejects the program with a clear diagnostic if not.  
Annotations are optional for local bindings where the type can be  
inferred, but they are required at public API boundaries and for  
recursive definitions. Aligning annotations visually, as shown here,  
is a common convention that improves readability.  

## Type inference

The Scala compiler infers the type of every binding from its  
right-hand side.  

```scala
@main def main() =
    val name    = "Alice"         // Inferred: String
    val count   = 42              // Inferred: Int
    val ratio   = 3.14            // Inferred: Double
    val flag    = true            // Inferred: Boolean
    val items   = List(1, 2, 3)   // Inferred: List[Int]
    val mapping = Map("a" -> 1)   // Inferred: Map[String, Int]

    println(name.getClass.getSimpleName)
    println(items.getClass.getSimpleName)
    println(mapping.getClass.getSimpleName)
end main
```

Type inference examines the right-hand side of each definition and  
assigns the most precise type that fits the expression. Numeric  
literals default to `Int` and `Double` unless a context type guides  
the compiler toward another choice. Collection factory methods such as  
`List(...)` and `Map(...)` infer their element and key-value types from  
the supplied arguments. The resulting inferred types are identical to  
what explicit annotations would declare.  

## Literal types

A literal type is a type that contains exactly one value.  

```scala
@main def main() =
    val zero: 0       = 0
    val yes: true     = true
    val greeting: "hi" = "hi"

    def describe(code: 1 | 2 | 3): String =
        code match
            case 1 => "one"
            case 2 => "two"
            case 3 => "three"

    println(describe(2))
    println(zero)
    println(yes)
    println(greeting)
end main
```

In Scala 3 every literal has a singleton literal type in addition to  
its widened type. The literal `0` has type `0`, which is a subtype of  
`Int`. Literal types are most useful in combination with union types  
(`1 | 2 | 3`) to enumerate a finite set of allowed values without  
defining a full enumeration. The compiler ensures at compile time that  
no value outside the declared set can be passed. Pattern matches on  
literal types are also exhaustively checked.  

## Widening conversions

Widening converts a narrower numeric type to a broader one  
automatically.  

```scala
@main def main() =
    val i: Int    = 42
    val l: Long   = i      // Int widens to Long
    val d: Double = i      // Int widens to Double
    val f: Float  = i      // Int widens to Float

    val s: Short  = 10
    val i2: Int   = s      // Short widens to Int

    println(s"Int $i -> Long $l")
    println(s"Int $i -> Double $d")
    println(s"Short $s -> Int $i2")
end main
```

Widening conversions preserve the exact numeric value and never lose  
information. The Scala compiler performs them automatically when a  
narrower numeric value is assigned to a variable of a broader numeric  
type. The numeric widening chain is: `Byte` → `Short` → `Int` →  
`Long` → `Float` → `Double`. No explicit cast syntax is required in  
any of these cases, and no precision is lost until `Int` widens to  
`Float` for values larger than 16 million.  

## Narrowing conversions

Narrowing converts a broader numeric type to a narrower one  
explicitly.  

```scala
@main def main() =
    val d: Double = 9.99
    val i: Int    = d.toInt    // Fractional part truncated: 9

    val l: Long   = 1234567890123L
    val s: Short  = l.toShort  // Overflow: value wraps around

    val big: Int  = 300
    val b: Byte   = big.toByte // Overflow: 300 wraps to 44

    println(s"Double $d -> Int $i")
    println(s"Long $l -> Short $s")
    println(s"Int $big -> Byte $b")
end main
```

Narrowing conversions require an explicit call such as `toInt`,  
`toShort`, or `toByte`. The compiler does not perform them silently  
because information may be lost. Converting `Double` to `Int` truncates  
the fractional part without rounding. Converting `Long` or `Int` to  
`Short` or `Byte` discards high bits, potentially producing a value  
with the opposite sign. Always validate the range before narrowing when  
data integrity matters.  

## Top types

`Any`, `AnyVal`, and `AnyRef` form the top of Scala's type hierarchy.  

```scala
@main def main() =
    val a: Any    = 42        // Any accepts value types
    val b: Any    = "hello"   // Any accepts reference types
    val c: AnyRef = "world"   // AnyRef is java.lang.Object
    val d: AnyVal = 3.14      // AnyVal wraps primitive types

    def printType(x: Any): Unit =
        println(s"${x.getClass.getName}: $x")

    printType(a)
    printType(b)
    printType(c)
    printType(d)
end main
```

`Any` is the root supertype of every Scala value. It defines a small  
number of universal methods including `equals`, `hashCode`, and  
`toString`. `AnyVal` is the common supertype of all eight primitive  
types plus `Unit`, and `AnyRef` is the common supertype of all  
heap-allocated objects. Because `Any` is the top type, a `List[Any]`  
can hold integers, strings, booleans, and custom objects side by side.  
Assigning a primitive to an `Any` reference causes the compiler to emit  
a boxing instruction that wraps the value in a heap object.  

## Bottom types

`Nothing` and `Null` sit at the very bottom of Scala's type hierarchy.  

```scala
@main def main() =
    def fail(msg: String): Nothing =
        throw RuntimeException(msg)

    // Nothing is a subtype of every type, so both branches type-check
    def parseInt(s: String): Int =
        if s.forall(_.isDigit) then s.toInt
        else fail(s"'$s' is not an integer")

    val maybeNull: String | Null = null
    val safe: String =
        if maybeNull != null then maybeNull else "default"

    println(parseInt("42"))
    println(safe)
end main
```

`Nothing` has no values: it is the return type of expressions that  
never complete normally, such as `throw` and infinite loops. Because  
`Nothing` is a subtype of every type, it can appear in either branch  
of a conditional without breaking type unification. The compiler knows  
that if a branch returns `Nothing` the overall expression still has a  
well-defined type. `Null` is the type of the `null` literal and is a  
subtype of every reference type. Scala 3's explicit nullability feature  
(`-Yexplicit-nulls`) removes `Null` from most reference type  
hierarchies to enforce null safety.  

## Unit type

`Unit` is the type of expressions evaluated purely for their side  
effects.  

```scala
@main def main() =
    def greet(name: String): Unit =
        println(s"Hello, $name!")

    val result: Unit = greet("Alice")
    println(s"greet returned: $result")

    val unitLiteral: Unit = ()
    println(s"Unit literal: $unitLiteral")

    val computed: Unit =
        val x = 10
        val y = 20
        println(x + y)   // side effect only; result discarded
end main
```

`Unit` corresponds to `void` in Java and C, but in Scala it is a  
proper type with exactly one value written `()`. Every expression in  
Scala has a type, so even a method that exists only for its side  
effects returns a typed value. The `Unit` value `()` carries no  
information, but its existence allows methods like `greet` to be used  
uniformly with other functions in higher-order contexts. When a block  
discards a non-`Unit` value the compiler may emit a warning.  

## Nothing for error signaling

`Nothing` as a return type signals that a function never returns  
normally.  

```scala
@main def main() =
    def required(value: Option[String], name: String): String =
        value.getOrElse(fail(s"Required field '$name' is missing"))

    def fail(msg: String): Nothing =
        throw IllegalArgumentException(msg)

    val config = Map("host" -> "localhost")

    val host = required(config.get("host"), "host")
    println(s"Host: $host")

    // required(config.get("port"), "port")  // would throw at runtime
end main
```

A function with return type `Nothing` communicates clearly to callers  
that the function will never produce a value: it always throws, loops  
forever, or terminates the program. Because `Nothing` is a subtype of  
every type, `fail(...)` can be used in any expression position without  
affecting the type of the surrounding context. This pattern is  
preferable to returning a placeholder value such as `null` or `-1`  
because the missing-case logic is explicit, and the compiler treats  
`fail` calls as dead-code branches.  

## Compile-time versus runtime types

Scala's compile-time types are richer than the classes available at  
runtime.  

```scala
@main def main() =
    val xs: List[Int] = List(1, 2, 3)

    // Compile-time: List[Int]
    // Runtime class: scala.collection.immutable.$colon$colon
    println(s"Runtime class : ${xs.getClass.getName}")

    // Type argument Int is erased at runtime
    xs match
        case _: List[?] => println("It is a List (type arg erased)")

    // ClassTag recovers the element class at runtime
    import scala.reflect.ClassTag
    def typed[A: ClassTag](xs: Seq[Any]): Seq[A] =
        xs.collect { case a: A => a }

    val mixed: Seq[Any] = Seq(1, "hello", 2, "world", 3)
    println(typed[Int](mixed))
    println(typed[String](mixed))
end main
```

Generic type arguments are erased at the bytecode level: `List[Int]`  
and `List[String]` share the same runtime class. This means that a  
pattern match on `List[Int]` only checks the outer `List` class, not  
the element type. The compiler emits an unchecked warning when a  
generic type test is attempted. `ClassTag` is an implicit evidence  
value that carries the erased `Class[A]` at runtime, enabling  
`collect` to filter by element type after erasure.  

## Type aliases

A type alias introduces a shorter or more descriptive name for an  
existing type.  

```scala
type UserId   = Int
type UserName = String
type UserMap  = Map[UserId, UserName]

@main def main() =
    val users: UserMap = Map(
        1 -> "Alice",
        2 -> "Bob",
        3 -> "Carol"
    )

    def findUser(id: UserId): Option[UserName] =
        users.get(id)

    println(findUser(1))
    println(findUser(99))

    type Transform[A] = A => A
    val double: Transform[Int] = _ * 2
    println(double(21))
end main
```

A type alias is fully transparent: `UserId` and `Int` are exactly the  
same type, and the compiler accepts one wherever the other is expected.  
Aliases improve readability by replacing long generic types with  
domain-specific names and by documenting the intended role of a type  
parameter. They can be parameterised, making them simple type  
constructors. Because they are transparent they provide no  
encapsulation: use opaque type aliases when information hiding is  
required.  

## Opaque types

An opaque type alias hides its underlying representation from outside  
its defining scope.  

```scala
object Newtype:
    opaque type EmailAddress = String

    object EmailAddress:
        def apply(s: String): EmailAddress =
            require(s.contains("@"), s"'$s' is not valid")
            s

    extension (e: EmailAddress)
        def value: String = e

@main def main() =
    import Newtype.*

    val email = EmailAddress("alice@example.com")
    println(email.value)

    // val raw: String = email  // Compile error outside Newtype
    // val bad = EmailAddress("notanemail")  // throws at runtime
end main
```

An opaque type alias is indistinguishable from its underlying type  
inside the defining object, but appears as a completely distinct type  
to all external code. This zero-overhead abstraction replaces  
`AnyVal`-based wrapper classes for the common newtype pattern.  
Extension methods defined inside the same object give callers a  
clean API without exposing the underlying `String`. Unlike a plain  
type alias, an opaque type cannot accidentally flow back to a raw  
`String` outside its scope, enforcing the domain invariant at compile  
time.  

---

## Function types

A function type describes a value that can be applied to arguments.  

```scala
@main def main() =
    val double: Int => Int          = x => x * 2
    val add: (Int, Int) => Int      = (x, y) => x + y
    val greet: String => String     = name => s"Hello, $name!"
    val always: () => Boolean       = () => true

    println(double(5))
    println(add(3, 4))
    println(greet("Alice"))
    println(always())

    val nums = List(1, 2, 3, 4, 5)
    println(nums.map(double))
    println(nums.filter(_ % 2 == 0))
end main
```

A function type `A => B` represents a value that accepts an argument  
of type `A` and returns a value of type `B`. Multi-argument function  
types use a tuple-like notation: `(A, B) => C`. Function values are  
first-class in Scala: they can be stored in variables, passed to  
methods, and returned from methods. Under the hood, function types  
compile to instances of `FunctionN` traits generated by the standard  
library, where `N` is the arity.  

## Method types

A method is defined with `def` and has a method type distinct from a  
function type.  

```scala
@main def main() =
    def square(n: Int): Int          = n * n
    def concat(a: String, b: String): String = a + b
    def repeat(s: String, n: Int): String    = s * n
    def greet(name: String): Unit    = println(s"Hello, $name!")

    println(square(7))
    println(concat("Scala", " 3"))
    println(repeat("ab", 3))
    greet("Bob")
end main
```

Methods defined with `def` are not values: they cannot be stored in a  
variable or passed directly to a higher-order function without  
eta-expansion. However, methods can be overloaded, can have type  
parameters, and can have multiple parameter lists, none of which is  
available on plain function values. The Scala compiler automatically  
converts a method reference to a function value (eta-expands it)  
whenever the context requires a `FunctionN` type.  

## Polymorphic function types

A polymorphic function type captures a generic function as a value.  

```scala
@main def main() =
    // [A] => A => A is a polymorphic function type (Scala 3)
    val identity: [A] => A => A =
        [A] => (x: A) => x

    val head: [A] => List[A] => Option[A] =
        [A] => (xs: List[A]) => xs.headOption

    println(identity("hello"))
    println(identity(42))
    println(identity(List(1, 2, 3)))

    println(head(List(10, 20)))
    println(head(List.empty[String]))
end main
```

Scala 2 could not express a polymorphic function as a first-class  
value: generic code required a method or a helper trait. Scala 3  
introduces the polymorphic function type `[A] => A => B`, which  
quantifies over a type parameter `A` just as a generic method does,  
but wraps the whole thing in a value. This makes it possible to store,  
pass, and return generic functions without losing the type parameter.  
The type annotation and the lambda body both carry the explicit `[A]`  
binder.  

## Dependent function types

A dependent function type has a return type that depends on the  
argument value.  

```scala
trait Box:
    type Content
    val value: Content

@main def main() =
    val extract: (b: Box) => b.Content =
        b => b.value

    val strBox = new Box:
        type Content = String
        val value    = "Scala 3"

    val intBox = new Box:
        type Content = Int
        val value    = 42

    val s: String = extract(strBox)
    val n: Int    = extract(intBox)

    println(s)
    println(n)
end main
```

A dependent function type `(x: T) => x.R` lets the return type vary  
with the identity of the argument, not just its class. This is the  
function equivalent of a dependent method type. The compiler tracks  
which specific `Box` was passed and uses its `Content` member to type  
the result precisely. Dependent function types are used in APIs that  
must preserve information across boundaries where a plain generic type  
would lose it.  

## Context function types

A context function type provides its argument as a `given` instance  
rather than an ordinary parameter.  

```scala
trait Config:
    def host: String
    def port: Int

type Configured[A] = Config ?=> A

@main def main() =
    def getUrl: Configured[String] =
        val cfg = summon[Config]
        s"http://${cfg.host}:${cfg.port}/api"

    def withConfig[A](cfg: Config)(body: Configured[A]): A =
        body(using cfg)

    given Config with
        def host = "localhost"
        def port = 8080

    println(getUrl)
    println(withConfig(new Config { def host = "prod.example.com"; def port = 443 })(getUrl))
end main
```

A context function type `A ?=> B` is a function that requires an  
implicit `given` instance of type `A` in scope when it is applied.  
This pattern is central to capability-passing style: instead of  
threading configuration or effects through every parameter list  
explicitly, they are provided as ambient context. The `Configured[A]`  
alias makes the dependency readable at call sites. Context functions  
are first-class values that can be stored and passed like any other  
function.  

## Eta-expansion

Eta-expansion converts a method reference into a function value.  

```scala
@main def main() =
    def double(x: Int): Int    = x * 2
    def addOne(x: Int): Int    = x + 1
    def square(x: Int): Int    = x * x

    // Explicit eta-expansion
    val fn1: Int => Int = double
    val fn2: Int => Int = addOne

    // Automatic eta-expansion at call site
    val nums = List(1, 2, 3, 4, 5)
    println(nums.map(double))    // automatic
    println(nums.map(fn1))       // explicit
    println(nums.map(square))    // automatic

    // Eta-expansion of a two-argument method
    def add(a: Int, b: Int): Int = a + b
    val addFn: (Int, Int) => Int = add
    println(addFn(3, 4))
end main
```

When the compiler expects a function value but encounters a method  
name, it automatically wraps the method in a lambda that forwards its  
arguments. This transformation is called eta-expansion, borrowing the  
term from lambda calculus. In Scala 3 eta-expansion is applied  
automatically whenever the expected type is a function type, removing  
the need for the trailing `_` placeholder that Scala 2 required.  
Methods with multiple parameter lists are eta-expanded to a curried  
function.  

## Functions as values

Functions are first-class values that can be stored, passed, and  
composed.  

```scala
@main def main() =
    val ops: List[Int => Int] = List(
        _ + 1,
        _ * 2,
        x => x * x
    )

    val result = ops.foldLeft(3)((acc, f) => f(acc))
    println(s"After pipeline on 3: $result")  // (3+1)*2=8, 8^2=64

    val composed: Int => Int =
        ops.reduce((f, g) => f andThen g)
    println(s"Composed on 5: ${composed(5)}")

    def applyTwice(f: Int => Int, x: Int): Int = f(f(x))
    println(s"applyTwice(+1, 10) = ${applyTwice(_ + 1, 10)}")
end main
```

Treating functions as values enables higher-order programming: storing  
pipelines of transformations in a list, building composed operations  
at runtime, and passing behaviour as a parameter. The `andThen` method  
on `Function1` chains two functions so that the output of the first  
feeds into the input of the second. `compose` does the same in reverse  
order. These combinators make it straightforward to build processing  
pipelines without naming every intermediate step.  

## Lambdas versus named functions

Lambdas are anonymous function literals; named definitions use `def`.  

```scala
@main def main() =
    // Anonymous lambda
    val square = (x: Int) => x * x

    // Named function
    def cube(x: Int): Int = x * x * x

    // Both work identically in higher-order position
    val nums = List(1, 2, 3, 4)
    println(nums.map(square))
    println(nums.map(cube))
    println(nums.map(x => x + 10))

    // Named functions support recursion; lambdas do not directly
    def factorial(n: Int): Int =
        if n <= 1 then 1 else n * factorial(n - 1)

    println(factorial(6))
end main
```

A lambda is an expression that produces a function value inline.  
It cannot refer to itself by name, so direct recursion requires either  
a `def` or a fixed-point combinator. Named `def` definitions support  
recursion, can be overloaded, can carry type parameters, and do not  
produce a heap object unless eta-expanded. Lambdas are cleaner for  
short, non-recursive transformations passed at call sites. Choose  
`def` when the logic is complex, recursive, or reused across multiple  
call sites.  

## Function types and tuples

Functions can accept tuples as arguments or return tuples as results.  

```scala
@main def main() =
    val sumPair: ((Int, Int)) => Int    = (a, b) => a + b
    val swapPair: ((Int, String)) => (String, Int) = (n, s) => (s, n)
    val split: String => (String, Int)  = s => (s, s.length)

    println(sumPair((3, 7)))
    println(swapPair((42, "hello")))
    println(split("Scala"))

    val pairs = List((1, 2), (3, 4), (5, 6))
    println(pairs.map(sumPair))

    // tupled converts a two-argument function to a tuple function
    val addFn: (Int, Int) => Int = _ + _
    val tupleFn = addFn.tupled
    println(pairs.map(tupleFn))
end main
```

In Scala a tuple `(A, B)` is a value of type `Tuple2[A, B]`. A  
function type `((A, B)) => C` takes a single `Tuple2` argument,  
whereas `(A, B) => C` takes two separate arguments. The compiler can  
convert between the two forms using the `.tupled` method, which wraps  
the multi-argument version into one that accepts a tuple. This is  
useful when passing a two-argument function to `map` over a list of  
pairs.  

## Function types and givens

Given instances can influence how function types are resolved and  
applied.  

```scala
trait Formatter[A]:
    def format(a: A): String

@main def main() =
    given Formatter[Int] with
        def format(n: Int): String = s"[Int: $n]"

    given Formatter[String] with
        def format(s: String): String = s"[Str: $s]"

    def display[A](xs: List[A])(using fmt: Formatter[A]): Unit =
        xs.foreach(x => println(fmt.format(x)))

    display(List(1, 2, 3))
    display(List("alpha", "beta", "gamma"))

    // Pass the formatter explicitly as a function value
    def formatAll[A](xs: List[A], fmt: Formatter[A]): List[String] =
        xs.map(fmt.format)

    println(formatAll(List(10, 20), summon[Formatter[Int]]))
end main
```

`given` instances are provided implicitly by the compiler when a  
parameter is marked `using`. This mechanism, known as the type-class  
pattern, allows a single generic function to behave differently for  
different types without requiring explicit dispatch. The `summon`  
function retrieves the `given` instance in scope for a specified type.  
Type classes decouple the algorithm from the data representation and  
make it easy to add new behaviour for existing types retroactively.  

## Curried function types

Currying splits a multi-argument function into a chain of  
single-argument functions.  

```scala
@main def main() =
    val add: Int => Int => Int      = x => y => x + y
    val multiply: Int => Int => Int = x => y => x * y

    val addFive  = add(5)       // partially applied
    val triple   = multiply(3)  // partially applied

    println(addFive(10))   // 15
    println(triple(7))     // 21

    val nums = List(1, 2, 3, 4, 5)
    println(nums.map(addFive))
    println(nums.map(triple))

    // def with multiple parameter lists also curries
    def power(base: Int)(exp: Int): Int =
        math.pow(base, exp).toInt

    val square = power(2)
    println(nums.map(square))
end main
```

A curried function `A => B => C` takes one argument at a time and  
returns a new function after each application. Partial application  
fixes the first argument and produces a specialised function ready for  
the second. This is useful for creating families of related functions  
from a single definition. In Scala, `def` methods with multiple  
parameter lists are the idiomatic way to curry: the compiler  
eta-expands them naturally when a partially applied form is needed.  

## By-name parameters

A by-name parameter delays evaluation of its argument until it is  
used.  

```scala
@main def main() =
    def lazyLog(enabled: Boolean, msg: => String): Unit =
        if enabled then println(msg)

    def expensiveMsg(): String =
        println("  (computing message...)")
        "expensive result"

    println("--- logging on ---")
    lazyLog(true, expensiveMsg())    // message IS computed

    println("--- logging off ---")
    lazyLog(false, expensiveMsg())   // message is NOT computed

    def retry(times: Int)(block: => Unit): Unit =
        var i = 0
        while i < times do
            block
            i += 1

    retry(3)(println("attempt"))
end main
```

A by-name parameter `=> T` wraps its argument in a thunk: the  
expression is re-evaluated each time the parameter is referenced  
inside the function body. This differs from a lazy val, which  
evaluates once and caches the result. By-name parameters are used to  
implement control-flow abstractions such as `retry`, `unless`, and  
`withLock` that look like language keywords at call sites. They are  
also key for making expensive computations conditional on a flag,  
avoiding unnecessary work when logging is disabled.  

---

## Generic methods

A generic method abstracts over one or more type parameters.  

```scala
@main def main() =
    def identity[A](x: A): A = x

    def swap[A, B](pair: (A, B)): (B, A) =
        (pair._2, pair._1)

    def firstOf[A](xs: List[A]): Option[A] =
        xs.headOption

    def pairWith[A, B](a: A, b: B): (A, B) =
        (a, b)

    println(identity(42))
    println(identity("hello"))
    println(swap((1, "one")))
    println(firstOf(List(10, 20, 30)))
    println(firstOf(List.empty[String]))
    println(pairWith("age", 30))
end main
```

A generic method introduces a type parameter in square brackets before  
the parameter list. The type parameter acts as a placeholder that the  
compiler fills in at each call site based on the supplied arguments.  
Generic methods avoid code duplication: `identity` works for any type  
`A` with a single implementation. The compiler verifies that the body  
uses `A` only in ways that are valid for any possible type, ensuring  
the method is truly polymorphic.  

## Generic classes

A generic class parameterises over one or more types, deferring the  
concrete choice to instantiation.  

```scala
class Pair[A, B](val first: A, val second: B):
    def swap: Pair[B, A] =
        Pair(second, first)
    def map[C](f: A => C): Pair[C, B] =
        Pair(f(first), second)
    override def toString: String =
        s"Pair($first, $second)"

@main def main() =
    val p1 = Pair(1, "one")
    val p2 = Pair("hello", true)

    println(p1)
    println(p1.swap)
    println(p1.map(_ * 10))
    println(p2)
end main
```

Generic classes carry type parameters that flow through all their  
methods. The `Pair[A, B]` class stores two values of potentially  
different types and exposes typed accessors without losing type  
information. The `swap` method returns `Pair[B, A]`, precisely  
reflecting the reversal. The `map` method introduces a third type  
parameter `C` local to that method, showing that methods on generic  
classes can introduce their own type parameters independently of the  
class-level ones.  

## Generic traits

A generic trait defines an interface that is polymorphic in one or  
more types.  

```scala
trait Container[A]:
    def value: A
    def map[B](f: A => B): Container[B]

case class Box[A](value: A) extends Container[A]:
    def map[B](f: A => B): Box[B] = Box(f(value))
    override def toString: String = s"Box($value)"

@main def main() =
    val intBox: Box[Int]    = Box(42)
    val strBox: Box[String] = intBox.map(_.toString)
    val doubled: Box[Int]   = intBox.map(_ * 2)

    println(intBox)
    println(strBox)
    println(doubled)

    val c: Container[Int] = Box(100)
    println(c.map(_ + 1))
end main
```

Generic traits express abstract interfaces that are independent of a  
specific type. `Container[A]` defines the contract for anything that  
holds a single value of type `A` and can transform it. Concrete  
implementations such as `Box` fill in the type parameter. Generic  
traits are the foundation of the type-class pattern, the collection  
hierarchy, and capability abstractions throughout the Scala ecosystem.  
They can be mixed into multiple classes, enabling horizontal  
composition of polymorphic behaviour.  

## Upper-bounded type parameters

An upper bound restricts a type parameter to a subtype of a given  
type.  

```scala
trait Named:
    def name: String

case class Person(name: String, age: Int) extends Named
case class Product(name: String, price: Double) extends Named

def greetAll[A <: Named](items: List[A]): Unit =
    items.foreach(item => println(s"Hello, ${item.name}!"))

def maxByName[A <: Named](a: A, b: A): A =
    if a.name >= b.name then a else b

@main def main() =
    val people   = List(Person("Alice", 30), Person("Bob", 25))
    val products = List(Product("Widget", 9.99), Product("Gadget", 24.50))

    greetAll(people)
    greetAll(products)
    println(maxByName(Person("Alice", 30), Person("Zara", 22)))
end main
```

The constraint `A <: Named` tells the compiler that `A` can be any  
type that is a subtype of `Named`. Inside the method body, the  
compiler knows that every `A` has a `name` method because `Named`  
guarantees it. Upper bounds are the most common form of constraint.  
They are used whenever a generic algorithm needs to call methods  
defined on a shared supertype. The bound is checked at each call site:  
passing a type that does not extend `Named` is a compile-time error.  

## Lower-bounded type parameters

A lower bound restricts a type parameter to a supertype of a given  
type.  

```scala
class ImmutableStack[+A](private val elems: List[A]):
    def push[B >: A](elem: B): ImmutableStack[B] =
        ImmutableStack(elem :: elems)
    def top: Option[A] = elems.headOption
    override def toString: String = s"Stack($elems)"

@main def main() =
    class Animal(val name: String):
        override def toString = s"Animal($name)"
    class Dog(name: String) extends Animal(name):
        override def toString = s"Dog($name)"
    class Cat(name: String) extends Animal(name):
        override def toString = s"Cat($name)"

    val dogs: ImmutableStack[Dog] = ImmutableStack(List(Dog("Rex")))
    val wider: ImmutableStack[Animal] = dogs.push(Cat("Whiskers"))

    println(dogs)
    println(wider)
    println(wider.top)
end main
```

A lower bound `B >: A` says that `B` must be `A` or any of its  
supertypes. Lower bounds appear together with covariance to enable  
functional update operations. In the stack example, pushing a `Cat`  
onto a `Stack[Dog]` is safe because the result is widened to  
`Stack[Animal]`, the nearest common supertype. Without the lower  
bound the covariant annotation `+A` would prevent `B` from appearing  
in a contravariant position (the method argument).  

## Combined type bounds

Type parameters can carry both an upper bound and a lower bound  
simultaneously.  

```scala
trait Printable:
    def print(): Unit

trait Loggable:
    def log(): String

def processItem[A <: Printable & Loggable](item: A): Unit =
    item.print()
    println(s"Log entry: ${item.log()}")

class Report(val title: String) extends Printable with Loggable:
    def print(): Unit  = println(s"Report: $title")
    def log(): String  = s"Processed '$title'"

class Invoice(val number: Int) extends Printable with Loggable:
    def print(): Unit  = println(s"Invoice #$number")
    def log(): String  = s"Invoice $number logged"

@main def main() =
    processItem(Report("Q4 Results"))
    processItem(Invoice(1042))
end main
```

The intersection bound `A <: Printable & Loggable` requires `A` to  
implement both `Printable` and `Loggable`. Inside the method the  
compiler knows that every `A` supports `.print()` and `.log()`.  
Intersection types in bounds are more precise than using a separate  
trait that extends both, because they do not require callers to create  
a combined trait in advance. Any class that independently mixes in  
both traits satisfies the bound automatically.  

## Multiple type parameters

A generic definition can introduce several independent type parameters.  

```scala
def zipWith[A, B, C](
    as: List[A],
    bs: List[B]
)(f: (A, B) => C): List[C] =
    as.zip(bs).map(f.tupled)

def partition[A, B](
    xs: List[Either[A, B]]
): (List[A], List[B]) =
    xs.foldRight((List.empty[A], List.empty[B])):
        case (Left(a),  (as, bs)) => (a :: as, bs)
        case (Right(b), (as, bs)) => (as, b :: bs)

@main def main() =
    val names  = List("Alice", "Bob", "Carol")
    val scores = List(95, 87, 92)

    println(zipWith(names, scores)((n, s) => s"$n: $s"))

    val mixed: List[Either[String, Int]] =
        List(Right(1), Left("err"), Right(2), Left("bad"))

    println(partition(mixed))
end main
```

Multiple type parameters allow generic definitions to express  
relationships between several types simultaneously. `zipWith`  
transforms two lists of potentially different element types into a  
third list whose element type is determined by the combining function.  
The `partition` function separates a `List[Either[A, B]]` into its  
left and right components, maintaining the separate types of errors  
and values without merging them.  

## Higher-kinded types

A higher-kinded type is a type that itself takes a type parameter.  

```scala
trait Functor[F[_]]:
    def map[A, B](fa: F[A])(f: A => B): F[B]

given Functor[List] with
    def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)

given Functor[Option] with
    def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)

def doubleAll[F[_]: Functor](fa: F[Int]): F[Int] =
    summon[Functor[F]].map(fa)(_ * 2)

@main def main() =
    println(doubleAll(List(1, 2, 3)))
    println(doubleAll(Some(21)))
    println(doubleAll(Option.empty[Int]))
end main
```

A type constructor `F[_]` takes one type and produces another.  
`List`, `Option`, and `Either[String, *]` are all type constructors.  
A higher-kinded type parameter `F[_]` in a trait or method means  
"some type constructor". The `Functor` trait abstracts over any  
container that supports structure-preserving transformation via `map`.  
Higher-kinded types are the foundation of the functional programming  
type-class hierarchy (`Functor`, `Monad`, `Applicative`) available in  
libraries such as Cats and ZIO.  

## Type constructors

A type constructor is a type-level function that maps one type to  
another.  

```scala
type Pair[A]       = (A, A)
type Validated[A]  = Either[List[String], A]
type Reader[R, A]  = R => A

@main def main() =
    val intPair:    Pair[Int]       = (3, 7)
    val strPair:    Pair[String]    = ("hello", "world")
    val good:       Validated[Int]  = Right(42)
    val bad:        Validated[Int]  = Left(List("too small", "not prime"))
    val getLength:  Reader[String, Int] = _.length

    println(intPair)
    println(strPair)
    println(good)
    println(bad)
    println(getLength("Scala"))
end main
```

A type constructor resembles a function, but at the type level: it  
accepts a type argument and returns a new type. `Pair[Int]` is  
`(Int, Int)`, and `Pair[String]` is `(String, String)`. Type aliases  
with parameters are the simplest way to define type constructors in  
Scala. More complex type constructors can be expressed with abstract  
type members or higher-kinded generics. Type constructors are the  
building blocks of parametric data types and algebraic data types.  

## Partially applied type constructors

A partially applied type constructor fixes some of the type arguments  
of a multi-parameter type.  

```scala
type StringMap[V]   = Map[String, V]
type IntEither[A]   = Either[Int, A]
type Endo[A]        = A => A

@main def main() =
    val scores:  StringMap[Int]    = Map("Alice" -> 95, "Bob" -> 87)
    val names:   StringMap[String] = Map("1" -> "Alice", "2" -> "Bob")
    val err:     IntEither[String] = Left(404)
    val ok:      IntEither[String] = Right("found")
    val double:  Endo[Int]         = _ * 2
    val upper:   Endo[String]      = _.toUpperCase

    println(scores)
    println(names)
    println(err)
    println(ok)
    println(double(21))
    println(upper("scala"))
end main
```

`Map[String, V]` fixes the key type to `String` and leaves the value  
type open. The resulting `StringMap[V]` is itself a type constructor  
with a single parameter. Partially applied type constructors are  
essential when a type-class instance is needed for a multi-parameter  
type: to write a `Functor[StringMap]` instance one must first  
partially apply `Map`. In Scala 3 this is done with a type alias or,  
for inline use, a type lambda.  

## Type lambdas

A type lambda is an anonymous type constructor defined inline.  

```scala
trait Functor[F[_]]:
    def map[A, B](fa: F[A])(f: A => B): F[B]

// [V] =>> Map[String, V] is a type lambda
given Functor[[V] =>> Map[String, V]] with
    def map[A, B](fa: Map[String, A])(f: A => B): Map[String, B] =
        fa.map((k, v) => k -> f(v))

@main def main() =
    val m = Map("a" -> 1, "b" -> 2, "c" -> 3)

    val doubled = summon[Functor[[V] =>> Map[String, V]]]
        .map(m)(_ * 10)

    println(doubled)

    // Type lambda used directly in a type position
    type StringPair = [A] =>> (String, A)
    val p: StringPair[Int] = ("answer", 42)
    println(p)
end main
```

A type lambda `[A] =>> F[A, B]` partially applies a type constructor  
at the type level, similar to how a value lambda partially applies a  
function. The syntax `[V] =>> Map[String, V]` reads as "given a type  
`V`, produce `Map[String, V]`". Type lambdas replace the  
`({type λ[V] = Map[String, V]})#λ` encoding from Scala 2, which  
required creating an anonymous type and projecting out its member.  
They are concise and composable.  

## Polymorphic recursion

Polymorphic recursion calls a function with a different type argument  
than the one received.  

```scala
sealed trait Nat
case object Zero          extends Nat
case class  Succ(n: Nat)  extends Nat

def toInt(n: Nat): Int =
    n match
        case Zero    => 0
        case Succ(k) => 1 + toInt(k)

// Nested-list example: each Nested level wraps elements in List
sealed trait NList[A]
case class  NLeaf[A](value: A)             extends NList[A]
case class  NNested[A](inner: NList[List[A]]) extends NList[A]

def depth[A](n: NList[A]): Int =
    n match
        case NLeaf(_)      => 0
        case NNested(inner) => 1 + depth(inner)  // type changes to NList[List[A]]

@main def main() =
    val two = Succ(Succ(Zero))
    println(toInt(two))

    val d0: NList[Int]  = NLeaf(42)
    val d1: NList[Int]  = NNested(NLeaf(List(1, 2, 3)))
    val d2: NList[Int]  = NNested(NNested(NLeaf(List(List(1)))))
    println(depth(d0))
    println(depth(d1))
    println(depth(d2))
end main
```

In ordinary recursion the recursive call has the same type as the  
outer call. In polymorphic recursion the type argument changes at  
each level: `depth[A]` calls itself as `depth[List[A]]`. This allows  
modeling data structures whose nesting depth encodes type information  
at compile time. The compiler requires an explicit return type  
annotation on polymorphically recursive functions because type  
inference cannot determine the type from the recursive call alone.  

---

## Upper bounds

An upper-bound constraint `A <: T` limits `A` to subtypes of `T`.  

```scala
trait Shape:
    def area: Double
    def name: String

case class Circle(radius: Double) extends Shape:
    def area: Double = math.Pi * radius * radius
    def name: String = "Circle"

case class Rectangle(w: Double, h: Double) extends Shape:
    def area: Double = w * h
    def name: String = "Rectangle"

def totalArea[S <: Shape](shapes: List[S]): Double =
    shapes.map(_.area).sum

def largest[S <: Shape](shapes: List[S]): S =
    shapes.maxBy(_.area)

@main def main() =
    val shapes = List(Circle(3.0), Rectangle(4.0, 5.0), Circle(1.5))
    println(s"Total area : ${totalArea(shapes)}")
    println(s"Largest    : ${largest(shapes).name}")
end main
```

The bound `S <: Shape` guarantees that every element of the list  
exposes `area` and `name`. The methods `totalArea` and `largest`  
therefore do not need to know the specific concrete class of `S`; they  
only need the interface `Shape` provides. Upper bounds are the  
standard mechanism for expressing that a generic algorithm requires  
a minimum interface. The bound is enforced at every call site and  
prevents passing types that do not satisfy the constraint.  

## Lower bounds

A lower-bound constraint `B >: A` limits `B` to supertypes of `A`.  

```scala
class Node[+A](val value: A, val next: Option[Node[A]] = None):
    def prepend[B >: A](elem: B): Node[B] =
        Node(elem, Some(this))
    override def toString: String =
        next.fold(s"[$value]")(n => s"[$value] -> $n")

@main def main() =
    class Animal(val name: String):
        override def toString = s"Animal($name)"
    class Dog(name: String) extends Animal(name):
        override def toString = s"Dog($name)"
    class Cat(name: String) extends Animal(name):
        override def toString = s"Cat($name)"

    val dogNode: Node[Dog] = Node(Dog("Rex"))
    val mixed: Node[Animal] = dogNode.prepend(Cat("Whiskers"))

    println(dogNode)
    println(mixed)
end main
```

A lower bound enables a covariant class to accept elements from a  
wider type in update operations. Without `B >: A`, the compiler would  
reject `prepend` because parameter position is contravariant, clashing  
with the covariant annotation `+A`. The lower bound resolves this  
conflict: `B` must be `A` or wider, so the returned `Node[B]` is  
always a valid supertype of the original `Node[A]`. This technique  
is used throughout the immutable collections in the standard library.  

## Recursive bounds

A recursive bound constrains a type parameter in terms of itself.  

```scala
trait Comparable2[A <: Comparable2[A]]:
    def lessThan(other: A): Boolean
    def greaterThan(other: A): Boolean = other.lessThan(this.asInstanceOf[A])

class Temperature(val celsius: Double) extends Comparable2[Temperature]:
    def lessThan(other: Temperature): Boolean = celsius < other.celsius
    override def toString: String = s"${celsius}°C"

def minimum[A <: Comparable2[A]](a: A, b: A): A =
    if a.lessThan(b) then a else b

@main def main() =
    val t1 = Temperature(20.0)
    val t2 = Temperature(15.5)
    val t3 = Temperature(22.0)

    println(minimum(t1, t2))
    println(minimum(t2, t3))
end main
```

A recursive bound `A <: T[A]` requires that `A` itself appears as a  
type argument to the bound type. The bound encodes the constraint  
that comparison operates on values of the same type, not on some  
unrelated supertype. This prevents accidentally comparing a  
`Temperature` with a `Weight` through a shared abstract comparison  
method. Recursive bounds are a direct precursor to F-bounded  
polymorphism.  

## F-bounded polymorphism

F-bounded polymorphism uses the self-referential pattern to enable  
fluent builder chains that return the concrete type.  

```scala
trait Builder[Self <: Builder[Self]]:
    def withName(name: String): Self
    def withTag(tag: String): Self
    def build(): String

class PersonBuilder(
    private val name: String = "",
    private val tag:  String = ""
) extends Builder[PersonBuilder]:
    def withName(n: String): PersonBuilder = PersonBuilder(n, tag)
    def withTag(t: String):  PersonBuilder = PersonBuilder(name, t)
    def build(): String = s"Person(name=$name, tag=$tag)"

@main def main() =
    val p = PersonBuilder()
        .withName("Alice")
        .withTag("admin")
        .build()

    println(p)
end main
```

Without F-bounded polymorphism, a `withName` method defined on a base  
`Builder` trait would return `Builder`, losing the concrete type and  
breaking method chaining on the subclass. By writing `Self <: Builder[Self]`  
the trait declares that `Self` is the concrete subclass, so every  
method returns the specific subtype. This pattern is common in builder  
APIs, persistent data structures with update methods, and DSLs that  
need to preserve the type through a chain of operations.  

## Self types

A self-type annotation declares that a trait requires another type to  
be mixed in.  

```scala
trait Logger:
    def log(msg: String): Unit = println(s"[LOG] $msg")

trait Metrics:
    def record(key: String, value: Double): Unit =
        println(s"[METRIC] $key = $value")

trait DatabaseService:
    this: Logger & Metrics =>   // requires both Logger and Metrics

    def query(sql: String): String =
        log(s"Executing: $sql")
        record("queries", 1.0)
        s"result of '$sql'"

class AppDatabase extends DatabaseService with Logger with Metrics

@main def main() =
    val db = AppDatabase()
    println(db.query("SELECT * FROM users"))
end main
```

A self-type annotation `this: T =>` says that any class mixing in  
this trait must also provide `T`. Unlike inheritance, self types do  
not add `T` to the trait's own type; they only assert that the  
mixin requirement is met at instantiation. This separates concerns  
cleanly: `DatabaseService` uses logging and metrics but does not  
directly extend them. The compiler enforces the constraint at the  
point where a concrete class mixes in `DatabaseService` without  
satisfying the self-type.  

## API design with bounds

Careful use of type bounds leads to APIs that are both flexible and  
safe.  

```scala
trait Entity[ID]:
    def id: ID

trait Repository[ID, E <: Entity[ID]]:
    def findById(id: ID): Option[E]
    def save(entity: E): Unit
    def findAll(): List[E]

case class User(id: Int, name: String) extends Entity[Int]

class InMemoryRepo[ID, E <: Entity[ID]] extends Repository[ID, E]:
    private var store = Map.empty[ID, E]
    def findById(id: ID): Option[E] = store.get(id)
    def save(entity: E): Unit       = store = store + (entity.id -> entity)
    def findAll(): List[E]          = store.values.toList

@main def main() =
    val repo = InMemoryRepo[Int, User]()
    repo.save(User(1, "Alice"))
    repo.save(User(2, "Bob"))
    println(repo.findById(1))
    println(repo.findAll())
end main
```

The bound `E <: Entity[ID]` ensures that every entity managed by the  
repository exposes an `id` of the correct type. This lets `save`  
extract the key without requiring a separate key parameter, reducing  
the surface area for caller mistakes. Pairing the bound with the  
abstract `Repository` trait makes it straightforward to swap  
implementations (in-memory, database, cache) without changing calling  
code. Bounds at the class level are preferable to runtime `isInstanceOf`  
checks because errors are caught at compile time.  

## Abstract type members

Abstract type members let a trait defer the choice of a type to  
concrete implementations.  

```scala
trait Storage:
    type Key
    type Value
    def get(key: Key): Option[Value]
    def put(key: Key, value: Value): Unit
    def keys(): Set[Key]

class StringStorage extends Storage:
    type Key   = String
    type Value = String
    private var data = Map.empty[String, String]
    def get(key: String): Option[String]  = data.get(key)
    def put(key: String, value: String): Unit =
        data = data + (key -> value)
    def keys(): Set[String] = data.keySet

@main def main() =
    val store = StringStorage()
    store.put("lang", "Scala")
    store.put("ver",  "3")
    println(store.get("lang"))
    println(store.keys())
end main
```

Abstract type members are an alternative to type parameters for  
expressing type-level variation in a trait. Unlike type parameters,  
which must be supplied at each use of the type, abstract type members  
are hidden inside the trait and refined by each implementation. This  
improves readability when multiple related types are needed: a trait  
with three type members is cleaner than a class with three type  
parameters. Abstract type members also work naturally with path-dependent  
types, where the identity of an object determines a type.  

## Context bounds

A context bound `A: TC` requires an implicit `given` instance of  
`TC[A]` to be in scope.  

```scala
@main def main() =
    def maxOf[A: Ordering](xs: List[A]): A =
        xs.reduce(Ordering[A].max)

    def sortedDesc[A: Ordering](xs: List[A]): List[A] =
        xs.sorted(using Ordering[A].reverse)

    def show[A: Ordering](xs: List[A]): Unit =
        println(s"max=${maxOf(xs)}  desc=${sortedDesc(xs)}")

    show(List(3, 1, 4, 1, 5, 9, 2, 6))
    show(List("banana", "apple", "cherry", "date"))

    case class Priority(level: Int)
    given Ordering[Priority] = Ordering.by(_.level)
    show(List(Priority(3), Priority(1), Priority(4)))
end main
```

A context bound is syntactic sugar for a `using` parameter. Writing  
`[A: Ordering]` is equivalent to `[A](using Ordering[A])`. Context  
bounds communicate clearly that the algorithm depends on a type-class  
instance rather than a subtyping relationship. The `Ordering[A]`  
companion object's `apply` method retrieves the `given` instance in  
scope, or `summon[Ordering[A]]` can be used for the same purpose.  
Custom `given` instances make the mechanism extensible to user-defined  
types.  

## Type aliases and bounds

Type aliases can be combined with bounded type parameters to create  
expressive domain vocabularies.  

```scala
type Predicate[A]    = A => Boolean
type Transform[A]    = A => A
type BinaryOp[A]     = (A, A) => A
type Relation[A, B]  = (A, B) => Boolean

@main def main() =
    val isEven:    Predicate[Int]   = _ % 2 == 0
    val double:    Transform[Int]   = _ * 2
    val add:       BinaryOp[Int]    = _ + _
    val startsWith: Relation[String, String] = _.startsWith(_)

    val nums = List(1, 2, 3, 4, 5, 6)
    println(nums.filter(isEven))
    println(nums.map(double))
    println(nums.reduce(add))
    println(startsWith("Scala", "Sc"))

    def applyIf[A](x: A, p: Predicate[A], f: Transform[A]): A =
        if p(x) then f(x) else x

    println(nums.map(n => applyIf(n, isEven, double)))
end main
```

Parameterised type aliases create vocabulary types that communicate  
intent. `Predicate[A]` is more expressive than `A => Boolean` at a  
glance. Because Scala type aliases are transparent they add no runtime  
overhead and interoperate freely with the function types they  
abbreviate. Combining aliases with generic methods lets a codebase  
develop a shared language that keeps signatures readable without  
sacrificing generality.  

## Structural types

A structural type matches any class that exposes certain methods,  
regardless of its position in the class hierarchy.  

```scala
import scala.reflect.Selectable.reflectiveSelectable

type HasName  = { def name: String }
type HasClose = { def close(): Unit }

def greet(x: HasName): String =
    s"Hello, ${x.name}!"

def withClose[A <: HasClose, B](resource: A)(body: A => B): B =
    try body(resource)
    finally resource.close()

@main def main() =
    class Person(val name: String)
    class Connection(val name: String):
        def close(): Unit = println(s"Closing $name")

    println(greet(Person("Alice")))
    println(greet(Connection("db-conn")))

    withClose(Connection("file-handle")): c =>
        println(s"Using ${c.name}")
end main
```

A structural type describes the shape of an interface using a  
record-like syntax without naming a concrete class or trait. Any  
object that satisfies the structural requirements is accepted,  
enabling duck-typed APIs. In Scala 3 structural types require importing  
`reflectiveSelectable` because member access is implemented via  
reflection, which carries a runtime cost. Use structural types  
sparingly, preferring named traits for performance-sensitive code.  

## Multiple bounds

A single type parameter can satisfy several constraints simultaneously.  

```scala
import scala.math.Ordered

trait Printable:
    def display(): Unit

def processAll[A <: Ordered[A] & Printable](items: List[A]): Unit =
    val sorted = items.sorted
    sorted.foreach(_.display())

class Score(val value: Int) extends Ordered[Score] with Printable:
    def compare(other: Score): Int = value.compare(other.value)
    def display(): Unit = println(s"Score: $value")

class Rating(val stars: Double) extends Ordered[Rating] with Printable:
    def compare(other: Rating): Int = stars.compare(other.stars)
    def display(): Unit = println(f"Rating: $stars%.1f★")

@main def main() =
    processAll(List(Score(75), Score(95), Score(80)))
    processAll(List(Rating(4.5), Rating(3.0), Rating(4.9)))
end main
```

The intersection bound `A <: Ordered[A] & Printable` requires `A` to  
satisfy both constraints at once. The compiler verifies this at each  
call site and rejects types that do not implement both traits. Multiple  
bounds are the idiomatic alternative to creating an aggregate trait  
just to combine two existing ones. They keep the constraints explicit  
in the signature, making it clear exactly what the method requires  
from its type parameter without introducing intermediate abstractions.  

## Existential types

Existential types express "there exists some type" without naming it.  

```scala
@main def main() =
    // ? is the existential wildcard in Scala 3
    def printAll(xs: List[?]): Unit =
        xs.foreach(x => println(x.toString))

    def sizeOf(xs: Seq[?]): Int = xs.length

    def heterogeneous(sources: List[Iterable[?]]): Int =
        sources.map(_.size).sum

    val ints:   List[Int]    = List(1, 2, 3)
    val strs:   List[String] = List("a", "b")
    val nested: List[Iterable[?]] = List(ints, strs, Set(true, false))

    printAll(ints)
    printAll(strs)
    println(sizeOf(ints))
    println(heterogeneous(nested))
end main
```

An existential type `List[?]` means "a `List` of some type, but we  
do not know or need to know which". Scala 3 uses `?` as the wildcard  
where Scala 2 used `_` or explicit `forSome`. Existential types arise  
naturally in heterogeneous collection APIs, Java interop, and any  
context where the element type must be forgotten. Inside a function  
that receives `List[?]`, only methods defined on `Any` are available  
on the elements.  

---

## Covariance

A covariant type parameter allows the container to be used where a  
container of a wider type is expected.  

```scala
class Box[+A](val value: A):
    override def toString: String = s"Box($value)"

@main def main() =
    class Animal(val name: String):
        override def toString = s"Animal($name)"
    class Dog(name: String) extends Animal(name):
        override def toString = s"Dog($name)"
    class Labrador(name: String) extends Dog(name):
        override def toString = s"Labrador($name)"

    val labBox: Box[Labrador] = Box(Labrador("Buddy"))
    val dogBox: Box[Dog]      = labBox   // covariance: Labrador <: Dog
    val anyBox: Box[Animal]   = labBox   // covariance: Labrador <: Animal

    println(labBox)
    println(dogBox)
    println(anyBox)
end main
```

Annotating a type parameter `+A` declares that `Container[Sub]` is  
a subtype of `Container[Super]` whenever `Sub` is a subtype of  
`Super`. This matches intuition for read-only containers: if you have  
a box of dogs you certainly have a box of animals. The compiler  
enforces that `+A` only appears in covariant (output) positions inside  
the class body: method return types and immutable field types. Placing  
`+A` in an argument type would be a compile error because it would  
violate type safety.  

## Contravariance

A contravariant type parameter allows a consumer of a wider type to  
be used where a consumer of a narrower type is expected.  

```scala
trait Printer[-A]:
    def print(value: A): Unit

class AnimalPrinter extends Printer[Animal]:
    def print(a: Animal): Unit = println(s"Animal: ${a.name}")

class Animal(val name: String)
class Dog(name: String) extends Animal(name)

@main def main() =
    val animalPrinter: Printer[Animal] = AnimalPrinter()

    // Printer[-A]: Printer[Animal] is a subtype of Printer[Dog]
    val dogPrinter: Printer[Dog] = animalPrinter

    dogPrinter.print(Dog("Rex"))
    dogPrinter.print(Dog("Buddy"))

    val printers: List[Printer[Dog]] =
        List(animalPrinter, new Printer[Dog]:
            def print(d: Dog) = println(s"Dog: ${d.name}"))

    printers.foreach(_.print(Dog("Fido")))
end main
```

Contravariance `-A` reverses the subtyping relationship: if `Dog <: Animal`  
then `Printer[Animal] <: Printer[Dog]`. This makes sense for consumers:  
a printer that handles any `Animal` can certainly handle a `Dog`. The  
compiler allows `-A` only in contravariant (input) positions, such as  
method parameter types. Placing `-A` in a return type would be unsound  
and is rejected at compile time. Function types illustrate this  
naturally: `Function1` is `[-A, +B]`.  

## Invariance

An invariant type parameter permits no implicit subtyping between  
different instantiations.  

```scala
class Container[A](var value: A):
    def get: A       = value
    def set(v: A): Unit = value = v
    override def toString: String = s"Container($value)"

@main def main() =
    class Animal
    class Dog extends Animal
    class Cat extends Animal

    val dogBox: Container[Dog] = Container(Dog())

    // val animalBox: Container[Animal] = dogBox  // Compile error!
    // If allowed: animalBox.set(Cat()) would put a Cat in a Dog box

    // Explicit widening through a new container is safe
    val animalBox: Container[Animal] = Container(dogBox.get)
    animalBox.set(Cat())  // safe: animalBox is truly Container[Animal]

    println(dogBox)
    println(animalBox)
end main
```

Invariance `A` (no annotation) means that `Container[Dog]` and  
`Container[Animal]` are completely unrelated types even though  
`Dog <: Animal`. Invariance is required for mutable containers  
because allowing subtyping in both directions at once would be  
unsound: you could write a `Cat` into what the compiler believes is  
a `Container[Dog]`. Java arrays are covariant and this exact bug is  
possible at runtime; Scala's invariant mutable containers prevent it  
at compile time.  

## Variance in generic types

Choosing the right variance annotation determines how a generic type  
participates in the subtype lattice.  

```scala
// Covariant: read-only (producer)
sealed trait Result[+A]:
    def map[B](f: A => B): Result[B]

case class  Ok[+A](value: A)  extends Result[A]:
    def map[B](f: A => B): Result[B] = Ok(f(value))
    override def toString = s"Ok($value)"

case object Fail extends Result[Nothing]:
    def map[B](f: Nothing => B): Result[B] = Fail
    override def toString = "Fail"

@main def main() =
    val r1: Result[Int]  = Ok(42)
    val r2: Result[Int]  = Fail

    println(r1.map(_ * 2))
    println(r2.map(_ * 2))

    // Covariance: Result[Int] <: Result[Any]
    val r3: Result[Any] = r1
    println(r3)

    val results: List[Result[Int]] = List(Ok(1), Fail, Ok(2), Fail)
    println(results.collect { case Ok(v) => v })
end main
```

Declaring `Result[+A]` means `Result[Int]` is a subtype of  
`Result[Any]`, enabling generic result-handling code to operate on  
any specific result without loss of safety. `Fail` is typed as  
`Result[Nothing]` and can be used wherever any `Result[X]` is  
expected because `Nothing <: X` for all `X`. This is the standard  
idiom for neutral values in covariant types throughout the standard  
library (`None`, `Nil`, `Left`, `Right`).  

## Variance in function types

Function types are contravariant in their input and covariant in  
their output.  

```scala
@main def main() =
    class Animal:
        override def toString = "Animal"
    class Dog extends Animal:
        override def toString = "Dog"
    class Labrador extends Dog:
        override def toString = "Labrador"

    val animalToAnimal: Animal  => Animal  = x => x
    val animalToDog:    Animal  => Dog     = _ => Dog()
    val dogToAnimal:    Dog     => Animal  = x => x

    // Contravariant input: Dog => * is a supertype of Animal => *
    // Covariant output:    * => Dog is a subtype of * => Animal
    val fn1: Dog    => Animal = animalToAnimal  // input widens, output narrows
    val fn2: Animal => Animal = animalToDog     // output narrows to Animal

    println(fn1(Dog()))
    println(fn2(Animal()))

    val fns: List[Animal => Animal] = List(animalToAnimal, animalToDog)
    fns.foreach(f => println(f(Animal())))
end main
```

The declaration of `Function1[-A, +B]` encodes the Liskov  
Substitution Principle at the type level. A function that accepts  
`Animal` is safe to use where a function that accepts `Dog` is  
expected, because it handles a strictly larger set of inputs. A  
function that returns `Dog` is safe to use where one returning  
`Animal` is expected, because a `Dog` is-an `Animal`. These two  
directions of variance in a single type reflect the fundamental  
duality between producers and consumers.  

## Variance pitfalls

Incorrect variance annotations lead to unsound code that the compiler  
rejects.  

```scala
// This class is correctly invariant: it both reads and writes A
class Cell[A](var value: A):
    def read: A       = value
    def write(v: A): Unit = value = v

// This would be unsound if Cell were covariant (+A):
// val dogCell: Cell[Dog] = Cell(Dog())
// val animalCell: Cell[Animal] = dogCell   // if allowed...
// animalCell.write(Cat())                  // ...puts a Cat in a Dog cell!

// Read-only view is safely covariant
trait ReadCell[+A]:
    def read: A

// Write-only view is safely contravariant
trait WriteCell[-A]:
    def write(v: A): Unit

@main def main() =
    class Animal
    class Dog extends Animal
    class Cat extends Animal

    val cell = Cell(Dog())

    val reader: ReadCell[Animal] = new ReadCell[Dog]:
        def read: Dog = cell.read   // covariant read is safe

    println(reader.read)
end main
```

The compiler checks each position where a type parameter appears and  
rejects annotations that would create unsoundness. A `+A` parameter  
may only appear in output (covariant) positions: return types,  
covariant generic arguments. A `-A` parameter may only appear in  
input (contravariant) positions: method parameter types. Splitting a  
mutable type into a read-only `ReadCell[+A]` and a write-only  
`WriteCell[-A]` is the standard pattern for safely exposing the two  
halves independently while keeping the implementation invariant.  

## Safe API design with variance

Designing APIs around variance separates read and write concerns,  
increasing composability.  

```scala
// Producer: covariant
trait Source[+A]:
    def get(): A

// Consumer: contravariant
trait Sink[-A]:
    def put(value: A): Unit

// Bidirectional: invariant
class Channel[A](private var data: A) extends Source[A] with Sink[A]:
    def get(): A         = data
    def put(value: A): Unit = data = value

@main def main() =
    class Animal(val name: String):
        override def toString = s"Animal($name)"
    class Dog(name: String) extends Animal(name):
        override def toString = s"Dog($name)"

    val channel: Channel[Dog] = Channel(Dog("Rex"))

    val source: Source[Animal] = channel  // Channel[Dog] <: Source[Animal]
    val sink:   Sink[Animal]   = channel  // Channel[Dog] <: Sink[Animal]

    println(s"Reading: ${source.get()}")
    sink.put(Dog("Buddy"))
    println(s"After write: ${channel.get()}")
end main
```

Encoding producers as `Source[+A]` and consumers as `Sink[-A]` gives  
callers maximum flexibility: a `Channel[Dog]` can be passed as a  
`Source[Animal]` for reading and as a `Sink[Animal]` for writing.  
This mirrors the `? extends T` and `? super T` wildcard pattern from  
Java generics (the PECS rule: Producer Extends, Consumer Super) but  
expresses it at declaration site rather than use site, making the  
intent visible in the type definition itself.  

## Variance and collections

The standard library collections use variance annotations to balance  
safety and flexibility.  

```scala
@main def main() =
    class Shape(val name: String):
        override def toString = s"Shape($name)"
    class Circle(name: String) extends Shape(name):
        override def toString = s"Circle($name)"

    // List is covariant: List[+A]
    val circles: List[Circle]  = List(Circle("c1"), Circle("c2"))
    val shapes:  List[Shape]   = circles   // safe upcast

    println(shapes)

    // Prepend a Shape to a List[Circle] -> widens to List[Shape]
    val more: List[Shape] = Shape("square") :: circles
    println(more)

    // mutable.ListBuffer is invariant
    import scala.collection.mutable
    val buf = mutable.ListBuffer(Circle("c3"))
    // val widened: mutable.ListBuffer[Shape] = buf  // Compile error

    // Safe widening through conversion
    val safeBuf: mutable.ListBuffer[Shape] =
        mutable.ListBuffer.from(circles)
    println(safeBuf)
end main
```

Immutable collections like `List`, `Vector`, and `Set` are covariant  
because they never allow mutation of existing elements. Adding an  
element to a `List[Circle]` with `::` returns a new `List[Shape]`  
rather than modifying the original, so the upcast is always safe.  
Mutable collections such as `ArrayBuffer` and `ListBuffer` are  
invariant because they support `update` and `append`, which would  
allow the type confusion described in the variance pitfalls section.  

## Variance and subtyping

Variance determines how generic types participate in the subtyping  
relationship between their instantiations.  

```scala
@main def main() =
    class Animal
    class Dog extends Animal
    class Cat extends Animal

    // Covariant: Option[+A]
    val maybeDog: Option[Dog]    = Some(Dog())
    val maybeAnimal: Option[Animal] = maybeDog   // Dog <: Animal => Option[Dog] <: Option[Animal]

    // Covariant on both sides: Either[+E, +A]
    val errDog: Either[String, Dog] = Right(Dog())
    val errAnimal: Either[String, Animal] = errDog

    // Contravariant: Ordering[-T]
    val animalOrd: Ordering[Animal] = Ordering.by(_ => 0)
    val dogOrd: Ordering[Dog]       = animalOrd  // Animal >: Dog => Ordering[Animal] <: Ordering[Dog]

    println(maybeAnimal)
    println(errAnimal)
    println(dogOrd.compare(Dog(), Dog()))
end main
```

Variance propagates through the subtype lattice in a predictable way.  
Covariance preserves the subtyping direction: `F[Sub] <: F[Super]`  
when `F` is covariant and `Sub <: Super`. Contravariance reverses  
it: `F[Super] <: F[Sub]`. Invariance breaks both relationships. These  
rules compose: in `Either[+E, +A]` both parameters are covariant, so  
both can be widened independently, giving the most flexible type  
assignment.  

## Covariant return types

A method in a subclass can return a more specific type than the  
method it overrides.  

```scala
abstract class AnimalFactory:
    def create(name: String): Animal

class Animal(val name: String):
    override def toString: String = s"Animal($name)"

class Dog(name: String) extends Animal(name):
    def fetch(): String = s"$name fetches!"
    override def toString: String = s"Dog($name)"

class DogFactory extends AnimalFactory:
    override def create(name: String): Dog = Dog(name)

@main def main() =
    val factory: AnimalFactory = DogFactory()

    // Through the base type, the result is Animal
    val animal: Animal = factory.create("Rex")
    println(animal)

    // Through the concrete type, the result is Dog
    val dogFactory = DogFactory()
    val dog: Dog    = dogFactory.create("Buddy")
    println(dog.fetch())
end main
```

Covariant return types are a direct consequence of covariance in  
method types. When a subclass overrides a method and narrows the  
return type, it provides additional information to callers who use  
the concrete class directly while remaining compatible with callers  
who use the base-class interface. This pattern is used throughout the  
Scala collections hierarchy: `map` on a `List` returns a `List`,  
not a generic `Iterable`, preserving the concrete collection type  
when accessed through the specific class.  

## Invariant mutable containers

Mutable containers must be invariant to prevent unsound type  
assignments.  

```scala
import scala.collection.mutable

@main def main() =
    class Animal(val name: String):
        override def toString = s"Animal($name)"
    class Dog(name: String) extends Animal(name):
        override def toString = s"Dog($name)"
    class Cat(name: String) extends Animal(name):
        override def toString = s"Cat($name)"

    val dogs = mutable.ArrayBuffer(Dog("Rex"), Dog("Buddy"))

    // Uncommenting the next line would cause a compile error:
    // val animals: mutable.ArrayBuffer[Animal] = dogs

    // Correct widening: copy to a new buffer
    val animals: mutable.ArrayBuffer[Animal] =
        mutable.ArrayBuffer.from(dogs)

    animals += Cat("Whiskers")  // safe: animals is truly ArrayBuffer[Animal]
    println(animals)

    // The original dogs buffer is unaffected
    println(dogs)
end main
```

If `ArrayBuffer[Dog]` were allowed to be used as `ArrayBuffer[Animal]`  
directly, then adding a `Cat` through the widened reference would  
corrupt the original buffer. The `from` copy creates a new  
`ArrayBuffer[Animal]` that truly owns animals of any subtype, so  
subsequent mutations are safe. The invariance of mutable containers  
is not a limitation but a correctness guarantee: the compiler ensures  
that every element in a `mutable.ArrayBuffer[Dog]` is always a `Dog`.  

## Declaration-site variance

Scala uses declaration-site variance, annotating variance on the type  
constructor itself rather than at each use site.  

```scala
// Covariant in A and E: declared once on the trait
sealed trait Outcome[+E, +A]:
    def map[B](f: A => B): Outcome[E, B]

case class  Succeeded[+A](result: A) extends Outcome[Nothing, A]:
    def map[B](f: A => B): Outcome[Nothing, B] = Succeeded(f(result))
    override def toString = s"Succeeded($result)"

case class  Failed[+E](error: E) extends Outcome[E, Nothing]:
    def map[B](f: Nothing => B): Outcome[E, B] = this
    override def toString = s"Failed($error)"

def firstSuccess[E, A](outcomes: List[Outcome[E, A]]): Option[A] =
    outcomes.collectFirst { case Succeeded(v) => v }

@main def main() =
    val outcomes: List[Outcome[String, Int]] = List(
        Failed("not found"),
        Succeeded(42),
        Succeeded(100)
    )

    println(firstSuccess(outcomes))

    // Declaration-site: no wildcards needed at use sites
    val wide: List[Outcome[Any, Int]] = outcomes
    println(wide.head)
end main
```

Java uses use-site variance (wildcards): every call site that needs  
flexibility must write `? extends T` or `? super T` explicitly,  
scattering variance annotations throughout the codebase. Scala's  
declaration-site variance annotates the type constructor once, and  
the variance propagates automatically to all use sites. This makes  
APIs cleaner, reduces boilerplate, and makes the intent of each  
type's variance visible at the definition rather than discovered  
piecemeal at call sites.  
