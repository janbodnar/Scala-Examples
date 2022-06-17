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
