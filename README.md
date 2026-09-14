# FastKVS

## Thread-Safe In-Memory Key-Value Store in C++17

FastKVS is a C++17 in-memory key-value store designed to explore practical
systems-programming problems around concurrent access, synchronization,
LRU caching, persistence, thread-pool execution, benchmarking, and
performance analysis.

The implementation keeps the core components separate:

- `KVStore` — public key-value API and synchronization
- `LRUCache` — access-order tracking and eviction support
- `Persistence` — file-based save/load
- `ThreadPool` — reusable worker threads and task execution
- Benchmarking — single-thread and multi-thread performance analysis
- Tests — functional verification of core KVStore behavior

The project intentionally exposes its current design trade-offs and
scalability limitations rather than hiding them behind benchmark numbers.

---

## Key Features

- C++17 implementation
- O(1) average `PUT`, `GET`, and `REMOVE`
- LRU eviction
- Thread-safe KVStore operations
- `std::shared_mutex` synchronization
- File-based persistence
- Reusable thread pool
- Single-thread benchmarking
- Multi-thread benchmarking
- Mixed 80% GET / 20% PUT workload
- CMake-based build
- Functional testing

---

## Architecture

```text
                         +------------------+
                         |     Client       |
                         +--------+---------+
                                  |
                                  v
                         +------------------+
                         |     KVStore      |
                         |------------------|
                         | PUT              |
                         | GET              |
                         | REMOVE           |
                         | GET_ALL_KEYS     |
                         | shared_mutex     |
                         +--------+---------+
                                  |
                    +-------------+-------------+
                    |                           |
                    v                           v
             +-------------+             +---------------+
             |   kv_map    |             |   LRUCache    |
             |-------------|             |---------------|
             | unordered_  |             | unordered_map |
             | map         |             | + std::list   |
             +-------------+             +---------------+
                    |                           |
                    +-------------+-------------+
                                  |
                                  v
                         +------------------+
                         |   Persistence    |
                         |------------------|
                         | Save / Load      |
                         | File I/O         |
                         +------------------+

                         +------------------+
                         |    ThreadPool    |
                         |------------------|
                         | Worker Threads   |
                         | Task Queue       |
                         +------------------+
```

---

## Component Responsibilities

### KVStore

KVStore is the primary public interface.

The current API provides:

- `put(key, value)`
- `get(key, value)`
- `remove(key)`
- `get_all_keys()`

Internally, KVStore maintains:

- `std::unordered_map<std::string, std::string> kv_map`
- `std::shared_mutex kv_mutex`
- `LRUCache lru`

The `kv_map` stores the actual key-value data. The `LRUCache` separately
maintains the access ordering used for eviction.

### LRUCache

LRUCache maintains the least-recently-used ordering using:

```text
std::unordered_map
        +
std::list
```

The map stores the key and its corresponding list iterator:

```text
key -> list iterator
```

The list maintains access order:

```text
Front                         Back
  |                             |
  v                             v
Most Recently Used      Least Recently Used
```

The primary operations are:

- `touch()`
- `evict()`
- `remove()`

These operations are O(1) with respect to the LRU data structures used by
the implementation.

---

## Data Structures

FastKVS uses two hash-map structures for different responsibilities.

**KVStore Data Map**

```cpp
std::unordered_map<std::string, std::string>
```

Purpose: `Key -> Value`. Provides average O(1) lookup, insertion, and removal.

**LRU Tracking Map**

```cpp
std::unordered_map<std::string, std::list<std::string>::iterator>
```

Purpose: `Key -> Position in LRU list`

**LRU List**

```cpp
std::list<std::string>
```

Purpose: Track key access order. This combination allows cache ordering to be
updated without scanning the complete list.

### Operation Complexity

| Operation    | Expected Complexity |
| ------------ | ------------------- |
| `put()`      | O(1) average        |
| `get()`      | O(1) average        |
| `remove()`   | O(1) average        |
| LRU `touch()`  | O(1)              |
| LRU `evict()`  | O(1)              |
| LRU `remove()` | O(1)              |

