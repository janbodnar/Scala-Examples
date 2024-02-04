# Classes

## Primary constructors

```scala
class User(var name: String, var occupation: String):
  override def toString(): String = s"$name - $occupation"

@main def main() =

  val u1 = User("John Doe", "gardener")
  println(u1)
```

## Getters & setter

```scala
class User:
  private var _name: String = ""
  private var _occupation: String = ""

  def name: String = _name
  def name_=(value: String): Unit = _name = value

  def occupation: String = _occupation
  def occupation_=(value: String): Unit = _occupation = value

  override def toString(): String = s"$name - $occupation"

@main def main() =

  val u1 = User()

  u1.name = "John Doe"
  u1.occupation = "gardener"
  println(u1)
```
