# Scala Conditionals

This document demonstrates conditional expressions and pattern matching in
Scala 3, using fully runnable script-style examples.

## Basic if statement

This example shows a simple `if` statement that executes code based on a
condition.

```scala
@main def main() =

    val temperature = 25

    if temperature > 20 then
        println("It's a warm day!")
end main
```

In Scala 3 the `if` keyword is followed by the condition and the `then`
keyword, with no parentheses required around the condition. The indented
body runs only when the condition evaluates to `true`. This is the most
fundamental way to make a program react to runtime values.

## If-else branching

This example demonstrates an `if-else` statement to handle two possible
outcomes.

```scala
@main def main() =

    val age = 16

    if age >= 18 then
        println("You are an adult")
    else
        println("You are a minor")
end main
```

When the condition `age >= 18` is `true`, the first branch executes; otherwise
the `else` branch runs. Exactly one of the two branches is always executed,
providing a clear binary choice. The `else` keyword introduces the alternative
branch without any additional keyword.

## Multiple conditions with else if

This example shows how to chain multiple conditions using `else if`.

```scala
@main def main() =

    val score = 85

    if score >= 90 then
        println("Grade: A")
    else if score >= 80 then
        println("Grade: B")
    else if score >= 70 then
        println("Grade: C")
    else
        println("Grade: D or F")
end main
```

Conditions are evaluated from top to bottom and the first branch whose
condition is `true` executes. Subsequent branches are skipped entirely. This
structure is ideal for logic with multiple distinct cases, such as assigning
letter grades from a numeric score.

## If expression

This example shows that `if` in Scala is an expression that produces a value,
not just a statement.

```scala
@main def main() =

    val number = 7
    val result = if number % 2 == 0 then "even" else "odd"

    println(s"$number is $result")
end main
```

Because `if` is an expression, it can appear on the right-hand side of a
`val` definition and its value can be stored directly. Both branches must
yield the same type, or Scala infers a common supertype. This eliminates the
need for a separate ternary operator that exists in Java and many other
languages.

## Multi-line if expression

This example demonstrates a multi-line `if` expression that returns a value
depending on several conditions.

```scala
import scala.util.Random

@main def main() =

    val r = Random.between(-10, 10)

    val message = if r > 0 then
        s"positive value ($r)"
    else if r == 0 then
        "zero value"
    else
        s"negative value ($r)"

    println(message)
end main
```

Multi-line `if` expressions use indentation for the body of each branch.
`Random.between(-10, 10)` produces a random integer in the half-open range
`[-10, 10)`. The resulting string is assigned to `message` and printed.
All three branches produce a `String`, so the inferred type of `message`
is `String`.

## Boolean logical operators

This example demonstrates combining conditions with the logical AND (`&&`)
operator.

```scala
@main def main() =

    val age = 25
    val hasLicense = true

    if age >= 18 && hasLicense then
        println("You can drive")
    else
        println("You cannot drive")
end main
```

The `&&` operator requires both operands to be `true` for the whole
expression to be `true`. If the left operand is `false` the right operand
is not evaluated at all (short-circuit evaluation). This pattern is useful
whenever multiple criteria must be satisfied simultaneously.

## Logical OR operator

This example shows how to use the logical OR (`||`) operator to check for
multiple possibilities.

```scala
@main def main() =

    val day = "Saturday"

    if day == "Saturday" || day == "Sunday" then
        println("It's the weekend!")
    else
        println("It's a weekday")
end main
```

The `||` operator returns `true` when at least one of its operands is `true`.
If the left operand is `true`, the right operand is skipped. In Scala, string
equality is tested with `==`, which delegates to `equals` under the hood, so
no `.equals` method call is needed as in Java.

## Logical NOT operator

This example shows the logical NOT (`!`) operator to invert a boolean
expression.

```scala
@main def main() =

    val isRaining = false

    if !isRaining then
        println("You don't need an umbrella")
    else
        println("Take an umbrella")
end main
```

The `!` operator flips `true` to `false` and vice versa. It is more readable
than writing `isRaining == false` because it directly expresses the intent of
the condition. Prefer `!flag` over comparing to `false` explicitly.

