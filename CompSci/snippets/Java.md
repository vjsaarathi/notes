---
tags:
  - dsa
  - interview
  - bit-manipulation
---

# Java Bit Manipulation

Check bit

```java
(x & (1 << n)) != 0
```

Set bit

```java
x |= 1 << n;
```

Clear bit

```java
x &= ~(1 << n);
```

Toggle

```java
x ^= 1 << n;
```

Power of Two

```java
(x & (x-1)) == 0
```

Count Bits

```java
Integer.bitCount(x)
```
