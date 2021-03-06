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
