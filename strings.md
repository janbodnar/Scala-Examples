# Scala 3 Strings  

A String in Scala 3 represents an immutable sequence of characters. It is the  
standard type for text such as names, messages, paths, and formatted output.  

Scala strings are backed directly by java.lang.String on the JVM. This means  
Scala code can call Java string methods naturally and can pass String values  
to Java libraries without conversion.  

Because strings are immutable, every apparent change creates a new value. This  
makes string code predictable, but repeated accumulation can allocate many  
intermediate objects, so StringBuilder is the better choice for heavy appends.  

Scala 3 also supports multiline strings with triple quotes. When indentation  
would otherwise become part of the result, stripMargin removes a chosen margin  
marker and keeps the source readable.  

String interpolation is one of Scala's most expressive features. The s  
interpolator inserts values, f adds printf-style formatting, raw keeps escape  
sequences literal, and custom interpolators can encode domain rules.  

Strings are appropriate whenever text itself is the data you need to model or  
display. When the content has stricter structure, such as dates, money, or  
identifiers, it is often better to wrap raw strings in richer types.  

---  

## Basic string literal  

A string literal is the simplest way to create text in Scala 3.  

```scala
@main def main() =
    val greeting = "Hello, Scala 3"
    println(greeting)
    println(greeting.length)
end main
```

Double quotes create a String value directly in source code. The example  
prints the text and then asks for its length, which is another String  
operation available immediately. This is the starting point for nearly every  
text-processing task in Scala.  

## Escape sequences  

Escape sequences let a single literal contain special characters.  

```scala
@main def main() =
    val message = "First line\nSecond line\tTabbed \"quote\""
    println(message)
end main
```

The \n sequence inserts a line break, while \t inserts a tab. The escaped  
quote keeps the surrounding literal valid without ending it too early. These  
small markers are useful when text must include formatting or characters with  
syntactic meaning.  

## Raw strings  

The raw interpolator keeps escape sequences as ordinary text.  

```scala
@main def main() =
    val path = raw"C:\projects\scala\notes.txt"
    val pattern = raw"\d+\s+items"
    println(path)
    println(pattern)
end main
```

A raw string does not treat backslash escapes like \n or \t specially. This  
makes it convenient for file paths, regular expressions, and other text where  
backslashes should remain visible. Interpolation still works, but the escape  
handling changes.  

## Multiline strings  

Triple-quoted literals can span several source lines without escapes.  

```scala
@main def main() =
    val poem = """Roses are red
Violets are blue
Scala 3 is modern
And indentation is too"""
    println(poem)
end main
```

Triple quotes preserve embedded newlines, so the string reads naturally in  
source code. They are helpful for sample text, templates, and small documents.  
This form keeps the content readable when many explicit \n sequences would be  
distracting.  

## stripMargin and custom margin characters  

stripMargin removes a chosen marker from each line of multiline text.  

```scala
@main def main() =
    val defaultMargin =
        """|Scala
           |keeps
           |text tidy""".stripMargin

    val customMargin =
        """#one
           #two
           #three""".stripMargin('#')

    println(defaultMargin)
    println(customMargin)
end main
```

The first string uses the default | marker, which is common in Scala examples.  
The second string chooses # instead, showing that the margin character is  
configurable. This method keeps code indentation pleasant without leaving  
unwanted spaces in the final value.  

## Concatenation with +  

The + operator joins string values from left to right.  

```scala
@main def main() =
    val first = "Scala"
    val second = "3"
    val label = first + " " + second + " strings"
    println(label)
end main
```

Concatenation is simple and direct for short combinations of text. Each +  
produces a new string, so the style is best for small results rather than long  
loops. For quick labels and messages, it remains perfectly idiomatic.  

## Concatenation with interpolation  

Interpolation often reads better than repeated + operators.  

```scala
@main def main() =
    val language = "Scala"
    val version = 3
    val text = s"$language $version strings"
    println(text)
end main
```

The s interpolator inserts values directly into surrounding text. This reduces  
visual noise and makes the final sentence easier to scan. When the result is  
mostly text with a few values inside it, interpolation is usually the clearest  
approach.  

