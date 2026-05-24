# Advanced Functional Programming in Scala 3

Advanced functional programming (FP) moves beyond basic immutable values and  
pure functions to embrace composable abstractions that scale gracefully from  
small utilities to entire application architectures. Where introductory FP  
teaches you to avoid side effects, advanced FP teaches you to *control* them  
through principled abstractions such as typeclasses, monads, and effect  
systems. The result is code that is easier to test, refactor, and reason  
about, even as it grows in complexity.  

Scala 3 is exceptionally well-suited to advanced functional programming. Its  
redesigned typeclass machinery — `given`, `using`, and `summon` — makes  
ad-hoc polymorphism concise and readable. Opaque types enable zero-cost domain  
modeling without boxing. `enum` and sealed traits make algebraic data types  
first-class citizens. Inline definitions and match types open doors to  
compile-time computation that would have required macros in Scala 2.  

Typeclasses are the backbone of functional design. A typeclass defines a  
capability (e.g., *equality*, *ordering*, *mapping*) as a trait with type  
parameters, then provides `given` instances for each type that supports that  
capability. Code parameterized over a typeclass works with any type for which  
an instance exists, achieving the kind of polymorphism that object hierarchies  
struggle to express.  

Functors, applicatives, and monads form a hierarchy of abstractions for  
working with values inside containers or computational contexts. A `Functor`  
lets you apply a pure function inside a context. An `Applicative` lets you  
apply a function that is itself inside a context. A `Monad` lets you sequence  
computations where each step may produce a new context. Together they cover  
most patterns that arise when composing effectful code.  

Monad transformers stack multiple monadic effects. `OptionT` adds optionality  
to any monad. `EitherT` adds typed error handling. Stacking transformers lets  
you combine effects (e.g., async + error + optional) without losing type  
safety. While transformers can become verbose, they are a powerful tool for  
controlled effect composition in domain logic.  

The tagless-final pattern separates an algebra (a trait describing operations)  
from its interpreters (concrete `given` implementations). Because the effect  
type `F[_]` is abstract, the same business logic can run against a production  
interpreter, a test stub, or a logging decorator without changing a line of  
domain code. Combined with Scala 3's given instances, tagless final achieves  
dependency injection without a framework.  

Functional domain modeling uses ADTs to represent valid states, opaque types  
to enforce invariants, and monads to chain workflows. A well-modeled domain  
makes illegal states unrepresentable and encodes business rules in types rather  
than documentation. This leads to code that is self-documenting, safe by  
construction, and straightforward to evolve.  

Taken together, these techniques produce software that is safer and more  
maintainable because correctness guarantees are encoded in types and enforced  
by the compiler. Bugs that once required runtime tests to catch become  
compile-time errors. Refactors that once required careful grep-based audits  
become mechanical type-directed changes. Advanced FP in Scala 3 is not about  
theoretical purity — it is a pragmatic path to better software.  

---

## Defining a typeclass

A typeclass is a trait that expresses a capability for an abstract type.  

```scala
trait Show[A]:
  def show(a: A): String

given Show[Int] with
  def show(a: Int): String = s"Int($a)"

given Show[String] with
  def show(a: String): String = s"\"$a\""

def printIt[A](a: A)(using s: Show[A]): Unit =
  println(s.show(a))

@main def main() =
  printIt(42)
  printIt("hello")
end main
```

`Show` is a typeclass that describes how to render a value as a `String`.  
Two `given` instances provide implementations for `Int` and `String`  
respectively. The `printIt` function is parameterised over any type `A`  
for which a `Show` instance exists; the compiler locates the right instance  
automatically at the call site. Typeclasses achieve open extensibility: new  
instances can be added for any type without modifying existing code.  

---

## Given instances

Given instances supply typeclass evidence that the compiler injects  
automatically at call sites.  

```scala
trait Describable[A]:
  def describe(a: A): String

given Describable[Boolean] with
  def describe(a: Boolean): String =
    if a then "yes" else "no"

given Describable[Double] with
  def describe(a: Double): String =
    f"$a%.2f"

def describe[A](a: A)(using d: Describable[A]): String =
  d.describe(a)

@main def main() =
  println(describe(true))
  println(describe(3.14159))
end main
```

The `using` parameter clause marks `d` as a *contextual parameter*: the  
caller does not pass it explicitly; the compiler searches the implicit  
scope and injects the matching `given` instance. Naming the parameter is  
optional — you can write `using Describable[A]` when you only forward it  
rather than use it directly. Given instances replace Scala 2 implicits  
with clearer syntax and a more predictable resolution algorithm.  

---

## Summon

`summon` retrieves a given instance by type, acting as the Scala 3  
replacement for `implicitly`.  

```scala
trait Codec[A]:
  def encode(a: A): String
  def decode(s: String): Option[A]

given Codec[Int] with
  def encode(a: Int): String = a.toString
  def decode(s: String): Option[Int] =
    s.toIntOption

@main def main() =
  val codec = summon[Codec[Int]]
  val encoded = codec.encode(123)
  println(s"encoded: $encoded")
  val decoded = codec.decode(encoded)
  println(s"decoded: $decoded")
  println(s"bad: ${codec.decode("abc")}")
end main
```

`summon[Codec[Int]]` asks the compiler to find and return the given  
`Codec[Int]` instance. This is useful when you want to obtain the  
instance as a value rather than forwarding it through a `using` parameter.  
The name `summon` was chosen to make the intent explicit: you are  
*summoning* a specific piece of evidence from the ambient context. It is  
more readable than `implicitly` and benefits from improved error messages  
in Scala 3.  

---

## Typeclass syntax via extension methods

Extension methods allow typeclass operations to be called with dot syntax,  
making generic code read like ordinary method calls.  

```scala
trait Printable[A]:
  def render(a: A): String

given Printable[Int] with
  def render(a: Int): String = s"#$a"

given Printable[List[Int]] with
  def render(a: List[Int]): String =
    a.map(_.toString).mkString("[", ", ", "]")

extension [A](a: A)(using p: Printable[A])
  def printed: String = p.render(a)

@main def main() =
  println(42.printed)
  println(List(1, 2, 3).printed)
end main
```

The `extension` block introduces `printed` as a method on any type `A`  
for which a `Printable` instance exists. At the call site `42.printed`  
looks like a plain method call, but the compiler desugars it to  
`summon[Printable[Int]].render(42)`. This syntactic sugar reduces  
boilerplate and makes generic APIs feel natural to use. Extension methods  
in Scala 3 are hygienic: they do not pollute the original type and are  
only in scope when the relevant import or given is visible.  

---

## Functor

A Functor expresses the ability to apply a pure function to a value  
wrapped in a context, leaving the structure of the context unchanged.  

```scala
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

given Functor[Option] with
  def map[A, B](fa: Option[A])(f: A => B): Option[B] =
    fa.map(f)

given Functor[List] with
  def map[A, B](fa: List[A])(f: A => B): List[B] =
    fa.map(f)

extension [F[_], A](fa: F[A])(using F: Functor[F])
  def fmap[B](f: A => B): F[B] = F.map(fa)(f)

@main def main() =
  val opt = Option(3).fmap(_ * 10)
  println(opt)
  val lst = List(1, 2, 3).fmap(_ + 100)
  println(lst)
end main
```

`Functor` is parameterised on a type constructor `F[_]` — a type that  
takes one type argument. The `map` method transforms the inner value  
without altering the outer structure. The extension method `fmap` gives  
every value wrapped in a `Functor` context convenient dot-syntax access.  
Functors satisfy two laws: mapping the identity function must return the  
original value, and mapping two composed functions must equal mapping  
them sequentially. These laws guarantee predictable behaviour and enable  
equational reasoning.  

---

## Applicative

An Applicative extends Functor with the ability to lift a pure value into  
a context and to apply a function that is itself inside a context.  

```scala
trait Applicative[F[_]] extends Functor[F]:
  def pure[A](a: A): F[A]
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]
  def map[A, B](fa: F[A])(f: A => B): F[B] =
    ap(pure(f))(fa)

given Applicative[Option] with
  def pure[A](a: A): Option[A] = Some(a)
  def ap[A, B](ff: Option[A => B])(fa: Option[A]): Option[B] =
    for f <- ff; a <- fa yield f(a)

@main def main() =
  val add: Int => Int => Int = x => y => x + y
  val result =
    summon[Applicative[Option]].ap(
      summon[Applicative[Option]].ap(
        Some(add))(Some(3)))(Some(4))
  println(result)
  println(summon[Applicative[Option]].pure(99))
end main
```

`pure` wraps a plain value in the minimal context — `Some` for `Option`.  
`ap` is the key operation: it unwraps both the function and the argument  
from their respective contexts and applies one to the other. When either  
context is `None`, the entire result is `None`, propagating the  
"no value" condition automatically. Applicatives are strictly more  
powerful than Functors but do not support the sequential dependency that  
Monads provide; this restriction makes them easier to reason about and  
opens the door to parallel execution.  

---

## Contravariant Functor

A Contravariant Functor maps a function *into* the input of a context,  
dual to the covariant Functor which maps over the output.  

```scala
trait Contravariant[F[_]]:
  def contramap[A, B](fa: F[A])(f: B => A): F[B]

case class Predicate[A](test: A => Boolean)

given Contravariant[Predicate] with
  def contramap[A, B](fa: Predicate[A])(f: B => A): Predicate[B] =
    Predicate(b => fa.test(f(b)))

extension [F[_], A](fa: F[A])(using C: Contravariant[F])
  def cmap[B](f: B => A): F[B] = C.contramap(fa)(f)

@main def main() =
  val isLong: Predicate[String]  = Predicate(_.length > 5)
  val isLargeInt: Predicate[Int] = isLong.cmap(_.toString)
  val isShortList: Predicate[List[Int]] =
    isLong.cmap(xs => xs.mkString(","))

  println(isLong.test("hello"))
  println(isLong.test("functional"))
  println(isLargeInt.test(123456))
  println(isLargeInt.test(99))
  println(isShortList.test(List(1, 2, 3, 4, 5, 6)))
end main
```

`Predicate[A]` wraps a function `A => Boolean`. Applying `contramap`  
with `f: B => A` produces `Predicate[B]` that first converts a `B` to  
an `A` and then applies the original test. The direction is *reversed*  
compared to covariant `map`: instead of transforming the output, we  
adapt the input. Contravariant functors appear naturally in comparators,  
serializers, and predicates — anywhere the type parameter is consumed  
rather than produced. The `cmap` extension method gives dot-syntax  
access, mirroring the `fmap` pattern from the Functor section.  

---

## Monad

A Monad extends Applicative with `flatMap`, enabling sequential  
computations where each step can inspect the previous result.  

```scala
trait Monad[F[_]] extends Applicative[F]:
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
  override def ap[A, B](ff: F[A => B])(fa: F[A]): F[B] =
    flatMap(ff)(f => map(fa)(f))

given Monad[Option] with
  def pure[A](a: A): Option[A] = Some(a)
  def flatMap[A, B](fa: Option[A])(f: A => Option[B]): Option[B] =
    fa.flatMap(f)

def safeDiv(a: Int, b: Int): Option[Int] =
  if b == 0 then None else Some(a / b)

@main def main() =
  val result =
    summon[Monad[Option]].flatMap(Some(10))(x =>
      summon[Monad[Option]].flatMap(Some(2))(y =>
        safeDiv(x, y)))
  println(result)
  println(safeDiv(10, 0))
end main
```

`flatMap` is the distinguishing operation of a Monad. Unlike `map`, the  
function `f` itself returns `F[B]`, so the Monad must *flatten* the  
resulting `F[F[B]]` into `F[B]`. This flattening behaviour is what  
enables sequential, dependent computations: each step can fail, short-  
circuit, produce multiple values, or perform effects, and the monad  
handles the bookkeeping automatically. The `safeDiv` helper demonstrates  
a common monad pattern — returning `None` on invalid input — so that  
callers never need to check for division-by-zero explicitly.  

