---
title: "Writing a Packet Analyzer in Rust with Ratatui"
description: "Building packrat — a Wireshark-style TUI. What the kernel tells you that libpcap doesn't."
pubDate: 2025-03-01
tags: ["rust", "ratatui", "networking", "kernel"]
---

Every few years I get frustrated enough with existing tools to build my own. This time it was packet analysis.

Wireshark is powerful but heavy. `tcpdump` is fast but blind. I wanted something in between — a TUI I could live inside, with the right level of detail for embedded network debugging.

So I built **packrat**.

## The architecture

Packrat is a Rust workspace with five tabs:

- **Packets** — live capture with filtering
- **Analysis** — protocol breakdown per packet
- **Strings** — printable string extraction from payloads
- **Dynamic** — flow statistics updated in real time
- **Visualize** — sparklines and traffic graphs

The UI layer is [Ratatui](https://ratatui.rs), which gives you full terminal control without fighting ncurses. The capture backend is optional: you can use raw sockets or link against libpcap.

## What libpcap doesn't tell you

When you capture via libpcap, you're seeing packets *after* the kernel has already made decisions about them. VLAN tags may be stripped. Timestamps are userspace-approximated. For most use cases this is fine.

For embedded debugging — where you're chasing timing issues between a network interface driver and an IIO subsystem — it's not fine.

The solution is a `AF_PACKET` socket with `PACKET_MMAP`, which gives you kernel-timestamped frames in a shared memory ring buffer. packrat uses this by default when libpcap is not explicitly requested.

```rust
let sock = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL) as i32)?;
setsockopt(sock, SOL_PACKET, PACKET_RX_RING, &tp)?;
let ring = mmap(sock, tp.tp_block_size * tp.tp_block_nr)?;
```

The timestamps you get from `PACKET_MMAP` are hardware-timestamped on NICs that support it, and software-timestamped (with much better accuracy than userspace) on everything else.

## The splash screen

There's an ASCII rat on startup. It's non-negotiable.

```
  (\(\
  ( -.-)
  o_(")(")
  packrat v0.1.0
```

It disappears after 800ms. I timed it precisely so it's long enough to be noticed and short enough not to be annoying.

## What's next

- BPF filter input in the UI
- Export to pcap format
- Better IPv6 support
- Publishing to crates.io

The source will be on GitHub once I clean up the worst parts. Some of it is embarrassing.
