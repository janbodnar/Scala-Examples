# Types

```scala
class Being

val objects: List[Any] = List(1,-2, 3.4f, 4.3, None, false, List(1, 2), 
    "Python", (2, 3), new Being(), Map("sky" -> "obloha"))

@main def main =
  for e <- objects do
    e match
      case None            => println(s"$e is a null value")
      case _: List[_]      => println(s"$e is a list")
      case _: Tuple2[_, _] => println(s"$e is a tuple")
      case _: Float        => println(s"$e is a float")
      case _: Double       => println(s"$e is a double")
      case _: Boolean      => println(s"$e is a boolean value")
      case _: String       => println(s"$e is a string")
      case _: Map[_, _]    => println(s"$e is a map")
      case _: Int          => println(s"$e is an integer")
      case _: Being        => println(s"$e is a Being")
      case _               => println(s"$e is of unknown type")
```
