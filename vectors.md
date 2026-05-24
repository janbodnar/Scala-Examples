# Vectors

A `Vector` in Scala 3 is an immutable, indexed sequence backed by a  
wide, shallow tree known as a bit-mapped vector trie. It is the  
default general-purpose immutable collection in the Scala standard  
library and the recommended choice whenever you need a collection  
that supports both efficient indexed access and efficient functional  
updates without mutating the original data.  

**Vectors vs Lists, Arrays, and Seqs**  

A `List` is a singly-linked structure optimised for O(1) head access  
and prepending. Random access on a `List` is O(n) because every  
lookup walks the chain from the start. A `Vector` stores elements in  
a tree with a branching factor of 32, making indexed access and  
functional update effectively O(1) (strictly O(log\u2083\u2082 n), which is  
at most five levels for any realistic collection size). An `Array` is  
a mutable, flat JVM array with true O(1) indexing, but it is not  
persistent and cannot be safely shared without defensive copying.  
`Seq` is an abstract trait that `List`, `Vector`, and other sequence  
types implement; it represents any ordered sequence without  
committing to a concrete representation.  

**Immutability and persistent data structures**  

`Vector` is fully immutable: no method ever modifies an existing  
instance. Operations such as `updated`, `appended`, and `prepended`  
return a new `Vector` that shares most of its internal tree nodes  
with the original. This structural sharing means that creating a  
modified copy costs time and memory proportional to the depth of  
the tree, not to the total number of elements.  

**Performance characteristics**  

| Operation            | Complexity              |  
|----------------------|-------------------------|  
| `apply` (index)      | effectively O(1)        |  
| `updated`            | effectively O(1)        |  
| `:+` (append)        | effectively O(1) amort. |  
| `+:` (prepend)       | effectively O(1) amort. |  
| `length`             | O(1)                    |  
| `++` (concatenation) | O(n)                    |  
| iteration            | O(n)                    |  

**When to use Vector**  

Prefer `Vector` when you need random access, when the collection  
will be updated functionally at arbitrary positions, or when you  
want a safe default for any immutable sequence. Use `List` when you  
primarily prepend and recurse from the head. Use `Array` only when  
mutable, flat storage is required, typically at Java interop  
boundaries.  

---

## Creating a vector

The companion object provides the most direct way to construct a  
`Vector`.  

```scala
@main def main() =
  val nums  = Vector(1, 2, 3, 4, 5)
  val words = Vector("sky", "blue", "forest")
  val empty = Vector.empty[Int]

  println(nums)
  println(words)
  println(empty)
end main
```

`Vector.apply` accepts a variable number of arguments and builds an  
immutable vector in O(n) time. The type parameter is inferred from  
the arguments; for an empty vector it must be stated explicitly  
because there are no elements to infer from. Printing a vector  
displays its elements in the standard `Vector(...)` format.  

## Creating from a range

A numeric range can be materialised into a `Vector` with a single  
call.  

```scala
@main def main() =
  val r1 = (1 to 10).toVector
  val r2 = (0 until 20 by 2).toVector

  println(r1)
  println(r2)
end main
```

The `Range` type's `toVector` method evaluates each integer in the  
range and stores it in a new `Vector`. The `to` endpoint is  
inclusive while `until` is exclusive. The `by` clause sets the step  
size, so `0 until 20 by 2` produces the even numbers from 0 to 18.  

## Accessing elements

Individual elements can be retrieved by index or by named helper  
methods.  

```scala
@main def main() =
  val v = Vector(10, 20, 30, 40, 50)

  println(v(0))
  println(v(2))
  println(v.head)
  println(v.last)
  println(v.tail)
  println(v.init)
end main
```

Indexed access via `apply` uses the same parenthesis syntax as  
ordinary method calls and runs in effectively O(1) time. `head`  
returns the first element and `last` returns the final element; both  
are O(1) on `Vector`, unlike on `List` where `last` is O(n). `tail`  
returns all elements except the first, and `init` returns all  
elements except the last. Accessing an out-of-bounds index throws  
`IndexOutOfBoundsException`.  

## Checking size and emptiness

The length and emptiness of a vector can be inspected in O(1) time.  

```scala
@main def main() =
  val v     = Vector(1, 2, 3)
  val empty = Vector.empty[String]

  println(v.size)
  println(v.length)
  println(v.isEmpty)
  println(v.nonEmpty)
  println(empty.isEmpty)
end main
```

`size` and `length` are aliases that return the number of elements  
in O(1) time; this differs from `List`, where `size` is O(n).  
`isEmpty` and `nonEmpty` are O(1) predicates that should always be  
preferred over comparing the length to zero.  

## Updating elements

`Vector` supports functional update at any index without mutating  
the original.  

