# Transport Feature Matrix {#user_transport_matrix}

This document summarizes which user-facing features are available across wirestead transport wrappers.

The matrix focuses on public wrapper and builder behavior. Internal transport implementation details may differ.

Legend:

- ✅ supported
- ⚠️ supported with transport-specific caveats
- — not applicable or not supported
- 🧪 design-only or future candidate

## Overview

`wirestead` provides a unified builder/wrapper surface across TCP, UDP, Serial, and UDS. The goal is consistent control flow, not identical protocol behavior.

Some differences are expected:

- TCP and UDS are stream transports.
- UDP is datagram-based and connectionless.
- Serial is device-oriented.
- Server wrappers expose multi-client APIs such as `send_to(...)` and `broadcast(...)`.
- Client wrappers expose peer-oriented APIs such as `send(...)` and `connected()`.

`send(...)` and `try_send(...)` report local send-path acceptance. They do not guarantee remote delivery or remote application processing.

## Transport Families

| Wrapper | Transport type | Role | Notes |
|---|---|---|---|
| `TcpClient` | TCP stream | Client | Connects to one remote endpoint |
| `TcpServer` | TCP stream | Server | Accepts multiple sessions |
| `UdpClient` | UDP datagram | Client-like | Sends to a configured remote endpoint |
| `UdpServer` | UDP datagram | Server-like | Binds a local port and tracks remote endpoints as virtual clients |
| `Serial` | Serial device | Client-like | Opens a serial device path |
| `UdsClient` | UDS stream | Client | Local IPC client |
| `UdsServer` | UDS stream | Server | Local IPC server |

For `TcpServer` and `UdsServer`, `client_id` identifies a stream session. For `UdpServer`, `client_id` identifies an observed remote endpoint, not a persistent connection.

## Lifecycle Support

| Feature | TcpClient | TcpServer | UdpClient | UdpServer | Serial | UdsClient | UdsServer |
|---|---:|---:|---:|---:|---:|---:|---:|
| Builder facade | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `.build()` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `start()` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `start_sync()` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `stop()` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `auto_start(true)` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `independent_context(true)` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `connected()` | ✅ | — | ⚠️ | — | ✅ | ✅ | — |
| `listening()` | — | ✅ | — | ✅ | — | — | ✅ |

`UdpClient::connected()` means the wrapper has an active UDP channel with a configured remote endpoint. UDP itself remains connectionless.

## Send API Support

| Feature | TcpClient | TcpServer | UdpClient | UdpServer | Serial | UdsClient | UdsServer |
|---|---:|---:|---:|---:|---:|---:|---:|
| `send(data)` | ✅ | — | ✅ | — | ✅ | ✅ | — |
| `try_send(data)` | ✅ | — | ✅ | — | ✅ | ✅ | — |
| `send_blocking(data)` | ✅ | — | ✅ | — | ✅ | ✅ | — |
| `send_line(text)` | ✅ | — | ⚠️ | — | ✅ | ✅ | — |
| `try_send_line(text)` | ✅ | — | ⚠️ | — | ✅ | ✅ | — |
| `send_move(...)` | ✅ | — | ✅ | — | ✅ | ✅ | — |
| `try_send_move(...)` | ✅ | — | ✅ | — | ✅ | ✅ | — |
| `send_shared(...)` | ✅ | — | ✅ | — | ✅ | ✅ | — |
| `try_send_shared(...)` | ✅ | — | ✅ | — | ✅ | ✅ | — |
| `send_to(client_id, data)` | — | ✅ | — | ✅ | — | — | ✅ |
| `try_send_to(client_id, data)` | — | ✅ | — | ✅ | — | — | ✅ |
| `send_to_blocking(client_id, data)` | — | ✅ | — | ✅ | — | — | ✅ |
| `broadcast(data)` | — | ✅ | — | ✅ | — | — | ✅ |
| `try_broadcast(data)` | — | ✅ | — | ✅ | — | — | ✅ |
| `send_to_line(client_id, text)` | — | ✅ | — | ⚠️ | — | — | ✅ |
| `broadcast_line(text)` | — | ✅ | — | ⚠️ | — | — | ✅ |
| `send_to_move(...)` | — | — | — | — | — | — | — |
| `send_to_shared(...)` | — | — | — | — | — | — | — |
| `broadcast_shared(...)` | — | — | — | — | — | — | — |