The hash-map operations are average-case O(1), assuming a suitable hash
distribution.

---

## LRU Eviction

The KVStore maintains a configured capacity.

When inserting a new key and the store is already at capacity:

```text
                 PUT(new_key)
                      |
                      v
             Store at capacity?
                 /          \
               No            Yes
               |              |
               v              v
             Insert      Is key new?
                              |
                         +----+----+
                         |         |
                        No        Yes
                        |          |
                        |          v
                        |      LRU evict()
                        |          |
                        |          v
                        |      Remove key
                        |      from kv_map
                        |          |
                        +----+-----+
                             |
                             v
                       Insert / Update
                             |
                             v
                       lru.touch()
```

The least recently used key is maintained at the back of the LRU list and is
selected for eviction.

---

## Concurrency Design

FastKVS uses a single coarse-grained `std::shared_mutex` per KVStore instance.

```text
                 +------------------+
Thread 1 ------> |                  |
Thread 2 ------> |    kv_mutex      | ----> KVStore state
Thread 3 ------> |                  |
                 +------------------+
```

### `put()`

Uses an **exclusive lock** because it modifies `kv_map` and LRU state.

### `get()`

Uses an **exclusive lock** because a successful `get()`:

1. Looks up the key
2. Copies the value
3. Updates the LRU ordering

Therefore the current implementation cannot treat `get()` as a pure read
operation.

```text
GET
 |
 v
unique_lock
 |
 +--> lookup kv_map
 |
 +--> copy value
 |
 +--> lru.touch(key)
 |
 v
unlock
```

### `remove()`

Uses an **exclusive lock** because it modifies both the key-value map and LRU
state.

### `get_all_keys()`

`get_all_keys()` is different from the mutating operations.

It uses a **shared lock**:

```text
shared_lock
    |
    v
iterate kv_map
    |
    v
return vector of keys
```

This API is also used by the persistence layer while saving the store.

---

## Thread Pool

FastKVS includes a reusable thread pool implemented using standard C++
threading primitives.

The implementation maintains:

- `std::vector<std::thread>`
- `std::queue<std::function<void()>>`
- `std::mutex`
- `std::condition_variable`

```text
                         +----------------+
                         |   Task Queue   |
                         +-------+--------+
                                 |
                         condition_variable
                                 |
             +-------------------+-------------------+
             |                   |                   |
             v                   v                   v
         Worker 1            Worker 2            Worker N
             |                   |                   |
             +-------------------+-------------------+
                                 |
                                 v
                              Task()
```

Worker threads wait on a condition variable until work becomes available or
the pool is being stopped. The destructor signals shutdown and joins all
worker threads.

---

## Persistence

FastKVS provides simple file-based persistence through the `Persistence` class.

The current interface provides:

- `save(const KVStore&)`
- `load(KVStore&)`

### Save

```text
KVStore
   |
   v
get_all_keys()
   |
   v
get(key, value)
   |
   v
std::ofstream
   |
   v
data file
```

The file format is:

```
key value
key value
key value
```

### Load

```text
File
 |
 v
std::ifstream
 |
 v
key + value
 |
 v
KVStore::put()
```

Persistence is synchronous and uses blocking file I/O.

### Persistence Limitations

The current persistence implementation is intentionally simple. It does not
currently provide:

- Write-Ahead Logging
- Crash recovery
- Atomic snapshots
- Checksums
- Transactional persistence
- Binary serialization
- Asynchronous persistence

The current file format is whitespace-delimited and is intended as a simple
persistence mechanism rather than a production-grade serialization format.

---

## Benchmarking

The benchmark implementation is located at `benchmarks/benchmark_kvstore.cpp`.

The benchmark configuration uses:

```
NUM_OPS     = 200,000
WARMUP_OPS  = 10,000
```