```scala
@main def main() =
  val v       = Vector(1, 2, 3, 4, 5)
  val updated = v.updated(2, 99)

  println(v)
  println(updated)
end main
```

`updated(index, value)` returns a new vector identical to the  
original except that the element at the given index is replaced  
with the new value. The original vector is unchanged. Because of  
structural sharing the new vector reuses most of the internal tree  
nodes from the original, making the operation effectively O(1)  
rather than O(n).  

## Appending an element

A new element can be appended to the end of a vector efficiently.  

```scala
@main def main() =
  val v  = Vector(1, 2, 3)
  val v2 = v :+ 4
  val v3 = v2 :+ 5

  println(v)
  println(v2)
  println(v3)
end main
```

The `:+` operator produces a new vector with the element added at  
the end. Unlike `List`, where appending is O(n), `Vector` supports  
amortised O(1) append thanks to its tree structure. Each  
intermediate vector is fully independent and the original is never  
mutated.  

## Prepending an element

A new element can be added to the front of a vector.  

```scala
@main def main() =
  val v  = Vector(3, 4, 5)
  val v2 = 2 +: v
  val v3 = 1 +: v2

  println(v)
  println(v2)
  println(v3)
end main
```

The `+:` operator adds an element at the start and returns a new  
vector in amortised O(1) time. Both `+:` and `:+` have the same  
asymptotic cost on `Vector`. By contrast, prepending to a `List`  
with `::` is O(1) but appending with `:+` on a `List` is O(n). The  
symmetric performance of `Vector` makes it the right choice when  
the insertion direction is not fixed in advance.  

## Concatenating vectors

Two vectors can be joined into a single vector with `++`.  

```scala
@main def main() =
  val a = Vector(1, 2, 3)
  val b = Vector(4, 5, 6)
  val c = a ++ b

  println(a)
  println(b)
  println(c)
end main
```

The `++` operator concatenates two sequences and returns a new  
`Vector`. The operation is O(n) in the total number of elements.  
The originals remain unchanged. `appendedAll` is an equivalent  
method that accepts any `IterableOnce`, making it useful for joining  
a vector with a `List`, an `Array`, or any other iterable.  

## Slicing a vector

A contiguous sub-range of elements can be extracted from a vector.  

```scala
@main def main() =
  val v  = Vector(10, 20, 30, 40, 50, 60, 70)
  val s1 = v.slice(1, 4)
  val s2 = v.take(3)
  val s3 = v.drop(4)
  val s4 = v.takeRight(2)
  val s5 = v.dropRight(3)

  println(s1)
  println(s2)
  println(s3)
  println(s4)
  println(s5)
end main
```

`slice(from, until)` returns elements from index `from` up to but  
not including `until`. `take(n)` returns the first `n` elements and  
`drop(n)` skips them. `takeRight(n)` and `dropRight(n)` operate from  
the end. All these operations return new vectors and leave the  
original unchanged. Indices are automatically clamped to valid  
bounds, so there is no risk of an exception from over-slicing.  

## Taking and dropping with a predicate

Elements can be taken or dropped based on a condition rather than  
a fixed count.  

```scala
@main def main() =
  val nums = Vector(2, 4, 6, 7, 8, 9, 10)

  val taken   = nums.takeWhile(_ % 2 == 0)
  val dropped = nums.dropWhile(_ % 2 == 0)

  println(taken)
  println(dropped)
end main
```

`takeWhile` collects elements from the front as long as the  
predicate holds and stops at the first element that fails.  
`dropWhile` removes elements from the front while the predicate  
holds and returns the rest. Both stop at the first failing element,  
so they are not the same as `filter` and `filterNot`, which scan  
the entire vector. Together they split the vector at the first  
mismatch.  

## Mapping over a vector

`map` transforms every element of a vector with a given function.  

```scala
@main def main() =
  val nums = Vector(1, 2, 3, 4, 5)

  val doubled  = nums.map(_ * 2)
  val asString = nums.map(n => s"item-$n")

  println(doubled)
  println(asString)
end main
```

`map` applies the supplied function to each element and collects the  
results into a new `Vector` of the same length. The result type can  
differ from the input type, as shown when converting integers to  
formatted strings. The original vector is not modified. Using `_`  
as a placeholder is idiomatic when the function body references the  
argument exactly once.  

## Filtering a vector

`filter` keeps only the elements that satisfy a predicate.  

```scala
@main def main() =
  val nums = Vector(1, 2, 3, 4, 5, 6, 7, 8)

  val evens = nums.filter(_ % 2 == 0)
  val odds  = nums.filterNot(_ % 2 == 0)
  val big   = nums.filter(_ > 5)

  println(evens)
  println(odds)
  println(big)
end main
```

