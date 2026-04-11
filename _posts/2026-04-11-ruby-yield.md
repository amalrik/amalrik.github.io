---
layout: post
title: "Why Ruby blocks are misunderstood"
date: 2026-04-11 10:00:00
lang: en
permalink: /posts/ruby-yield/
categories: [ruby]
tags: [ruby, yield, closures]
---

yield, honestly, is a word that translates poorly to other languages. The closest translation I could think in my native language (Portuguese) was "Ceder" (to yield) or "Desistir" (to give up), which doesn't convey the concept well. Throughout my career, I've had the opportunity to work with programmers of different seniority levels, and I've been surprised how often Ruby blocks aren't used properly. This post is my attempt to demystify this topic once and for all.

## Why do you forget to use blocks in your solutions?

This is because we think of methods as closed boxes - a mental model inherited from C or (old) Java, where everything known must be passed as a parameter. In other words, you haven't had enough exposure to the concept of closures, one of the most underestimated, relevant, and powerful concepts in programming.

The mental model of closures doesn't fit well with the object and noun-based mental model - it forces you to think in verbs and processes.

Whenever you find yourself writing the same "frame" of code in multiple places (e.g., open connection, do something, close connection), stop. Create a method that uses `yield` and turn the "do something" into a block.

Example instead of doing this:
```ruby
db = Connection.connect
db.query("...")
db.disconnect
```

Do this:
```ruby
def with_database
  db = Connection.connect
  yield(db)
ensure
  db.disconnect
end

# Usage:
with_database { |db| db.query("...") }
```

Think of yield as a way to express:
"I know how to do something, but I let you (who called the method) implement the details"

This concept is so simple and powerful that Ruby itself uses it exhaustively, like in `File.open { ... }` where you open a file, hand it to you, and guarantee it closes afterward.

## Blocks are Ruby's implementation of closures
In languages that don't support closures natively or simply (like pure C), a function is like a worker with amnesia. It receives orders (parameters), executes, and leaves. If it needs information that wasn't passed in the "contract" (parameters), it simply can't access it, unless the information is **Global** (which is dangerous).

The fundamental problem that closures solve is **Global Context Preservation without Global Pollution**.

### The Mute Scope Problem
Without closures, to carry a piece of data from point A to point B within complex logic, you would have to:

1. Pass the data through all intermediate methods (even if they don't use it).
2. Or create a global variable (that everyone can break).

The closure resolves this by "freezing" the environment around the code. It's not just a list of instructions; it's an **object that carries its place of origin**.

### What is yield?
yield is a direct instruction to the VM that creates a pointer to the context where the block was called. In other words: in Ruby, yield looks at a pointer called EP (Environment Pointer). This pointer tells the VM: "Hey, if the block code asks for variable x and it's not here inside, look at this memory address here, which was where the block was born."

That's why this works:
```ruby
name = "Coffee"
3.times do
  puts "time for #{name}"
end
```

### What is yield for?

| **Function**                    | **What it does**                                                                                          | **Practical Example**                  |
| ------------------------------- | ------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| **Encapsulation**               | Allows variables to live beyond the end of a scope execution, but stay protected from external access. | Create private counters without using classes. |
| **Lazy Evaluation**            | You define _what_ to do now, but decide _when_ to run later, keeping access to original data. | Event callbacks or HTTP requests. |
| **Behavior Injection**         | The method defines the infrastructure (`start_time`, `end_time`), and the closure injects the intelligence. | benchmarks/execution time measurement |

Note: Not everyone who uses your method will send a block. To prevent your code from "exploding" with a `LocalJumpError`, Ruby offers the method `block_given?`. It's the doorman that checks if there's a "strategy" before trying to yield control.

Example:
```ruby
def execute_command
  return puts "Warning: No command was sent for execution." unless block_given?
  
  puts "Starting system..."
  yield
  puts "System finished."
end

# Safe usage (without block):
execute_command 
# Output: Warning: No command was sent for execution.

# Standard usage (with block):
execute_command { puts "Executing: Cache Cleanup" }
# Output:
# Starting system...
# Executing: Cache Cleanup
# System finished.
```

### Yield is a simplified design pattern
Yield is the Strategy design pattern implemented natively in the language.
That's right, to do what yield does in Java (before Java 8), you would need to create an interface, a class that implements that interface, and instantiate an object.
Ruby resolves this very elegantly by bringing within the language the ability for a method to yield control to let the caller define whatever details (strategies) they want.

### Yield is a direct transfer of flow within the VM

Contrary to what many think, `yield` doesn't create an object in memory to work. While using parameters like `&block` transforms the block into an instance of the `Proc` class (which costs memory and processing), `yield` is a "jump" instruction directly to **YARV** (Ruby's Virtual Machine).

This means:

1. **Surgical Performance:** Ruby just pauses the current method and jumps to the block's memory address. No heavy object creation.
    
2. **Scope Efficiency:** Thanks to the **EP (Environment Pointer)**, the block accesses outside variables without Ruby needing to copy data. It just points to where they already are.


## Conclusion
`yield` is Ruby's secret to elegance. It transforms methods into **platforms**, where you provide the structure and leave the blank space for the user to fill.

Understanding that it's a **Closure** (code + context), you stop fighting the language and start writing code that is both flexible and protected. The next time you see a repetitive pattern (open/close, start/stop), don't create a rigid method: **yield control**.