## Repeating strings  

A string can be repeated when you need a repeated pattern or border.  

```scala
@main def main() =
    val border = "-".repeat(20)
    val chant = "ha".repeat(3)
    println(border)
    println(chant)
end main
```

The repeat method creates a new string by duplicating the receiver a chosen  
number of times. This is handy for separators, padding, and test data. It  
avoids a manual loop when the intention is simple repetition.  

## Converting to upper and lower case  

Case conversion changes the visual form of letters in a string.  

```scala
@main def main() =
    val text = "Scala 3"
    println(text.toUpperCase)
    println(text.toLowerCase)
end main
```

The original string remains unchanged because String is immutable. Each  
conversion returns a fresh value with the requested casing. These methods are  
common when preparing text for display or comparison.  

## Trimming whitespace  

Trimming removes unwanted whitespace at the edges of a string.  

```scala
@main def main() =
    val padded = "   hello Scala 3   "
    println(padded.trim)
    println(padded.stripLeading)
    println(padded.stripTrailing)
end main
```

The trim method removes leading and trailing whitespace together. The  
stripLeading and stripTrailing methods let you control each side separately.  
This is especially useful when processing user input or lines read from files.  

## Checking prefixes and suffixes  

Prefix and suffix checks answer common questions about text structure.  

```scala
@main def main() =
    val filename = "report.csv"
    println(filename.startsWith("rep"))
    println(filename.endsWith(".csv"))
end main
```

startsWith checks the beginning of the string, while endsWith checks the end.  
These methods are clearer than slicing text manually for such cases. They are  
often used with file names, paths, commands, and small text protocols.  

## Checking emptiness and blankness  

Scala strings can be empty or contain only whitespace characters.  

```scala
@main def main() =
    val empty = ""
    val blank = "   "
    println(empty.isEmpty)
    println(blank.isBlank)
end main
```

An empty string has length zero and contains no characters at all. A blank  
string may contain spaces or tabs, but no visible content. The distinction  
matters when validation should treat whitespace as missing input.  

## Comparing strings  

String comparison can test equality, case-insensitive equality, and ordering.  

```scala
@main def main() =
    val first = "Scala"
    val second = "scala"
    println(first == second)
    println(first.equalsIgnoreCase(second))
    println(first.compareTo(second))
end main
```

In Scala, == compares string contents rather than object identity. The  
equalsIgnoreCase method is useful when letter case should not matter.  
compareTo provides ordering information, which can help with sorting or  
implementing simple lexical rules.  

## Unicode basics  

Strings can hold Unicode text such as accented letters and symbols.  

```scala
@main def main() =
    val text = "café ☕"
    println(text)
    println(text.length)
    println(text.codePointCount(0, text.length))
end main
```

Scala strings handle Unicode because they are backed by Java strings. The  
length method counts UTF-16 code units, while codePointCount counts Unicode  
code points across the given range. For everyday text, both are often fine,  
but advanced Unicode work should know the difference.  

## Iterating over characters  

A for loop can visit each character in a string in order.  

```scala
@main def main() =
    val word = "Scala"
    for ch <- word do
        println(ch)
end main
```

Strings behave like sequences for many small traversal tasks. The loop  
receives one character at a time, which makes inspection simple and readable.  
This pattern is useful when validating, transforming, or counting individual  
characters.  

## Counting characters  

Character counts can measure total text or selected subsets of it.  

```scala
@main def main() =
    val text = "one two three"
    val total = text.length
    val letters = text.count(_.isLetter)
    println(total)
    println(letters)
end main
```

The length method counts every character position in the string. The count  
method is more selective because it uses a predicate to keep only matching  
characters. Together they show the difference between raw size and meaningful  
content.  

## Counting words  

Word counting often starts by splitting on runs of whitespace.  

```scala
@main def main() =
    val sentence = "Scala 3 makes text processing pleasant"
    val words = sentence.split("\\s+").count(_.nonEmpty)
    println(words)
end main
```

The split call breaks the sentence into pieces using a regular expression.  
Counting non-empty parts avoids accidental empty items when there are repeated  
spaces. This is a practical first step for simple text analysis.  

