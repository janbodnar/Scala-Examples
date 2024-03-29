# Scala-Examples
Scala code examples

Scala is a functional language with a powerful type system.  

## Install & setup 

Install Scala from https://www.scala-lang.org/download/  

`sbt new scala/scala3.g8` - creates new Scala 3 project  

`cs install scala:3.3.1 && cs install scalac:3.3.1` - install specific Scala version with Courstier  

Get Scala version  

```
$ scala --version
Scala code runner version 3.3.1 -- Copyright 2002-2023, LAMP/EPFL
```

Coursier setup  

```
$ cs setup
Checking if a JVM is installed
...
```

Run script with ammonite 

`amm First.scala`  

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

## Multi-word methods

```scala
class Message(who: String):
  def `send message`(msg: String): Unit = println(s"$who said: $msg")

@main def main() =

  val m1 = Message("Martin")
  m1 `send message` "Hello there!"

  val m2 = Message("Jozef")
  m2 `send message` "How are you?"
```

## Tuples

```scala
@main def main() =

  val ui = ("John Doe", "gardener", 34)
  println(ui)

  println(ui._1)
  println(ui._2)
  println(ui._3)

  println("-------------------------")

  ui.productIterator.foreach(println)

  println("-------------------------")

  val (name, occupation, _) = ui
  
  println(name)
  println(occupation)
```

## Array to string

```scala
val a = Array(1, 2, 3, 4, 5)
println(a.mkString(", "))
```

## Singleton

```scala
object Counter:
  private var counter: Int = 0
  val label: String = "Counter"

  def increment(): Unit =
    counter += 1

  def get: Int = counter


@main def main(): Unit =

    val c = Counter
    c.increment()
    c.increment()
    c.increment()

    println(c.get)
```

## Infix types 

```scala
class -->[Int, String]
val c: Int --> String = ???

class <[Int, Int]
val lessThen: Int < Int = ???
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

## Enums

```scala
enum Season:
  case Spring, Summer, Autumn, Winter

object Season:
  def rand(): Season = Random.shuffle(Season.values).head

def checkSeason(season: Season) = season match
  case Season.Spring => "Its spring"
  case Season.Summer => "Its summer"
  case Season.Autumn => "Its autumn"
  case Season.Winter => "Its winter"

@main def main(): Unit =

  println(checkSeason(Season.rand()))
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

## Extension methods

Extension methods enhance the capabilities of objects.  

```scala
import scala.io.Source
import java.io.File

extension (f: File)
    def read() = Source.fromFile(f).getLines()

@main def hello(): Unit =

    val f = new File("src/main/scala/words.txt")
    f.read().foreach(println)
```

```scala
case class Circle(x: Double, y: Double, radius: Double)

extension (c: Circle)
  def circumference: Double = c.radius * math.Pi * 2
  def diameter: Double = c.radius * 2
  def area: Double = math.Pi * c.radius * c.radius


@main def main() = 

    val c = Circle(4, 5, 5)
    println(c.area)
    println(c.diameter)
    println(c.circumference)
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
