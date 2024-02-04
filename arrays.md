# Arrays


## ToString

```scala
val nums = Array(1, 2, 3, 4)
println(nums.mkString(", "))
println(nums.mkString("[", ", ", "]"))
```

## Array comprehension

```scala
val a = Array(1, 2, 3, 4, 5)
val b = for (e <- a if e > 2) yield e

println(b.mkString(", "))
```

## Update

```scala
val a = Array(1, 2, 3, 4, 5)

a(0) = 11 // a.update(0, 11)
a(1) = 22 // a.update(1, 22)

println(a.mkString(", "))
```
