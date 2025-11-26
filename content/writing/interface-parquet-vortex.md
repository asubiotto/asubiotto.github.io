---
title: "Questioning an Interface: From Parquet to Vortex"
date: 2025-11-25T00:00:00Z
draft: false
tags: ["databases", "datafusion", "vortex", "apache parquet"]
originalPost:
  site: "Polar Signals"
  url: "https://www.polarsignals.com/blog/posts/2025/11/25/interface-parquet-vortex"
---

An exploration of how Polar Signals migrated their profiling database from Apache Parquet to Vortex, achieving a 70% average performance improvement across all queries. The shift demonstrates how interface design choices can impose unexpected limitations on system performance.

Parquet's "retrieval" differs fundamentally from "querying"—converting Parquet files to Arrow format for actual computation consumed significant CPU resources. Vortex, with its zero-copy conversion to Arrow and SIMD-friendly encodings, proved to be a better fit for our use case.