UDP line helpers append `"\n"` to a datagram payload. They do not add stream-style message recovery because UDP already preserves datagram boundaries.

In `Reliable` mode, `send(...)` may block under backpressure. Use `try_send(...)` for non-blocking producer loops.

In `BestEffort` mode, a successful send may still coincide with older queued payloads being dropped.

## Server API Support

| Feature | TcpServer | UdpServer | UdsServer |
|---|---:|---:|---:|
| `send_to(client_id, data)` | ✅ | ✅ | ✅ |
| `try_send_to(client_id, data)` | ✅ | ✅ | ✅ |
| `send_to_blocking(client_id, data)` | ✅ | ✅ | ✅ |
| `broadcast(data)` | ✅ | ✅ | ✅ |
| `try_broadcast(data)` | ✅ | ✅ | ✅ |
| `client_count()` | ✅ | ⚠️ | ✅ |
| `connected_clients()` | ✅ | ⚠️ | ✅ |
| `listening()` | ✅ | ✅ | ✅ |
| `max_clients(...)` | ✅ | ⚠️ | ✅ |
| `idle_timeout(...)` | ✅ | ⚠️ | ✅ |

For `UdpServer`, client identifiers represent remote endpoints observed by the server rather than persistent stream connections. Endpoint entries may be affected by datagram traffic and timeout policy.

## Backpressure Support

| Feature | TcpClient | TcpServer | UdpClient | UdpServer | Serial | UdsClient | UdsServer |
|---|---:|---:|---:|---:|---:|---:|---:|
| `backpressure_strategy(Reliable)` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `backpressure_strategy(BestEffort)` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `backpressure_threshold(bytes)` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `on_backpressure(...)` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Sender-side queue pressure | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Receiver-side flow control | TCP-level | TCP-level | — | — | device/driver-dependent | UDS stream-level | UDS stream-level |
| BestEffort drop accounting | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

Backpressure is a local sender-side queue management mechanism. It does not guarantee that the remote peer has processed the payload.

For UDP, backpressure applies only to the local sender-side queue. UDP has no receiver-side flow control.

## Runtime Statistics Support

| Statistic | Meaning |
|---|---|
| `messages_accepted` | Payloads accepted into the local send path |
| `bytes_accepted` | Bytes accepted into the local send path |
| `messages_sent` | Payloads completed by transport write success |
| `bytes_sent` | Bytes completed by transport write success |
| `messages_received` | Incoming payloads observed by wrapper/transport |
| `bytes_received` | Incoming bytes observed by wrapper/transport |
| `failed_sends` | Send attempts rejected before local acceptance |
| `dropped_messages` | Previously accepted queued payloads dropped by BestEffort |
| `dropped_bytes` | Bytes dropped from previously accepted queued payloads |
| `queued_bytes` | Current outgoing queue size |
| `pending_bytes` | Current pending queue size, where applicable |
| `max_queued_bytes` | Highest queued byte count since reset |
| `backpressure_events` | Backpressure high/low watermark transitions |

| Feature | TcpClient | TcpServer | UdpClient | UdpServer | Serial | UdsClient | UdsServer |
|---|---:|---:|---:|---:|---:|---:|---:|
| `stats()` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `reset_stats()` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Accepted counters | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Sent counters | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Received counters | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Failed send counters | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| BestEffort drop counters | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Queue gauges | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

`failed_sends` and dropped counters describe different events. `failed_sends` means the new payload was rejected before local acceptance. `dropped_messages` and `dropped_bytes` mean previously accepted queued payloads were discarded by BestEffort.

## Framing Support

| Feature | TcpClient | TcpServer | UdpClient | UdpServer | Serial | UdsClient | UdsServer |
|---|---:|---:|---:|---:|---:|---:|---:|
| `use_line_framer(...)` | ✅ | ✅ | ⚠️ | ⚠️ | ✅ | ✅ | ✅ |
| `use_packet_framer(...)` | ✅ | ✅ | ⚠️ | ⚠️ | ✅ | ✅ | ✅ |
| `use_length_prefix_framer(...)` | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ | ✅ | ✅ |
| `on_message(...)` | ✅ | ✅ | ⚠️ | ⚠️ | ✅ | ✅ | ✅ |
| `on_message_batch(...)` | ✅ | ✅ | ⚠️ | ⚠️ | ✅ | ✅ | ✅ |

