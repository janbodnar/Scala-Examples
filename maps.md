# Scala 3 Maps

A `Map` in Scala 3 is an immutable, ordered collection of key-value pairs  
where every key is unique. The default `Map` factory in `scala.collection.  
immutable` returns a `HashMap`-backed structure for large maps and a  
compact, array-backed structure for maps with four entries or fewer.  
Because the default map is immutable, every operation that appears to  
modify it actually produces a new map, leaving the original unchanged.  

Maps differ from other Scala collections in a fundamental way. A `List`  
is an ordered sequence of values indexed by position; a `Set` is an  
unordered collection of unique values; an `Array` is a fixed-length,  
mutable indexed sequence. A `Map` adds a second dimension: each element  
carries both a *key* and a *value*, and values are retrieved by key  
rather than by numeric position.  

Immutability is the default in Scala, and `Map` is no exception. The  
immutable `Map` is thread-safe and fits naturally into functional code  
because operations compose without side effects. When mutation is  
unavoidable, `scala.collection.mutable.Map` provides in-place update  
with `+=`, `-=`, and direct assignment, at the cost of losing the  
safety guarantees of pure functions.  

Performance characteristics of the default immutable `HashMap` are:  
lookup, insertion, and deletion all run in effectively O(1) amortised  
time. Converting to a sorted representation via `toSeq.sortBy` costs  
O(n log n). The mutable `HashMap` shares the same asymptotic bounds  
while offering better constant factors when bulk mutation is required.  

In Scala's functional programming model, `Map` is a `Functor` and an  
`Iterable` of `(K, V)` pairs. This means you can use `map`, `flatMap`,  
`filter`, `fold`, and `for` comprehensions directly on a map, treating  
it as a sequence of tuples. The functional API keeps code declarative  
and easy to reason about, which is why maps appear throughout idiomatic  
Scala programs for configuration, caching, grouping, and aggregation.  

## Creating a map

The `Map` factory creates an immutable map from a list of key-value pairs.  

```scala
@main def run() =
  val cts = Map(
    "sk" -> "Slovakia",
    "ru" -> "Russia",
    "de" -> "Germany",
    "no" -> "Norway"
  )
  println(cts)
  println(cts.size)
```

The `->` operator is syntactic sugar for constructing a `Tuple2`; writing  
`"sk" -> "Slovakia"` is identical to `("sk", "Slovakia")`. The `Map`  
factory accepts any number of such pairs and builds an efficient  
immutable structure. Calling `println` on a map prints all pairs in  
an implementation-defined order.  

## Accessing values

Direct key lookup and the safe `get` method retrieve values from a map.  

```scala
@main def run() =
  val cts = Map(
    "sk" -> "Slovakia",
    "ru" -> "Russia",
    "de" -> "Germany"
  )

  println(cts("sk"))
  println(cts.get("sk"))
  println(cts.get("xx"))
```

The `apply` method `cts("sk")` retrieves the value directly and throws  
a `NoSuchElementException` when the key is absent. The `get` method is  
safer: it wraps the result in `Option`, returning `Some(value)` for a  
present key and `None` for a missing one. Preferring `get` over direct  
lookup avoids runtime exceptions in code that cannot guarantee key  
presence.  

## Adding entries

Because the default map is immutable, adding an entry returns a new map.  

```scala
@main def run() =
  val cts = Map("sk" -> "Slovakia", "de" -> "Germany")
  val cts2 = cts + ("hu" -> "Hungary")
  val cts3 = cts2 + ("pl" -> "Poland") + ("cz" -> "Czech Republic")

  println(cts)
  println(cts2)
  println(cts3)
```

The `+` operator accepts a `(K, V)` tuple and returns a new `Map`  
containing all existing pairs plus the new one. Chaining several `+`  
calls on a single expression is idiomatic when inserting a small number  
of known keys. The original `cts` is never modified; each variable  
holds an independent, immutable snapshot.  

## Updating an entry

Updating a key in an immutable map produces a new map with the  
replacement value.  

```scala
@main def run() =
  val scores = Map("Alice" -> 80, "Bob" -> 75, "Carol" -> 90)
  val updated = scores.updated("Bob", 85)

  println(scores("Bob"))
  println(updated("Bob"))
```

`updated` returns a copy of the map where the given key is bound to the  
new value. If the key did not exist it is inserted; if it already existed  
the old binding is replaced. The immutable semantics are preserved: both  
`scores` and `updated` coexist as separate values, which makes reasoning  
about state straightforward.  

## Removing an entry

The `-` operator removes a key from a map and returns a new map.  

```scala
@main def run() =
  val cts = Map(
    "sk" -> "Slovakia",
    "ru" -> "Russia",
    "de" -> "Germany",
    "no" -> "Norway"
  )

  val fewer = cts - "ru"
  val evenFewer = cts -- List("ru", "no")

  println(fewer)
  println(evenFewer)
```

Single-key removal uses the `-` operator, while `--` accepts any  
`IterableOnce[K]` and removes all supplied keys at once. Both return  
a fresh immutable map. If a supplied key is not present the operation  
silently succeeds, leaving the map unchanged for that key.  

## Iterating over pairs

A `for` expression destructures each entry into its key and value.  

```scala
@main def run() =
  val cts = Map(
    "sk" -> "Slovakia",
    "ru" -> "Russia",
    "de" -> "Germany",
    "no" -> "Norway"
  )

  for (k, v) <- cts do
    println(s"$k -> $v")
```

