# Control flow

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

## Iterate array & list

```scala
val nums = Array(1, 2, 3, 4)
for e <- nums do println(e)

val nums2 = List(1, 2, 3, 4)
for e <- nums2 do println(e)
```


## For with map

```scala
val cts = Map("sk" -> "Slovakia", "ru" -> "Russia", 
    "de" -> "Germany", "no" -> "Norway")

for (k, v) <- cts do println(s"$k $v")
```

## For with range

```scala
for n <- 1 to 10 do
    println(n)
```

## Multiple for generators

```scala
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
val grades = List("A", "B", "C", "D", "E", "F", "FX")

for grade <- grades do
    grade match
        case "A" | "B" | "C" | "D" | "E" | "F" => println("passed")
        case "FX" => println("failed")
```

