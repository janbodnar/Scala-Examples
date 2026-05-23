# Functions

Scala 3 treats functions as first-class values. A function can be passed  
to another function, returned from a function, stored in a variable, or  
composed with other functions. Scala supports both `def` definitions and  
function values (lambdas), and the two styles integrate seamlessly. The  
language provides concise syntax for anonymous functions, currying,  
partial application, and higher-order patterns. Scala 3 also adds inline  
functions, extension methods, and context functions for advanced use  
cases. Together these features make Scala a powerful platform for both  
object-oriented and functional programming styles.  

## Simple function

A minimal function definition that takes an integer and returns a value.  

```scala
@main def main() =

    def square(n: Int): Int = n * n

    println(square(4))
    println(square(9))
end main
```

The `def` keyword introduces a function definition. `square` takes a  
single `Int` parameter `n` and returns its product with itself. The  
return type `: Int` is written after the parameter list. In Scala 3,  
single-expression functions do not need a block; the body follows the  
`=` sign directly. The function is called like any method with  
parentheses surrounding the argument.  

## Parameters and return types

A function with multiple parameters and an explicit return type.  

```scala
@main def main() =

    def add(x: Int, y: Int): Int = x + y

    def describe(name: String, age: Int): String =
        s"$name is $age years old"

    println(add(3, 7))
    println(describe("Alice", 30))
end main
```

`add` accepts two `Int` parameters and returns their sum. `describe`  
accepts a `String` and an `Int` and returns a formatted `String` using  
an interpolated string literal. When the body spans more than one  
expression, it can be placed on the next line with indentation. Scala  
infers the return type from the body, but writing it explicitly improves  
readability and catches accidental type mismatches early.  

## Default parameter values

Functions can provide default values so callers may omit those arguments.  

```scala
@main def main() =

    def greet(name: String, greeting: String = "Hello"): String =
        s"$greeting, $name!"

    println(greet("Bob"))
    println(greet("Alice", "Hi"))
    println(greet("Eve", "Good morning"))
end main
```

The parameter `greeting` is given the default value `"Hello"`. When  
`greet` is called with only one argument, Scala substitutes the default  
automatically. Callers that need a different greeting pass it explicitly.  
Default parameters reduce the need for overloaded function definitions  
and keep the call sites concise.  

## Named arguments

Arguments can be passed by name, in any order, for clarity.  

```scala
@main def main() =

    def connect(host: String, port: Int, secure: Boolean = false): String =
        val scheme = if secure then "https" else "http"
        s"$scheme://$host:$port"

    println(connect("example.com", 8080))
    println(connect(port = 443, host = "api.example.com", secure = true))
end main
```

Named arguments let callers label each value with the corresponding  
parameter name. This is especially useful when a function has several  
parameters of similar types, or when the order of arguments would  
otherwise be unclear. Here the second call passes `port` before `host`,  
which is allowed because both are identified by name.  

## Unit return type

A function whose purpose is a side effect returns `Unit`.  

```scala
@main def main() =

    def printBanner(title: String): Unit =
        val line = "-" * title.length
        println(line)
        println(title)
        println(line)

    printBanner("Scala 3 Functions")
    printBanner("Examples")
end main
```

`Unit` is the Scala equivalent of `void` in Java or C. A function  
declared with return type `Unit` is called for its side effects — here,  
printing decorated output to the console. The body contains multiple  
statements and therefore uses an indented block. The final expression  
of a `Unit` function is discarded; it does not need to be `()`.  

## Recursive function

A function that calls itself to compute the factorial of a number.  

```scala
@main def main() =

    def factorial(n: Int): Int =
        if n <= 1 then 1
        else n * factorial(n - 1)

    for i <- 0 to 10 do
        println(s"$i! = ${factorial(i)}")
end main
```

Recursion replaces explicit loops when a problem is naturally  
self-similar. `factorial` reduces `n` by 1 on each call until it  
reaches the base case `n <= 1`, at which point it returns 1 and the  
call stack unwinds, multiplying the accumulated values together.  
Scala 3's `if … then … else` syntax keeps the definition compact  
and reads like a mathematical specification.  

## Tail recursion

A tail-recursive function optimised by the compiler using `@tailrec`.  

```scala
import scala.annotation.tailrec

@main def main() =

    def factorial(n: Int): BigInt =

        @tailrec
        def loop(n: Int, acc: BigInt): BigInt =
            if n <= 1 then acc
            else loop(n - 1, n * acc)

        loop(n, 1)

    for i <- 0 to 15 do
        println(s"$i! = ${factorial(i)}")
end main
```

