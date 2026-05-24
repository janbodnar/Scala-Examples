# Scala 3 Data Types

A **data type** in Scala 3 defines the set of values an expression can  
produce and the operations permitted on those values. Every expression  
carries a static type that the compiler verifies before the program runs,  
eliminating an entire class of runtime errors at the earliest possible  
stage. Types are first-class citizens in Scala and can be inferred,  
aliased, composed, and even computed at compile time.  

Scala distinguishes between primitive-like types and reference types  
through a unified type hierarchy. **Value types** extend `AnyVal` and  
map directly to JVM primitives such as `int`, `long`, `double`, and  
`boolean`. The compiler erases them to unboxed primitives wherever  
possible, avoiding heap allocation and enabling efficient arithmetic.  
**Reference types** extend `AnyRef`, which aliases `java.lang.Object`,  
and always reside on the heap.  

The JVM operates with a strict separation between primitive types and  
object types. Scala bridges that divide with a hierarchy rooted at  
`Any`. The compiler automatically **boxes** a value type when context  
requires an object — for example when storing an `Int` in a generic  
collection — and **unboxes** it transparently in the reverse direction.  
Understanding this mechanism is essential when writing  
performance-sensitive code that must avoid repeated allocation.  

Value types in Scala include `Byte`, `Short`, `Int`, `Long`, `Float`,  
`Double`, `Char`, `Boolean`, and `Unit`. All extend `AnyVal`. Reference  
types include `String`, all collection classes, user-defined classes,  
and the special types `Null` and `Nothing`. The type `Any` sits at the  
top of the hierarchy and is the common supertype of both branches.  

Scala supports a rich set of **literals** for expressing values directly  
in source code: integer literals, floating-point literals, character  
literals, string literals, boolean literals, and the unit literal `()`.  
The compiler infers the most specific type that satisfies the context,  
so writing `val x = 42` yields an `Int` without any annotation. When the  
inferred type would be too broad or ambiguous, an explicit annotation  
clarifies intent and serves as documentation.  

**Immutability** is a cornerstone of idiomatic Scala. Prefer `val` over  
`var` and immutable collection types over mutable ones. Immutable data  
cannot be corrupted by concurrent reads, simplifies reasoning about  
program state, and lets the compiler apply optimisations that would  
otherwise be unsafe. When shared state must evolve over time, prefer  
controlled mutation through dedicated abstractions.  

Choosing the right data type improves correctness, performance, and  
clarity. Use `Int` for general-purpose integers, `Long` when the range  
may exceed two billion, and `BigInt` only when arbitrary precision is  
required. Prefer `Double` for floating-point work and reach for  
`BigDecimal` only for exact decimal arithmetic such as currency  
calculations. Use `Option` instead of `null` to express optional  
values, `Either` to signal recoverable errors, and sealed hierarchies  
or enums to model finite sets of choices.  

---

## Integer types

Scala 3 provides four signed integer types that differ in bit width  
and value range.  

```scala
@main def main() =

    val b: Byte  = 127
    val s: Short = 32_767
    val i: Int   = 2_147_483_647
    val l: Long  = 9_223_372_036_854_775_807L

    println(s"Byte  : $b  (${Byte.MinValue} to ${Byte.MaxValue})")
    println(s"Short : $s  (${Short.MinValue} to ${Short.MaxValue})")
    println(s"Int   : $i")
    println(s"Long  : $l")

end main
```

`Byte` stores values from -128 to 127 in a single octet and is useful  
for binary data and network protocols. `Short` doubles that range to  
sixteen bits but sees little use in modern Scala. `Int` is the default  
integer type, covering roughly ±2.1 billion, and maps to the JVM  
`int` primitive. `Long` extends coverage to ±9.2 × 10¹⁸ and must be  
written with an `L` suffix when expressed as a literal. Underscores in  
numeric literals are purely cosmetic and improve readability at no  
runtime cost.  

## Floating-point types

`Float` and `Double` represent IEEE 754 single- and double-precision  
numbers respectively.  

```scala
@main def main() =

    val f: Float  = 3.14f
    val d: Double = 3.141592653589793

    println(f"Float  : $f%.6f")
    println(f"Double : $d%.15f")
    println(s"Float  min positive : ${Float.MinPositiveValue}")
    println(s"Double min positive : ${Double.MinPositiveValue}")

    val nan = Double.NaN
    val inf = Double.PositiveInfinity
    println(s"NaN: $nan, Infinity: $inf")
    println(s"NaN == NaN: ${nan == nan}")

end main
```

`Double` is the default floating-point type and should be preferred  
in almost all cases because it provides fifteen significant decimal  
digits of precision. `Float` requires an `f` or `F` suffix on its  
literals and offers only seven digits of precision, but consumes half  
the memory. IEEE 754 specifies the special values `NaN`,  
`PositiveInfinity`, and `NegativeInfinity`; notably `NaN` does not  
equal itself, a property that can surprise unwary developers.  

## BigInt

`BigInt` holds integers of arbitrary magnitude, limited only by  
available memory.  

```scala
@main def main() =

    val a = BigInt("123456789012345678901234567890")
    val b = BigInt(2).pow(100)

    println(a + b)
    println(a * b)
    println(b.isProbablePrime(10))
    println((a + 1) % a)

end main
```

`BigInt` wraps `java.math.BigInteger` and provides all standard  
arithmetic operators through Scala's implicit conversion mechanism,  
so you can write `a + b` instead of `a.add(b)`. The `pow` method  
computes exact integer exponentiation without floating-point  
rounding. `isProbablePrime` runs the Miller-Rabin primality test with  
a configurable certainty level. Use `BigInt` when working with  
cryptographic keys, combinatorics, or any domain where `Long`  
overflow is a real concern.  

## BigDecimal

`BigDecimal` represents decimal numbers with configurable precision  
and avoids the rounding errors inherent in binary floating-point.  

```scala
@main def main() =

    val price = BigDecimal("19.99")
    val tax   = BigDecimal("0.07")
    val total = price + price * tax

    println(f"Price : $price%.2f")
    println(f"Tax   : ${price * tax}%.2f")
    println(f"Total : $total%.4f")

    val a = 0.1 + 0.2
    val b = BigDecimal("0.1") + BigDecimal("0.2")
    println(s"Double    : $a")
    println(s"BigDecimal: $b")

end main
```

Binary floating-point cannot represent 0.1 exactly, so `0.1 + 0.2`  
with `Double` produces a result slightly above 0.3. `BigDecimal`  
stores the decimal representation directly and performs arithmetic  
without binary rounding. It wraps `java.math.BigDecimal` and supports  
the same rich set of arithmetic operators as `BigInt`. Financial  
applications, tax calculations, and any code requiring exact decimal  
results should always use `BigDecimal` instead of `Double`.  

## Numeric literals

Scala 3 supports multiple literal formats that improve readability  
and expressiveness for different numeric domains.  

```scala
@main def main() =

    val decimal   = 1_000_000
    val hex       = 0xFF
    val longVal   = 42L
    val floatVal  = 1.5f
    val doubleVal = 1.5e10
    val negExp    = 2.5e-3

    println(s"decimal   = $decimal")
    println(s"hex       = $hex")
    println(s"long      = $longVal")
    println(s"float     = $floatVal")
    println(s"double e  = $doubleVal")
    println(s"neg exp   = $negExp")

end main
```

Integer literals can be written in decimal or hexadecimal (prefixed  
with `0x` or `0X`). Underscores may appear anywhere inside a numeric  
literal to group digits for readability, such as separating thousands  
or bytes. A trailing `L` promotes an integer literal to `Long`. The  
suffix `f` or `F` marks a `Float` literal, while `e` or `E` with an  
optional sign denotes scientific notation for `Double`. Scala 3  
dropped octal literals from older versions, so a leading zero no  
longer has special meaning.  

## Numeric operations

The standard arithmetic and bitwise operators work uniformly across  
all numeric types in Scala 3.  

```scala
@main def main() =

    val a = 17
    val b = 5

    println(s"add      : ${a + b}")
    println(s"subtract : ${a - b}")
    println(s"multiply : ${a * b}")
    println(s"divide   : ${a / b}")
    println(s"modulo   : ${a % b}")
    println(s"power    : ${math.pow(a, b).toLong}")

    println(s"bitAnd   : ${a & b}")
    println(s"bitOr    : ${a | b}")
    println(s"bitXor   : ${a ^ b}")
    println(s"leftSh   : ${a << 2}")
    println(s"rightSh  : ${a >> 2}")
    println(s"unsignSh : ${a >>> 2}")

end main
```

