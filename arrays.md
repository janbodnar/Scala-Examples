# Arrays  

An array in Scala 3 is a fixed-size, mutable, indexed collection of elements  
of the same type. Under the hood, a Scala `Array[T]` maps directly to a JVM  
primitive array (`T[]` in Java), which makes it the most performant collection  
type available on the JVM. Because of this direct mapping, arrays offer  
constant-time element access and constant-time in-place updates.  

Compared to `List`, arrays are backed by a contiguous block of memory and  
allow random access in O(1) time. A `List` is a linked structure where  
accessing the nth element requires traversing n nodes. `Vector` provides  
effectively constant-time random access through a tree structure, but carries  
higher overhead than a raw array for simple indexed reads and writes. `Seq`  
is a general abstraction that may be backed by any sequential structure, so  
it offers no performance guarantees of its own.  

Arrays are mutable: any element can be reassigned after creation. However,  
the size of an array is fixed at creation time. If you need a collection  
that grows dynamically, use `scala.collection.mutable.ArrayBuffer` instead.  
`ArrayBuffer` wraps a resizable array and exposes append, prepend, and  
remove operations.  

Because `Array[T]` compiles directly to Java's `T[]`, Scala arrays integrate  
seamlessly with Java APIs. A Scala array can be passed directly to any Java  
method that expects an array, and Java arrays returned from Java methods can  
be used as Scala arrays without any explicit conversion.  

Key performance characteristics:  

- Element access: O(1)  
- Element update: O(1)  
- Linear search: O(n)  
- Binary search on a sorted array: O(log n)  
- Appending an element: not supported; use `ArrayBuffer` for growth  

Arrays are the right choice when the collection size is known in advance,  
when maximum performance for indexed access or updates is required, when  
Java interoperability is needed, or when working with primitive numeric  
data in performance-critical code. For most general-purpose Scala code,  
prefer immutable collections such as `List`, `Vector`, or `IArray`.  

## Creating an array  

This example shows how to create arrays with predefined values.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4)
  val words = Array("sky", "blue", "forest")

  println(nums.mkString(", "))
  println(words.mkString(", "))
end main
```  

The example creates two arrays: one of integers and one of strings. The  
`mkString` method joins all elements into a readable string separated by  
the given delimiter. Arrays have a fixed size, so elements cannot be added  
or removed after creation.  

## Accessing elements  

This example demonstrates how to read values from an array using indices.  

```scala
@main def main() =
  val nums = Array(10, 20, 30, 40)

  println(nums(0))
  println(nums(3))
  println(nums.head)
  println(nums.last)
end main
```  

Array indices start at zero. Accessing an element uses parentheses, which  
calls the `apply` method under the hood. The `head` and `last` convenience  
methods return the first and last element respectively. Accessing an index  
outside the valid range throws an `ArrayIndexOutOfBoundsException`.  

## Updating elements  

This example shows how to modify existing values in an array.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4)

  nums(0) = 11
  nums(1) = 22

  println(nums.mkString(", "))
end main
```  

Arrays in Scala are mutable, so elements can be reassigned at any index.  
The syntax `nums(i) = value` desugars to a call to the `update` method,  
which is the standard way to perform in-place element replacement. The  
declaration keyword `val` refers to the array reference, not its contents,  
so the contents can still be modified freely.  

## Iterating with for  

This example shows how to iterate through array elements using a `for` loop.  

```scala
@main def main() =
  val fruits = Array("apple", "banana", "cherry")

  for fruit <- fruits do
    println(fruit)
end main
```  

A `for` loop without `yield` is used purely for its side effects, such as  
printing. The loop variable `fruit` takes each element in turn. This syntax  
is idiomatic Scala 3 and reads almost like plain English.  

## Iterating with foreach  

This example demonstrates the `foreach` higher-order method on arrays.  

```scala
@main def main() =
  val nums = Array(3, 6, 9, 12)

  nums.foreach(n => println(n))
  nums.foreach(println)
end main
```  

The `foreach` method applies a function to each element and discards the  
result. When the function is a single-argument method like `println`, the  
argument can be omitted using eta-expansion, making the call even more  
concise. Both forms produce identical output.  