## Reversing strings  

The reverse method returns the characters in the opposite order.  

```scala
@main def main() =
    val word = "stressed"
    println(word.reverse)
end main
```

reverse creates a new string and leaves the original untouched. It is a  
compact example of immutable transformation. While often used for small  
puzzles, it is also handy for formatting tricks and quick inspections.  

## Filtering characters  

Filtering keeps only characters that satisfy a chosen rule.  

```scala
@main def main() =
    val rawInput = "phone: +421 903 111 222"
    val digits = rawInput.filter(_.isDigit)
    println(digits)
end main
```

The filter method walks through the string and keeps matching characters. Here  
it removes spaces, punctuation, and letters while preserving only digits. This  
style is concise for small sanitizing or extraction tasks.  

## Mapping characters  

map transforms each character and builds a new string from the results.  

```scala
@main def main() =
    val text = "scala"
    val loud = text.map(_.toUpper)
    println(loud)
end main
```

Each input character passes through the mapping function in sequence. The  
example converts every letter to upper case and collects the new characters  
into another string. map is useful whenever a transformation applies uniformly  
across the whole text.  

## Folding characters  

foldLeft accumulates a result while scanning characters from left to right.  

```scala
@main def main() =
    val text = "a1b2c3"
    val total =
        text.foldLeft(0): (sum, ch) =>
            if ch.isDigit then sum + ch.asDigit else sum
    println(total)
end main
```

foldLeft carries an accumulator through the string one character at a time.  
The example adds only digits and ignores letters, so the final result is six.  
This pattern is powerful because it can express many custom analyses without  
mutable state.  

## s-interpolator basics  

The s interpolator inserts values into a readable template.  

```scala
@main def main() =
    val name = "Scala"
    val version = 3
    println(s"Welcome to $name $version")
end main
```

Variable names can appear directly after the dollar sign when the expression  
is simple. The surrounding string remains easy to read as a sentence. This is  
one of the most common tools for building display text in Scala.  

## Embedding expressions in s-interpolator  

Braces let the s interpolator evaluate full Scala expressions.  

```scala
@main def main() =
    val items = List("pen", "notebook", "eraser")
    println(s"You have ${items.size} items: ${items.mkString(", ")}")
end main
```

The ${...} form is required when the inserted part is more than a name. Here  
it evaluates both a size lookup and a mkString call inline. The result stays  
compact without introducing temporary variables for every little fragment.  

## f-interpolator for numeric formatting  

The f interpolator combines interpolation with printf-style formats.  

```scala
@main def main() =
    val price = 12.3456
    val weight = 7.0
    println(f"Price: $$${price}%.2f, weight: ${weight}%.1f kg")
end main
```

The % directives control how numeric values appear in the final text. In this  
example the price keeps two decimal places and the weight keeps one. This is  
useful for reports, tables, and user-facing output where formatting should be  
consistent.  

## raw-interpolator for regex and escapes  

The raw interpolator is handy when text contains many backslashes.  

```scala
@main def main() =
    val regex = raw"""^\d{4}-\d{2}-\d{2}$"""
    val sample = raw"first\nsecond"
    println(regex)
    println(sample)
end main
```

The regular expression remains readable because the backslashes are not  
treated as escape markers by the interpolator. The second value shows that raw  
keeps the characters \ and n literally. This reduces visual noise in patterns,  
paths, and other escape-heavy text.  

## Custom interpolators  

Scala 3 lets you define domain-specific interpolators on StringContext.  

```scala
extension (sc: StringContext)
    def bracket(args: Any*): String =
        val rendered = sc.s(args*)
        s"[${rendered.toUpperCase}]"

@main def main() =
    val language = "scala"
    println(bracket"hello $language")
end main
```

A custom interpolator is just a method on StringContext with a special calling  
syntax. The example reuses the built-in s interpolator and then transforms the  
final text. This technique is useful when a project needs validation,  
escaping, or formatting tailored to one domain.  

## Interpolation with multiline strings  

Interpolation works naturally inside triple-quoted string templates.  

