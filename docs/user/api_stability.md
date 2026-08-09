# API Stability Policy {#user_api_stability}

This document defines which parts of wirestead are considered user-facing public API, which parts are supported but advanced, and which parts are internal implementation details.

Before v1.0, wirestead aims to preserve source compatibility within the same minor release line where practical, but minor releases may still refine APIs when needed to improve clarity, safety, or runtime predictability.

## Stable Public Surface

The primary public entry point is:

- `<wirestead/wirestead.hpp>`

The following facade aliases and builder functions are considered user-facing API:

- `wirestead::tcp_client(...)`
- `wirestead::tcp_server(...)`
- `wirestead::udp_client(...)`
- `wirestead::udp_server(...)`
- `wirestead::serial(...)`
- `wirestead::uds_client(...)`
- `wirestead::uds_server(...)`

The following wrapper aliases are also user-facing:

- `wirestead::TcpClient`
- `wirestead::TcpServer`
- `wirestead::UdpClient`
- `wirestead::UdpServer`
- `wirestead::Serial`
- `wirestead::UdsClient`
- `wirestead::UdsServer`

The following callback context types are user-facing:

- `wirestead::MessageContext`
- `wirestead::ConnectionContext`
- `wirestead::ErrorContext`

The following diagnostics type is user-facing:

- `wirestead::RuntimeStats`

## Supported But Advanced Headers

These headers are supported, but most users should prefer including `<wirestead/wirestead.hpp>`:

- `wirestead/builder/*`
- `wirestead/wrapper/*`
- `wirestead/wrapper/context.hpp`

Use these headers directly only when you need narrower includes or lower-level wrapper control.

## Internal Or Not Source-Stable Before v1.0

The following areas are implementation details or advanced internals and may change before v1.0:

- `wirestead/transport/*`
- `wirestead/interface/*`
- `wirestead/factory/*`
- `wirestead/concurrency/*`
- `wirestead/config/*`
- `wirestead/memory/*`
- Boost adapter headers

Some memory utilities are documented to explain callback data ownership and safety rules. They should still be treated as advanced APIs unless they are exposed through the public facade or callback context types.

## Source Compatibility

Before v1.0, wirestead aims to preserve source compatibility within the same minor release line where practical.

However, minor releases may still refine APIs when needed to improve:

- API clarity
- runtime predictability
- memory safety
- packaging correctness
- cross-platform behavior

## Builder Return Convention

Every builder setter returns `Derived&` and mutates the builder in place, so a
chain and a sequence of separate statements do the same thing:

```cpp
auto b = wirestead::tcp_client("127.0.0.1", 8080);
b.on_data(handler);                     // b is still usable
auto client = b.on_error(eh).build();
```

Before v0.9.3 the handler setters — `on_data`, `on_data_batch`, `on_message`,
`on_message_batch`, `on_error` — instead returned a *new* builder of a
different type and left the original moved-from, while the remaining setters
returned a reference. That split is gone, along with the `BuilderState`
template parameter and the `Rebind` alias behind it.

Chained code is unaffected, which is how every example in these docs is
written. Two forms needed updating in v0.9.3: spelling a builder type with its
state argument (`TcpClientBuilder<>` became `TcpClientBuilder`), and capturing
the result of a handler setter, which now hands back a reference to the same
builder rather than a new object. The `*BuilderDefault` aliases still name the
builders.

## ABI Compatibility

C++ ABI compatibility is not guaranteed before v1.0.

Applications and binary packages should be rebuilt against the exact wirestead version they consume. Source compatibility is the primary compatibility goal before v1.0.

## Design-Only APIs

`SendResult` is currently documented as a design proposal and is not part of
the runtime public API unless explicitly implemented in a future release.

Existing `send(...)` and `try_send(...)` APIs remain the runtime APIs for send
acceptance. Use `RuntimeStats` for cumulative diagnostics such as failed sends,
dropped messages, dropped bytes, and queue pressure.

## Deprecation Policy

Deprecated APIs should normally remain available for at least one minor release before removal.

Exceptions may be made when an API is:

- unsafe
- broken
- misleading
- preventing correct runtime behavior
- incompatible with the documented public API contract

## Python Bindings

Python bindings are maintained separately in the Wirestead Python repository.

This API stability policy applies to the C++ core repository. Python package compatibility is documented in the Wirestead Python repository.

## Recommended Include Policy

Most applications should include:

```cpp
#include <wirestead/wirestead.hpp>
```

Direct builder or wrapper includes are supported for advanced users, but the umbrella header is the recommended stable entry point for application code.

Move/shared buffer send APIs are user-facing advanced APIs intended for large binary payloads.

Socket tuning builder options are user-facing advanced APIs intended for workload-specific latency, throughput, and tail-behavior tuning.

Idle timeout builder options (`idle_timeout(...)`, `idle_timeout_action(...)`) are user-facing APIs for application-level stale-session policy. They are supported on TCP client, TCP server, UDP server, and UDS server.
