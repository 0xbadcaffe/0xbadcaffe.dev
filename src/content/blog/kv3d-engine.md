---
title: "KV3D Engine: rethinking KV cache compression for LLM serving"
pubDate: 2026-03-20
tags: ["llm", "inference", "systems", "kv-cache", "bm3d", "llama.cpp", "ollama"]
---

## An embedded engineer’s first serious dive into LLM serving, KV-cache waste, and a prototype runtime built around shared-prefix state.

## My first real dive into AI

I’ve spent most of my career nowhere near AI.

My comfort zone is kernel space, network drivers, DMA rings, firmware bugs, hardware that lies to you, and systems that only fail when the logs are off. LLMs always felt like someone else’s stack.

Then I started actually using them.

And like any embedded engineer, I couldn’t leave it at “it works.” I wanted to know what was underneath, where the cycles went, where the memory went, and which part of the architecture was getting away with something expensive.

This post is the result of that curiosity.

Not a grand theory. Just the first serious idea that made me stop and think: _there’s redundant state here, and we’re paying for it over and over again_.

## Tokens, briefly

An LLM does not read text the way we do. Before the model sees anything, the input is split into **tokens**: small chunks of text that may be whole words, parts of words, punctuation, or formatting.

Inference then becomes a loop:

1. read a sequence of tokens,
2. compute an internal representation,
3. predict the next token,
4. append it,
5. repeat.

That much is well known. The part that got my attention was what the model keeps around between steps.

## KV cache, briefly

Inside a transformer, attention is expensive.

While processing a prompt, the model computes internal **keys** and **values** for each token at each layer. Those tensors get cached so the model does not need to recompute the entire history every time it generates the next token.

That saved state is the **KV cache**.

The good news is that it saves compute.

The bad news is that it can consume a lot of memory, especially for long contexts or many concurrent sessions. Once I started looking at serving systems through that lens, the architecture stopped feeling mystical and started feeling familiar: hot state, warm state, cold state, and a scheduler trying to decide what deserves fast memory.

## The waste that made this click for me

What really made the idea click was not “AI magic.” It was a very old systems smell: duplicated state hidden behind a convenient abstraction.

In most practical deployments, sessions do **not** begin from scratch in any meaningful sense. Enterprise copilots, support assistants, agent frameworks, and RAG systems usually start from a repeated scaffold:

- the same system prompt,
- the same tool definitions,
- the same response format contract,
- the same orchestration wrapper.

Different users, different questions, but a lot of the prefix is identical.

And yet the serving stack often computes and stores KV state for that shared prefix separately for every session.

That felt wasteful in exactly the way that would get you a sharp comment in a kernel review.

So the first idea behind **KV3D Engine** is simple:

```text
KV_session ≈ KV_shared_prefix + delta_session
```

Compute the shared-prefix state once. Keep it as a reusable snapshot. Store only what is specific to the session.

That is the whole instinct in one line.

## Why I think this is plausible

I am not claiming all KV state is globally compressible, or that arbitrary sessions naturally cluster into something elegant.

I am making a much narrower claim:

> for repeated-prefix workloads, early KV state should be highly correlated across sessions, and that correlation is worth exploiting as a systems primitive.

That is why the first prototype is intentionally conservative:

- exact shared prefix only,
- no fuzzy matching,
- no learned codec,
- no magical transform yet.

First prove that **state deduplication** is useful. Then get more ambitious.

## Where BM3D entered the picture

The BM3D connection came later.

BM3D, short for **Block-Matching and 3D Filtering**, is a classical image denoising method. Its core idea is elegant: do not process each patch in isolation. Find similar patches across the image, stack them together, move to a transform domain, preserve the coherent structure, and suppress the rest.

What mattered to me was not image denoising itself. It was the design principle:

> when the redundancy lives across many similar objects, compressing each object alone leaves performance on the table.

That maps surprisingly well onto KV cache serving.

If many sessions share a prompt family, then KV blocks from those sessions may also share structure. The common prefix-induced part acts like the repeated signal. The session-specific variation becomes the residual.

That does **not** mean KV tensors are “basically images.” It means repeated prompt families may induce repeated activation structure, and that makes grouped compression plausible.

## The BM3D-style mapping

The analogy I have in mind looks like this:

| BM3D concept | KV3D equivalent |
|---|---|
| Similar image patches | KV blocks from sessions with the same prompt family |
| Block matching | Exact-prefix hashing first, near-prefix clustering later |
| 3D transform/grouped basis | DCT, PCA, FFT, or learned basis across grouped KV blocks |
| Keep coherent coefficients | Preserve the shared prototype |
| Store the remainder | Quantize and store only the per-session residual |