Integer division truncates toward zero, so `17 / 5` yields `3` and  
the remainder `17 % 5` yields `2`. The bitwise operators `&`, `|`,  
`^`, `~`, `<<`, `>>`, and `>>>` behave identically to their Java  
and C counterparts. The `>>>` operator performs an unsigned right  
shift, filling the high-order bits with zeros regardless of the sign  
bit. Scala does not provide an exponentiation operator; use  
`math.pow` for floating-point exponentiation or `BigInt.pow` for  
exact integer exponentiation.  

## Overflow and underflow

Integer arithmetic in Scala silently wraps around at the boundaries  
of each type without throwing an exception.  

```scala
@main def main() =

    val maxInt = Int.MaxValue
    val minInt = Int.MinValue

    println(s"Int.MaxValue  = $maxInt")
    println(s"MaxValue + 1  = ${maxInt + 1}")
    println(s"Int.MinValue  = $minInt")
    println(s"MinValue - 1  = ${minInt - 1}")

    val maxLong = Long.MaxValue
    println(s"Long.MaxValue = $maxLong")
    println(s"MaxValue + 1  = ${maxLong + 1L}")

    val safe = BigInt(Int.MaxValue) + 1
    println(s"BigInt safe   = $safe")

end main
```

Scala inherits the JVM's two's-complement wraparound semantics for  
integer overflow. When `Int.MaxValue` is incremented by one, the  
result wraps to `Int.MinValue` silently and without any runtime  
exception or warning. The same behaviour applies to `Byte`, `Short`,  
and `Long`. This can introduce subtle bugs in production code. When  
overflow is a concern, either switch to `Long` for a larger range,  
use `Math.addExact` which throws `ArithmeticException` on overflow,  
or use `BigInt` for unlimited range.  

## Type inference for numbers

The Scala compiler infers the most appropriate numeric type from the  
literal value and surrounding context.  

```scala
@main def main() =

    val a = 42          // Int
    val b = 42L         // Long
    val c = 3.14        // Double
    val d = 3.14f       // Float
    val e = 42: Short   // Short via ascription

    println(a.getClass.getSimpleName)
    println(b.getClass.getSimpleName)
    println(c.getClass.getSimpleName)
    println(d.getClass.getSimpleName)
    println(e.getClass.getSimpleName)

    def square(x: Double): Double = x * x
    val result = square(10)   // Int widened to Double
    println(result)

end main
```

Without a suffix or annotation, an integer literal is inferred as  
`Int` and a floating-point literal as `Double`. Adding an `L` suffix  
narrows the inference to `Long`, and `f` to `Float`. A type  
ascription written after the value, such as `42: Short`, forces the  
compiler to treat the literal as the specified type. When a numeric  
value is passed to a method expecting a wider type, the compiler  
inserts an implicit widening conversion automatically, so `square(10)`  
compiles without an explicit `10.toDouble`.  

## Numeric conversions

Every numeric type provides explicit conversion methods that widen  
or narrow the value.  

```scala
@main def main() =

    val i: Int    = 1000
    val b: Byte   = i.toByte    // narrowing — may lose data
    val s: Short  = i.toShort
    val l: Long   = i.toLong    // widening — always safe
    val f: Float  = i.toFloat
    val d: Double = i.toDouble

    println(s"Byte   : $b")
    println(s"Short  : $s")
    println(s"Long   : $l")
    println(s"Float  : $f")
    println(s"Double : $d")

    val pi = 3.14159
    println(s"truncate : ${pi.toInt}")
    println(s"round    : ${math.round(pi)}")

end main
```

Widening conversions such as `Int` to `Long` or `Int` to `Double`  
are always safe and preserve the exact value. Narrowing conversions  
such as `Int` to `Byte` silently discard the high-order bits, which  
may change the value significantly if it lies outside the target  
range. Scala requires narrowing conversions to be explicit, preventing  
accidental data loss that Java allows implicitly. Converting a  
floating-point value to an integer with `toInt` truncates toward  
zero; use `math.round` to obtain the nearest integer instead.  

## Math functions

The `scala.math` package exposes common mathematical functions  
available through the `math` object.  

```scala
@main def main() =

    println(math.abs(-42))
    println(math.sqrt(144.0))
    println(math.pow(2.0, 10.0))
    println(math.log(math.E))
    println(math.log10(1000.0))
    println(math.sin(math.Pi / 2))
    println(math.ceil(2.3))
    println(math.floor(2.9))
    println(math.max(17, 42))
    println(math.min(17, 42))
    println(math.hypot(3.0, 4.0))

end main
```

`scala.math` is an object alias for `java.lang.Math`, so all  
functions delegate to native JVM implementations and benefit from  
hardware optimisations on most platforms. `math.sqrt` and `math.pow`  
accept and return `Double`. `math.abs` is overloaded for `Int`,  
`Long`, `Float`, and `Double`. Note that `math.abs(Int.MinValue)`  
returns `Int.MinValue` unchanged because the mathematical absolute  
value exceeds `Int.MaxValue`; this is a well-known edge case.  
Constants such as `math.Pi` and `math.E` are available as  
`Double` precision values.  

## Comparing numeric types

Numeric values are compared using relational operators that return  
`Boolean` and follow standard mathematical ordering.  

```scala
@main def main() =

    val a = 10
    val b = 20

    println(a < b)
    println(a > b)
    println(a <= 10)
    println(a >= 10)
    println(a == b)
    println(a != b)

    val nums = List(5, 1, 9, 3, 7)
    println(nums.sorted)
    println(nums.max)
    println(nums.min)

    val x = 0.1 + 0.2
    println(math.abs(x - 0.3) < 1e-9)   // floating-point equality

end main
```

Relational operators on numeric types produce `Boolean` results and  
can be composed with logical operators. Direct equality comparison  
with `==` is exact for integers but problematic for floating-point  
values due to rounding. The standard idiom for floating-point  
equality is to check that the absolute difference is smaller than a  
small tolerance, often called epsilon. The `Ordered` and `Ordering`  
type classes from the standard library allow numeric types to  
integrate with sorting and comparison APIs such as `List.sorted`.  

## Mixing numeric types

When operands of different numeric types appear in one expression,  
the compiler applies implicit widening to a common type.  

```scala
@main def main() =

    val i: Int    = 10
    val l: Long   = 100L
    val f: Float  = 1.5f
    val d: Double = 2.5

    val r1 = i + l    // Int + Long   => Long
    val r2 = l + f    // Long + Float => Float
    val r3 = f + d    // Float + Double => Double
    val r4 = i * d    // Int * Double => Double

    println(r1.getClass.getSimpleName)
    println(r2.getClass.getSimpleName)
    println(r3.getClass.getSimpleName)
    println(r4.getClass.getSimpleName)

end main
```

The numeric promotion rules in Scala mirror those of Java: when two  
operands differ in type, the narrower type is widened to the broader  
one before the operation. The widening order is `Byte` → `Short` →  
`Int` → `Long` → `Float` → `Double`. The result type of the  
expression therefore reflects the broader of the two operands. This  
promotion happens implicitly and silently, so mixing `Int` and  
`Double` in an expression yields `Double` without any explicit cast.  

## Boolean literals

The `Boolean` type has exactly two literal values: `true` and `false`.  

```scala
@main def main() =

    val isOpen:   Boolean = true
    val isClosed: Boolean = false
    val isEnabled          = true     // inferred

    println(isOpen)
    println(isClosed)
    println(isEnabled)

    val flag = if isOpen then "open" else "closed"
    println(flag)

    val b: java.lang.Boolean = isOpen  // autoboxing
    println(b.getClass)

end main
```

`Boolean` is a value type that maps to the JVM `boolean` primitive.  
It is the only type accepted by `if`, `while`, and logical operators,  
making it impossible to accidentally use an integer as a truthy value  
the way C permits. When a `Boolean` is stored in a generic collection  
or passed where `java.lang.Object` is expected, the compiler  
automatically boxes it to `java.lang.Boolean`. The two values `true`  
and `false` are keywords, not identifiers, and cannot be shadowed.  

## Boolean operations