---

## Semigroup

A Semigroup provides an associative binary operation `combine` for  
merging two values of the same type.  

```scala
trait Semigroup[A]:
  def combine(x: A, y: A): A

given Semigroup[String] with
  def combine(x: String, y: String): String = x + y

given Semigroup[Int] with
  def combine(x: Int, y: Int): Int = x + y

given [A](using s: Semigroup[A]): Semigroup[List[A]] with
  def combine(x: List[A], y: List[A]): List[A] = x ++ y

extension [A](a: A)(using s: Semigroup[A])
  def |+|(b: A): A = s.combine(a, b)

@main def main() =
  println("Hello, " |+| "world!")
  println(10 |+| 32)
  println(List(1, 2) |+| List(3, 4))
end main
```

The associativity law — `combine(a, combine(b, c))` equals  
`combine(combine(a, b), c)` — guarantees that the order in which we  
group a sequence of combines does  
not affect the final result. This law makes Semigroups safe to use in  
parallel reduction: any partition of the input produces the same answer.  
The derived `List[A]` instance shows how a `given` can be parameterised  
over another `given`, building richer instances from simpler ones.  

---

## Monoid

A Monoid extends Semigroup with an identity element `empty` such that  
combining any value with `empty` returns that value unchanged.  

```scala
trait Monoid[A] extends Semigroup[A]:
  def empty: A

given Monoid[String] with
  def empty: String = ""
  def combine(x: String, y: String): String = x + y

given Monoid[Int] with
  def empty: Int = 0
  def combine(x: Int, y: Int): Int = x + y

def combineAll[A](xs: List[A])(using m: Monoid[A]): A =
  xs.foldLeft(m.empty)(m.combine)

@main def main() =
  println(combineAll(List("a", "b", "c")))
  println(combineAll(List(1, 2, 3, 4, 5)))
  println(combineAll(List.empty[Int]))
end main
```

`empty` is the zero element: `combine(x, empty) == x` and  
`combine(empty, x) == x`. This identity law means `combineAll` is  
well-defined even on an empty list: it simply returns `empty`. Monoids  
underpin many higher-level abstractions — `foldMap`, `WriterT`, and  
streaming pipelines all rely on the guarantee that values can be  
combined freely, in any order, with a safe fallback for the empty case.  

---

## Foldable

A Foldable typeclass abstracts over data structures that can be  
reduced to a summary value using a binary operation and a seed.  

```scala
trait Foldable[F[_]]:
  def foldLeft[A, B](fa: F[A], seed: B)(f: (B, A) => B): B
  def toList[A](fa: F[A]): List[A] =
    foldLeft(fa, List.empty[A])((acc, a) => acc :+ a)
  def size[A](fa: F[A]): Int =
    foldLeft(fa, 0)((n, _) => n + 1)

given Foldable[List] with
  def foldLeft[A, B](fa: List[A], seed: B)(f: (B, A) => B): B =
    fa.foldLeft(seed)(f)

given Foldable[Option] with
  def foldLeft[A, B](fa: Option[A], seed: B)(f: (B, A) => B): B =
    fa.fold(seed)(a => f(seed, a))

extension [F[_], A](fa: F[A])(using F: Foldable[F])
  def foldedLeft[B](seed: B)(f: (B, A) => B): B =
    F.foldLeft(fa, seed)(f)
  def asListF: List[A]  = F.toList(fa)
  def sizeF: Int        = F.size(fa)

@main def main() =
  println(List(1, 2, 3, 4).foldedLeft(0)(_ + _))
  println(Option(42).foldedLeft(0)(_ + _))
  println(Option.empty[Int].foldedLeft(0)(_ + _))
  println(List("a", "b", "c").asListF)
  println(Option("x").sizeF)
end main
```

`Foldable` is parameterised on a type constructor `F[_]` and requires  
only `foldLeft`; the derived methods `toList` and `size` are implemented  
in terms of it. The `Option` instance treats `None` as empty (returning  
the seed) and `Some(a)` as a single-element collection. Foldable is one  
of the most fundamental typeclasses in functional libraries: it is the  
foundation for `combineAll`, `find`, `exists`, and many other utilities  
that work uniformly across lists, trees, streams, and other containers.  

---

## Deriving typeclass instances

Scala 3 allows typeclasses to be derived automatically for product and  
sum types using `derives`.  

```scala
import scala.deriving.Mirror

trait Eq[A]:
  def eqv(a: A, b: A): Boolean

object Eq:
  given Eq[Int] with
    def eqv(a: Int, b: Int): Boolean = a == b
  given Eq[String] with
    def eqv(a: String, b: String): Boolean = a == b
  inline def derived[A](using m: Mirror.Of[A]): Eq[A] =
    (a, b) => a == b   // structural equality via mirror

case class Point(x: Int, y: Int) derives Eq

extension [A](a: A)(using e: Eq[A])
  def ===(b: A): Boolean = e.eqv(a, b)

@main def main() =
  println(Point(1, 2) === Point(1, 2))
  println(Point(1, 2) === Point(3, 4))
  println(42 === 42)
end main
```

The `derives` clause on `Point` instructs the compiler to synthesise an  
`Eq[Point]` instance using the `derived` method defined in the `Eq`  
companion object. The `Mirror.Of[A]` mechanism provides compile-time  
structural information about the type — its labels, arity, and element  
types — which the `inline derived` method can use to build the instance  
automatically. This eliminates tedious boilerplate for data classes and  
ensures derived instances stay in sync with the type definition.  

---

## Typeclass coherence

Typeclass coherence ensures that for any given type, exactly one instance  
is selected, preventing ambiguity and non-deterministic behaviour.  

```scala
trait Hash[A]:
  def hash(a: A): Int

object Hash:
  given Hash[Int] with
    def hash(a: Int): Int = a.hashCode
  given Hash[String] with
    def hash(a: String): Int = a.hashCode
  given [A, B](using ha: Hash[A], hb: Hash[B]): Hash[(A, B)] with
    def hash(p: (A, B)): Int =
      31 * ha.hash(p._1) + hb.hash(p._2)

def hashOf[A](a: A)(using h: Hash[A]): Int = h.hash(a)

@main def main() =
  println(hashOf(42))
  println(hashOf("hello"))
  println(hashOf((1, "x")))
end main
```

Scala 3 enforces coherence by treating given instances as unique per  
type in a given scope. Two competing instances for the same type cause a  
*diverging implicit* or *ambiguous implicit* error at compile time, not a  
silent wrong result at runtime. The derived `Hash[(A, B)]` instance  
demonstrates *instance composition*: a pair is hashable whenever both  
components are hashable, and the compiler constructs this composed  
instance automatically. Coherence is what makes typeclass-based  
programming safe to use in large codebases with multiple collaborators.  

---

## Typeclasses with match types

Match types let you compute a type at compile time, enabling typeclasses  
that adapt their behaviour based on the shape of the input type.  

```scala
type Elem[X] = X match
  case List[a] => a
  case Option[a] => a
  case Array[a] => a

trait Container[F[_]]:
  def elements[A](fa: F[A]): List[A]

given Container[List] with
  def elements[A](fa: List[A]): List[A] = fa

given Container[Option] with
  def elements[A](fa: Option[A]): List[A] = fa.toList

def elems[F[_], A](fa: F[A])(using c: Container[F]): List[A] =
  c.elements(fa)

@main def main() =
  println(elems(List(1, 2, 3)))
  println(elems(Option(42)))
  println(elems(Option.empty[String]))
end main
```

The `Elem` match type is a type-level function: given `List[Int]` it  
returns `Int`, given `Option[String]` it returns `String`. This pattern  
is useful in generic utilities that need to express relationships between  
a container type and its element type at the type level. The `Container`  
typeclass then provides the runtime capability to extract elements,  
while the match type captures the invariant in the type signature. The  
combination of match types and typeclasses allows compile-time checked,  
adaptive generic programming.  

---

## Option as a monad

`Option` models computations that may produce a value or nothing, with  
short-circuit propagation of the empty case.  

```scala
def parseInt(s: String): Option[Int] = s.toIntOption

def safeSqrt(n: Int): Option[Double] =
  if n >= 0 then Some(math.sqrt(n.toDouble)) else None

def parseAndSqrt(s: String): Option[Double] =
  for
    n   <- parseInt(s)
    res <- safeSqrt(n)
  yield res

@main def main() =
  println(parseAndSqrt("16"))
  println(parseAndSqrt("-4"))
  println(parseAndSqrt("abc"))
end main
```

The for-comprehension desugars to nested `flatMap` and `map` calls.  
If `parseInt` returns `None`, the rest of the chain is never executed  
and the overall result is immediately `None`. This automatic short-  
circuiting eliminates deeply nested null-checks and makes the happy  
path the primary narrative. Using `Option` as a monad is idiomatic  
Scala: it communicates to the reader that a computation may legitimately  
produce no value, without resorting to exceptions or sentinel values.  

---

## Either as a monad

`Either[E, A]` models computations that may fail with a typed error `E`  
or succeed with a value `A`, supporting for-comprehension sequencing.  

```scala
enum AppError:
  case ParseError(msg: String)
  case DomainError(msg: String)

def parseInt(s: String): Either[AppError, Int] =
  s.toIntOption.toRight(AppError.ParseError(s"Not a number: $s"))

def validate(n: Int): Either[AppError, Int] =
  if n > 0 then Right(n)
  else Left(AppError.DomainError(s"Must be positive: $n"))

def process(s: String): Either[AppError, String] =
  for
    n <- parseInt(s)
    v <- validate(n)
  yield s"OK: $v"

@main def main() =
  println(process("42"))
  println(process("-1"))
  println(process("xyz"))
end main
```

`Either` is right-biased in Scala 3: `map` and `flatMap` operate on the  
`Right` value, while `Left` values short-circuit and propagate unchanged.  
Using a sealed `enum` for the error type `AppError` ensures all error  
cases are explicit and exhaustively handled in pattern matches. The  
for-comprehension sequences the two validation steps cleanly; if either  
step fails, the corresponding `Left` error is returned to the caller  
without any exception being thrown.  

---

## Try as a monad

`Try[A]` wraps computations that may throw exceptions, capturing the  
exception as a `Failure` value rather than propagating it.  

```scala
import scala.util.{Try, Success, Failure}

def readInt(s: String): Try[Int] = Try(s.toInt)
def divide(a: Int, b: Int): Try[Int] = Try(a / b)

@main def main() =
  val result =
    for
      a <- readInt("100")
      b <- readInt("4")
      r <- divide(a, b)
    yield r

  result match
    case Success(v) => println(s"result: $v")
    case Failure(e) => println(s"error: ${e.getMessage}")

  val bad =
    for
      a <- readInt("100")
      b <- readInt("0")
      r <- divide(a, b)
    yield r

  bad match
    case Success(v) => println(s"result: $v")
    case Failure(e) => println(s"error: ${e.getMessage}")
end main
```

`Try(expr)` evaluates `expr` and wraps any thrown `Throwable` in a  
`Failure`. It is monadic: `flatMap` on a `Failure` short-circuits and  
returns the failure without executing subsequent steps. This makes `Try`  
useful as a bridge between exception-based legacy code and functional  
pipelines. For new code, `Either` is generally preferred because it  
forces you to name the error type rather than relying on `Throwable`.  
However, `Try` remains the pragmatic choice when interoperating with  
Java APIs that signal errors via exceptions.  

---

## Future as a monad

`Future[A]` models an asynchronous computation that will eventually  
produce a value `A` or fail with an exception.  

