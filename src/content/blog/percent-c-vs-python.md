---
title: "% in C vs Python: same symbol, different behavior"
description: "It looks identical. It bites you the same way every time. A note on modulo, remainders, and the one row in the table that breaks your circular buffer."
pubDate: 2026-02-20
tags: ["c", "python", "embedded", "bugs"]
---

I was porting some ring-buffer indexing code from Python to C. Positive test cases passed. Negative offsets quietly produced wrong results.

The operator was `%`. The bug was one character wide and completely invisible until runtime.

I've seen this enough times that it's worth writing down.

## The short version

For positive numbers, both languages give the same result:

```c
5 % 3   // 2
```

```python
5 % 3   # 2
```

But with negative numbers:

- **C** keeps the remainder with the sign of the **dividend** (left operand).
- **Python** gives a remainder that follows the **divisor** — staying consistent with floor division.

| Expression | C | Python |
|---|---:|---:|
| `5 % 3` | `2` | `2` |
| `-5 % 3` | `-2` | `1` |
| `5 % -3` | `2` | `-1` |
| `-5 % -3` | `-2` | `-2` |

That `-5 % 3` row is where ports go wrong.

## How C thinks about `%`

Integer division in C truncates toward zero. The remainder is whatever is left after that.

The standard is direct: "the result of the / operator is the algebraic quotient with any fractional part discarded." The `%` operator is defined to satisfy:

```text
a == (a / b) * b + (a % b)
```

So in C:

```c
-5 / 3 == -1
-5 % 3 == -2
```

because `(-1 * 3) + (-2) = -5`. The math checks out. The sign follows the dividend.

One historical note worth knowing: before C99, signed integer division with negative operands was actually implementation-defined. WG14 standardized truncation toward zero in C99 because it was already what most hardware did and made numeric porting easier. If you're reading old C code, keep that in mind.

```c
#include <stdio.h>

static int positive_mod(int a, int n) {
    return ((a % n) + n) % n;
}

int main(void) {
    printf("5 %% 3 = %d\n", 5 % 3);
    printf("-5 %% 3 = %d\n", -5 % 3);
    printf("5 %% -3 = %d\n", 5 % -3);
    printf("-5 %% -3 = %d\n", -5 % -3);

    puts("");
    puts("wrap-to-[0, n-1] using positive_mod(i, 4):");
    for (int i = -5; i <= 5; ++i)
        printf("%d -> %d\n", i, positive_mod(i, 4));

    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -pedantic c_mod_examples.c -o c_mod_examples
./c_mod_examples
```

## How Python thinks about `%`

Python ties `%` to **floor division**, not truncation toward zero.

The result always has the same sign as the divisor, and `%` stays paired cleanly with `//`. This was a deliberate design from PEP 238 — Python wanted its division semantics to be mathematically consistent, so `//` became the unambiguous floor-division operator and `%` was defined to match.

```text
a == (a // b) * b + (a % b)
```

So in Python:

```python
-5 // 3 == -2
-5 % 3 == 1
```

because `(-2 * 3) + 1 = -5`. Different truncation direction, different remainder.

```python
print(f"5 % 3 = {5 % 3}")
print(f"-5 % 3 = {-5 % 3}")
print(f"5 % -3 = {5 % -3}")
print(f"-5 % -3 = {-5 % -3}")

print()
print("wrap-to-[0, n-1] using i % 4:")
for i in range(-5, 6):
    print(f"{i} -> {i % 4}")
```

## Where it actually breaks things

### Circular index / ring buffer

Python:

```python
def wrap_index(x, size):
    return x % size  # always lands in [0, size-1]
```

C — naive port that silently breaks for negative `x`:

```c
int wrap_index(int x, int size) {
    return x % size;  // can return negative
}
```

C — correct version:

```c
int wrap_index(int x, int size) {
    return ((x % size) + size) % size;
}
```

This is the pattern I see broken most often. Someone ports the Python version directly, all positive tests pass, a negative offset hits production, and suddenly the circular buffer is indexing out of bounds.

### Bucket / hash assignment

```python
bucket = key % bucket_count  # safe if bucket_count > 0
```

Same issue in C — if `key` is negative, you might get a negative bucket index and corrupt your array access.

### Even/odd checks

```python
x % 2 == 0
```

This one is actually fine to port. `-4 % 2 == 0` in both languages. Parity checks are safe — it's when you care about the exact signed remainder that things diverge.

## The mental model

When you see `%`, don't just think "modulo." Think:

- **C**: remainder after truncating toward zero. Sign follows the dividend.
- **Python**: remainder that makes floor division work. Sign follows the divisor.

That one sentence difference explains the whole table.

## Practical checklist for porting

1. Test negative inputs explicitly — don't rely on positive cases passing.
2. Any wraparound or circular indexing logic needs a second look.
3. In C, normalize with `((x % n) + n) % n` when you need a result in `[0, n-1]`.
4. Don't copy Python `%` expressions into C unchanged.

## Sources

- Python language reference: expression semantics for `%` and `//`
- PEP 238: rationale for Python's division model
- C99 standard: multiplicative operators, §6.5.5
- WG14 N617: background on standardizing signed integer division behavior
