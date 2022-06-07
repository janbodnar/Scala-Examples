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

    val url = URL("http://example.com/favicon.ico")

    Using(url.openStream) { in =>

        Files.copy(in, Paths.get("favicon2.ico"))
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
