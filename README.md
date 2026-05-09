# High-Performance In-Memory Join Engine

[![Build and Test](https://github.com/alechos/high-performance-join-engine/actions/workflows/build_test.yml/badge.svg)](https://github.com/alechos/high-performance-join-engine/actions)

Based on the [SIGMOD 2025 Programming Contest](https://sigmod-contest-2025.github.io/) workload and dataset, an international database systems competition. The task was to implement and optimize an in-memory hash join execution pipeline over columnar data. Execution time was reduced from **340,000 ms to 15,000 ms , a 22× speedup** through iterative pipeline redesign, custom hash table implementations, and parallelization.

## Overview

The engine executes multi-way hash join pipelines over in-memory columnar tables. Given a query plan (a tree of scan and hash join nodes) and pre-filtered input data derived from the [JOB benchmark](https://github.com/gregrahn/join-order-benchmark) over the IMDB dataset, it produces correct joined output as efficiently as possible.

The project was developed in three phases, each targeting a different bottleneck:

**Phase 1** focused on replacing `std::unordered_map` with custom hash table implementations to reduce collision overhead and improve probe efficiency.

**Phase 2** redesigned the execution pipeline itself, introducing late materialization, column-store intermediate results, and a join-specific unchained hash structure.

**Phase 3** parallelized the build and probe phases using a partitioned strategy and introduced a slab-based memory allocator to reduce allocation overhead.

## System Architecture

Input tables are stored in a paged columnar format. Intermediate results are represented as vectors of `column_t` columns containing compact `value_t` values (either integers or string references), enabling late materialization by deferring expensive string reconstruction until the final output stage.

Joins follow a build–probe model. The final design uses a directory-based unchained structure with contiguous tuple storage, improving cache locality during probing. Parallel execution is achieved through partitioned build phases, work-stealing probe phases over row chunks, and deterministic merging of per-thread outputs before producing the final columnar result.

Tested on: Intel Core i7, 16GB RAM, Ubuntu 24.04, Clang 18.

## Hash Table Implementations

Four hash table implementations were built and benchmarked as drop-in replacements for `std::unordered_map`.

**Robin Hood Hashing** uses open addressing with linear probing and probe-sequence-length (PSL) balancing. A Murmur3-style finalizer is applied on top of `std::hash` to improve key distribution, reducing maximum probe sequence lengths by up to 95% in testing. Load factor: 0.9.

**Cuckoo Hashing** maintains two separate tables with two independent hash functions, guaranteeing constant-time lookups by checking at most two positions. Displacement chains are bounded to avoid cycles; the table doubles on rehash. Load factor: 0.5.

**Hopscotch Hashing** uses a 32-bit neighborhood bitmap per bucket for fast bitwise filtering and cache-friendly access within a fixed-size window (H=32). Load factor: 0.9.

**Unchained Hashing** is a directory-based structure designed specifically for join processing. Tuples are stored in a contiguous adjacency array grouped by hash prefix using counting sort and prefix sums, maximizing cache locality during probing. Each directory entry embeds a lightweight 16-bit Bloom filter packed into a single 64-bit word alongside the range pointer, enabling fast rejection of non-matching probe keys. Fibonacci hashing was selected over CRC32 after benchmarking (CRC32: 275k ms vs Fibonacci: 240k ms).

## Optimizations

### Late Materialization and String Handling

Rather than eagerly copying all columns through the join pipeline, intermediate results propagate row identifiers and only materialize result columns at the final output stage. This is especially beneficial for VARCHAR attributes, which are expensive to copy and typically only appear in final output.

To support this, a compact 64-bit string reference (`StrRef`) was introduced that stores the table, column, page, and offset of the original string without copying bytes. The runtime value type (`value_t`) encodes INT32, StrRef, or NULL using tag bits, eliminating heavier variant-based representations and significantly reducing intermediate memory footprint.

### Column-Store Intermediate Results

Intermediate results were redesigned from row-oriented to column-oriented layout. Since join processing primarily accesses specific columns, storing intermediates as columns of `value_t` pages improves spatial locality and reduces cache misses. For dense INT32 input columns without NULLs, a direct access mode was added that bypasses intermediate copying entirely.

### Parallel Execution

The build phase uses a partitioned hash join strategy: threads independently distribute tuples into thread-local partition buffers, a prefix sum determines final memory ranges, and tuples are copied into contiguous storage in parallel. The probe phase is fully parallel with no synchronization required since partitions are disjoint and build-side storage is read-only.

Threads synchronize only at well-defined phase boundaries, keeping overhead low.

| Threads | Execution Time (ms) |
|---------|-------------------|
| 2       | 25,000            |
| 4       | 17,000            |
| **8**   | **15,000**        |

### Slab-Based Memory Allocation

A three-level slab allocator was implemented to avoid contention on the global allocator during parallel tuple collection. The first level acquires large chunks from the system, the second subdivides them into blocks, and the third allocates individual tuples from partition-local bump allocators. Memory is released in bulk after the build phase, eliminating fine-grained deallocation overhead entirely.

## Performance

**Phase 1 - Hash table comparison (row-store pipeline):**

| Implementation     | Time (ms) |
|--------------------|-----------|
| std::unordered_map | 340,000   |
| Robin Hood         | 333,000   |
| Hopscotch          | 338,000   |
| Cuckoo             | 330,000   |
| Unchained          | 240,000   |

**Phase 2 - After late materialization and column-store intermediates:**

| Implementation     | Time (ms) |
|--------------------|-----------|
| std::unordered_map | 60,000    |
| Robin Hood         | 54,000    |
| Hopscotch          | 50,000    |
| Cuckoo             | 58,000    |
| Unchained          | 37,000    |

**Phase 3 - Final (unchained + slab allocator + 8-thread parallelization):**

| Implementation     | Time (ms) |
|--------------------|-----------|
| Final              | 15,000    |

The key takeaway from the progression is that redesigning the execution model and memory layout had a far greater impact than swapping hash table variants. The jump from 240k to 37k ms came entirely from late materialization and column-store intermediates, not from the hash table itself.

## Build Instructions

Requires Linux and Clang 18.

Note: Building DuckDB from source requires ~16GB of RAM. On machines with limited memory, skip the full build and use the prebuilt cache instead.

### Full Build
```bash
# Download the IMDB dataset
./download_imdb.sh

# Build
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -Wno-dev
cmake --build build -- -j $(nproc)

# Build DuckDB database for correctness checking
./build/build_database imdb.db

# Run with correctness checking
./build/run plans.json

# Build cache for faster repeated runs (Linux x86_64)
./build/build_cache plans.json

# Or download prebuilt cache
wget http://share.uoa.gr/protected/all-download/sigmod25/sigmod25_cache_x86.tar.gz
tar -xvf sigmod25_cache_x86.tar.gz

# Run using cache
./build/fast plans.json
```


### Pre-built Cache
```bash
# Download the IMDB dataset
./download_imdb.sh

# Build
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -Wno-dev
cmake --build build -- -j $(nproc) fast

# Download prebuilt cache
wget http://share.uoa.gr/protected/all-download/sigmod25/sigmod25_cache_x86.tar.gz

# Extract cache
tar -xvf sigmod25_cache_x86.tar.gz

# Run using cache
./build/fast plans.json
```

### Selecting a Hash Algorithm

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -Wno-dev -DHASH_ALGO={algo}
cmake --build build --target fast
```

Replace `{algo}` with: `robinhood`, `cuckoo`, `hopscotch`, or `unchained` (default).

### Running Unit Tests

```bash
cmake --build build -- -j $(nproc) robinhood_tests
cmake --build build -- -j $(nproc) cuckoo_tests
cmake --build build -- -j $(nproc) hopscotch_tests
cmake --build build -- -j $(nproc) unchained_tests
cmake --build build -- -j $(nproc) slab_allocator_tests
```

## Contributors

**Ελεάνα Λύτη**
Cuckoo hashing, unchained hash table, parallel build phase setup, final report.

**Αλέξιος Σούλι**
Robin Hood hashing, Hopscotch emplace, late materialization and column-store pipeline, value_t and StrRef design, slab allocator, final optimization pass, CI/GitHub Actions setup.
