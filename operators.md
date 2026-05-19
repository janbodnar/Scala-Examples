
# Scala Operators

An **operator** is a symbol (or combination of symbols) that represents a method call.  
An **operand** is one of the inputs (arguments) of that method. Expressions are built  
from operands and operators, and their evaluation order is determined by **precedence**  
and **associativity** rules.

Operators typically work with one or two operands. **Unary operators** (like `-a`)  
correspond to methods named `unary_!`, `unary_~`, `unary_+`, or `unary_-`.  
**Binary operators** (like `a + b`) are just methods invoked on the left operand with  
the right operand as argument: `a.+(b)`.  

Scala has no ternary operator; instead, it uses an `if-else` **expression** that  
returns a value.

Because operators are methods, they can be overloaded by any class. For example, `+`  
adds numbers, concatenates strings, or can be defined for custom types. This is a  
natural consequence of Scala’s unified object model.  


## Sign operators

Unary `+` and `-` are implemented by the methods `unary_+` and `unary_-`.  

```scala
@main def signDemo(): Unit =
  println(2)
  println(+2)
  println(-2)
```

The unary `-` negates the value. Double negation returns the original number.

```scala
@main def doubleNegation(): Unit =
  var a = 1
  println(-a)       // -1
  println(-(-a))    // 1
```

Behind the scenes, `-a` calls `a.unary_-`.

---

## Multiple assignment

Scala does not support comma‑separated multiple assignment of separate variables in one statement, but you can declare several variables on one line or use tuple extraction.

```scala
@main def multipleVals(): Unit =
  val x = 10
  val y = 20
  val z = 30
  val sum = x + y + z
  println(s"Sum: $sum")
```

With **pattern matching** you can extract tuple values in one step:

```scala
val (a, b, c) = (10, 20, 30)
println(s"a=$a, b=$b, c=$c")
```

---

## Assignment operator

The `=` sign is used for assignment; it is **not** an operator in the same sense as methods.

```scala
@main def simpleAssignment(): Unit =
  var x = 1
  println(x)   // 1
```

Assignment is an action: evaluate the right side, then store the result.

```scala
@main def reassign(): Unit =
  var x = 1
  x = x + 1   // x.+(1)
  println(x)  // 2
```

Scala encourages immutability (`val`), but `var` is available when mutation is needed.

---

## String concatenation

The `+` method on strings works as expected. Scala also offers string interpolation, which is often more readable.

```scala
@main def stringConcat(): Unit =
  println("Return " + "of " + "the king.")
  println("Return".concat(" of").concat(" the king."))
  // Using string interpolation
  val first = "Hello"
  val second = "World"
  println(s"$first $second")  // Hello World
```

String interpolation `s"..."` embeds expressions directly, avoiding many explicit concatenations.

---

## Increment and decrement operators

Scala does **not** have `++` or `--` operators. Use `+= 1` and `-= 1` instead.

```scala
@main def incDec(): Unit =
  var x = 6
  x += 1
  x += 1
  println(x)   // 8
  x -= 1
  println(x)   // 7
```

The `+=` is desugared to `x = x + 1`, which calls `x.+(1)`.

For pre‑/post‑increment behaviour in expressions, you need to use an explicit temporary variable or separate statements, because the compound assignment returns `Unit`.

```scala
var a = 5
val b = { val tmp = a; a += 1; tmp }  // post‑increment workaround
val c = { a += 1; a }                 // pre‑increment workaround
println(s"a=$a, b=$b, c=$c")
```

---

## Arithmetic operators

The familiar arithmetic operators are all method calls.

```scala
@main def arithmetic(): Unit =
  val a = 10
  val b = 11
  val c = 12

  val add = a + b + c
  val sub = c - a
  val mult = a * b
  val div = c / 3
  val rem = c % a

  println(s"Addition: $add")
  println(s"Subtraction: $sub")
  println(s"Multiplication: $mult")
  println(s"Division: $div")
  println(s"Remainder: $rem")
```

Modulo works identically:

```scala
println(9 % 4)  // 1
```

### Integer vs floating‑point division

Division of two integers yields an integer (truncated), unless one operand is a floating‑point type.

