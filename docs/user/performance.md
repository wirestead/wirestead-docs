# Performance Guide {#user_performance}

This guide covers performance optimization strategies for `wirestead`.

---

## Table of Contents

1. Runtime Optimization
2. Memory Optimization
3. Network Optimization

---

## Runtime Optimization

### 1. Threading Model & IO Context

**Use Shared IO Context (Default)**
Wirestead defaults to a shared IO context model, which is highly efficient for most use cases.

```cpp
// ✅ GOOD: Shared context (efficient)
auto client1 = wirestead::tcp_client("server1.com", 8080).build();
auto client2 = wirestead::tcp_client("server2.com", 8080).build();
// All share ONE I/O thread - efficient!

// ❌ BAD: Independent contexts (wasteful)
auto client1 = wirestead::tcp_client("server1.com", 8080)
    .independent_context(true)  // Creates dedicated thread
    .build();
```

### 2. Async Logging

Logging can be a major bottleneck. Enable async logging for high-performance applications.

```cpp
// ✅ GOOD: Async logging (non-blocking)
wirestead::diagnostics::AsyncLogConfig config;
config.batch_size = 1000;
config.flush_interval = std::chrono::milliseconds(1000);

wirestead::diagnostics::Logger::instance().set_async_logging(true, config);
```

### 3. Non-Blocking Callbacks

Never perform heavy computation or blocking operations (like `sleep`) inside callbacks.

```cpp
// ❌ BAD: Blocking I/O thread
.on_data([](const wirestead::MessageContext& ctx) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    process_data(ctx.data());
})

// ✅ GOOD: Offload to worker thread/pool
.on_data([&thread_pool](const wirestead::MessageContext& ctx) {
    std::string payload(ctx.data());
    thread_pool.submit([payload = std::move(payload)]() {
        process_data(payload);
    });
})
```

---

## Memory Optimization

### 1. Avoid Data Copies
Use move semantics and avoid unnecessary string copies.

```cpp
// GOOD: transfer a large binary buffer into the send path
std::vector<uint8_t> large_data = generate_frame(1024 * 1024);
client->send_move(std::move(large_data));

// GOOD: Use string_view for parsing (if supported)
void parse(std::string_view msg) { ... }
```

For large binary payloads, prefer move/shared send APIs when available:

- `send_move(...)` transfers vector ownership into the send path.
- `send_shared(...)` shares immutable payload ownership with wirestead.
- `try_send_move(...)` and `try_send_shared(...)` provide non-blocking variants.

After calling `send_move(...)` or `try_send_move(...)`, treat the moved vector as consumed regardless of the return value.

### 2. Reserve Vector Capacity
When building vectors of data, always `reserve` capacity to avoid reallocations.

```cpp
std::vector<std::string> messages;
messages.reserve(1000);  // Pre-allocate
for (int i = 0; i < 1000; ++i) messages.push_back(msg);
```

---

## Network Optimization

### 1. Batching Is Automatic On Stream Transports
Since v0.9.3 the stream transports — TCP, UDS and serial — drain several queued
buffers into one scatter-gather write rather than issuing one send syscall per
message. A loop of 1000 small sends measured 65 `sendmsg` calls, not 1000
`sendto` calls.

```cpp
// Fine. The transport batches these for you.
for (int i = 0; i < 1000; ++i) client->send("msg");
```

Concatenating messages yourself to save syscalls is no longer worth doing, and
on a framed protocol it is actively wrong — joining payloads into one buffer
destroys the message boundaries the framer needs.

Two caps bound a batch: 16 buffers, and 256 KiB. Whichever binds first decides
how much collapses, which is why the gain is largest for small payloads and
tapers above 16 KiB where a single write already carries plenty. Measured on a
Jetson Orin Nano against v0.9.2, sustained throughput rose 2.3x to 5.2x for
payloads from 64 B to 4 KiB, and 1.1x to 1.4x at 16 KiB and above.

UDP is unchanged: datagrams are not a stream, so there is nothing to coalesce.

### 2. Connection Reuse
Reusing connections avoids repeated TCP handshake and connection setup overhead. For workloads with many short requests to the same peer, create the client once and reuse it across calls.

### 3. Socket Tuning
Use builder-level socket tuning first. Socket tuning is workload-dependent and does not guarantee higher throughput or lower latency for every application.

```cpp
auto client = wirestead::tcp_client("127.0.0.1", 8080)
    .tcp_no_delay(true)
    .send_buffer_size(4 * 1024 * 1024)
    .receive_buffer_size(4 * 1024 * 1024)
    .build();
```