## Iterating over indices  

This example shows how to iterate using both index and value together.  

```scala
@main def main() =
  val colors = Array("red", "green", "blue")

  for i <- colors.indices do
    println(s"$i: ${colors(i)}")
end main
```  

The `indices` method returns a `Range` from zero to the last valid index.  
Iterating over it gives access to both the position and the value, which  
is useful when the index itself carries meaning, such as when building  
formatted output or accessing a parallel data structure.  

## Array comprehension  

This example demonstrates how to build a new array using a `for` expression.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4, 5, 6)

  val evens = for
    n <- nums
    if n % 2 == 0
  yield n * n

  println(evens.mkString(", "))
end main
```  

A `for` expression with `yield` produces a new collection. Because the  
source is an `Array`, the result is also an `Array`. The guard `if n % 2 == 0`  
filters elements before they are transformed by `n * n`. The original array  
is not modified.  

## Mapping arrays  

This example shows how to apply a transformation to every element of an array.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4, 5)

  val doubled = nums.map(_ * 2)
  val asStrings = nums.map(n => s"item-$n")

  println(doubled.mkString(", "))
  println(asStrings.mkString(", "))
end main
```  

The `map` method applies a function to each element and collects the results  
into a new array of the same length. The original array is left unchanged.  
`map` is one of the most frequently used higher-order functions in Scala and  
works identically across all standard collection types.  

## Filtering arrays  

This example demonstrates how to select elements that satisfy a predicate.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4, 5, 6, 7, 8)

  val evens = nums.filter(_ % 2 == 0)
  val odds = nums.filterNot(_ % 2 == 0)

  println(evens.mkString(", "))
  println(odds.mkString(", "))
end main
```  

The `filter` method returns a new array containing only the elements for  
which the predicate returns `true`. The complementary method `filterNot`  
keeps elements where the predicate returns `false`. Both methods leave the  
original array intact.  

## Flat-mapping arrays  

This example shows how to map and flatten in a single step with `flatMap`.  

```scala
@main def main() =
  val words = Array("hello world", "foo bar baz")

  val allWords = words.flatMap(_.split(" "))

  println(allWords.mkString(", "))
end main
```  

The `flatMap` method applies a function that returns an array to each element  
and then concatenates all the resulting arrays into one. It is equivalent to  
calling `map` followed by `flatten`. Here each string is split into words,  
and all words are collected into a single flat array.  

## Folding arrays  

This example demonstrates how to reduce an array to a single value using `fold`.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4, 5)

  val sum = nums.foldLeft(0)(_ + _)
  val product = nums.foldLeft(1)(_ * _)

  println(s"Sum: $sum")
  println(s"Product: $product")
end main
```  

The `foldLeft` method starts with an initial accumulator value and applies a  
binary function to the accumulator and each element from left to right. The  
final accumulator value is returned. This pattern generalises summation,  
multiplication, and many other aggregations into a single abstraction.  

## Reducing arrays  

This example shows how to aggregate array elements without an initial value.  

```scala
@main def main() =
  val nums = Array(3, 7, 2, 9, 4)

  val max = nums.reduce(_ max _)
  val sum = nums.reduce(_ + _)

  println(s"Max: $max")
  println(s"Sum: $sum")
end main
```  

The `reduce` method combines adjacent elements using a binary function,  
starting from the left. Unlike `fold`, it requires no initial value but  
throws an exception on an empty array. For safe aggregation on potentially  
empty arrays, prefer `reduceOption`, which returns an `Option`.  

## Sorting arrays  

This example demonstrates natural-order sorting of an array.  

```scala
@main def main() =
  val nums = Array(5, 3, 8, 1, 9, 2)
  val words = Array("banana", "apple", "cherry", "date")

  val sortedNums = nums.sorted
  val sortedWords = words.sorted

  println(sortedNums.mkString(", "))
  println(sortedWords.mkString(", "))
end main
```  

The `sorted` method returns a new array with elements arranged in ascending  
natural order, as defined by the implicit `Ordering` in scope. For integers  
this means numerical order; for strings it means lexicographical order. The  
original array is not modified.  

## Sorting with custom ordering  

