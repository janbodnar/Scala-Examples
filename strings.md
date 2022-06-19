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