`filter` returns a new vector containing only the elements for which  
the predicate returns `true`. `filterNot` is the complement and keeps  
elements where the predicate is `false`. Both operations traverse the  
full vector in O(n) time. When you need to split the vector into two  
groups simultaneously, `partition` is more efficient than calling  
`filter` twice.  

## FlatMapping a vector

`flatMap` maps each element to a vector and flattens the results  
into one.  

```scala
@main def main() =
  val sentences = Vector("hello world", "foo bar baz")
  val words     = sentences.flatMap(_.split(" ").toVector)
  val pairs     = Vector(1, 2, 3).flatMap(n => Vector(n, n * 10))

  println(words)
  println(pairs)
end main
```

`flatMap` applies a function that returns a sequence to every  
element and concatenates all results into a single vector. It is the  
fundamental operation behind multi-generator `for`-expressions. The  
first example tokenises each sentence by splitting on spaces; the  
second duplicates each number alongside its tenfold.  

## Folding a vector

`foldLeft` accumulates a result by visiting elements left to right.  

```scala
@main def main() =
  val nums = Vector(1, 2, 3, 4, 5)

  val sum     = nums.foldLeft(0)(_ + _)
  val product = nums.foldLeft(1)(_ * _)
  val joined  = nums.foldLeft("")((acc, n) => acc + n.toString)

  println(sum)
  println(product)
  println(joined)
end main
```

`foldLeft(initial)(op)` starts with the initial value and applies  
the binary operation to the accumulator and each element in  
left-to-right order. `foldRight` works analogously from right to  
left. The fold abstraction can express summation, product, string  
building, and nearly any other sequential aggregation. Because  
`foldLeft` is implemented iteratively on `Vector`, it is safe for  
large inputs without risk of a stack overflow.  

## Reducing a vector

`reduce` combines all elements without requiring an initial value.  

```scala
@main def main() =
  val nums = Vector(3, 1, 4, 1, 5, 9, 2, 6)

  val sum = nums.reduce(_ + _)
  val max = nums.reduce(_ max _)
  val min = nums.reduce(_ min _)

  println(sum)
  println(max)
  println(min)
end main
```

`reduce` requires a non-empty collection; calling it on an empty  
vector throws `UnsupportedOperationException`. The safer alternative  
`reduceOption` returns `Some(result)` or `None`. `reduceLeft` and  
`reduceRight` make the associativity explicit. For commutative and  
associative operations such as addition and multiplication the  
direction does not affect the result.  

## Scanning a vector

`scanLeft` builds a running accumulation of every prefix.  

```scala
@main def main() =
  val nums = Vector(1, 2, 3, 4, 5)

  val runningSum  = nums.scanLeft(0)(_ + _)
  val runningProd = nums.scanLeft(1)(_ * _)

  println(runningSum)
  println(runningProd)
end main
```

`scanLeft(initial)(op)` returns a vector of length `n + 1` where  
the first element is the initial value and each subsequent element  
is the result of applying the operation to the previous accumulator  
and the next input element. This makes it easy to compute running  
totals, cumulative products, or any other progressive aggregation.  
`scanRight` produces the same result working from the other end.  

## Zipping two vectors

Two vectors of equal length can be paired element-by-element.  

```scala
@main def main() =
  val names  = Vector("Alice", "Bob", "Carol")
  val scores = Vector(95, 87, 92)

  val zipped = names.zip(scores)
  zipped.foreach((name, score) => println(s"$name: $score"))
end main
```

`zip` creates a `Vector` of tuples, pairing the element at each  
index from the two input vectors. If the vectors have different  
lengths the result is truncated to the shorter one. `zipAll` fills  
in a default value for the shorter side. The two-argument lambda  
in `foreach` uses the automatic tuple-destructuring available in  
Scala 3.  

## Unzipping a vector of pairs

A vector of pairs can be split back into two separate vectors.  

```scala
@main def main() =
  val pairs = Vector(("Alice", 95), ("Bob", 87), ("Carol", 92))

  val (names, scores) = pairs.unzip

  println(names)
  println(scores)
end main
```

`unzip` decomposes a `Vector[(A, B)]` into a pair of vectors  
`(Vector[A], Vector[B])`. The result is destructured directly in a  
`val` pattern so both halves are available under meaningful names.  
The operation is O(n) and produces two fully independent vectors that  
do not share structure with the original pair vector.  

## Zipping with index

`zipWithIndex` attaches the position of each element to it.  

```scala
@main def main() =
  val fruits = Vector("apple", "banana", "cherry")

  val indexed = fruits.zipWithIndex
  indexed.foreach((fruit, i) => println(s"$i: $fruit"))
end main
```