The benchmark measures:

- Single-thread PUT
- Single-thread GET
- Multi-thread PUT
- Mixed workload (80% GET / 20% PUT)

Multi-thread tests are executed with 1, 2, 4, and 8 threads.

Throughput is calculated as:

```
Throughput = (Operations × 1000) / Time(ms)
```

---

## Benchmark Results

### Single Thread

| Operation | Time (ms) | Throughput (ops/sec) |
| --------- | --------- | -------------------- |
| PUT       | 314       | 636,943              |
| GET       | 146       | 1,369,860            |

### Multi-Thread PUT

| Threads | Time (ms) | Throughput (ops/sec) |
| ------- | --------- | -------------------- |
| 1       | 523       | 382,409              |
| 2       | 1178      | 169,779              |
| 4       | 2118      | 94,428.7             |
| 8       | 2737      | 73,072.7             |

### Mixed Workload — 80% GET / 20% PUT

| Threads | Time (ms) | Throughput (ops/sec) |
| ------- | --------- | -------------------- |
| 1       | 595       | 336,134              |
| 2       | 1783      | 112,170              |
| 4       | 1494      | 133,869              |
| 8       | 2501      | 79,968               |

Detailed benchmark information: [docs/benchmark_results.md](docs/benchmark_results.md)

---

## Benchmark Analysis

The benchmark results demonstrate an important characteristic of the current
design. Increasing the number of worker threads does not automatically
increase throughput.

For the pure PUT workload:

```
1 thread  -> 382,409 ops/sec
2 threads -> 169,779 ops/sec
4 threads ->  94,428 ops/sec
8 threads ->  73,072 ops/sec
```

The main architectural reason is the single coarse-grained mutex protecting
the KVStore. Multiple worker threads can execute tasks concurrently, but
access to the shared KVStore state is serialized by `kv_mutex`.

In addition, `get()` requires exclusive locking because it updates LRU
ordering.

This makes the current implementation useful for studying:

- Lock contention
- Critical-section size
- Thread-pool overhead
- Read/write workload behavior
- Scalability limits

The benchmark therefore measures not only raw throughput, but also exposes
where the current architecture stops scaling.

---

## Testing

The current test program is located at `test/lru_evict_test.cpp`.

The test verifies:

- `PUT`
- `GET`
- Value correctness
- LRU eviction
- `REMOVE`

The current test scenario uses a store with capacity 2:

```
PUT A
PUT B
GET A
GET B
PUT C  -> evicts A (least recently used)
```

The test verifies that A was evicted, B and C exist, and B can be removed.

Run the test executable:

```bash
./test
```

---

## Build

### Requirements

- C++17
- CMake
- GCC or Clang
- POSIX threads
- Linux / Unix-like environment

### Build with CMake

```bash
mkdir -p build
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build . -j"$(nproc)"
```

---

## Running

Run the example application:

```bash
./fastkv
```

Run benchmarks:

```bash
./benchmark
```

Run tests:

```bash
./test
```

---

## Example Application

The example application is implemented in `src/main.cpp`.

It demonstrates:

1. Creating a KVStore
2. Loading persisted data
3. Creating a ThreadPool
4. Enqueuing concurrent PUT operations
5. Enqueuing concurrent GET operations
6. Saving the KVStore

Configuration used:

- KVStore capacity: 1000
- ThreadPool workers: 4

```text
                  Application
                       |
                       v
                Persistence
                    load()
                       |
                       v
                  KVStore
                       |
                       v
                  ThreadPool
                  /        \
                 /          \
              PUT            GET
               |              |
               +------+-------+
                      |
                      v
                   KVStore
                      |
                      v
                 Persistence
                    save()
```

---

## Current Limitations

### 1. Coarse-Grained Locking

One `std::shared_mutex` protects the complete KVStore. This limits
scalability under highly concurrent workloads.

### 2. Exclusive GET