## Short-circuit evaluation

This example shows how short-circuiting avoids a potential null-pointer error.

```scala
@main def main() =

    val text: String | Null = null

    if text != null && text.length > 0 then
        println(s"Text: $text")
    else
        println("Text is null or empty")
end main
```

In a `&&` chain, if the left operand evaluates to `false` the right operand
is never evaluated. Here, `text != null` guards the `text.length` call: if
`text` is `null` the second condition is skipped, preventing a
`NullPointerException`. The union type `String | Null` makes nullability
explicit in Scala 3.

## Complex boolean expressions

This example demonstrates combining multiple conditions with grouping.

```scala
@main def main() =

    val temperature = 25
    val humidity    = 60
    val isSunny     = true

    if (temperature > 20 && temperature < 30) &&
       (humidity < 70) && isSunny then
        println("Perfect weather for outdoor activities!")
    else
        println("Maybe stay indoors")
end main
```

Parentheses group sub-expressions and make evaluation order explicit. Without
parentheses Scala follows standard operator precedence, but grouping makes the
logic far easier to read and reason about. Each condition can be evaluated
independently, and `&&` short-circuits as soon as any operand is `false`.

## Match expression

This example demonstrates a `match` expression to branch on discrete values,
the Scala equivalent of a `switch` statement.

```scala
@main def main() =

    val dayNumber = 3

    dayNumber match
        case 1 => println("Monday")
        case 2 => println("Tuesday")
        case 3 => println("Wednesday")
        case 4 => println("Thursday")
        case 5 => println("Friday")
        case 6 | 7 => println("Weekend")
        case _ => println("Invalid day")
end main
```

A `match` expression compares the scrutinee (`dayNumber`) against each `case`
pattern in order. The first matching case executes and no fall-through occurs.
The `|` operator inside a single case matches either value. The wildcard
pattern `_` acts as a catch-all, similar to `default` in Java's `switch`.

## Match expression returning a value

This example shows how a `match` expression returns a value that can be
assigned to a variable.

```scala
@main def main() =

    val month = 4

    val season = month match
        case 12 | 1 | 2 => "Winter"
        case 3 | 4 | 5  => "Spring"
        case 6 | 7 | 8  => "Summer"
        case 9 | 10 | 11 => "Fall"
        case _ => "Unknown"

    println(s"Season: $season")
end main
```

Like `if`, `match` in Scala is an expression that produces a value. The value
of the matched branch is returned and assigned to `season`. When all branches
yield the same type the compiler infers that type automatically. Using `match`
as an expression makes conditional assignments concise and readable.

## Match on strings

This example shows how a `match` expression works with `String` values.

```scala
@main def main() =

    val fruit = "apple"

    val color = fruit match
        case "apple"  => "red"
        case "banana" => "yellow"
        case "orange" => "orange"
        case "grape"  => "purple"
        case _        => "unknown"

    println(s"Color: $color")
end main
```

`match` can compare any value that supports structural equality, including
strings. Scala string equality via `==` is safe and does not require calling
`.equals` as in Java. Each `case` literal is compared with the scrutinee in
order from top to bottom.

## Guard clauses in match

This example demonstrates adding a condition (`if` guard) to a `case` clause.

```scala
@main def main() =

    val score = 85

    val grade = score match
        case s if s >= 90 => "A"
        case s if s >= 80 => "B"
        case s if s >= 70 => "C"
        case s if s >= 60 => "D"
        case _            => "F"

    println(s"Grade: $grade")
end main
```

A guard is an `if` expression appended to a `case` pattern. The case only
matches when both the pattern and the guard condition are satisfied. The
variable `s` in each case binds to the scrutinee value, making it available
inside the guard and the case body. Guards allow fine-grained control beyond
simple value matching.

## Type pattern matching

This example demonstrates matching on the runtime type of a value.

```scala
@main def main() =

    val value: Any = 42

    value match
        case i: Int    => println(s"Integer: ${i * 2}")
        case s: String => println(s"String: ${s.toUpperCase}")
        case d: Double => println(s"Double: ${d * 2.0}")
        case _         => println("Unknown type")
end main
```