`zipWithIndex` is equivalent to `zip(indices)` but more concise and  
slightly more efficient. It returns a `Vector[(A, Int)]` where the  
second component is the zero-based index. The destructuring syntax  
in the lambda avoids explicit `._1` and `._2` access, keeping the  
loop body readable.  

## Grouping elements

`groupBy` partitions a vector into a map of groups keyed by a  
classifier function.  

```scala
@main def main() =
  val nums = Vector(1, 2, 3, 4, 5, 6, 7, 8)

  val byParity = nums.groupBy(n =>
    if n % 2 == 0 then "even" else "odd"
  )

  byParity.foreach((key, group) => println(s"$key -> $group"))
end main
```

`groupBy` applies the classifier function to every element and  
collects elements that produce the same key into a `Vector` stored  
under that key in a `Map`. The order of elements within each group  
preserves the original order. The resulting map type is  
`Map[K, Vector[A]]` when called on a `Vector[A]`.  

## Partitioning a vector

`partition` splits a vector into two parts based on a predicate.  

```scala
@main def main() =
  val nums = Vector(1, 2, 3, 4, 5, 6, 7, 8)

  val (evens, odds) = nums.partition(_ % 2 == 0)

  println(evens)
  println(odds)
end main
```

`partition` traverses the vector once and returns a pair  
`(Vector[A], Vector[A])`, where the first vector contains elements  
satisfying the predicate and the second contains the rest. It is  
more efficient than calling `filter` and `filterNot` separately  
because the input is only traversed once. The result pair can be  
destructured directly in a `val` declaration.  

## Sorting a vector

`sorted` arranges elements in their natural order.  

```scala
@main def main() =
  val nums  = Vector(5, 1, 3, 2, 4)
  val words = Vector("banana", "apple", "cherry", "date")

  println(nums.sorted)
  println(words.sorted)
  println(nums.sortWith(_ > _))
end main
```

`sorted` requires an implicit `Ordering` for the element type.  
Standard types such as `Int`, `String`, and `Double` already have  
orderings provided by the standard library. `sortWith` accepts a  
two-argument comparison function, making it easy to express  
descending or custom orderings without defining a full `Ordering`  
instance.  

## Sorting with a custom comparator

Complex objects can be sorted by one or more fields using `sortBy`.  

```scala
@main def main() =
  case class Person(name: String, age: Int)

  val people = Vector(
    Person("Alice", 30),
    Person("Bob",   25),
    Person("Carol", 35),
    Person("Dave",  25)
  )

  val byAge         = people.sortBy(_.age)
  val byAgeThenName = people.sortBy(p => (p.age, p.name))

  byAge.foreach(println)
  println("---")
  byAgeThenName.foreach(println)
end main
```

`sortBy` extracts a sort key from each element and uses the implicit  
`Ordering` for that key type. Returning a tuple from the key  
extractor provides lexicographic multi-field sorting: Scala derives  
an `Ordering` for tuples automatically. `sortBy` is generally  
cleaner than `sortWith` when the comparison can be expressed as a  
key projection.  

## Working with Option elements

A vector may contain `Option` values that need to be filtered and  
unwrapped.  

```scala
@main def main() =
  val maybes = Vector(Some(1), None, Some(3), None, Some(5))

  val defined = maybes.flatten
  val doubled = maybes.collect { case Some(n) => n * 2 }
  val withDef = maybes.map(_.getOrElse(0))

  println(defined)
  println(doubled)
  println(withDef)
end main
```

`flatten` on a `Vector[Option[A]]` discards all `None` values and  
unwraps the `Some` values into a `Vector[A]`. `collect` with a  
partial function achieves the same filtering and transformation in  
a single pass. `map` combined with `getOrElse` keeps the length of  
the vector the same by substituting a default for each absent value.  

## Collecting with a partial function

`collect` applies a partial function and filters in a single pass.  

```scala
@main def main() =
  val values: Vector[Any] = Vector(1, "two", 3, "four", 5)

  val ints    = values.collect { case n: Int    => n }
  val strings = values.collect { case s: String => s.toUpperCase }

  println(ints)
  println(strings)
end main
```

`collect` applies the partial function to every element and includes  
only those for which the function is defined. It combines type  
checking, pattern matching, and transformation in a single concise  
expression. This is more idiomatic than pairing `filter` with `map`  
when the filter condition and the transformation are tightly coupled.  

## Removing duplicates

`distinct` eliminates duplicate elements while preserving order.  

```scala
@main def main() =
  val nums  = Vector(3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5)
  val words = Vector("apple", "banana", "apple", "cherry")

  println(nums.distinct)
  println(words.distinct)
  println(nums.distinctBy(_ % 3))
end main
```

