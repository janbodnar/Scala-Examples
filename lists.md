# Lists  

A `List` in Scala 3 is an immutable, singly-linked sequence of elements.  
It is one of the most fundamental collection types in Scala's standard  
library. Every list is either the empty list `Nil` or a cons cell `::`,  
which holds a head element and a reference to the rest of the list.  

Unlike `Array`, which is mutable and backed by a JVM array, a `List`  
cannot be updated in place. Any transformation produces a new list while  
the original remains unchanged. `Vector` is a tree-structured immutable  
sequence that provides effectively O(1) random access and update, making  
it better suited for workloads that mix reads and writes at arbitrary  
positions. `Seq` is an abstract trait that both `List` and `Vector`  
implement; it represents any ordered sequence without specifying the  
underlying representation.  

**Performance characteristics**  

Prepending an element with `::` is O(1), as is reading `head` and `tail`.  
Appending at the end with `:+` requires traversing the entire list and is  
O(n). Accessing an arbitrary element by index with `apply` is also O(n)  
because there is no direct addressing. These properties make `List` ideal  
for head-first recursive processing and functional pipelines, but a poor  
choice for random-access or append-heavy workloads.  

**Persistent data structures**  

When you prepend an element to a list, the new list shares its tail with  
the original. No data is copied. This structural sharing makes lists  
memory-efficient even in purely functional programs that produce many  
derived collections.  

**Functional programming model**  

`List` is the workhorse of functional Scala. It integrates directly with  
`map`, `filter`, `flatMap`, `fold`, `for`-expressions, and pattern  
matching. Because lists are immutable, they can be freely shared between  
functions and across threads without defensive copying or synchronisation.  

---

## Creating a list  

This example shows the most common ways to construct a `List`.  

```scala
@main def creatingList(): Unit =
  val nums = List(1, 2, 3, 4, 5)
  val words = List("sky", "blue", "forest")
  val mixed = List(1, "two", true)
  val empty: List[Int] = List()

  println(nums)
  println(words)
  println(mixed)
  println(empty)
```  

The `List` companion object's `apply` method accepts a variable number of  
arguments and builds an immutable list from them. The type parameter is  
usually inferred by the compiler, but it must be stated explicitly for the  
empty list because there are no elements to infer from. An empty list has  
the type `List[Nothing]` by default unless a type annotation is provided.  

## Nil and :: constructor  

This example builds a list using the primitive cons operator `::` and `Nil`.  

```scala
@main def nilAndCons(): Unit =
  val nums = 1 :: 2 :: 3 :: Nil
  val words = "hello" :: "world" :: Nil

  println(nums)
  println(words)
```  

Every `List` is ultimately constructed from `::` (pronounced "cons") and  
`Nil`. The expression `1 :: 2 :: 3 :: Nil` is right-associative, so it  
evaluates as `1 :: (2 :: (3 :: Nil))`. Understanding this structure is  
essential for writing recursive algorithms that deconstruct lists through  
pattern matching.  

## Accessing elements  

This example demonstrates head, tail, and indexed access.  

```scala
@main def accessingElements(): Unit =
  val nums = List(10, 20, 30, 40, 50)

  println(nums.head)
  println(nums.tail)
  println(nums(2))
  println(nums.last)
  println(nums.init)
```  

`head` returns the first element and `tail` returns every element except  
the first; both are O(1) operations. Indexed access via `apply` (the  
parenthesis syntax) is O(n) because lists have no backing array. `last`  
and `init` are also O(n). Calling `head` or `tail` on an empty list throws  
a `NoSuchElementException`, so guard with `isEmpty` or use pattern matching  
when the list may be empty.  

## Checking size and emptiness  

This example shows how to test whether a list is empty and how to count  
its elements.  

```scala
@main def checkingSizeAndEmptiness(): Unit =
  val nums = List(1, 2, 3)
  val empty: List[Int] = Nil

  println(nums.isEmpty)
  println(nums.nonEmpty)
  println(nums.size)
  println(nums.length)
  println(empty.isEmpty)
```  

`isEmpty` and `nonEmpty` are O(1) predicates. `size` and `length` are  
aliases that both traverse the full list and therefore run in O(n) time.  
When you only need to know whether a list contains at least one element,  
always prefer `isEmpty` or `nonEmpty` over comparing the size to zero.  

## Pattern matching on a list  

This example uses pattern matching to deconstruct a list into its head and  
tail.  

```scala
@main def patternMatchingList(): Unit =
  def describe(xs: List[Int]): String =
    xs match
      case Nil        => "empty list"
      case x :: Nil   => s"one element: $x"
      case x :: rest  => s"head $x, tail $rest"

  println(describe(Nil))
  println(describe(List(42)))
  println(describe(List(1, 2, 3)))
```  

Pattern matching on a list mirrors its recursive structure. The `Nil`  
pattern matches an empty list. `x :: Nil` matches a singleton list and  
binds the single element to `x`. `x :: rest` matches any non-empty list,  
binding the head to `x` and the remainder to `rest`. The compiler checks  
all cases for exhaustiveness when the input type is sealed.  

