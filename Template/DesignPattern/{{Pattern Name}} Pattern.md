---
tags:
  - design-pattern
  - oo-programming
  - java
---
# {{Pattern Name}} Pattern
## Intent

> One-sentence description of what problem this pattern solves.

---

## Classification

|Property|Value|
|---|---|
|Category|Creational / Structural / Behavioral|
|Complexity|Low / Medium / High|
|Frequency of Use|Rare / Common / Very Common|
|Introduced In|GoF|

---

## Problem

Describe the situation that motivates the pattern.

### Symptoms

- Symptom 1
    
- Symptom 2
    
- Symptom 3
    

### Example Scenario

```text
Describe the real software problem.
```

---

## Solution

Explain the core idea.

```mermaid
graph TD

Client --> Pattern
Pattern --> ComponentA
Pattern --> ComponentB
Pattern --> ComponentC
```

---

## Structure

### UML Diagram

```mermaid
classDiagram

class Client

class ComponentA
class ComponentB
class ComponentC

Client --> ComponentA
ComponentA --> ComponentB
ComponentB --> ComponentC
```

---

## Participants

### Client

Responsibilities:

- Responsibility 1
    
- Responsibility 2
    

### Participant A

Responsibilities:

- Responsibility 1
    
- Responsibility 2
    

### Participant B

Responsibilities:

- Responsibility 1
    
- Responsibility 2
    

---

## Collaboration

### Runtime Interaction

```mermaid
sequenceDiagram

participant Client
participant A
participant B

Client->>A: Request
A->>B: Delegate
B-->>A: Response
A-->>Client: Result
```

---

## Implementation

### Example

```java
// Example code
```

### Key Points

- Point 1
    
- Point 2
    
- Point 3
    

---

## Object Graph

Show how objects relate at runtime.

```mermaid
graph LR

Client --> ObjectA
ObjectA --> ObjectB
ObjectA --> ObjectC
```

---

## Before Applying Pattern

```mermaid
graph LR

Client --> A
Client --> B
Client --> C
Client --> D
```

Problems:

- Tight coupling
    
- Repeated logic
    
- Difficult maintenance
    

---

## After Applying Pattern

```mermaid
graph LR

Client --> Pattern
Pattern --> A
Pattern --> B
Pattern --> C
Pattern --> D
```

Benefits:

- Lower coupling
    
- Better maintainability
    
- Simpler API
    

---

## Real-World Analogy

### Analogy

Describe the real-world equivalent.

```mermaid
graph TD

Actor --> Pattern

Pattern --> ServiceA
Pattern --> ServiceB
Pattern --> ServiceC
```

---

## Benefits

### 1. Benefit Name

Explanation.

### 2. Benefit Name

Explanation.

### 3. Benefit Name

Explanation.

---

## Drawbacks

### 1. Drawback Name

Explanation.

### 2. Drawback Name

Explanation.

### 3. Drawback Name

Explanation.

---

## Recognition

You probably need this pattern when:

- Condition 1
    
- Condition 2
    
- Condition 3
    
- Condition 4
    

### Code Smells

- Large switch statements
    
- Tight coupling
    
- Excessive object creation
    
- Repeated workflows
    

---

## Common Use Cases

### Use Case 1

```mermaid
graph TD

Application --> Pattern
Pattern --> Service
```

### Use Case 2

```mermaid
graph TD

Application --> Pattern
Pattern --> LegacySystem
```

### Use Case 3

```mermaid
graph TD

Application --> Pattern
Pattern --> Database
```

---

## Trade-Offs

|Advantage|Cost|
|---|---|
|Advantage 1|Cost 1|
|Advantage 2|Cost 2|
|Advantage 3|Cost 3|

---

## Comparison with Related Patterns

### vs Pattern A

|{{Pattern Name}}|Pattern A|
|---|---|
|Purpose|Purpose|
|Structure|Structure|
|Use Case|Use Case|

```mermaid
graph LR

PatternA --> Difference
PatternB --> Difference
```

---

### vs Pattern B

|{{Pattern Name}}|Pattern B|
|---|---|
|Purpose|Purpose|
|Structure|Structure|
|Use Case|Use Case|

---

## Refactoring Recipe

1. Identify the problem.
    
2. Locate the varying behavior.
    
3. Extract abstractions.
    
4. Introduce the pattern.
    
5. Move responsibilities.
    
6. Simplify client code.
    

```mermaid
flowchart TD

A[Existing Code]
--> B[Identify Problem]
--> C[Extract Abstraction]
--> D[Apply Pattern]
--> E[Refactor Client]
```

---

## Performance Considerations

|Aspect|Impact|
|---|---|
|CPU|Low / Medium / High|
|Memory|Low / Medium / High|
|Complexity|Low / Medium / High|

Notes:

- Considerations here.
    

---

## Concurrency Considerations

- Thread safety concerns
    
- Synchronization concerns
    
- Shared state concerns
    

```mermaid
graph TD

Thread1 --> SharedResource
Thread2 --> SharedResource
Thread3 --> SharedResource
```

---

## Testing Strategy

### Unit Tests

- Test scenario 1
    
- Test scenario 2
    

### Integration Tests

- Test scenario 1
    
- Test scenario 2
    

### Edge Cases

- Edge case 1
    
- Edge case 2
    

---

## Interview Notes

### When to Mention

- Scenario 1
    
- Scenario 2
    

### Common Interview Question

> Why would you use {{Pattern Name}} instead of {{Related Pattern}}?

Answer:

- Point 1
    
- Point 2
    
- Point 3
    

---

## Rule of Thumb

> One-sentence mental model of the pattern.

---

## Related Patterns

- [[Pattern A]]
    
- [[Pattern B]]
    
- [[Pattern C]]
    

---

## References

- GoF Design Patterns
    
- Head First Design Patterns
    
- Pattern-Oriented Software Architecture