Destructuring `(k, v)` in the `for` pattern eliminates the need to  
call `._1` and `._2` manually. The iteration order of an immutable  
`HashMap` is not guaranteed to match insertion order; use a  
`ListMap` or sort the entries if a deterministic order is required.  

## Iterating over keys and values

The `keys` and `values` views let you iterate over each dimension  
independently.  

```scala
@main def run() =
  val cts = Map(
    "sk" -> "Slovakia",
    "ru" -> "Russia",
    "de" -> "Germany"
  )

  println("Keys:")
  cts.keys.foreach(println)

  println("Values:")
  cts.values.foreach(println)
```

`keys` returns an `Iterable[K]` and `values` returns an `Iterable[V]`.  
Both are lazy views over the underlying map, so no new collection is  
allocated until you begin consuming elements. Using `foreach` is  
idiomatic when the goal is side-effecting iteration without building  
a new collection.  

## Checking membership

`contains` and `isDefinedAt` test whether a key exists in the map.  

```scala
@main def run() =
  val cts = Map("sk" -> "Slovakia", "de" -> "Germany")

  println(cts.contains("sk"))
  println(cts.contains("xx"))
  println(cts.isDefinedAt("de"))
  println(cts.isEmpty)
  println(cts.nonEmpty)
```

`contains` is the idiomatic membership test; `isDefinedAt` is its  
alias inherited from the `PartialFunction` supertype. Both return a  
`Boolean` in O(1) time. The `isEmpty` and `nonEmpty` helpers are  
useful guards before iterating or accessing entries.  

## Using getOrElse

`getOrElse` returns a fallback value when the key is absent.  

```scala
@main def run() =
  val capitals = Map(
    "France"  -> "Paris",
    "Germany" -> "Berlin",
    "Japan"   -> "Tokyo"
  )

  val c1 = capitals.getOrElse("France", "unknown")
  val c2 = capitals.getOrElse("Brazil", "unknown")

  println(c1)
  println(c2)
```

`getOrElse` is a concise alternative to pattern-matching on `get`.  
The second argument is a by-name parameter, so it is only evaluated  
when the key is missing; expensive default computations are not wasted  
on hits. This makes it safe to use expensive expressions as defaults  
without any performance penalty on the happy path.  

## Default values with withDefaultValue

`withDefaultValue` attaches a constant fallback to a map.  

```scala
@main def run() =
  val scores = Map("Alice" -> 10, "Bob" -> 7).withDefaultValue(0)

  println(scores("Alice"))
  println(scores("Carol"))
  println(scores("Dave"))
```

`withDefaultValue` wraps the original map in a thin proxy that returns  
the given constant for any missing key instead of throwing. The  
wrapper satisfies the full `Map` interface, so it can be passed  
anywhere a `Map[K, V]` is expected. Note that `get` still returns  
`None` for absent keys; only direct `apply` uses the default.  

## Using Option with maps

Combining `get` with `Option` combinators chains safe lookups.  

```scala
@main def run() =
  val env = Map(
    "HOST" -> "localhost",
    "PORT" -> "8080"
  )

  val host = env.get("HOST").getOrElse("127.0.0.1")
  val port = env.get("PORT").map(_.toInt).getOrElse(80)
  val dbUrl = env.get("DB_URL").map(u => s"jdbc:$u")

  println(host)
  println(port)
  println(dbUrl)
```

`get` returns an `Option[V]`, which unlocks the full `Option` API:  
`map`, `flatMap`, `filter`, `getOrElse`, and `fold`. Chaining `map`  
on the result transforms the value only when it exists, keeping the  
code linear and exception-free. `dbUrl` is a `None` because `DB_URL`  
is absent, and no `NullPointerException` is possible.  

## Pattern matching on map entries

Pattern matching destructures tuples when iterating with `collect`.  

```scala
@main def run() =
  val inventory = Map(
    "apple"  -> 50,
    "banana" -> 0,
    "cherry" -> 120,
    "date"   -> 0
  )

  val inStock = inventory.collect:
    case (item, qty) if qty > 0 => s"$item: $qty"

  inStock.foreach(println)
```

`collect` applies a partial function to each entry and keeps only  
the results where the pattern matches. The guard `if qty > 0` filters  
out zero-stock items at the same time as the transformation, combining  
`filter` and `map` into one pass. The result is a new collection  
containing only the formatted strings for items that are available.  

## Merging maps

The `++` operator merges two maps, with the right-hand side winning  
on duplicate keys.  

```scala
@main def run() =
  val defaults = Map("timeout" -> 30, "retries" -> 3, "debug" -> 0)
  val overrides = Map("timeout" -> 60, "debug" -> 1)

  val config = defaults ++ overrides
  println(config)
```

`++` concatenates two `Map` instances. When the same key appears in  
both maps the value from the right operand is kept, making `++` a  
natural way to implement layered configuration where specific settings  
override generic defaults. The result is a fresh immutable map;  
neither input is modified.  

## Merging maps with custom conflict resolution

`merged` resolves duplicate keys using a caller-supplied function.  

```scala
@main def run() =
  val a = Map("x" -> 1, "y" -> 2, "z" -> 3)
  val b = Map("y" -> 20, "z" -> 30, "w" -> 40)

  val combined = a.merged(b)((_, av, bv) => av + bv)
  println(combined)
```