## Pattern matching with a guard  

This example combines a list pattern with a guard condition.  

```scala
@main def patternMatchingGuard(): Unit =
  def classify(xs: List[Int]): String =
    xs match
      case Nil              => "empty"
      case x :: _ if x < 0 => s"starts with negative: $x"
      case x :: _           => s"starts with non-negative: $x"

  println(classify(List(-5, 1, 2)))
  println(classify(List(3, 4, 5)))
  println(classify(Nil))
```  

A guard clause (`if` after the pattern) allows additional conditions to be  
checked after the structural pattern has matched. If the guard is false,  
the match engine moves to the next case. Guards do not affect exhaustiveness  
checking, so it is good practice to include a default catch-all case.  

## List concatenation  

This example joins two lists together using `:::` and `++`.  

```scala
@main def listConcatenation(): Unit =
  val a = List(1, 2, 3)
  val b = List(4, 5, 6)

  val c = a ::: b
  val d = a ++ b

  println(c)
  println(d)
```  

Both `:::` and `++` produce a new list containing all elements of the left  
list followed by all elements of the right. The operation is O(n) in the  
length of the left list because the left list must be rebuilt and its last  
node connected to the beginning of the right list. The right list is shared  
unchanged thanks to structural sharing.  

## Prepending elements  

This example prepends a single element and a list to an existing list.  

```scala
@main def prependingElements(): Unit =
  val nums = List(3, 4, 5)

  val a = 1 :: 2 :: nums
  val b = List(0, 1, 2) ::: nums

  println(a)
  println(b)
```  

Prepending with `::` is O(1) because it creates a single new cons cell  
pointing to the existing list. This makes `::` the preferred way to build  
lists incrementally. When you need to prepend an entire list, `:::` walks  
the left list and is O(k) where k is the size of the prefix.  

## Appending elements  

This example appends elements to the end of a list.  

```scala
@main def appendingElements(): Unit =
  val nums = List(1, 2, 3)

  val a = nums :+ 4
  val b = nums :+ 4 :+ 5

  println(a)
  println(b)
```  

The `:+` operator creates a new list with the element added at the end.  
Because the entire list must be traversed before appending, this operation  
is O(n). Repeated appending in a loop is therefore O(n²). When you need  
to build a large list by appending, prefer `List.newBuilder` or  
`scala.collection.mutable.ListBuffer` and convert to an immutable list  
once construction is complete.  

## Mapping over a list  

This example transforms each element of a list with `map`.  

```scala
@main def mappingList(): Unit =
  val nums = List(1, 2, 3, 4, 5)

  val doubled = nums.map(_ * 2)
  val asStrings = nums.map(n => s"num-$n")

  println(doubled)
  println(asStrings)
```  

`map` applies a function to every element and collects the results into a  
new list of the same length. The original list is not modified. The  
resulting type can differ from the input type, as shown when converting  
integers to formatted strings. Using an underscore (`_`) for the parameter  
is idiomatic when the body refers to the parameter exactly once.  

## Filtering a list  

This example keeps only elements that satisfy a predicate.  

```scala
@main def filteringList(): Unit =
  val nums = List(1, 2, 3, 4, 5, 6, 7, 8)

  val evens = nums.filter(_ % 2 == 0)
  val odds = nums.filterNot(_ % 2 == 0)
  val big = nums.filter(_ > 5)

  println(evens)
  println(odds)
  println(big)
```  

`filter` returns a new list containing only the elements for which the  
predicate returns `true`. `filterNot` is the complement and keeps elements  
for which the predicate returns `false`. Both operations traverse the full  
list in O(n) time. When you need to split a list into two groups at once,  
`partition` is more efficient than calling `filter` twice.  

## FlatMapping a list  

This example maps each element to a list and flattens the result.  

```scala
@main def flatMappingList(): Unit =
  val words = List("hello world", "foo bar baz")

  val tokens = words.flatMap(_.split(" ").toList)
  val pairs = List(1, 2, 3).flatMap(n => List(n, n * 10))

  println(tokens)
  println(pairs)
```  

`flatMap` combines `map` and `flatten`. It applies a function that returns  
a collection to every element and then concatenates all the resulting  
collections into a single list. It is the fundamental operation behind  
`for`-expressions with multiple generators. When chained, `flatMap` can  
express complex data transformations concisely and safely.  

## Folding left  

This example accumulates a result by traversing a list from left to right.  

```scala
@main def foldingLeft(): Unit =
  val nums = List(1, 2, 3, 4, 5)

  val sum = nums.foldLeft(0)(_ + _)
  val product = nums.foldLeft(1)(_ * _)
  val reversed = nums.foldLeft(List.empty[Int])((acc, x) => x :: acc)

  println(sum)
  println(product)
  println(reversed)
```  

