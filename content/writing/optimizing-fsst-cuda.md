---
title: "My Journey Optimizing a CUDA Kernel with Polar Signals"
date: 2026-06-24T00:00:00Z
draft: false
tags: ["gpu", "cuda", "performance", "compression", "vortex", "databases"]
originalPost:
  site: "Polar Signals"
  url: "https://www.polarsignals.com/blog/posts/2026/06/24/optimizing-fsst-cuda"
---

An account of building and optimizing a CUDA kernel to decompress FSST-encoded strings for Vortex, Polar Signals' columnar file format. Starting from a naive implementation that was slower than CPU decompression, the bottlenecks are found methodically with a GPU profiler and closed through targeted optimizations until the kernel pulls well ahead.

The key breakthrough came from optimizing memory stores rather than loads—buffering decompressed bytes in registers and emitting aligned, wide stores instead of byte-by-byte writes. Combined with a split-kernel optimization inspired by the GSST paper, the kernel reached 78+ GiB/s, roughly 3x the CPU baseline and 3x NVIDIA's cuCompute zstd, hitting 40% of measured device bandwidth. GPU performance engineering, it turns out, is fundamentally about answering "where is my data?"—managing memory throughput and hiding latency through architectural awareness.