`merged` is available on `HashMap` and accepts a collision function  
`(K, V, V) => V` that receives the key and both values and returns  
the resolved value. Here the two values are summed for keys that exist  
in both maps. This is more expressive than `++` whenever the merge  
semantics depend on the existing values rather than simply replacing  
one with the other.  

## Transforming values with map

`map` transforms every key-value pair and returns a new collection.  

```scala
@main def run() =
  val prices = Map("apple" -> 1.20, "banana" -> 0.50, "cherry" -> 2.80)
  val discounted = prices.map((item, price) => item -> price * 0.9)

  discounted.foreach((item, price) =>
    println(f"$item: $price%.2f")
  )
```

Calling `map` on a `Map[K, V]` iterates over each `(K, V)` pair and  
applies the function to produce a new `(K2, V2)` pair. The result type  
is inferred from the function's return type; here it stays  
`Map[String, Double]` because both key and value types are unchanged.  
String interpolation with `f` enables formatted decimal output.  

## Transforming values with transform

`transform` keeps all keys and replaces each value using a  
key-aware function.  

```scala
@main def run() =
  val scores = Map("Alice" -> 80, "Bob" -> 75, "Carol" -> 90)
  val graded = scores.transform((name, score) =>
    if score >= 90 then "A"
    else if score >= 80 then "B"
    else "C"
  )
  println(graded)
```

`transform` is a convenience over `map` when the key set must stay  
identical and only values change. It guarantees that the resulting map  
has exactly the same keys as the original, which prevents accidental  
removal of entries. The function receives both the key and the value,  
so decisions can depend on either.  

## Filtering a map

`filter` keeps only the entries that satisfy a predicate.  

```scala
val capitals = Map(
  "Bratislava" -> 424207, "Vilnius" -> 556723,
  "Lisbon"     -> 564657, "Riga"    -> 713016,
  "Warsaw"     -> 1711324,"Budapest"-> 1729040,
  "Prague"     -> 1241664,"Helsinki"-> 596661,
  "Tokyo"      -> 13189000,"Madrid" -> 3233527
)

@main def run() =
  val large  = capitals.filter((_, pop) => pop > 1_000_000)
  val small  = capitals.filterNot((_, pop) => pop > 1_000_000)

  println("Large capitals:")
  large.foreach((city, pop) => println(s"  $city: $pop"))

  println("Small capitals:")
  small.foreach((city, pop) => println(s"  $city: $pop"))
```

`filter` accepts a predicate over `(K, V)` pairs and returns a new  
map containing only the matching entries. `filterNot` is the logical  
complement, keeping entries for which the predicate is `false`. Both  
methods preserve the `Map` type, so the result supports the full map  
API without any explicit conversion.  

## Counting and partitioning

`count` tallies matching entries; `partition` splits a map in two.  

```scala
val capitals = Map(
  "Bratislava" -> 424207, "Vilnius"  -> 556723,
  "Lisbon"     -> 564657, "Warsaw"   -> 1711324,
  "Prague"     -> 1241664,"Tokyo"    -> 13189000,
  "Madrid"     -> 3233527
)

@main def run() =
  val n = capitals.count((_, pop) => pop > 1_000_000)
  println(s"Cities over 1 million: $n")

  val (big, small) = capitals.partition((_, pop) => pop > 1_000_000)
  println(s"Big:   ${big.keys.mkString(", ")}")
  println(s"Small: ${small.keys.mkString(", ")}")
```

`count` returns the number of entries for which the predicate holds,  
performing a single pass over the map. `partition` makes two passes  
and returns a tuple of two maps; the left map contains matching entries  
and the right map contains the rest. Destructuring the tuple with  
`val (big, small)` gives both halves meaningful names immediately.  

## Sorting a map by key

Converting to a sequence enables key-ordered display.  

```scala
val capitals = Map(
  "Bratislava" -> 424207, "Vilnius"  -> 556723,
  "Lisbon"     -> 564657, "Riga"     -> 713016,
  "Warsaw"     -> 1711324,"Prague"   -> 1241664,
  "Helsinki"   -> 596661, "Tokyo"    -> 13189000,
  "Madrid"     -> 3233527
)

@main def run() =
  val byKey = capitals.toSeq.sortBy(_._1)
  byKey.foreach((city, pop) => println(s"$city: $pop"))
```

`Map` has no intrinsic sort order, so sorting requires converting to a  
`Seq[(K, V)]` first. `sortBy(_._1)` sorts the resulting sequence by  
the first element of each tuple, which is the key. The sorted sequence  
can then be printed, folded, or converted back to a `Map` with  
`.toMap` if ordering must be preserved via a `ListMap`.  

## Sorting a map by value

Sorting by value reveals ranked data such as population rankings.  

```scala
val capitals = Map(
  "Bratislava" -> 424207, "Vilnius"  -> 556723,
  "Lisbon"     -> 564657, "Tokyo"    -> 13189000,
  "Warsaw"     -> 1711324,"Madrid"   -> 3233527,
  "Helsinki"   -> 596661
)

@main def run() =
  println("Ascending by population:")
  capitals.toSeq.sortBy(_._2)
    .foreach((city, pop) => println(s"  $city: $pop"))

  println("Descending by population:")
  capitals.toSeq.sortWith(_._2 > _._2)
    .foreach((city, pop) => println(s"  $city: $pop"))
```