`foldLeft` takes an initial accumulator value and a combining function.  
It walks the list from left to right, updating the accumulator at each  
step. The final value of the accumulator is returned. Because the  
combining function receives the current accumulator as its first argument,  
any kind of result type can be produced, including another list. The  
example shows that prepending each element to the accumulator is an  
efficient O(n) list reversal.  

## Folding right  

This example accumulates a result by traversing a list from right to left.  

```scala
@main def foldingRight(): Unit =
  val nums = List(1, 2, 3, 4, 5)

  val sum = nums.foldRight(0)(_ + _)
  val cons = nums.foldRight(List.empty[Int])(_ :: _)

  println(sum)
  println(cons)
```  

`foldRight` starts from the last element and works toward the head. Because  
it must recurse to the end of the list before combining, it can cause a  
stack overflow on very large lists. The `cons` example reconstructs the  
original list by prepending each element onto the accumulator, illustrating  
that `foldRight` with `::` and `Nil` is the identity transformation for  
lists.  

## Reducing a list  

This example combines all elements of a list without an initial value.  

```scala
@main def reducingList(): Unit =
  val nums = List(1, 2, 3, 4, 5)

  val sum = nums.reduce(_ + _)
  val max = nums.reduce((a, b) => if a > b then a else b)

  println(sum)
  println(max)
```  

`reduce` is like `foldLeft` but uses the first element as the initial  
accumulator. It requires the combining function to return the same type  
as the elements. Calling `reduce` on an empty list throws an exception;  
use `reduceOption` if the list might be empty, which returns `None` for  
an empty list and `Some(result)` otherwise.  

## Zipping two lists  

This example pairs elements from two lists into a list of tuples.  

```scala
@main def zippingLists(): Unit =
  val names = List("Alice", "Bob", "Carol")
  val scores = List(85, 92, 78)

  val pairs = names.zip(scores)
  val indexed = names.zipWithIndex

  println(pairs)
  println(indexed)
```  

`zip` pairs corresponding elements from two lists into `(A, B)` tuples.  
If one list is longer, the extra elements are discarded. `zipWithIndex`  
is a convenient shorthand that pairs each element with its zero-based  
position. Both methods are O(n) in the length of the shorter list.  

## Unzipping a list  

This example splits a list of pairs back into two separate lists.  

```scala
@main def unzippingList(): Unit =
  val pairs = List(("Alice", 85), ("Bob", 92), ("Carol", 78))

  val (names, scores) = pairs.unzip

  println(names)
  println(scores)
```  

`unzip` is the inverse of `zip`. It transforms a list of pairs into a  
pair of lists, where the first list contains all first components and the  
second list contains all second components. The result is returned as a  
tuple, which can be destructured immediately with pattern matching in the  
`val` declaration. For triples there is `unzip3`.  

## Sliding windows  

This example produces overlapping sub-lists of a fixed size.  

```scala
@main def slidingWindows(): Unit =
  val nums = List(1, 2, 3, 4, 5, 6)

  val windows = nums.sliding(3).toList
  val steps = nums.sliding(3, 2).toList

  windows.foreach(println)
  println("---")
  steps.foreach(println)
```  

`sliding(n)` creates an iterator of windows of length `n`, advancing by  
one position at a time. The optional second argument controls the step  
size; `sliding(3, 2)` advances by two positions between windows. The  
result is an `Iterator`, so `.toList` materialises it. Sliding windows  
are useful for moving-average calculations and sequential pattern  
detection.  

## Grouping into fixed-size chunks  

This example splits a list into consecutive sublists of equal size.  

```scala
@main def groupedChunks(): Unit =
  val nums = List(1, 2, 3, 4, 5, 6, 7)

  val chunks = nums.grouped(3).toList

  chunks.foreach(println)
```  

`grouped(n)` returns an `Iterator` of lists, each containing at most `n`  
elements. The final group may be smaller if the list length is not a  
multiple of `n`. This is different from `sliding`, which produces  
overlapping windows; `grouped` produces non-overlapping partitions. It  
is useful for batching API calls or processing data in fixed-size pages.  

## Grouping elements by a key  

This example groups list elements into a `Map` using a key function.  

```scala
@main def groupByKey(): Unit =
  val words = List("ant", "bear", "apple", "bat", "cat", "avocado")

  val byFirstLetter = words.groupBy(_.head)

  byFirstLetter.toSeq.sortBy(_._1).foreach { (letter, ws) =>
    println(s"$letter -> $ws")
  }
```  

`groupBy` partitions a list into a `Map` whose keys are produced by the  
given function and whose values are sub-lists of all elements that share  
that key. The order within each group matches the order in the original  
list. The result is an unsorted `Map`, so sorting on the keys before  
printing is needed to get deterministic output.  

## Partitioning a list  

This example splits a list into two based on a predicate.  

```scala
@main def partitioningList(): Unit =
  val nums = List(1, 2, 3, 4, 5, 6, 7, 8)

  val (evens, odds) = nums.partition(_ % 2 == 0)

  println(evens)
  println(odds)
```  