A naive recursive function grows the call stack with each invocation.  
A tail-recursive function performs its recursive call as the very last  
operation so the compiler can reuse the current stack frame, avoiding  
stack overflow for large inputs. The `@tailrec` annotation causes a  
compile error if the function is not actually tail-recursive. The  
accumulator pattern — passing `acc` alongside `n` — is the standard  
idiom for converting recursion to tail-recursion.  

## Anonymous function

A function value (lambda) stored in a variable.  

```scala
@main def main() =

    val double: Int => Int = n => n * 2
    val isEven: Int => Boolean = n => n % 2 == 0

    val nums = List(1, 2, 3, 4, 5, 6)

    println(nums.map(double))
    println(nums.filter(isEven))
end main
```

An anonymous function (lambda) is written as `parameters => body`. The  
type `Int => Int` describes a function that takes an `Int` and returns  
an `Int`. Lambdas can be stored in `val` bindings and reused anywhere  
a function value is expected. Here `double` and `isEven` are passed to  
`map` and `filter`, which are higher-order collection methods.  

## Higher-order functions

A function that accepts another function as a parameter.  

```scala
@main def main() =

    def applyTwice(f: Int => Int, x: Int): Int = f(f(x))

    def triple(n: Int): Int = n * 3

    println(applyTwice(triple, 2))
    println(applyTwice(_ + 10, 5))
    println(applyTwice(n => n * n, 2))
end main
```

`applyTwice` takes a function `f` of type `Int => Int` and an integer  
`x`, then applies `f` to `x` twice. This is a classic higher-order  
function — it abstracts over behaviour rather than data. The callers  
supply a named function `triple`, a shorthand placeholder lambda  
`_ + 10`, and an inline lambda `n => n * n`. Higher-order functions are  
the foundation of functional programming in Scala.  

## Returning a function

A function that returns another function as its result.  

```scala
@main def main() =

    def multiplier(factor: Int): Int => Int =
        n => n * factor

    val double = multiplier(2)
    val triple = multiplier(3)

    println(double(7))
    println(triple(7))
    println(multiplier(5)(4))
end main
```

`multiplier` returns a lambda that captures `factor` from its enclosing  
scope. Each call to `multiplier` produces a new function specialised for  
a different factor. The returned lambda is a *closure* because it closes  
over the variable `factor` defined outside its own body. The expression  
`multiplier(5)(4)` calls `multiplier` and immediately applies the result  
to `4`, demonstrating that the return value is a genuine function.  

## Closures

A lambda that captures and reads a variable from its enclosing scope.  

```scala
@main def main() =

    var count = 0

    val increment = () =>
        count += 1
        count

    println(increment())
    println(increment())
    println(increment())
    println(s"Final count: $count")
end main
```

A closure is a function that references variables defined outside its  
own parameter list. Here `increment` captures the mutable variable  
`count`. Each call to `increment` reads and modifies `count`, so  
successive calls return 1, 2, and 3. The final `println` confirms that  
`count` in the outer scope was also updated, because the closure holds  
a reference to the same variable, not a copy.  

## Curried function

A function defined with multiple parameter lists so arguments can be  
applied one group at a time.  

```scala
@main def main() =

    def add(x: Int)(y: Int): Int = x + y

    val add5 = add(5)

    println(add(3)(4))
    println(add5(10))
    println(add5(20))
end main
```

Currying transforms a function of multiple arguments into a chain of  
single-argument functions. In Scala 3 this is expressed by writing  
multiple parameter lists separated by `()`. Calling `add(5)` with only  
the first list returns a partial function `Int => Int` that adds 5 to  
its argument. The derived `add5` function can then be applied to any  
integer. Currying facilitates partial application and makes it easy to  
create families of specialised functions from a single definition.  

## Partial application

Creating a specialised function by fixing some arguments of a  
multi-parameter function.  

```scala
@main def main() =

    def power(base: Int, exp: Int): Int =
        Math.pow(base, exp).toInt

    val square = power(_, 2)
    val cube   = power(_, 3)

    println(square(4))
    println(cube(3))
    println(List(1, 2, 3, 4, 5).map(square))
end main
```

The underscore `_` acts as a placeholder for an argument that has not  
yet been provided. `power(_, 2)` fixes the `exp` argument at 2, leaving  
the `base` open, and returns a new function `Int => Int`. Partial  
application differs from currying: rather than chaining single-argument  
functions, it fills a subset of parameters and leaves the rest as  
placeholders. The resulting `square` and `cube` values are passed  
directly to `map`.  