```scala
import scala.concurrent.{Future, Await}
import scala.concurrent.duration.*
import scala.concurrent.ExecutionContext.Implicits.global

def fetchUser(id: Int): Future[String] =
  Future(s"User-$id")

def fetchScore(user: String): Future[Int] =
  Future(user.length * 10)

@main def main() =
  val pipeline =
    for
      user  <- fetchUser(7)
      score <- fetchScore(user)
    yield s"$user scored $score"

  val result = Await.result(pipeline, 5.seconds)
  println(result)
end main
```

`Future` is monadic: the for-comprehension sequences two asynchronous  
steps so that `fetchScore` only runs after `fetchUser` completes. If  
`fetchUser` fails, `fetchScore` is never invoked. The `ExecutionContext`  
governs on which thread pool the `Future` computations run; it is  
provided as a `given` so the compiler injects it automatically. While  
`Future` is eager (it starts computing immediately upon creation) and  
not referentially transparent, it remains a practical tool for  
asynchronous work in standard Scala projects that do not use a  
dedicated effect library.  

---

## Custom monads

You can define your own monadic type to model any domain-specific  
computational context.  

```scala
case class Result[+A](value: Option[A], log: List[String]):
  def map[B](f: A => B): Result[B] =
    Result(value.map(f), log)
  def flatMap[B](f: A => Result[B]): Result[B] =
    value match
      case None    => Result(None, log)
      case Some(a) =>
        val next = f(a)
        Result(next.value, log ++ next.log)

object Result:
  def ok[A](a: A, msg: String = ""): Result[A] =
    Result(Some(a), if msg.isEmpty then Nil else List(msg))
  def fail[A](msg: String): Result[A] =
    Result(None, List(msg))

@main def main() =
  val r =
    for
      a <- Result.ok(10, "got 10")
      b <- Result.ok(5,  "got 5")
      _ <- if b == 0 then Result.fail("div by zero")
           else Result.ok(())
      c <- Result.ok(a / b, s"$a / $b = ${a / b}")
    yield c

  println(r.value)
  r.log.foreach(println)
end main
```

`Result` combines an optional value with an accumulated log, forming a  
*Writer-like* monad. The `flatMap` implementation propagates `None` on  
failure and concatenates log messages on success. Defining `map` and  
`flatMap` on the case class enables for-comprehension syntax without  
any typeclass machinery. Custom monads like this are useful when the  
standard library types do not capture all the contextual information your  
domain needs — here, both the computation outcome and its audit trail  
travel together through the pipeline.  

---

## Reader monad

The Reader monad threads a shared environment through a computation,  
providing a pure form of dependency injection.  

```scala
case class Reader[R, A](run: R => A):
  def map[B](f: A => B): Reader[R, B] =
    Reader(r => f(run(r)))
  def flatMap[B](f: A => Reader[R, B]): Reader[R, B] =
    Reader(r => f(run(r)).run(r))

object Reader:
  def ask[R]: Reader[R, R]         = Reader(identity)
  def pure[R, A](a: A): Reader[R, A] = Reader(_ => a)

case class Config(dbUrl: String, maxRetries: Int)

def readDb: Reader[Config, String]   = Reader.ask.map(_.dbUrl)
def readRetry: Reader[Config, Int]   = Reader.ask.map(_.maxRetries)

def buildQuery: Reader[Config, String] =
  for
    db  <- readDb
    max <- readRetry
  yield s"Connect to $db with up to $max retries"

@main def main() =
  val cfg = Config("postgres://localhost/app", 3)
  println(buildQuery.run(cfg))
  val devCfg = Config("sqlite::memory:", 1)
  println(buildQuery.run(devCfg))
end main
```

`Reader[R, A]` is essentially a function `R => A` wrapped in a case  
class that provides `map` and `flatMap`. The for-comprehension in  
`buildQuery` reads two fields from the configuration without any  
mutable state or global reference. The `run` method at the end supplies  
the concrete `Config`, separating the *construction* of the computation  
from its *execution*. Reader monad is the functional analogue of  
constructor injection: the environment is provided exactly once at the  
top level, and every sub-computation receives it automatically.  

---

## Writer monad

The Writer monad pairs a computation result with an accumulated log,  
threading the log through the pipeline without explicit passing.  

```scala
case class Writer[W, A](run: (W, A)):
  def map[B](f: A => B): Writer[W, B] =
    Writer((run._1, f(run._2)))
  def flatMap[B](f: A => Writer[W, B])(using
      combine: (W, W) => W): Writer[W, B] =
    val (log1, a) = run
    val (log2, b) = f(a).run
    Writer((combine(log1, log2), b))

object Writer:
  def tell[W](w: W): Writer[W, Unit]   = Writer((w, ()))
  def pure[W, A](a: A, empty: W): Writer[W, A] = Writer((empty, a))

type Logged[A] = Writer[List[String], A]

def logged[A](a: A, msg: String): Logged[A] =
  Writer((List(msg), a))

def pipeline: Logged[Int] =
  given (List[String], List[String]) => List[String] = _ ++ _
  for
    a <- logged(10, "start with 10")
    b <- logged(a * 2, s"doubled to ${a * 2}")
    c <- logged(b + 5, s"added 5 to get ${b + 5}")
  yield c

@main def main() =
  val (log, result) = pipeline.run
  println(s"result: $result")
  log.foreach(msg => println(s"  $msg"))
end main
```

`Writer[W, A]` holds a pair `(W, A)` where `W` is the accumulated  
output (a log, metrics, or any semigroup value). `flatMap` uses the  
`combine` function to merge the logs from each step in order. The  
for-comprehension in `pipeline` reads as a sequence of business steps,  
with each `logged` call transparently appending a message to the  
running log. At the end, `run` returns both the final value and the  
complete audit trail without any mutable state having been used during  
the computation.  

---

## State monad

The State monad threads a mutable-looking state through a pure  
computation, making state transformations explicit and composable.  

```scala
case class State[S, A](run: S => (S, A)):
  def map[B](f: A => B): State[S, B] =
    State(s => val (s2, a) = run(s); (s2, f(a)))
  def flatMap[B](f: A => State[S, B]): State[S, B] =
    State(s => val (s2, a) = run(s); f(a).run(s2))

object State:
  def pure[S, A](a: A): State[S, A] = State(s => (s, a))
  def get[S]: State[S, S]            = State(s => (s, s))
  def put[S](s: S): State[S, Unit]   = State(_ => (s, ()))
  def modify[S](f: S => S): State[S, Unit] =
    State(s => (f(s), ()))

case class BankAccount(balance: Double, txCount: Int)

type AccountState[A] = State[BankAccount, A]

def deposit(amount: Double): AccountState[Double] =
  State.modify[BankAccount](a =>
    a.copy(balance = a.balance + amount, txCount = a.txCount + 1))
    .flatMap(_ => State.get.map(_.balance))

def withdraw(amount: Double): AccountState[Either[String, Double]] =
  State.get[BankAccount].flatMap { acct =>
    if acct.balance >= amount then
      State.modify[BankAccount](a =>
        a.copy(balance = a.balance - amount, txCount = a.txCount + 1))
        .map(_ => Right(acct.balance - amount))
    else
      State.pure(Left(s"Insufficient: ${acct.balance} < $amount"))
  }

@main def main() =
  val program: AccountState[Either[String, Double]] =
    for
      _   <- deposit(100.0)
      _   <- deposit(50.0)
      res <- withdraw(30.0)
    yield res

  val init = BankAccount(0.0, 0)
  val (finalAcct, result) = program.run(init)
  println(s"result:  $result")
  println(s"balance: ${finalAcct.balance}")
  println(s"txCount: ${finalAcct.txCount}")
end main
```

`State[S, A]` is a function `S => (S, A)`: given an initial state it  
produces a new state and a result value. `flatMap` threads the state  
automatically: the output state of the first step becomes the input  
state of the next, just as if state were a mutable variable. The  
`deposit` and `withdraw` functions express stateful business operations  
as pure values; they can be tested in isolation by supplying an initial  
`BankAccount` and inspecting the returned pair. There is no shared  
mutable field anywhere — the state variable exists only within the  
chain of `run` calls.  

---

## Monadic laws

The three monad laws — left identity, right identity, and associativity —  
can be informally verified with concrete values.  

```scala
trait Monad[F[_]]:
  def pure[A](a: A): F[A]
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]

given Monad[Option] with
  def pure[A](a: A): Option[A] = Some(a)
  def flatMap[A, B](fa: Option[A])(f: A => Option[B]): Option[B] =
    fa.flatMap(f)

def check[F[_], A, B, C](
    a: A, f: A => F[B], g: B => F[C])(using M: Monad[F]): Unit =
  val fa = M.pure(a)
  val leftId  = M.flatMap(M.pure(a))(f) == M.flatMap(fa)(f)
  val rightId = M.flatMap(fa)(M.pure) == fa
  val assoc   =
    M.flatMap(M.flatMap(fa)(f))(g) ==
    M.flatMap(fa)(x => M.flatMap(f(x))(g))
  println(s"left identity:  $leftId")
  println(s"right identity: $rightId")
  println(s"associativity:  $assoc")

@main def main() =
  check[Option, Int, Int, Int](
    5,
    x => if x > 0 then Some(x * 2) else None,
    y => Some(y + 1))
end main
```

Left identity states that `pure(a).flatMap(f) == f(a)`: wrapping a  
value and immediately binding it is the same as applying the function  
directly. Right identity states that `fa.flatMap(pure) == fa`: binding  
with the constructor is a no-op. Associativity states that chaining two  
`flatMap`s is equivalent to nesting them inside a single `flatMap`. These  
laws are not enforced by the Scala compiler; they are a contract between  
the implementer and the user. Violating any law leads to surprising  
behaviour when code is refactored or composed.  

---

## For-comprehensions in depth

For-comprehensions desugar to `map`, `flatMap`, and `withFilter`, and  
can model rich sequential pipelines with guards.  

```scala
case class Config(host: String, port: Int)

def getHost: Option[String] = Some("localhost")
def getPort: Option[Int]    = Some(8080)
def validatePort(p: Int): Option[Int] =
  Option.when(p > 0 && p < 65536)(p)

@main def main() =
  val config =
    for
      host <- getHost
      rawPort <- getPort
      port <- validatePort(rawPort)
      if host.nonEmpty
    yield Config(host, port)

  println(config)

  val nums = List(1, 2, 3)
  val letters = List('a', 'b')
  val pairs =
    for
      n <- nums
      l <- letters
      if n % 2 != 0
    yield (n, l)

  println(pairs)
end main
```

The `if` guard inside a for-comprehension desugars to a `withFilter`  
call, filtering out elements that do not satisfy the predicate. When  
used with `Option`, a failed guard produces `None`. When used with  
`List`, it removes non-matching elements. Mixing `Option` and `List` in  
the same for-comprehension is not permitted because the desugar requires  
a single consistent monad; if you need to mix effects, monad transformers  
or explicit lifting is required. For-comprehensions are the idiomatic way  
to write sequential monadic code in Scala, making the data flow obvious.  

---

## Sequencing computations

Sequencing runs a list of monadic actions in order and collects the  
results, short-circuiting on the first failure.  

```scala
def sequence[A](opts: List[Option[A]]): Option[List[A]] =
  opts.foldRight(Option(List.empty[A])) { (oa, acc) =>
    for
      a  <- oa
      as <- acc
    yield a :: as
  }

def traverse[A, B](xs: List[A])(f: A => Option[B]): Option[List[B]] =
  sequence(xs.map(f))

@main def main() =
  println(sequence(List(Some(1), Some(2), Some(3))))
  println(sequence(List(Some(1), None, Some(3))))
  println(traverse(List("1", "2", "3"))(_.toIntOption))
  println(traverse(List("1", "x", "3"))(_.toIntOption))
end main
```

`sequence` converts `List[Option[A]]` into `Option[List[A]]`. It uses  
`foldRight` so that the resulting list preserves element order. When any  
element is `None`, the for-comprehension short-circuits and the entire  
result becomes `None`. `traverse` combines `map` and `sequence` in one  
pass, which is both more efficient and more expressive. These two  
combinators — `sequence` and `traverse` — appear throughout functional  
libraries and form the foundation of *Traversable* typeclasses. They  
demonstrate how monadic composition scales from single values to  
collections.  

