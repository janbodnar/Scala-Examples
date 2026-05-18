Absolutely, Jan — I’ve reviewed your existing `arrays.md` file  
from the Scala‑Examples repo (tab reference:   [github.com](https://github.com/janbodnar/Scala-Examples/blob/main/arrays.md)).  
I’ll now generate a **new, fully rewritten, repository‑consistent**,  
**Scala 3‑style**, **80‑character‑wrapped**, **two‑spaces‑per‑line**  
documentation file for **Scala arrays**.

This will **replace** the current minimal version with a complete,  
progressive, idiomatic Scala 3 tutorial — matching the tone and structure  
of your Go, Java, and Dart examples.

Here is the full document.

---

# **arrays.md**  

```markdown
# Arrays  

Arrays store a fixed number of elements of the same type. They provide fast  
indexed access and are useful when the size of the collection is known in  
advance. Scala arrays are mutable, interoperable with the JVM, and support  
common operations such as updating values, iterating, and transforming data.  

## Creating arrays  

This example shows how to create arrays with predefined values.  

```scala
@main def creatingArrays(): Unit =
  val nums = Array(1, 2, 3, 4)
  val words = Array("sky", "blue", "forest")

  println(nums.mkString(", "))
  println(words.mkString(", "))
```  

The example creates two arrays: one of integers and one of strings. The  
`mkString` method converts the array into a readable string. Arrays have a  
fixed size, so elements cannot be added or removed after creation.  

## Accessing elements  

This example demonstrates how to read values from an array using indices.  

```scala
@main def accessingElements(): Unit =
  val nums = Array(10, 20, 30, 40)

  println(nums(0))
  println(nums(3))
```  

Array indices start at zero. Accessing an element uses parentheses, not square  
brackets. Attempting to read an index outside the valid range throws an  
exception.  

## Updating elements  

This example shows how to modify existing values in an array.  

```scala
@main def updatingElements(): Unit =
  val nums = Array(1, 2, 3, 4)

  nums(0) = 11
  nums(1) = 22

  println(nums.mkString(", "))
```  

Arrays in Scala are mutable, so elements can be reassigned. The syntax  
`nums(i) = value` updates the element at index `i`. The underlying call uses  
the `update` method, but the assignment syntax is preferred for clarity.  

## Array comprehension  

This example demonstrates how to build a new array using a `for` expression.  

```scala
@main def arrayComprehension(): Unit =
  val nums = Array(1, 2, 3, 4, 5)

  val filtered = for
    n <- nums
    if n > 2
  yield n

  println(filtered.mkString(", "))
```  

A `for` expression with `yield` creates a new array. The original array remains  
unchanged. Filtering inside the comprehension allows concise transformation of  
data.  

## Iterating over arrays  

This example shows several ways to iterate through array elements.  

```scala
@main def iteratingArrays(): Unit =
  val nums = Array(3, 6, 9)

  for n <- nums do println(n)

  nums.foreach(println)

  for i <- nums.indices do
    println(s"Index $i has value ${nums(i)}")
```  

Scala provides multiple iteration styles. The `foreach` method is concise,  
while iterating over `indices` gives access to both index and value. All  
approaches produce the same result, so choose the one that fits the context.  

## Transforming arrays  

This example demonstrates mapping and filtering operations.  

```scala
@main def transformingArrays(): Unit =
  val nums = Array(1, 2, 3, 4)

  val doubled = nums.map(_ * 2)
  val evens = nums.filter(_ % 2 == 0)

  println(doubled.mkString(", "))
  println(evens.mkString(", "))
```  

Although arrays are mutable, transformation methods like `map` and `filter`  
return new arrays. These operations do not modify the original array. This  
functional style is common in Scala and improves readability.  

## Converting arrays  

This example shows how to convert arrays to other collection types.  

```scala
@main def convertingArrays(): Unit =
  val nums = Array(1, 2, 3)

  val list = nums.toList
  val vector = nums.toVector
  val seq = nums.toSeq

  println(list)
  println(vector)
  println(seq)
```  

Scala collections interoperate smoothly. Converting an array to a list, vector,  
or sequence is straightforward. These conversions are useful when a more  
flexible or immutable collection type is needed.  
```

---

If you want, I can also generate:  
- the next document (e.g., **maps**, **strings**, **loops**, etc.)  
- a full rewrite of the entire Scala‑Examples repo in this new style  
- a script to validate 80‑character wrapping  

Just tell me what you want next.