## Function composition

Combining two functions into one with `andThen` and `compose`.  

```scala
@main def main() =

    val double: Int => Int    = _ * 2
    val addOne: Int => Int    = _ + 1
    val square: Int => Int    = n => n * n

    val doubleAndAdd = double andThen addOne
    val squareThenDouble = square andThen double
    val doubleThenSquare = double compose square

    println(doubleAndAdd(5))
    println(squareThenDouble(3))
    println(doubleThenSquare(3))
end main
```

`andThen` applies the left function first, then passes its result to  
the right function. `compose` works in reverse order — the right  
function runs first. Both methods return a new function without  
evaluating anything immediately. Function composition is a clean way  
to build complex pipelines from simple, reusable pieces without  
introducing intermediate variables.  

## Using functions with collections

Applying higher-order collection methods with function arguments.  

```scala
@main def main() =

    val nums = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    val evens    = nums.filter(_ % 2 == 0)
    val doubled  = nums.map(_ * 2)
    val total    = nums.reduce(_ + _)
    val grouped  = nums.groupBy(_ % 3)

    println(evens)
    println(doubled)
    println(total)
    println(grouped)
end main
```

`filter` keeps elements that satisfy a predicate; `map` transforms each  
element; `reduce` collapses the list to a single value by applying the  
function pairwise from left to right; `groupBy` partitions elements into  
a `Map` keyed by the result of the classifier function. All four methods  
accept lambdas as arguments. The placeholder syntax `_ + _` is shorthand  
for `(a, b) => a + b` and keeps these one-liner pipelines readable.  

## FlatMap and chaining

Using `flatMap` to chain operations that each return a collection.  

```scala
@main def main() =

    val sentences = List(
        "the quick brown fox",
        "jumps over the lazy dog"
    )

    val words  = sentences.flatMap(_.split(" "))
    val unique = words.distinct.sorted

    println(words)
    println(unique)
end main
```

`flatMap` applies the function to every element and then flattens the  
resulting list of lists into a single list. Splitting each sentence  
produces a `List[Array[String]]`; `flatMap` collapses this into a  
single `List[String]`. Chaining `distinct` and `sorted` removes  
duplicates and orders the result alphabetically. `flatMap` is  
fundamental to monadic composition and underlies `for` comprehensions  
with `yield`.  

## Pattern matching in functions

Using pattern matching inside a function body to dispatch on value  
shapes.  

```scala
@main def main() =

    def describe(x: Any): String =
        x match
            case 0          => "zero"
            case n: Int     => if n > 0 then s"positive int $n"
                               else s"negative int $n"
            case s: String  => s"string of length ${s.length}"
            case _          => "unknown"

    println(describe(0))
    println(describe(42))
    println(describe(-7))
    println(describe("hello"))
    println(describe(3.14))
end main
```

The `match` expression tests `x` against each `case` in order and  
returns the first matching branch. Type patterns like `n: Int` both  
test the runtime type and bind the value to a variable. The wildcard  
`_` acts as a catch-all that handles any value not matched by earlier  
cases. Pattern matching in Scala is exhaustive when the compiler can  
verify all cases are covered, which improves safety over chains of  
`if-else` or `instanceof` checks.  

## Generic functions

A function parameterised over a type variable for reuse across types.  

```scala
@main def main() =

    def repeat[A](value: A, times: Int): List[A] =
        List.fill(times)(value)

    def first[A](xs: List[A]): Option[A] =
        if xs.isEmpty then None else Some(xs.head)

    println(repeat("ha", 3))
    println(repeat(42, 4))
    println(first(List(10, 20, 30)))
    println(first(List.empty[String]))
end main
```

Type parameters are declared in square brackets before the parameter  
list. `repeat[A]` works for any type `A` — strings, integers, or  
any other type. `first[A]` returns `Option[A]`: `Some` wrapping the  
head element if the list is non-empty, or `None` if it is empty.  
Generic functions avoid duplication by capturing algorithms that work  
identically regardless of the concrete type involved.  

## Variadic function

A function that accepts a variable number of arguments.  

```scala
@main def main() =

    def sum(nums: Int*): Int = nums.sum

    def joinWith(sep: String, parts: String*): String =
        parts.mkString(sep)

    println(sum(1, 2, 3))
    println(sum(10, 20, 30, 40, 50))
    println(joinWith(", ", "apple", "banana", "cherry"))

    val data = Array(1, 2, 3, 4, 5)
    println(sum(data*))
end main
```

