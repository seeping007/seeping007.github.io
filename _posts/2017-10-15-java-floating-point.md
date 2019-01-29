---
layout: post
title: Java Floating Point
description: Java floating point tips
category: blog
---

## 1. Floating point values equality test

> Floating point numbers are a just approximation, they are imprecise.

- Sample 1

```java
float myNumber = 3.146;
// Noncompliant. Because of floating point imprecision, this will be false
if (myNumber == 3.146f) {}
// Noncompliant. Because of floating point imprecision, this will be true
if (myNumber != 3.146f) {}
// Noncompliant. Indirect inequality test
if (myNumber < 4 || myNumber > 4) {}

float zeroFloat = 0.0f;
// Noncompliant. Computations may end up with a value close but not equal to zero.
if (zeroFloat == 0) {}
```

- Sample 2

```java
// This will cause an infinite loop!
// Because 0.1 is an infinite binary decimal, the balance will never be exactly 0.
for (double balance = 10; balance != 0; balance -= 0.1) {
    System.out.println(balance);
}

// This will end the loop, but don't expect it to print as our expectation.
for (double balance = 10; balance > 0; balance -= 0.1) {
    System.out.println(balance);
}
```

> Equality tests should not be made with floating point values. (never use == and != for float/double)

```java
// bad!
if(a == b) {}

// solution 1: look not for equality but for whether the value is close enough
if(Math.abs(a - b) < 1e-7) {}

// solution 2: BigDecimal
```

> Do all calculation in float/double but for comparison, always compare approximation instead of precise values. Alternatively, you can use **BigDecimal** for exact calculation.