```scala
@main def divisionTypes(): Unit =
  val intResult = 5 / 2         // 2
  val floatResult = 5 / 2.0     // 2.5
  println(s"Integer division: $intResult")
  println(s"Floating-point division: $floatResult")

  val a = 7
  val b = 2
  val floatDiv = a.toDouble / b // explicit conversion
  println(s"7 / 2 (double): $floatDiv")
```

---

## Boolean operators

Logical AND (`&&`), OR (`||`), and NOT (`!`) are all methods (with `!` being `unary_!`).

```scala
@main def logicalOps(): Unit =
  val x = 3
  val y = 8
  println(s"x == y: ${x == y}")
  println(s"y > x: ${y > x}")
  if y > x then
    println("y is greater than x")
```

Truth tables:

```scala
@main def andTable(): Unit =
  println(s"true && true: ${true && true}")
  println(s"true && false: ${true && false}")
  println(s"false && true: ${false && true}")
  println(s"false && false: ${false && false}")
```

`||` behaves similarly.

---

## Negation operator

`!` flips a Boolean. It is defined as `unary_!` on `Boolean`.

```scala
@main def negation(): Unit =
  println(!true)           // false
  println(!false)          // true
  println(!(4 < 3))        // true
```

---

## Short-circuit evaluation

`&&` and `||` use short‑circuiting: the right operand is only evaluated if needed.

```scala
def checkOne(): Boolean =
  println("Inside checkOne")
  false

def checkTwo(): Boolean =
  println("Inside checkTwo")
  true

@main def shortCircuit(): Unit =
  println("Testing AND:")
  if checkOne() && checkTwo() then
    println("Pass")

  println("\nTesting OR:")
  if checkTwo() || checkOne() then
    println("Pass")
```

Output:  
```
Testing AND:
Inside checkOne

Testing OR:
Inside checkTwo
Pass
```

---

## Relational operators

`>`, `<`, `>=`, `<=`, `==`, `!=` compare values. In Scala, `==` checks **structural equality** (like Java’s `equals`), not reference equality.

```scala
@main def relational(): Unit =
  println(s"3 < 4: ${3 < 4}")
  println(s"3 == 4: ${3 == 4}")
  println(s"4 >= 3: ${4 >= 3}")
  println(s"4 != 3: ${4 != 3}")

  val age = 25
  val minAge = 18
  val maxAge = 65
  if age >= minAge && age <= maxAge then
    println("Age is within range")
```

---

## String equality

In Scala, `==` already compares string **content**, not references. For reference identity, use `eq` and `ne`.

```scala
@main def stringEquality(): Unit =
  val str1 = "hello"
  val str2 = "hello"
  val str3 = new String("hello")

  println(s"str1 == str2: ${str1 == str2}")       // true
  println(s"str1 == str3: ${str1 == str3}")       // true (structural)
  println(s"str1 eq str3: ${str1 eq str3}")       // false (reference)
```

---

## Bitwise operators

Bitwise operations `&`, `|`, `^`, `~`, `<<`, `>>`, `>>>` are available as methods on integral types (`Int`, `Long`, etc.).

```scala
@main def bitwise(): Unit =
  println(~7)       // -8
  println(~ -8)     // 7

  val a = 6 & 3     // 2
  val b = 6 | 3     // 7
  val c = 6 ^ 3     // 5
  println(s"6 & 3 = $a")
  println(s"6 | 3 = $b")
  println(s"6 ^ 3 = $c")
```

---

## Bit shifting operators

Shifts are methods: `<<`, `>>`, `>>>`.

```scala
@main def shifts(): Unit =
  val num = 8
  val left = num << 1    // 16
  val right = num >> 1   // 4
  println(s"8 << 1 = $left")
  println(s"8 >> 1 = $right")
  println(s"-8 >> 1 = ${-8 >> 1}")
  println(s"-8 >>> 1 = ${-8 >>> 1}")
```

---

## Compound assignment operators

`+=`, `-=`, `*=`, etc. are shorthand for updating a `var`. They work for any method that has the corresponding symbolic method (e.g., `+=` requires a `+` method).

```scala
@main def compound(): Unit =
  var a = 1
  a += 1     // a = a + 1
  println(a) // 2
  a += 5
  println(a) // 7
  a *= 3
  println(a) // 21

  var count = 10
  count -= 3
  println(s"After subtraction: $count") // 7
  count /= 2
  println(s"After division: $count")    // 3
  count %= 2
  println(s"Remainder: $count")         // 1
```

---