The `*` suffix on the last parameter type makes it variadic. Inside the  
function body the parameter behaves as a `Seq`. Any number of arguments  
(including none) may be passed at the call site. To forward an existing  
array or sequence, append `*` to the argument — `data*` — so the  
compiler treats each element as a separate argument rather than passing  
the array itself.  

## Inline function

A function marked `inline` so the compiler substitutes its body at  
every call site.  

```scala
inline def clamp(value: Double, lo: Double, hi: Double): Double =
    if value < lo then lo
    else if value > hi then hi
    else value

@main def main() =

    println(clamp(5.0,  0.0, 10.0))
    println(clamp(-1.0, 0.0, 10.0))
    println(clamp(12.0, 0.0, 10.0))
end main
```

The `inline` modifier is a Scala 3 feature that instructs the compiler  
to replace every call to `clamp` with the function's body, similar to  
a macro expansion. This eliminates the overhead of a function call for  
performance-critical code and enables compile-time evaluation of  
constant arguments. Inline functions are best suited for small, hot  
utility functions where call-site overhead matters.  

## Extension method

Adding a new method to an existing type without subclassing.  

```scala
extension (n: Int)
    def factorial: Int =
        if n <= 1 then 1 else n * (n - 1).factorial

    def isPrime: Boolean =
        n > 1 && (2 until n).forall(n % _ != 0)

@main def main() =

    println(5.factorial)
    println(10.factorial)
    println(7.isPrime)
    println(10.isPrime)
end main
```

Extension methods allow new methods to be defined on types that the  
author does not control. Here `factorial` and `isPrime` are added to  
`Int` using the `extension` block. The receiver type `(n: Int)` is the  
implicit `this`. At the call site `5.factorial` reads as if `factorial`  
were always a member of `Int`. Extension methods replace the older  
implicit class pattern in Scala 2 with cleaner, more explicit syntax.  

## Local function

A helper function defined inside another function, invisible outside.  

```scala
@main def main() =

    def isPrime(n: Int): Boolean =

        def hasNoDivisor(d: Int): Boolean =
            if d * d > n then true
            else if n % d == 0 then false
            else hasNoDivisor(d + 1)

        n > 1 && hasNoDivisor(2)

    val primes = (2 to 50).filter(isPrime)
    println(primes)
end main
```

Functions can be nested inside other functions. `hasNoDivisor` is a  
tail-recursive helper visible only within `isPrime`. Keeping it local  
hides implementation details, prevents name pollution in the outer  
scope, and clearly communicates that the helper has no meaning outside  
its parent. The local function closes over `n` from its enclosing scope,  
so it does not need to receive `n` as an additional parameter.  

## Function as a value

Lifting a `def` into a function value with the eta-expansion operator.  

```scala
@main def main() =

    def negate(n: Int): Int = -n
    def isOdd(n:  Int): Boolean = n % 2 != 0

    val negFn: Int => Int          = negate
    val oddFn: Int => Boolean      = isOdd

    val nums = List(1, 2, 3, 4, 5, 6, 7)

    println(nums.map(negFn))
    println(nums.filter(oddFn))
    println(nums.map(negate))
end main
```

A `def` is not automatically a function value in Scala, but the compiler  
performs *eta-expansion* when a `def` appears where a function type is  
expected. Assigning `negate` to a `val` of type `Int => Int` triggers  
this expansion, wrapping the method in a function object. Passing a  
`def` name directly to a higher-order function like `map` also triggers  
eta-expansion implicitly.  

## Single abstract method

A trait with one abstract method can be instantiated with a lambda.  

```scala
trait Transformer:
    def transform(x: Int): Int

@main def main() =

    val doubler: Transformer  = x => x * 2
    val negator: Transformer  = x => -x
    val square:  Transformer  = x => x * x

    val nums = List(1, 2, 3, 4, 5)

    println(nums.map(doubler.transform))
    println(nums.map(negator.transform))
    println(nums.map(square.transform))
end main
```

SAM (Single Abstract Method) conversion lets Scala treat any trait or  
abstract class with exactly one unimplemented method as a functional  
interface. Instead of writing an anonymous class, the programmer  
provides a lambda directly. The compiler generates the anonymous class  
automatically. SAM conversion aligns Scala with Java functional  
interfaces such as `Runnable` and `Comparator`.  

## Context function

A function that requires an implicit value to be in scope at the call  
site.  

```scala
case class Config(debug: Boolean, prefix: String)

def log(msg: String)(using cfg: Config): Unit =
    if cfg.debug then
        println(s"[${cfg.prefix}] $msg")

def runPipeline()(using Config): Unit =
    log("Starting pipeline")
    log("Processing data")
    log("Pipeline complete")

@main def main() =

    given Config = Config(debug = true, prefix = "APP")
    runPipeline()
end main
```