---

## Error accumulation vs short-circuiting

`Either` short-circuits on the first error, while `Validated` (or a  
custom type) accumulates all errors before failing.  

```scala
case class Errors(msgs: List[String]):
  def ++(other: Errors): Errors = Errors(msgs ++ other.msgs)

type Valid[+A] = Either[Errors, A]

def validateName(s: String): Valid[String] =
  if s.nonEmpty then Right(s)
  else Left(Errors(List("Name is empty")))

def validateAge(n: Int): Valid[Int] =
  if n >= 0 && n <= 150 then Right(n)
  else Left(Errors(List(s"Age $n is out of range")))

def validateBoth(name: String, age: Int): Valid[(String, Int)] =
  (validateName(name), validateAge(age)) match
    case (Right(n), Right(a)) => Right((n, a))
    case (Left(e1), Left(e2)) => Left(e1 ++ e2)
    case (Left(e),  _)        => Left(e)
    case (_,        Left(e))  => Left(e)

@main def main() =
  println(validateBoth("Alice", 30))
  println(validateBoth("", -5))
  println(validateBoth("Bob", 200))
end main
```

When `validateBoth` is called with both an empty name and an invalid  
age, both `Left` values are combined into a single `Errors` holding all  
messages. This is *applicative error accumulation*: the two validations  
are independent, so neither needs to wait for the other. By contrast,  
a for-comprehension over `Either` would stop at the first `Left` and  
never evaluate the second validation. Choosing between accumulation and  
short-circuiting depends on user experience requirements: form validation  
typically accumulates errors so users see all problems at once, while  
pipeline processing typically short-circuits on the first unrecoverable  
failure.  

---

## OptionT

`OptionT` is a monad transformer that adds optional values to any  
underlying monad `F`, producing `F[Option[A]]` with monadic composition.  

```scala
import scala.concurrent.{Future, Await}
import scala.concurrent.duration.*
import scala.concurrent.ExecutionContext.Implicits.global

case class OptionT[F[_], A](value: F[Option[A]]):
  def map[B](f: A => B)(using fmap: [X, Y] => (X => Y) => F[X] => F[Y])
      : OptionT[F, B] =
    OptionT(fmap((_: Option[A]).map(f))(value))
  def flatMap[B](f: A => OptionT[F, B])
      (using M: [X, Y] => (X => F[Y]) => F[X] => F[Y])
      : OptionT[F, B] =
    OptionT(M((oa: Option[A]) => oa match
      case None    => summon[F[Option[B]] =:= F[Option[B]]].apply(
                        M((_: Unit) => summon[F[Option[B]] =:= F[Option[B]]].apply(
                          null.asInstanceOf[F[Option[B]]]))(null.asInstanceOf[F[Unit]]))
      case Some(a) => f(a).value)(value))

// Simpler hand-rolled OptionT for Future
case class FutureOption[A](run: Future[Option[A]]):
  def map[B](f: A => B): FutureOption[B] =
    FutureOption(run.map(_.map(f)))
  def flatMap[B](f: A => FutureOption[B]): FutureOption[B] =
    FutureOption(run.flatMap {
      case None    => Future.successful(None)
      case Some(a) => f(a).run
    })

object FutureOption:
  def fromFuture[A](fa: Future[A]): FutureOption[A] =
    FutureOption(fa.map(Some(_)))
  def none[A]: FutureOption[A] =
    FutureOption(Future.successful(None))

def findUser(id: Int): FutureOption[String] =
  if id > 0 then FutureOption(Future.successful(Some(s"User-$id")))
  else FutureOption.none

def findEmail(user: String): FutureOption[String] =
  FutureOption(Future.successful(Some(s"$user@example.com")))

@main def main() =
  val pipeline =
    for
      user  <- findUser(1)
      email <- findEmail(user)
    yield s"$user -> $email"

  println(Await.result(pipeline.run, 5.seconds))

  val missing =
    for
      user  <- findUser(-1)
      email <- findEmail(user)
    yield email

  println(Await.result(missing.run, 5.seconds))
end main
```

`FutureOption[A]` wraps `Future[Option[A]]` and provides `map` and  
`flatMap` that operate on the inner `Option` without unwrapping the  
`Future` manually. When `findUser(-1)` returns `FutureOption(Future(None))`,  
the `flatMap` implementation short-circuits by returning  
`Future.successful(None)` immediately, so `findEmail` is never called.  
This is the power of `OptionT`: the optionality propagates through the  
async layer transparently, and the for-comprehension reads as cleanly  
as if there were only one effect.  

---

## EitherT

`EitherT` combines typed error handling with any underlying monad,  
producing `F[Either[E, A]]` with seamless for-comprehension sequencing.  

```scala
import scala.concurrent.{Future, Await}
import scala.concurrent.duration.*
import scala.concurrent.ExecutionContext.Implicits.global

case class EitherT[F[_], E, A](value: F[Either[E, A]]):
  def map[B](f: A => B)(
      using Functor: F[Either[E, A]] => (Either[E, A] => Either[E, B]) => F[Either[E, B]]
  ): EitherT[F, E, B] =
    EitherT(Functor(value)(_.map(f)))

case class FutEither[E, A](run: Future[Either[E, A]]):
  def map[B](f: A => B): FutEither[E, B] =
    FutEither(run.map(_.map(f)))
  def flatMap[B](f: A => FutEither[E, B]): FutEither[E, B] =
    FutEither(run.flatMap {
      case Left(e)  => Future.successful(Left(e))
      case Right(a) => f(a).run
    })

object FutEither:
  def right[E, A](a: A): FutEither[E, A] =
    FutEither(Future.successful(Right(a)))
  def left[E, A](e: E): FutEither[E, A] =
    FutEither(Future.successful(Left(e)))
  def liftF[E, A](fa: Future[A]): FutEither[E, A] =
    FutEither(fa.map(Right(_)))

type Err = String

def authenticate(token: String): FutEither[Err, String] =
  if token == "secret" then FutEither.right("alice")
  else FutEither.left("Invalid token")

def fetchData(user: String): FutEither[Err, List[Int]] =
  FutEither.right(List(1, 2, 3))

@main def main() =
  val ok =
    for
      user <- authenticate("secret")
      data <- fetchData(user)
    yield s"$user: $data"

  println(Await.result(ok.run, 5.seconds))

  val bad =
    for
      user <- authenticate("wrong")
      data <- fetchData(user)
    yield s"$user: $data"

  println(Await.result(bad.run, 5.seconds))
end main
```

`FutEither[E, A]` is a manual implementation of `EitherT[Future, E, A]`.  
Its `flatMap` passes `Left` values through unchanged and only calls the  
continuation `f` when the result is `Right`. The `authenticate` function  
demonstrates the pattern: it rejects unknown tokens by returning  
`FutEither.left`, causing all downstream steps in the for-comprehension  
to be skipped. This transformer is invaluable for async services where  
every step can fail with a domain error and you want a single propagation  
path instead of nested pattern matches.  

---

## Stacking transformers

Transformers can be stacked to handle multiple simultaneous effects,  
though each layer adds syntactic overhead.  

```scala
import scala.concurrent.{Future, Await}
import scala.concurrent.duration.*
import scala.concurrent.ExecutionContext.Implicits.global

// Stack: Future[Either[String, Option[A]]]
case class AppEffect[A](run: Future[Either[String, Option[A]]]):
  def map[B](f: A => B): AppEffect[B] =
    AppEffect(run.map(_.map(_.map(f))))
  def flatMap[B](f: A => AppEffect[B]): AppEffect[B] =
    AppEffect(run.flatMap {
      case Left(e)       => Future.successful(Left(e))
      case Right(None)   => Future.successful(Right(None))
      case Right(Some(a)) => f(a).run
    })

object AppEffect:
  def pure[A](a: A): AppEffect[A] =
    AppEffect(Future.successful(Right(Some(a))))
  def fail[A](msg: String): AppEffect[A] =
    AppEffect(Future.successful(Left(msg)))
  def empty[A]: AppEffect[A] =
    AppEffect(Future.successful(Right(None)))

@main def main() =
  val program =
    for
      a <- AppEffect.pure(10)
      b <- AppEffect.pure(5)
      _ <- if b == 0 then AppEffect.fail("div by zero")
           else AppEffect.pure(())
      c <- AppEffect.pure(a / b)
    yield c

  println(Await.result(program.run, 5.seconds))

  val failing =
    for
      a <- AppEffect.pure(10)
      _ <- AppEffect.fail[Int]("something broke")
      b <- AppEffect.pure(20)
    yield a + b

  println(Await.result(failing.run, 5.seconds))
end main
```

`AppEffect` stacks three effects: `Future` for async execution,  
`Either[String, _]` for typed errors, and `Option[_]` for optional  
values. The `flatMap` implementation must handle all three layers: it  
propagates errors and empty values without running the continuation.  
While hand-rolling stacked transformers quickly becomes verbose, this  
approach makes the effect structure explicit and avoids runtime  
reflection. In practice, libraries like Cats provide generic `EitherT`  
and `OptionT` combinators that compose correctly for any monad, hiding  
this machinery behind a clean API.  

---

## Composing effects

Effect composition wires together independent effectful operations into  
a coherent, type-safe workflow.  

```scala
type Err = String
type Result[A] = Either[Err, A]

def parseAge(s: String): Result[Int] =
  s.toIntOption.toRight(s"Not a number: $s")

def validateAge(n: Int): Result[Int] =
  Either.cond(n >= 0 && n < 150, n, s"Age out of range: $n")

def greet(name: String, ageStr: String): Result[String] =
  for
    age <- parseAge(ageStr)
    v   <- validateAge(age)
  yield s"Hello $name, you are $v years old."

@main def main() =
  println(greet("Alice", "30"))
  println(greet("Bob", "abc"))
  println(greet("Carol", "-1"))
end main
```

Each small function has a focused, testable responsibility: parsing,  
validation, and presentation. The for-comprehension in `greet` wires  
them together; if any step returns a `Left`, the error propagates to  
the caller without any explicit error-handling code in `greet` itself.  
This is *effect composition by construction*: the `Either` monad  
provides the composition glue, and the type system ensures that a  
`Left` can never be silently ignored. Adding a new validation step  
requires only inserting one more `<-` line in the for-comprehension.  

---

## Transformer-based APIs

Transformer-based APIs expose a stack type as the return type of  
service methods, keeping effect management uniform across the codebase.  

```scala
import scala.concurrent.{Future, Await}
import scala.concurrent.duration.*
import scala.concurrent.ExecutionContext.Implicits.global

type AppErr = String

case class FE[A](run: Future[Either[AppErr, A]]):
  def map[B](f: A => B): FE[B] = FE(run.map(_.map(f)))
  def flatMap[B](f: A => FE[B]): FE[B] =
    FE(run.flatMap {
      case Left(e)  => Future.successful(Left(e))
      case Right(a) => f(a).run
    })

object FE:
  def ok[A](a: A): FE[A]      = FE(Future.successful(Right(a)))
  def err[A](e: AppErr): FE[A] = FE(Future.successful(Left(e)))

trait UserService:
  def find(id: Int): FE[String]
  def activate(user: String): FE[Boolean]

object MockUserService extends UserService:
  def find(id: Int): FE[String] =
    if id > 0 then FE.ok(s"User$id") else FE.err("Not found")
  def activate(user: String): FE[Boolean] =
    FE.ok(true)

@main def main() =
  val svc: UserService = MockUserService

  val result =
    for
      user    <- svc.find(1)
      active  <- svc.activate(user)
    yield s"$user active=$active"

  println(Await.result(result.run, 5.seconds))

  val missing =
    for
      user   <- svc.find(-1)
      active <- svc.activate(user)
    yield active

  println(Await.result(missing.run, 5.seconds))
end main
```