`distinct` returns a new vector with the first occurrence of each  
element kept and all subsequent duplicates removed. Equality is  
determined by the element's `equals` method. `distinctBy` accepts a  
key extractor and removes elements whose extracted key has already  
been seen, giving fine-grained control over what counts as a  
duplicate.  

## For-expressions with a vector

A `for`-expression desugars into `map`, `flatMap`, and `filter`  
calls on the underlying collection.  

```scala
@main def main() =
  val xs = Vector(1, 2, 3)
  val ys = Vector(10, 20)

  val products = for
    x <- xs
    y <- ys
  yield x * y

  println(products)
end main
```

The `for`-expression with two generators desugars into a `flatMap`  
over `xs` and a `map` over `ys`, producing the Cartesian product of  
the two vectors. The result type is inferred as `Vector[Int]` because  
the first generator is a `Vector`. Using `for`-expressions instead  
of chained `flatMap` and `map` calls improves readability when  
multiple generators are involved.  

## For-expressions with guards

A `for`-expression can include `if` guards to filter combinations.  

```scala
@main def main() =
  val pairs = for
    x <- Vector(1, 2, 3, 4, 5)
    y <- Vector(1, 2, 3, 4, 5)
    if x < y
    if (x + y) % 2 == 0
  yield (x, y)

  pairs.foreach(println)
end main
```

Guards inside a `for`-expression desugar into `withFilter` calls,  
which filter elements before passing them to the next stage. Multiple  
guards are chained and all must hold for the combination to be  
included in the result. The first guard `x < y` ensures each pair  
appears only once, and the second retains only pairs whose sum is  
even.  

## Lazy views

A view defers element processing until the results are consumed.  

```scala
@main def main() =
  val v = Vector(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

  val result = v.view
    .filter(_ % 2 == 0)
    .map(_ * 3)
    .take(3)
    .toVector

  println(result)
end main
```

Calling `.view` on a vector returns a lazy `VectorView`. Subsequent  
`filter`, `map`, and `take` operations build a chain of  
transformations that are not applied until a terminal operation such  
as `toVector` or `foreach` forces evaluation. This avoids creating  
intermediate collections for each step and can improve performance  
when only a small prefix of the result is needed.  

## Ordering with a given instance

A custom `Ordering` provided as a `given` makes `sorted` work on  
user-defined types.  

```scala
@main def main() =
  case class Product(name: String, price: Double)

  given Ordering[Product] = Ordering.by(_.price)

  val products = Vector(
    Product("chair",  49.99),
    Product("table", 199.00),
    Product("lamp",   24.50)
  )

  val sorted = products.sorted
  sorted.foreach(p => println(s"${p.name}: ${p.price}"))
end main
```

Defining a `given Ordering[Product]` instance makes `sorted` work  
directly on `Vector[Product]` without passing the ordering  
explicitly. `Ordering.by` builds an ordering from a key-extraction  
function, delegating comparison to the natural ordering of the  
extracted type. This approach integrates cleanly with Scala 3's  
given/using mechanism and keeps the call site clean.  

## Generic vector function

A parametric function can operate on a `Vector` of any element type.  

```scala
@main def main() =
  def rotate[A](v: Vector[A], n: Int): Vector[A] =
    val k = n % v.size
    v.drop(k) ++ v.take(k)

  val nums  = Vector(1, 2, 3, 4, 5)
  val words = Vector("a", "b", "c", "d")

  println(rotate(nums, 2))
  println(rotate(words, 1))
end main
```

The type parameter `[A]` makes `rotate` work for `Vector[Int]`,  
`Vector[String]`, and any other element type without duplicating  
code. The function normalises the rotation count with `% v.size` to  
handle values larger than the vector length, then splits and  
rejoins the vector. Because `Vector` supports efficient `drop`,  
`take`, and `++`, the implementation is concise and runs in O(n).  

## Vectors of tuples

A vector can hold heterogeneous tuples of fixed arity.  

```scala
@main def main() =
  val records: Vector[(String, Int, Boolean)] = Vector(
    ("Alice", 30, true),
    ("Bob",   25, false),
    ("Carol", 35, true)
  )

  val names  = records.map(_._1)
  val active = records.filter(_._3)

  println(names)
  active.foreach((name, age, _) => println(s"$name is $age"))
end main
```

Tuples provide a lightweight way to group related values without  
defining a named type. The individual components are accessed with  
`._1`, `._2`, `._3`. In Scala 3 a tuple pattern can be used directly  
in a lambda parameter list, as shown in the `foreach` call, making  
the code more readable than accessing fields by numeric position.  

## Vectors of case classes

Case classes provide named, typed fields for structured records.  

