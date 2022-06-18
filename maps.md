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

## Sorting

```scala
val capitals = Map("Bratislava" -> 424207, "Vilnius" -> 556723,
    "Lisbon" -> 564657, "Riga" -> 713016, "Jerusalem" -> 780200, 
    "Warsaw" -> 1711324, "Budapest" -> 1729040, "Prague" -> 1241664,
    "Helsinki" -> 596661, "Tokyo" -> 13189000, "Madrid" -> 3233527)

@main 
def main() =

    println(capitals.toSeq.sortBy(_._1))
    println(capitals.toSeq.sortBy(_._2))

    println("----------------------------------")

    println(capitals.toSeq.sortWith((a, b) => a._2 < b._2))
    println(capitals.toSeq.sortWith((a, b) => a._2 > b._2))
 ```
