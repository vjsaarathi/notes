---
tags:
  - dsa
  - interview
  - bit-manipulation
---
# Bitwise Operators

---

## AND (&)

```
1101
1011
----
1001
```

Meaning

Keeps only bits present in both operands.

Used For

- Checking if a bit exists
- Creating masks
- Modulo power of two

Interview Problems

- Check permissions
- Check nth bit
- Count bits
- Power of Two

---

## OR (|)

```
1000
0010
----
1010
```

Meaning

Turns bits on.

Used For

- Setting flags
- Combining permissions
- Building bitsets

Interview Problems

- Design a permission system
- Build a visited bitset

---

## XOR (^)

```
1100
1010
----
0110
```

Meaning

Same bits disappear.

Different bits remain.

Properties

```
x ^ x = 0

x ^ 0 = x
```

Interview Problems

- Single Number
- Missing Number
- Find Odd Occurrence
- Swap numbers

---

## NOT (~)

Flips every bit.

Used For

- Clearing bits
- Masks

Interview Problems

- Clear nth bit
