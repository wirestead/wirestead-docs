# Wirestead Documentation {#docs_index}

**Unified async communication for modern C++20.**

Serial · TCP · UDP · UDS

Choose the guide that matches your role.

> **Where documentation lives.** This site answers *how do I use this*: quick
> start, installation, the API guide, the transport matrix, troubleshooting. The
> [core repository's `docs/`](https://github.com/wirestead/wirestead/tree/main/docs)
> answers *how does this behave, and why* - the callback data lifetime, the error
> model, the threat model, what a tuning knob costs. API signatures are in
> neither: Doxygen generates those from the headers, so they cannot go stale.
>
> A few documents exist in both because the core repository has to stand on its
> own; the versions there are minimal and the ones here are the extended ones.
> Neither should grow into the other.

---

## 📖 For Library Users

You are building an application using wirestead.

→ **[User Guide](user/index.md)**

| Document | What it covers |
|----------|----------------|
| [Quick Start](user/quickstart.md) | First working client in minutes |
| [Installation](user/installation.md) | vcpkg, source, release packages, containers |
| [Requirements](user/requirements.md) | Platform and dependency expectations |
| [API Reference](user/api_guide.md) | Full public API: builders, wrappers, callbacks |
| [API Stability](user/api_stability.md) | Public API, source compatibility, ABI, and deprecation policy |
| [Migration from Unilink](https://github.com/wirestead/wirestead/blob/main/docs/migration-from-unilink.md) | Package, header, namespace, build, and Python migration guidance |
| [Changelog](https://github.com/wirestead/wirestead/blob/main/CHANGELOG.md) | Release history and compatibility notes |
| [Transport Feature Matrix](user/transport_matrix.md) | Feature support across wrappers and transports |
| [Troubleshooting](user/troubleshooting.md) | Common failures and debugging steps |
| [Python Bindings](user/python_bindings.md) | Moved to the Wirestead Python repository |
| [Performance](user/performance.md) | Build and runtime tuning |

**Tutorials:**

| Tutorial | Focus |
|----------|-------|
| [Getting Started](user/tutorials/01_getting_started.md) | First TCP client |
| [TCP Server](user/tutorials/02_tcp_server.md) | Server lifecycle and callbacks |
| [UDS Communication](user/tutorials/03_uds_communication.md) | Local IPC with Unix domain sockets |
| [Serial Communication](user/tutorials/04_serial_communication.md) | Device I/O and virtual-port testing |
| [UDP Communication](user/tutorials/05_udp_communication.md) | Connectionless send/receive workflow |
| [Asynchronous Patterns](user/tutorials/06_asynchronous_patterns.md) | Advanced non-blocking usage and safety |

---

## 🔧 For Contributors

You are developing or extending wirestead itself.

→ **[Contributor Guide](contributor/index.md)**

| Document | What it covers |
|----------|----------------|
| [Build Guide](contributor/build_guide.md) | CMake options, build profiles, sanitizers |
| [Testing](contributor/testing.md) | Running tests, CI integration |
| [Orin Nano Validation](contributor/orin_nano_validation.md) | Ubuntu 22.04 ARM64 build and test runbook |
| [Release Checklist](https://github.com/wirestead/wirestead/blob/main/docs/release_checklist.md) | Release validation and packaging checklist, maintained in the core repository |
| [Implementation Status](contributor/implementation_status.md) | Verified scope and known gaps |
| [Test Structure](contributor/test_structure.md) | Test organization and coverage |
| [Architecture Overview](contributor/architecture/) | Layers, responsibilities, design patterns |
| [Design Notes](contributor/design/) | Proposed APIs and runtime semantics before implementation |
| [Runtime Behavior](contributor/architecture/runtime_behavior.md) | Lifecycle, retries, callback behavior |
| [Memory Safety](contributor/architecture/memory_safety.md) | Ownership and buffer handling rules |
| [Channel Contract](contributor/architecture/channel_contract.md) | Transport-layer contract and stop semantics |
| [Wrapper Contract](contributor/architecture/wrapper_contract.md) | Wrapper lifecycle and callback guarantees |

---

## Examples and Tests

- [Examples Repository](https://github.com/wirestead/wirestead-examples)
- [Core repository tests](https://github.com/wirestead/wirestead/tree/main/test)

[Back to Repository](https://github.com/wirestead/wirestead-docs)
