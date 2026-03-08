---
title: "IIO Subsystem Internals: When the Kernel Lies"
description: "A deep dive into the Industrial I/O subsystem, ADC drivers, and the lies that buffer modes tell you."
pubDate: 2025-01-14
tags: ["kernel", "iio", "drivers", "embedded"]
---

The Linux IIO subsystem is one of those parts of the kernel that looks simple from the outside and reveals its true nature only after you've spent a week debugging a 4-sample jitter that shouldn't exist.

This is that story.

## Background

I was working with an ADC connected over SPI, driven by an Analog Devices IIO driver. The device samples at 1MSPS. We were seeing consistent 4-sample timing anomalies at the output of the buffer.

IIO buffers sound simple: the kernel fills a ring buffer, userspace reads from it. The ADC fires an interrupt, the driver pushes samples, you read them out.

The reality is more interesting.

## The buffer pipeline

When you enable an IIO buffer, samples flow through this chain:

```
Hardware → DMA → kfifo (driver) → iio_buffer → /dev/iio:deviceN
```

Each stage has its own locking, its own copy, and its own opportunity to introduce latency. The `kfifo` in the driver is protected by a spinlock. The `iio_buffer` push is protected by a mutex.

A mutex. In an interrupt handler path.

This is legal because the IIO push happens from a threaded IRQ handler, not hard IRQ context. But it means your "1MSPS" stream is now subject to scheduler jitter.

## Finding the real problem

The 4-sample anomaly turned out to be cache-related. The DMA buffer wasn't being properly flushed before the CPU read it. On a cached architecture, you can read stale data even after DMA completes if you don't issue an explicit `dma_sync_single_for_cpu()`.

The driver was calling it — but after the data copy, not before.

```c
/* Wrong order in original driver */
memcpy(dst, dma_buf, len);
dma_sync_single_for_cpu(dev, dma_handle, len, DMA_FROM_DEVICE);

/* Correct */
dma_sync_single_for_cpu(dev, dma_handle, len, DMA_FROM_DEVICE);
memcpy(dst, dma_buf, len);
```

Four samples of stale data, exactly the size of a cache line on this particular SoC.

## The fix

One line change. One week of debugging. This is embedded kernel work.

The patch went upstream. The commit message is longer than the diff.