A successful `get()` modifies LRU ordering and therefore requires an
exclusive lock.

### 3. Synchronous Persistence

File persistence is synchronous and uses blocking I/O.

### 4. Simple Persistence Format

The current format stores `key value` pairs and does not provide a robust
serialization or crash-consistency protocol.

### 5. No Distributed Architecture

FastKVS is currently an in-process key-value store. It does not provide
replication, sharding, leader election, a network protocol, or distributed
consensus.

### 6. Time-Based Wait in Example Application

The example application currently uses:

```cpp
std::this_thread::sleep_for(std::chrono::seconds(2));
```

to allow queued work to complete before persistence. This is suitable as a
simple demonstration, but it is not a robust task completion mechanism. A
production implementation should use futures, counters, or another explicit
synchronization mechanism rather than a fixed sleep duration.

---

## Design Trade-offs

FastKVS intentionally favors a simple and understandable concurrency model.

### Coarse-Grained Synchronization

**Advantages:** simple ownership model, straightforward correctness reasoning,
low implementation complexity.

**Disadvantages:** contention under concurrent access, limited scalability,
all operations compete for the same synchronization boundary.

### Exclusive GET for LRU Ordering

**Advantages:** accurate access ordering, simple LRU implementation, O(1)
update.

**Disadvantages:** GET becomes a write-like operation; shared reads cannot
proceed concurrently.

This trade-off is one of the main performance characteristics studied by the
benchmark suite.

---

## Future Improvements

Potential improvements include:

- Lock striping
- Sharded KVStore
- Per-shard LRU
- Reduced critical sections
- Optional LRU updates on GET
- Asynchronous persistence
- Crash-safe persistence
- Metrics instrumentation
- Latency percentile measurements
- Improved task completion primitives
- Network-facing API
- Replication

A possible sharded architecture would partition the key space across multiple
independent synchronization domains:

```text
                         KVStore
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          Shard 0        Shard 1        Shard N
             |              |              |
          kv_map          kv_map          kv_map
             |              |              |
            LRU            LRU            LRU
             |              |              |
           Lock           Lock           Lock
```

This would reduce contention by allowing unrelated keys to be processed under
different synchronization domains.

---

## Project Structure

```
fastkvs/
|
├── README.md
├── CMakeLists.txt
|
├── docs/
│   ├── architecture.md
│   ├── design.md
│   └── benchmark_results.md
|
├── src/
│   ├── main.cpp
│   ├── kvstore.h
│   ├── kvstore.cpp
│   ├── lru_cache.h
│   ├── lru_cache.cpp
│   ├── thread_pool.h
│   ├── thread_pool.cpp
│   ├── persistence.h
│   └── persistence.cpp
|
├── benchmarks/
│   └── benchmark_kvstore.cpp
|
└── test/
    └── lru_evict_test.cpp
```

---

## Documentation

Additional design information is available in:

- [docs/architecture.md](docs/architecture.md) — KVStore architecture, synchronization model, LRU behavior, and future architectural improvements
- [docs/design.md](docs/design.md) — Data structures, complexity, persistence, testing, benchmarking, and current limitations
- [docs/benchmark_results.md](docs/benchmark_results.md) — Recorded benchmark results and performance observations

---

## Engineering Focus

FastKVS is primarily a systems-programming and performance-engineering project.

It demonstrates practical work with:

- Modern C++17
- STL — `std::unordered_map`, `std::list`
- RAII
- Multithreading — `std::thread`, `std::mutex`, `std::shared_mutex`, condition variables
- Thread pools
- LRU caching
- File I/O and persistence
- CMake
- Benchmarking and performance analysis
- Functional testing

The project focuses not only on implementing a working key-value store, but
also on understanding the engineering trade-offs involved in concurrency,
cache management, persistence, and scalability.

---

## Author

**Sai Krishna Varanasi**

C++ Systems Engineering | Linux | Multithreading | Performance Engineering
