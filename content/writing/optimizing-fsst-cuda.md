---
title: "My Journey Optimizing a CUDA Kernel with Polar Signals"
date: 2026-06-24T00:00:00Z
draft: false
tags: ["gpu", "cuda", "vortex", "performance", "profiling"]
originalPost:
  site: "Polar Signals"
  url: "https://www.polarsignals.com/blog/posts/2026/06/24/optimizing-fsst-cuda"
---

An account of building and optimizing an FSST string decompression CUDA kernel for Vortex, and using the Polar Signals GPU profiler to find out why the first version was slower than the CPU implementation.

Memory access patterns dominate GPU performance. Buffering decompressed bytes in registers to enable wider, aligned writes instead of storing a byte at a time ended up making decompression 3x faster than comparable solutions.