This example shows how to sort an array using a custom comparator.  

```scala
@main def main() =
  val words = Array("banana", "fig", "apple", "kiwi", "date")

  val byLength = words.sortBy(_.length)
  val descending = words.sortWith(_.compareTo(_) > 0)

  println(byLength.mkString(", "))
  println(descending.mkString(", "))
end main
```  

The `sortBy` method accepts a key-extraction function and sorts by the  
values it produces. The `sortWith` method accepts a binary predicate that  
returns `true` when the first argument should come before the second. Both  
methods return a new array and leave the original unchanged.  

## Searching with find  

This example shows how to locate the first element matching a predicate.  

```scala
@main def main() =
  val nums = Array(4, 7, 2, 9, 1, 5)

  val firstOver5 = nums.find(_ > 5)

  firstOver5 match
    case Some(n) => println(s"Found: $n")
    case None    => println("Not found")
end main
```  

The `find` method returns an `Option` containing the first element for  
which the predicate holds, or `None` if no such element exists. Using  
`Option` avoids null checks and integrates naturally with pattern matching  
and other functional combinators such as `map`, `flatMap`, and `getOrElse`.  

## Searching with indexOf  

This example demonstrates how to find the position of an element.  

```scala
@main def main() =
  val fruits = Array("apple", "banana", "cherry", "banana")

  val first = fruits.indexOf("banana")
  val last = fruits.lastIndexOf("banana")
  val missing = fruits.indexOf("grape")

  println(s"First banana at: $first")
  println(s"Last banana at:  $last")
  println(s"Grape at: $missing")
end main
```  

The `indexOf` method performs a linear scan and returns the zero-based index  
of the first matching element, or `-1` if the element is absent. The  
`lastIndexOf` method scans from the end. Both methods use equality (`==`)  
for comparison.  

## Checking membership  

This example shows how to test for the presence or absence of elements.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4, 5)

  println(nums.contains(3))
  println(nums.contains(9))
  println(nums.exists(_ > 4))
  println(nums.forall(_ > 0))
  println(nums.forall(_ > 3))
end main
```  

The `contains` method checks whether an element equal to the argument is  
present. The `exists` method tests whether at least one element satisfies  
a predicate, and `forall` checks whether every element does. These methods  
short-circuit: they stop scanning as soon as the outcome is determined.  

## Slicing arrays  

This example demonstrates how to extract a sub-array by index range.  

```scala
@main def main() =
  val nums = Array(0, 1, 2, 3, 4, 5, 6, 7, 8, 9)

  val middle = nums.slice(3, 7)
  val first3 = nums.take(3)
  val last3 = nums.takeRight(3)

  println(middle.mkString(", "))
  println(first3.mkString(", "))
  println(last3.mkString(", "))
end main
```  

The `slice(from, until)` method returns elements at indices `from` up to but  
not including `until`. The `take` and `takeRight` methods extract a given  
number of elements from the start or end respectively. All three methods  
return new arrays without modifying the original.  

## Taking and dropping  

This example shows how to split an array into parts.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4, 5, 6)

  val (small, large) = nums.partition(_ < 4)
  val dropped = nums.drop(2)
  val droppedRight = nums.dropRight(2)

  println(small.mkString(", "))
  println(large.mkString(", "))
  println(dropped.mkString(", "))
  println(droppedRight.mkString(", "))
end main
```  

The `partition` method splits an array into two arrays: one with elements  
satisfying the predicate and one with elements that do not. The `drop` and  
`dropRight` methods discard a number of elements from the start or end of  
the array and return the remainder as a new array.  

## Copying arrays  

This example shows how to duplicate an array.  

```scala
@main def main() =
  val original = Array(1, 2, 3, 4, 5)

  val copy1 = original.clone()
  val copy2 = original.toArray

  copy1(0) = 99

  println(original.mkString(", "))
  println(copy1.mkString(", "))
  println(copy2.mkString(", "))
end main
```  

The `clone` method creates a shallow copy of the array. Modifying the copy  
does not affect the original. For arrays of primitive types or immutable  
objects a shallow copy is sufficient. The `toArray` method on an existing  
array also produces an independent copy.  

## Partial copying  