`sortBy(_._2)` sorts ascending by value; `sortWith` accepts a binary  
predicate and sorts in any order. The underscore shorthand keeps the  
lambda concise. After sorting, `foreach` with destructuring prints each  
pair without temporary variables.  

## Grouping a list into a map

`groupBy` partitions a collection into a map of lists.  

```scala
@main def run() =
  val words = List(
    "apple", "avocado", "banana", "blueberry",
    "cherry", "apricot", "blackberry"
  )

  val byLetter = words.groupBy(_.head)
  byLetter.toSeq.sortBy(_._1).foreach((ch, ws) =>
    println(s"$ch: ${ws.mkString(", ")}")
  )
```

`groupBy` applies a classifier function to each element and builds a  
`Map[K, List[A]]` where every list holds all elements that produced  
the same key. Here the classifier is `_.head`, the first character of  
each word, so all words starting with the same letter end up in the  
same bucket. Sorting the resulting map by key before printing gives  
alphabetical output.  

## Converting a list to a map

`zip` and `toMap` turn two parallel sequences into a map.  

```scala
@main def run() =
  val keys   = List("a", "b", "c", "d")
  val values = List(1, 2, 3, 4)

  val m = keys.zip(values).toMap
  println(m)
```

`zip` pairs up elements from two lists by position, producing a  
`List[(String, Int)]`. Calling `toMap` on any `Iterable[(K, V)]`  
converts it into an immutable `Map`. If duplicate keys appear in the  
zipped list, later entries overwrite earlier ones, keeping only the  
last value for each key.  

## Building a map with a for comprehension

A `for` comprehension with `yield` can construct a `Map` directly  
when combined with `toMap`.  

```scala
@main def run() =
  val squares = (for i <- 1 to 10 yield i -> i * i).toMap
  squares.toSeq.sortBy(_._1).foreach((k, v) =>
    println(s"$k² = $v")
  )
```

The `for` expression yields `(Int, Int)` tuples, and `.toMap` collects  
them into a `Map[Int, Int]`. The range `1 to 10` produces an inclusive  
sequence, so all integers from one to ten are present as keys. This  
pattern is a clean alternative to `Map.tabulate` for mappings that  
follow a formula.  

## Building a map with tabulate

`Map.tabulate` generates a map from an integer range via a function.  

```scala
@main def run() =
  val cubes = Map.from((1 to 8).map(i => i -> i * i * i))
  cubes.toSeq.sortBy(_._1).foreach((k, v) =>
    println(s"$k³ = $v")
  )
```

`Map.from` accepts any `Iterable[(K, V)]` and is the most explicit  
constructor when the entries come from a transformation chain. Mapping  
the range `1 to 8` with a tuple-producing lambda and feeding the result  
to `Map.from` is both readable and efficient. The pattern extends  
naturally to multi-dimensional keys such as `(i, j)`.  

## Converting between Map and List

`toList`, `toSeq`, and `toVector` materialise a map into a sequence.  

```scala
@main def run() =
  val m = Map("x" -> 10, "y" -> 20, "z" -> 30)

  val pairs: List[(String, Int)] = m.toList
  val keys:  List[String]        = m.keys.toList
  val vals:  List[Int]           = m.values.toList

  println(pairs)
  println(keys)
  println(vals)
```

Every `Map` extends `Iterable[(K, V)]`, so converting to a `List`,  
`Seq`, or `Vector` is a single method call. The resulting sequences  
can be sorted, grouped, or passed to functions that do not accept a  
map. Note that the order of elements in these sequences reflects the  
map's internal order, which is unspecified for `HashMap`.  

## Using tuples as map values

Maps can hold structured values such as tuples for multi-field records.  

```scala
@main def run() =
  val people = Map(
    "alice" -> ("Alice Smith",  32, "engineer"),
    "bob"   -> ("Bob Jones",   28, "designer"),
    "carol" -> ("Carol White", 45, "manager")
  )

  for (id, (name, age, role)) <- people do
    println(s"$id: $name, age $age, $role")
```

Tuple values group related fields without defining a separate class.  
Destructuring the tuple directly in the `for` pattern makes the  
individual fields immediately available without additional `._N` calls.  
For more than three or four fields a case class is clearer, but tuples  
are convenient for ad-hoc lightweight records.  

## Immutable vs mutable map

The default `Map` is immutable; `mutable.Map` supports in-place changes.  

```scala
import scala.collection.mutable

@main def run() =
  val immut = Map("a" -> 1, "b" -> 2)
  val mut   = mutable.Map("a" -> 1, "b" -> 2)

  // immut("a") = 10  // compile error
  mut("a") = 10

  println(immut)
  println(mut)
```

The immutable `Map` does not expose an `update` method, so any attempt  
to assign through `apply` is a compile error. The mutable variant  
exposes `update`, making `mut("a") = 10` syntactic sugar for  
`mut.update("a", 10)`. Choose the mutable variant only when profiling  
shows that creating new maps is a bottleneck, or when integrating with  
imperative code that requires in-place mutation.  

## Mutable map operations

`+=`, `-=`, and `++=` modify a mutable map in place.  

```scala
import scala.collection.mutable

@main def run() =
  val m = mutable.Map("a" -> 1, "b" -> 2)

  m += ("c" -> 3)
  m += ("d" -> 4, "e" -> 5)
  m -= "a"
  m ++= Map("f" -> 6, "g" -> 7)

  println(m)
```

