
# RadixCache

**A request-aware memory manager for LLM prefix caching, built with Rust and Go.**

RadixCache explores how shared-prefix caching and explicit memory management can reduce duplicated KV-cache storage across concurrent inference requests.

The project focuses on memory allocation, prefix reuse, safe reclamation, and performance measurement rather than building a complete inference server.

## Motivation

LLM inference repeatedly processes token sequences that may share common prefixes. When requests have the same token prefix, their corresponding KV-cache entries may be reusable instead of being stored independently.

For example:

```text
Request A: The cat
Request B: The cat is sleeping
Request C: The cat is drinking tea
```

The prefix `The cat` can be represented once and shared by all three requests.

RadixCache uses a radix tree to organize cached prefixes and associate them with their corresponding cache blocks. When a request finishes, its private blocks can be reclaimed while shared blocks remain available to other requests.

## Goals

- Implement radix-tree-based prefix lookup and sharing.
- Manage cache memory through fixed-size blocks.
- Track shared ownership and reclaim blocks safely.
- Handle requests with overlapping lifetimes.
- Measure memory usage, cache reuse, allocation overhead, and reclamation performance.
- Compare the implementation against a baseline without prefix sharing.

## Architecture

### Rust: Cache Engine

The Rust engine is responsible for:

- Radix-tree indexing and prefix lookup.
- Fixed-size block allocation and reuse.
- Mapping cached prefixes to KV-cache blocks.
- Tracking references to shared cache entries.
- Reclaiming blocks when they are no longer needed.

### Go: Workload Generator

The Go component generates synthetic inference workloads and measures their behavior.

It will support:

- Configurable request counts and token-prefix overlap.
- Concurrent request lifetimes.
- Repeated and partially shared prefixes.
- Collection of memory and performance metrics.

The initial implementation uses simulated KV-cache blocks and does not require a GPU or an actual LLM runtime.

## Design Boundaries

RadixCache is a cache and memory-management experiment, not an inference server.

It does not initially implement:

- Neural-network inference.
- GPU kernels or hardware-specific memory management.
- Request batching or inference scheduling.
- Model loading or token generation.
- Distributed caching.

These can be considered separately if the core experiment justifies them.

## Planned Benchmarks

RadixCache will evaluate:

| Metric | Purpose |
|---|---|
| Peak allocated blocks | Measure maximum cache memory demand |
| Cache hit rate | Measure successful prefix reuse |
| Allocation latency | Measure the cost of obtaining blocks |
| Reclamation latency | Measure the cost of releasing request-owned memory |
| Memory fragmentation | Measure wasted capacity in the block pool |
| Prefix lookup latency | Measure radix-tree lookup performance |

Results will be compared against a baseline that stores request prefixes independently. Workloads will vary in prefix overlap, request duration, and concurrency.

## Project Structure

```text
radixcache/
├── rust-engine/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── radix_tree.rs
│   │   ├── block_pool.rs
│   │   ├── prefix_cache.rs
│   │   └── request.rs
│   └── benches/
└── go-workload/
    ├── go.mod
    ├── main.go
    ├── workload.go
    └── metrics.go
```

The structure may evolve as the ownership model and benchmark requirements become clearer.

## Technology Stack

- **Rust:** cache engine and explicit memory management.
- **Go:** workload generation and benchmark orchestration.
- **Criterion:** Rust microbenchmarks.
- **proptest:** property-based testing of cache invariants.
- **tracing:** instrumentation and diagnostic events.

The first version uses standard-library data structures wherever practical.

## Development Status

**Status: Initial design**

The project is currently in the design and implementation-planning stage. The architecture and benchmark goals are planned; implementation and performance results have not yet been established.

## Long-Term Direction

The objective is to determine how prefix indexing, block allocation, ownership tracking, and reclamation policies affect cache efficiency under realistic request patterns.

Any performance improvements will be reported against defined baselines and reproducible workloads.

