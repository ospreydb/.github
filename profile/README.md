# OspreyDB

**100% Open-Source (Apache-2.0) vector search engine and database server built in Rust for S3 and object storage.**

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-blue.svg)](https://github.com/ospreydb/ospreydb/blob/main/LICENSE)
[![Website](https://img.shields.io/badge/docs-ospreydb.dev-09090B)](https://ospreydb.dev)

---

## The Problem We Solve

Conventional vector databases duplicate Parquet data lakes and media embeddings into **RAM-resident HNSW graphs**, where DRAM costs ~300–450× more per GB/month than S3 Standard. At the same time, placing HNSW directly on object storage collapses under **12–30 serial pointer-chasing `Range-GET` hops** per query (`300–1,200 ms` of network latency alone), while ColBERT/ColPali late-interaction reranking typically forces teams to rent dedicated GPU instances or pay per-call external `/v1/rerank` API tolls.

**OspreyDB decouples routing codes from storage slabs across three 64-byte cache-line aligned files:**

1. **`.osi` (RAM / `mmap`)** — 1-bit FWHT-rotated `RaBitQ` sign codes + TSP-ordered `f16` centroids + `RoaringBitmap` tombstones, scanned linearly with `AVX-512 VPOPCNTDQ`, `AVX2`, or `ARM64 NEON`. Prunes >99% of vectors locally without network I/O.
2. **`.osv` (S3 / MinIO / Disk)** — Zstd-compressed ~16 KiB minipages ordered by TSP cluster locality (`doc_id` zone maps, exact `f16` vectors, JSON payloads). Coalesced at query time via **DP range coalescing (`RangePlanner`)** (1D Dynamic Programming `< 0.5 ms`) into a single parallel HTTP/2 `get_ranges` round-trip.
3. **`.osr` (CPU `LUT-Rank`)** — 64-byte/token nested-lattice store (`2 × 4-bit` layers, `gamma = 0.28`) for ColBERT/ColPali `MaxSim` reranking on commodity CPUs using 16-entry `int8` LUTs (`< 8 µs` synthesis) and `vpshufb`/`vtbl` register shuffles—no GPUs required.

---

## Repositories

| Repository | Purpose | Stack / Notes |
| :--- | :--- | :--- |
| [`ospreydb/ospreydb`](https://github.com/ospreydb/ospreydb) | Unified engine, CLI & server (`osprey`): `osprey-core`, `osprey-simd`, `osprey-storage`, `osprey-cli`, WAL/LSM delta buffer, CoW S3 compactor, Qdrant REST `:6333`, `/v1/embeddings`, `/v1/rerank`, and Arrow Flight `:8815` | Rust, **100% Apache-2.0** |
| [`ospreydb/benchmarks`](https://github.com/ospreydb/benchmarks) | Reproducible benchmark suite against Qdrant (`FP32`, `SQ8`, `1-bit BQ`), `pgvector` (`FP32`, `halfvec`), and ColBERTv2 | Controlled recall-matched harness, `cgroup v2 memory.peak`, `perf` |
| [`ospreydb/homebrew-tap`](https://github.com/ospreydb/homebrew-tap) | Official Homebrew tap for macOS & Linux (`brew install ospreydb/tap/osprey`) | Prebuilt release binaries |

---

## Quickstart

```bash
curl -fsSL https://ospreydb.dev/install.sh | sh
# or: brew install ospreydb/tap/osprey

# Hardware & SIMD diagnostics
osprey doctor

# Start the unified server locally (with local S3 store fallback)
osprey serve --data-dir ./data --s3-uri file:///tmp/osprey-s3
```

[ospreydb.dev](https://ospreydb.dev) · [Architecture](https://ospreydb.dev/architecture) · [Benchmarks](https://ospreydb.dev/benchmarks) · [Getting Started](https://ospreydb.dev/getting-started)