`partition` traverses the list once and returns a pair of lists: the  
first contains all elements for which the predicate holds, and the second  
contains the rest. This is more efficient than calling `filter` and  
`filterNot` separately because the list is only traversed a single time.  
Destructuring the result tuple in the `val` declaration is idiomatic.  

## Sorting with the natural order  

This example sorts a list using the default `Ordering`.  

```scala
@main def sortingNatural(): Unit =
  val nums = List(5, 2, 8, 1, 9, 3)
  val words = List("banana", "apple", "cherry", "date")

  println(nums.sorted)
  println(words.sorted)
  println(nums.sorted(using Ordering[Int].reverse))
```  

`sorted` requires an implicit `Ordering` for the element type. Scala  
provides built-in orderings for all primitive types and `String`. The  
`reverse` ordering is obtained by calling `.reverse` on an existing  
`Ordering` instance. The method returns a new list and does not modify  
the original.  

## Sorting with a custom comparator  

This example sorts a list of records by a specific field.  

```scala
@main def sortingCustom(): Unit =
  val words = List("banana", "kiwi", "apple", "fig", "cherry")

  val byLength = words.sortBy(_.length)
  val byLengthDesc = words.sortBy(_.length)(using Ordering[Int].reverse)
  val custom = words.sortWith((a, b) => a.length < b.length)

  println(byLength)
  println(byLengthDesc)
  println(custom)
```  

`sortBy` extracts a sort key from each element and uses the implicit  
`Ordering` for that key's type. `sortWith` accepts a raw comparison  
function that returns `true` when its first argument should come before  
the second. Both methods produce stable sorts: elements with equal keys  
retain their original relative order.  

## Working with Options inside a list  

This example processes a list that contains `Option` values.  

```scala
@main def optionsInList(): Unit =
  val maybes: List[Option[Int]] = List(Some(1), None, Some(3), None, Some(5))

  val defined = maybes.flatten
  val doubled = maybes.collect { case Some(n) => n * 2 }

  println(defined)
  println(doubled)
```  

`flatten` on a `List[Option[A]]` discards all `None` values and unwraps  
the `Some` values into a `List[A]`. `collect` achieves the same filtering  
and transformation in one step by applying a partial function that only  
handles the `Some` case. Using `collect` is preferred when you also want  
to transform the unwrapped values at the same time.  

## Removing duplicates  

This example eliminates repeated elements from a list.  

```scala
@main def removingDuplicates(): Unit =
  val nums = List(1, 2, 3, 2, 1, 4, 3, 5)
  val words = List("apple", "banana", "Apple", "cherry", "banana")

  val unique = nums.distinct
  val uniqueIgnoreCase = words.distinctBy(_.toLowerCase)

  println(unique)
  println(uniqueIgnoreCase)
```  

`distinct` removes duplicate elements while preserving the order of first  
occurrence. It uses equality (`==`) to detect duplicates. `distinctBy`  
accepts a key function and removes elements whose key has already appeared,  
which is useful for case-insensitive deduplication or deduplication by a  
specific field of a case class.  

## List comprehension  

This example builds a new list using a `for`-expression with a guard.  

```scala
@main def listComprehension(): Unit =
  val nums = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

  val result = for
    n <- nums
    if n % 2 == 0
  yield n * n

  println(result)
```  

A `for`-expression with `yield` is syntactic sugar for a chain of `map`,  
`filter`, and `flatMap` calls. Adding an `if` guard inside the `for` block  
filters elements before the `yield` clause transforms them. The result  
type matches the collection being iterated; iterating over a `List` yields  
a `List`.  

## Nested for-expression  

This example uses two generators to compute a Cartesian product.  

```scala
@main def nestedForExpression(): Unit =
  val xs = List(1, 2, 3)
  val ys = List("a", "b")

  val pairs = for
    x <- xs
    y <- ys
  yield (x, y)

  println(pairs)
```  

Multiple generators in a `for`-expression desugar to nested `flatMap`  
calls. Every combination of `x` from `xs` and `y` from `ys` appears in  
the result, producing the Cartesian product. The outer generator is the  
outer loop; the result length equals `xs.size * ys.size`.  

## Lazy operations with views  

This example applies transformations lazily using a list view.  

```scala
@main def lazyView(): Unit =
  val nums = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

  val result = nums.view
    .filter(_ % 2 == 0)
    .map(_ * 3)
    .take(3)
    .toList

  println(result)
```  

`view` wraps the list in a lazy sequence. Operations on a view are not  
executed immediately; they are fused and applied element-by-element when  
the result is forced with `toList`. This avoids creating intermediate  
collections for each transformation step. Views are especially useful  
when only a small prefix of the transformed sequence is needed, because  
processing stops as soon as enough elements have been produced.  

## Take and drop  

This example extracts or removes prefixes and suffixes of a list.  

```scala
@main def takeAndDrop(): Unit =
  val nums = List(1, 2, 3, 4, 5, 6, 7)

  println(nums.take(3))
  println(nums.drop(3))
  println(nums.takeRight(2))
  println(nums.dropRight(2))
```  

