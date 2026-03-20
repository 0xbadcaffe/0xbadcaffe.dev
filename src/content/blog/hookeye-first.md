---
title: "how an old ELF hooking exercise turned into HookEye"
description: "An interview question sent me down a rabbit hole I never fully climbed out of. This is how GOT/PLT redirection stopped feeling like magic and started feeling inevitable."
pubDate: 2024-05-01
tags: ["elf", "linux", "reverse-engineering", "dynamic-linking"]
---

A long time ago I did an interview exercise that sent me down a rabbit hole I ended up enjoying much more than the exercise itself.

The assignment started with a simple enough question: understand ELF internals, understand common Linux hooking techniques, and figure out how to reason about them in a real process. What stayed with me was not the task itself, but the mental shift that happened once I stopped thinking about hooking as some vague reverse-engineering magic and started seeing it as a consequence of how dynamic linking already works.

That shift eventually turned into **HookEye** — a small tool I built to inspect a live Linux process and answer one very concrete question:

> for each imported function, where does this process actually jump right now?

That question sounds narrow, but it captures a lot.

## The part that clicked for me

When I first learned about ELF, the GOT and PLT felt like one of those topics everyone repeats but few people really internalize. You memorize the acronyms, read that one is a table and the other is a set of stubs, and move on.

What changed for me was realizing that imported function calls in a dynamically linked binary already go through an indirection path by design. That means the mechanism a hook abuses is often the exact same mechanism the runtime needs in order to work normally.

Once I saw that, hooking stopped feeling exotic.

It became much more mechanical:

- the binary does not always jump directly to the final function body
- the dynamic linker participates in resolution
- the PLT and GOT sit in the middle of that path
- if something changes the destination along that path, the process will execute somewhere else

That sounds obvious in hindsight, but for me it was the key unlock.

## GOT and PLT, in the way I now think about them

I do not think of the GOT and PLT as isolated ELF trivia anymore. I think of them as part of the runtime control path of imported calls.

The **GOT** exists because position-independent code cannot just bake in final absolute addresses for everything. It needs a level of indirection. So the runtime stores resolved addresses in data, and code can load or jump through that data.

The **PLT** exists because external function calls need a stable call target in the code section even before the real destination has been fully resolved. So the PLT gives the binary a place to call, and that eventually funnels execution through the right GOT slot.

That means an imported call is not just "call `printf`."

It is closer to:

- call a PLT stub
- let that PLT stub use a GOT-backed destination
- land at whatever address the runtime currently considers correct

And that last part — *whatever address the runtime currently considers correct* — is exactly why this becomes interesting when hooks enter the picture.

## Why `LD_PRELOAD` suddenly made sense

Before digging into this, `LD_PRELOAD` felt like a neat Linux trick.

After digging into it, it felt almost inevitable.

If the dynamic linker is already responsible for resolving symbols at runtime, and if symbol resolution order can be influenced, then of course a preloaded shared object can win that race for certain symbols.

So `LD_PRELOAD` stopped looking like a hack to me and started looking like a very natural extension of the dynamic linking model.

You are not really "breaking" the loader. You are taking advantage of the fact that the loader was always the one deciding where imported symbols resolve.

That also made me appreciate why `LD_PRELOAD` is such a popular technique. It is simple, early, and effective. No binary patching on disk, no rebuilding the target, no need to understand every instruction in the program. If your goal is to intercept library calls, the loader already gives you a front door.

## Then I got curious about the next question

Once I understood `LD_PRELOAD`, the next thing I wanted to know was: what about cases where symbol interposition is not the whole story?

That led me into trampoline hooks and PLT/GOT redirection.

Conceptually, trampoline hooks felt different from `LD_PRELOAD`. Instead of just winning symbol resolution, they wrap or redirect control flow. Execution reaches some shim first, that shim does its own work, and then maybe it jumps to the original destination, maybe it modifies the result, or maybe it diverts somewhere else entirely.

What mattered to me was not just the implementation detail, but the broader realization:

there are multiple ways to alter where an imported call ends up, but from the outside, the most useful thing to inspect is still the same — the *effective destination*.

That became the center of the whole project.

## The question that turned into HookEye

I kept coming back to the same practical thought:

If I attach to a live process and look at the imported-function machinery it is using right now, can I tell whether a call path still lands somewhere expected?

Not in theory.

Not based on what the binary looked like on disk.

Not based on assumptions.

But right now, in memory, in this process.

That was the seed of HookEye.

I did not want a tool that guessed. I wanted one that rebuilt enough of the process's dynamic-link state to say:

- this slot belongs to this imported symbol
- this slot currently points here
- this address belongs to this mapping or module
- and that destination is either expected, strange, unresolved, or suspicious

That felt much more grounded than generic "hook detection" claims.

## Building the tool forced me to respect ELF a lot more

In the abstract, it sounds straightforward.