Defining the service trait in terms of `FE[A]` means all callers  
understand that operations are asynchronous and may fail with `AppErr`.  
The service implementation (`MockUserService`) only concerns itself with  
domain logic; the transformer handles async and error propagation.  
Swapping the implementation for a real one backed by a database or HTTP  
client requires no changes to the call sites. This is a direct  
application of the Liskov Substitution Principle enabled by functional  
abstractions.  

---

## Transformer-based domain logic

Domain logic written against a transformer stack stays pure and  
testable while supporting real-world effects.  

```scala
type ErrMsg = String

case class IO[A](unsafeRun: () => Either[ErrMsg, A]):
  def map[B](f: A => B): IO[B] =
    IO(() => unsafeRun().map(f))
  def flatMap[B](f: A => IO[B]): IO[B] =
    IO(() => unsafeRun() match
      case Left(e)  => Left(e)
      case Right(a) => f(a).unsafeRun())

object IO:
  def ok[A](a: A): IO[A]       = IO(() => Right(a))
  def fail[A](e: ErrMsg): IO[A] = IO(() => Left(e))
  def delay[A](a: => A): IO[A]  = IO(() => Right(a))

def loadConfig: IO[Map[String, String]] =
  IO.ok(Map("db" -> "postgres://localhost/mydb", "port" -> "5432"))

def getKey(cfg: Map[String, String], key: String): IO[String] =
  cfg.get(key).fold(IO.fail(s"Missing key: $key"))(IO.ok)

def parsePort(s: String): IO[Int] =
  s.toIntOption.fold(IO.fail(s"Bad port: $s"))(IO.ok)

@main def main() =
  val program =
    for
      cfg  <- loadConfig
      db   <- getKey(cfg, "db")
      pStr <- getKey(cfg, "port")
      port <- parsePort(pStr)
    yield s"Connect $db on port $port"

  println(program.unsafeRun())

  val bad =
    for
      cfg  <- loadConfig
      key  <- getKey(cfg, "missing")
    yield key

  println(bad.unsafeRun())
end main
```

This `IO` type is a simplified synchronous effect monad that pairs an  
either-based error channel with a deferred computation. Nothing is  
executed until `unsafeRun` is called, preserving referential  
transparency throughout the construction of the pipeline. The `for`-  
comprehension in `program` reads like an imperative sequence but is  
fully pure: it is just a description of what to do. Replacing the in-  
memory map with file-system or environment-variable reads would require  
only changing the `loadConfig` function.  

---

## Defining an algebra

In the tagless-final style, an algebra is a trait parameterised on  
an effect type `F[_]` that describes the operations of a component.  

```scala
trait Console[F[_]]:
  def readLine: F[String]
  def writeLine(s: String): F[Unit]

trait Clock[F[_]]:
  def currentTimeMs: F[Long]

// identity-like wrapper for pure computation
type Id[A] = A

given Console[Id] with
  def readLine: Id[String]             = "Alice"
  def writeLine(s: String): Id[Unit]   = println(s)

given Clock[Id] with
  def currentTimeMs: Id[Long] = System.currentTimeMillis()

def greet[F[_]](using C: Console[F], K: Clock[F]): F[Unit] =
  val name = C.readLine
  val t    = K.currentTimeMs
  C.writeLine(s"Hello $name, time=$t")

@main def main() =
  greet[Id]
end main
```

The algebra traits `Console` and `Clock` describe *what* operations are  
available without specifying *how* they are executed. The effect type  
`F[_]` is kept abstract: the caller chooses a concrete `F` (here `Id`)  
and supplies the corresponding `given` instances. The `greet` function  
requires both algebras simultaneously via two `using` clauses and  
combines them without knowing anything about the underlying effect. This  
separation of description from execution is the core idea of tagless  
final.  

---

## Defining interpreters

An interpreter is a `given` instance of an algebra trait that  
provides a concrete implementation for a specific effect type.  

```scala
trait KVStore[F[_]]:
  def put(key: String, value: String): F[Unit]
  def get(key: String): F[Option[String]]
  def delete(key: String): F[Unit]

import scala.collection.mutable

type Id[A] = A

given KVStore[Id] with
  private val store = mutable.Map.empty[String, String]
  def put(key: String, value: String): Id[Unit] =
    store(key) = value
  def get(key: String): Id[Option[String]] =
    store.get(key)
  def delete(key: String): Id[Unit] =
    store -= key

def program[F[_]](using kv: KVStore[F]): F[Unit] =
  kv.put("lang", "Scala")
  kv.put("version", "3")
  val v = kv.get("lang")
  println(s"lang = $v")
  kv.delete("lang")
  println(s"after delete: ${kv.get("lang")}")

@main def main() =
  program[Id]
end main
```

The `KVStore[Id]` given instance backs the store with a mutable map,  
suitable for tests or simple in-process use. The `program` function is  
written purely in terms of the `KVStore[F]` algebra; it never mentions  
`mutable.Map` or any other concrete type. Swapping to a Redis-backed  
interpreter would require only providing a `given KVStore[Future]` with  
a different concrete body — the program logic itself is untouched. This  
is the key benefit of separating algebra definition from interpretation.  

---

## Pure interpreters

A pure interpreter uses an immutable data structure as its effect,  
making the interpreter fully referentially transparent.  

```scala
case class State[S, +A](run: S => (S, A)):
  def map[B](f: A => B): State[S, B] =
    State(s => val (s2, a) = run(s); (s2, f(a)))
  def flatMap[B](f: A => State[S, B]): State[S, B] =
    State(s => val (s2, a) = run(s); f(a).run(s2))

object State:
  def pure[S, A](a: A): State[S, A] = State(s => (s, a))
  def get[S]: State[S, S]            = State(s => (s, s))
  def set[S](s: S): State[S, Unit]   = State(_ => (s, ()))
  def modify[S](f: S => S): State[S, Unit] =
    State(s => (f(s), ()))

type Counter[A] = State[Int, A]

def increment: Counter[Unit]    = State.modify(_ + 1)
def decrement: Counter[Unit]    = State.modify(_ - 1)
def getCount: Counter[Int]      = State.get

@main def main() =
  val program: Counter[Int] =
    for
      _ <- increment
      _ <- increment
      _ <- increment
      _ <- decrement
      n <- getCount
    yield n

  val (finalState, result) = program.run(0)
  println(s"result=$result, state=$finalState")
end main
```

`State[S, A]` encodes a computation that reads and transforms a state  
value of type `S` and produces a result `A`, all without any mutable  
state. The `run` function is just a pure function from an initial state  
to a pair of the updated state and the result. Sequencing `State` values  
with `flatMap` threads the state through each step automatically. This  
is a pure interpreter of the counter algebra: it models stateful  
computation without ever modifying a variable.  

---

## Effectful interpreters

An effectful interpreter uses `Future`, `IO`, or another effect monad  
to perform real side effects like I/O or network calls.  

```scala
import scala.concurrent.{Future, Await}
import scala.concurrent.duration.*
import scala.concurrent.ExecutionContext.Implicits.global

trait Logger[F[_]]:
  def info(msg: String): F[Unit]
  def error(msg: String): F[Unit]

given Logger[Future] with
  def info(msg: String): Future[Unit] =
    Future(println(s"[INFO]  $msg"))
  def error(msg: String): Future[Unit] =
    Future(println(s"[ERROR] $msg"))

def processRequest[F[_]](
    id: Int)(using L: Logger[F], ev: F[Unit] =:= F[Unit]): F[Unit] =
  if id > 0 then L.info(s"Processing id=$id")
  else L.error(s"Invalid id=$id")

// simplified: runs with Future directly
def run(id: Int)(using L: Logger[Future]): Future[Unit] =
  processRequest[Future](id)

@main def main() =
  Await.result(run(42), 5.seconds)
  Await.result(run(-1), 5.seconds)
end main
```

The `Logger[Future]` given instance executes each log statement inside  
a `Future`, simulating asynchronous I/O. Because `processRequest` is  
written against the `Logger[F]` algebra, it can also be tested with a  
pure `Logger[Id]` that accumulates log messages into a `List` instead of  
printing them, enabling deterministic unit tests without mocking  
frameworks. Effectful interpreters make the production side-effects  
explicit in the type signature, while pure interpreters keep tests fast  
and reproducible.  

---

## Dependency injection via givens

Scala 3's `given`/`using` mechanism provides a lightweight, type-safe  
form of dependency injection without frameworks or runtime reflection.  

```scala
trait EmailSender[F[_]]:
  def send(to: String, body: String): F[Boolean]

trait UserRepo[F[_]]:
  def findEmail(userId: Int): F[Option[String]]

type Id[A] = A

given EmailSender[Id] with
  def send(to: String, body: String): Id[Boolean] =
    println(s"Sending to $to: $body"); true

given UserRepo[Id] with
  def findEmail(userId: Int): Id[Option[String]] =
    if userId == 1 then Some("alice@example.com") else None

def notifyUser[F[_]](userId: Int, msg: String)(using
    R: UserRepo[F],
    E: EmailSender[F],
    fmap: Option[String] => (String => F[Boolean]) => F[Option[Boolean]]
): Unit =
  R.findEmail(userId) match
    case Some(email) => E.send(email, msg); ()
    case None        => println(s"User $userId not found")

// direct version using Id
def notify(userId: Int, msg: String)(using
    R: UserRepo[Id],
    E: EmailSender[Id]): Unit =
  R.findEmail(userId) match
    case Some(email) => E.send(email, msg); ()
    case None        => println(s"User $userId not found")

@main def main() =
  notify(1, "Hello Alice!")
  notify(2, "Hello Bob!")
end main
```

The `notify` function declares its two dependencies (`UserRepo[Id]` and  
`EmailSender[Id]`) as `using` parameters. The compiler supplies both  
from the ambient `given` scope at the call site — no DI container, no  
annotations, no XML. Replacing the production instances with test  
doubles is as simple as providing different `given` instances in the  
test scope, which takes precedence over the outer scope due to Scala 3's  
prioritised implicit resolution rules.  

---

## Composing algebras

Multiple algebras can be combined into a single capability constraint  
using intersection types or compound `using` clauses.  

```scala
trait Reader[F[_]]:
  def read(path: String): F[String]

trait Writer[F[_]]:
  def write(path: String, content: String): F[Unit]

type Id[A] = A

given Reader[Id] with
  def read(path: String): Id[String] = s"content of $path"

given Writer[Id] with
  def write(path: String, content: String): Id[Unit] =
    println(s"Writing to $path: $content")

def copyFile[F[_]](src: String, dst: String)(using
    R: Reader[F], W: Writer[F]): F[Unit] =
  W.write(dst, R.read(src))

def processFiles[F[_]](files: List[(String, String)])(using
    R: Reader[F], W: Writer[F]): Unit =
  files.foreach((src, dst) => copyFile[F](src, dst))

@main def main() =
  copyFile[Id]("a.txt", "b.txt")
  processFiles[Id](List(
    "x.txt" -> "y.txt",
    "m.txt" -> "n.txt"))
end main
```

`copyFile` composes `Reader` and `Writer` by requiring both as `using`  
parameters. `processFiles` can use the same combined capability across  
a list of file pairs. Because the effect type `F` is shared, both  
algebras operate in the same context — here `Id`, but it could equally  
be `Future` or a custom IO type. Composing algebras this way produces  
fine-grained capability constraints in function signatures, making the  
dependencies of each function explicit and auditable.  

---

## Layering interpreters

Interpreters can be layered so that one interpreter wraps another,  
adding cross-cutting behaviour like logging or caching.  

