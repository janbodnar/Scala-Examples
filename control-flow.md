# Control flow

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

## For expression

```scala
@main def main() = 

    val vals = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    var msum = 0

    for e <- vals do 
        msum = msum + e

    println(msum)
```

## Multiple fors

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

## While 

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