Scala 3 provides logical operators for conjunction, disjunction,  
and negation of `Boolean` values.  

```scala
@main def main() =

    val a = true
    val b = false

    println(a && b)    // logical AND
    println(a || b)    // logical OR
    println(!a)        // logical NOT
    println(a ^ b)     // logical XOR

    val x = 5
    println(x > 0 && x < 10)
    println(x < 0 || x > 3)
    println(!(x == 5))

end main
```

The `&&` operator returns `true` only when both operands are `true`.  
The `||` operator returns `true` when at least one operand is `true`.  
Unary `!` negates a boolean value. The `^` operator performs  
exclusive-or and returns `true` when exactly one operand is `true`.  
These operators can be combined to express complex conditions;  
parentheses clarify evaluation order when multiple operators appear  
in one expression. All boolean operators produce a `Boolean` result,  
which can itself be stored in a `val` and reused.  

## Short-circuit evaluation

The `&&` and `||` operators do not evaluate their right operand if  
the result is already determined by the left operand alone.  

```scala
@main def main() =

    def check(name: String, value: Boolean): Boolean =
        println(s"  evaluating $name")
        value

    println("AND short-circuit:")
    val r1 = check("left=false", false) && check("right", true)
    println(r1)

    println("OR short-circuit:")
    val r2 = check("left=true", true) || check("right", false)
    println(r2)

    val s: String = null
    val safe = s != null && s.length > 0
    println(safe)

end main
```

When the left operand of `&&` is `false`, the entire expression is  
`false` regardless of the right side, so Scala skips evaluation of  
the right operand entirely. Similarly, when the left operand of `||`  
is `true`, the right side is not evaluated. This property enables  
safe null-guard patterns where the right side would throw if the  
left guard failed, as shown by the `s != null && s.length > 0`  
example. It also improves performance by avoiding expensive  
computations that are unnecessary.  

## Char literals

The `Char` type represents a single Unicode code unit stored as an  
unsigned 16-bit value.  

```scala
@main def main() =

    val letter:  Char = 'A'
    val digit:   Char = '7'
    val space:   Char = ' '
    val tab:     Char = '\t'
    val newline: Char = '\n'
    val quote:   Char = '\''
    val backsl:  Char = '\\'

    println(letter)
    println(digit)
    println(quote)
    println(backsl)
    println(s"Tab code: ${tab.toInt}")

end main
```

`Char` literals are enclosed in single quotes and may contain a  
single character or one of the standard escape sequences: `\t` (tab),  
`\n` (newline), `\r` (carriage return), `\\` (backslash), `\'`  
(single quote), `\"` (double quote), and `\0` (null character). The  
`Char` type maps to the JVM `char` primitive, which holds a single  
UTF-16 code unit. Characters outside the Basic Multilingual Plane  
require two `Char` values (a surrogate pair), so be cautious when  
iterating over strings containing emoji or rare Unicode characters.  

## Unicode characters

`Char` values can be expressed as Unicode escape sequences using  
the `\uXXXX` notation directly in source code.  

```scala
@main def main() =

    val copyright: Char = '\u00A9'
    val euro:      Char = '\u20AC'
    val omega:     Char = '\u03A9'
    val smiley:    Char = '\u263A'

    println(copyright)
    println(euro)
    println(omega)
    println(smiley)

    val greeting = "\u0048\u0065\u006C\u006C\u006F"
    println(greeting)

end main
```

A Unicode escape `\uXXXX` is replaced by the corresponding UTF-16  
code unit during lexical analysis, before any other processing. This  
means Unicode escapes work everywhere a character or string literal  
is valid in Scala source. The four hexadecimal digits identify the  
code point within the Basic Multilingual Plane. Characters such as  
the copyright sign `©`, the euro sign `€`, and Greek letters can  
be embedded in string literals directly when the source file is  
saved in UTF-8, making Unicode escapes mainly useful for characters  
that are difficult to type or display in the editor.  

## Char operations

The `Char` type provides a rich set of methods for classification  
and transformation inherited from `java.lang.Character`.  

```scala
@main def main() =

    val ch = 'g'

    println(ch.isLetter)
    println(ch.isDigit)
    println(ch.isWhitespace)
    println(ch.isUpper)
    println(ch.isLower)
    println(ch.toUpper)
    println(ch.toLower)
    println(ch.isLetterOrDigit)

    val sentence = "Hello, World 123!"
    val letters  = sentence.count(_.isLetter)
    val digits   = sentence.count(_.isDigit)
    println(s"letters=$letters, digits=$digits")

end main
```

Methods like `isLetter`, `isDigit`, `isWhitespace`, `isUpper`, and  
`isLower` delegate to `java.lang.Character` and correctly handle the  
full Unicode range, not just ASCII. `toUpper` and `toLower` perform  
Unicode-aware case conversion. Because `String` is a sequence of  
`Char` values, all collection operations such as `filter`, `map`,  
`count`, and `exists` are available on strings and operate character  
by character, making it straightforward to validate or transform  
individual characters without an explicit loop.  

## Char and Int conversion

Every `Char` has a numeric code-point value and can be converted  
to and from `Int` explicitly.  

```scala
@main def main() =

    val ch   = 'A'
    val code = ch.toInt
    println(s"'$ch' -> $code")

    val back = code.toChar
    println(s"$code -> '$back'")

    for i <- 65 to 90 do
        print(i.toChar + " ")
    println()

    val encrypted = "Hello".map(c => (c + 1).toChar)
    println(encrypted)

end main
```

`Char.toInt` returns the UTF-16 code unit value as a non-negative  
`Int`. Conversely, `Int.toChar` casts the low sixteen bits of the  
integer to a `Char`. These conversions enable simple cipher  
algorithms — the Caesar cipher example shifts each character by one  
position in the Unicode table. Iterating over a numeric range and  
converting each value to `Char` is a concise way to generate  
alphabets or symbol tables. Be aware that not every `Int` in the  
range 0–65535 maps to a valid Unicode character; some code points  
are reserved surrogates.  

## Basic strings

`String` in Scala is an alias for `java.lang.String` and represents  
an immutable sequence of UTF-16 characters.  

```scala
@main def main() =

    val name  = "Scala"
    val empty = ""
    val num   = 42.toString

    println(name)
    println(name.length)
    println(name.charAt(0))
    println(name.substring(1, 4))
    println(name + " 3")
    println(name.reverse)
    println(name.toLowerCase)
    println(name.toUpperCase)
    println(empty.isEmpty)

end main
```

`String` literals are enclosed in double quotes and support the same  
escape sequences as `Char` literals. Because `String` is immutable,  
every method that appears to modify a string actually returns a new  
`String` object while the original remains unchanged. String  
concatenation with `+` is convenient for small numbers of operands;  
for building long strings from many pieces, prefer `StringBuilder`  
or `String.format` to avoid creating many intermediate objects.  
Scala also adds extension methods to `java.lang.String`, making  
collection operations like `filter` and `map` available directly.  

## Multiline strings

Triple-quoted string literals can span multiple lines without  
embedded escape sequences.  

```scala
@main def main() =

    val poem =
        """Roses are red,
          |Violets are blue,
          |Scala is awesome,
          |And so are you.""".stripMargin

    println(poem)

    val json =
        """
          |{
          |  "name": "Alice",
          |  "age": 30
          |}
          |""".stripMargin.trim

    println(json)

end main
```

A triple-quoted string begins with `"""` and ends with `"""` and  
preserves all characters between the delimiters verbatim, including  
newlines, tabs, and backslashes, without escape sequences. This  
makes it ideal for embedding code snippets, JSON templates, SQL  
queries, and prose. The pipe character `|` acts as a margin marker  
when combined with `stripMargin`, which removes all leading  
whitespace up to and including the first `|` on each line. The  
resulting string is aligned to the left margin, making the  
indentation in source code purely for readability.  

## String interpolation

Scala 3 offers three built-in string interpolators: `s`, `f`,  
and `raw`.  

```scala
@main def main() =

    val name  = "Alice"
    val score = 95.75
    val items = List("pen", "book")

    val s1 = s"Name: $name, Score: $score"
    val s2 = f"Formatted: $name%-10s score: $score%.2f"
    val s3 = raw"Raw: \n is not a newline here"
    val s4 = s"Items: ${items.mkString(", ")}"
    val s5 = s"Doubled: ${score * 2}"

    println(s1)
    println(s2)
    println(s3)
    println(s4)
    println(s5)

end main
```

