---
tags:
  - design-pattern
  - oo-programming
  - java
---
# Facade Pattern

## Intent

Provide a **unified, simplified interface** to a set of interfaces in a subsystem.

The Facade pattern hides the complexity of a subsystem and exposes a single entry point for clients.

---

## Problem

A subsystem may contain many classes that must be used in a specific order.

Without a facade, clients must understand:

- Which classes exist
- How they interact
- The correct sequence of operations
- Internal implementation details

This leads to:

- Tight coupling
- Complex client code
- Poor maintainability

---

## Solution

Introduce a **Facade** class that encapsulates the interaction with the subsystem.

```mermaid
graph LR
    Client --> Facade

    Facade --> SubsystemA
    Facade --> SubsystemB
    Facade --> SubsystemC
```

Clients communicate only with the facade.

---

## Structure

```mermaid
classDiagram

class Client

class Facade {
    +operation()
}

class SubsystemA {
    +methodA()
}

class SubsystemB {
    +methodB()
}

class SubsystemC {
    +methodC()
}

Client --> Facade
Facade --> SubsystemA
Facade --> SubsystemB
Facade --> SubsystemC
```

---

## Example

### Without Facade

```java
VideoFile file = new VideoFile("movie.mp4");

Codec sourceCodec = CodecFactory.extract(file);

Buffer buffer =
    BitrateReader.read(file, sourceCodec);

File result =
    BitrateReader.convert(buffer, destinationCodec);

AudioMixer mixer = new AudioMixer();
mixer.fix(result);
```

The client must coordinate every subsystem component.

### With Facade

```java
VideoConverter converter = new VideoConverter();

File result =
    converter.convert("movie.ogg", "mp4");
```

The facade handles all complexity internally.

---

## Sequence of Calls

### Without Facade

```mermaid
sequenceDiagram

participant Client
participant CodecFactory
participant BitrateReader
participant AudioMixer

Client->>CodecFactory: extract()
CodecFactory-->>Client: Codec

Client->>BitrateReader: read()
BitrateReader-->>Client: Buffer

Client->>BitrateReader: convert()
BitrateReader-->>Client: File

Client->>AudioMixer: fix()
AudioMixer-->>Client: Result
```

### With Facade

```mermaid
sequenceDiagram

participant Client
participant Facade
participant Subsystems

Client->>Facade: convert()
Facade->>Subsystems: Coordinate workflow
Subsystems-->>Facade: Result
Facade-->>Client: Converted File
```

---

## Real-World Analogy

### Hotel Front Desk

A hotel contains many departments:

- Housekeeping
- Billing
- Maintenance
- Room Service

Customers interact only with the front desk.

```mermaid
graph TD

Guest --> FrontDesk

FrontDesk --> Housekeeping
FrontDesk --> Billing
FrontDesk --> Maintenance
FrontDesk --> RoomService
```

The front desk acts as a facade.

---

## Benefits

### Simpler API

Clients interact with a small, focused interface.

```java
paymentService.process(order);
```

instead of multiple subsystem calls.

### Reduced Coupling

```mermaid
graph LR

Client --> Facade
Facade --> Subsystem
```

Instead of:

```mermaid
graph LR

Client --> A
Client --> B
Client --> C
Client --> D
```

### Encapsulation

Internal subsystem changes do not necessarily affect clients.

### Better Layering

```mermaid
graph TD

UI --> Facade
Facade --> BusinessLogic
BusinessLogic --> Database
```

---

## Drawbacks

### God Object Risk

A facade can become too large and accumulate unrelated responsibilities.

```java
ApplicationFacade facade = new ApplicationFacade();
```

### Limited Functionality

The facade often exposes only common use cases.

Advanced clients may still need direct access to subsystem classes.

---

## Facade vs Adapter

| Facade | Adapter |
|----------|----------|
| Simplifies an interface | Converts an interface |
| Hides complexity | Resolves incompatibility |
| Works with an existing subsystem | Makes two interfaces work together |

### Facade

```mermaid
graph LR

SubsystemA --> Facade
SubsystemB --> Facade
SubsystemC --> Facade

Facade --> Client
```

### Adapter

```mermaid
graph LR

LegacyAPI --> Adapter
Adapter --> ExpectedAPI
Client --> ExpectedAPI
```

---

## Facade vs Mediator

### Facade

Coordinates access **from outside** the subsystem.

```mermaid
graph TD

Client --> Facade
Facade --> Subsystem
```

### Mediator

Coordinates communication **between objects**.

```mermaid
graph TD

ObjectA --> Mediator
ObjectB --> Mediator
ObjectC --> Mediator
```

---

## Common Use Cases

### Repository Layer

```mermaid
graph TD

Application --> UserRepository

UserRepository --> JDBC
UserRepository --> ConnectionPool
UserRepository --> SQL
```

### Compiler

```mermaid
graph TD

Client --> Compiler

Compiler --> Lexer
Compiler --> Parser
Compiler --> SemanticAnalyzer
Compiler --> Optimizer
Compiler --> CodeGenerator
```

### Home Theater

```mermaid
graph TD

User --> HomeTheaterFacade

HomeTheaterFacade --> Projector
HomeTheaterFacade --> SoundSystem
HomeTheaterFacade --> DVDPlayer
```

---

## Recognition

You probably need a Facade when:

- A subsystem contains many classes.
- Client code knows too much about implementation details.
- Object creation requires many steps.
- Clients repeatedly perform the same sequence of operations.
- You want a stable API over a changing subsystem.

---

## Rule of Thumb

> Use a Facade when you want to provide a simple entry point to a complex subsystem while reducing coupling between clients and implementation details.

---

## Related Patterns

- [[Adapter pattern]]
- [[Mediator Pattern]]
- [[Abstract Factory Pattern]]
- [[Proxy Pattern]]