```scala
@main def main() =
    val title = "Scala 3"
    val banner =
        s"""|Welcome to $title
            |Indentation stays readable.
            |Interpolation still works.""".stripMargin
    println(banner)
end main
```

The s interpolator and stripMargin combine well for readable templates. Values  
can appear in the middle of larger blocks of text without extra concatenation.  
This is a pleasant way to build messages, emails, and small configuration  
snippets.  

## Interpolation with case classes  

Case classes expose fields that fit neatly into interpolated strings.  

```scala
case class User(name: String, age: Int)

@main def main() =
    val user = User("Nina", 29)
    println(s"${user.name} is ${user.age} years old")
end main
```

Case classes are common data carriers in Scala applications. Their fields can  
be interpolated directly to produce friendly output. This keeps simple  
rendering logic close to the data without much ceremony.  

## Interpolation with tuples  

Tuple elements can also be inserted into interpolated text.  

```scala
@main def main() =
    val point = (3, 4)
    println(s"Point(${point._1}, ${point._2})")
end main
```

Tuples are lightweight containers for grouped values. The example reads each  
component with _1 and _2 and places them into a label. This is a quick option  
when a full case class would be unnecessary.  

## Interpolation with collections  

Collection values often appear inside strings after a small conversion.  

```scala
@main def main() =
    val langs = List("Scala", "Java", "Kotlin")
    println(s"JVM languages: ${langs.mkString(", ")}")
end main
```

A collection rarely prints exactly as you want by default. mkString turns the  
elements into one readable fragment before interpolation. This pattern is  
common for logs, summaries, and user-facing reports.  

## Interpolation inside for-comprehensions  

Strings are often built while mapping over a collection of values.  

```scala
@main def main() =
    val names = List("Ada", "Grace", "Linus")
    val greetings =
        for name <- names yield s"Hello, $name"
    greetings.foreach(println)
end main
```

The for-comprehension transforms each name into a greeting. Because the output  
is another collection, the code stays declarative and avoids a mutable buffer.  
Interpolation fits well here because each element turns into a small piece of  
display text.  

## Interpolation with givens  

A given value can provide context for building interpolated text.  

```scala
case class Prefix(value: String)

given Prefix = Prefix("user")

def render(name: String)(using prefix: Prefix): String =
    s"${prefix.value}: $name"

@main def main() =
    println(render("Alice"))
end main
```

The render function receives its prefix from the surrounding context. This  
keeps call sites short while still allowing configuration to vary. Combining  
givens with interpolation works well when labels, locales, or formatting  
settings should be supplied implicitly.  

## Interpolation with inline methods  

Inline methods can return interpolated strings with minimal overhead.  

```scala
inline def tag(inline value: String): String =
    s"<<$value>>"

@main def main() =
    println(tag("scala"))
end main
```

The method body uses normal interpolation even though the method is marked  
inline. For small helper methods, this can preserve readability while letting  
the compiler inline the call site. The result is a tidy abstraction for  
recurring text patterns.  

## Interpolation for building SQL-like strings  

Interpolation can assemble readable query text for demos and logs.  

```scala
@main def main() =
    val table = "users"
    val id = 42
    val query = s"select * from $table where id = $id"
    println(query)
end main
```

The example shows how interpolation can express query structure clearly. This  
is fine for demonstration output or internal logging of values. In real  
database code, prefer parameterized queries rather than inserting raw values  
into SQL strings.  

## Interpolation for building JSON-like strings  

Interpolation can also build small JSON-shaped snippets of text.  

```scala
@main def main() =
    val name = "Scala"
    val stable = true
    val json = s"""{"name":"$name","stable":$stable}"""
    println(json)
end main
```

The resulting value looks like JSON, and the variable parts remain easy to  
spot. This can be convenient for samples, tests, or tiny fragments of text  
output. For real JSON processing, a dedicated data model or library is safer  
than manual string assembly.  

## Substring operations  

Substring methods extract selected ranges from a larger string.  

```scala
@main def main() =
    val text = "functional"
    println(text.substring(0, 4))
    println(text.takeRight(4))
    println(text.drop(4))
end main
```

