
# RadixCache

> **Status: experimental design / in progress.**\
> A Rust project exploring fixed-size KV-cache block allocation and
> reuse through a flat global index, with a per-request vector of block
> references.

## Overview

RadixCache explores an alternative to representing cached token prefixes
with an explicit radix tree.

The proposed design separates three responsibilities:

1.  **Block allocator** --- owns a pool of fixed-size memory blocks and
    reuses released blocks.
2.  **Global block index** --- maps a block identity to the logical
    block allocated for that content.
3.  **Request block vector** --- records the ordered blocks a request
    uses.

The goal is to investigate whether this flatter representation can
provide a simpler implementation and competitive lookup, allocation, and
reclamation performance compared with a tree-based baseline.

This repository is an infrastructure experiment, not an LLM
implementation. The initial work can use simulated KV-cache blocks and
workloads without requiring a running model or GPU.

## Design

``` text
Incoming token IDs
        |
        v
Split into fixed-size token blocks
        |
        v
Compute a chained key for each block
        |
        v
Global index: BlockKey -> BlockId
        |
        +---- hit ----> reuse an eligible cached block
        |
        +---- miss ---> allocate a block and publish its identity
        |
        v
Request-local Vec<BlockId>
        |
        v
Inference engine resolves block IDs to physical storage
```

The diagram describes the intended flow. Physical addresses, logical
block IDs, and cache keys are distinct concepts and should not be
treated as interchangeable.

### Fixed-size blocks

Each block stores KV data for a configured number of tokens. The byte
size of a block depends on the model configuration, including the number
of layers, KV heads, head dimension, and data type.

Fixed-size blocks simplify allocation and free-list management. The
allocator does not need to understand tokenization or model
computations; it manages storage and block lifecycle.

### Chained block keys

For block-sized token chunks, a key can incorporate both the current
chunk and the preceding prefix:

``` text
key_0 = hash(model/config identity, tokens[0..B])
key_1 = hash(key_0, tokens[B..2B])
key_2 = hash(key_1, tokens[2B..3B])
```

Here, `B` is the number of tokens per block.

This lets identical token prefixes produce matching block keys. Keys
must include relevant model and execution configuration, and the
implementation must account for hash collisions rather than assuming
hashes are mathematically unique.

### Request-local block vectors

A request maintains an ordered vector of logical block references:

``` rust
Vec<BlockId>
```

The vector describes the blocks used by that request. It is not itself
the KV data, and a `BlockId` is not necessarily a raw physical address.
Keeping identity separate from physical location allows the allocator to
manage storage safely.

### Allocation and reclamation

On a cache miss, the manager obtains a free block or allocates one, then
publishes the mapping. When a request finishes, it releases the
references it holds.

A block may be returned to the free pool only when no active request or
cache-retention policy still requires it. Reclamation, reference
tracking, and concurrent lookup/publication are part of the design, not
assumed to be solved by dropping a vector.

## Core invariants

-   A logical block ID refers to at most one currently allocated block.
-   A block cannot be reused while an active request still holds a
    reference to it.
-   A cache key identifies the relevant token prefix and model/execution
    configuration, not merely an individual token ID.
-   Hash collisions are checked or handled safely.
-   Request vectors preserve block order.
-   Lookup, allocation, publication, and reclamation remain consistent
    under concurrent access.
-   The allocator manages storage; it does not compute KV tensors or
    decide model semantics.

## Scope

### In scope

-   Fixed-size block pool and free list.
-   Global block index.
-   Chained block-key generation.
-   Per-request block vectors.
-   Cache-hit and cache-miss paths.
-   Reference tracking and safe block reclamation.
-   A no-sharing baseline for comparison.
-   Workload replay and benchmarks for lookup latency, allocation cost,
    memory overhead, and reuse rate.

### Out of scope initially

-   Implementing a language model or tokenizer.
-   Computing attention or KV tensors.
-   Building a frontend.
-   GPU-specific integration before the core data model and benchmarks
    are validated.

## Evaluation

The flat design should be compared against a simple baseline and, where
useful, a radix-tree-based implementation using the same workloads.

Measure at least:

-   Lookup latency for cache hits and misses.
-   Allocation and release latency.
-   Memory used by metadata and block storage.
-   Cache reuse rate under repeated and overlapping prefixes.
-   Behavior under concurrent requests and eviction.
-   Correctness under block reuse and hash-collision tests.

No performance advantage is assumed in advance. The purpose of the
project is to measure the trade-offs.

## Repository layout

The planned layout is:

``` text
.
├── README.md
├── docs/
│   ├── design.md
│   └── benchmarks.md
├── rust-engine/
│   └── ...
└── go-workload/
    └── ...
```

-   `rust-engine/`: block allocator, index, request tracking, and
    replay/benchmark logic.
-   `go-workload/`: workload generation for repeatable cache
    experiments.
-   `docs/design.md`: ownership rules, invariants, and design decisions.
-   `docs/benchmarks.md`: benchmark methodology, workloads, and measured
    results.

Adjust this layout to match the actual repository as implementation
evolves.

## Development approach

1.  Implement and test the fixed-size block allocator.
2.  Add the global index and chained block keys.
3.  Track request-owned block references in vectors.
4.  Implement release and free-list reuse.
5.  Add a no-sharing baseline and reproducible workloads.
6.  Test correctness under collisions, eviction, and concurrent
    requests.
7.  Benchmark before making optimization claims.

The central question is whether a flat index plus request-local block
vectors offers a worthwhile simplicity/performance trade-off for the
target workloads.