This example demonstrates low-level element copying with `Array.copy`.  

```scala
@main def main() =
  val src = Array(10, 20, 30, 40, 50)
  val dst = Array.ofDim[Int](5)

  Array.copy(src, 1, dst, 0, 3)

  println(dst.mkString(", "))
end main
```  

`Array.copy(src, srcPos, dst, dstPos, length)` copies `length` elements  
from `src` starting at `srcPos` into `dst` starting at `dstPos`. It maps  
directly to `java.lang.System.arraycopy`, making it the fastest way to  
move blocks of data between arrays. The destination array must already  
exist and be large enough to accommodate the copied data.  

## Concatenating arrays  

This example shows how to join two or more arrays together.  

```scala
@main def main() =
  val a = Array(1, 2, 3)
  val b = Array(4, 5, 6)
  val c = Array(7, 8, 9)

  val ab = a ++ b
  val abc = Array.concat(a, b, c)

  println(ab.mkString(", "))
  println(abc.mkString(", "))
end main
```  

The `++` operator returns a new array containing all elements of the left  
array followed by all elements of the right array. For joining more than  
two arrays at once, `Array.concat` accepts a variable number of arrays and  
is more efficient than chaining multiple `++` calls.  

## Array fill  

This example demonstrates how to create an array with a repeated value.  

```scala
@main def main() =
  val zeros = Array.fill(5)(0)
  val greet = Array.fill(3)("hello")

  println(zeros.mkString(", "))
  println(greet.mkString(", "))
end main
```  

`Array.fill(n)(elem)` creates an array of length `n` where every element is  
the result of evaluating `elem`. The second argument is a by-name parameter,  
so it is re-evaluated for each element. This is useful for initialising  
arrays with mutable objects, where each slot needs its own instance.  

## Array tabulate  

This example shows how to create an array using a function of its index.  

```scala
@main def main() =
  val squares = Array.tabulate(6)(n => n * n)
  val letters = Array.tabulate(5)(i => ('a' + i).toChar)

  println(squares.mkString(", "))
  println(letters.mkString(", "))
end main
```  

`Array.tabulate(n)(f)` creates an array of length `n` where element `i` is  
the value of `f(i)`. This is the preferred way to build an array whose  
contents are derived from the index. It avoids mutable loop variables and  
is more readable than an imperative initialization loop.  

## Arrays with default values  

This example shows how to allocate an array without initial values.  

```scala
@main def main() =
  val ints = Array.ofDim[Int](4)
  val bools = Array.ofDim[Boolean](3)
  val strs = Array.ofDim[String](3)

  println(ints.mkString(", "))
  println(bools.mkString(", "))
  println(strs.mkString(", "))
end main
```  

`Array.ofDim[T](n)` allocates an array of the specified size and fills it  
with the JVM default value for the element type. For numeric types this is  
zero, for `Boolean` it is `false`, and for reference types including  
`String` it is `null`. This mirrors the behaviour of Java array allocation.  

## Multi-dimensional arrays  

This example demonstrates how to create and access a two-dimensional array.  

```scala
@main def main() =
  val matrix = Array.ofDim[Int](3, 3)

  for
    row <- 0 until 3
    col <- 0 until 3
  do
    matrix(row)(col) = row * 3 + col

  for row <- matrix do
    println(row.mkString(" "))
end main
```  

`Array.ofDim[T](rows, cols)` creates an array of arrays, representing a  
two-dimensional grid. Elements are accessed with two consecutive index  
operations: `matrix(row)(col)`. The inner arrays are independent, so rows  
can have different lengths if constructed manually.  

## Arrays of tuples  

This example shows how to store pairs of values in an array.  

```scala
@main def main() =
  val pairs = Array(("Alice", 30), ("Bob", 25), ("Carol", 35))

  for (name, age) <- pairs do
    println(s"$name is $age years old")

  val sorted = pairs.sortBy(_._2)
  sorted.foreach((name, age) => println(s"$name: $age"))
end main
```  

An array of tuples groups related values without defining a full class.  
Destructuring in a `for` loop or `foreach` lambda extracts tuple fields  
directly into named variables. When sorting, `_._2` selects the second  
element as the sort key.  