`+=` inserts or replaces a single entry; passing multiple pairs inserts  
all of them atomically. `-=` removes a key, silently doing nothing if  
the key is absent. `++=` merges another `Iterable[(K, V)]` into the  
map. All three operators return the map itself, which allows chaining  
but also signals that the map is being mutated rather than copied.  

## getOrElseUpdate for mutable maps

`getOrElseUpdate` is a thread-unsafe but convenient cache pattern.  

```scala
import scala.collection.mutable

@main def run() =
  val cache = mutable.Map.empty[String, Int]

  def expensiveCompute(s: String): Int =
    println(s"computing $s...")
    s.length * 42

  val r1 = cache.getOrElseUpdate("hello", expensiveCompute("hello"))
  val r2 = cache.getOrElseUpdate("hello", expensiveCompute("hello"))
  val r3 = cache.getOrElseUpdate("world", expensiveCompute("world"))

  println(cache)
  println(r1, r2, r3)
```

`getOrElseUpdate` looks up the key; if found it returns the existing  
value without evaluating the second argument; if absent it evaluates  
the argument, stores the result under the key, and returns it. The  
trace message printed by `expensiveCompute` confirms that the function  
runs only on the first call per key. This pattern implements a simple  
memoisation cache without any extra infrastructure.  

## Maps with case class keys

Case classes implement structural equality and are safe map keys.  

```scala
case class Point(x: Int, y: Int)

@main def run() =
  val grid = Map(
    Point(0, 0) -> "origin",
    Point(1, 0) -> "right",
    Point(0, 1) -> "up",
    Point(1, 1) -> "diagonal"
  )

  println(grid(Point(0, 0)))
  println(grid.get(Point(2, 2)))
  println(grid.contains(Point(1, 1)))
```

Case classes automatically synthesise `equals` and `hashCode` based on  
their fields, which are the two contracts a map key must honour. Two  
`Point` instances with the same coordinates are therefore considered  
equal and hash to the same bucket, so `grid(Point(0, 0))` reliably  
finds the entry created with a different `Point(0, 0)` instance.  

## Maps with complex keys

Composite keys formed from tuples group multi-dimensional data.  

```scala
@main def run() =
  val temps = Map(
    ("London",  "Jan") -> -1.2,
    ("London",  "Jul") -> 18.5,
    ("Tokyo",   "Jan") ->  5.2,
    ("Tokyo",   "Jul") -> 27.8,
    ("Sydney",  "Jan") -> 22.1,
    ("Sydney",  "Jul") ->  8.6
  )

  val londonJul = temps(("London", "Jul"))
  println(f"London July average: $londonJul%.1f °C")

  temps.toSeq.sortBy(_._1).foreach:
    case ((city, month), temp) =>
      println(f"$city%-10s $month: $temp%5.1f °C")
```

Tuple keys combine multiple fields into a single compound key without  
defining a separate class. Scala's structural equality for tuples  
ensures correct lookup behaviour. The pattern `case ((city, month),  
temp)` in the `foreach` lambda destructures both the key tuple and  
the outer pair in one step, keeping the code concise.  

## Maps with generic types

Generic functions accept maps of any type through type parameters.  

```scala
def topN[K, V: Ordering](m: Map[K, V], n: Int): Map[K, V] =
  m.toSeq.sortBy(_._2).takeRight(n).toMap

@main def run() =
  val scores  = Map("Alice" -> 88, "Bob" -> 72, "Carol" -> 95, "Dave" -> 80)
  val prices  = Map("pen" -> 1.50, "book" -> 12.99, "bag" -> 24.50)

  println(topN(scores, 2))
  println(topN(prices, 2))
```

The type parameter `V: Ordering` is a context bound that requires an  
implicit `Ordering[V]` to be available, enabling `sortBy` to compare  
values. Standard types such as `Int` and `Double` have built-in  
orderings, so no extra import is needed. The function works for any  
`Map[K, V]` where values can be ordered, demonstrating how generics  
and type classes interact in Scala 3.  

## Maps with given instances

A `given` instance can provide a default map for a context parameter.  

```scala
case class AppConfig(host: String, port: Int, debug: Boolean)

object AppConfig:
  given default: AppConfig = AppConfig("localhost", 8080, false)

def describe(using cfg: AppConfig): String =
  s"${cfg.host}:${cfg.port} debug=${cfg.debug}"

@main def run() =
  println(describe)

  given AppConfig = AppConfig("prod.example.com", 443, false)
  println(describe)
```

`given` instances participate in implicit resolution. The function  
`describe` declares a context parameter `using cfg: AppConfig`, so the  
compiler searches for a `given AppConfig` in scope. The companion  
object supplies a default; a locally declared `given` shadows it in  
the second call. Although this example does not use a `Map` directly,  
the same pattern is used to inject maps as context parameters in  
larger applications.  

## Map with given Ordering

A custom `given Ordering` changes how map entries are sorted.  

```scala
@main def run() =
  val words = Map(
    "banana" -> 6,
    "apple"  -> 5,
    "cherry" -> 6,
    "date"   -> 4
  )

  given Ordering[(String, Int)] =
    Ordering.by((_, len) => (-len, _))

  val sorted = words.toSeq.sorted
  sorted.foreach((w, n) => println(s"$w ($n)"))
```