UDP already preserves datagram boundaries. Framers may still be useful when the datagram payload itself contains multiple logical messages, but they are not needed to recover stream boundaries.

`use_length_prefix_framer(...)` is the one to reach for with binary payloads:
`use_packet_framer(...)` ends a frame at the first occurrence of its end pattern
and has no escaping, so a payload containing those bytes is silently truncated.
It is marked ⚠️ for Serial because it cannot resynchronise - it needs a stream
that starts on a frame boundary, which a serial line joined mid-stream does not
give you.

## Reconnect / Retry Support

| Feature | TcpClient | TcpServer | UdpClient | UdpServer | Serial | UdsClient | UdsServer |
|---|---:|---:|---:|---:|---:|---:|---:|
| `retry_interval(...)` | ✅ | — | — | — | ✅ | ✅ | — |
| `max_retries(...)` | ✅ | — | — | — | — | ✅ | — |
| `connection_timeout(...)` | ✅ | — | — | — | — | ✅ | — |
| Port retry | — | ✅ | — | — | — | — | — |
| `reopen_on_error(...)` | — | — | — | — | ✅ | — | — |
| `idle_timeout(...)` | ✅ | ✅ | — | ✅ | — | — | ✅ |
| `idle_timeout_action(...)` | ✅ | — | — | — | — | — | — |

TCP and UDS client retry settings apply to connect/reconnect behavior. TCP server port retry applies to bind/listen startup. Serial `reopen_on_error(...)` is a builder option for device reopen behavior.

## Socket Tuning Support

| Feature | TcpClient | TcpServer | UdpClient | UdpServer | Serial | UdsClient | UdsServer |
|---|---:|---:|---:|---:|---:|---:|---:|
| `tcp_no_delay(true)` | ✅ | ✅ | — | — | — | — | — |
| `keep_alive(true)` | ✅ | ✅ | — | — | — | — | — |
| `send_buffer_size(bytes)` | ✅ | ✅ | ✅ | ✅ | — | — | — |
| `receive_buffer_size(bytes)` | ✅ | ✅ | ✅ | ✅ | — | — | — |
| `broadcast(true)` | — | — | ✅ | ✅ | — | — | — |
| `multicast_group(address)` | — | — | ✅ | ✅ | — | — | — |

`multicast_group(...)` joins the group after bind, so the socket receives what
the group is sending. It is unrelated to `broadcast(...)`, which only sets
`SO_BROADCAST` for outgoing datagrams - joining a group is what a lidar or
camera stream needs on the receive side. A failed join fails `start()` rather
than being ignored, because a socket that silently did not join looks exactly
like a sensor that stopped sending.

Socket tuning options request OS socket settings. The operating system may clamp or ignore requested values depending on platform limits.

Configure socket tuning options before `build()` / `start()`. TCP server tuning is applied to accepted client session sockets.

## Platform Notes

- UDS is intended for local IPC. Platform support and path length limits vary by operating system.
- Serial support depends on OS device naming, permissions, and driver behavior.
- UDP server clients are endpoint abstractions, not persistent stream connections.
- TCP and UDS stream flow control is provided by the OS/protocol stack. It is not an application-level delivery acknowledgement.

## Planned / Design-Only APIs

The following APIs are design-stage or future candidates and are not part of the current public wrapper API:

| API | Status | Notes |
|---|---|---|
| `SendResult` | 🧪 design-only | See [SendResult API Design](../contributor/design/send_result_api.md) |
| `send_ex(...)` / `try_send_ex(...)` | 🧪 design-only | Intended to explain per-call send outcomes |
| `send_to_move(...)` / `send_to_shared(...)` | 🧪 future candidate | Server single-target move/shared result and ownership semantics need separate design |
| `broadcast_shared(...)` | 🧪 future candidate | Broadcast result aggregation semantics are still open |
| Zero-copy receive callback | 🧪 future candidate | Advanced receive-side ownership API |
| Multi-thread runtime manager | 🧪 future candidate | Future scaling API candidate |