## Arrays of case classes  

This example demonstrates storing structured data in an array.  

```scala
case class Point(x: Double, y: Double):
  def distanceTo(other: Point): Double =
    val dx = x - other.x
    val dy = y - other.y
    math.sqrt(dx * dx + dy * dy)

@main def main() =
  val points = Array(
    Point(0.0, 0.0),
    Point(3.0, 4.0),
    Point(1.0, 1.0)
  )

  val origin = Point(0.0, 0.0)
  val closest = points.minBy(_.distanceTo(origin))

  println(s"Closest to origin: $closest")
  points.foreach(p => println(f"(${p.x}%.1f, ${p.y}%.1f)"))
end main
```  

Case classes provide named, immutable fields and work naturally inside  
arrays. The `minBy` method scans the array and returns the element that  
produces the smallest value when the key function is applied, making  
it easy to find the nearest point to the origin in a single expression.  

## Converting to List  

This example shows how to convert an array to a `List`.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4, 5)

  val list = nums.toList
  val prepended = 0 :: list

  println(list)
  println(prepended)
end main
```  

The `toList` method creates an immutable `List` from an array. The resulting  
list is a snapshot: later changes to the array do not affect it. Once in  
`List` form, you gain access to list-specific operations such as cons (`::`)  
prepend, which runs in O(1) time.  

## Converting to Set and Map  

This example demonstrates converting an array into a `Set` and a `Map`.  

```scala
@main def main() =
  val words = Array("apple", "banana", "apple", "cherry")
  val pairs = Array(("one", 1), ("two", 2), ("three", 3))

  val set = words.toSet
  val map = pairs.toMap

  println(set)
  println(map)
  println(map("two"))
end main
```  

Converting to a `Set` removes duplicate elements automatically. Converting  
an array of two-element tuples to a `Map` treats the first element as the  
key and the second as the value. Both resulting collections are immutable by  
default.  

## Converting collections to arrays  

This example shows how to obtain an array from various collection types.  

```scala
@main def main() =
  val fromList = List(1, 2, 3).toArray
  val fromVector = Vector(4, 5, 6).toArray
  val fromRange = (1 to 5).toArray
  val fromSet = Set(9, 7, 8).toArray.sorted

  println(fromList.mkString(", "))
  println(fromVector.mkString(", "))
  println(fromRange.mkString(", "))
  println(fromSet.mkString(", "))
end main
```  

Every standard Scala collection provides a `toArray` method that copies its  
elements into a new array. Ranges convert eagerly, materialising all values.  
Sets have no defined order, so the resulting array is sorted explicitly here  
for predictable output.  

## ArrayBuffer  

This example demonstrates `ArrayBuffer` as a growable alternative to arrays.  

```scala
import scala.collection.mutable.ArrayBuffer

@main def main() =
  val buf = ArrayBuffer(1, 2, 3)

  buf += 4
  buf.prepend(0)
  buf.remove(2)

  println(buf.mkString(", "))
  println(buf.toArray.mkString(", "))
end main
```  

`ArrayBuffer` is backed by a resizable array and provides amortised O(1)  
append. It is the right choice when the final number of elements is not  
known at construction time. Once all elements have been collected, calling  
`toArray` produces an ordinary fixed-size array.  

## Zipping arrays  

This example shows how to combine two arrays element-by-element.  

```scala
@main def main() =
  val names = Array("Alice", "Bob", "Carol")
  val scores = Array(85, 92, 78)

  val combined = names.zip(scores)
  combined.foreach((name, score) => println(s"$name: $score"))

  val withIndex = names.zipWithIndex
  withIndex.foreach((name, i) => println(s"$i. $name"))
end main
```  

The `zip` method pairs elements from two arrays by position, producing an  
array of tuples. If one array is longer, excess elements are discarded. The  
`zipWithIndex` method pairs each element with its zero-based position,  
which is more efficient than calling `zip(indices)` separately.  

## Unzipping arrays  

This example demonstrates splitting an array of pairs into two arrays.  

```scala
@main def main() =
  val pairs = Array(("Alice", 30), ("Bob", 25), ("Carol", 35))

  val (names, ages) = pairs.unzip

  println(names.mkString(", "))
  println(ages.mkString(", "))
  println(s"Average age: ${ages.sum.toDouble / ages.length}")
