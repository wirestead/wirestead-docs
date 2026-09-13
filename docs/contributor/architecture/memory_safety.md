# Memory Safety Architecture {#contrib_arch_memory}

`wirestead` provides comprehensive memory safety features to ensure robust and secure applications. This document describes the memory safety architecture, features, and best practices.

---

## Table of Contents

1. Overview
2. Safe Data Handling
3. Thread-Safe State Management
4. Memory Tracking
5. AddressSanitizer Support
6. Best Practices

---

## Overview

### Memory Safety Model

| Feature             | Description                      | Performance Impact                |
| ------------------- | -------------------------------- | --------------------------------- |
| **Checked Access**  | Bounds-checked access is available through `at()` and checked slicing helpers; unchecked access remains available for performance-sensitive paths | Minimal (<1%) |
| **Type Safety**     | Safe conversions reduce UB risk  | Zero (compile-time)               |
| **Leak Detection**  | Track allocations/deallocations  | Zero (Release builds)             |
| **Thread Safety**   | Thread-safe state helpers are available where shared state requires synchronization | Minimal (~2-5%) |
| **Memory Pools**    | Reduce fragmentation             | Positive (+30% for small buffers) |

---

### Safety Levels

`wirestead` provides multiple levels of memory safety:

```mermaid
flowchart TD
    L1[Level 1: Compile-Time Safety<br/>- Type safety<br/>- RAII resource management] --> L2
    L2[Level 2: Runtime Safety (Release)<br/>- Bounds checking<br/>- Safe conversions<br/>- Thread synchronization] --> L3
    L3[Level 3: Debug Safety (Debug)<br/>- Memory tracking<br/>- Allocation tracking<br/>- Leak detection] --> L4
    L4[Level 4: Sanitizers (Optional)<br/>- AddressSanitizer<br/>- ThreadSanitizer<br/>- UndefinedBehaviorSanitizer]
```

---

## Safe Data Handling

### SafeDataBuffer

Immutable, type-safe buffer wrapper that owns copied data:

```cpp
#include "wirestead/memory/memory_tracker.hpp"
#include "wirestead/memory/safe_span.hpp"
#include "wirestead/memory/safe_data_buffer.hpp"

// Create from existing data
wirestead::memory::SafeDataBuffer from_vec(std::vector<uint8_t>{1, 2, 3});
wirestead::memory::SafeDataBuffer from_str(std::string("hello"));

// Read access
auto span = from_vec.as_span();      // Non-owning view
auto byte = from_vec.at(1);          // Bounds-checked
auto unchecked = from_vec.data()[0]; // Pointer access
```

`SafeDataBuffer` owns its storage and provides checked `at()` access. Pointer access and unchecked indexing should be used only when the caller already controls bounds.

---

### Features

#### 1. Construction Validation

- Rejects null pointers when size > 0
- Rejects buffers >100MB
- Copies incoming data into owned storage

---

#### 2. Safe Type Conversions

Utility functions prevent undefined behavior:

```cpp
#include "wirestead/base/common.hpp"

using namespace wirestead::base::safe_convert;

// Safe uint8_t* to string conversion
const uint8_t* data = ...;
size_t size = ...;
std::string str = uint8_to_string(data, size);  // Null-check included

// Safe string to byte conversion (owning copy)
std::string input = "Hello";
std::vector<uint8_t> bytes = string_to_uint8(input);

// Non-owning view over the same bytes, no allocation
auto [ptr, len] = string_to_bytes(input);
```

---

#### 3. Memory Validation

Comprehensive validation of data integrity:

```cpp
// Validate buffer state
if (buffer.is_valid()) {
    // Buffer is in consistent state
    process_data(buffer.data(), buffer.size());
} else {
    // Buffer corrupted or uninitialized
    handle_error();
}

// Explicit validation (throws on oversize)
buffer.validate();
```

**Mutation model:** Data is populated at construction; no in-place `write/append` helpers exist. Use `clear()/resize()/reserve()` only when you control the backing data size.

---

### Safe Span

Lightweight, non-owning view of contiguous data:

```cpp
#include "wirestead/memory/safe_span.hpp"

void process_data(wirestead::memory::ConstByteSpan data) {
    // Iteration (operator[] is unchecked)
    for (size_t i = 0; i < data.size(); i++) {
        uint8_t byte = data[i];
        process_byte(byte);
    }

    // Bounds-checked access
    uint8_t checked = data.at(0);
    auto subset = data.subspan(1, 2);  // throws on invalid ranges
}

// Usage
std::vector<uint8_t> buffer = {1, 2, 3, 4, 5};
process_data(wirestead::memory::ConstByteSpan(buffer));
```

**Features:**

- No ownership (lightweight)
- `at()/subspan()` are checked; `operator[]` is unchecked
- Project-local span abstraction retained for API stability
- Zero overhead in release builds

`SafeSpan::at()` and checked slicing helpers validate ranges. `operator[]` follows `std::span` semantics and is unchecked.

---

## Thread-Safe State Management

### ThreadSafeState

Read-write lock based state management:

```cpp
#include "wirestead/concurrency/thread_safe_state.hpp"

enum class ConnectionState {
    Closed,
    Connecting,
    Connected,
    Error
};

wirestead::concurrency::ThreadSafeState<ConnectionState> state(ConnectionState::Closed);

// Thread 1: Write
state.set_state(ConnectionState::Connecting);

// Thread 2: Read (concurrent safe)
ConnectionState current = state.state();

// Thread 3: Conditional update
bool updated = state.compare_and_set(
    ConnectionState::Connecting,  // Expected
    ConnectionState::Connected     // New value
);
```

---

### AtomicState

Lock-free atomic state operations:

```cpp
#include "wirestead/concurrency/thread_safe_state.hpp"

AtomicState<int> counter(0);

// Atomic increment (thread-safe)
counter.fetch_add(1);

// Atomic compare-and-swap
int expected = 10;
bool success = counter.compare_exchange_strong(expected, 20);
```

**Use when:**

- High contention scenarios
- Low latency required
- Simple atomic types (int, bool, etc.)

---

### ThreadSafeCounter

Thread-safe counter with atomic operations:

```cpp
ThreadSafeCounter counter;

// Thread 1
counter.increment();

// Thread 2
counter.decrement();

// Thread 3
size_t value = counter.get();
```

---

### ThreadSafeFlag

Condition variable supported flags:

```cpp
wirestead::concurrency::ThreadSafeFlag ready_flag;

// Thread 1: Wait for flag. The timeout is not optional - the wait returns
// when it elapses whether or not the flag was set, so re-check afterwards.
ready_flag.wait_for_true(std::chrono::seconds(5));
if (ready_flag.get()) {
    std::cout << "Ready!" << std::endl;
}

// Thread 2: Set flag
std::this_thread::sleep_for(std::chrono::seconds(1));
ready_flag.set();  // Unblocks waiting thread

// Thread 3: Check without blocking
if (ready_flag.get()) {
    // Flag is set
}
```

---

### Thread Safety Summary

| Primitive             | Lock-Free | Blocking | Use Case                 |
| --------------------- | --------- | -------- | ------------------------ |
| **ThreadSafeState**   | No        | No       | Complex state management |
| **AtomicState**       | Yes       | No       | Simple atomic types      |
| **ThreadSafeCounter** | Yes       | No       | Counters, statistics     |
| **ThreadSafeFlag**    | No        | Yes      | Synchronization, signals |

---

## Memory Tracking

### Built-in Memory Tracking

Enable for debugging and development:

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DWIRESTEAD_ENABLE_MEMORY_TRACKING=ON
```

---

### Features

#### 1. Allocation Tracking

Monitor all memory allocations and deallocations:

```cpp
#include "wirestead/memory/memory_tracker.hpp"

// Tracking is not automatic - nothing hooks global new/delete. An allocation
// is only recorded if it is reported through the tracker, which is what the
// MEMORY_TRACK_* macros do (they compile to nothing unless the build sets
// WIRESTEAD_ENABLE_MEMORY_TRACKING).
auto* data = new uint8_t[1024];
MEMORY_TRACK_ALLOCATION(data, 1024);
MEMORY_TRACK_DEALLOCATION(data);
delete[] data;