## Pattern matching (instanceof alternative)

Scala does not have the `instanceof` operator. Instead, it uses **pattern matching** or the `isInstanceOf[T]` method (less idiomatic).

```scala
class Base
class Derived extends Base

@main def typeCheck(): Unit =
  val b: Base = new Base
  val d: Base = new Derived

  println(s"d is Base: ${d.isInstanceOf[Base]}")
  println(s"b is Derived: ${b.isInstanceOf[Derived]}")

  // Idiomatic pattern matching
  d match
    case _: Derived => println("d is a Derived")
    case _: Base    => println("d is a Base")
```

Pattern matching can also extract fields and bind variables:

```scala
val obj: Any = "Hello"
obj match
  case s: String => println(s"String length: ${s.length}")
  case _         => println("Not a string")
```

---

## Lambda operator

The **lambda operator** in Scala is `=>`. It defines anonymous functions.

```scala
@main def lambdaSort(): Unit =
  val words = Array("kind", "massive", "atom", "car", "blue")
  scala.util.Sorting.quickSort(words)(Ordering.by(_.length))
  // or using a lambda with sortWith
  val sorted = words.sortWith((s1, s2) => s1 < s2)
  println(sorted.mkString(", "))
```

A lambda can be assigned to a variable that expects a function type:

```scala
trait GreetingService:
  def greet(msg: String): Unit

@main def lambdaGreet(): Unit =
  val gs: GreetingService = (msg) => println(msg)
  gs.greet("Good night")
  gs.greet("Hello there")
```

Often you use the standard `Function1` type:

```scala
val square: Int => Int = x => x * x
println(s"5 squared: ${square(5)}")  // 25
```

Scala also supports placeholders (`_`) for concise lambdas: `_ + 1` is equivalent to `x => x + 1`.

---

## Method reference operator

Scala does not have a single `::` operator for method references. Instead, you can **eta‑expand** a method into a function using an underscore after the method name.

```scala
def greet(msg: String): Unit = println(msg)

@main def methodRef(): Unit =
  val f: String => Unit = greet // method is automatically eta-expanded when assigned
  f("Hello there")

  // Explicit eta‑expansion
  val g = greet _
  g("Hello again")
```

For instance methods, you can use a lambda placeholder:

```scala
val numbers = List(1, 2, 3, 4, 5)
numbers.foreach(println)   // println is eta‑expanded
```

---

## Operator precedence

Precedence in Scala is determined by the **first character** of the operator/method name (not by the symbol set). For example, all letters have lower precedence than most special characters. The full table:

| Precedence (lowest to highest) | Category / first character            |
|--------------------------------|---------------------------------------|
| lowest                         | (all letters)                         |
|                                | `|`                                   |
|                                | `^`                                   |
|                                | `&`                                   |
|                                | `= !`                                 |
|                                | `< >`                                 |
|                                | `:`                                   |
|                                | `+ -`                                 |
|                                | `* / %`                               |
| highest                        | (all other special characters, e.g. `~`, `?`) |

Parentheses can override any precedence.

```scala
@main def precedenceDemo(): Unit =
  println(3 + 5 * 5)      // 28 ( * before + )
  println((3 + 5) * 5)    // 40
  println(!true | true)   // true  ( ! has higher precedence than | )
  println(!(true | true)) // false
```

The rule applies uniformly: `a + b * c` is `a.+(b.*(c))`. Every symbolic method call respects the first‑character precedence.

---

## Associativity rule

**Associativity** is determined by the **last character** of the method name:
- If it ends with `:` (e.g., `::`, `+:`, `:+`), the method is **right‑associative**, and it operates on the right operand (the method belongs to the right operand).
- Otherwise, it is **left‑associative**.

Example: the `::` (cons) operator for lists is right‑associative:

```scala
val list = 1 :: 2 :: 3 :: Nil  // interpreted as 1 :: (2 :: (3 :: Nil))
```

Left‑associative operators behave conventionally: `9 / 3 * 3` is `(9 / 3) * 3` = `9`.

```scala
@main def associativity(): Unit =
  val result = 9 / 3 * 3
  println(s"9 / 3 * 3 = $result")   // 9
```

Assignment operators like `=` and `+=` are not methods and are always handled by the compiler as special forms (they do not follow the colon rule). However, the right‑associativity of `:`‑ending methods is a powerful design for building data structures.

