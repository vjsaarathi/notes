---
tags:
  - dsa
  - interview
  - bit-manipulation
---

# Common Bit Tricks

---

## Check nth Bit

```cpp
x & (1 << n)
```

Interview

- Bitsets
- Permissions

---

## Set nth Bit

```cpp
x |= (1 << n)
```

Interview

- Flags
- Visited array

---

## Clear nth Bit

```cpp
x &= ~(1 << n)
```

Interview

- Disable feature
- Permission systems

---

## Toggle nth Bit

```cpp
x ^= (1 << n)
```

Interview

- Flip switches
- State machines

---

## Is Power Of Two

```cpp
x & (x-1)
```

Interview

- Power of Two
- Memory allocation

---

## Remove Lowest Set Bit

```cpp
x &= x-1
```

Interview

- Count bits
- Iterate subsets

---

## Lowest Set Bit

```cpp
x & -x
```

Interview

- Fenwick Tree
- Binary Indexed Tree

---

## Count Bits

Brian Kernighan

```cpp
while(x){

x &= x-1;

count++;

}
```

Interview

- Hamming Weight
- Population Count

---

## Lower k Bits

```cpp
(1<<k)-1
```

Interview

- Masks
- Buckets

---

## Bucket + Offset

```cpp
bucket = value >> 5

offset = value & 31
```

Interview

- Bitsets
- Memory allocators