The `s` interpolator substitutes `$identifier` and `${expression}`  
placeholders by calling `toString` on each value. The `f` interpolator  
extends `s` with `printf`-style format specifiers placed after the  
placeholder, such as `%.2f` for two decimal places or `%-10s` for a  
left-aligned ten-character field. The `raw` interpolator performs  
substitution like `s` but does not interpret escape sequences, which  
is useful for regular expression patterns and Windows file paths.  
Arbitrary expressions can appear inside `${}`, including method  
calls, arithmetic, and collection operations.  

## String conversions

Scala provides a uniform API for converting between `String` and  
other types.  

```scala
@main def main() =

    val n    = 42.toString
    val pi   = 3.14.toString
    val flag = true.toString
    println(n.getClass.getSimpleName)

    val i = "42".toInt
    val d = "3.14".toDouble
    val l = "1000000".toLong
    val b = "true".toBoolean

    println(i + 1)
    println(d + 0.01)
    println(l * 2)
    println(!b)

    val opt = "abc".toIntOption
    println(opt)

end main
```

Every Scala type inherits a `toString` method from `Any`, so any  
value can be converted to its string representation without  
additional imports. In the other direction, `String` provides  
`toInt`, `toDouble`, `toLong`, `toFloat`, `toByte`, `toShort`, and  
`toBoolean` methods that parse the string and return the  
corresponding value. If the string does not represent a valid value,  
these methods throw a `NumberFormatException`. The safer  
alternatives `toIntOption`, `toDoubleOption`, and similar methods  
return `Option[T]` instead, yielding `None` when parsing fails.  

## String parsing

Splitting and parsing strings are common operations when processing  
text data.  

```scala
@main def main() =

    val csv   = "Alice,30,engineer"
    val parts = csv.split(",")
    println(parts.toList)

    val (name, age, role) = parts match
        case Array(n, a, r) => (n, a.toInt, r)
        case _              => ("unknown", 0, "unknown")

    println(s"name=$name, age=$age, role=$role")

    val sentence = "  Hello   World   "
    println(sentence.trim.split("\\s+").toList)

    "2024-05-15".split("-") match
        case Array(y, m, d) => println(s"year=$y month=$m day=$d")
        case _              => println("invalid date")

end main
```

`String.split` accepts a regular expression pattern and returns an  
`Array[String]` of the tokens. Passing `"\\s+"` splits on any run  
of whitespace, which is more robust than splitting on a single  
space. Combining `split` with pattern matching on the resulting  
`Array` provides a concise and type-safe way to destructure  
structured text such as CSV records or ISO date strings. The `trim`  
method removes leading and trailing whitespace before splitting,  
preventing empty tokens at the boundaries.  

## String comparison

Strings in Scala are compared for equality with `==` and for  
lexicographic order with `compareTo` and relational operators.  

```scala
@main def main() =

    val a = "Hello"
    val b = "hello"
    val c = "Hello"

    println(a == c)
    println(a == b)
    println(a.equalsIgnoreCase(b))
    println(a.compareTo(b))
    println(a < b)
    println(a.startsWith("He"))
    println(a.endsWith("lo"))
    println(a.contains("ell"))

    val words = List("banana", "apple", "cherry")
    println(words.sorted)
    println(words.sortBy(_.length))

end main
```

Scala's `==` on strings calls `equals` under the hood, so it  
compares content rather than reference identity, unlike Java's `==`  
which compares object references. `equalsIgnoreCase` performs a  
case-insensitive equality check. `compareTo` returns a negative  
integer, zero, or positive integer to indicate whether the receiver  
is lexicographically less than, equal to, or greater than the  
argument. This makes `String` usable anywhere an `Ordering[String]`  
is required, including collection sort operations.  

## Strings and regex

Scala provides first-class support for regular expressions through  
the `Regex` class and the `.r` extension method.  

```scala
@main def main() =

    val email   = "user@example.com"
    val pattern = """[\w.]+@[\w.]+\.\w+""".r

    pattern.findFirstIn(email) match
        case Some(m) => println(s"matched: $m")
        case None    => println("no match")

    val digits = """\d+""".r
    val text   = "Order 123 has 45 items"
    val found  = digits.findAllIn(text).toList
    println(found)

    val date   = "2024-05-15"
    val dateRe = """(\d{4})-(\d{2})-(\d{2})""".r
    date match
        case dateRe(y, m, d) => println(s"$y/$m/$d")
        case _               => println("no match")

end main
```

The `.r` method on any string converts it to a  
`scala.util.matching.Regex` object. Triple-quoted strings are ideal  
for regex patterns because backslashes do not need to be escaped.  
`findFirstIn` returns an `Option[String]` with the first match and  
`findAllIn` returns an iterator over all non-overlapping matches.  
A `Regex` can also be used directly in a `match` expression;  
capturing groups become the pattern variables, enabling clean and  
readable data extraction from structured text.  

## Strings and collections

A `String` in Scala implements `IndexedSeq[Char]`, making all  
collection operations available directly on strings.  

```scala
@main def main() =

    val text = "Hello, Scala 3!"

    println(text.filter(_.isLetter))
    println(text.map(_.toUpper))
    println(text.count(_.isDigit))
    println(text.exists(_.isDigit))
    println(text.forall(_.isLetter))
    println(text.take(5))
    println(text.drop(7))
    println(text.toList.distinct.sorted.mkString)

end main
```

Because `String` is implicitly convertible to `WrappedString`, which  
extends `IndexedSeq[Char]`, you can apply `map`, `filter`, `fold`,  
`zip`, `grouped`, and all other collection methods directly to a  
string. The result type depends on the operation: `filter` and `map`  
return a `String` when the transformation yields `Char` values, while  
`toList` returns `List[Char]`. Chaining `distinct`, `sorted`, and  
`mkString` extracts unique characters and reassembles them into a  
sorted string, demonstrating how string and collection APIs compose  
naturally.  

## List

`List` is an immutable singly-linked sequence that supports efficient  
head access and prepend operations.  

```scala
@main def main() =

    val nums  = List(1, 2, 3, 4, 5)
    val words = List("apple", "banana", "cherry")
    val empty: List[Int] = Nil

    println(nums.head)
    println(nums.tail)
    println(0 :: nums)
    println(nums :+ 6)
    println(nums ++ List(6, 7))
    println(nums.map(_ * 2))
    println(nums.filter(_ % 2 == 0))
    println(nums.foldLeft(0)(_ + _))

end main
```

`List` is the workhorse of functional Scala. Prepending with `::` is  
O(1) because the new cell simply points to the existing list as its  
tail. Appending with `:+` is O(n) because the entire list must be  
traversed to reach the end. The `Nil` object represents the empty  
list and is the base case for recursive list processing. `map`,  
`filter`, and `foldLeft` are the fundamental operations that  
transform and aggregate list contents without mutation, making  
`List` ideal for functional pipelines.  

## Vector

`Vector` is an immutable indexed sequence that provides effectively  
constant-time random access and update operations.  

```scala
@main def main() =

    val v = Vector(10, 20, 30, 40, 50)

    println(v(2))
    println(v.updated(2, 99))
    println(v.prepended(0))
    println(v.appended(60))
    println(v.slice(1, 4))
    println(v.indexOf(30))
    println(v.reverse)

end main
```

`Vector` is implemented as a persistent bit-partitioned trie with a  
branching factor of 32. Random access and functional update via  
`updated` run in O(log₃₂ n), which is effectively constant for  
realistic collection sizes. Unlike `List`, `Vector` performs well  
for both head-first and tail-first access patterns, making it the  
preferred default immutable sequence when the access pattern is not  
exclusively head-focused. `prepended` and `appended` both run in  
effectively constant time, so `Vector` is well-suited for queues.  

## Array

`Array` is a mutable, fixed-size collection that maps directly to a  
JVM array for maximum performance.  

```scala
@main def main() =

    val arr = Array(1, 2, 3, 4, 5)
    arr(2) = 99
    println(arr.toList)

    val zeros = Array.fill(5)(0)
    val range = Array.range(1, 6)
    val tabul = Array.tabulate(5)(i => i * i)

    println(zeros.toList)
    println(range.toList)
    println(tabul.toList)

    java.util.Arrays.sort(arr)
    println(arr.toList)

end main
```