```scala
@main def main() =
  case class Point(x: Double, y: Double):
    def distanceTo(other: Point): Double =
      math.sqrt(
        math.pow(x - other.x, 2) + math.pow(y - other.y, 2)
      )

  val points = Vector(Point(0, 0), Point(3, 4), Point(1, 1))
  val origin = Point(0, 0)

  points
    .map(p => (p, p.distanceTo(origin)))
    .foreach((p, d) => println(f"$p -> distance ${d%.2f}"))
end main
```

Case classes are the idiomatic way to model structured data in  
Scala. They automatically provide `equals`, `hashCode`, `toString`,  
and `copy`. Defining behaviour directly on the case class, as with  
`distanceTo`, keeps domain logic close to the data. The `map` call  
pairs each point with its computed distance so both are available  
in the subsequent `foreach`.  

## Transforming case class fields

`map` and `copy` update case class fields without mutation.  

```scala
@main def main() =
  case class Employee(name: String, salary: Double)

  val staff = Vector(
    Employee("Alice", 50_000),
    Employee("Bob",   60_000),
    Employee("Carol", 55_000)
  )

  val raised = staff.map(e => e.copy(salary = e.salary * 1.10))
  raised.foreach(println)
end main
```

`copy` creates a new instance of the case class with specified  
fields replaced and all other fields carried over from the original.  
Combining `map` with `copy` is the standard functional pattern for  
updating records in a collection. No existing `Employee` instance is  
modified; `raised` is an entirely new `Vector`.  

## Grouping by field

`groupBy` organises records by the value of a field.  

```scala
@main def main() =
  case class Order(id: Int, status: String, amount: Double)

  val orders = Vector(
    Order(1, "open",    120.0),
    Order(2, "closed",   85.5),
    Order(3, "open",    200.0),
    Order(4, "closed",   45.0),
    Order(5, "pending",  60.0)
  )

  val byStatus = orders.groupBy(_.status)
  byStatus.foreach((status, group) =>
    println(s"$status: ${group.map(_.id)}")
  )
end main
```

`groupBy(_.status)` produces a `Map[String, Vector[Order]]` where  
each key is a distinct status value and the associated value is the  
vector of all orders with that status. The order of elements within  
each group preserves the original order in the input vector. The  
pattern is common in reporting and aggregation pipelines where  
records must be bucketed by a categorical field.  

## Converting to List and back

Vectors and lists can be converted into each other with a single  
method call.  

```scala
@main def main() =
  val v  = Vector(1, 2, 3, 4, 5)
  val ls = v.toList
  val v2 = ls.toVector

  println(v)
  println(ls)
  println(v2)
  println(v == v2)
end main
```

`toList` traverses the vector and builds an equivalent singly-linked  
list in O(n) time. `toVector` on a `List` performs the reverse  
conversion. Both collections implement the `Seq` trait, so code  
written against `Seq` works with either. Converting between them is  
O(n) and produces a completely independent data structure with no  
shared nodes.  

## Converting to Array

A vector can be converted to a mutable JVM array for Java interop.  

```scala
@main def main() =
  val v   = Vector("alpha", "beta", "gamma")
  val arr = v.toArray

  arr(1) = "BETA"

  println(arr.mkString(", "))
  println(v)
end main
```

`toArray` copies all elements into a new JVM array of the inferred  
element type. Mutating the array does not affect the original vector,  
demonstrating that the two data structures are completely  
independent. This conversion is necessary when calling Java APIs that  
require plain arrays rather than Scala collections.  

## Converting to Seq

A vector can be widened to the abstract `Seq` type.  

```scala
@main def main() =
  def printAll(seq: Seq[Int]): Unit =
    seq.foreach(n => print(s"$n "))
    println()

  val v  = Vector(1, 2, 3)
  val ls = List(4, 5, 6)

  printAll(v)
  printAll(ls)
  printAll(v ++ ls)
end main
```

Widening to `Seq` allows a function to accept both `Vector` and  
`List` arguments without duplication. The function sees the  
collection through the abstract `Seq` interface, which provides all  
standard higher-order operations. The concrete runtime type is still  
`Vector` or `List`; widening is purely a compile-time view with no  
runtime overhead.  

## Building a vector incrementally

`Vector.newBuilder` assembles a vector efficiently from individual  
elements.  

```scala
@main def main() =
  val builder = Vector.newBuilder[Int]

  for i <- 1 to 10 do
    if i % 2 == 0 then builder += i

  val evens = builder.result()
  println(evens)
end main
```

`Vector.newBuilder` returns a `mutable.Builder[A, Vector[A]]` that  
accumulates elements with `+=` and materialises the result with  
`result()`. Using a builder avoids the O(n) cost of calling `:+` in  
a loop. It is the idiomatic way to build a vector when the elements  
are produced imperatively, such as inside a loop or a conditional  
branch.  

## Tabulating a vector

`Vector.tabulate` creates a vector from an index-to-value function.  

