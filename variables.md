# Scala Variables  

Variables in Scala represent named references to data held in memory.  
They act as containers that persist values throughout program execution.  
The language draws a sharp distinction between immutable bindings and mutable state.  
A val keyword binds an identifier to a value that cannot change later.  
A var keyword binds an identifier to a value that may be reassigned freely.  
Functional programming strongly favours immutable bindings for correctness.  
Immutability eliminates hidden side effects and simplifies logical reasoning.  
It naturally prevents data races in concurrent code without requiring locks.  
Immutable data guarantees pure functions and highly reliable unit tests.  

## `val` versus `var`  

The val keyword creates an immutable reference in Scala 3.  
Once a value is assigned to a val, the reference cannot be replaced.  

```scala
@main def run() =

    val maxRetries = 3
    // maxRetries = 5  // Compiler error: reassignment to val
end run
```  

Attempting to reassign a val triggers a strict compilation failure.  
The compiler reports a clear diagnostic about illegal reassignment.  
In contrast, the var keyword creates a fully mutable reference.  
You may assign a new value to a var at any point during execution.  

```scala
@main def run() =

    var currentScore = 0
    currentScore = 10
    currentScore = 25
    println(currentScore)
end run
```  

Mutable bindings are occasionally useful for counters or accumulators.  
They should be avoided when the same goal can be achieved with val.  
Idiomatic Scala code defaults to val unless mutation is strictly required.  

## Reassignment versus object mutation  

Reassigning a variable replaces the reference pointer itself.  
Mutating an object modifies its internal fields while keeping the reference intact.  
Scala separates these two operations to clarify developer intent.  

```scala
@main def run() =

    import scala.collection.mutable.ArrayBuffer

    val buffer = ArrayBuffer(1, 2, 3)
    buffer.append(4)  // Mutation is allowed on the underlying object
    // buffer = ArrayBuffer(5, 6) // Reassignment is forbidden for val

    var items = List("apple", "banana")
    items = List("cherry", "date") // Reassignment is allowed for var
end run
```  

The val reference stays fixed while the collection grows internally.  
The var reference points to an entirely new list after reassignment.  
Understanding this distinction prevents subtle bugs when tracking state.  

## Type inference  

The Scala compiler automatically deduces the type of every val and var.  
Explicit type annotations are optional but occasionally recommended.  
The type checker inspects the right-hand side of the assignment.  

```scala
@main def run() =

    val count = 42                // Inferred as Int
    val greeting = "Hello"        // Inferred as String
    val tags = Set("scala", "jvm") // Inferred as Set[String]

    val config: Map[String, Int] = Map( // Explicit type added
        "timeout" -> 30,
        "port"    -> 8080
    )
end run
```  

Type inference works seamlessly with primitives and standard collections.  
It also handles custom case classes and traits without verbose declarations.  
Use explicit annotations when the compiler infers an overly general type.  
Annotations are highly recommended for public APIs and library boundaries.  
They serve as clear documentation for developers reading the code.  

## Naming conventions and constants  

Variable names consistently use lower camelCase across the ecosystem.  
Classes, traits, and objects use UpperCamelCase instead.  
Avoid leading or trailing underscores in ordinary variable names.  
Underscores are strictly reserved for pattern matching and generated code.  
Choose descriptive names that communicate business purpose immediately.  
Single-letter names like x, y, or i should only appear in tight loops.  

```scala
@main def run() =

    val maximumRetryAttempts = 3  // Clear and descriptive
    val userId = "USR-992"        // Acceptable for standard abbreviations

    object DatabaseConfig:
        val DEFAULT_PORT = 5432               // Top-level constant
        final val SCHEMA_VERSION = "2.1"       // Compile-time constant
end run
```  

Top-level val declarations inside objects behave as module constants.  
A final val forces compile-time evaluation and bytecode inlining.  
It mirrors the static final pattern commonly used in other languages.  

## Scope and lifetime  

Every variable exists within a strictly defined lexical boundary.  
Local variables declared inside a brace block vanish after it closes.  
Method parameters are confined to the method body only.  
Class fields persist in memory as long as their parent instance lives.  

```scala
class DataProcessor:
    private var processedCount = 0 // Field scope: instance lifetime

    def execute(input: String): Int =
        val sanitized = input.trim()   // Method scope
        if sanitized.isEmpty then return 0

        var index = 0                  // Block scope
        var total = 0

        while index < sanitized.length do
            total += sanitized(index)
            index += 1

        processedCount = total         // Updates instance field
        total
end class
```  

Variables cannot escape outside their defining lexical region.  
Inner scopes may declare identifiers that shadow outer ones.  
Shadowing is technically valid but actively discouraged for clarity.  

## Progressive examples and best practices  

Begin with simple numeric bindings before introducing collections.  
Prefer functional pipelines over step-by-step imperative mutation.  
Restrict variable scope to the smallest possible enclosing region.  
Use lazy val sparingly, only for costly initialisation or circular deps.  

```scala
@main def run() =

    // Stage 1: Primitive immutable bindings
    val basePrice = 19.99
    val taxRate = 0.08

    // Stage 2: Strings and sequence collections
    val products = List("Widget", "Gadget", "Tool")
    val rawPrices = List(19.99, 24.50, 12.75)

    // Stage 3: Transformation replaces in-place mutation
    val taxedPrices = rawPrices.map(_ * (1 + taxRate))

    // Stage 4: Real-world aggregation with implicit inference
    val orderSummary = products
        .zip(taxedPrices)
        .map((name, price) => s"$name: $$$price")
        .mkString(", ")

    println(s"Invoice: $orderSummary")
end run
```  

The pipeline chains operations without altering any intermediate state.  
Each step yields a fresh immutable collection or formatted string.  
The compiler seamlessly infers List[Double] and String types.  
This functional style scales effortlessly to parallel execution threads.  
Always declare val first, and reach for var only after careful profiling.