// Query statistics
wirestead::memory::MemoryTracker::MemoryStats stats =
    wirestead::memory::MemoryTracker::instance().stats();
std::cout << "Total allocations: " << stats.total_allocations << std::endl;
std::cout << "Total deallocations: " << stats.total_deallocations << std::endl;
std::cout << "Current usage: " << stats.current_bytes_allocated << " bytes" << std::endl;
```

---

#### 2. Leak Detection

Identify potential memory leaks:

```cpp
// At program exit, check for leaks
using wirestead::memory::MemoryTracker;

auto stats = MemoryTracker::instance().stats();

if (stats.total_allocations != stats.total_deallocations) {
    size_t leaked = stats.total_allocations - stats.total_deallocations;
    std::cerr << "⚠️ Memory leak detected: "
              << leaked << " allocations not freed" << std::endl;

    // Get detailed report
    MemoryTracker::instance().print_leak_report();
}
```

**Output example:**

```
=== Memory Leak Report ===
Found 2 potential memory leaks:
Leaked: 1024 bytes at 0x5561f0a2c2a0 allocated in src/session.cc:87 (start)
Leaked: 1024 bytes at 0x5561f0a2c6b0 allocated in src/session.cc:87 (start)
Total leaked bytes: 2048
```

`print_memory_report()` prints the counter block instead — every `MemoryStats`
field, followed by one line per still-live allocation.

---

#### 3. Performance Monitoring

Track memory usage patterns:

```cpp
// One snapshot carries every counter; there are no per-metric getters.
auto stats = wirestead::memory::MemoryTracker::instance().stats();

std::cout << "Peak memory: " << stats.peak_bytes_allocated << " bytes" << std::endl;
std::cout << "Allocations: " << stats.total_allocations << std::endl;
```

---

#### 4. Debug Reports

Detailed memory usage reports:

```cpp
using wirestead::memory::MemoryTracker;

// Print detailed reports to stdout
MemoryTracker::instance().print_memory_report();
MemoryTracker::instance().print_leak_report();

// Or route them through the logger (recommended for production)
MemoryTracker::instance().log_memory_report();
MemoryTracker::instance().log_leak_report();
```

---

### Zero Overhead in Release

Memory tracking has **zero overhead** in Release builds:

```cpp
// In Release builds (WIRESTEAD_ENABLE_MEMORY_TRACKING=OFF):
// All tracking calls are compiled out
// No runtime overhead
// No memory overhead
```

---

## AddressSanitizer Support

### Enable AddressSanitizer

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DWIRESTEAD_ENABLE_SANITIZERS=ON

cmake --build build -j
```

---

### What ASan Detects

AddressSanitizer detects:

- ✅ **Use-after-free**
- ✅ **Heap buffer overflow**
- ✅ **Stack buffer overflow**
- ✅ **Use-after-return**
- ✅ **Memory leaks**
- ✅ **Double-free**
- ✅ **Invalid free**

---

### Running with ASan

```bash
# Run application
./my_app

# Run tests
cd build
ctest --output-on-failure
```

**ASan output example:**

```
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000050
WRITE of size 4 at 0x602000000050 thread T0
    #0 0x7f8a4b5c6d42 in main
    #1 0x7f8a4b3c7082 in __libc_start_main
```

---

### Performance Impact

- **Runtime overhead:** ~2-3x slowdown
- **Memory overhead:** ~2x memory usage
- **Only for testing** - not for production

---

## Best Practices

### 1. Buffer Management

#### ✅ DO

```cpp
// Use SafeDataBuffer to wrap existing data
std::vector<uint8_t> raw(data, data + len);
SafeDataBuffer buffer(raw);
process_buffer(buffer);

// Use safe_span for views
void process(safe_span<const uint8_t> data) {
    // ...
}
```

#### ❌ DON'T

```cpp
// Avoid raw pointers without bounds checking
uint8_t* buf = new uint8_t[size];
memcpy(buf, data, len);  // No bounds check!

// Avoid pointer arithmetic without validation
uint8_t* ptr = buffer + offset;  // May go out of bounds
```

---

### 2. Type Conversions

#### ✅ DO