substring uses explicit index positions, while takeRight and drop focus on  
counts from one side. Together they cover many common slicing needs. These  
operations are useful when text has a known layout or prefix that should be  
separated from the remainder.  

## Splitting strings  

split breaks a string into smaller parts using a separator or pattern.  

```scala
@main def main() =
    val csv = "red,green,blue"
    val parts = csv.split(",").toList
    println(parts)
end main
```

The result of split is an array, which can then be converted into other  
collection types. This example uses a comma separator to produce a list of  
color names. Splitting is one of the most common entry points for lightweight  
parsing.  

## Joining strings  

mkString joins many small pieces into one larger string.  

```scala
@main def main() =
    val words = List("Scala", "3", "rocks")
    val sentence = words.mkString(" ")
    println(sentence)
end main
```

mkString inserts the chosen separator between collection elements. It is  
easier to read than repeated concatenation when many items are involved. This  
method is especially useful when collections must become messages, labels, or  
command arguments.  

## Replacing substrings  

replace swaps one literal piece of text for another.  

```scala
@main def main() =
    val text = "Scala 2 and Scala 2 examples"
    val updated = text.replace("Scala 2", "Scala 3")
    println(updated)
end main
```

Literal replacement is straightforward when the target text is known exactly.  
The method returns a new string and does not mutate the old one. This is  
useful for normalization, migrations, and small cleanup steps in pipelines.  

## Replacing with regex  

Regular-expression replacement handles more flexible text patterns.  

```scala
@main def main() =
    val text = "Room 12, shelf 9, box 104"
    val masked = text.replaceAll("\\d+", "#")
    println(masked)
end main
```

The pattern \d+ matches one or more digits wherever they occur. Using a  
regular expression makes the replacement apply to many shapes of input at  
once. This is helpful when a simple literal match would be too narrow.  

## Searching for substrings  

Search methods answer whether and where a literal substring appears.  

```scala
@main def main() =
    val text = "Scala makes strings pleasant"
    println(text.contains("strings"))
    println(text.indexOf("makes"))
    println(text.lastIndexOf("s"))
end main
```

contains returns a Boolean, while indexOf and lastIndexOf return positions.  
This lets you choose between a simple presence test and more detailed location  
information. These methods are efficient tools for straightforward search  
tasks.  

## Searching with regex  

Regex searches can find text that matches a pattern rather than a literal.  

```scala
@main def main() =
    val ticket = raw"TKT-\d{4}".r
    val text = "Opened TKT-2024 and then TKT-2025"
    println(ticket.findFirstIn(text))
end main
```

The .r call turns the pattern text into a Regex value. findFirstIn then  
returns the first matching substring wrapped in an Option. This is a clean  
approach when the structure matters more than exact literal text.  

## Extracting groups from regex matches  

Capture groups pull out specific parts of a matched string.  

```scala
val setting = raw"(\w+)=(\d+)".r

@main def main() =
    val text = "timeout=30"
    val result =
        text match
            case setting(key, value) => s"$key -> $value"
            case _ => "no match"
    println(result)
end main
```

The regular expression defines two groups, one for the key and one for the  
number. Pattern matching then unpacks those groups directly into named  
variables. This is one of Scala's nicest combinations of regexes and  
expressive pattern matching.  

## Normalizing whitespace  

Whitespace normalization collapses inconsistent spacing into one style.  

```scala
@main def main() =
    val messy = "  Scala   3\n\tstrings   are   neat  "
    val clean = messy.trim.split("\\s+").mkString(" ")
    println(clean)
end main
```

The trim call removes extra edges first, and the split pattern treats all runs  
of whitespace as separators. mkString then rebuilds the text with single  
spaces only. This is a practical cleanup step before search, display, or  
comparison.  

## Padding left and right  

Padding adds fill characters to align string values to a fixed width.  

```scala
def leftPad(text: String, width: Int, ch: Char): String =
    text.reverse.padTo(width, ch).reverse

@main def main() =
    val id = leftPad("42", 5, '0')
    val label = "cat".padTo(6, '.')
    println(id)
    println(label)
end main
```