`Array` is the only mutable collection type in Scala's standard  
collection hierarchy that maps directly to a JVM typed array.  
An `Array[Int]` compiles to an `int[]` on the JVM, avoiding the  
boxing overhead of `Array[java.lang.Integer]`. Elements are accessed  
and mutated with `apply` and `update` methods, which the compiler  
desugars from the familiar `arr(i)` and `arr(i) = value` syntax.  
`Array.fill`, `Array.range`, and `Array.tabulate` are convenient  
factory methods for initialising arrays without explicit loops.  

## Set

`Set` is an immutable collection of distinct elements with no  
guaranteed ordering.  

```scala
@main def main() =

    val s1 = Set(1, 2, 3, 4, 5)
    val s2 = Set(3, 4, 5, 6, 7)

    println(s1.contains(3))
    println(s1 + 6)
    println(s1 - 2)
    println(s1 union s2)
    println(s1 intersect s2)
    println(s1 diff s2)
    println(s1.size)

    val words = List("apple", "apple", "banana", "cherry", "banana")
    println(words.toSet)

end main
```

`Set` guarantees that each element appears at most once, making it  
the right choice whenever uniqueness is a business constraint. The  
default `Set` implementation uses a hash-based structure and  
provides O(1) amortised `contains`, `+`, and `-` operations. Set  
operations `union`, `intersect`, and `diff` correspond to the  
mathematical set operations ∪, ∩, and \\. Converting a `List` with  
duplicates to a `Set` is a concise way to deduplicate a collection.  
For ordering guarantees, use `SortedSet` from the standard library.  

## Map

`Map` stores key-value pairs and provides efficient key-based lookup  
through a hash table.  

```scala
@main def main() =

    val capitals = Map(
        "France" -> "Paris",
        "Japan"  -> "Tokyo",
        "Brazil" -> "Brasília"
    )

    println(capitals("France"))
    println(capitals.get("Germany"))
    println(capitals.getOrElse("Germany", "unknown"))
    println(capitals + ("Germany" -> "Berlin"))
    println(capitals - "Japan")
    println(capitals.keys.toList)
    println(capitals.map((k, v) => k -> v.length))

end main
```

The default `Map` is immutable and hash-based, giving O(1) average  
performance for `apply`, `get`, `+`, and `-`. The `->` operator  
creates a `Tuple2` pair that `Map` uses as a key-value entry.  
Calling `apply` with a missing key throws `NoSuchElementException`;  
prefer `get` which returns `Option[V]`, or `getOrElse` which returns  
a default value. Adding a key-value pair with `+` returns a new  
`Map` containing the original entries plus the new one. `SortedMap`  
is an alternative when key ordering must be preserved.  

## Seq and Range

`Seq` is the root trait for ordered sequences, and `Range` is an  
efficient representation of arithmetic progressions.  

```scala
@main def main() =

    val seq: Seq[Int] = Seq(10, 20, 30, 40)

    val r1 = 1 to 5
    val r2 = 1 until 5
    val r3 = 1 to 10 by 2
    val r4 = 10 to 1 by -1

    println(r1.toList)
    println(r2.toList)
    println(r3.toList)
    println(r4.toList)

    println(r1.sum)
    println(r1.product)
    println((1 to 1_000_000).sum)   // no List allocated

end main
```

`Seq` is an abstract type that `List`, `Vector`, `Array`, and  
`Range` all implement. Programming to `Seq` in method signatures  
makes code more flexible and composable. `Range` is special: it  
stores only the start, end, and step values rather than  
materialising every element in memory, so `(1 to 1_000_000).sum`  
computes the sum in a tight loop without allocating a  
million-element collection. The `to` operator creates an inclusive  
range and `until` creates an exclusive one.  

## Option

`Option[A]` represents a value that may or may not be present,  
replacing `null` with a type-safe container.  

```scala
@main def main() =

    val some: Option[Int] = Some(42)
    val none: Option[Int] = None

    println(some.isDefined)
    println(none.isEmpty)
    println(some.getOrElse(0))
    println(none.getOrElse(0))

    val doubled = some.map(_ * 2)
    println(doubled)

    some match
        case Some(v) => println(s"got $v")
        case None    => println("empty")

    val nums = List(Some(1), None, Some(3), None, Some(5))
    println(nums.flatten)

end main
```

`Option` eliminates null-pointer exceptions by making the  
possibility of absence explicit in the type. `Some(value)` wraps a  
present value, while `None` represents absence. `getOrElse` extracts  
the value or returns a fallback. `map` applies a function to the  
contained value if present and returns `None` if the option is  
empty. `flatMap` chains optional computations that may themselves  
fail. Pattern matching on `Option` is exhaustive-checked by the  
compiler, ensuring both cases are always handled.  

## Either

`Either[L, R]` models a value that is one of two possibilities,  
commonly used to represent success or failure with a meaningful  
error.  

```scala
@main def main() =

    def divide(a: Int, b: Int): Either[String, Double] =
        if b == 0 then Left("Division by zero")
        else Right(a.toDouble / b)

    val r1 = divide(10, 2)
    val r2 = divide(10, 0)

    println(r1)
    println(r2)

    r1 match
        case Right(v) => println(s"result: $v")
        case Left(e)  => println(s"error: $e")

    val result = for
        x <- divide(10, 2)
        y <- divide(x.toInt, 2)
    yield y

    println(result)

end main
```

By convention `Left` holds the error value and `Right` holds the  
success value — "right is right". `Either` is right-biased since  
Scala 2.12, meaning `map` and `flatMap` operate on the `Right` value  
and pass `Left` values through unchanged. This makes `Either`  
composable in `for`-expressions: if any step yields `Left`, the  
entire chain short-circuits and returns that `Left`. Unlike  
`Option`, `Either` carries a meaningful error description instead  
of just signalling absence.  

## Tuples

Tuples are fixed-size, heterogeneous collections whose element types  
are tracked individually in the type signature.  

```scala
@main def main() =

    val t2: (Int, String)          = (1, "one")
    val t3: (String, Int, Boolean) = ("Alice", 30, true)

    println(t2._1)
    println(t2._2)

    val (name, age, active) = t3
    println(s"$name is $age, active=$active")

    val pairs = List((1, "a"), (2, "b"), (3, "c"))
    pairs.foreach((n, s) => println(s"$n -> $s"))

    val swapped = t2.swap
    println(swapped)

end main
```

Tuple types are written with parentheses and commas in Scala 3, and  
elements are accessed via `._1`, `._2`, and so on. The maximum tuple  
arity supported by the standard library is 22. Destructuring  
assignment (`val (name, age, active) = t3`) binds each element to a  
separate `val` in one step. Scala 3 also supports generic tuple  
operations through the `Tuple` trait and type-level tuple  
manipulation. Prefer case classes over large tuples when the  
elements have distinct semantic meanings.  

## Nested collections

Collections can contain other collections, enabling hierarchical  
data representation.  

```scala
@main def main() =

    val matrix: List[List[Int]] =
        List(
            List(1, 2, 3),
            List(4, 5, 6),
            List(7, 8, 9)
        )

    matrix.foreach(row => println(row.mkString(" ")))

    val flat = matrix.flatten
    println(flat)

    val doubled = matrix.map(row => row.map(_ * 2))
    doubled.foreach(row => println(row))

    val mapOfLists: Map[String, List[Int]] =
        Map("odds" -> List(1, 3, 5), "evens" -> List(2, 4, 6))
    println(mapOfLists("odds").sum)

end main
```

Nested collections are typed precisely: `List[List[Int]]` makes it  
clear that each outer element is itself a list of integers. The  
`flatten` method collapses one level of nesting, turning a  
`List[List[A]]` into `List[A]`. `flatMap` combines `map` and  
`flatten` in one step. Nesting collections in a `Map` is a common  
pattern for grouping related values; `groupBy` on a collection  
produces exactly this structure automatically.  

## Unit type

`Unit` is the return type of expressions evaluated solely for their  
side effects, equivalent to `void` in Java.  

```scala
@main def main() =

    val u: Unit = ()
    println(u)
    println(u.getClass)

    def greet(name: String): Unit =
        println(s"Hello, $name!")

    val result = greet("World")
    println(result)

    val units: List[Unit] = List(1, 2, 3).map(_ => ())
    println(units)

end main
```