```cpp
// Use safe conversion utilities
std::string str = wirestead::base::safe_convert::uint8_to_string(data, size);
std::vector<uint8_t> bytes = wirestead::base::safe_convert::string_to_uint8(str);
```

#### ❌ DON'T

```cpp
// Avoid unsafe casts
std::string str = reinterpret_cast<const char*>(data);  // May not be null-terminated
uint8_t* bytes = const_cast<uint8_t*>(str.data());  // Removes const incorrectly
```

---

### 3. Thread Safety

#### ✅ DO

```cpp
// Use thread-safe primitives
ThreadSafeState<State> state;
ThreadSafeCounter counter;

// Let wirestead handle synchronization
client->send(data);  // Already thread-safe
```

#### ❌ DON'T

```cpp
// Avoid manual locking of wirestead internals
// std::lock_guard<std::mutex> lock(internal_mutex);  // DON'T DO THIS

// Avoid shared state without synchronization
// state = NewState;  // Race condition!
```

---

### 4. Memory Tracking

#### ✅ DO

Enable in Debug builds:

```bash
cmake -DCMAKE_BUILD_TYPE=Debug -DWIRESTEAD_ENABLE_MEMORY_TRACKING=ON
```

Check for leaks at exit:

```cpp
auto stats = wirestead::memory::MemoryTracker::instance().stats();
assert(stats.total_allocations == stats.total_deallocations);
```

#### ❌ DON'T

```cpp
// Don't enable in Release builds (performance)
// cmake -DCMAKE_BUILD_TYPE=Release -DWIRESTEAD_ENABLE_MEMORY_TRACKING=ON

// Don't ignore leak reports

```

---

### 5. Sanitizers

#### ✅ DO

```cpp
// Use during development and testing
cmake -DCMAKE_BUILD_TYPE=Debug -DWIRESTEAD_ENABLE_SANITIZERS=ON

// Run full test suite with sanitizers
ctest --output-on-failure

// Fix all reported issues
```

#### ❌ DON'T

```cpp
// Never deploy with sanitizers enabled
// (2-3x performance penalty)

// Don't suppress sanitizer warnings without investigation
```

---

## Memory Safety Benefits

### Prevents Common Vulnerabilities

| Vulnerability       | Traditional C++ | wirestead                    |
| ------------------- | --------------- | -------------------------- |
| **Buffer Overflow** | Possible        | Prevented (bounds checked) |
| **Use-After-Free**  | Possible        | Detected (ASan)            |
| **Memory Leak**     | Possible        | Detected (tracking)        |
| **Data Race**       | Possible        | Prevented (thread-safe)    |
| **Type Confusion**  | Possible        | Prevented (type-safe)      |

---

### Performance

| Feature              | Debug Overhead | Release Overhead   |
| -------------------- | -------------- | ------------------ |
| **Bounds Checking**  | ~5%            | <1%                |
| **Memory Tracking**  | ~10%           | 0% (disabled)      |
| **Thread Safety**    | ~5%            | ~2-5%              |
| **Memory Pools**     | 0%             | Negative (faster!) |
| **AddressSanitizer** | ~200-300%      | N/A (not used)     |

---

## Testing Memory Safety

### Unit Tests

```bash
# Run memory safety tests
ctest --test-dir build --output-on-failure -L unit_memory_fast
```

**Tests cover:**

- Buffer bounds checking
- Memory leak detection
- Safe type conversions
- Thread safety
- Memory pool correctness

---

### Integration Tests

```bash
# Run with AddressSanitizer
cmake -S . -B build -DWIRESTEAD_ENABLE_SANITIZERS=ON
cmake --build build
cd build && ctest
```

---

### Continuous Integration

All memory safety features are tested in CI/CD:

- ✅ Memory tracking enabled
- ✅ AddressSanitizer enabled
- ✅ ThreadSanitizer enabled (selected tests)
- ✅ Valgrind memcheck

See [Testing Guide](../testing.md) for details.

---

## Next Steps

- [Runtime Behavior](runtime_behavior.md) - Threading and execution model
- [System Overview](README.md) - High-level architecture
- [Testing Guide](../testing.md) - Memory safety testing