`take(n)` returns the first `n` elements; `drop(n)` discards them and  
returns the rest. `takeRight` and `dropRight` operate on the end of the  
list. All four methods are safe on lists shorter than `n`; they return  
the entire list or an empty list as appropriate rather than throwing an  
exception.  

## TakeWhile and dropWhile  

This example takes or drops elements based on a predicate.  

```scala
@main def takeWhileDropWhile(): Unit =
  val nums = List(1, 2, 3, 4, 5, 4, 3, 2, 1)

  val prefix = nums.takeWhile(_ < 4)
  val suffix = nums.dropWhile(_ < 4)

  println(prefix)
  println(suffix)
```  

`takeWhile` collects elements from the front of the list as long as the  
predicate holds and stops at the first element that fails it. `dropWhile`  
discards elements from the front while the predicate holds and returns  
the remaining list starting at the first failing element. Neither method  
examines the entire list if the predicate becomes false early.  

## Span  

This example splits a list into a prefix and a suffix at the first failing  
element.  

```scala
@main def spanList(): Unit =
  val nums = List(1, 2, 3, 4, 5, 4, 3)

  val (prefix, suffix) = nums.span(_ < 4)

  println(prefix)
  println(suffix)
```  

`span` is equivalent to `(takeWhile(p), dropWhile(p))` but traverses the  
list only once, making it more efficient than calling those two methods  
separately. The first element of the suffix, if it exists, is always the  
first element that did not satisfy the predicate.  

## Exists and forall  

This example tests whether any or all elements satisfy a predicate.  

```scala
@main def existsForall(): Unit =
  val nums = List(1, 2, 3, 4, 5)

  println(nums.exists(_ > 4))
  println(nums.exists(_ > 10))
  println(nums.forall(_ > 0))
  println(nums.forall(_ > 3))
```  

`exists` returns `true` if at least one element satisfies the predicate  
and short-circuits as soon as such an element is found. `forall` returns  
`true` if every element satisfies the predicate and short-circuits on the  
first failure. Both return meaningful results for an empty list: `exists`  
returns `false` and `forall` returns `true` (vacuous truth).  

## Find and indexOf  

This example searches for an element in a list.  

```scala
@main def findAndIndexOf(): Unit =
  val nums = List(10, 20, 30, 40, 50)

  println(nums.find(_ > 25))
  println(nums.find(_ > 100))
  println(nums.indexOf(30))
  println(nums.indexWhere(_ > 25))
```  

`find` returns the first element wrapped in `Some`, or `None` if no  
element satisfies the predicate. It is safe to use on an empty list.  
`indexOf` returns the zero-based index of the first occurrence, or `-1`  
if the element is absent. `indexWhere` behaves like `find` but returns  
the index rather than the value.  

## Collecting with a partial function  

This example combines filtering and mapping using `collect`.  

```scala
@main def collectPartial(): Unit =
  val values: List[Any] = List(1, "two", 3, "four", 5)

  val ints = values.collect { case n: Int => n * 10 }
  val strings = values.collect { case s: String => s.toUpperCase }

  println(ints)
  println(strings)
```  

`collect` applies a partial function to the list. Elements for which the  
partial function is defined are transformed; all others are silently  
dropped. This makes `collect` ideal for type-safe downcast-and-transform  
patterns. Internally it checks `isDefinedAt` before applying, so no  
`MatchError` is thrown.  

## ScanLeft  

This example produces running totals using `scanLeft`.  

```scala
@main def scanLeftExample(): Unit =
  val nums = List(1, 2, 3, 4, 5)

  val running = nums.scanLeft(0)(_ + _)
  val products = nums.scanLeft(1)(_ * _)

  println(running)
  println(products)
```  

`scanLeft` is like `foldLeft` but returns a list of all intermediate  
accumulator values, starting with the initial value. The result list  
always has one more element than the input. `scanRight` is the right-to-  
left variant. Running totals, prefix sums, and cumulative statistics are  
classic use cases for scan operations.  

## Transposing a list of lists  

This example swaps rows and columns of a rectangular matrix.  

```scala
@main def transposeLists(): Unit =
  val matrix = List(
    List(1, 2, 3),
    List(4, 5, 6),
    List(7, 8, 9)
  )

  val transposed = matrix.transpose

  transposed.foreach(row => println(row.mkString(" ")))
```  

`transpose` converts a list of rows into a list of columns. All inner  
lists must have the same length; otherwise an exception is thrown. The  
operation is O(n × m) where n is the number of rows and m is the number  
of columns. It is useful for matrix operations and for regrouping data  
that is organised as parallel sequences.  

## Lists of tuples  

This example stores and processes records as tuples inside a list.  

```scala
@main def listsOfTuples(): Unit =
  val points = List((1, 2), (3, 4), (5, 6))
  val people = List(("Alice", 30), ("Bob", 25), ("Carol", 35))

  val distances = points.map { (x, y) => math.sqrt(x * x + y * y) }
  val names = people.map(_._1)
  val sorted = people.sortBy(_._2)

  println(distances)
  println(names)
  println(sorted)
```  

