---
title: "packrat — building a packet analyzer in Rust because I was annoyed"
description: "Wireshark is too heavy and tcpdump is too blind. Building something in between, with a TUI and an ASCII rat."
pubDate: 2025-03-01
tags: ["rust", "ratatui", "networking", "linux"]
---

Every few years I get frustrated enough with existing tools to write my own. This time it was packet capture.

The problem: I'm debugging network behavior on an embedded board. Wireshark is overkill and won't run there. `tcpdump` gives me raw text I have to pipe somewhere. I want something I can SSH into a device, run, and actually *see* what's happening — timing, flows, payload strings — without piping through three other tools.

So I built packrat. It's a TUI packet analyzer written in Rust.

## why Rust

Mostly because I wanted to learn it properly and this felt like the right kind of project. Low-level, performance matters a bit, good opportunity to fight the borrow checker in interesting ways. I was also tired of writing C networking code where every second line is a potential use-after-free.

The UI layer is [Ratatui](https://ratatui.rs), which gives you full terminal control without fighting ncurses. It's genuinely good. The widget model takes an afternoon to understand and then mostly stays out of your way.

## the capture backend

This is the part I spent the most time on.

The obvious approach is libpcap. It works, it's portable, everyone uses it. For most purposes it's fine. But libpcap gives you packets *after* the kernel has already touched them — VLAN tags may be stripped, timestamps are approximated in userspace, and you're not seeing exactly what hit the wire.

For the embedded debugging I care about, that's not good enough. I need to know *when* something happened, not a userspace approximation of when something happened.

The answer is `AF_PACKET` with `PACKET_MMAP`. You create a raw socket, set up a shared memory ring buffer between your process and the kernel, and get hardware-timestamped frames directly — no copying until you decide to copy them. On NICs that support hardware timestamping, you get the actual time the frame was received by the NIC. On everything else, you get software timestamps with much better accuracy than anything userspace could achieve.

```rust
let sock = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL) as i32)?;
setsockopt(sock, SOL_PACKET, PACKET_RX_RING, &tp)?;
let ring = mmap(sock, tp.tp_block_size * tp.tp_block_nr)?;
```

packrat uses this by default and falls back to libpcap if you explicitly request it.

## the tabs

The UI has five tabs. This is probably three too many but I've used all of them:

**Packets** — live capture table with BPF filter input. The thing you look at 90% of the time.

**Analysis** — protocol dissection for the selected packet. Ethernet → IP → TCP, field by field.

**Strings** — printable strings extracted from the payload. Useful more often than I expected.

**Dynamic** — per-flow statistics updating in real time. Packet counts, byte totals, rate estimates.

**Visualize** — sparklines. I built these last and they're the feature I show people first.

## the splash screen

There's an ASCII rat on startup. It's non-negotiable.

```
  (\(\
  ( -.-)
  o_(")(")
  packrat v0.1.0
```

It stays for 800ms. Long enough that you notice it, short enough that it's not annoying. I tuned this carefully.

## what's still rough

BPF filter input in the UI exists but error handling is incomplete. Export to pcap is on the list. IPv6 support is minimal. The ring buffer management in the capture backend has one edge case I haven't fully tracked down yet.

Source is on [GitHub](https://github.com/0xbadcaffe/packrat) when I clean up the worst parts. I keep saying that.
