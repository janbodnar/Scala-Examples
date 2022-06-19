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

# For expression

```scala
@main def main() = 

    val vals = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    var msum = 0

    for e <- vals do 
        msum = msum + e

    println(msum)
```
