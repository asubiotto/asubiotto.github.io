---
title: "Deterministic Simulation Testing in Rust: A Theater Of State Machines"
date: 2025-07-08T00:00:00Z
draft: false
tags: ["rust", "testing", "databases", "distributed-systems", "deterministic-simulation"]
originalPost:
  site: "Polar Signals"
  url: "https://www.polarsignals.com/blog/posts/2025/07/08/dst-rust"
---

At Polar Signals, we're building a new Rust database using a state machine architecture for deterministic simulation testing (DST). This approach provides complete control over concurrency, time, randomness, and failure injection by implementing core components as single-threaded state machines that communicate through messages.

## The Theater of State Machines

Our architecture uses a "dimensionality reduction" technique by constraining all interactions to message passing. This allows us to centralize control of testing ingredients in a single message bus, which acts as the central director of the testing process.

Key benefits include:
- Complete control over system concurrency and timing
- Reproducible testing that can uncover complex bugs
- Comprehensive failure injection capabilities
- Powerful mental model for reasoning about system behavior

While this approach introduces cognitive overhead, it ultimately leads to more robust and correct system design by making it possible to deterministically simulate and test complex distributed system behaviors.