UDP builders also support `.send_buffer_size(bytes)` and `.receive_buffer_size(bytes)`. TCP servers apply these options to accepted client session sockets.

OS-level limits may still cap the effective value:

```bash
sudo sysctl -w net.core.rmem_max=16777216  # 16 MB (OS limit)
sudo sysctl -w net.core.wmem_max=16777216  # 16 MB (OS limit)
```

### 4. Read Buffer
Distinct from the socket buffers above. `receive_buffer_size` sets the kernel's
`SO_RCVBUF`; the read buffer is how much of a filled socket buffer one read
completion can take in userspace.

```cpp
auto client = wirestead::tcp_client("127.0.0.1", 8080)
    .read_buffer_size(64 * 1024)
    .build();
```

TCP and UDS builders call it `read_buffer_size`; serial calls the same setting
`read_chunk`, a name it had first. All default to 4 KiB and clamp to
512 B – 1 MiB.

Raising it cuts read completions, and so callback dispatches, on bulk
transfers. It buys nothing for small messages that already arrive one per read.
The ceiling is deliberately far below the socket buffer ceiling because this
buffer is **per connection** — a server multiplies it by `max_connections`, so
1 MiB against the default 1024 connection limit is a gigabyte of userspace
buffers before any payload arrives.

---

## Backpressure Management {#backpressure-management}

When the data generation rate exceeds the network's transmission capacity, Wirestead's internal send queues grow. Managing this "backpressure" is critical for stability and latency.

### 1. Choosing a Strategy

Wirestead provides two strategies for handling full queues:

| Strategy | Behavior | Best For |
|:---|:---|:---|
| `Reliable` (Default) | Preserves queued outgoing data until the configured threshold is reached (default: 1 MiB), bounded by internal queue limits. | Reliable data (files, logs, commands). |
| `BestEffort` | Drops oldest data when a threshold is reached. | Real-time sensors (LiDAR, Video, Telemetry). |

### Interpreting Strategy Results

For throughput or pressure tests, distinguish between:

- **accepted throughput**: data accepted into the local send path
- **received throughput**: data observed by the receiver
- **delivery rate**: received data relative to accepted data
- **failed sends**: send calls that were rejected before being accepted
- **dropped data**: queued data discarded by a freshness-oriented policy such as `BestEffort`

In `BestEffort` mode, accepted throughput can be much higher than received throughput because older queued payloads may be dropped to keep newer data fresh. This is expected for unbounded producer pressure, but it must be monitored in real applications.

In `Reliable` mode, throughput is usually limited by queue pressure and receiver progress, but accepted data is preserved unless the queue cannot accept more data.

Use `stats()` to monitor queue pressure and drops:

- `queued_bytes`
- `pending_bytes`
- `max_queued_bytes`
- `dropped_messages`
- `dropped_bytes`
- `failed_sends`
- `backpressure_events`

When using `BestEffort`, monitor `dropped_messages` and `dropped_bytes`. A high accepted rate does not imply that every queued payload was preserved.

### 2. High-Throughput Sensors (LiDAR/Camera)

For robotics perception, processing stale data is often worse than skipping frames. Use `BestEffort` with a low threshold to prevent **Bufferbloat**.

```cpp
// 🤖 Best configuration for robotics sensors
auto lidar = wirestead::tcp_client("192.168.1.10", 2368)
    .backpressure_strategy(wirestead::base::constants::BackpressureStrategy::BestEffort)
    .backpressure_threshold(1024 * 512)  // 0.5 MB threshold
    .build();
```

Python-specific performance notes are maintained in
[Wirestead Python repository](https://github.com/wirestead/wirestead-python).

Aggressive flushing during network disconnects ensures that once reconnection
occurs, the receiver instantly gets the most recent frame instead of waiting for
a large backlog to drain.

For freshness-oriented streams, monitor dropped data and received rate. A high accepted rate alone does not mean the receiver is processing every payload.

### 3. Critical Reliable Data

For data where local queue drops are unacceptable, use `Reliable` combined with application-level send throttling.

For producer loops that must never block, use `try_send()` and handle `false` explicitly. `send()` in `Reliable` mode may block while waiting for backpressure to clear.

```cpp
bool paused = false;
client->on_backpressure([&paused](size_t queued) {
    // Pause/resume based on your application's queued-byte budget.
    if (queued > 12 * 1024 * 1024) paused = true;
    else if (queued < 4 * 1024 * 1024) paused = false;
});

// In your sender loop
if (!paused) client->send(critical_data);
```