```scala
@main def main() =
  val squares = Vector.tabulate(6)(n => n * n)
  val grid    = Vector.tabulate(3, 3)((r, c) => r * 3 + c)

  println(squares)
  grid.foreach(println)
end main
```

`Vector.tabulate(n)(f)` creates a vector of length `n` where the  
element at index `i` equals `f(i)`. The two-dimensional overload  
`tabulate(rows, cols)(f)` produces a `Vector[Vector[Int]]`  
representing a matrix. This is more expressive than a  
`for`-expression when the values are purely a function of their  
indices and the intent is to initialise a fixed-size structure.  

## Unfolding a vector

`Vector.unfold` generates a vector from a seed and a step function.  

```scala
@main def main() =
  val fibs = Vector.unfold((0, 1)) { (a, b) =>
    if a > 100 then None
    else Some((a, (b, a + b)))
  }

  println(fibs)
end main
```

`Vector.unfold(seed)(f)` repeatedly applies `f` to the current  
state until it returns `None`. Each invocation returns  
`Some((element, nextState))` to emit one element and advance the  
state, or `None` to terminate. This is the functional dual of a  
fold and is well suited for generating sequences defined by a  
recurrence, such as Fibonacci numbers, without an explicit loop or  
mutable state.  

## Vectors with match types

A match type selects a return type based on the type of the input  
at compile time.  

```scala
@main def main() =
  type Elem[C] = C match
    case Vector[t] => t

  def firstElem[C <: Vector[?]](c: C): Elem[C] =
    c.head.asInstanceOf[Elem[C]]

  val ints  = Vector(1, 2, 3)
  val strs  = Vector("a", "b", "c")

  println(firstElem(ints))
  println(firstElem(strs))
end main
```

Match types, introduced in Scala 3, let the return type of a  
function depend on its type parameter at compile time. `Elem[Vector[Int]]`  
reduces to `Int` and `Elem[Vector[String]]` reduces to `String`.  
The `asInstanceOf` cast bridges the gap between the compile-time  
match type and the runtime value; in practice, match types are  
most useful in type-level computations and advanced library design.  

## Context functions with vectors

A context function threads configuration implicitly through a  
pipeline.  

```scala
@main def main() =
  case class Config(prefix: String)

  type Configured[A] = Config ?=> A

  def label(s: String): Configured[String] =
    summon[Config].prefix + s

  def labelAll(v: Vector[String]): Configured[Vector[String]] =
    v.map(label)

  given Config = Config(">> ")

  val items = Vector("alpha", "beta", "gamma")
  println(labelAll(items))
end main
```

A context function type `Config ?=> A` declares that the function  
requires an implicit `Config` in scope rather than an explicit  
parameter. `summon[Config]` retrieves the given instance. The  
`labelAll` function threads the configuration through the `map`  
call without mentioning it explicitly. This is a clean alternative  
to passing configuration through every function in a pipeline.  

## Functional pipeline

Chained higher-order operations form a declarative data pipeline.  

```scala
@main def main() =
  case class Sale(product: String, units: Int, price: Double)

  val sales = Vector(
    Sale("pen",      100, 1.50),
    Sale("notebook",  40, 3.99),
    Sale("ruler",     60, 0.99),
    Sale("pen",       80, 1.50)
  )

  val topRevenue = sales
    .groupBy(_.product)
    .view
    .mapValues(g => g.map(s => s.units * s.price).sum)
    .toVector
    .sortBy(-_._2)
    .take(3)

  topRevenue.foreach((prod, rev) =>
    println(f"$prod%-12s $$rev%.2f")
  )
end main
```

The pipeline groups sales by product, computes total revenue per  
product with `mapValues`, converts back to a vector, sorts  
descending by revenue, and takes the top three. Calling `.view` on  
the map defers `mapValues` until `toVector` forces evaluation,  
avoiding an intermediate map copy. Each step is independently  
testable and the intent is clear from the method names.  

## Domain modeling with vectors

Vectors are well suited for holding ordered collections of domain  
objects inside model types.  

```scala
@main def main() =
  case class Student(name: String, grades: Vector[Int]):
    def average: Double = grades.sum.toDouble / grades.size
    def passed: Boolean = average >= 50.0

  val students = Vector(
    Student("Alice", Vector(80, 90, 70)),
    Student("Bob",   Vector(40, 55, 35)),
    Student("Carol", Vector(95, 88, 92))
  )

  students
    .map(s => f"${s.name}%-6s avg=${s.average%.1f} passed=${s.passed}")
    .foreach(println)
end main
```

Nesting a `Vector` inside a case class is a common pattern for  
one-to-many relationships in domain models. The `Student` case  
class encapsulates its grades and exposes derived properties as  
methods. Because both the outer vector of students and the inner  
vector of grades are immutable, the data is safe to share across  
threads without synchronisation.  