The leftPad helper reverses the string so padTo can add characters to what  
becomes the front after reversing back. padTo on the second line adds dots to  
the right side directly. Fixed-width text is common in reports, tables, and  
formatted identifiers.  

## Sanitizing input  

Filtering and trimming can remove unwanted characters from free text.  

```scala
@main def main() =
    val input = "  user<script>alert(1)</script>-42  "
    val safe =
        input.trim.filter: ch =>
            ch.isLetterOrDigit || ch.isWhitespace || ch == '-'
    println(safe)
end main
```

The code keeps only letters, digits, spaces, and hyphens from the input.  
Everything else is dropped, which produces a conservative cleaned value. Real  
sanitization rules depend on the context, but this example shows a simple and  
explicit character whitelist.  

## Parsing CSV-like strings  

Simple comma-separated text can be parsed with split and trimming.  

```scala
@main def main() =
    val row = "book, 19, in-stock"
    val columns = row.split(",").map(_.trim).toList
    val List(name, quantity, status) = columns
    println(name)
    println(quantity.toInt)
    println(status)
end main
```

This example deliberately handles a CSV-like format, not full CSV with quoting  
rules. After splitting, each field is trimmed and then unpacked with pattern  
matching into named values. It is a neat approach when the input format is  
small and tightly controlled.  

## Converting strings to numbers  

Numeric parsers turn textual digits into numeric types when the format fits.  

```scala
import scala.util.Try

@main def main() =
    val maybeInt = Try("42".toInt).toOption
    val maybeDouble = Try("3.14".toDouble).toOption
    println(maybeInt)
    println(maybeDouble)
end main
```

The toInt and toDouble methods parse strings according to the target type.  
Wrapping them with Try avoids throwing an exception on bad input and yields an  
Option instead. This is a safer pattern when text comes from users, files, or  
network boundaries.  

## Converting numbers to strings  

Numeric values can become strings for display or further formatting.  

```scala
@main def main() =
    val values = List(1, 2, 3)
    val text = values.map(_.toString).mkString(", ")
    println(text)
end main
```

Every numeric type provides a toString representation. The example maps each  
number into text and then joins the results with commas. This is a common step  
when turning computed values into logs, labels, or reports.  

## Converting to Boolean  

Small parsing functions can map several textual forms onto Boolean values.  

```scala
def toBoolean(text: String): Option[Boolean] =
    text.trim.toLowerCase match
        case "true" | "yes" | "1" => Some(true)
        case "false" | "no" | "0" => Some(false)
        case _ => None

@main def main() =
    println(toBoolean(" yes "))
    println(toBoolean("no"))
    println(toBoolean("maybe"))
end main
```

The helper normalizes the input before matching on a small set of known forms.  
Returning an Option makes failure explicit rather than guessing. This is a  
clear and idiomatic technique for controlled text-to-value conversions.  

## Converting to Char arrays  

A string can be expanded into a mutable array of characters when needed.  

```scala
@main def main() =
    val chars = "Scala".toCharArray
    chars.foreach(println)
    println(chars.mkString("-"))
end main
```

toCharArray produces an Array[Char], which is useful when an API expects array  
input or mutable character storage. The example prints each character  
separately and then joins the array back into a string. This shows both  
directions of the conversion.  

## Using StringBuilder  

StringBuilder is better than repeated concatenation in accumulation loops.  

```scala
import scala.collection.mutable.StringBuilder

@main def main() =
    val words = List("Scala", "3", "strings")
    val builder = StringBuilder()
    for word <- words do
        builder.append(word).append(" ")
    println(builder.result().trim)
end main
```

Each append updates the builder in place instead of creating a brand new  
String value. That reduces unnecessary allocations when many small parts must  
be combined. Once the content is complete, result returns the final immutable  
string.  

## Java interop  

Scala strings work directly with Java APIs because they are Java strings.  

```scala
@main def main() =
    val text: java.lang.String = "Scala"
    val joined = java.lang.String.join(" / ", text, "Java")
    println(joined.toLowerCase)
    println(text.substring(1))
end main
```

The explicit java.lang.String type annotation shows the underlying type  
without any conversion step. Java static methods and instance methods are both  
available immediately. This close interop is one reason Scala fits well in  
existing JVM ecosystems.  

