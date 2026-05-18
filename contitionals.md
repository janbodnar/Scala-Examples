# Conditionals

## If conditions

```scala
val r = Random.between(-10, 10)

if r > 0 then 
    println(s"positive value (${r})") 
else if r == 0 then
    println("zero value")
else
    println(s"negative value (${r})")
```

## If expressions

```scala 
import scala.util.Random

@main def main() =

    val r = Random.between(-10, 10)

    val res = if r > 0 then
        s"positive value ($r)"
    else if r == 0 then
        "zero value"
    else
        s"negative value ($r)"

    println(res)
```