`Unit` has exactly one value, the unit literal `()`. It is a proper  
type in the Scala type hierarchy, extending `AnyVal`, so it can  
appear as a type argument, be stored in a `val`, and be placed in a  
collection. Functions that return `Unit` are called for their  
side effects and their return value is conventionally discarded.  
Mapping a list with a function that returns `Unit` produces a  
`List[Unit]` of the same length, which is legal but usually  
indicates that `foreach` should have been used instead.  

## Null type

`Null` is the type of the `null` literal and is a subtype of every  
reference type in Scala's type hierarchy.  

```scala
@main def main() =

    val s: String = null
    println(s == null)

    val opt = Option(s)
    println(opt)

    def safeLength(str: String): Int =
        if str == null then 0
        else str.length

    println(safeLength(null))
    println(safeLength("hello"))

    val wrapped = Option(null: String)
    println(wrapped.getOrElse("default"))

end main
```

The `null` literal has type `Null`, which is a subtype of all  
reference types. Assigning `null` to a variable of any reference  
type is legal but considered bad practice in idiomatic Scala. The  
`Option(value)` factory returns `None` if `value` is `null` and  
`Some(value)` otherwise, providing a clean bridge between  
null-returning Java APIs and Scala's `Option`-based style. In  
Scala 3, the `-Yexplicit-nulls` compiler flag makes `Null` no longer  
a subtype of reference types, forcing nullable values to be typed  
explicitly as `T | Null`.  

## Nothing type

`Nothing` is the bottom type that is a subtype of every other type  
and has no values at all.  

```scala
@main def main() =

    def fail(msg: String): Nothing =
        throw RuntimeException(msg)

    def headOrFail(lst: List[Int]): Int =
        if lst.isEmpty then fail("empty list")
        else lst.head

    println(headOrFail(List(1, 2, 3)))

    val result: Int = if true then 42 else fail("unreachable")
    println(result)

end main
```

Because `Nothing` is a subtype of every type, a function returning  
`Nothing` can appear in any position without causing a type error.  
This property is used by `throw` expressions: throwing an exception  
has type `Nothing`, so it can be placed in either branch of an `if`  
or inside a `match` case without widening the result type  
unexpectedly. `Nothing` also serves as the element type of the empty  
collection `List.empty[Nothing]`, which is a subtype of `List[A]`  
for any `A`.  

## Any type

`Any` is the root of the entire Scala type hierarchy and is the  
supertype of both value types and reference types.  

```scala
@main def main() =

    val values: List[Any] = List(1, "hello", true, 3.14, List(1, 2))

    for v <- values do
        v match
            case i: Int     => println(s"Int: $i")
            case s: String  => println(s"String: $s")
            case b: Boolean => println(s"Boolean: $b")
            case d: Double  => println(s"Double: $d")
            case l: List[?] => println(s"List: $l")
            case _          => println(s"Other: $v")

end main
```

`Any` provides three universal methods: `==` for equality, `!=`  
for inequality, and `hashCode` for hash computation. Using `Any` as  
a type weakens static guarantees because the compiler cannot check  
which operations are valid; every operation requires a pattern match  
to recover the concrete type. Nevertheless, `Any` is occasionally  
necessary for heterogeneous collections, logging utilities, and  
reflection-based code. A type parameter `[A]` is more precise than  
`Any` because it preserves the relationship between the caller's  
type and the function's behaviour.  

## AnyVal and AnyRef

`AnyVal` is the supertype of all value types and `AnyRef` is the  
supertype of all reference types in the Scala type hierarchy.  

```scala
@main def main() =

    val intVal:  AnyVal = 42
    val boolVal: AnyVal = true
    val charVal: AnyVal = 'X'
    val unitVal: AnyVal = ()

    val strRef:  AnyRef = "hello"
    val listRef: AnyRef = List(1, 2, 3)

    println(intVal)
    println(boolVal)
    println(strRef)
    println(listRef)

    println(intVal.isInstanceOf[AnyVal])
    println(strRef.isInstanceOf[AnyRef])

end main
```

`AnyVal` subtypes are `Byte`, `Short`, `Int`, `Long`, `Float`,  
`Double`, `Char`, `Boolean`, and `Unit`. They are stored as JVM  
primitives wherever possible, but are boxed to their wrapper types  
when an `AnyVal` reference is required. `AnyRef` is an alias for  
`java.lang.Object` and is the supertype of every class and trait  
defined in Scala or Java. The distinction matters for performance:  
storing many `Int` values in an `Array[Int]` avoids boxing, while  
an `Array[AnyVal]` would box every element.  

## Bottom and top types

`Nothing` and `Null` are the bottom types; `Any` is the top type  
that unifies the entire Scala type hierarchy.  

```scala
@main def main() =

    // Nothing is a subtype of every type
    val x: Int = if false then 0 else throw RuntimeException("!")

    // Null is a subtype of reference types only
    val s: String = null

    def safeDiv(a: Int, b: Int): Either[Nothing, Int] =
        Right(a / b)

    println(safeDiv(10, 2))

    // Any is the supertype of everything
    def describe(v: Any): String =
        v match
            case av: AnyVal => s"value type: $av"
            case ar: AnyRef => s"ref type: ${ar.getClass.getSimpleName}"

    println(describe(42))
    println(describe("hello"))
    println(describe(()))

end main
```

`Nothing` is a subtype of every type, so a `throw` expression fits  
anywhere without widening the surrounding type. `Null` is only a  
subtype of reference types (those extending `AnyRef`), mirroring the  
JVM's design where only object references can be null. `Any` sits at  
the apex of the hierarchy and is the common supertype of both  
`AnyVal` and `AnyRef`. The three-level structure `Any` /  
`AnyVal` + `AnyRef` / concrete types is the conceptual backbone of  
the Scala type system and must be understood before working with  
generics or reflection.  

## Type aliases

A type alias introduces an alternative name for an existing type,  
improving readability without creating a new type.  

```scala
type Name       = String
type Age        = Int
type Score      = Double
type Pair[A, B] = (A, B)
type StringPair = Pair[String, String]
type StudentMap = Map[Name, Age]

@main def main() =

    val student: StudentMap        = Map("Alice" -> 30, "Bob" -> 25)
    val grade: Pair[Name, Score]   = ("Alice", 95.5)
    val coords: Pair[Int, Int]     = (10, 20)

    println(student)
    println(grade)
    println(coords)

end main
```

Type aliases are transparent to the compiler: `Name` and `String`  
are interchangeable and the compiler treats them as the same type.  
They improve code readability by giving semantically meaningful names  
to types without adding runtime overhead. Parameterised aliases like  
`Pair[A, B]` work the same way as concrete ones and can reduce  
repetition in complex generic signatures. Unlike opaque types,  
aliases do not enforce any abstraction boundary.  

## Opaque types

An opaque type alias creates a new type that is distinct from its  
underlying representation outside the defining scope.  

```scala
object Domain:
    opaque type UserId = Int
    opaque type Email  = String

    object UserId:
        def apply(n: Int): UserId  = n
        def value(id: UserId): Int = id

    object Email:
        def apply(s: String): Email  = s
        def value(e: Email): String  = e

@main def main() =

    import Domain.*

    val id    = UserId(42)
    val email = Email("alice@example.com")

    println(UserId.value(id))
    println(Email.value(email))

    // val bad: UserId = 99  // compile error: Int is not UserId

end main
```

Opaque types prevent callers from treating a `UserId` as a plain  
`Int` or accidentally passing an `Email` where a `UserId` is  
expected, eliminating a whole class of type-confusion bugs without  
any runtime cost. Inside the companion object that defines the  
opaque type, the alias is transparent and arithmetic or string  
operations work normally. Outside the defining scope, the underlying  
type is hidden and only the API exposed by the companion is  
available. This makes opaque types the preferred tool for  
newtype-style programming in Scala 3.  

## Enums as data types

Scala 3 enums define a closed set of named values or parameterised  
cases that the compiler knows exhaustively.  

```scala
enum Color:
    case Red, Green, Blue

enum Shape:
    case Circle(radius: Double)
    case Rectangle(width: Double, height: Double)

    def area: Double = this match
        case Circle(r)       => math.Pi * r * r
        case Rectangle(w, h) => w * h

@main def main() =

    val c = Color.Red
    println(c)
    println(c.ordinal)
    println(Color.values.toList)

    val s = Shape.Circle(5.0)
    println(s.area)
    println(Shape.Rectangle(3, 4).area)

end main
```

