# Functions

## Blocks

Functions accept blocks

```scala
@main def main(): Unit =

    List(1, 2, 3).map { e =>  println( e * 2) }
```

## Single abstract methods

Single abstract methods can be written as lambdas.  

```scala
trait Action:
    def act(x: Int): Int

@main def hello(): Unit =

    val a = new Action {
      def act(x: Int): Int = x * 10
    }

    println(a.act(10))

    val a2: Action = (x: Int) => x * 20
    println(a2.act(10))
```