A type pattern `x: T` checks whether the scrutinee is an instance of `T` and,
if so, binds it to the variable `x` with type `T`. This replaces the
`instanceof` check and explicit cast that Java requires. The compiler knows the
exact type inside each branch, so type-specific operations like `.toUpperCase`
are available without casting.

## Case class pattern matching

This example shows how to destructure a case class using pattern matching.

```scala
case class Point(x: Int, y: Int)

@main def main() =

    val obj: Any = Point(5, 10)

    obj match
        case Point(x, y) =>
            println(s"Point: x=$x, y=$y")
            println(s"Sum: ${x + y}")
        case _ =>
            println("Not a Point")
end main
```

Pattern matching on a case class extracts the constructor arguments into named
variables in a single step, combining the type check and the field extraction.
This is similar to Java record patterns but is a first-class feature of the
Scala type system. The `case class` provides an auto-generated `unapply` method
that the compiler uses when matching.

## Tuple pattern matching

This example demonstrates matching and destructuring a tuple.

```scala
@main def main() =

    val coordinate = (3, -7)

    val description = coordinate match
        case (0, 0)          => "origin"
        case (x, 0)          => s"on x-axis at $x"
        case (0, y)          => s"on y-axis at $y"
        case (x, y) if x > 0 && y > 0 => s"first quadrant ($x, $y)"
        case (x, y)          => s"other quadrant ($x, $y)"

    println(description)
end main
```

Tuple patterns match on the structure of a tuple and bind its components to
variables at the same time. Specific values like `0` can be used as sub-patterns
to test for constant positions within the tuple. Guards can further narrow the
match, as shown for the first quadrant test.

## Wildcard and variable patterns

This example illustrates the difference between wildcard (`_`) and variable
patterns.

```scala
@main def main() =

    val values: List[Any] = List(1, "hello", 3.14, true)

    for v <- values do
        v match
            case _: Boolean => println("boolean — ignored")
            case n: Int     => println(s"int: $n")
            case s: String  => println(s"string: $s")
            case other      => println(s"other: $other")
end main
```

The wildcard `_` matches anything but does not bind a variable, useful when
the value inside a branch is not needed. A named variable like `other` also
matches anything but binds the value so it can be referenced in the branch
body. Using `_` communicates that the value is intentionally unused.

## Nested patterns

This example shows patterns nested inside other patterns for deep
destructuring.

```scala
case class Address(city: String, country: String)
case class Person(name: String, address: Address)

@main def main() =

    val person: Any = Person("Alice", Address("Prague", "CZ"))

    person match
        case Person(name, Address(city, "CZ")) =>
            println(s"$name lives in $city, Czech Republic")
        case Person(name, Address(city, country)) =>
            println(s"$name lives in $city, $country")
        case _ =>
            println("Unknown")
end main
```

Patterns can be nested arbitrarily deep. The inner pattern
`Address(city, "CZ")` matches only when the country field equals `"CZ"` while
simultaneously binding the city to a variable. Nested patterns provide a
compact and expressive way to inspect hierarchical data structures.

## Enum pattern matching

This example demonstrates pattern matching on a Scala 3 enum.

```scala
enum Direction:
    case North, South, East, West

@main def main() =

    val dir = Direction.North

    val message = dir match
        case Direction.North => "Heading north"
        case Direction.South => "Heading south"
        case Direction.East  => "Heading east"
        case Direction.West  => "Heading west"

    println(message)
end main
```

Scala 3 enums define a sealed set of named values. Matching on an enum is
exhaustive: the compiler warns if any case is missing. This makes enum-based
`match` expressions safer than matching on arbitrary integers. No `default`
or wildcard case is needed when all enum values are covered.

## Range checking with conditionals

This example shows how to check whether a value falls within a specific range.

```scala
@main def main() =

    val value = 50

    if value >= 0 && value <= 25 then
        println("Low range")
    else if value > 25 && value <= 75 then
        println("Medium range")
    else if value > 75 && value <= 100 then
        println("High range")
    else
        println("Out of range")
end main
```

Combining comparison operators with `&&` is the standard way to test whether
a number lies between two bounds. Each branch is mutually exclusive because
the upper bound of one range is the lower bound of the next. This pattern is
common in data validation and tiered classification.

