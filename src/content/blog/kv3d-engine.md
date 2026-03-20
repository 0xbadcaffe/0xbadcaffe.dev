---
title: "KV3D Engine: rethinking KV cache compression for LLM serving"
description: "My first real dive into AI, LLMs, and tokens — and what happens when I got curious about where all the memory goes."
pubDate: 2026-03-20
tags: ["llm", "inference", "systems", "kv-cache", "bm3d"]
---

## My first try on AI, LLM and tokens

I've spent most of my career as far from AI as you can get — kernel space, network drivers, hardware that lies to you. LLMs always felt like someone else's problem.

Then I started actually using them. And like any embedded engineer, I couldn't just use a thing. I had to understand what was happening underneath.

So here's where I landed after a few weeks of reading. Starting simple.

## What's a token? (for a 3 year old)

Imagine you're reading a book out loud, but instead of reading one letter at a time, you read little chunks — "the", "dog", "ran". Those chunks are tokens.

An LLM doesn't see words or sentences. It sees a stream of tokens — little pieces of text, sometimes a full word, sometimes just part of one. Everything you type gets chopped into tokens before the model ever touches it.

The model reads all your tokens, thinks really hard, and then picks the next token that makes the most sense. Then the next. Then the next. That's how it writes back to you — one token at a time.

## What's a KV cache? (also for a 3 year old)

When the model reads your tokens, it does a lot of math on each one. It figures out how each token relates to every other token — which words matter for understanding which other words.

That math is expensive. And here's the thing: once it's done, you don't need to redo it.

So the model saves the results. Key-Value cache — KV cache. Keys and values are the two parts of that math it saves. Every token you've sent, the model has already processed. When you send the next message, it picks up right where it left off without recomputing everything from scratch.

The problem: that saved math takes up a lot of memory. A lot. Especially when you have hundreds of sessions running at the same time, each with their own history.

## Where it gets interesting

Here's what caught my attention.

In most real deployments — enterprise copilots, agent systems, RAG pipelines — most sessions start with the exact same prompt prefix. The same system instructions, the same tool definitions, the same scaffolding. Repeated thousands of times across thousands of sessions.

And yet every session computes and stores its own KV cache for that shared prefix. Independently. Every time.

That felt wasteful in a way I recognized — the kind of redundancy that would get you a comment in a kernel review.

**KV3D Engine** is the idea I started building around: compute the shared prefix cache once, store it as a snapshot, and keep only the session-specific delta per user.

```
KV_session ≈ KV_shared_prefix + delta_session
```

More sessions on the same hardware. Less redundant work. Same output quality.

## The BM3D connection

This is where it gets more interesting — and where the name comes from.

**BM3D** (Block-Matching and 3D Filtering) is a classical image denoising algorithm. The idea is elegant: instead of denoising each image patch in isolation, find *similar* patches across the image, group them into a 3D stack, apply a transform (DCT or wavelet) along the grouping dimension, threshold in that transform domain, then inverse-transform and aggregate back.

The key insight is **non-local grouping**: similar patches share structure. Compressing them together is dramatically more efficient than compressing each independently.

The paper that started connecting this to KV caches for me is [KVSharer (arxiv 2412.19821)](https://arxiv.org/html/2412.19821v1), which explores sharing and compressing KV cache layers across LLM inference. Reading it alongside BM3D literature made something click.

**KV blocks across sessions with the same prompt family look a lot like similar patches in an image.** They share structure — the shared prefix induces similar activation patterns across layers. The per-session variation is the "noise" or residual on top of that shared structure.

The BM3D pipeline maps onto the KV problem like this:

| BM3D concept | KV3D equivalent |
|---|---|
| Similar image patches | KV blocks from sessions with the same prompt family |
| Block matching / grouping | Exact-prefix hash, then near-prefix clustering |
| 3D transform stack (DCT/wavelet) | Transform applied across the grouped KV block dimension |
| Threshold / compress in transform domain | Compress the shared prototype; store only the residual delta |
| Inverse transform + aggregate | Reconstruct working KV view before decode |

## What we're not doing

To be clear: **we are not literally porting BM3D**.

We are porting the *principle* of non-local grouping.

Then, instead of BM3D's classic DCT/wavelet transform stack, we test transforms that make sense for KV tensors — DCT, FFT, PCA, or learned dictionaries applied across grouped KV blocks. The right transform is an empirical question. The principle is what matters.

The starting point (MVP) is simpler: exact-prefix matching with a shared snapshot and per-session delta. No grouping, no transforms yet. That's Phase 1. The BM3D-inspired collaborative codec is Phase 2 — once the baseline proves the idea works at all.

## The architecture

The engine is built as an inference runtime layered on top of `llama.cpp`. The novel logic lives in the cache manager and scheduler, not the execution path.

At a high level:

```
request → prefix detector → session manager → execution backend
                                   ↕
              shared prefix snapshot store + delta codec
                                   ↕
                  GPU hot cache / RAM warm cache / SSD cold cache
```

For each incoming request:
1. Hash and canonicalize the prompt prefix to detect the prefix family
2. Check if a shared KV snapshot exists for that family
3. If yes — restore the shared prefix state and run only the session-specific suffix through `llama.cpp`
4. Compress and store the session-specific delta
5. On next request — reconstruct the working KV view from shared snapshot + delta

The fallback is always raw KV. If something looks wrong — latency spikes, quality drift, codec failure — the engine reverts to baseline behavior automatically.

## The MVP

The first version is intentionally narrow:

- `llama.cpp` as the backend
- exact prefix matching only — no fuzzy clustering yet
- one shared KV snapshot per prefix family
- per-session deltas stored separately
- GPU hot cache + host RAM warm cache
- OpenAI-compatible chat/completions API
- honest benchmarks against a baseline runtime

The goal isn't to solve everything. The goal is to prove one claim: *for repeated-prefix workloads, this approach fits more useful session state on the same hardware.*

If that holds up, the BM3D-inspired phase follows.

## Ongoing work

I'm building this at [github.com/0xbadcaffe/kv3d](https://github.com/0xbadcaffe/kv3d). Early days. The architecture document covers the full design including the roadmap from exact-prefix MVP through collaborative block codec.

I don't know yet if this works at the scale I'm imagining. The delta representation might not compress as well in practice as it does on paper. The right transform for grouped KV blocks might be none of the ones I listed. The quality guardrails might be too aggressive or not aggressive enough.

More on this as it develops.