```scala
trait Compute[F[_]]:
  def run(input: Int): F[Int]

type Id[A] = A

val baseCompute: Compute[Id] = new Compute[Id]:
  def run(input: Int): Id[Int] = input * input

def loggingCompute(inner: Compute[Id]): Compute[Id] =
  new Compute[Id]:
    def run(input: Int): Id[Int] =
      println(s"[LOG] input=$input")
      val result = inner.run(input)
      println(s"[LOG] result=$result")
      result

def cachingCompute(inner: Compute[Id]): Compute[Id] =
  val cache = scala.collection.mutable.Map.empty[Int, Int]
  new Compute[Id]:
    def run(input: Int): Id[Int] =
      cache.getOrElseUpdate(input, inner.run(input))

@main def main() =
  val compute = cachingCompute(loggingCompute(baseCompute))
  println(compute.run(4))
  println(compute.run(4))
  println(compute.run(5))
end main
```

Each layer wraps the previous interpreter and adds behaviour before or  
after delegating to the inner `run`. The caching layer calls the inner  
interpreter only on a cache miss, while the logging layer records every  
call. Because both layers work with the same `Compute[Id]` interface,  
they can be composed in any order. This is the *decorator pattern*  
expressed functionally: pure interpreter layering with no inheritance,  
no reflection, and full testability at each layer independently.  

---

## Pure business logic

Pure business logic functions take plain values, return plain values,  
and contain no side effects — making them trivially testable.  

```scala
case class Order(id: Int, items: List[String], discount: Double)
case class Invoice(orderId: Int, total: Double, lines: List[String])

def lineTotal(item: String): Double = item.length * 1.5

def applyDiscount(amount: Double, discount: Double): Double =
  amount * (1.0 - discount)

def buildInvoice(order: Order): Invoice =
  val subtotal = order.items.map(lineTotal).sum
  val total    = applyDiscount(subtotal, order.discount)
  val lines    = order.items.map(i => s"$i: ${lineTotal(i)}")
  Invoice(order.id, total, lines)

@main def main() =
  val order = Order(1, List("Scala book", "FP guide"), 0.1)
  val inv   = buildInvoice(order)
  println(s"Invoice #${inv.orderId}")
  inv.lines.foreach(l => println(s"  $l"))
  println(f"Total: ${inv.total}%.2f")
end main
```

`buildInvoice` is a pure function: given the same `Order`, it always  
produces the same `Invoice`, with no database calls, no logging, and no  
exceptions. This makes it trivial to unit-test with ordinary values and  
assertions. Side effects (persistence, notifications) are pushed to the  
edges of the application; the core domain stays clean. This layered  
approach — pure core, effectful shell — is sometimes called the  
*functional core, imperative shell* pattern and is one of the most  
practical recommendations in functional design.  

---

## Functional services

A functional service is an interface expressed as a trait with  
typeclass-style given instances, making it easy to substitute in tests.  

```scala
case class Product(id: Int, name: String, price: Double)

trait ProductService[F[_]]:
  def findById(id: Int): F[Option[Product]]
  def findAll: F[List[Product]]

type Id[A] = A

given ProductService[Id] with
  private val db = Map(
    1 -> Product(1, "Keyboard", 79.99),
    2 -> Product(2, "Mouse",    29.99))
  def findById(id: Int): Id[Option[Product]] = db.get(id)
  def findAll: Id[List[Product]]             = db.values.toList

def summarise[F[_]]()(using S: ProductService[F],
    flat: F[List[Product]] => (List[Product] => F[String]) => F[String]
): F[String] =
  flat(S.findAll)(ps =>
    summon[F[String] =:= F[String]].apply(
      ps.map(p => f"${p.name}: $$${p.price}%.2f").mkString("\n")
        .asInstanceOf[F[String]]))

def summariseId()(using S: ProductService[Id]): String =
  S.findAll.map(p => f"${p.name}: $$${p.price}%.2f").mkString("\n")

@main def main() =
  println(summariseId())
  println("---")
  println(summon[ProductService[Id]].findById(1))
  println(summon[ProductService[Id]].findById(99))
end main
```

`ProductService[Id]` is a pure in-memory implementation suitable for  
tests and prototyping. The `given` instance is self-contained and  
trivially replaceable: for integration tests, a `given ProductService[Future]`  
backed by a real database would be provided instead, without changing  
any call sites. Encoding services as typeclasses rather than concrete  
classes keeps the dependency graph explicit in types and the overall  
architecture more modular.  

---

## Functional modules

A functional module groups related algebras and their given instances  
into a coherent unit that can be imported as a whole.  

```scala
object AuthModule:
  trait TokenValidator[F[_]]:
    def validate(token: String): F[Boolean]
  trait SessionStore[F[_]]:
    def create(userId: Int): F[String]
    def lookup(session: String): F[Option[Int]]

  type Id[A] = A

  given TokenValidator[Id] with
    def validate(token: String): Id[Boolean] =
      token.startsWith("Bearer ")

  given SessionStore[Id] with
    private val store = scala.collection.mutable.Map.empty[String, Int]
    def create(userId: Int): Id[String] =
      val s = s"sess-$userId-${System.nanoTime()}"
      store(s) = userId; s
    def lookup(session: String): Id[Option[Int]] =
      store.get(session)

import AuthModule.*

def login[F[_]](token: String, userId: Int)(using
    V: TokenValidator[F],
    S: SessionStore[F]): Unit =
  if V.validate(token) then
    val sess = S.create(userId)
    println(s"Session: $sess")
  else
    println("Auth failed")

@main def main() =
  login[AuthModule.Id]("Bearer tok123", 42)
  login[AuthModule.Id]("bad", 1)
end main
```

The `AuthModule` object acts as a self-contained module: it defines  
algebras, their `Id`-typed given instances, and a type alias for  
convenience. Importing `AuthModule.*` brings all the givens and types  
into scope. Organising code this way mirrors the *ML module* concept:  
each module encapsulates a sub-system with a clear interface. Modules  
can depend on other modules by requiring additional `using` parameters,  
and the compiler verifies that all required instances are present at  
compile time.  

---

## Functional dependency injection

Functional DI passes all dependencies as `using` parameters,  
making the dependency graph visible in type signatures.  

```scala
trait Cache[F[_]]:
  def set(k: String, v: String): F[Unit]
  def get(k: String): F[Option[String]]

type Id[A] = A

given Cache[Id] with
  private val m = scala.collection.mutable.Map.empty[String, String]
  def set(k: String, v: String): Id[Unit] = m(k) = v
  def get(k: String): Id[Option[String]]  = m.get(k)

def cachedLookup[F[_]](key: String, compute: => String)(using
    C: Cache[F],
    pure: String => F[String],
    flatMap: F[Option[String]] => (Option[String] => F[String]) => F[String]
): F[String] =
  flatMap(C.get(key)) {
    case Some(v) => pure(v)
    case None =>
      val v = compute
      C.set(key, v)
      pure(v)
  }

def cachedLookupId(key: String, compute: => String)(using
    C: Cache[Id]): Id[String] =
  C.get(key) match
    case Some(v) => v
    case None    => val v = compute; C.set(key, v); v

@main def main() =
  println(cachedLookupId("pi", { println("computing"); "3.14" }))
  println(cachedLookupId("pi", { println("computing"); "3.14" }))
  println(cachedLookupId("e",  { println("computing"); "2.71" }))
end main
```

`cachedLookupId` depends on `Cache[Id]` injected via `using`, so the  
caller never constructs the cache directly. In a test, a different  
`given Cache[Id]` that always misses or always hits can be provided.  
In production, a `given Cache[Future]` backed by Redis is dropped in.  
The `"computing"` print confirms that the cache is hit on the second  
call for `"pi"`. Functional DI scales well: adding a new dependency to  
a function is as simple as adding a `using` clause; the compiler then  
points out every call site that needs to supply the new instance.  

---

## Functional pipelines

A functional pipeline is a sequence of pure transformations chained  
with `andThen` or `map`, making data flow explicit and composable.  

```scala
case class RawEvent(payload: String)
case class ParsedEvent(value: Int)
case class EnrichedEvent(value: Int, label: String)
case class OutputEvent(summary: String)

val parse: RawEvent => Option[ParsedEvent] =
  e => e.payload.toIntOption.map(ParsedEvent.apply)

val enrich: ParsedEvent => EnrichedEvent =
  e => EnrichedEvent(e.value, if e.value > 0 then "positive" else "non-positive")

val format: EnrichedEvent => OutputEvent =
  e => OutputEvent(s"${e.label}: ${e.value}")

def pipeline(raw: RawEvent): Option[OutputEvent] =
  parse(raw).map(enrich).map(format)

@main def main() =
  List(RawEvent("42"), RawEvent("-7"), RawEvent("abc"))
    .map(pipeline)
    .foreach(println)
end main
```

Each stage of the pipeline is a standalone, testable function. The  
`Option` wrapping propagates the parse failure automatically: if  
`parse` returns `None`, the `enrich` and `format` stages are skipped.  
Composing the pipeline with `map` is possible because the output type  
of each stage matches the input type of the next. Adding a new stage  
(e.g., a validation step between `parse` and `enrich`) requires only  
inserting one more `.map(validate)` call. The entire pipeline is a  
value and can itself be passed as a function argument.  

---

## Functional builders

Functional builders use type-safe accumulation to construct complex  
objects without mutable state or runtime exceptions.  

```scala
case class ServerConfig(
    host: String,
    port: Int,
    maxConn: Int,
    timeout: Int)

case class Builder(
    host: Option[String]    = None,
    port: Option[Int]       = None,
    maxConn: Option[Int]    = None,
    timeout: Option[Int]    = None):

  def withHost(h: String): Builder    = copy(host = Some(h))
  def withPort(p: Int): Builder        = copy(port = Some(p))
  def withMaxConn(n: Int): Builder     = copy(maxConn = Some(n))
  def withTimeout(t: Int): Builder     = copy(timeout = Some(t))

  def build: Either[String, ServerConfig] =
    for
      h <- host.toRight("host is required")
      p <- port.toRight("port is required")
      m <- maxConn.toRight("maxConn is required")
      t <- timeout.toRight("timeout is required")
    yield ServerConfig(h, p, m, t)

@main def main() =
  val result =
    Builder()
      .withHost("localhost")
      .withPort(8080)
      .withMaxConn(100)
      .withTimeout(30)
      .build
  println(result)

  val incomplete =
    Builder()
      .withHost("localhost")
      .build
  println(incomplete)
end main
```

The builder is an immutable value: each `with*` method returns a new  
`Builder` with one more field populated. The `build` method uses a  
for-comprehension over `Option` converted to `Either` to collect all  
missing fields into a descriptive error message. No `null`, no  
`NullPointerException`, and no partial objects are possible. The  
functional builder pattern is particularly useful for constructing  
configuration objects that need to be validated before use.  

---

## Functional state machines

A functional state machine encodes states and transitions as ADTs  
and pure functions, making illegal transitions unrepresentable.  

```scala
enum OrderState:
  case Pending
  case Paid
  case Shipped
  case Delivered
  case Cancelled

enum OrderEvent:
  case Pay
  case Ship
  case Deliver
  case Cancel

def transition(
    state: OrderState,
    event: OrderEvent): Either[String, OrderState] =
  import OrderState.*, OrderEvent.*
  (state, event) match
    case (Pending,   Pay)     => Right(Paid)
    case (Pending,   Cancel)  => Right(Cancelled)
    case (Paid,      Ship)    => Right(Shipped)
    case (Paid,      Cancel)  => Right(Cancelled)
    case (Shipped,   Deliver) => Right(Delivered)
    case _                    =>
      Left(s"Invalid: $event in state $state")

@main def main() =
  val steps = List(
    OrderEvent.Pay, OrderEvent.Ship, OrderEvent.Deliver)

  val finalState = steps.foldLeft(Right(OrderState.Pending): Either[String, OrderState]) {
    (st, ev) => st.flatMap(transition(_, ev))
  }
  println(finalState)

  println(transition(OrderState.Shipped, OrderEvent.Cancel))
end main
```