The important part is the grouping logic, not loyalty to one specific transform.

So when I say “BM3D-inspired,” I mean I’m borrowing the **non-local grouping principle**, not literally porting an image algorithm into transformer serving.

## What the MVP actually is

The first prototype is intentionally boring in the best possible way.

It is **not** a fancy grouped codec yet.

It is a prefix-aware runtime with a reusable shared snapshot and separate per-session state:

- detect an exact shared prefix,
- build one canonical KV snapshot for that prefix,
- store per-session suffix state separately,
- reconstruct a working KV view before decode,
- fall back to baseline behavior whenever needed.

That gives me a clean baseline to measure:

- memory saved,
- sessions per GPU,
- latency impact,
- output drift.

Only after that baseline works do I earn the right to try grouped transforms and collaborative residual coding.

## Why `llama.cpp` is the right place to prototype this

I do **not** want to rebuild the entire inference stack from scratch.

For a prototype like this, `llama.cpp` is the obvious substrate: it is a widely used C/C++ inference engine designed for local and cloud execution across a wide range of hardware, and it already ships with a library interface, local build path, and even an HTTP server.

That matters because KV3D is not trying to innovate on matmul kernels or token sampling. The novelty lives in:

- KV-cache layout,
- shared-prefix snapshotting,
- session delta storage,
- restore and eviction policy,
- memory tiering,
- scheduler decisions.

So the sensible plan is:

- use `llama.cpp` for model execution,
- add KV3D around the cache manager and runtime layer,
- expose the result through an OpenAI-compatible or Ollama-like API.

Ollama is useful here as a product reference point. It provides a polished local runtime and API surface, and by default serves its API locally at `http://localhost:11434/api`.

That distinction is important:

- **`llama.cpp`** is the inference backend and developer substrate.
- **Ollama** is the product/runtime experience many developers already understand.

For this project, `llama.cpp` is where I want control. Ollama is the UX shape I want to stay compatible with.

Useful references:
- `llama.cpp` repo: [github.com/ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)
- `llama.cpp` build docs: [docs/build.md](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md)
- `llama.cpp` HTTP server: [tools/server/README.md](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- Ollama API intro: [docs.ollama.com/api/introduction](https://docs.ollama.com/api/introduction)
- Ollama quickstart: [docs.ollama.com/quickstart](https://docs.ollama.com/quickstart)

## The runtime architecture

At a high level, the engine looks like this:

```text
request → prefix detector → session manager → llama.cpp backend
                                   ↕
              shared prefix snapshot store + delta state
                                   ↕
                  GPU hot cache / RAM warm cache / SSD cold cache
```

For each incoming request:

1. Canonicalize and hash the prompt prefix.
2. Map it to a prompt family.
3. Check whether a shared KV snapshot already exists.
4. If it does, restore that shared state and run only the session-specific suffix.
5. Store per-session state separately.
6. On resume, reconstruct a working KV view from shared snapshot + session delta.

And the most important engineering rule is this:

> the fallback is always raw KV.

If restore cost is too high, quality drifts, or the assumptions stop holding, the engine should degrade gracefully to ordinary serving.

## What I’m actually trying to prove

The first claim is narrow and measurable:

> on repeated-prefix workloads, a prefix-aware KV runtime should fit more useful session state on the same hardware.

Not “solve LLM serving.”
Not “replace every inference engine.”
Just that one claim.

If it holds, then Phase 2 becomes interesting:

- near-prefix clustering,
- grouped KV blocks,
- transform-domain residual coding,
- shared prototypes plus quantized deltas.

That is where the BM3D inspiration becomes an actual codec instead of just an architectural instinct.

## Failure modes I expect

There are several ways this can fail:

- the session-specific delta may still be too large,
- reconstruction cost may hurt latency,
- some layers may be too sensitive to compression,
- exact-prefix reuse may help less than I expect on real traffic,
- the best transform for grouped KV blocks may turn out not to be DCT, FFT, or PCA at all.

That is fine. I would rather discover that with a disciplined baseline than hide it behind a clever-sounding compression story.

## Ongoing work

I’m building this at [github.com/0xbadcaffe/kv3d](https://github.com/0xbadcaffe/kv3d).

It is early. The architecture document covers the longer roadmap, from exact-prefix snapshotting through collaborative block grouping and a real residual codec.

What I like about this project is that it stopped feeling like “AI” and started feeling like systems work:

there is redundant state,  
there are memory tiers,  
there is a scheduler,  
there is a restore path,  
and there is probably a simpler baseline hiding underneath a more complicated idea.

That is usually where the interesting work starts.