Defining a `given Ordering[(String, Int)]` in scope replaces the  
default lexicographic ordering used by `sorted`. The custom ordering  
sorts by descending length first, then alphabetically within equal  
lengths, so longer words appear before shorter ones. Providing the  
ordering as a `given` rather than passing it explicitly keeps the  
call site clean and allows it to be overridden by any inner scope.  

## Folding a map

`foldLeft` accumulates a single result by visiting all entries.  

```scala
@main def run() =
  val cart = Map(
    "apple"  -> (3, 0.99),
    "bread"  -> (1, 2.49),
    "milk"   -> (2, 1.15),
    "cheese" -> (1, 4.75)
  )

  val total = cart.foldLeft(0.0):
    case (acc, (_, (qty, price))) => acc + qty * price

  println(f"Total: $$$total%.2f")
```

`foldLeft` starts with an accumulator (`0.0`) and applies the function  
to each `(K, V)` pair in turn. Destructuring the pair inline with a  
`case` clause keeps the lambda readable even with nested tuples. The  
result is the running total of `quantity * price` across all cart  
items. `foldLeft` is the functional equivalent of an imperative loop  
with a mutable accumulator.  

## Aggregating with groupMapReduce

`groupMapReduce` groups, transforms, and reduces in a single pass.  

```scala
@main def run() =
  val orders = List(
    ("Alice", "apple",  3),
    ("Bob",   "banana", 5),
    ("Alice", "banana", 2),
    ("Bob",   "apple",  1),
    ("Carol", "cherry", 4)
  )

  val totalByCustomer = orders.groupMapReduce(_._1)(_._3)(_ + _)
  totalByCustomer.toSeq.sortBy(_._1).foreach:
    (customer, total) => println(s"$customer: $total items")
```

`groupMapReduce` is a powerful one-liner that replaces a common  
`groupBy` + `mapValues` + `reduce` chain. The first lambda selects  
the grouping key (`_._1` is the customer name); the second transforms  
each element to the value to accumulate (`_._3` is the quantity); the  
third combines two accumulated values with `+`. The result is a  
`Map[String, Int]` of total quantities per customer.  

## Flattening a map of lists

`flatMap` turns a map of lists into a flat sequence of pairs.  

```scala
@main def run() =
  val grouped = Map(
    "fruits"     -> List("apple", "banana", "cherry"),
    "vegetables" -> List("carrot", "broccoli"),
    "grains"     -> List("rice", "wheat", "oat")
  )

  val flat = grouped.flatMap((category, items) =>
    items.map(item => item -> category)
  )

  flat.toSeq.sortBy(_._1).foreach((item, cat) =>
    println(s"$item -> $cat")
  )
```

`flatMap` on a `Map` applies a function that returns an  
`IterableOnce[(K2, V2)]` to every entry and concatenates the  
results into a single `Map`. Here each `(category, items)` entry is  
expanded into one `(item, category)` pair per item, effectively  
inverting the grouping. The final `sortBy` and `foreach` print the  
resulting reverse index in alphabetical order.  

## Recursion over a map

Recursive functions process maps without mutable state.  

```scala
@main def run() =
  val tree = Map(
    1 -> List(2, 3),
    2 -> List(4, 5),
    3 -> List(6),
    4 -> Nil,
    5 -> Nil,
    6 -> Nil
  )

  def reachable(node: Int, visited: Set[Int] = Set.empty): Set[Int] =
    if visited.contains(node) then visited
    else
      val children = tree.getOrElse(node, Nil)
      children.foldLeft(visited + node)(reachable)

  println(reachable(1))
  println(reachable(3))
```

`reachable` is a depth-first traversal that uses a `Set` to track  
visited nodes and avoids infinite loops in cyclic graphs. The map  
`tree` represents an adjacency list. `foldLeft` threads the growing  
`visited` set through recursive calls, accumulating all nodes  
reachable from the starting node without any mutable variable.  

## Inverting a map

Swapping keys and values produces the inverse mapping.  

```scala
@main def run() =
  val codeToName = Map(
    "sk" -> "Slovakia",
    "de" -> "Germany",
    "fr" -> "France",
    "jp" -> "Japan"
  )

  val nameToCode = codeToName.map((k, v) => v -> k)
  println(nameToCode("Slovakia"))
  println(nameToCode.get("Canada"))
```

Inverting is a one-liner: `map` the pairs and swap key and value.  
If the original map has duplicate values the inversion is lossy because  
`toMap` keeps only the last entry for each new key. Ensuring that  
values are unique before inverting is the caller's responsibility.  
The resulting map is a plain immutable `Map` with the same performance  
characteristics as the original.  

## Finding the maximum value

`maxBy` locates the entry with the largest value in one expression.  

```scala
val capitals = Map(
  "Bratislava" -> 424207,  "Vilnius" -> 556723,
  "Lisbon"     -> 564657,  "Warsaw"  -> 1711324,
  "Tokyo"      -> 13189000,"Madrid"  -> 3233527,
  "Helsinki"   -> 596661
)

@main def run() =
  val (city, pop) = capitals.maxBy(_._2)
  println(s"Largest: $city with population $pop")

  val (smallest, minPop) = capitals.minBy(_._2)
  println(s"Smallest: $smallest with population $minPop")
```

`maxBy` and `minBy` traverse the map in one pass and return the  
`(K, V)` pair for which the selector produces the maximum or minimum  
value. Destructuring the result immediately avoids `._1`/`._2` noise.  
Both methods throw `UnsupportedOperationException` on an empty map,  
so guard with `nonEmpty` or use `maxByOption`/`minByOption` which  
return `Option[(K, V)]`.  