Read the ELF data.
Find the GOT and PLT.
Walk the relocation entries.
Print the destinations.

In practice, the hard part was not learning what those structures are called. The hard part was reconstructing them correctly from a **live process image**.

That meant I had to stop thinking like someone reading a file on disk and start thinking like someone reading a running program through its mapped memory.

That shift affected everything.

### `/proc/<pid>/maps` became the starting point

I needed a reliable view of what the process had actually mapped, where those mappings lived, and which module owned a given address.

That is why HookEye begins with `/proc/<pid>/maps` parsing. Before I could say whether a destination was suspicious, I needed a solid answer to the simpler question: *what mapping owns this address?*

That sounds basic, but it is one of the things that turns a raw pointer into an interpretable result.

### Remote memory reads changed the project from static parsing into runtime inspection

Reading ELF structures from another process is where the idea stopped being just a paper exercise.

HookEye had to safely inspect a live target, which meant attach/detach logic, remote reads, and a disciplined approach to reconstructing state from memory instead of trusting whatever the file on disk might suggest.

That part made the tool feel real.

### The dynamic section turned out to be the real center of gravity

Once I got comfortable walking `PT_DYNAMIC`, the rest of the design started to settle into place.

This is where the process tells you where its runtime symbol table is, where the dynamic string table is, where the PLT relocation records are, what relocation flavor is in use, and where the PLT-backed GOT region lives.

That was the moment the project became less "scan for suspicious things" and more "rebuild the dynamic linker's view of imported calls."

And I liked that framing much more.

## One of the biggest lessons was load bias

If I had to point to one detail that made the tool much better, it would be this: taking load bias computation seriously.

A lot of rough ELF tooling makes naive assumptions about where an image begins in memory. That works right up until it does not.

Shared objects move. PIE executables move. Runtime addresses are not just file offsets with a wish attached.

I had to compute the image layout from the program headers and process mappings carefully enough that addresses into `PT_DYNAMIC`, `.dynsym`, `.dynstr`, and `DT_JMPREL` all lined up correctly.

That sounds like a small implementation detail, but it is the difference between a tool that looks convincing and a tool that is actually trustworthy.

If your load-bias math is off, then everything downstream becomes fiction.

## Another lesson: the symbol table is easy until it is not

I also learned that dynamic symbol table sizing is one of those problems that seems solved until you try to make a parser that works across real binaries.

In the happy path, you get exactly the metadata you want.
In the annoying path, you do not.

So HookEye ended up using a few strategies, depending on what the target exposes, instead of relying on one ideal case. That was one of those boring-but-important parts of the tool: not flashy, but essential if I wanted the results to hold up outside a toy binary.

## What HookEye ended up becoming

By the time I was done, HookEye had become a compact live inspector with a pretty simple philosophy.

It does not try to be magical.
It does not try to "prove malware."
It does not pretend every unusual destination is malicious.

It just reconstructs the imported-call resolution path and tells me what it sees.

That means it can inspect a live process, walk its ELF dynamic linking state, enumerate PLT relocation slots, read their current runtime destinations, and map those destinations back to owning modules or mappings. From there, I can reason about whether a call still lands in the library I would expect or whether it has been redirected somewhere more interesting.

The repo is on [GitHub](https://github.com/0xbadcaffe/hookeye).

That style of output is exactly what I wanted when I started. Not a verdict, but visibility.

## I also stopped thinking about hooks as one thing

Another thing this project changed for me is that I no longer think about "hooking" as a single category.

There is symbol interposition.
There is GOT rewriting.
There is PLT-oriented redirection.
There are trampoline-style wrappers.
There are hybrids that blur the line.

From an implementation perspective, those differences matter.
But from an inspection perspective, the most useful common denominator is still:

**where does the imported call resolve right now?**

That question gives me a stable anchor even when the specific mechanism varies.

## Why I still like this project

I wrote the original research a long time ago, but I still like the path it pushed me onto.

It started with a topic that can feel annoyingly academic when you first meet it. GOT, PLT, relocation tables, dynamic sections — it is easy to treat them like glossary items.

But once I had to build something against them, those pieces became concrete.
They became observable.
They became useful.

And that, for me, is when systems topics really become fun.

HookEye is not huge, and it is not trying to be. What I like about it is that it grew out of a simple curiosity and stayed focused on one question all the way through.

Not:

> what should this binary do on paper?

But:

> what does this live process actually do at the imported-call boundary right now?

That is still the question I care about most.

## Closing

If there is one thing this project gave me, it is a much more intuitive way of thinking about ELF hooking.

The GOT and PLT are not just linker implementation details sitting off to the side. They are part of the actual execution path. And once that really sinks in, hook detection starts feeling less like dark art and more like careful runtime inspection.

That is what HookEye became for me: a way to turn that mental model into something I could point at, run, and verify.