## State machine using vectors

A vector of transitions can drive a simple finite state machine.  

```scala
@main def main() =
  enum State:
    case Idle, Running, Paused, Stopped

  case class Event(from: State, to: State, label: String)

  import State.*

  val transitions = Vector(
    Event(Idle,    Running, "start"),
    Event(Running, Paused,  "pause"),
    Event(Paused,  Running, "resume"),
    Event(Running, Stopped, "stop")
  )

  def next(state: State, label: String): Option[State] =
    transitions
      .find(e => e.from == state && e.label == label)
      .map(_.to)

  var current = Idle
  val inputs  = Vector("start", "pause", "resume", "stop")

  for input <- inputs do
    next(current, input) match
      case Some(s) =>
        println(s"$current --[$input]--> $s")
        current = s
      case None =>
        println(s"no transition from $current on '$input'")
end main
```

The transition table is stored as an immutable `Vector[Event]`.  
The `next` function queries the table with `find`, returning  
`Some(nextState)` when a matching transition exists and `None`  
otherwise. Using a vector for the transition table keeps the  
definition declarative and easy to extend. The `enum` and  
pattern-matching features of Scala 3 make the state labels  
type-safe and self-documenting.  

## Sliding windows

`sliding` produces overlapping sub-sequences of a fixed width.  

```scala
@main def main() =
  val temps = Vector(18.0, 21.5, 19.0, 22.3, 24.1, 20.5)

  val windows  = temps.sliding(3).toVector
  val averages = windows.map(w => w.sum / w.size)

  windows.zip(averages).foreach((w, avg) =>
    println(f"$w -> avg ${avg%.2f}")
  )
end main
```

`sliding(n)` returns an iterator of vectors, each of length `n`,  
advancing one element at a time. The call to `toVector` materialises  
the iterator. Computing a moving average is the canonical use case:  
each window contains the values for one averaging period.  
`sliding(n, step)` accepts a second argument to control how far the  
window advances between steps.  

## Chunking with grouped

`grouped` splits a vector into non-overlapping chunks of a fixed  
size.  

```scala
@main def main() =
  val items = Vector(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

  val chunks = items.grouped(3).toVector
  chunks.foreach(println)

  println("---")

  items.grouped(4).toVector.zipWithIndex.foreach((page, i) =>
    println(s"page ${i + 1}: $page")
  )
end main
```

`grouped(n)` partitions the vector into consecutive chunks of at  
most `n` elements each. The last chunk may be smaller if the total  
length is not a multiple of `n`. The result is an iterator, so  
`toVector` is needed to materialise all chunks. This is useful for  
pagination, batch processing, or any scenario that requires  
processing elements in fixed-size groups.  

## Aggregating records

`groupBy` and `map` implement a simple group-and-sum aggregation.  

```scala
@main def main() =
  case class Transaction(account: String, amount: Double)

  val txns = Vector(
    Transaction("A", 100.0),
    Transaction("B",  50.0),
    Transaction("A",  75.0),
    Transaction("C", 200.0),
    Transaction("B",  30.0)
  )

  val totals = txns
    .groupBy(_.account)
    .map((acc, ts) => acc -> ts.map(_.amount).sum)
    .toVector
    .sortBy(_._1)

  totals.foreach((acc, total) => println(f"$acc: $$total%.2f"))
end main
```

The pipeline groups transactions by account with `groupBy`, sums the  
amounts in each group, converts the resulting map to a vector of  
pairs, and sorts alphabetically by account name. Each step is a pure  
transformation; no mutable state is involved. This pattern  
generalises to any GROUP BY / SUM query that would otherwise require  
a mutable accumulator map.  

## Idiomatic accumulation pattern

`foldLeft` with an immutable accumulator replaces mutable loop  
variables.  

```scala
@main def main() =
  case class Summary(count: Int, total: Double, max: Double)

  val prices = Vector(12.5, 99.9, 34.0, 7.8, 55.0)

  val init = Summary(0, 0.0, Double.MinValue)

  val result = prices.foldLeft(init) { (acc, p) =>
    Summary(
      count = acc.count + 1,
      total = acc.total + p,
      max   = acc.max max p
    )
  }

  println(
    f"count=${result.count}, total=${result.total%.2f}, " +
    f"max=${result.max%.2f}"
  )
end main
```

Instead of declaring `var count`, `var total`, and `var max` and  
mutating them inside a loop, this example folds the vector with an  
immutable `Summary` accumulator. Each step returns a new `Summary`  
rather than modifying the previous one. The result is single-pass,  
free of mutable state, and easy to reason about. This is the  
idiomatic Scala approach to building summary statistics over a  
collection.  