## Summing values

`values.sum` or `foldLeft` aggregates all map values.  

```scala
@main def run() =
  val budget = Map(
    "engineering" -> 120_000,
    "marketing"   ->  80_000,
    "support"     ->  45_000,
    "research"    -> 200_000
  )

  val total   = budget.values.sum
  val average = total.toDouble / budget.size

  println(f"Total:   $$$total%,d")
  println(f"Average: $$$average%,.0f")
```

`values` returns a lazy `Iterable[V]`, and calling `sum` on it adds  
all values in one traversal. Dividing by `budget.size` gives the mean  
without an explicit loop. The `%,d` format specifier inserts thousands  
separators, making large numbers easier to read.  

## Collecting map results

`collect` applies a partial function to filter and transform simultaneously.  

```scala
@main def run() =
  val mixed: Map[String, Any] = Map(
    "age"    -> 30,
    "name"   -> "Alice",
    "score"  -> 95,
    "active" -> true,
    "level"  -> 5
  )

  val ints: Map[String, Int] = mixed.collect:
    case (k, v: Int) => k -> v

  println(ints)
```

`collect` uses a partial function to match only entries whose values  
are of type `Int`, ignoring `String` and `Boolean` entries. The type  
annotation `Map[String, Int]` on `ints` confirms that the result  
contains only integer values. This pattern is a clean alternative to  
`filter` followed by `map` when the transformation and the predicate  
are naturally expressed as a single pattern.  

## Converting map to a sorted map

`TreeMap` maintains keys in sorted order automatically.  

```scala
import scala.collection.immutable.TreeMap

@main def run() =
  val unsorted = Map("banana" -> 2, "apple" -> 5, "cherry" -> 1, "date" -> 3)
  val sorted   = TreeMap.from(unsorted)

  sorted.foreach((k, v) => println(s"$k: $v"))
```

`TreeMap` is a red-black tree that keeps all keys in their natural  
`Ordering`, so iteration is always alphabetical (or numerically  
ordered for numeric keys) without any explicit sort call. Constructing  
it from an existing map via `TreeMap.from` is idiomatic. Lookup,  
insertion, and deletion run in O(log n) — slightly slower than  
`HashMap`'s O(1) amortised — but the deterministic order is often  
worth the trade-off.  

## Using ListMap for insertion-ordered maps

`ListMap` preserves the insertion order of its entries.  

```scala
import scala.collection.immutable.ListMap

@main def run() =
  val lm = ListMap(
    "first"  -> 1,
    "second" -> 2,
    "third"  -> 3,
    "fourth" -> 4
  )

  lm.foreach((k, v) => println(s"$k -> $v"))
```

`ListMap` is backed by a linked list of pairs, so iteration always  
follows insertion order. This comes at a cost: lookup is O(n) rather  
than O(1) because there is no hash table. `ListMap` is appropriate  
when the map is small, order matters, and random access is rare.  
For large ordered maps, `TreeMap` or a sorted `Seq` is a better choice.  

## Chaining map operations

Multiple transformations can be chained without intermediate variables.  

```scala
@main def run() =
  val raw = Map(
    " Alice " -> " 85 ",
    " Bob "   -> " 72 ",
    " Carol " -> " 91 ",
    " Dave "  -> " 60 "
  )

  val cleaned = raw
    .map((k, v) => k.trim -> v.trim.toInt)
    .filter((_, score) => score >= 70)
    .transform((name, score) =>
      if score >= 90 then s"$name: A"
      else s"$name: B"
    )

  cleaned.toSeq.sortBy(_._2).foreach(println)
```

Chaining `map`, `filter`, and `transform` on a single expression  
reads like a data pipeline: trim and parse the raw data, discard  
failing scores, then format the passing ones. Each step returns a  
new `Map`, and the chain is evaluated lazily where possible. This  
style avoids mutable intermediate state and makes the transformation  
sequence easy to audit.  

## Real-world word frequency count

Counting word occurrences in a sentence with `groupMapReduce`.  

```scala
@main def run() =
  val text = "to be or not to be that is the question to be"

  val freq = text.split(" ")
    .groupMapReduce(identity)(_ => 1)(_ + _)

  freq.toSeq.sortBy(-_._2).foreach:
    (word, count) => println(s"$word: $count")
```

`split(" ")` tokenises the text into an `Array[String]`. Then  
`groupMapReduce` groups by the word itself (`identity`), maps each  
occurrence to `1`, and sums the ones. The result is a frequency map  
in a single chained expression. Sorting by `-_._2` (negated count)  
places the most frequent words first, which is the conventional  
presentation of frequency tables.  

## Inverting a grouped map

Transforming a `Map[K, List[V]]` into a `Map[V, K]` flattens groups.  

```scala
@main def run() =
  val teamMembers = Map(
    "engineering" -> List("Alice", "Bob", "Carol"),
    "marketing"   -> List("Dave", "Eve"),
    "support"     -> List("Frank", "Grace", "Heidi")
  )

  val memberTeam = teamMembers.flatMap:
    (team, members) => members.map(_ -> team)

  memberTeam.toSeq.sortBy(_._1).foreach:
    (member, team) => println(s"$member -> $team")
```

