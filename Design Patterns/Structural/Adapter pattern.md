---
tags:
  - design-pattern
  - oo-programming
  - java
---
# Adapter Pattern

## Intent

Convert the interface of a class into another interface that clients expect.

The Adapter pattern allows otherwise incompatible classes to work together without modifying their source code.

---

## Problem

You have an existing class whose functionality you want to reuse, but its interface does not match what the client expects.

### Example

Your application expects:

```java
public interface PaymentProcessor {
    void pay(double amount);
}
```

But a third-party library provides:

```java
public class LegacyPaymentGateway {
    public void makePayment(double value) {
        // ...
    }
}
```

The client cannot use `LegacyPaymentGateway` directly because the interfaces are incompatible.

---

## Solution

Create an adapter that implements the expected interface and delegates calls to the existing class.

```mermaid
classDiagram

class Client

class Target {
    <<interface>>
    +request()
}

class Adapter {
    +request()
}

class Adaptee {
    +specificRequest()
}

Client --> Target
Adapter ..|> Target
Adapter --> Adaptee
```

---

## Structure

### Object Adapter (Composition)

Uses composition and delegation.

```mermaid
classDiagram

class Client

class Target {
    <<interface>>
    +request()
}

class Adapter {
    -adaptee: Adaptee
    +request()
}

class Adaptee {
    +specificRequest()
}

Client --> Target
Adapter ..|> Target
Adapter --> Adaptee
```

This is the most common form.

---

## Example

### Existing Interface

```java
public interface Printer {
    void print(String text);
}
```

### Legacy Class

```java
public class LegacyPrinter {

    public void printDocument(String text) {
        System.out.println(text);
    }
}
```

### Adapter

```java
public class PrinterAdapter implements Printer {

    private final LegacyPrinter legacyPrinter;

    public PrinterAdapter(LegacyPrinter legacyPrinter) {
        this.legacyPrinter = legacyPrinter;
    }

    @Override
    public void print(String text) {
        legacyPrinter.printDocument(text);
    }
}
```

### Client

```java
Printer printer =
    new PrinterAdapter(new LegacyPrinter());

printer.print("Hello");
```

---

## Sequence of Calls

```mermaid
sequenceDiagram

participant Client
participant Adapter
participant Adaptee

Client->>Adapter: request()
Adapter->>Adaptee: specificRequest()
Adaptee-->>Adapter: Result
Adapter-->>Client: Result
```

---

## Real-World Analogy

### Travel Adapter Plug

A laptop charger from one country may not fit a wall socket in another country.

The power adapter converts one interface into another.

```mermaid
graph LR

LaptopPlug --> TravelAdapter
TravelAdapter --> WallSocket
```

The charger remains unchanged.
The wall socket remains unchanged.
The adapter bridges the incompatibility.

---

## Benefits

### Reuse Existing Code

You can integrate legacy or third-party components without modifying them.

### Open/Closed Principle

New adapters can be introduced without changing existing code.

### Decoupling

Clients depend on the target interface rather than concrete implementations.

### Incremental Migration

Useful when replacing old systems gradually.

---

## Drawbacks

### Additional Indirection

Adds another layer to the design.

### Increased Complexity

Too many adapters can make the system harder to understand.

### Potential Performance Cost

Usually negligible, but every call passes through another object.

---

## Types of Adapters

### Object Adapter (Preferred)

Uses composition.

```mermaid
graph TD

Client --> Adapter
Adapter --> Adaptee
```

Advantages:

- Flexible
- Works with final classes
- Follows composition over inheritance

---

### Class Adapter

Uses inheritance.

```mermaid
classDiagram

class Target

class Adaptee

class Adapter

Target <|-- Adapter
Adaptee <|-- Adapter
```

Advantages:

- Slightly simpler

Disadvantages:

- Requires multiple inheritance (or interface inheritance)
- Less flexible

---

## Adapter vs Facade

### Adapter

Changes an interface.

```mermaid
graph LR

LegacyAPI --> Adapter
Adapter --> ExpectedAPI
Client --> ExpectedAPI
```

Purpose:

> Make incompatible interfaces work together.

---

### Facade

Simplifies an interface.

```mermaid
graph LR

SubsystemA --> Facade
SubsystemB --> Facade
SubsystemC --> Facade

Facade --> Client
```

Purpose:

> Hide complexity behind a simpler interface.

---

## Adapter vs Decorator

### Adapter

Changes the interface.

```mermaid
graph LR

Client --> Adapter
Adapter --> Service
```

---

### Decorator

Keeps the same interface and adds behavior.

```mermaid
graph LR

Client --> Decorator
Decorator --> Service
```

---

## Adapter vs Proxy

### Adapter

Makes interfaces compatible.

```mermaid
graph LR

Client --> Adapter
Adapter --> Adaptee
```

---

### Proxy

Controls access to an object.

```mermaid
graph LR

Client --> Proxy
Proxy --> RealObject
```

---

## Common Use Cases

### Legacy System Integration

```mermaid
graph TD

ModernApplication --> Adapter
Adapter --> LegacySystem
```

---

### Third-Party Libraries

```mermaid
graph TD

Application --> Adapter
Adapter --> ExternalSDK
```

---

### Database Drivers

JDBC acts as an adapter layer.

```mermaid
graph TD

Application --> JDBC
JDBC --> MySQL
JDBC --> PostgreSQL
JDBC --> Oracle
```

Applications use a common interface while drivers adapt vendor-specific implementations.

---

### Logging Frameworks

```mermaid
graph TD

Application --> LoggingAdapter
LoggingAdapter --> Log4j
LoggingAdapter --> SLF4J
LoggingAdapter --> JUL
```

---

## Recognition

You probably need an Adapter when:

- You want to use an existing class but its interface doesn't match.
- You are integrating a third-party library.
- You are migrating from a legacy system.
- You want to isolate vendor-specific APIs.
- You cannot modify the existing class.

---

## Rule of Thumb

> Use an Adapter when you need to make two incompatible interfaces work together without changing either side.

Think of it as a translator sitting between two parties that speak different languages.

---

## Related Patterns

- [[Facade pattern]]
- [[Decorator Pattern]]
- [[Proxy Pattern]]
- [[Bridge Pattern]]