The `using` keyword marks a parameter as a *context parameter* — the  
compiler searches for a `given` instance of the required type in the  
implicit scope at each call site. `log` and `runPipeline` both require  
a `Config` given instance, but callers never pass it explicitly. This  
pattern eliminates repetitive argument threading for cross-cutting  
concerns such as logging configuration, database connections, or  
execution contexts.  

## Given instance with a function

Defining a `given` that provides a default implementation for an  
abstract operation.  

```scala
trait Formatter[A]:
    def format(value: A): String

given Formatter[Int] with
    def format(value: Int): String = s"Int($value)"

given Formatter[Double] with
    def format(value: Double): String = f"Double($value%.2f)"

def display[A](value: A)(using fmt: Formatter[A]): Unit =
    println(fmt.format(value))

@main def main() =

    display(42)
    display(3.14159)
end main
```

`Formatter[A]` is a *type class* — an interface parameterised on `A`.  
The `given` declarations provide implementations for `Int` and  
`Double`. `display` summons the appropriate `given` automatically  
based on the type of `value`. Type classes decouple the data type from  
the behaviour, so new implementations can be added in any file without  
modifying the original type.  

## Opaque type with a function

Combining opaque types and functions to enforce domain invariants.  

```scala
object Domain:

    opaque type Celsius    = Double
    opaque type Fahrenheit = Double

    def celsius(d: Double): Celsius       = d
    def fahrenheit(d: Double): Fahrenheit = d

    def toFahrenheit(c: Celsius): Fahrenheit = c * 9.0 / 5.0 + 32.0
    def toCelsius(f: Fahrenheit): Celsius    = (f - 32.0) * 5.0 / 9.0

    extension (c: Celsius)
        def show: String = f"${c}%.1f °C"

    extension (f: Fahrenheit)
        def show: String = f"${f}%.1f °F"

@main def main() =

    import Domain.*

    val boiling = celsius(100.0)
    val body    = fahrenheit(98.6)

    println(boiling.show)
    println(toFahrenheit(boiling).show)
    println(body.show)
    println(toCelsius(body).show)
end main
```

Opaque types define a new named type backed by an existing type, but  
the backing representation is hidden outside the defining object.  
`Celsius` and `Fahrenheit` are both `Double` at runtime, but the  
compiler prevents mixing them up — you cannot pass a `Celsius` where  
a `Fahrenheit` is expected. Smart constructors `celsius` and  
`fahrenheit` are the only way to create values, enabling validation  
if needed. Extension methods restore convenient syntax within the  
encapsulated module.  

## Memoization with a function

Caching expensive function results using a mutable map.  

```scala
import scala.collection.mutable

def memoize[A, B](f: A => B): A => B =

    val cache = mutable.Map.empty[A, B]
    key =>
        cache.getOrElseUpdate(key, f(key))

@main def main() =

    def slowSquare(n: Int): Int =
        Thread.sleep(100)
        n * n

    val fastSquare = memoize(slowSquare)

    val t0 = System.currentTimeMillis()
    println(fastSquare(5))
    println(fastSquare(5))
    println(fastSquare(6))
    val t1 = System.currentTimeMillis()

    println(s"Elapsed: ${t1 - t0} ms (2 cache hits expected)")
end main
```

`memoize` is a generic higher-order function that wraps any pure  
function `f: A => B` and caches its results. The first call for a  
given key invokes `f` and stores the result; subsequent calls with  
the same key return the cached value immediately. `getOrElseUpdate`  
performs both the lookup and the insert atomically on the mutable map.  
The elapsed time demonstrates that the second call to `fastSquare(5)`  
does not sleep, confirming cache reuse.  

## Blocks

Functions accept block arguments passed with curly braces.  

```scala
@main def main() =

    val nums = List(1, 2, 3, 4, 5)

    nums.foreach { n =>
        val doubled = n * 2
        println(s"$n * 2 = $doubled")
    }

    val result = nums.map { n =>
        val sq = n * n
        sq + 1
    }

    println(result)
end main
```

A block `{ … }` can be passed as the last argument to a method using  
Scala's trailing-lambda syntax. Inside the block, multiple statements  
may appear; the last expression is the return value of the block.  
Here the `map` block computes the square of each element and then adds  
one, returning the final expression `sq + 1` as the mapped value.  
Block arguments are idiomatic when the lambda body spans more than one  
line.  