`flatMap` expands each `(team, members)` entry into a list of  
`(member, team)` pairs and flattens all lists into a single map.  
This is the canonical way to produce a reverse index: from a one-to-many  
structure to a many-to-one structure. The sorted output makes it easy  
to look up any team member alphabetically.  

## Histogram from a list

`groupBy` on a transformed list builds a distribution map.  

```scala
@main def run() =
  val scores = List(55, 72, 88, 61, 90, 73, 85, 44, 92, 67, 78, 82)

  val histogram = scores
    .groupBy(_ / 10 * 10)
    .map((bucket, vals) => bucket -> vals.size)

  histogram.toSeq.sortBy(_._1).foreach:
    (bucket, count) =>
      val bar = "#" * count
      println(f"$bucket%3d: $bar")
```

Dividing each score by ten and multiplying back by ten buckets it into  
the nearest decade (40, 50, 60, …). `groupBy` collects all scores in  
the same decade into a list. `map` then replaces each list with its  
length, producing the histogram counts. The `#` bar gives a quick  
visual representation of the distribution.  

## Configuration map pattern

Parsing a settings string into a typed configuration map.  

```scala
case class Config(settings: Map[String, String]):
  def get(key: String): Option[String] = settings.get(key)
  def getInt(key: String): Option[Int] = settings.get(key).map(_.toInt)
  def getBool(key: String): Option[Boolean] =
    settings.get(key).map(_.toBoolean)

@main def run() =
  val raw = "host=localhost port=5432 debug=true max_conn=10"

  val cfg = Config(
    raw.split(" ")
      .map(_.split("="))
      .collect { case Array(k, v) => k -> v }
      .toMap
  )

  println(cfg.get("host"))
  println(cfg.getInt("port"))
  println(cfg.getBool("debug"))
  println(cfg.getInt("missing"))
```

The `raw` string is split by spaces into `key=value` tokens, each  
token is split again on `=` to produce a two-element array, and  
`collect` uses a pattern guard to safely skip malformed tokens. The  
result is a `Map[String, String]` wrapped in a `Config` value class  
that provides typed accessors. All accessors return `Option` so the  
caller can handle missing keys explicitly.  

## Memoisation with a mutable map

A mutable map caches expensive recursive computations.  

```scala
import scala.collection.mutable

val memo = mutable.Map.empty[Int, BigInt]

def fib(n: Int): BigInt =
  if n <= 1 then BigInt(n)
  else memo.getOrElseUpdate(n, fib(n - 1) + fib(n - 2))

@main def run() =
  for i <- 0 to 15 do
    print(s"${fib(i)} ")
  println()
```

Without memoisation, computing Fibonacci numbers recursively is  
exponential in time. Adding a `mutable.Map` as a cache turns it into  
a linear algorithm: `getOrElseUpdate` returns the stored result on  
the second call without re-evaluating the recursive expression. The  
`memo` map persists between calls because it is defined at the top  
level, not inside `fib`.  

## Performance: HashMap vs TreeMap

Comparing lookup times illustrates when each structure is appropriate.  

```scala
import scala.collection.immutable.TreeMap

@main def run() =
  val n       = 1_000_000
  val pairs   = (1 to n).map(i => i -> i.toString)
  val hashMap = pairs.toMap
  val treeMap = TreeMap.from(pairs)

  val keys = List(1, 100_000, 500_000, 999_999)

  val t0 = System.nanoTime()
  keys.foreach(k => hashMap(k))
  val hashMs = (System.nanoTime() - t0) / 1_000_000.0

  val t1 = System.nanoTime()
  keys.foreach(k => treeMap(k))
  val treeMs = (System.nanoTime() - t1) / 1_000_000.0

  println(f"HashMap lookup:  $hashMs%.3f ms")
  println(f"TreeMap lookup:  $treeMs%.3f ms")
```

Building one million entries and looking up four keys demonstrates  
the difference in constant factors between hash-based and tree-based  
lookup. `HashMap` offers O(1) amortised lookup because it computes a  
hash and directly addresses a bucket. `TreeMap` offers O(log n) lookup  
by navigating a balanced binary tree. For most workloads `HashMap`  
is faster; choose `TreeMap` when sorted iteration or range queries  
matter more than raw lookup speed.  

## Idiomatic map processing

Combining `groupBy`, `mapValues`, and `toSeq` in a real-world pipeline.  

```scala
case class Sale(region: String, product: String, amount: Double)

@main def run() =
  val sales = List(
    Sale("North", "Widget", 120.0),
    Sale("South", "Gadget", 340.0),
    Sale("North", "Gadget",  85.0),
    Sale("East",  "Widget", 200.0),
    Sale("South", "Widget",  95.0),
    Sale("East",  "Gadget", 175.0),
    Sale("North", "Widget",  60.0)
  )

  val revenueByRegion = sales
    .groupBy(_.region)
    .map((region, ss) => region -> ss.map(_.amount).sum)

  println("Revenue by region:")
  revenueByRegion.toSeq.sortBy(-_._2).foreach:
    (region, total) => println(f"  $region%-6s  $$$total%7.2f")
```

`groupBy(_.region)` partitions the sale list into per-region lists.  
Mapping over the grouped result with `map(_.amount).sum` reduces each  
list to a single revenue total. Sorting by negated total puts the  
highest-revenue region first. This pipeline is idiomatic Scala: no  
mutable state, no loops, and each transformation is expressed as a  
single method call whose intent is immediately clear.
