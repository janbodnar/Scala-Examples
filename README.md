# Scala-Examples
Scala code examples

Scala is a functional language with a powerful type system.  


## Basic string operations

```scala
@main def main() = 

    val msg = "Hello" + " there!"
    println(msg)

    val name = "John Doe"
    val occupation = "gardener"

    val msg2 = s"$name is a $occupation"
    println(msg2)

    val word = "falcon"
    println(s"$word has ${word.size} letters")
  ```

## Main function

```scala

@main def main(vals:Int*) =

    val sum = vals.sum
    println(sum)
```

## Read input 

```scala
@main def main() = 

    print("Enter your name: ")

    val name = io.StdIn.readLine

    printf("Hello %s!\n", name)
```

## Singleto 

```scala
object Counter:
  private var counter: Int = 0
  val label: String = "Counter"

  def increment(): Unit =
    counter += 1

  def get: Int = counter


@main def hello(): Unit =

    val c = Counter
    c.increment()
    c.increment()
    c.increment()

    println(c.get)
```

## Type inference 

In many cases, Scala can infer the types for many identifiers.   
Sometimes; however, we need to specify the types explicitly.  

```scala
val square = (x: Int) => x * x
val triple: (x: Int) => Int = (x) => x * x * x

@main def main() =

    val res = square(3)
    println(res)

    val res2 = triple(5)
    println(res2)
```


## Ranges

```scala

@main def main() = 

    for (i <- 1 to 3) println(i)

    println("---------------------")

    for (i <- 20 to 30 by 2) do println(i)

    println("---------------------")

    for (i <- 1 until 3) println(i)

    println("---------------------")

    val nums = List.range(10, 14)
    nums.foreach(println)

    println("---------------------")

    val res = (1 to 6).map(e => e * 2)
    for e <- res do println(e)

    println("---------------------")

    val r = 100 to 111

    println(r.size)
    println(r.contains(106))

    println(r.filter(_ % 2 == 0))
```

## :: and #:: operators

The operators are right-to-left associated. Scala looks at the last character of the operator.  

```scala
    val nums = 1 :: 2 :: 3 :: 6 :: List(4, 5)
    println(nums)

    val nums2 = 1 #:: 2 #:: 3 #:: LazyList(4, 5) 
    nums2.foreach(e => println(e))
```

## Shuffle list of cards

```scala
import scala.util.Random


@main
def main() = 

    val cards = List(
        "🂡", "🂱", "🃁", "🃑",
        "🂢", "🂲", "🃂", "🃒",
        "🂣", "🂳", "🃃", "🃓",
        "🂤", "🂴", "🃄", "🃔",
        "🂥", "🂵", "🃅", "🃕",
        "🂦", "🂶", "🃆", "🃖",
        "🂧", "🂷", "🃇", "🃗",
        "🂨", "🂸", "🂸", "🂸",
        "🂩", "🂩", "🃉", "🃙",
        "🂪", "🂺", "🃊", "🃚",
        "🂫", "🂻", "🃋", "🃛",
        "🂭", "🂽", "🃍", "🃝",
        "🂮", "🂾", "🃎", "🃞"
        )

    show(cards)

    println

    println("---------------")

    val shuffled = Random.shuffle(cards)

    show(shuffled)

    println
    
def show(cards: List[String]) = 
    
    cards.zipWithIndex.foreach { case (e, i) => 
        if i != 0 && i % 13 == 0 then println
        print(s"${e} ")
    }
```


## Download image

```scala
import java.io.File
import java.net.URL
import scala.sys.process._
import scala.language.postfixOps

@main def main() =

    URL("http://webcode.me/favicon.ico") #> File("favicon.ico") !
```

---

```scala
import java.nio.file.Files
import java.nio.file.Paths
import java.net.URL
import scala.util.Using

@main def main() =

    val url = URL("http://webcode.me/favicon.ico")

    Using(url.openStream) { in =>

        Files.copy(in, Paths.get("favicon.ico"))
    }
```


## Get HTML of web page

```scala
import scala.io.Source
import scala.util.Using

@main def main(): Unit  =

    Using(Source.fromURL("http://webcode.me")) { page =>

        val lines = page.getLines
        lines.foreach(println)
    }
```

## Execute process

The `!!` operator executes command and returns its output.  

```scala
import sys.process._
import scala.language.postfixOps

@main def main() = 

    val res = "ls -al" !!

    println(res)
```