The `transition` function is a total, pure function: every (state,  
event) combination is handled, either producing a `Right` with the new  
state or a `Left` with an error message. The `foldLeft` over the event  
sequence propagates the first error automatically, so invalid event  
sequences are caught without defensive checks scattered throughout the  
code. Encoding the state machine as an ADT also enables exhaustiveness  
checking: the compiler will warn if a new `OrderState` or `OrderEvent`  
variant is added but not handled in `transition`.  

---

## Functional configuration loading

Configuration loading can be made purely functional by combining  
parsers with the `Either` monad for validation.  

```scala
type Cfg = Map[String, String]

def require(cfg: Cfg, key: String): Either[String, String] =
  cfg.get(key).toRight(s"Missing config key: '$key'")

def requireInt(cfg: Cfg, key: String): Either[String, Int] =
  require(cfg, key).flatMap(v =>
    v.toIntOption.toRight(s"Config '$key' is not an integer: '$v'"))

case class DbConfig(host: String, port: Int, name: String)
case class AppConfig(db: DbConfig, logLevel: String)

def loadDb(cfg: Cfg): Either[String, DbConfig] =
  for
    host <- require(cfg, "db.host")
    port <- requireInt(cfg, "db.port")
    name <- require(cfg, "db.name")
  yield DbConfig(host, port, name)

def loadApp(cfg: Cfg): Either[String, AppConfig] =
  for
    db  <- loadDb(cfg)
    lvl <- require(cfg, "log.level")
  yield AppConfig(db, lvl)

@main def main() =
  val good: Cfg = Map(
    "db.host"   -> "localhost",
    "db.port"   -> "5432",
    "db.name"   -> "mydb",
    "log.level" -> "INFO")

  println(loadApp(good))

  val bad: Cfg = Map("db.host" -> "localhost")
  println(loadApp(bad))
end main
```

The `require` and `requireInt` helpers build a small, composable  
vocabulary for configuration parsing. Each returns `Either[String, A]`,  
so failures are accumulated naturally through the for-comprehension:  
the first missing or malformed key stops processing and reports a  
precise error. `loadDb` and `loadApp` compose these helpers into  
higher-level parsers without duplicating any validation logic. This  
approach makes the configuration schema self-documenting (it is  
expressed in code) and guarantees that `AppConfig` is always valid  
when obtained through `loadApp`.  

---

## Modeling domain errors with ADTs

Algebraic data types make every possible error an explicit, named  
value that the compiler requires callers to handle.  

```scala
enum PaymentError:
  case InsufficientFunds(required: Double, available: Double)
  case CardExpired(expiry: String)
  case NetworkTimeout(attempt: Int)
  case Declined(reason: String)

def charge(
    amount: Double,
    balance: Double,
    expired: Boolean): Either[PaymentError, Double] =
  if expired then
    Left(PaymentError.CardExpired("2023-01"))
  else if balance < amount then
    Left(PaymentError.InsufficientFunds(amount, balance))
  else
    Right(balance - amount)

def describe(err: PaymentError): String =
  err match
    case PaymentError.InsufficientFunds(r, a) =>
      s"Need $$${r}, have $$${a}"
    case PaymentError.CardExpired(exp) =>
      s"Card expired $exp"
    case PaymentError.NetworkTimeout(n) =>
      s"Timeout on attempt $n"
    case PaymentError.Declined(r) =>
      s"Declined: $r"

@main def main() =
  println(charge(50.0, 100.0, false))
  println(charge(150.0, 100.0, false).left.map(describe))
  println(charge(50.0, 100.0, true).left.map(describe))
end main
```

Every branch of `PaymentError` carries structured data relevant to that  
failure. The `describe` function performs an exhaustive pattern match:  
if a new variant were added to `PaymentError`, the compiler would emit  
a warning at every non-exhaustive match site, guiding developers to  
handle the new case. Using `Either[PaymentError, Double]` instead of  
throwing exceptions means error handling is visible in the type signature  
and cannot be accidentally swallowed by an unrelated catch clause.  

---

## Modeling domain values with opaque types

Opaque types wrap primitive values to prevent mixing semantically  
different quantities that share the same underlying representation.  

```scala
object Domain:
  opaque type UserId  = Int
  opaque type OrderId = Int
  opaque type Email   = String

  object UserId:
    def apply(n: Int): UserId = n
    extension (u: UserId) def value: Int = u

  object OrderId:
    def apply(n: Int): OrderId = n
    extension (o: OrderId) def value: Int = o

  object Email:
    def parse(s: String): Option[Email] =
      Option.when(s.contains("@"))(s)
    extension (e: Email) def value: String = e

import Domain.*

def findUser(id: UserId): String = s"User(${id.value})"

@main def main() =
  val uid = UserId(1)
  val oid = OrderId(1)
  // findUser(oid)  // compile error: OrderId is not UserId
  println(findUser(uid))
  println(Email.parse("alice@example.com"))
  println(Email.parse("notanemail"))
end main
```

Outside the `Domain` object, `UserId` and `OrderId` are distinct types  
even though both are `Int` at runtime. Passing an `OrderId` where a  
`UserId` is expected is a compile-time type error, not a silent logic  
bug. Opaque types are zero-cost: the compiler erases the wrapper  
entirely in generated bytecode. The extension methods inside each  
companion object provide the only sanctioned way to access the  
underlying value, preserving the abstraction boundary.  

---

## Modeling workflows with monads

A domain workflow is modelled as a monadic pipeline where each step  
can fail with a domain-specific error.  

```scala
enum WorkflowError:
  case NotFound(id: Int)
  case AlreadyProcessed(id: Int)
  case ValidationFailed(msg: String)

case class Order(id: Int, processed: Boolean, amount: Double)

def findOrder(id: Int): Either[WorkflowError, Order] =
  if id == 42 then Right(Order(42, false, 99.99))
  else Left(WorkflowError.NotFound(id))

def checkNotProcessed(o: Order): Either[WorkflowError, Order] =
  if o.processed then Left(WorkflowError.AlreadyProcessed(o.id))
  else Right(o)

def validateAmount(o: Order): Either[WorkflowError, Order] =
  if o.amount > 0 then Right(o)
  else Left(WorkflowError.ValidationFailed("amount must be positive"))

def processOrder(id: Int): Either[WorkflowError, String] =
  for
    order     <- findOrder(id)
    unchecked <- checkNotProcessed(order)
    validated <- validateAmount(unchecked)
  yield s"Processed order ${validated.id} for $$${validated.amount}"

@main def main() =
  println(processOrder(42))
  println(processOrder(99))
end main
```

The `processOrder` function reads as a sequence of business steps.  
Each step either advances the workflow (`Right`) or terminates it with  
a precise `WorkflowError` (`Left`). No exception is thrown; no  
null-check is needed. Adding a new step — say, a fraud check — is a  
single line in the for-comprehension. This pattern makes the workflow  
logic the primary focus of the code, with error handling as a  
transparent structural concern rather than cross-cutting noise.  

---

## Modeling capabilities with typeclasses

Typeclasses model fine-grained capabilities that a computation may  
require, making dependencies explicit and composable in type signatures.  

```scala
trait CanRead[F[_]]:
  def read(key: String): F[Option[String]]

trait CanWrite[F[_]]:
  def write(key: String, value: String): F[Unit]

type Id[A] = A

given CanRead[Id] with
  private val store = Map("greeting" -> "hello", "name" -> "Scala")
  def read(key: String): Id[Option[String]] = store.get(key)

given CanWrite[Id] with
  def write(key: String, value: String): Id[Unit] =
    println(s"write($key, $value)")

def readAndLog[F[_]](key: String)(using
    R: CanRead[F],
    W: CanWrite[F]): Unit =
  val v = R.read(key)
  W.write("log", s"read $key => $v")
  println(s"value: $v")

def readOnly[F[_]](key: String)(using R: CanRead[F]): Unit =
  println(s"$key => ${R.read(key)}")

@main def main() =
  readOnly[Id]("greeting")
  readAndLog[Id]("name")
  readAndLog[Id]("missing")
end main
```

`readOnly` requires only `CanRead`, while `readAndLog` requires both  
`CanRead` and `CanWrite`. The type signatures communicate precisely  
what each function can do — a function with only `CanRead` in scope  
cannot accidentally write, because the compiler will reject any attempt  
to call `write`. This is the *capability pattern*: capabilities are  
granted by the presence of typeclass instances in the calling context,  
not by inheritance or role-based access control at runtime.  

---

## Modeling interpreters for domain logic

Domain interpreters translate a description of operations into  
concrete effects, decoupling the what from the how.  

```scala
enum Notification:
  case Email(to: String, subject: String, body: String)
  case SMS(to: String, message: String)
  case Push(deviceId: String, message: String)

trait NotificationInterpreter[F[_]]:
  def send(n: Notification): F[Boolean]

type Id[A] = A

given NotificationInterpreter[Id] with
  def send(n: Notification): Id[Boolean] =
    n match
      case Notification.Email(to, sub, body) =>
        println(s"EMAIL to=$to sub=$sub"); true
      case Notification.SMS(to, msg) =>
        println(s"SMS to=$to msg=$msg"); true
      case Notification.Push(did, msg) =>
        println(s"PUSH device=$did msg=$msg"); true

def notifyAll[F[_]](
    ns: List[Notification])(using I: NotificationInterpreter[F]): Unit =
  ns.foreach(n => I.send(n))

@main def main() =
  val notifications = List(
    Notification.Email("alice@x.com", "Hi", "Hello Alice"),
    Notification.SMS("+1234", "Your code: 9999"),
    Notification.Push("device-abc", "New message"))

  notifyAll[Id](notifications)
end main
```

`Notification` is a pure ADT describing what should happen; it carries  
no side effects. The `NotificationInterpreter[Id]` given instance is  
the only place where I/O occurs. For tests, a logging interpreter that  
accumulates `Notification` values into a `List` can be substituted,  
allowing assertions like "exactly one SMS was sent to +1234" without  
sending real messages. This pattern — a pure description language  
paired with swappable interpreters — is the foundation of the *free  
monad* and *tagless-final* approaches to effect management.  

---

## Using match types in FP

Match types compute result types from input types at compile time,  
enabling type-safe generic utilities that adapt to structure.  

```scala
type Unwrap[F[_], A] = F[A] match
  case Option[x]   => x
  case List[x]     => x
  case Either[e, x] => x

type Flattened[A] = A match
  case List[List[x]] => List[x]
  case List[x]       => List[x]

def flatten[A](xs: List[List[A]]): List[A] = xs.flatten

def headOption[A](xs: List[A]): Option[A] = xs.headOption

@main def main() =
  val nested: List[List[Int]] = List(List(1, 2), List(3, 4))
  val flat: List[Int] = flatten(nested)
  println(flat)

  val xs = List(10, 20, 30)
  println(headOption(xs))

  // Demonstrating match type alias in a generic context
  def unwrapOption[A](oa: Option[A]): Option[A] = oa
  println(unwrapOption(Some(42)))
  println(unwrapOption(None))
end main
```

`Unwrap` is a type-level function: given `Option[Int]` it returns  
`Int`; given `Either[String, Boolean]` it returns `Boolean`. This  
allows generic functions to express precise type relationships without  
phantom types or complex implicits. `Flattened` shows a match type  
that normalises nested list structures at the type level. Match types  
compose with `given` instances to produce libraries where the return  
type adapts to the input, caught at compile time rather than  
discovered at runtime.  

---

## Using union types in FP

Union types express that a value may be one of several unrelated types,  
enabling concise, type-safe alternatives to sealed hierarchies.  

