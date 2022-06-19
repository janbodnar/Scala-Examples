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
