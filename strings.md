# Strings


## Concatenate

```scala
@main def main() = 

    val w1 = "an"
    val w2 = "old"
    val w3 = "falcon"

    val msg = w1 + " " + w2 + " " + w3
    println(msg)

    println(w1.concat(" ").concat(w2).concat(" ").concat(w3))

    println(s"$w1 $w2 $w3")
    println(f"$w1%s $w2%s $w3%s")
    printf("%s %s %s%n", w1, w2, w3)
```

## String to Int

```scala
@main def main() = 

    val vals = List[String | Int]("3", 12, "11", 5, 6, "8") 
 
    val vals2 = vals.map(_.toString.toInt)
    println(vals2.sum)

    var msum = 0

    for e <- vals do 
        msum = msum + e.toString.toInt

    println(msum)
```

## StringBuilder

Immutable strings with `StringBuilder`.   

```scala
import scala.collection.mutable.StringBuilder

@main def main() =

    val name = "Jane"
    val name2 = name.replace('J', 'K')
    val name3 = name2.replace('n', 't')

    println(name)
    println(name3)

    val sb = StringBuilder("Jane")
    println(sb)

    sb.setCharAt(0, 'K')
    sb.setCharAt(2, 't')

    println(sb)
```

## Multiline strings

```scala
@main def main() =

    val sonnet55 =
        """Not marble nor the gilded monuments
        |Of princes shall outlive this powerful rhyme,
        |But you shall shine more bright in these contents
        |Than unswept stone besmeared with sluttish time.
        |When wasteful war shall statues overturn,
        |And broils root out the work of masonry,
        |Nor Mars his sword nor war's quick fire shall burn
        |The living record of your memory.
        |'Gainst death and all-oblivious enmity
        |Shall you pace forth; your praise shall still find room
        |Even in the eyes of all posterity
        |That wear this world out to the ending doom.
        |So, till the Judgement that yourself arise,
        |You live in this, and dwell in lovers' eyes.""".stripMargin

    println(sonnet55)
```

## Escape characters

```scala
@main def main() =

    println("Three\t bottles of wine")
    println("He said: \"I love ice skating\"")
    println("Line 1:\nLine 2:\nLine 3:")
```

## Raw strings 

```scala
@main def main() = 

    println(raw"snow\tshow\tsnow")
    println("becomes") 
    println("snow\tshow\tsnow")
```

## String to integer

```scala
@main def main() =

    val vals = List[String | Int]("3", 12, "11", 5, 6, "8")

    val vals2 = vals.map(_.toString.toInt)
    println(vals2.sum)

    var msum = 0

    for e <- vals do
        msum = msum + e.toString.toInt

    println(msum)
```


## Multiply

```scala
@main def main() =

    val msg = "The shadown of the beast"
    val n = msg.length

    println("-" * n)
    println(msg)
    println("-" * n)
```

## Iterate runes

```scala
import java.text.BreakIterator

@main def main() = 

    val text = "🐜🐬🐄🐘🦂🐫🐑🦍🐯🐞"

    val it = BreakIterator.getCharacterInstance
    it.setText(text)

    var start = it.first
    var end = it.next

    while start < end do 

        println(text.substring(start, end))
        start = end 
        end = it.next
```

## Scala regex matches

```scala
@main def main() =

    var words = """
            |book
            |bookshelf
            |bookworm
            |bookcase
            |bookish
            |bookkeeper
            |booklet
            |bookmark
            """.stripMargin

    var wstream = words.lines

    wstream.forEach(word =>

        if word.matches("book(worm|mark|keeper)?") then
            println(word)
    )
```