end main
```  

The `unzip` method is the inverse of `zip`. It splits an array of two-element  
tuples into a pair of arrays, where the first contains all first elements and  
the second all second elements. Destructuring the result with `val (a, b) =`  
names both arrays in a single statement.  

## Grouping elements  

This example shows how to group array elements by a computed key.  

```scala
@main def main() =
  val words = Array("ant", "bear", "cat", "deer", "elk", "fox")

  val byLength = words.groupBy(_.length)

  byLength.toSeq.sortBy(_._1).foreach { (len, ws) =>
    println(s"Length $len: ${ws.mkString(", ")}")
  }
end main
```  

The `groupBy` method partitions array elements into a `Map` whose keys are  
the distinct values produced by the key function. Each key maps to an array  
of all elements that produced that key. This is useful for bucketing data  
without sorting it first.  

## Partitioning arrays  

This example demonstrates splitting an array into two parts.  

```scala
@main def main() =
  val nums = Array(1, 2, 3, 4, 5, 6, 7, 8)

  val (evens, odds) = nums.partition(_ % 2 == 0)
  val (small, large) = nums.span(_ < 5)

  println(s"Evens: ${evens.mkString(", ")}")
  println(s"Odds:  ${odds.mkString(", ")}")
  println(s"Small: ${small.mkString(", ")}")
  println(s"Large: ${large.mkString(", ")}")
end main
```  

The `partition` method tests every element and distributes them into two new  
arrays. The `span` method is different: it splits at the first element that  
fails the predicate, keeping a prefix that all satisfies the condition in  
the first result and the remainder in the second.  

## Distinct elements  

This example shows how to remove duplicate values from an array.  

```scala
@main def main() =
  val nums = Array(3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5)

  val unique = nums.distinct
  val sorted = unique.sorted

  println(s"Original: ${nums.mkString(", ")}")
  println(s"Distinct: ${unique.mkString(", ")}")
  println(s"Sorted:   ${sorted.mkString(", ")}")
end main
```  

The `distinct` method returns a new array with duplicate elements removed,  
preserving the order of first occurrence. This is equivalent to converting  
to a `Set` and back, but `distinct` maintains order whereas a `Set` does  
not guarantee any particular ordering.  

## Flattening nested arrays  

This example demonstrates how to collapse an array of arrays into one.  

```scala
@main def main() =
  val nested = Array(
    Array(1, 2, 3),
    Array(4, 5),
    Array(6, 7, 8, 9)
  )

  val flat = nested.flatten

  println(flat.mkString(", "))
  println(s"Total elements: ${flat.length}")
end main
```  

The `flatten` method concatenates all inner arrays into a single flat array.  
It is available on any array whose element type is itself a collection. The  
result contains the same elements in the same relative order as the nested  
structure. This is equivalent to `nested.flatMap(identity)`.  

## Zip with index  

This example shows how to pair each element with its position.  

```scala
@main def main() =
  val langs = Array("Scala", "Java", "Python", "Go")

  langs.zipWithIndex.foreach { (lang, i) =>
    println(s"${i + 1}. $lang")
  }
end main
```  

The `zipWithIndex` method produces an array of `(element, index)` tuples  
without requiring a separate range. Using `i + 1` in the output converts  
the zero-based index to a one-based rank. This pattern is cleaner than  
manually maintaining a counter variable inside a loop.  

## Collecting with collect  

This example demonstrates selective transformation using a partial function.  

```scala
@main def main() =
  val values: Array[Any] = Array(1, "hello", 2, "world", 3, true)

  val ints = values.collect { case n: Int => n * 10 }
  val strs = values.collect { case s: String => s.toUpperCase }

  println(ints.mkString(", "))
  println(strs.mkString(", "))
end main
```  

The `collect` method applies a partial function to each element, keeping  
only those for which the partial function is defined. It combines filtering  
and mapping in a single pass. Pattern-matching on runtime types inside  
`collect` is a clean way to extract and transform elements of a mixed-type  
array.  

## Arrays with generics  

This example shows how to write generic functions that operate on arrays.  

```scala
def printAll[A](arr: Array[A]): Unit =
  arr.foreach(println)

