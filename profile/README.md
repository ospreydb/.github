# OspreyDB

**Open-core vector search and multi-vector CPU reranking engine built in Rust for S3 and object storage.**

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-blue.svg)](https://github.com/ospreydb/osprey-engine/blob/main/LICENSE)
[![Website](https://img.shields.io/badge/docs-ospreydb.dev-09090B)](https://ospreydb.dev)
[![Crates.io](https://img.shields.io/crates/v/osprey-cli?label=crates.io)](https://crates.io/crates/osprey-cli)

---

## The Problem We Solve

Conventional vector databases duplicate Parquet data lakes and media embeddings into **RAM-resident HNSW graphs**, where DRAM costs ~300–450× more per GB/month than S3 Standard. At the same time, placing HNSW directly on object storage collapses under **12–30 serial pointer-chasing `Range-GET` hops** per query (`300–1,200 ms` of network latency alone), while ColBERT/ColPali late-interaction reranking typically forces teams to rent dedicated GPU instances or pay per-call external `/v1/rerank` API tolls.

**OspreyDB decouples routing codes from storage slabs across three 64-byte cache-line aligned files:**

1. **`.osi` (RAM / `mmap`)** — 1-bit FWHT-rotated `RaBitQ` sign codes + TSP-ordered `f16` centroids + `RoaringBitmap` tombstones, scanned linearly with `AVX-512 VPOPCNTDQ`, `AVX2`, or `ARM64 NEON`. Prunes >99% of vectors locally without network I/O.
2. **`.osv` (S3 / MinIO / Disk)** — Zstd-compressed ~16 KiB minipages ordered by TSP cluster locality (`doc_id` zone maps, exact `f16` vectors, JSON payloads). Coalesced at query time via **DP range coalescing** (1D Dynamic Programming `< 0.5 ms`) into a single parallel HTTP/2 `get_ranges` round-trip.
3. **`.osr` (CPU `LUT-Rank`)** — 64-byte/token nested-lattice store (`2 × 4-bit` layers, `gamma = 0.28`) for ColBERT/ColPali `MaxSim` reranking on commodity CPUs using 16-entry `int8` LUTs (`< 8 µs` synthesis) and `vpshufb`/`vtbl` register shuffles—no GPUs required.

---

## Public Repositories

| Repository | Purpose | Stack / Notes |
| :--- | :--- | :--- |
| [`ospreydb/osprey-engine`](https://github.com/ospreydb/osprey-engine) | Open-core engine (`osprey-core`, `osprey-simd`, `osprey-storage`) and static CLI (`osprey-engine`) | Rust, Apache-2.0, `memmap2`, `object_store`, Zstd |
| [`ospreydb/ospreydb`](https://github.com/ospreydb/ospreydb) | Commercial database server daemon (`osprey`), WAL, LSM delta buffer, CoW compactor, REST & Arrow Flight | Rust, consumes `osprey-core` / `osprey-storage` public traits |
| [`ospreydb/benchmarks`](https://github.com/ospreydb/benchmarks) | Reproducible benchmark suite against Qdrant (`FP32` & `1-bit BQ`), `pgvector`, and ColBERTv2 | Controlled recall-matched harness, MinIO `tc netem 15ms`, `perf` |
| [`ospreydb/homebrew-tap`](https://github.com/ospreydb/homebrew-tap) | Official Homebrew tap for macOS & Linux (`brew install ospreydb/tap/osprey` & `osprey-engine`) | Automated SHA-256 updates on tagged releases |
| [`ospreydb/site`](https://github.com/ospreydb/site) | Website, architecture docs, AWS/S3 TCO estimator, and offline Ed25519 licensing webhook | Astro, Cloudflare Pages, `ospreydb.dev` |

> Note: Commercial production extensions (`WAL`, `LSM Delta Buffer`, `CoW` compactor, `manifest.json` S3 CAS coordination, Qdrant REST `:6333`, `/v1/embeddings`, `/v1/rerank`, and Apache Arrow Flight `:8815`) live in the `osprey` daemon (`ospreydb/ospreydb`), consuming `osprey-core` and `osprey-storage` exclusively through public traits.

---

## Validated Benchmark Summary

All numbers below come directly from raw CSV artifacts in [`ospreydb/benchmarks/results/`](https://github.com/ospreydb/benchmarks/tree/main/results) on a 12-vCPU x86_64 AVX-512 host (`29 GiB` RAM, Linux). OspreyDB ran with a **bounded 256 MiB page cache** (`VmHWM`), while Qdrant and `pgvector` ran in Docker with unrestricted host page cache (`cgroup v2 memory.peak`).

### 1. Recall-Matched ANN (`compare_engines_controlled.csv`)

| Dataset | Engine | Target / Test R@10 | Client P50 | Server P50 | Peak RAM | Disk |
| :--- | :--- | :---: | ---: | ---: | ---: | ---: |
| **Cohere (`100k × 768d`)** | **OspreyDB (`.osi + .osv`)** | `0.95` → **`0.9669`** | `0.795 ms` | `0.542 ms` | **`234.6 MiB`** | **`151.9 MiB`** |
| **Cohere (`100k × 768d`)** | **OspreyDB (`.osi + .osv`)** | `0.97` → **`0.9889`** | `0.972 ms` | `0.714 ms` | **`234.6 MiB`** | **`151.9 MiB`** |
| Cohere (`100k × 768d`) | Qdrant HNSW FP32 (`ef=32/64`) | `0.9647` / `0.9855` | `0.703` / `0.856 ms` | `0.281` / `0.437 ms` | `2,730.5 MiB` (11.6×) | `1,368.2 MiB`¹ |
| Cohere (`100k × 768d`) | Qdrant 1-bit BQ (`ef=256, os=3`) | `0.9342` *(miss)* | `0.678` / `0.701 ms` | `0.269` / `0.283 ms` | `2,799.4 MiB` | `661.3 MiB`¹ |
| Cohere (`100k × 768d`) | pgvector HNSW FP32 (`ef=64`) | `0.9771` | `1.788` / `1.782 ms` | `1.733` / `1.209 ms` | `1,332.1 MiB` (5.7×) | `1,184.5 MiB` |
| **SIFT1M (`1M × 128d`)** | **OspreyDB (`.osi + .osv`)** | `0.95` → **`0.9641`** | `2.982 ms` | `2.674 ms` | **`462.2 MiB`** | **`212.9 MiB`** |
| **SIFT1M (`1M × 128d`)** | **OspreyDB (`.osi + .osv`)** | `0.97` → **`0.9738`** | `3.434 ms` | `3.126 ms` | **`462.2 MiB`** | **`212.9 MiB`** |
| SIFT1M (`1M × 128d`) | Qdrant HNSW FP32 (`ef=32/64`) | `0.9672` / `0.9905` | `0.854` / `0.888 ms` | `0.459` / `0.497 ms` | `4,194.5 MiB` (9.1×) | `868.2 MiB` |
| SIFT1M (`1M × 128d`) | Qdrant 1-bit BQ (`ef=256, os=3`) | `0.0624` *(collapse)* | `0.726` / `0.750 ms` | `0.344` / `0.361 ms` | `4,780.9 MiB` | `910.7 MiB` |
| SIFT1M (`1M × 128d`) | pgvector HNSW FP32 (`ef=64/96`) | `0.9726` / `0.9863` | `2.374` / `3.262 ms` | `2.089` / `2.785 ms` | `2,594.0 MiB` (5.6×) | `2,143.7 MiB` |

¹ Sampled prior to Qdrant background compaction (`804 MiB` FP32 / `355 MiB` BQ post-shutdown). In the single-process uncalibrated run (`compare_engines.csv` on Cohere `100k × 768d`), OspreyDB reaches **`1.455 ms` P50** at `R@10 = 0.9695` with **`15.01 MiB`** `.osi` resident index RAM vs. **`634.9 MiB`** for Qdrant HNSW FP32 (**42.3× smaller resident index**; `26.0×` vs. `pgvector` `390.62 MiB`).

### 2. Late-Interaction Reranking & Object Storage (`lutrank_vs_colbert.csv` & `pareto_s3.csv`)

- **CPU `LUT-Rank` (`MS MARCO`, 200 candidates/query):** **`0.329 ms` P50**, **`2,849.1 QPS`** (fastest CPU engine vs. PyTorch CPU `0.416 ms` / `2,300.7 QPS`, ONNX Runtime `0.476 ms`, NumPy BLAS `0.801 ms`), **`97.2 MiB`** `.osr` index (**7.9× smaller** than ColBERTv2 `768.43 MiB`), `0.7768` vs. `0.7876` `NDCG@10` (`98.6%` retention).
- **BEIR SciFact / FiQA (`LUT-Rank`):** `0.505 ms` / `0.519 ms` P50, `2,121.7` / `2,005.7 QPS`, `10.24 MiB` / `112.82 MiB` index (`7.9×` smaller than ColBERTv2).
- **DP Range S3 Coalescing (`MinIO + tc netem 15ms`, `17.03 ms` RTT on Cohere 768d):** At `R@10 = 0.960` (`nprobe=32, refine_m=200`), coalesces `158.6` candidate `.osv` pages into **`18.5` S3 `Range-GET` requests** (**8.6× fewer requests**, up to **49.3×** at `nprobe=4, refine_m=300`) while keeping `VmHWM` under `103 MiB`.

---

## Transparent Open-Core Model

| Tier | License / Scope | Key Highlights |
| :--- | :--- | :--- |
| **`osprey-engine` CLI & Core Crates** | **Apache-2.0** (`ospreydb/osprey-engine`) | Full SIMD math (`FWHT`, `1-Bit Binary K-Means`, `RaBitQ`, DP range coalescing, `LUT-Rank`), Parquet/JSONL/NPY ingestion, S3/local search, tombstone deletes, and TCO benchmarking (`osprey-engine bench`). |
| **`osprey` Single-Node Free** | **Free forever (no key required)** | Full resident daemon, CRC32/fsync WAL, LSM delta buffer, HTTP `:6333`, and `/v1/embeddings` up to **5,000,000 vectors** on a single node (reads remain online above 5M; Docker Desktop style). |
| **`osprey` Pro / Scale** | **Commercial (100% BYOC)** | Unlimited vectors, multi-node S3 `manifest.json` CAS sync, background CoW compaction, `/v1/rerank`, and Arrow Flight `:8815`. **$360/yr** launch offer (`$49/mo`) up to 3 nodes. Verified **100% offline via `Ed25519` (`< 50 µs`) — zero phone-home, zero telemetry.** |

---

## Quickstart

```bash
curl -fsSL https://ospreydb.dev/install-engine.sh | sh
# or: brew install ospreydb/tap/osprey-engine | cargo install osprey-cli

osprey-engine doctor
osprey-engine bench --with-lutrank --project-vectors 50000000 --project-dim 1536 --project-qpm 10000000
```

[ospreydb.dev](https://ospreydb.dev) · [Architecture](https://ospreydb.dev/architecture) · [Benchmarks](https://ospreydb.dev/benchmarks) · [Pricing](https://ospreydb.dev/pricing)
