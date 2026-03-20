---
title: "KV3D Engine: rethinking KV cache compression for LLM serving"
description: "My first real dive into AI, LLMs, and tokens — and what happens when I got curious about where all the memory goes."
pubDate: 2026-03-20
tags: ["llm", "inference", "systems", "kv-cache"]
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

## The plan

The first version is intentionally narrow:

- `llama.cpp` as the backend
- exact prefix matching only — no fuzzy clustering yet
- one shared KV snapshot per prefix family
- per-session deltas stored separately
- OpenAI-compatible API on top
- honest benchmarks against a baseline runtime

The goal isn't to solve everything. The goal is to prove one claim: *for repeated-prefix workloads, this approach fits more useful session state on the same hardware.*

If that holds up, the rest follows.

## As always — I'll try to build it and break it

I don't know yet if this works at the scale I'm imagining. The load bias math might be wrong. The delta representation might not compress as well in practice as it does on paper. The quality guardrails might be too aggressive or not aggressive enough.

That's fine. That's the job.

Build it. Break it. Figure out where the real gains come from.

More soon.