Simple enum cases like `Color.Red` are singletons and carry an  
automatically generated `ordinal` field and a `values` array for  
iteration. Parameterised cases like `Shape.Circle` behave like  
lightweight case classes and can carry fields and methods. Defining  
methods inside the enum body allows each case to provide its own  
behaviour through pattern matching on `this`. The compiler performs  
exhaustiveness checking on pattern matches against enum types,  
eliminating the possibility of unhandled cases.  

## Case classes as data types

Case classes are immutable value-object types whose equality,  
hashing, and pattern matching are derived automatically by the  
compiler.  

```scala
case class Point(x: Double, y: Double):
    def distance(other: Point): Double =
        math.hypot(x - other.x, y - other.y)

case class Person(name: String, age: Int)

@main def main() =

    val p1 = Point(0, 0)
    val p2 = Point(3, 4)
    println(p1.distance(p2))

    val alice  = Person("Alice", 30)
    val alice2 = alice.copy(age = 31)
    println(alice)
    println(alice2)
    println(alice == Person("Alice", 30))

    alice match
        case Person(n, a) => println(s"$n is $a")

end main
```

The compiler generates `equals`, `hashCode`, `toString`, `copy`, and  
`unapply` for every case class, making them immediately usable in  
collections and pattern matching. The `copy` method creates a new  
instance with selected fields changed, which is the idiomatic way  
to "modify" an immutable case class. Case classes model domain  
entities such as `Point`, `Person`, `Order`, and `Event` with  
minimal boilerplate. They are the building blocks of algebraic data  
types (ADTs) when combined with `sealed` traits or enums.  

## Union types

A union type `A | B` represents a value that can be of type `A`  
or type `B`, without requiring a common supertype.  

```scala
@main def main() =

    type StringOrInt = String | Int

    def describe(value: StringOrInt): String =
        value match
            case s: String => s"string of length ${s.length}"
            case i: Int    => s"integer $i"

    println(describe("hello"))
    println(describe(42))

    def parse(input: String): Int | String =
        input.toIntOption.getOrElse(input)

    println(parse("123"))
    println(parse("abc"))

    val items: List[Int | String] = List(1, "two", 3, "four")
    println(items)

end main
```

Union types are a Scala 3 feature that eliminates the need for a  
wrapper sealed hierarchy when a value legitimately belongs to one  
of several unrelated types. The compiler ensures pattern matching  
on a union type is exhaustive: if a case for one of the union  
members is missing, the compiler emits a warning. Unlike `Any`, a  
union type carries precise type information, so the compiler can  
report an error if an operation not shared by both members is  
attempted without a prior pattern match.  

## Intersection types

An intersection type `A & B` represents a value that simultaneously  
satisfies both type `A` and type `B`.  

```scala
trait Flyable:
    def fly(): Unit

trait Swimmable:
    def swim(): Unit

class Duck extends Flyable, Swimmable:
    def fly()  = println("Duck flying")
    def swim() = println("Duck swimming")

def move(animal: Flyable & Swimmable): Unit =
    animal.fly()
    animal.swim()

@main def main() =

    val duck = Duck()
    move(duck)

    val list: List[Flyable & Swimmable] = List(duck)
    list.foreach(_.fly())

end main
```

An intersection type `A & B` is a subtype of both `A` and `B`,  
meaning a value of type `A & B` can be used wherever `A` or `B` is  
expected. Intersection types are dual to union types and are  
especially useful in mixin-style APIs where a function requires  
a value that satisfies multiple independent capabilities. The  
compiler resolves method calls on intersection types by looking up  
the method in each constituent type and verifying there is no  
ambiguity.  

## Literal types

A literal type is a type whose only inhabitant is a specific  
compile-time constant value.  

```scala
@main def main() =

    val one: 1         = 1
    val t: true        = true
    val greeting: "hi" = "hi"

    def toggle(b: true | false): Boolean = !b
    println(toggle(true))

    println(one)
    println(t)
    println(greeting)

    val flag: 42 = 42
    println(flag)

end main
```

Literal types allow the type of a value to be pinned to a specific  
constant such as `1`, `true`, or `"hi"`. They are the most precise  
types possible and are used in type-level programming to carry  
compile-time information through the type system. A function  
accepting `true | false` is more informative than one accepting  
`Boolean` because the union of literal types documents that only  
the two specific boolean constants are valid. Literal types are  
closely related to singleton types and are the foundation for  
type-level computations via match types.  

## Match types

A match type computes a result type by matching a type argument  
against a sequence of type patterns at compile time.  

```scala
type Elem[X] = X match
    case String      => Char
    case List[t]     => t
    case Array[t]    => t

@main def main() =

    val s: Elem[String]        = 'A'
    val l: Elem[List[Int]]     = 42
    val a: Elem[Array[Double]] = 3.14

    println(s)
    println(l)
    println(a)

    type IsInt[X] = X match
        case Int => true
        case _   => false

    val yes: IsInt[Int]    = true
    val no:  IsInt[String] = false
    println(yes)
    println(no)

end main
```

Match types perform type-level computation analogous to value-level  
pattern matching. The compiler evaluates a match type by substituting  
the type argument and selecting the first branch whose pattern covers  
the argument. `Elem[String]` reduces to `Char`, `Elem[List[Int]]`  
reduces to `Int`, and so on. Match types enable type-safe generic  
utilities that return different types depending on their input type,  
without runtime reflection or casting. They are a key ingredient in  
Scala 3's approach to dependently-typed programming.  

## Generic data types

A generic class or trait is parameterised by one or more type  
variables that the caller supplies at instantiation.  

```scala
class Box[A](val value: A):
    def map[B](f: A => B): Box[B] = Box(f(value))
    override def toString: String  = s"Box($value)"

class Pair[A, B](val first: A, val second: B):
    def swap: Pair[B, A]          = Pair(second, first)
    override def toString: String = s"Pair($first, $second)"

@main def main() =

    val intBox = Box(42)
    val strBox = intBox.map(_.toString)
    println(intBox)
    println(strBox)

    val pair    = Pair("Alice", 30)
    val swapped = pair.swap
    println(pair)
    println(swapped)

end main
```

Generic types let you write code that operates uniformly over  
multiple concrete types while preserving static type safety. The  
compiler substitutes the type argument when the generic class is  
instantiated, so `Box[Int]` and `Box[String]` are distinct types  
with no implicit conversion between them. The `map` method on `Box`  
follows the functor pattern, applying a function to the contained  
value and wrapping the result in a new `Box`. Generics are the  
foundation of Scala's collections library, which provides `List[A]`,  
`Map[K, V]`, and `Option[A]` as parameterised types.  

## Recursive types

A recursive type is a type definition that refers to itself, enabling  
the modelling of hierarchical or self-similar data structures.  

```scala
sealed trait Tree[+A]
case class Leaf[A](value: A)         extends Tree[A]
case class Branch[A](left:  Tree[A],
                     right: Tree[A]) extends Tree[A]

def sum(tree: Tree[Int]): Int =
    tree match
        case Leaf(v)      => v
        case Branch(l, r) => sum(l) + sum(r)

def depth[A](tree: Tree[A]): Int =
    tree match
        case Leaf(_)      => 1
        case Branch(l, r) => 1 + (depth(l) max depth(r))

@main def main() =

    val t = Branch(Branch(Leaf(1), Leaf(2)), Leaf(3))
    println(sum(t))
    println(depth(t))

end main
```

A recursive type refers to itself in one or more of its variants,  
which is the natural way to model trees, linked lists, expressions,  
and other recursive structures. The `sealed` modifier ensures the  
compiler knows all variants and can enforce exhaustive pattern  
matches. The `+A` covariance annotation allows `Tree[Int]` to be  
used where `Tree[Any]` is expected, consistent with the substitution  
principle. Recursive functions over recursive types follow the  
structure of the data directly, with one case per variant.  

## Context-dependent types

Scala 3 uses `given` instances and `using` parameters to provide  
type class instances that influence how values are treated.  