Tuples are anonymous product types. When a `List[(A, B)]` is passed to a  
function expecting a single argument, the tuple must be matched with a  
destructuring lambda `(a, b) => ...`. Accessing fields via `._1`, `._2`  
is concise but less readable than named fields; case classes are preferred  
when the meaning of each field matters.  

## Lists of case classes  

This example models domain objects as case classes and stores them in a  
list.  

```scala
@main def listsOfCaseClasses(): Unit =
  case class Person(name: String, age: Int)

  val people = List(
    Person("Alice", 30),
    Person("Bob", 25),
    Person("Carol", 35),
    Person("Dave", 25)
  )

  val names = people.map(_.name)
  val adults = people.filter(_.age >= 30)
  val byAge = people.sortBy(_.age)
  val grouped = people.groupBy(_.age)

  println(names)
  println(adults)
  println(byAge)
  grouped.foreach((age, ps) => println(s"$age -> ${ps.map(_.name)}"))
```  

Case classes are the idiomatic way to model structured data in Scala.  
They come with auto-generated `equals`, `hashCode`, `copy`, and `toString`  
methods. Accessing named fields (`.name`, `.age`) is far more readable  
than tuple projection. The `groupBy` result demonstrates how a list of  
objects can be indexed by any field.  

## Generic list functions  

This example defines functions that operate on lists of any element type.  

```scala
def second[A](xs: List[A]): Option[A] =
  xs match
    case _ :: x :: _ => Some(x)
    case _            => None

def pairs[A, B](as: List[A], bs: List[B]): List[(A, B)] =
  as.zip(bs)

@main def genericListFunctions(): Unit =
  println(second(List(10, 20, 30)))
  println(second(List(1)))
  println(second(List("a", "b", "c")))
  println(pairs(List(1, 2, 3), List("one", "two", "three")))
```  

Type parameters make list functions reusable across element types. The  
`second` function returns the second element safely by wrapping it in  
`Option`. Pattern matching is used instead of index access to avoid  
exceptions on short lists. Generic functions compose naturally with  
Scala's type inference, so callers rarely need to specify type arguments  
explicitly.  

## Recursive list processing  

This example implements classic recursive algorithms on lists.  

```scala
def mySum(xs: List[Int]): Int =
  xs match
    case Nil       => 0
    case h :: tail => h + mySum(tail)

def myReverse[A](xs: List[A]): List[A] =
  def go(remaining: List[A], acc: List[A]): List[A] =
    remaining match
      case Nil       => acc
      case h :: tail => go(tail, h :: acc)
  go(xs, Nil)

@main def recursiveListProcessing(): Unit =
  println(mySum(List(1, 2, 3, 4, 5)))
  println(myReverse(List(1, 2, 3, 4, 5)))
```  

Structural recursion on lists follows the shape of the data: handle the  
base case `Nil`, then handle `head :: tail` by processing the head and  
recurring on the tail. The `myReverse` function uses an accumulator  
parameter to avoid rebuilding the list after the recursive call returns,  
making the tail-recursive version O(n) in stack space rather than O(n²).  

## Building lists with :: and Nil  

This example constructs lists programmatically using only `::` and `Nil`.  

```scala
def range(from: Int, to: Int): List[Int] =
  if from > to then Nil
  else from :: range(from + 1, to)

def repeatVal[A](value: A, n: Int): List[A] =
  if n <= 0 then Nil
  else value :: repeatVal(value, n - 1)

@main def buildWithConsNil(): Unit =
  println(range(1, 5))
  println(repeatVal("x", 4))
```  

Constructing lists with `::` and `Nil` is the most direct way to  
understand how Scala lists work at a structural level. Each recursive  
call contributes one cons cell; the recursion terminates by returning  
`Nil`. These implementations mirror the standard library's `List.range`  
and `List.fill`, but expressed in terms of the primitive constructors.  

## List.fill and List.tabulate  

This example generates lists using factory methods.  

```scala
@main def fillAndTabulate(): Unit =
  val fives = List.fill(5)(0)
  val repeated = List.fill(3)("hello")
  val squares = List.tabulate(6)(n => n * n)
  val grid = List.tabulate(3, 3)((r, c) => r * 3 + c)

  println(fives)
  println(repeated)
  println(squares)
  println(grid)
```  

`List.fill(n)(elem)` creates a list of `n` copies of `elem`. `elem` is  
a by-name parameter, so it is evaluated once for each element; this  
matters when `elem` has side effects. `List.tabulate(n)(f)` creates a  
list of `n` elements where element `i` is `f(i)`. The two-argument  
overload of `tabulate` creates a list of lists, useful for initialising  
matrices.  

## Converting between List, Seq, Vector, and Array  

This example converts a list to and from other collection types.  