```scala
type StringOrInt = String | Int
type Primitive   = String | Int | Boolean | Double

def describe(v: Primitive): String =
  v match
    case s: String  => s"string: $s"
    case n: Int     => s"int: $n"
    case b: Boolean => s"bool: $b"
    case d: Double  => s"double: $d"

def normalise(v: StringOrInt): String =
  v match
    case s: String => s.trim.toLowerCase
    case n: Int    => n.toString

def processAll(vs: List[Primitive]): List[String] =
  vs.map(describe)

@main def main() =
  val values: List[Primitive] =
    List("Hello", 42, true, 3.14, "world", -1)
  processAll(values).foreach(println)
  println(normalise("  SCALA  "))
  println(normalise(99))
end main
```

Union types in Scala 3 are first-class: `String | Int` is a genuine  
type, not just a documentation convention. Pattern matching on a union  
type is exhaustive — the compiler verifies that all members are handled.  
Unlike sealed traits, union types require no common base class, making  
them ideal for integrating types from different libraries. The `Primitive`  
alias demonstrates that union types can have multiple members and can  
be used as ordinary type parameters. This enables lightweight sum types  
without the syntactic overhead of case class hierarchies.  

---

## Using intersection types in FP

Intersection types express that a value simultaneously satisfies  
multiple type constraints, replacing complex mixin hierarchies.  

```scala
trait Readable:
  def read: String

trait Writable:
  def write(s: String): Unit

trait Closeable:
  def close(): Unit

class FileHandle(path: String) extends Readable, Writable, Closeable:
  private var content = s"content of $path"
  def read: String = content
  def write(s: String): Unit = content = s
  def close(): Unit = println(s"closed $path")

def withResource[R <: Readable & Closeable](r: R)(f: R => Unit): Unit =
  try f(r) finally r.close()

def copyContent(src: Readable & Closeable,
                dst: Writable & Closeable): Unit =
  withResource(src)(r => dst.write(r.read))

@main def main() =
  val src = FileHandle("source.txt")
  val dst = FileHandle("dest.txt")
  copyContent(src, dst)
  println(s"dst content: ${dst.read}")
end main
```

`Readable & Closeable` is an intersection type: `withResource` accepts  
any value that implements both interfaces, regardless of the class  
hierarchy. This is more flexible than requiring a common abstract base  
class and more precise than accepting any `Readable` without the  
closure guarantee. `copyContent` demonstrates composing two different  
intersection types in a single function signature. Intersection types  
interact cleanly with Scala 3's structural typing and with `given`  
evidence, producing expressive, fine-grained type constraints.  

---

## Using inline for FP optimizations

The `inline` keyword evaluates expressions at compile time, enabling  
zero-overhead generic programming without runtime dispatch.  

```scala
inline def ifDefined[A](opt: Option[A], inline msg: String): A =
  opt match
    case Some(a) => a
    case None    => throw new NoSuchElementException(msg)

inline def logged[A](label: String, inline body: A): A =
  val start = System.nanoTime()
  val result = body
  val elapsed = System.nanoTime() - start
  println(s"[$label] ${elapsed}ns")
  result

inline def pipe[A, B, C](a: A, f: A => B, g: B => C): C =
  g(f(a))

@main def main() =
  val opt = Some(42)
  println(ifDefined(opt, "value must be present"))

  val v = logged("computation") {
    (1 to 1000).sum
  }
  println(s"sum = $v")

  val result = pipe("hello", _.length, _ * 2)
  println(s"pipe result = $result")
end main
```

`inline` functions are expanded at every call site by the compiler,  
eliminating the overhead of a method call and enabling the compiler to  
optimize the resulting code as if the body were written inline. The  
`inline` modifier on the `msg` parameter of `ifDefined` means that  
the string literal is embedded directly at the call site rather than  
allocated as a heap object on a code path that may never be reached.  
`logged` wraps any expression with timing instrumentation at zero  
abstraction cost. These patterns are idiomatic in performance-sensitive  
functional libraries written in Scala 3.  

---

## Using recursion schemes (basic)

Recursion schemes separate the recursion structure from the  
computation, allowing algorithms to be defined once and reused.  

```scala
enum Tree[+A]:
  case Leaf(value: A)
  case Branch(left: Tree[A], right: Tree[A])

def fold[A, B](tree: Tree[A])(
    leaf: A => B,
    branch: (B, B) => B): B =
  tree match
    case Tree.Leaf(v)       => leaf(v)
    case Tree.Branch(l, r)  =>
      branch(fold(l)(leaf, branch), fold(r)(leaf, branch))

def sum(tree: Tree[Int]): Int =
  fold(tree)(identity, _ + _)

def depth[A](tree: Tree[A]): Int =
  fold(tree)(_ => 1, (l, r) => 1 + l.max(r))

def toList[A](tree: Tree[A]): List[A] =
  fold(tree)(List(_), _ ++ _)

@main def main() =
  import Tree.*
  val t = Branch(Branch(Leaf(1), Leaf(2)), Branch(Leaf(3), Leaf(4)))
  println(s"sum   = ${sum(t)}")
  println(s"depth = ${depth(t)}")
  println(s"list  = ${toList(t)}")
end main
```

`fold` is a *catamorphism* for the `Tree` type: it replaces each  
constructor with a provided function. `Leaf(v)` is replaced by `leaf(v)`  
and `Branch(l, r)` is replaced by `branch(result_of_left, result_of_right)`.  
By supplying different algebra functions, we implement `sum`, `depth`,  
and `toList` without re-writing any recursive boilerplate. Recursion  
schemes make algorithms compositional: multiple traversals can be  
fused by passing both algebras to a single `fold`, preventing multiple  
passes over the data structure.  

---

## Using fixed-point types (intro)

A fixed-point type `Fix[F]` separates the recursive shell of a data  
type from its payload, enabling generic recursion over any functor.  

```scala
case class Fix[F[_]](unfix: F[Fix[F]])

enum ExprF[+A]:
  case Num(n: Int)
  case Add(l: A, r: A)
  case Mul(l: A, r: A)

type Expr = Fix[ExprF]

def num(n: Int): Expr = Fix(ExprF.Num(n))
def add(l: Expr, r: Expr): Expr = Fix(ExprF.Add(l, r))
def mul(l: Expr, r: Expr): Expr = Fix(ExprF.Mul(l, r))

def eval(expr: Expr): Int =
  expr.unfix match
    case ExprF.Num(n)    => n
    case ExprF.Add(l, r) => eval(l) + eval(r)
    case ExprF.Mul(l, r) => eval(l) * eval(r)

def pretty(expr: Expr): String =
  expr.unfix match
    case ExprF.Num(n)    => n.toString
    case ExprF.Add(l, r) => s"(${pretty(l)} + ${pretty(r)})"
    case ExprF.Mul(l, r) => s"(${pretty(l)} * ${pretty(r)})"

@main def main() =
  // (2 + 3) * 4
  val e = mul(add(num(2), num(3)), num(4))
  println(pretty(e))
  println(eval(e))
end main
```

`Fix[F]` ties the recursive knot: a value of type `Fix[ExprF]` contains  
an `ExprF[Fix[ExprF]]`, which may itself contain more `Fix[ExprF]`  
values. `ExprF` is not recursive — `A` appears where the recursion  
would be, making `ExprF` a plain functor. This separation lets us write  
generic traversal functions (catamorphisms, anamorphisms) once for  
`Fix[F]` and reuse them for any `F`. The `eval` and `pretty` functions  
are primitive catamorphisms; a library like Matryoshka generalises them  
fully, but the core idea is visible here.  

---

## Natural transformations

A natural transformation converts values from one functor to another  
while preserving the structure of the contained data.  

```scala
trait ~>[F[_], G[_]]:
  def apply[A](fa: F[A]): G[A]

val optionToList: Option ~> List = new ~>[Option, List]:
  def apply[A](fa: Option[A]): List[A] = fa.toList

val listToOption: List ~> Option = new ~>[List, Option]:
  def apply[A](fa: List[A]): Option[A] = fa.headOption

def transform[F[_], G[_], A](fa: F[A], nt: F ~> G): G[A] =
  nt(fa)

def mapThenTransform[F[_], G[_], A, B](
    fa: F[A],
    f: A => B,
    nt: F ~> G)(using funF: Functor[F]): G[B] =
  nt(funF.map(fa)(f))

trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

given Functor[Option] with
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)

@main def main() =
  println(transform(Some(42), optionToList))
  println(transform(None: Option[Int], optionToList))
  println(transform(List(1, 2, 3), listToOption))
  println(transform(List.empty[String], listToOption))
  println(mapThenTransform(Some(5), (_: Int) * 2, optionToList))
end main
```

A natural transformation `F ~> G` must work uniformly for *every* type  
`A`; this is enforced by the rank-1 polymorphic `apply[A]` method.  
`optionToList` converts presence/absence into a zero-or-one-element  
list; `listToOption` collapses a list to its first element or absence.  
Natural transformations satisfy the *naturality condition*: mapping  
with `f` and then transforming is the same as transforming and then  
mapping. This commutativity property means natural transformations are  
safe to use at structural boundaries — for example, when switching the  
effect type in a tagless-final interpreter stack.  

---

## Idiomatic advanced FP in Scala 3

This final example integrates typeclasses, ADTs, opaque types, effect  
composition, and tagless final into a cohesive micro-application.  

```scala
object App:
  // --- Domain ---
  opaque type UserId = Int
  object UserId:
    def apply(n: Int): UserId = n
    extension (u: UserId) def value: Int = u

  enum UserError:
    case NotFound(id: UserId)
    case Inactive(id: UserId)

  case class User(id: UserId, name: String, active: Boolean)

  // --- Algebra ---
  trait UserAlgebra[F[_]]:
    def find(id: UserId): F[Either[UserError, User]]
    def greet(user: User): F[String]

  // --- Typeclass ---
  trait Show[A]:
    def show(a: A): String

  given Show[User] with
    def show(u: User): String =
      s"User(${u.id.value}, ${u.name}, active=${u.active})"

  given Show[UserError] with
    def show(e: UserError): String =
      e match
        case UserError.NotFound(id)  => s"Not found: ${id.value}"
        case UserError.Inactive(id)  => s"Inactive: ${id.value}"

  extension [A](a: A)(using s: Show[A]) def display: String = s.show(a)

  // --- Interpreter ---
  type Id[A] = A

  given UserAlgebra[Id] with
    private val db = Map(
      UserId(1) -> User(UserId(1), "Alice", true),
      UserId(2) -> User(UserId(2), "Bob",   false))

    def find(id: UserId): Id[Either[UserError, User]] =
      db.get(id) match
        case None    => Left(UserError.NotFound(id))
        case Some(u) =>
          if u.active then Right(u)
          else Left(UserError.Inactive(id))

    def greet(user: User): Id[String] =
      s"Hello, ${user.name}!"

  // --- Program ---
  def run[F[_]](id: UserId)(using A: UserAlgebra[F]): F[String] =
    A.find(id) match
      case Right(user) => A.greet(user)
      case Left(err)   => summon[F[String] =:= F[String]].apply(
                            err.display.asInstanceOf[F[String]])

  def runId(id: UserId)(using A: UserAlgebra[Id]): String =
    A.find(id) match
      case Right(user) => A.greet(user)
      case Left(err)   => err.display

@main def main() =
  import App.*
  println(runId(UserId(1)))
  println(runId(UserId(2)))
  println(runId(UserId(99)))
  println(User(UserId(1), "Alice", true).display)
end main
```

This example weaves together every major theme in the document.  
`UserId` is an opaque type preventing accidental integer mixing.  
`UserError` is an ADT making all failure modes explicit. `UserAlgebra[F]`  
is a tagless-final algebra whose `Id` interpreter uses a pure in-memory  
map. `Show` is a typeclass with derived extension method syntax.  
`runId` sequences algebra operations and uses `Show` to format errors —  
all in a single, cohesive pipeline. The program is fully pure up to the  
`@main` entry point; the only side effect is the final `println`. This  
is the shape of idiomatic advanced FP in Scala 3: composable,  
type-safe, testable, and expressive.  
