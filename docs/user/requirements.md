# System Requirements {#user_requirements}

This guide describes the system requirements and dependencies needed to build and use `wirestead`.

---

## System Requirements

### Recommended Platform

- **Ubuntu 22.04 LTS or later**
- **C++20 compatible compiler and standard library** (GCC 10+, recent Clang/libc++, or MSVC 2022+)
- **CMake 3.12 or later** for plain builds; **CMake 3.21 or later** for the repository presets
- **Boost 1.74.0 or later**. The system packages on Ubuntu 22.04 (1.74), RHEL 9
  (1.75) and Ubuntu 24.04 (1.83) all satisfy this; vcpkg is still the simplest
  way to build against a newer Boost than a distribution supplies.

### Supported Platforms

| Platform                   | Status             | Notes                                                  |
| -------------------------- | ------------------ | ------------------------------------------------------ |
| Ubuntu 22.04 LTS           | ✅ Fully Supported | Recommended for production                             |
| Ubuntu 24.04 LTS           | ✅ Fully Supported | Latest features and optimizations                      |
| Ubuntu 22.04 ARM64 (Orin)  | ✅ Validated       | Jetson Orin Nano testbed passed full C++ test sweep    |
| Ubuntu 24.04 ARM64         | 🔄 Validation Path | Secondary ARM64 target in CI/build matrix              |
| Ubuntu 20.04 LTS           | ⚠️ Local Build Only | GCC 10+ required manually                              |
| Other Linux                | 🔄 Should Work     | Not officially tested across all distros/architectures |
| macOS                      | ✅ Fully Supported | Tested in CI (macOS 26, Clang)                         |
| Windows                    | ✅ Fully Supported | Tested in CI (Windows 2022, MSVC)                      |

---

## Dependencies

### Core Library Dependencies

For most application consumers, install the packaged library:

```bash
vcpkg install wirestead
```

Source builds need the underlying third-party dependencies:

Use vcpkg to supply those source-build dependencies:

```bash
vcpkg install \
  boost-asio \
  boost-system \
  spdlog
cmake -S . -B build \
  -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
```

If you use system packages instead of vcpkg for source builds, the selected
Boost installation must be 1.74.0 or later. The default apt Boost on Ubuntu
22.04 (1.74) and 24.04 (1.83) both satisfy that, so no extra Boost is needed on
either.

### Dependency Details

| Dependency  | Version        | License      | Purpose                                                                   |
| ----------- | -------------- | ------------ | ------------------------------------------------------------------------- |
| **GCC/G++** | 10+            | GPL          | C++20 compiler                                                            |
| **Clang**   | 14+ (optional) | Apache 2.0   | Alternative C++20 compiler                                                |
| **CMake**   | 3.12+ / 3.21+  | BSD-3-Clause | Build system; 3.21+ is needed only for repository presets                 |
| **Boost**   | 1.74.0+        | BSL 1.0      | Asio for async I/O                                                        |

---

## Compiler Requirements

### Minimum Compiler Versions

| Compiler | Minimum Version | Recommended Version |
| -------- | --------------- | ------------------- |
| GCC      | 10.0            | 10.0+               |
| Clang    | 14.0            | Recent Clang/libc++ |

### C++ Standard

- **C++20** is required
- C++23 is supported but not required

---

## Runtime Requirements

### For Applications Using wirestead

Applications must be able to resolve the same Boost 1.74.0+ dependency set used to build `wirestead`. vcpkg consumers get this through the vcpkg toolchain; source and distribution-package consumers can use the system Boost on any of the platforms listed above.

### Thread Support

- POSIX threads (pthread) support required
- Typically included in standard Linux distributions

---

## Platform-Specific Notes

### Ubuntu 22.04 LTS

- **The default GCC and Boost packages are enough** as of v0.9.5, which lowered
  the Boost minimum to 1.74 - exactly what Jammy ships. GCC 11 covers the C++20
  subset in use.
- `apt install build-essential cmake libboost-all-dev libspdlog-dev` is a
  complete dependency set; vcpkg is optional here rather than required.
- CI builds and runs the test suite on this platform against those system
  packages, on amd64 and arm64.

### Ubuntu ARM64 / Jetson Orin Nano

- Source builds use the same Linux/POSIX code path as x86_64
- Validated on Jetson Orin Nano with Ubuntu 22.04 on `aarch64`
- Current secondary validation target: Ubuntu 24.04 on `aarch64`
- Full C++ test sweep passed on the Orin Nano testbed: 481 tests passed, 0 failed
- Python binding validation now belongs to the Wirestead Python repository
- ARM64 release packaging validation passed, including installed-package consumer smoke via `find_package(wirestead)` and `wirestead::wirestead`
- One test remains intentionally disabled in that run: `UdsErrorTest.ServerStopWithActiveSessions`
- Serial integration tests require `socat` or physical loopback hardware

### Ubuntu 20.04 LTS

- Default GCC 9.4 does **not** meet requirements
- Must install GCC 10+ or a C++20-capable Clang/libc++ toolchain manually
- See [Ubuntu 20.04 Build Guide](../contributor/build_guide.md#ubuntu-2004-build)
- **Note**: Ubuntu 20.04 reached end-of-life in April 2025; local builds still work

### Other Linux Distributions

- Debian/Fedora/RHEL/Arch builds should work when GCC 10+ and Boost 1.74.0+ are supplied
- RHEL 9 / CentOS Stream 9: system `boost-devel` is 1.75, which is enough. CI
  builds and tests this platform. `spdlog-devel` comes from EPEL.
- RHEL 8 is **not** supported: its Boost is 1.66, below the minimum, and its
  default GCC 8 does not implement the C++20 features this library uses.
- Arch Linux: Fully supported with latest packages

---

## Verifying Your Environment

### Check Compiler Version

```bash
# GCC
g++ --version
# Should show version 10.0 or higher

# Clang (if using)
clang++ --version
# Should show version 14.0 or higher
```

### Check CMake Version

```bash
cmake --version
# Should show 3.12 or higher for plain builds, or 3.21 or higher for repository presets
```

### Check Boost Version

```bash
grep BOOST_LIB_VERSION /path/to/boost/include/boost/version.hpp
# Should show 1_83 or higher
```

### Quick Environment Test

```bash
# Test compilation with C++20
echo '#include <string>
int main() { return 0; }' > test.cpp
g++-10 -std=c++20 test.cpp -o test
./test && echo "C++20 support: OK"
rm test test.cpp
```

---

## Troubleshooting

### Problem: Compiler Too Old

```bash
# Ubuntu 20.04: Install newer GCC
sudo apt install -y gcc-10 g++-10
export CC=gcc-10
export CXX=g++-10
```

### Problem: Boost Not Found

```bash
# Source builds: install dependencies with vcpkg
vcpkg install \
  boost-asio \
  boost-system \
  spdlog

# Or specify Boost location to CMake
cmake -DBOOST_ROOT=/path/to/boost ...
```

### Problem: CMake Too Old

```bash
# Install latest CMake from official repository
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | gpg --dearmor - | sudo tee /etc/apt/trusted.gpg.d/kitware.gpg >/dev/null
sudo apt-add-repository 'deb https://apt.kitware.com/ubuntu/ focal main'
sudo apt update
sudo apt install cmake
```

---

## Next Steps

- [Quick Start Guide](quickstart.md) - Get started with wirestead
- [Installation Guide](installation.md) - Installation options
- [Troubleshooting](troubleshooting.md) - Common issues and solutions