```scala
@main def convertingCollections(): Unit =
  val list = List(1, 2, 3, 4, 5)

  val seq: Seq[Int] = list.toSeq
  val vector: Vector[Int] = list.toVector
  val array: Array[Int] = list.toArray
  val set = list.toSet

  println(seq)
  println(vector)
  println(array.mkString(", "))
  println(set)

  val backToList = array.toList
  println(backToList)
```  

All Scala collections expose `toList`, `toSeq`, `toVector`, `toArray`,  
and `toSet`. Converting to `Seq` does not copy the data when the  
underlying collection already satisfies the `Seq` interface; `List`  
extends `Seq`, so `toSeq` is essentially free. Converting to `Array`  
always copies the data to a JVM array. Converting back with `toList`  
allocates a new immutable list.  

## Performance considerations  

This example illustrates efficient list construction with `ListBuffer`.  

```scala
import scala.collection.mutable.ListBuffer

@main def performanceConsiderations(): Unit =
  val buf = ListBuffer[Int]()

  for i <- 1 to 10 do
    buf += i

  val list = buf.toList

  println(list)

  val prepended = List.newBuilder[Int]
  for i <- (1 to 5).reverse do
    prepended += i
  println(prepended.result())
```  

When you need to build a list by appending many elements, `ListBuffer`  
provides O(1) amortised append via a mutable backing structure. Calling  
`toList` at the end converts it to an immutable list in O(1) without  
copying. `List.newBuilder` is the builder-pattern equivalent; it is what  
methods like `map` and `filter` use internally to assemble the result.  
Avoid repeated `:+` in a loop, which produces O(n²) total work.  

## Using a given Ordering with lists  

This example provides a custom implicit `Ordering` for a case class.  

```scala
@main def givenOrdering(): Unit =
  case class Student(name: String, grade: Int)

  given Ordering[Student] = Ordering.by(s => (s.grade, s.name))

  val students = List(
    Student("Alice", 90),
    Student("Bob", 85),
    Student("Carol", 90),
    Student("Dave", 85)
  )

  println(students.sorted)
  println(students.min)
  println(students.max)
```  

A `given` instance of `Ordering[A]` is automatically picked up by  
`sorted`, `min`, `max`, and `sortBy` without explicit arguments.  
`Ordering.by` creates an ordering by projecting each element to a  
comparable key. A tuple key `(grade, name)` sorts first by grade and  
then by name when grades are equal, providing a stable, multi-field  
ordering.  

## Lists with context functions  

This example uses a context function to thread configuration through list  
transformations.  

```scala
@main def listsContextFunctions(): Unit =
  case class Config(prefix: String)

  type Configured[A] = Config ?=> A

  def tag(s: String): Configured[String] = summon[Config].prefix + s

  def tagAll(xs: List[String]): Configured[List[String]] =
    xs.map(tag)

  given Config = Config("[INFO] ")

  val tagged = tagAll(List("startup", "shutdown", "error"))
  println(tagged)
```  

Context functions (`?=>`) are a Scala 3 feature that lets a function  
implicitly pass a context value through a call chain without threading it  
explicitly as a parameter. Here `Configured[A]` is an alias for  
`Config ?=> A`. Every call to `tag` inside `tagAll` automatically  
receives the `Config` from the enclosing scope, keeping the transformation  
logic clean.  

## Lists with match types  

This example uses a match type to determine the element type of a nested  
list.  

```scala
type Flatten[X] = X match
  case List[a] => a
  case _       => X

def flattenOne[A](xs: List[List[A]]): List[A] = xs.flatten

@main def listsMatchTypes(): Unit =
  val nested: List[List[Int]] = List(List(1, 2), List(3, 4), List(5))
  val flat: List[Flatten[List[Int]]] = flattenOne(nested)

  println(flat)

  val words: List[List[String]] = List(List("hello", "world"), List("foo"))
  println(flattenOne(words))
```  

Match types let the compiler compute a type from another type using  
pattern-like syntax. `Flatten[List[Int]]` reduces to `Int` at compile  
time, expressing that flattening a `List[List[Int]]` yields a `List[Int]`.  
Match types are a Scala 3 feature that enables precise type-level  
reasoning without macro-level complexity.  

## Flattening nested lists  

This example demonstrates several ways to flatten a list of lists.  

```scala
@main def flatteningLists(): Unit =
  val nested = List(List(1, 2, 3), List(4, 5), List(6))

  val flat1 = nested.flatten
  val flat2 = nested.flatMap(identity)
  val flat3 = nested.foldLeft(List.empty[Int])(_ ::: _)

  println(flat1)
  println(flat2)
  println(flat3)
```  

`flatten` is the most direct approach and is optimised internally. Passing  
`identity` to `flatMap` produces the same result and makes the `flatMap`/  
`flatten` equivalence visible. The `foldLeft` approach builds the result  
by concatenating each inner list, which is instructive but O(n²) due to  
repeated list concatenation; `flatten` is preferred in production code.  

## Working with mkString  

This example formats list contents as human-readable strings.  