```scala
trait Describable[A]:
    def describe(a: A): String

given Describable[Int] with
    def describe(n: Int) = s"integer $n"

given Describable[String] with
    def describe(s: String) = s"string '$s'"

def printDesc[A](value: A)(using d: Describable[A]): Unit =
    println(d.describe(value))

@main def main() =

    printDesc(42)
    printDesc("hello")

    given Describable[Boolean] with
        def describe(b: Boolean) = s"flag $b"

    printDesc(true)

end main
```

The `given` keyword introduces a contextual instance that the  
compiler locates automatically whenever a `using` parameter of the  
corresponding type is required. This is Scala 3's mechanism for  
type classes: the `Describable[A]` trait defines the capability,  
`given` instances provide implementations for specific types, and  
`printDesc` requires a `Describable` instance without the caller  
needing to pass it explicitly. Contextual abstraction decouples the  
definition of a capability from its usage site, enabling highly  
modular and composable code.  

## Scala types vs Java types

Scala's built-in types map directly to JVM types, with Scala  
providing a richer API and a unified type hierarchy on top.  

```scala
@main def main() =

    // Scala Int  <-> JVM int / java.lang.Integer
    val scalaInt: Int               = 42
    val javaInt:  java.lang.Integer = scalaInt   // autoboxing

    // Scala String is java.lang.String
    val s: String = "hello"
    println(s.getClass)

    // Scala Array[Int] maps to int[]
    val arr = Array(1, 2, 3)
    println(arr.getClass.getSimpleName)

    // Scala List is NOT java.util.List
    val scalaList = List(1, 2, 3)
    println(scalaList.getClass)

    println(scalaInt.getClass)
    println(javaInt.getClass)

end main
```

Scala value types such as `Int`, `Long`, and `Double` compile to JVM  
primitives when used in local variables, method parameters, and  
return values. They are boxed to `java.lang.Integer`, `java.lang.Long`,  
and `java.lang.Double` when stored in generic contexts. `String` in  
Scala is identical to `java.lang.String` and carries no wrapper.  
`Array[T]` maps to a JVM typed array. Scala's immutable `List`,  
`Map`, and `Set` are entirely different classes from the  
`java.util` counterparts and are not interchangeable without  
explicit conversion.  

## Converting Scala and Java types

The `scala.jdk.CollectionConverters` object provides bidirectional  
conversions between Scala and Java collection types.  

```scala
import scala.jdk.CollectionConverters.*

@main def main() =

    val scalaList = List(1, 2, 3)
    val javaList  = scalaList.asJava
    println(javaList.getClass)

    val backToScala = javaList.asScala.toList
    println(backToScala)

    val scalaMap = Map("a" -> 1, "b" -> 2)
    val javaMap  = scalaMap.asJava
    println(javaMap)

    val javaAl = new java.util.ArrayList[String]()
    javaAl.add("x")
    javaAl.add("y")
    val scalaBuffer = javaAl.asScala
    println(scalaBuffer.toList)

end main
```

`asJava` wraps a Scala collection in a thin adapter that implements  
the corresponding `java.util` interface without copying the data.  
`asScala` performs the reverse wrapping. Because both conversions  
are wrappers rather than copies, mutations through the Java view  
are visible through the Scala view for mutable collections. For  
immutable Scala collections, the Java view throws  
`UnsupportedOperationException` on mutation attempts. Import  
`scala.jdk.CollectionConverters.*` to bring all extension methods  
into scope.  

## Data types in pattern matching

Pattern matching inspects the shape and type of a value and  
extracts its components in a single, readable expression.  

```scala
@main def main() =

    def classify(x: Any): String =
        x match
            case 0               => "zero"
            case i: Int if i > 0 => s"positive int $i"
            case i: Int          => s"negative int $i"
            case s: String       => s"string '$s'"
            case (a, b)          => s"pair ($a, $b)"
            case xs: List[?]     => s"list of ${xs.size}"
            case None            => "no value"
            case Some(v)         => s"some $v"
            case _               => "unknown"

    println(classify(0))
    println(classify(5))
    println(classify(-3))
    println(classify("hi"))
    println(classify((1, 2)))
    println(classify(List(1, 2, 3)))
    println(classify(Some(42)))

end main
```

Pattern matching in Scala covers literal patterns, type tests,  
deconstruction of case classes and tuples, and guards. A guard is  
an additional `if` condition after the pattern that must be true  
for the arm to match. The compiler uses the `unapply` method of  
case classes and objects to extract their fields during matching,  
making custom extractors possible for any type. Matching on `Any`  
is the canonical runtime type dispatch mechanism in Scala and is  
preferable to chains of `isInstanceOf` checks.  

## Data types in domain modeling

Sealed traits and case classes combine to model rich domain entities  
with precise type-safety and compiler-enforced exhaustiveness.  

```scala
sealed trait Payment
case class CreditCard(number: String, expiry: String) extends Payment
case class BankTransfer(iban: String)                  extends Payment
case class Cash(amount: BigDecimal)                    extends Payment

def process(p: Payment): String =
    p match
        case CreditCard(num, exp) =>
            s"Card ending ${num.takeRight(4)}, exp $exp"
        case BankTransfer(iban)   => s"Transfer to $iban"
        case Cash(amount)         => s"Cash $amount"

@main def main() =

    val payments = List(
        CreditCard("4111111111111111", "12/26"),
        BankTransfer("DE89370400440532013000"),
        Cash(BigDecimal("50.00"))
    )

    payments.foreach(p => println(process(p)))

end main
```

Sealed hierarchies guarantee that all subtypes are known at compile  
time, enabling the compiler to check that a `match` expression  
handles every case. Adding a new `Payment` subclass outside the  
defining file produces a compile error, preventing the silent  
introduction of unhandled cases in production code. This pattern,  
known as an algebraic data type (ADT), is the standard Scala idiom  
for modelling discriminated unions and is far safer than the  
equivalent approach in dynamically typed languages.  

## Data types in APIs

Using precise types in API signatures communicates intent, prevents  
misuse, and removes the need for runtime validation.  

```scala
opaque type Port = Int
object Port:
    def apply(n: Int): Option[Port] =
        if n > 0 && n < 65536 then Some(n) else None
    def value(p: Port): Int = p

case class Config(host: String, port: Port)

def connect(cfg: Config): String =
    s"Connecting to ${cfg.host}:${Port.value(cfg.port)}"

@main def main() =

    Port(8080) match
        case Some(p) =>
            val cfg = Config("localhost", p)
            println(connect(cfg))
        case None =>
            println("invalid port")

    println(Port(99999))
    println(Port(443).map(Port.value))

end main
```

Modelling a `Port` as an opaque type rather than a plain `Int` makes  
it impossible to pass an arbitrary integer where a validated port is  
required. The smart constructor `Port.apply` returns `Option[Port]`  
to force callers to handle the invalid-input case at the call site  
rather than deep inside the business logic. This style eliminates  
primitive obsession and is idiomatic Scala 3. `case class Config`  
bundles related settings with structural equality and pattern  
matching for free.  

## Idiomatic data type usage

Idiomatic Scala 3 code combines immutable data, algebraic types,  
and extension methods to express domain logic clearly and safely.  

```scala
case class Student(name: String, grade: Double)

extension (s: Student)
    def isPassing: Boolean = s.grade >= 50.0
    def letterGrade: String = s.grade match
        case g if g >= 90 => "A"
        case g if g >= 75 => "B"
        case g if g >= 60 => "C"
        case g if g >= 50 => "D"
        case _            => "F"

@main def main() =

    val roster = List(
        Student("Alice", 92.0),
        Student("Bob",   45.0),
        Student("Carol", 78.5),
        Student("Dave",  55.0)
    )

    val passing = roster.filter(_.isPassing)
    println(s"Passing: ${passing.map(_.name)}")

    roster.foreach(s => println(s"${s.name}: ${s.letterGrade}"))

    val avg = roster.map(_.grade).sum / roster.size
    println(f"Class average: $avg%.1f")

end main
```

Extension methods in Scala 3 add operations to an existing type  
without modifying its source, keeping domain logic cohesive and  
avoiding utility-class proliferation. The `Student` case class is  
a pure data container with compiler-generated equality and  
`toString`. Business rules like `isPassing` and `letterGrade` live  
in the extension block, keeping them close to the type they  
describe. Combining `filter`, `map`, and `sum` on the immutable  
roster list expresses queries declaratively, without any mutable  
state or explicit loops.  
