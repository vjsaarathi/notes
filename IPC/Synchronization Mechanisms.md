---
tags:
  - basics
  - architecture
---
## Definition

Synchronization mechanisms coordinate execution between threads or processes.

Their purpose is to:

- Prevent race conditions
- Protect shared resources
- Coordinate execution order
- Implement waiting and notification semantics

Unlike IPC mechanisms, synchronization primitives generally do **not transfer application data**.

---

# Synchronization Mechanisms Overview

```mermaid
mindmap
root((Synchronization))

    Mutex
    Semaphore
    RWLock
    Spinlock
    Futex
    ConditionVariable
    Barrier
    Monitor
```

---

# Classification

```mermaid
flowchart TD

    Sync[Synchronization]

    Sync --> MutualExclusion[Mutual Exclusion]
    Sync --> Coordination[Coordination]
    Sync --> Waiting[Waiting & Notification]

    MutualExclusion --> Mutex
    MutualExclusion --> Spinlock
    MutualExclusion --> RWLock

    Coordination --> Semaphore
    Coordination --> Barrier

    Waiting --> Futex
    Waiting --> ConditionVariable
```

---

# The Race Condition Problem

Synchronization exists to prevent races.

```mermaid
sequenceDiagram

    participant T1 as Thread 1
    participant Counter
    participant T2 as Thread 2

    T1->>Counter: Read = 10
    T2->>Counter: Read = 10

    T1->>Counter: Write 11
    T2->>Counter: Write 11
```

Expected:

```text
12
```

Actual:

```text
11
```

One increment is lost.

---

# Mutex

## Definition

A mutex (Mutual Exclusion Lock) allows only one thread to enter a critical section at a time.

---

## Architecture

```mermaid
flowchart TD

    Thread1 --> Mutex
    Thread2 --> Mutex
    Thread3 --> Mutex

    Mutex --> CriticalSection
```

---

## Critical Section

```mermaid
flowchart LR

    Lock --> Work[Critical Section]
    Work --> Unlock
```

---

## Characteristics

|Property|Value|
|---|---|
|Ownership|Yes|
|Blocking|Yes|
|Fairness|Implementation Dependent|
|Reentrant|Usually No|

---

## Use Cases

- Protecting shared data structures
- Database connection pools
- Caches
- Reference counters

---

# Spinlock

## Definition

A lock where waiting threads continuously retry instead of sleeping.

---

## Architecture

```mermaid
flowchart TD

    Thread --> LockAttempt

    LockAttempt -->|Busy| Retry

    Retry --> LockAttempt

    LockAttempt -->|Free| CriticalSection
```

---

## Characteristics

|Property|Value|
|---|---|
|Sleeps|No|
|CPU Usage|High|
|Latency|Very Low|
|Kernel Usage|Minimal|

---

## Best For

Short critical sections.

---

## Bad For

Long waits.

---

# Read-Write Lock (RWLock)

## Definition

Allows:

- Multiple readers simultaneously
- One writer exclusively

---

## Architecture

```mermaid
flowchart LR

    Reader1 --> RWLock
    Reader2 --> RWLock
    Reader3 --> RWLock

    Writer --> RWLock
```

---

## Access Rules

```mermaid
flowchart TD

    Access --> Reader

    Access --> Writer

    Reader --> SharedAccess

    Writer --> ExclusiveAccess
```

---

## Best For

Read-heavy workloads.

Examples:

- Configuration stores
    
- In-memory databases
    
- Caches
    

---

# Semaphore

## Definition

A counter-based synchronization primitive.

Controls access to a limited number of resources.

---

## Architecture

```mermaid
flowchart TD

    ResourcePool[Count = N]

    Thread1 --> ResourcePool
    Thread2 --> ResourcePool
    Thread3 --> ResourcePool
    Thread4 --> ResourcePool
```

---

## Types

### Binary Semaphore

```text
0 or 1
```

Similar to a mutex.

---

### Counting Semaphore

```text
0 ... N
```

Controls access to multiple resources.

---

## Example

Database pool:

```text
10 connections available
```

Semaphore count:

```text
10
```

---

# Condition Variable

## Definition

Allows threads to sleep until a condition becomes true.

---

## Architecture

```mermaid
sequenceDiagram

    participant Consumer
    participant Condition
    participant Producer

    Consumer->>Condition: Wait()

    Producer->>Condition: Signal()

    Condition->>Consumer: Wake Up
```

---

## Typical Pattern

```text
while (!condition)
    wait()

proceed()
```

---

## Use Cases

- Producer-consumer systems
    
- Task queues
    
- Job schedulers
    

---

# Futex

## Definition

Fast Userspace Mutex.

Linux synchronization primitive that only enters the kernel when contention occurs.

---

## Architecture

```mermaid
flowchart TD

    Thread1 --> UserSpaceLock

    Thread2 --> UserSpaceLock

    UserSpaceLock -->|Contended| Kernel

    UserSpaceLock -->|Uncontended| Success
```

---

## Why Futex Is Fast

Most lock operations occur entirely in userspace.

Kernel involvement happens only when blocking becomes necessary.

---

## Used By

- pthread mutexes
    
- Rust synchronization primitives
    
- Java synchronization
    
- Go runtime
    

---

# Barrier

## Definition

Forces a group of threads to wait until all participants arrive.

---

## Architecture

```mermaid
flowchart TD

    T1 --> Barrier
    T2 --> Barrier
    T3 --> Barrier
    T4 --> Barrier

    Barrier --> Continue
```

---

## Example

```text
4 threads required

Thread 1 arrives
Thread 2 arrives
Thread 3 arrives

Wait...

Thread 4 arrives

Release all 4
```

---

## Use Cases

- Scientific computing
    
- Parallel algorithms
    
- Simulation systems
    

---

# Monitor

## Definition

A monitor combines:

- Shared data
    
- Mutex
    
- Condition variables
    

into a single abstraction.

---

## Architecture

```mermaid
flowchart TD

    Monitor

    Monitor --> Data
    Monitor --> Mutex
    Monitor --> Conditions
```

---

## Languages Using Monitors

- Java
    
- C#
    
- Kotlin
    

---

# Synchronization Primitive Comparison

|Primitive|Ownership|Multiple Holders|Sleep|Best For|
|---|---|---|---|---|
|Mutex|Yes|No|Yes|General locking|
|Spinlock|Yes|No|No|Very short sections|
|RWLock|Yes|Readers Only|Yes|Read-heavy workloads|
|Semaphore|No|Yes|Yes|Resource limits|
|Condition Variable|No|N/A|Yes|Waiting for events|
|Futex|Depends|No|Yes|High-performance locking|
|Barrier|No|Many|Yes|Parallel phases|

---

# Relationships

```mermaid
flowchart TD

    Futex --> Mutex

    Mutex --> ConditionVariable

    Mutex --> Monitor

    Semaphore --> ResourceControl

    Barrier --> PhaseCoordination

    RWLock --> MutexFamily

    Spinlock --> MutexFamily
```

---

# How Modern Languages Use Them

```mermaid
flowchart TD

    Language[Programming Languages]

    Language --> Java
    Language --> Go
    Language --> Rust
    Language --> Cpp

    Java --> Monitor
    Java --> RWLock

    Rust --> Mutex
    Rust --> RWLock

    Go --> Mutex
    Go --> Cond

    Cpp --> Mutex2[Mutex]
    Cpp --> CV[Condition Variable]
```

---

# Key Takeaways

1. Synchronization primitives coordinate execution rather than transfer data.
    
2. Mutexes are the most common locking primitive.
    
3. Spinlocks trade CPU usage for lower latency.
    
4. RWLocks improve performance in read-heavy workloads.
    
5. Semaphores control access to limited resources.
    
6. Condition variables implement wait/notify patterns.
    
7. Futexes are the foundation of many modern locking implementations on Linux.
    
8. Barriers synchronize phases of parallel execution.
    
9. Monitors combine data protection and coordination into a single abstraction.