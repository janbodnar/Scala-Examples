# Maps

## Basics 

```scala
@main 
def main() =

    val cts = Map("sk" -> "Slovakia", "ru" -> "Russia", 
        "de" -> "Germany", "no" -> "Norway")

    println(cts("sk"))
    println(cts.get("sk"))
    println(cts.size)

    val cts2 = cts + ("hu" -> "Hungary")
    println(cts2.contains("hu"))
```

## Traversing 

```scala
@main 
def main() =

    val cts = Map("sk" -> "Slovakia", "ru" -> "Russia", 
        "de" -> "Germany", "no" -> "Norway")

    for (k, v) <- cts do println(s"${k} ${v}")

    cts.keys.foreach(k => println(k))
    cts.values.foreach(v => println(v))
```