```scala
var j = 0
j *= 3 + 1    // j = j * (3 + 1)  -> 0
println(j)    // 0
```

Chained assignments work because `a = b = c = 0` is parsed as `a = (b = (c = 0))` (assignments are not expressions returning a value; they produce `Unit`, so this is rarely useful in Scala).

---

## If‑else as ternary alternative

Instead of `?:`, Scala uses the `if‑else` expression, which always returns a value.

```scala
@main def ifElseExpression(): Unit =
  val age = 31
  val adult = if age >= 18 then true else false
  println(s"Adult: $adult")

  val score = 85
  val grade = if score >= 90 then "A"
              else if score >= 80 then "B"
              else if score >= 70 then "C"
              else if score >= 60 then "D"
              else "F"
  println(s"Grade: $grade")
```

---

## Null coalescing with Option

Scala discourages `null` in favour of `Option`. For cases where `null` exists, you can use `Option(...)` and `getOrElse` as a safe default.

```scala
val name: String = null
val displayName = Option(name).getOrElse("Anonymous")
println(s"User: $displayName")

val score = 0
val message = if score > 0 then "Positive"
              else if score < 0 then "Negative"
              else "Zero"
println(s"Score status: $message")
```

The `if‑else` expression covers all ternary patterns without a separate operator.

---

## Prime number calculation

A classic example bringing together many operators.

```scala
@main def primes(): Unit =
  val nums = (0 to 28).toArray
  print("Prime numbers: ")
  for num <- nums do
    if num == 0 || num == 1 then ()
    else if num == 2 || num == 3 then print(s"$num ")
    else
      var i = math.sqrt(num).toInt
      var isPrime = true
      while i > 1 && isPrime do
        if num % i == 0 then isPrime = false
        i -= 1
      if isPrime then print(s"$num ")
  println()
```

---

## Combining multiple operators

Precedence, associativity, and side effects (when using `var`) must be considered together.

```scala
@main def complexExpression(): Unit =
  var a = 5
  var b = 10
  var c = 15

  val result1 = a + b * c          // 5 + 150 = 155
  val result2 = (a + b) * c        // 15 * 15 = 225

  println(s"5 + 10 * 15 = $result1")
  println(s"(5 + 10) * 15 = $result2")

  // Note: ++ does not exist; we simulate increment expressions
  val tempA = a
  a += 1
  b += 1
  val result3 = tempA + b * c
  c -= 1
  // After this, a=6, b=11, c=14, result3 = 5 + 11*15 = 170
  println(s"After complex expression:")
  println(s"a = $a, b = $b, c = $c")
```

---

## Operators with collections

Compound assignment and lambda operators make collection processing concise.

```scala
@main def collectionsOps(): Unit =
  val numbers = List(1, 2, 3, 4, 5)
  var sum = 0
  var product = 1
  for num <- numbers do
    sum += num
    product *= num
  println(s"Sum: $sum")
  println(s"Product: $product")
  println(s"Average: ${sum.toDouble / numbers.length}")
```

More idiomatic Scala uses higher‑order functions:

```scala
val sum2 = numbers.sum
val product2 = numbers.product
val avg = numbers.sum.toDouble / numbers.size
```

---

## Logical operators with predicates

Boolean logic remains straightforward; Scala’s expression‑oriented nature shines when building predicates.

```scala
@main def predicates(): Unit =
  val age = 25
  val hasLicense = true
  val hasInsurance = true

  val canDrive = age >= 18 && hasLicense && hasInsurance
  println(s"Can drive: $canDrive")

  val needsAttention = !hasLicense || !hasInsurance
  println(s"Needs attention: $needsAttention")
```

---

## Key takeaways

- In Scala, **operators are methods** – this unifies object‑oriented and functional programming.
- Unary prefixes are special methods (`unary_!`, `unary_-`, etc.).
- Precedence is based on the **first character** of the method name, not the symbol itself.
- Associativity depends on the **last character** (colon‑ending methods are right‑associative).
- There is no `++`, `--`, or `?:` – use `+=`, `if‑else`, and pattern matching.
- `==` checks **structural equality**; `eq`/`ne` checks reference identity.
- Custom types can define operators to create natural, expressive DSLs.

This document has shown the core operator idioms in Scala. By understanding these rules, you can read and write code that is both concise and precise.
