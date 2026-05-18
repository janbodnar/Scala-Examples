# Instructions  

## Introduction  

At the beginning of the document, provide a detailed, in-depth overview of the  
topic of agents in Scala 3. Explain what agents are in the context of modern  
Scala applications, how they relate to concurrency, message passing, and state  
management, and where they fit among other concurrency tools such as futures,  
promises, and effect systems.  

Describe typical use cases where agents are useful, such as coordinating work  
between background tasks, isolating mutable state, or integrating with external  
services. Highlight the benefits of using agents, including safer concurrency,  
clearer ownership of state, and improved modularity, as well as their limits,  
such as potential overhead, complexity, or debugging challenges.  

Place agents in the broader Scala ecosystem. Mention that Scala 3 encourages  
composable, type-safe abstractions, and that agents can be implemented using  
standard libraries, lightweight wrappers, or higher-level frameworks. Emphasize  
that the examples in this document focus on clear, idiomatic Scala 3 code that  
can be easily adapted to real-world projects.  

## Code descriptions  

When asked to add a description for a section that contains code, follow these  
rules consistently throughout the document.  

- Include a brief summary directly below the section title. This summary should  
  explain in one or two sentences what the example demonstrates and why it is  
  relevant. Keep it concise but informative.  
- Provide the code in a fenced code block using the Scala language identifier,  
  so that it is rendered with a dark theme in typical editors and viewers.  
- After the code block, add a detailed explanation of the example. Use at least  
  two to three sentences for simple snippets, and more for complex examples.  
  Explain the control flow, important types, and any concurrency or agent logic.  
- Limit all prose lines to 80 characters or fewer, and end each line with two  
  spaces. Apply this rule to headings, paragraphs, list items, and comments in  
  descriptive text.  
- Maintain consistency with existing documents in the repository. Match the  
  tone, level of detail, and structure used in other Scala-Examples markdown  
  files, including spacing, heading levels, and naming conventions.  

The explanation following each code block should help readers understand not  
only what the code does, but also how it relates to the concept of agents.  
Where appropriate, mention trade-offs, alternative approaches, and how the  
pattern scales to larger applications.  

## Example structure  

Examples in this document should build progressively, starting from simple  
concepts and moving toward more advanced patterns. Organize sections so that  
each new example assumes familiarity with the previous ones, without repeating  
unnecessary details.  

Avoid numbering section titles. Do not use titles such as *Example 1: Basic  
agent* or *Example 2: Message passing*. Instead, use plain, descriptive titles  
like *Basic agent*, *Message passing between agents*, or *Supervising failing  
agents*. This keeps the document clean and consistent with other examples.  

Follow standard Java-style naming conventions for section titles and identifiers  
where it makes sense, while still writing idiomatic Scala 3 code. Titles should  
be in title case, short, and focused on the concept being demonstrated.  

Ensure that each example introduces exactly one main idea. For instance, start  
with a minimal agent that processes messages sequentially, then add examples  
that show stateful agents, error handling, supervision, and integration with  
futures or effect systems.  

## Language features (Scala 3)  

Utilize modern Scala 3 features where they improve clarity and safety, while  
keeping examples accessible to readers who know basic Scala syntax. Prefer  
simple, direct code over overly clever constructs.  

You may use the following Scala 3 features when appropriate:  

- Top-level definitions for simple examples, avoiding unnecessary wrapper  
  objects when a standalone `@main` entry point is sufficient.  
- Given instances and extension methods when they clarify agent configuration  
  or message handling, but avoid heavy type-level programming in basic sections.  
- Enums for modeling message types or agent states, providing clear and  
  exhaustive pattern matching.  
- Using clauses and context parameters where they make concurrency or execution  
  context handling explicit and readable.  

Avoid relying on external libraries unless the section explicitly focuses on  
integration with a specific framework. When external libraries are used, mark  
the section clearly and keep the core ideas understandable even without that  
library.  

## Code format  

Each example should follow a consistent structure that mirrors other language  
example files in the repository. Use a simple entry point and keep the code  
focused on the agent-related concept being demonstrated.  

Use fenced code blocks with the Scala language tag to ensure dark-theme syntax  
highlighting:  

```scala
@main def basicAgentDemo(): Unit =
  import scala.concurrent.ExecutionContext
  import scala.concurrent.ExecutionContext.Implicits.global
  import scala.concurrent.Future
  import scala.collection.mutable.Queue

  final class Agent[A](initial: A)(using ec: ExecutionContext):
    private val mailbox = Queue[A]()
    private var state: A = initial

    def send(msg: A): Unit =
      mailbox.enqueue(msg)
      Future:
        processNext()

    private def processNext(): Unit =
      if mailbox.nonEmpty then
        val msg = mailbox.dequeue()
        state = msg
        println(s"New state: $state")

  val agent = Agent[Int](0)
  agent.send(1)
  agent.send(2)
  agent.send(3)
```

Keep the example self-contained and runnable. Prefer small, focused snippets  
over large, multi-file setups. Use clear names for functions, values, and types,  
and keep indentation consistent with standard Scala 3 style.  

When comments are necessary inside code blocks, keep them short and aligned with  
the 80-character limit where practical. Comments should clarify intent, not  
repeat what the code already states.  

## Terminology  

When referencing functions or methods in the prose, use their names without  
parentheses. For example, write *the `basicAgentDemo` function* instead of  
*`basicAgentDemo()`*. Apply the same rule to other methods such as `send` or  
`processNext`.  

Be precise and consistent in terminology across the document. Use *agent* for  
the abstraction that owns and processes messages, *message* for the values sent  
to an agent, and *mailbox* or *queue* for the internal buffer of pending  
messages.  

When discussing concurrency, distinguish clearly between *logical concurrency*  
(the structure of tasks and agents) and *execution* (threads, execution  
contexts, or runtimes). This helps readers reason about behavior without  
confusing implementation details with conceptual models.  

Use the same terms and phrasing throughout all sections so that readers can  
transfer understanding from one example to the next without re-learning the  
vocabulary.  