def sumOf[A](arr: Array[A])(using num: Numeric[A]): A =
  arr.foldLeft(num.zero)(num.plus)

@main def main() =
  val ints = Array(1, 2, 3, 4, 5)
  val doubles = Array(1.5, 2.5, 3.0)

  printAll(ints)
  println(sumOf(ints))
  println(sumOf(doubles))
end main
```  

Generic functions accept type parameters so they work with any element type.  
The `using` clause brings a `Numeric[A]` instance into scope, allowing  
arithmetic operations without knowing the concrete type at the call site.  
The compiler resolves the implicit automatically for `Int` and `Double`.  

## Arrays with given Ordering  

This example demonstrates custom sorting using a `given` instance.  

```scala
case class Person(name: String, age: Int)

given Ordering[Person] = Ordering.by(_.age)

@main def main() =
  val people = Array(
    Person("Alice", 30),
    Person("Bob", 25),
    Person("Carol", 35),
    Person("Dave", 28)
  )

  val byAge = people.sorted
  val byName = people.sortBy(_.name)

  byAge.foreach(p => println(s"${p.age}  ${p.name}"))
  println("---")
  byName.foreach(p => println(p.name))
end main
```  

Defining a `given Ordering[Person]` in scope allows `sorted` to work on  
an `Array[Person]` without any explicit comparator. The `Ordering.by`  
factory lifts a key-extraction function into a full `Ordering`. When a  
different ordering is needed, `sortBy` or `sortWith` can override it  
inline.  

## Arrays with match types  

This example introduces match types to reflect on the element type of an array.  

```scala
type Elem[C] = C match
  case Array[t] => t

def headOf[C <: Array[?]](c: C): Elem[C] =
  c(0).asInstanceOf[Elem[C]]

@main def main() =
  val nums = Array(10, 20, 30)
  val words = Array("hello", "world")

  val n: Int = headOf(nums)
  val w: String = headOf(words)

  println(n)
  println(w)
end main
```  

A match type reduces to a concrete type by pattern-matching on a type  
argument at compile time. `Elem[Array[Int]]` reduces to `Int` and  
`Elem[Array[String]]` reduces to `String`. The function `headOf` uses this  
to express a precise return type that reflects the input array's element  
type, providing full static safety without extra wrapper types.  

## Arrays with context functions  

This example shows context functions used to thread implicit capacity.  

```scala
type WithCapacity[A] = Int ?=> A

def filledArray[A](value: A): WithCapacity[Array[A]] =
  Array.fill(summon[Int])(value)

@main def main() =
  given capacity: Int = 5

  val nums = filledArray(0)
  val flags = filledArray(false)

  println(nums.mkString(", "))
  println(flags.mkString(", "))
end main
```  

A context function type `A ?=> B` is a function that implicitly requires a  
value of type `A` to produce a result of type `B`. Here `WithCapacity[A]`  
threads an implicit `Int` through the call without requiring explicit  
parameter passing. The `summon[Int]` call retrieves the given integer from  
the current implicit scope.  

## Java API interoperability  

This example demonstrates using Java's `Arrays` utility class with Scala arrays.  

```scala
import java.util.Arrays as JArrays

@main def main() =
  val nums = Array(5, 3, 1, 4, 2)

  JArrays.sort(nums)
  println(nums.mkString(", "))

  val idx = JArrays.binarySearch(nums, 3)
  println(s"Found 3 at index $idx")

  val copy = JArrays.copyOf(nums, nums.length)
  copy(0) = 99
  println(nums.mkString(", "))
  println(copy.mkString(", "))
end main
```  

Because a Scala `Array[Int]` is a plain Java `int[]`, all methods of  
`java.util.Arrays` work directly on it without any wrapping or conversion.  
`binarySearch` requires the array to be sorted first and returns a negative  
value if the element is absent. `copyOf` is the Java equivalent of `clone`.  

## Performance-critical computation  

This example shows a tight numeric loop over a large array.  

```scala
@main def main() =
  val n = 1_000_000
  val data = Array.tabulate(n)(i => i.toDouble + 1.0)

  var sum = 0.0
  var i = 0
  while i < data.length do
    sum += data(i)
    i += 1

  println(f"n      = $n")
  println(f"Sum    = $sum%.0f")
  println(f"Mean   = ${sum / n}%.2f")
