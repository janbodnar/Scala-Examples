# Control flow

## If conditions

```scala
@main def main() = 

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

## For with range

```scala
@main def main() =

    for n <- 1 to 10 do
        println(n)
```

## Multiple for generators

```scala
@main def main() = 

    val vals1 = List('A', 'B', 'C', 'D', 'E')
    val vals2 = List(1, 2, 3, 4, 5)

    val res = for 
        e1 <- vals1 
        e2 <- vals2
    yield 
        s"$e1$e2"

    println(res)
```

## Guards

```scala
@main def main() =

    val words = List("sky", "war", "water", "rain", 
        "some", "cup", "train", "wrinkle", "worry")

    val res = for
        word <- words
        if word.startsWith("w")
    yield
        word.length 

    println(res)
```

## Multiple guards

```scala
@main def main() =

    val words = List("sky", "war", "water", "rain", 
        "some", "cup", "train", "wrinkle", "worry")

    val res = for
        word <- words
        if word.startsWith("w")
        if word.endsWith("r")
    yield
        word 

    println(res)
```

## For comprehension

```scala
case class User(name: String, age: Int)

val users = List(
  User("John", 18),
  User("Brian", 31),
  User("Veronika", 23),
  User("Lucia", 48),
  User("Peter", 21))

val res =

  for (user <- users if user.age >= 20 && user.age < 30)
  yield user


@main def main() = 

    res.foreach(println) 
```

## While loop

```scala
@main def main() = 

    val vals = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    val n = vals.length

    var i = 0
    var msum = 0

    while i < n do 
        msum = msum + vals(i)
        i = i + 1

    println(msum)
```

## Match expression

```scala
@main def main() = 
    
    val grades = List("A", "B", "C", "D", "E", "F", "FX")
    
    for grade <- grades do
        grade match
            case "A" | "B" | "C" | "D" | "E" | "F" => println("passed")
            case "FX" => println("failed")
```

