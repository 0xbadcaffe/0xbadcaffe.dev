---
title: "building hookeye — crawling through /proc to catch liars"
description: "I needed to know if a function had been hooked. One weekend turned into six weeks inside ptrace and the ELF spec."
pubDate: 2024-06-15
tags: ["c", "linux", "elf", "ptrace", "reverse-engineering"]
---

I didn't plan to spend six weeks on this.

The original problem was annoying but simple: I was testing a userspace hooking framework and I needed a way to verify that the hooks were actually in place. Not "did the library load" — I needed to see where a given imported function was *actually* jumping at runtime. Whether the GOT entry for `open` was pointing at glibc or at something I put there.

I looked for an existing tool. `readelf` works on files, not live processes. `gdb` can do it but it's slow and interactive. There were a couple of Python scripts on Stack Overflow that half-worked until they didn't. None of them handled stripped binaries or modern RELRO setups consistently.

Fine. I'll write it myself. How hard could it be.

## what it does

hookeye attaches to a running process, reads its memory, and for each entry in the GOT/PLT tells you exactly where execution would go if you called that function right now. Normal resolution: `open` points somewhere inside `libc-2.x.so`. Hooked: it points into an anonymous mapping, or `libsneaky.so`, or somewhere else you didn't expect.

That's the whole thing. "Where does this function actually jump to?"

## the part where I learned about load bias

The first version read `/proc/<pid>/maps` to get the memory layout, found the ELF header, found the PT_DYNAMIC segment, walked the dynamic entries, found JMPREL and SYMTAB and STRTAB. Straightforward. The ELF spec is actually well-written once you accept that it's 300 pages long.

Then it worked on exactly one binary and segfaulted on everything else.

The issue is position-independent executables. When a shared library loads at `0x7f...` instead of its preferred base address, every address in the ELF structures is still relative to the *preferred* base. You need to compute the load bias — the difference between where it actually ended up and where it thought it would be — and apply it to every pointer you dereference.

This sounds obvious. It is obvious, in retrospect. At the time I spent about half a day staring at hex dumps wondering why my symbol addresses were consistently off by exactly `0x7f3a40000000` or whatever the ASLR offset happened to be that run.

You get the load bias from the program headers: find the PT_LOAD segment with the lowest `p_vaddr`, compare it against the mapped start address from `/proc/maps`. That's it. One subtraction.

## ptrace is weirder than I expected

Reading remote process memory with `ptrace(PTRACE_PEEKDATA, ...)` reads one word at a time. One 8-byte word per syscall. If you need to read a 200-byte string you're doing 25 syscalls.

There's `process_vm_readv` which does bulk reads, but it requires the same permissions as ptrace and isn't always available depending on kernel config and security policies. hookeye uses `process_vm_readv` with a fallback to `PEEKDATA` loops. The fallback path is genuinely painful to look at.

The attach/detach dance also matters if you care about not disturbing the process. `PTRACE_ATTACH` sends SIGSTOP. You need to `waitpid` before reading. Then `PTRACE_DETACH` with SIGCONT to let it resume. If you crash or bail out mid-attach, the process stays stopped. I have a "how do I unstop a process" incident I'd rather not repeat.

## the part that actually works

After all that, the GOT walking itself is straightforward. For each relocation entry in JMPREL, you have a symbol index and an offset into the GOT. Look up the symbol name via SYMTAB + DYNSTR. Read the 8 bytes at `(GOT base + load bias + reloc offset)`. That's the current jump target.

Then you resolve that address back against the memory maps — which library owns that address range, or is it something anonymous. "Anonymous executable mapping" is the interesting answer. That's either JIT code or something was injected.

The output looks like:

```
open          → libc-2.38.so      [resolved]
malloc        → libc-2.38.so      [resolved]
SSL_read      → libssl.so.3       [resolved]
fwrite        → /tmp/.hook.so     [!!! unexpected]
```

That last line is why the tool exists.

## what I'd do differently

The ELF parsing should be a proper library, not hand-rolled struct casts with architecture ifdefs. I was lazy and it's x86-64 only because of that. There are enough edge cases in the symbol table (GNU hash vs. SysV hash, stripped binaries with no SYMTAB, etc.) that anyone who tries to extend it will have a bad time.

Also the error messages are terrible. It just says `failed` in several places where it should tell you why.

The code is on [GitHub](https://github.com/0xbadcaffe/hookeye). It works, it's useful, and some of it is embarrassing. That's probably true of most tools that were written to solve an actual problem rather than to be clean.