## Encoding and decoding with UTF-8  

Text can be converted to bytes and back with an explicit character set.  

```scala
import java.nio.charset.StandardCharsets

@main def main() =
    val text = "café"
    val bytes = text.getBytes(StandardCharsets.UTF_8)
    val decoded = new String(bytes, StandardCharsets.UTF_8)
    println(bytes.mkString("[", ", ", "]"))
    println(decoded)
end main
```

Using UTF-8 explicitly makes the encoding choice clear and portable. The byte  
array can be written to files, sockets, or other binary APIs, and the new  
String call reconstructs the text again. Being explicit about encodings  
prevents subtle bugs across machines and environments.  

## Converting strings to collections  

Strings can become standard collections for richer transformations.  

```scala
@main def main() =
    val word = "scala"
    val letters = word.toList
    val unique = word.toSet
    println(letters)
    println(unique)
end main
```

Converting to a collection opens the door to collection-focused methods and  
algorithms. A List preserves order and duplicates, while a Set keeps unique  
elements only. This is often the cleanest route for non-trivial character  
analysis.  

## Pattern matching on strings  

Pattern matching expresses string-driven branching in a direct style.  

```scala
@main def main() =
    val command = "start"
    val response =
        command match
            case "start" => "service started"
            case "stop" => "service stopped"
            case other if other.startsWith("help") => "show help"
            case _ => "unknown command"
    println(response)
end main
```

The match expression lists expected command shapes clearly and locally. Guard  
clauses can add extra logic without abandoning pattern matching. This style is  
readable when a string value selects one among several behaviors.  

## Opaque types wrapping strings  

Opaque types let you give raw strings stronger meaning at compile time.  

```scala
object UserId:
    opaque type UserId = String

    def apply(value: String): UserId =
        value.trim.toUpperCase

    extension (id: UserId)
        def value: String = id

import UserId.*

@main def main() =
    val userId: UserId = UserId(" ab-42 ")
    println(userId.value)
end main
```

Outside the companion object, UserId is not treated as an ordinary String even  
though the runtime representation is still a string. This adds type safety  
without additional allocation. Opaque types are an idiomatic Scala 3 tool for  
modeling identifiers and validated text.  

## Extension methods for strings  

Extension methods add project-specific string operations in a natural style.  

```scala
extension (text: String)
    def initials: String =
        text
            .split("\\s+")
            .filter(_.nonEmpty)
            .map(_.head.toUpper)
            .mkString

@main def main() =
    val name = "Ada Lovelace"
    println(name.initials)
end main
```

The new method reads like it belongs to String, but it is defined separately  
and keeps your domain logic modular. The example extracts one upper-case  
initial from each word. Extension methods are a good way to make string-heavy  
code more expressive without inheritance.  

## Using strings in pipelines  

The pipe method can express step-by-step text transformation clearly.  

```scala
import scala.util.chaining.*

@main def main() =
    val result =
        "  scala 3 pipelines  "
            .pipe(_.trim)
            .pipe(_.split("\\s+").toList)
            .pipe(_.map(_.capitalize))
            .pipe(_.mkString(" -> "))
    println(result)
end main
```

Each pipe call forwards the previous value into the next transformation. This  
makes the flow easy to read from top to bottom. Pipelines are nice when  
several small string operations should remain linear and explicit.  

## Idiomatic string usage in Scala 3  

Idiomatic code prefers clear transformations and strong types around text.  

```scala
case class Task(name: String, done: Boolean)

@main def main() =
    val tasks = List(
        Task("Write", true),
        Task("Review", false),
        Task("Ship", false)
    )

    val report =
        tasks
            .map: task =>
                val status = if task.done then "done" else "pending"
                s"${task.name}: $status"
            .mkString("Tasks(", ", ", ")")

    println(report)
end main
```

The example keeps business data in a case class rather than as raw text alone,  
and it delays string rendering until the final reporting step. The  
transformation uses val, map, and interpolation instead of mutable state. That  
combination is a good summary of idiomatic Scala 3 string handling in everyday  
programs.  
