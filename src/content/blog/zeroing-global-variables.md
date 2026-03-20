---
title: "why is there a zeroing loop before main()?"
description: "One of my favorite embedded moments — opening a fresh disassembly and the first loop you see isn't yours."
pubDate: 2022-07-01
tags: ["c", "embedded", "elf", "linker", "bare-metal"]
---

One of my favorite embedded moments is opening a fresh disassembly, ready to inspect *my* code, and the first real loop I see is not mine at all.

It is usually something like this:

```asm
ldr   r1, =__bss_start__
ldr   r2, =__bss_end__
movs  r0, #0
.Lzero:
cmp   r1, r2
bcc   .Lstore
b     .Ldone
.Lstore:
str   r0, [r1], #4
b     .Lzero
.Ldone:
```

That loop is there to make the C language rules true **before** `main()` starts.

This is often explained as "the compiler adds a loop to zero globals." That is *almost* right, but the real story is a little lower-level and more interesting:

- C requires certain objects to begin as zero
- the linker groups those objects into `.bss`
- startup code or the loader clears that memory range
- only then do we get a valid C program state

## The tiny C example

Here is the smallest example that triggers the whole thing:

```c
int g_uninit;
int g_zero = 0;
int g_nonzero = 7;
char big[4096];

int main(void)
{
    return g_uninit + g_zero + g_nonzero + big[0];
}
```

What C promises here:

```c
g_uninit   == 0
g_zero     == 0
big[i]     == 0 for all i
g_nonzero  == 7
```

What C does **not** promise:

```c
void f(void)
{
    int x;   // garbage / indeterminate
}
```

That distinction is the whole reason the startup loop exists.

## The rule from C

The standard wording lives in the initialization rules for static storage duration objects. In plain English:

- file-scope globals with no initializer start at zero
- `static` locals start at zero too
- arrays/structs get recursively zero-initialized
- pointers become null pointers

That is the language contract. The implementation details are up to the toolchain and environment.

## Why `.bss` exists at all

If the toolchain stored all those zero bytes literally in the executable, it would waste space.

So the usual layout is:

```text
.data   = bytes that must be copied from ROM/flash/file image
.bss    = bytes that only need to start as zero
```

In ELF, `.bss` is `SHT_NOBITS`: it occupies memory at runtime, but it does not occupy file bytes the way `.data` does. That is the trick. The file stays smaller, and the runtime pays the cost by clearing RAM at startup.

```c
int a;        // usually .bss
int b = 0;    // usually .bss too
int c = 7;    // usually .data
char buf[4096]; // usually .bss
```

## What I usually see in a real startup path

On bare metal:

```text
reset
  -> set SP
  -> maybe low-level clock/init
  -> copy .data to RAM
  -> zero .bss
  -> call runtime init hooks
  -> call main()
```

A C version of the zeroing step:

```c
extern unsigned int __bss_start__;
extern unsigned int __bss_end__;

static void zero_bss(void)
{
    unsigned int *p = &__bss_start__;
    while (p < &__bss_end__) {
        *p++ = 0;
    }
}
```

The disassembly is usually more recognizable than the C:

```asm
ldr   r1, =__bss_start__
ldr   r2, =__bss_end__
movs  r0, #0
.Lloop:
cmp   r1, r2
bcs   .Ldone
str   r0, [r1], #4
b     .Lloop
.Ldone:
```

Same idea, just no abstraction left.

## What GCC and Clang actually do

I compiled the demo locally and inspected with `nm` / `readelf`.

```bash
gcc -O2 -c demo_zero.c -o demo_gcc.o
nm -S demo_gcc.o
```

```text
0000000000000000 0000000000001000 B big
0000000000000000 0000000000000004 D g_nonzero
0000000000001004 0000000000000004 B g_uninit
0000000000001000 0000000000000004 B g_zero
```

- `B` = `.bss`
- `D` = `.data`

Exactly what you'd expect. Clang gives the same high-level behavior.

## The fun switch: force zeros out of `.bss`

```bash
-fno-zero-initialized-in-bss
```

By default, GCC puts variables explicitly initialized to zero into `.bss` when the target supports it, because it saves space. This flag forces them into `.data` instead.

```bash
gcc   -O2 -fno-zero-initialized-in-bss -c demo_zero.c -o demo_gcc_nobss.o
clang -O2 -fno-zero-initialized-in-bss -c demo_zero.c -o demo_clang_nobss.o
```

With GCC, only the explicitly zero-initialized `g_zero` moves to `.data`. With Clang in my local test, it was more aggressive — moved significantly more into `.data`. That is exactly the kind of thing that makes checking the object file worth the 10 seconds.

## ARM: where people usually *see* the loop

On hosted Linux, the loader can give you zero-filled `.bss` without your program showing an explicit loop in its own entry code.

On Cortex-M with GNU startup code, the loop is much more visible. The startup file loads `_sbss` / `_ebss` (or similarly named linker symbols) and writes zero until the range is done.

So when people say "ARM GCC added a loop before `main()`" — what they usually mean is: "The GNU/CMSIS-style startup object is clearing `.bss` so the C rules are true."

## TI: same idea, more explicit

TI's runtime names the startup routine directly: `_c_int00`. That routine brings the C/C++ environment into a valid initialized state before your code runs. TI also documents `--zero_init`, which controls automatic zero-initialization of uninitialized global/static objects.

```text
reset -> _c_int00 -> initialize runtime state -> zero init -> main
```

Honestly a much clearer mental model than "the compiler added a loop."

## "Compiler adds a loop" vs what really happens

The front-end compiler usually does **not** just decide, out of nowhere, to inject a zeroing loop at the top of `main()`.

What actually happens:

1. the compiler places symbols into `.bss` / `.data`
2. the linker defines the section boundaries
3. startup runtime or loader establishes the initial memory image
4. then your C code starts

That distinction matters when you are debugging startup, writing your own linker script, or trying to keep boot time under control.

## What it costs

On a microcontroller, the cost is paid on every boot.

```text
stores = bss_size_bytes / 4
```

A 64 KiB `.bss` means about 16384 word stores, plus loop overhead. In many systems that is nothing. In some systems it is the first boot-time bug.

I start caring when I have:

- huge global capture buffers
- DSP workspaces
- framebuffers
- fast restart requirements
- early interrupt deadlines after reset

That is when I ask whether the object really belongs in normal `.bss`, or whether it should live in a `NOLOAD`/retained section and be initialized lazily.

## The shortest useful takeaway

When I see a zeroing loop before `main()`:

> The runtime is making the abstract C machine true.

That loop exists because C promises zero-initialized static storage, and `.bss` is the space-saving way to represent that promise. The real reason is the combined behavior of the **language**, **object format**, **linker**, and **startup runtime** — not the compiler doing something arbitrary.

## References

- ISO C draft N1570, initialization rules for static storage duration objects
- ELF `elf(5)` manual, `.bss` and `SHT_NOBITS`
- GCC manual, `-fno-zero-initialized-in-bss`
- TI Arm Clang Compiler Tools docs for `_c_int00` and system initialization
