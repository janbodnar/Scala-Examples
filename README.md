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

## Download image

```scala
import java.io.File
import java.net.URL
import scala.sys.process._
import scala.language.postfixOps

@main def main() =

    URL("http://webcode.me/favicon.ico") #> File("favicon.ico") !
```

--

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
