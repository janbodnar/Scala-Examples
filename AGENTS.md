Got it, Jan — thanks for the clarification.  
You want **AGENTS.md**, but *not about agents themselves*.  
This file is meant to be **instructions for your AI agents** that generate  
Scala 3 documentation inside your `Scala-Examples` repository.

Below is the **complete AGENTS.md**, rewritten to:  
- match the style of your Java/Go/Dart AGENTS.md files  
- remove all references to “agents” as a programming concept  
- focus purely on **documentation‑generation rules**  
- enforce **80‑character lines**, each ending with **two spaces**  
- target **Scala 3**  
- use the same tone and structure as your other language repos  

I will now generate the full file.

# **AGENTS.md**  
*(for generating Scala 3 documentation)*  

```markdown
# Instructions  

## Introduction  

This document defines the rules for generating Scala 3 documentation within the  
Scala-Examples repository. The goal is to maintain a consistent structure, tone,  
and formatting style across all example files. These guidelines ensure that the  
generated content is clear, concise, and aligned with the conventions used in  
other language example repositories. All text must follow the 80-character line  
limit, and each line must end with two spaces.  

The documentation should help readers understand Scala 3 concepts through short  
explanations and progressively structured examples. The writing style should be  
simple and direct, avoiding unnecessary theory or advanced language features  
unless the topic explicitly requires them. The examples must be runnable, easy  
to follow, and written in idiomatic Scala 3.  

## Code descriptions  

When a section contains code, include a brief summary directly below the title.  
This summary should explain the purpose of the example in one or two sentences.  
Use fenced code blocks with the Scala language tag to ensure dark-theme syntax  
highlighting. The code should be minimal, focused, and runnable without external  
dependencies unless the section explicitly introduces them.  

After each code block, provide a detailed explanation of the example. Use at  
least two or three sentences for simple snippets, and more for complex ones.  
Explain the control flow, the purpose of key constructs, and any Scala 3  
features used. Keep all prose lines within the 80-character limit and end each  
line with two spaces.  

Maintain consistency with existing documentation. Follow the same tone, spacing  
rules, and formatting conventions used in other example files across the  
repository.  

## Example structure  

Examples must build progressively, starting with simple concepts and moving  
toward more advanced topics. Each section should introduce only one idea at a  
time. Avoid numbering section titles. Do not use titles such as *Example 1:  
Variables*. Instead, use plain descriptive titles like *Variables*, *Functions*,  
or *Pattern matching*.  

Follow standard Java-style naming conventions for section titles. Keep titles  
short and focused on the concept being demonstrated. Ensure that each example  
is self-contained and does not rely on previous examples unless the section  
explicitly states that it builds on earlier material.  

## Language features (Scala 3)  

Use modern Scala 3 features where they improve clarity. Prefer simple, readable  
syntax over advanced constructs. You may use top-level definitions, enums,  
extension methods, and given instances when appropriate. Avoid complex type  
level programming or macros unless the section specifically covers them.  

When demonstrating concurrency, collections, or functional patterns, rely on  
standard library features. External libraries should be introduced only in  
clearly marked sections, and the examples should remain understandable even  
without them.  

## Code format  

Use fenced code blocks with the Scala language tag:  

```scala
@main def demo(): Unit =
  println("Hello, Scala 3")
end demo
```  

Keep examples short and runnable. Use clear names for variables, functions, and  
types. Follow standard Scala 3 indentation and formatting conventions. Comments  
inside code should be brief and helpful, staying within the 80-character limit  
when possible.  

Avoid unnecessary boilerplate. Use top-level definitions for simple examples.  
When a more complex structure is required, introduce it gradually and explain  
each part clearly.  

## Terminology  

Use consistent terminology throughout the documentation. Refer to functions  
without parentheses when mentioned in prose, such as *the `demo` function*.  
Apply the same rule to other definitions like `run`, `apply`, or `process`.  

Use precise language when describing Scala features. Distinguish between  
expressions, values, types, and definitions. When discussing concurrency or  
asynchronous behavior, clearly separate conceptual behavior from execution  
details such as threads or execution contexts.  

Maintain consistent phrasing across all sections so that readers can easily  
transfer understanding from one topic to another.  

Just tell me what you want next.