end main
```  

For maximum throughput, a `while` loop with a manual index variable avoids  
the small overhead of closure allocation that higher-order methods like  
`foreach` or `foldLeft` may incur. On the JVM, this pattern is compiled  
to bytecode nearly identical to a Java `for` loop and is fully optimised  
by the JIT compiler. Arrays of primitives do not box their elements.  

## Functional pipeline  

This example chains multiple transformations over an array in a single  
expression.  

```scala
@main def main() =
  val data = Array(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

  val result = data
    .filter(_ % 2 == 0)
    .map(n => n * n)
    .foldLeft(0)(_ + _)

  println(s"Sum of squares of even numbers: $result")
end main
```  

Chaining `filter`, `map`, and `foldLeft` expresses a multi-step computation  
declaratively. Each intermediate step produces a new array, so the pipeline  
reads top-to-bottom as a sequence of transformations. For very large arrays  
where allocation matters, consider converting to a `LazyList` or using a  
`View` to fuse the operations.  

## Domain modeling  

This example uses an array of case classes to represent an inventory.  

```scala
case class Product(name: String, price: Double, quantity: Int)

@main def main() =
  val inventory = Array(
    Product("Apple",  0.5, 100),
    Product("Banana", 0.3, 200),
    Product("Cherry", 2.0,  50)
  )

  val totalValue = inventory.map(p => p.price * p.quantity).sum
  val expensive  = inventory.filter(_.price > 0.4)
  val cheapest   = inventory.minBy(_.price)

  println(f"Total value: $$$totalValue%.2f")
  expensive.foreach(p => println(s"Expensive: ${p.name}"))
  println(s"Cheapest: ${cheapest.name}")
end main
```  

Arrays of case classes provide a lightweight data model for structured  
records. Higher-order methods make queries and aggregations concise: `map`  
computes per-item value, `sum` adds them, `filter` selects items by  
criteria, and `minBy` finds the record with the smallest key. This style  
scales cleanly to more complex domain objects.  

## Sliding windows  

This example demonstrates processing an array in overlapping and  
non-overlapping windows.  

```scala
@main def main() =
  val temps = Array(18.0, 20.0, 19.5, 21.0, 22.5, 20.0, 18.5)

  val movingAvg = temps
    .sliding(3)
    .map(w => w.sum / w.length)
    .toArray

  println("3-day moving averages:")
  movingAvg.foreach(avg => println(f"  $avg%.2f"))

  println("Daily batches:")
  temps.grouped(2).foreach(g => println(s"  ${g.mkString(", ")}"))
end main
```  

The `sliding(n)` method produces an iterator of overlapping windows of  
length `n`, advancing one element at a time. The `grouped(n)` method  
produces non-overlapping batches of up to `n` elements. Both return  
iterators; calling `toArray` materialises the results when needed. Moving  
averages are a classic use case for sliding windows.  

## Idiomatic array usage  

This example illustrates best practices for using arrays in Scala 3 code.  

```scala
@main def main() =
  // Use Array.tabulate for index-derived values
  val squares = Array.tabulate(6)(n => n * n)

  // Convert from collections with toArray
  val fromRange = (1 to 5).toArray

  // Prefer IArray for safe immutable sharing
  val fixed: IArray[Int] = IArray.from(squares)

  // Use higher-order methods for transformation
  val doubled = squares.map(_ * 2)

  println(squares.mkString(", "))
  println(fromRange.mkString(", "))
  println(fixed.mkString(", "))
  println(doubled.mkString(", "))
end main
```  

For arrays that must be shared across components without risk of mutation,  
`IArray[T]` (immutable array) is the idiomatic Scala 3 choice. It wraps  
a regular array but exposes no update method, providing the same O(1)  
access performance with a stronger safety guarantee. Use plain `Array`  
only when in-place mutation or Java interoperability is required.  

