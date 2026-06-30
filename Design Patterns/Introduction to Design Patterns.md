---
tags:
  - introduction
  - design-pattern
  - oo-programming
---
# What are design patterns
Solutions already written by some of the advanced and experienced developers while facing and solving similar designing problems

They're great reusable solution to commonly occurring problems. These are best practices, used by experienced developers.

each pattern has 4 essential elements 
1. name
2. the problem it solves
3. solution
4. results/consequences
# Why design patterns
1. Flexibility : codebase is always flexible and extensible because that's what the patterns are designed around most of the time.
2. reusability: getting the pattern right has to be done once, it can be reused for lots of different things within the same codebase
3. shared vocabulary: means LLMs understand what you're talking about and can write industry grade code without u having to give 2 fucks about it.
4. capture best practices: the solutions come from boomers which means that even though they're old they're useful to a newbie like me, I don't have to go through the trauma they've been through, so this is more like "watch and learn" moment

## How to select
most of the patterns sound, look and act the same, you have to understand each pattern deeply enough to find the difference so that you may not write shit code or ask an LLM to do the same.

## Categories
1. Creational -- deals with how to create objects (mothers basically)
2. Structural -- Deals with how to structure ur codebase so that it's easy to read and work with in the future, both to urself and others
3. Behavior -- define how the objects behave with each other most likely with responsibilities and how they take them etc.

## The holy grail of patterns 
| Creational patterns  | structural patterns | behavioural patterns        |
| -------------------- | ------------------- | --------------------------- |
| [[Abstract Factory]] | [[Adapter pattern]]         | [[Chain of Responsibility]] |
| [[Builder]]          | [[Bridge]]          | [[Command]]                 |
| [[Factory Method]]   | [[Composite Pattern]]       | [[Interpreter]]             |
| [[Prototype]]        | [[Decorator]]       | [[Iterator]]                |
| [[Singleton]]        | [[Facade pattern]]          | [[Mediator]]                |
|                      | [[Flyweight]]       | [[Memento]]                 |
|                      | [[Proxy]]           | [[Observer]]                |
|                      |                     | [[State]]                   |
|                      |                     | [[Strategy]]                |
|                      |                     | [[Template Method]]         |
|                      |                     | [[Visitor]]                 |