```scala
@main def workingWithMkString(): Unit =
  val nums = List(1, 2, 3, 4, 5)
  val words = List("alpha", "beta", "gamma")

  println(nums.mkString)
  println(nums.mkString(", "))
  println(nums.mkString("[", ", ", "]"))
  println(words.mkString(" | "))
```  

`mkString` has three overloads: with no arguments it concatenates the  
string representations of all elements; with one argument it inserts a  
separator between elements; with three arguments it wraps the result in a  
prefix and suffix. It is far more readable than manual string building  
and avoids a trailing separator that a naïve loop would produce.  

## Lists with Union types  

This example stores heterogeneous data in a list using a Union type.  

```scala
@main def listsUnionTypes(): Unit =
  val data: List[Int | String] = List(1, "two", 3, "four", 5)

  val total = data.foldLeft(0) { (acc, x) =>
    x match
      case n: Int    => acc + n
      case s: String => acc + s.length
  }

  println(total)

  data.foreach {
    case n: Int    => println(s"int: $n")
    case s: String => println(s"str: $s")
  }
```  

Union types (`A | B`) are a Scala 3 feature. A `List[Int | String]` can  
hold both integers and strings without boxing them in a wrapper type like  
`Either` or a sealed trait. Pattern matching on the elements uses  
type-test patterns to branch on the runtime type. Union types reduce  
boilerplate when heterogeneous collections are needed.  

## Collecting results with Either  

This example processes a list and collects successes and failures.  

```scala
@main def collectingEithers(): Unit =
  def parse(s: String): Either[String, Int] =
    s.toIntOption match
      case Some(n) => Right(n)
      case None    => Left(s"not a number: $s")

  val inputs = List("1", "two", "3", "four", "5")
  val results = inputs.map(parse)

  val (errors, values) = results.partitionMap(identity)

  println(errors)
  println(values)
  println(values.sum)
```  

`partitionMap` splits a `List[Either[A, B]]` into a `(List[A], List[B])`  
pair in a single traversal. The `Left` values end up in the first list  
and the `Right` values in the second. This pattern is useful for  
collecting all errors from a bulk parsing or validation step rather than  
failing on the first error.  

## Real-world pattern: word frequency count  

This example counts word frequencies in a block of text.  

```scala
@main def wordFrequency(): Unit =
  val text =
    "the quick brown fox jumps over the lazy dog " +
    "the fox and the dog sat under the tree"

  val frequencies = text
    .split(" ")
    .toList
    .groupBy(identity)
    .view
    .mapValues(_.size)
    .toMap
    .toSeq
    .sortBy(-_._2)

  frequencies.foreach { (word, count) =>
    println(f"$word%-10s $count")
  }
```  

The pipeline splits the text into tokens, groups identical words, maps  
each group to its size, sorts by count descending, and prints the result.  
Using `.view.mapValues` avoids creating an intermediate map. The `f`  
interpolator with `%-10s` left-aligns the word in a field of ten  
characters for neat tabular output.  

## Advanced functional transformations  

This example chains multiple higher-order operations on a list of records.  

```scala
@main def advancedTransformations(): Unit =
  case class Order(id: Int, product: String, amount: Double)

  val orders = List(
    Order(1, "apple",  1.50),
    Order(2, "banana", 0.75),
    Order(3, "apple",  2.00),
    Order(4, "cherry", 3.25),
    Order(5, "banana", 1.10)
  )

  val report = orders
    .groupBy(_.product)
    .view
    .mapValues(os => os.map(_.amount).sum)
    .toMap
    .toSeq
    .sortBy(_._1)

  report.foreach { (product, total) =>
    println(f"$product%-8s $$${total%.2f}")
  }
```  

The pipeline groups orders by product name, sums the amounts within each  
group using `map` and `sum`, converts back to a sortable sequence, and  
sorts alphabetically. Each step is a pure transformation that produces a  
new collection. The `view` prevents the intermediate `Map` from being  
materialised before `mapValues` has been applied.  

## Idiomatic list usage in Scala 3  

This example demonstrates idiomatic Scala 3 style when working with lists.  

```scala
@main def idiomaticListUsage(): Unit =
  case class User(name: String, active: Boolean, score: Int)

  extension (users: List[User])
    def activeUsers: List[User] = users.filter(_.active)
    def topN(n: Int): List[User] = users.sortBy(-_.score).take(n)
    def names: List[String] = users.map(_.name)

  val users = List(
    User("Alice",  true,  95),
    User("Bob",    false, 80),
    User("Carol",  true,  88),
    User("Dave",   true,  72),
    User("Eve",    false, 91)
  )

  val top2 = users.activeUsers.topN(2).names

  println(top2)
```  

Extension methods in Scala 3 let you add domain-specific operations to  
`List[User]` without subclassing. The pipeline `activeUsers.topN(2).names`  
reads naturally as a sentence while remaining purely functional. Each  
method returns a new list, composing seamlessly. This pattern keeps  
business logic close to the data type it operates on without polluting  
the `User` or `List` definitions themselves.  