## Guard clause pattern

This example illustrates the guard clause pattern for early exit and cleaner
code.

```scala
@main def main() =

    def checkAge(age: Int) =

        if age < 0 then
            println("Invalid age")
            return

        if age < 18 then
            println("Minor")
            return

        println("Adult")

    checkAge(-5)
    checkAge(14)
    checkAge(30)
end main
```

Guard clauses place precondition checks at the top of a function and exit
early when those conditions are not met. The main logic is left at the bottom
with minimal nesting. This improves readability compared to deeply nested
`if-else` chains and makes the happy path immediately obvious.

## Nested conditionals

This example demonstrates nested `if` statements for multi-level checks.

```scala
@main def main() =

    val hasAccount  = true
    val isVerified  = true
    val balance     = 100

    if hasAccount then
        if isVerified then
            if balance > 0 then
                println("You can make a purchase")
            else
                println("Insufficient balance")
        else
            println("Account not verified")
    else
        println("No account found")
end main
```

Nesting conditionals enables complex, hierarchical decision logic but can
quickly make code difficult to follow. Deep nesting is often a sign that the
logic should be refactored using guard clauses, helper functions, or a `match`
expression to make the control flow explicit.

## Conditional with collections

This example demonstrates conditional logic based on collection properties.

```scala
@main def main() =

    val numbers = List(1, 2, 3, 4, 5)

    if numbers.isEmpty then
        println("List is empty")
    else if numbers.size == 1 then
        println(s"List has one element: ${numbers.head}")
    else
        println(s"List has ${numbers.size} elements")
end main
```

You can branch on properties of collections such as `isEmpty` or `size`.
These boolean methods make the intent clear at a glance. Checking `isEmpty`
before calling `head` ensures no exception is thrown on an empty list, making
the conditional both correct and safe.

## Conditional in collections

This example shows using conditional logic inside collection operations.

```scala
@main def main() =

    val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    val evens = numbers.filter(n => n % 2 == 0)
    val odds  = numbers.filter(n => n % 2 != 0)

    println(s"Even numbers: $evens")
    println(s"Odd numbers: $odds")
end main
```

The `filter` method takes a predicate — a function from element to `Boolean` —
and returns a new list containing only the elements for which the predicate
holds. This is declarative conditional processing: instead of writing an
explicit loop with an `if` inside, you express *which* elements to keep. The
original list is not modified.

## Match with collection patterns

This example demonstrates matching against list structure using built-in
list patterns.

```scala
@main def main() =

    def describe(lst: List[Int]): String =
        lst match
            case Nil          => "empty list"
            case x :: Nil     => s"single element: $x"
            case x :: y :: Nil => s"two elements: $x and $y"
            case head :: tail  => s"starts with $head, ${tail.size} more"

    println(describe(List()))
    println(describe(List(42)))
    println(describe(List(1, 2)))
    println(describe(List(1, 2, 3, 4)))
end main
```

The `::` constructor pattern deconstructs a `List` into its `head` (first
element) and `tail` (remaining list). `Nil` matches an empty list. These
patterns can be nested to inspect deeper structure. Matching on list shape is
a common technique in functional Scala for writing recursive algorithms.

## Combining type and value patterns

This example showcases a `match` expression that handles multiple types and
value-level conditions together.

```scala
@main def main() =

    val values: List[Any] = List(42, "hello", -3, "", 3.14, true, null)

    for v <- values do
        val description = v match
            case null            => "null value"
            case i: Int if i < 0 => s"negative integer: $i"
            case i: Int if i == 0 => "zero"
            case i: Int          => s"positive integer: $i"
            case s: String if s.isEmpty => "empty string"
            case s: String       => s"string: $s"
            case d: Double       => s"double: $d"
            case b: Boolean      => s"boolean: $b"
            case _               => "unknown"

        println(description)
end main
```

This advanced example combines null checking, type patterns, and guards in a
single `match`. Cases are tried top to bottom, so more specific conditions
like `i < 0` are placed before the general `Int` case. This ordering is
essential: if the general `case i: Int` appeared first it would swallow all
integers, making the guarded cases unreachable.
