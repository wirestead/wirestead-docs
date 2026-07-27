# Installation Guide {#user_installation}

This guide covers the supported ways to install and use the **wirestead** library in your project. For most users, **vcpkg** is the recommended and simplest option.

## Prerequisites

- **CMake**: 3.12 or higher for plain builds; 3.21 or higher for the repository presets
- **C++ Compiler**: C++20 compatible (GCC 10+, recent Clang/libc++, MSVC 2022+)
- **Boost**: 1.83.0 or higher
- **Platform**: Linux, Windows, macOS

**Dependency policy:** vcpkg is the recommended dependency supplier. CMake owns the version policy and rejects Boost versions older than 1.83.0. OS package manager Boost packages are supported only when they meet that minimum.

## Installation Methods

### Method 1: vcpkg (Recommended)

The easiest and most reliable way to consume **wirestead** is via **vcpkg**, which provides a fully integrated CMake workflow and cross-platform builds.

#### Step 1: Install via vcpkg

```bash
vcpkg install wirestead
```

#### Step 2: Use in your project

```cmake
cmake_minimum_required(VERSION 3.12)
project(my_app LANGUAGES CXX)

find_package(wirestead CONFIG REQUIRED)
add_executable(my_app main.cpp)
target_link_libraries(my_app PRIVATE wirestead::wirestead)
target_compile_features(my_app PRIVATE cxx_std_20)
```

```cpp
#include <wirestead/wirestead.hpp>

int main() {
    auto client = wirestead::tcp_client("127.0.0.1", 8080).build();
    // Callbacks are optional for construction.
    // Register on_error/on_data for production workflows.
    // ...
}
```

**Note:** The vcpkg port name is `wirestead`, while the CMake package and target name remain `wirestead`.

### Method 2: Install from Source

Use this method if you need a custom local build.

Source builds require Boost 1.83.0+. On Ubuntu 22.04/24.04, system Boost
packages may be older than this requirement. Prefer the repository-managed
vcpkg setup unless you already have a suitable Boost installation.

#### Option A: Repository-managed vcpkg setup

The setup script creates an untracked repository-local vcpkg checkout and uses
it for the repository presets.

```bash
git clone https://github.com/wirestead/wirestead.git
cd wirestead
./scripts/setup_dev_env.sh
cmake --preset dev-linux-x64
cmake --build --preset dev-linux-x64 --parallel 1
```

The preset names may be platform-specific. Use the closest preset for your
platform or use the vcpkg toolchain option below.

#### Option B: Plain CMake with existing dependencies

Use this only when Boost 1.83.0+ and other dependencies are already
discoverable by CMake.

```bash
git clone https://github.com/wirestead/wirestead.git
cd wirestead
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel 1
sudo cmake --install build
```

#### Option C: Plain CMake with vcpkg toolchain

Use this when you want a plain build directory but still want vcpkg to provide
Boost and other third-party dependencies.

```bash
git clone https://github.com/wirestead/wirestead.git
cd wirestead
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
cmake --build build --parallel 1
sudo cmake --install build
```

#### Use in your project

```cmake
find_package(wirestead CONFIG REQUIRED)
add_executable(my_app main.cpp)
target_link_libraries(my_app PRIVATE wirestead::wirestead)
target_compile_features(my_app PRIVATE cxx_std_20)
```

### Method 3: Release Packages

Pre-built binary packages are available from GitHub Releases.

#### Step 1: Download and extract

Choose the archive matching your OS and architecture. Release assets use this naming pattern:

```text
wirestead-${VERSION}-${OS_LABEL}-${ARCH}.${EXT}
```

`VERSION` is the package version from the release asset name. It normally matches the root `CMakeLists.txt` project version for the GitHub Release. For example, a `v0.1.0` release typically publishes assets named with `0.1.0`.

Common asset names:

| Platform           | Asset                                                   |
| ------------------ | ------------------------------------------------------- |
| Ubuntu 22.04 x64   | `wirestead-${VERSION}-ubuntu-22.04-amd64.tar.gz`          |
| Ubuntu 22.04 ARM64 | `wirestead-${VERSION}-ubuntu-22.04-arm64.tar.gz`          |
| Ubuntu 24.04 x64   | `wirestead-${VERSION}-ubuntu-24.04-amd64.tar.gz`          |
| Ubuntu 24.04 ARM64 | `wirestead-${VERSION}-ubuntu-24.04-arm64.tar.gz`          |
| macOS 15 ARM64     | `wirestead-${VERSION}-macos-15-arm64.tar.gz`              |
| macOS 15 DMG       | `wirestead-${VERSION}-macos-15-arm64.dmg`                 |
| Windows x64        | `wirestead-${VERSION}-windows-amd64.zip`                  |
| Windows ARM64      | `wirestead-${VERSION}-windows-arm64.zip`                  |

```bash
# Example for Ubuntu 22.04 x64
export WIRESTEAD_VERSION="<latest-release-version>"
wget https://github.com/wirestead/wirestead/releases/latest/download/wirestead-${WIRESTEAD_VERSION}-ubuntu-22.04-amd64.tar.gz
tar -xzf wirestead-${WIRESTEAD_VERSION}-ubuntu-22.04-amd64.tar.gz
cd wirestead-${WIRESTEAD_VERSION}-ubuntu-22.04-amd64
```

```bash
# Example for Ubuntu 22.04 ARM64 / aarch64
export WIRESTEAD_VERSION="<latest-release-version>"
wget https://github.com/wirestead/wirestead/releases/latest/download/wirestead-${WIRESTEAD_VERSION}-ubuntu-22.04-arm64.tar.gz
tar -xzf wirestead-${WIRESTEAD_VERSION}-ubuntu-22.04-arm64.tar.gz
cd wirestead-${WIRESTEAD_VERSION}-ubuntu-22.04-arm64
```

ARM64 release artifacts are intended to be produced from an Ubuntu 22.04 baseline so Jetson/Orin systems can consume the same package without relying on Ubuntu 24.04 userspace.

#### Step 2: Choose an install prefix

Release archives are already laid out as an install prefix. You can use the extracted directory directly or copy it to a stable location.

```bash
# Use the extracted directory directly
export WIRESTEAD_PREFIX="$PWD"

# Or install under /opt/wirestead on Unix
sudo mkdir -p /opt/wirestead
sudo cp -a include lib share /opt/wirestead/
export WIRESTEAD_PREFIX="/opt/wirestead"
```

#### Step 3: Use in your project

```cmake
set(CMAKE_PREFIX_PATH "$ENV{WIRESTEAD_PREFIX}")
find_package(wirestead CONFIG REQUIRED)
```

### Method 4: Git Submodule Integration

For projects that want to vendor **wirestead** directly.

#### Step 1: Add submodule

```bash
git submodule add https://github.com/wirestead/wirestead.git third_party/wirestead
git submodule update --init --recursive
```

#### Step 2: Use in CMake

```cmake
add_subdirectory(third_party/wirestead)
add_executable(my_app main.cpp)
target_link_libraries(my_app PRIVATE wirestead::wirestead)
```

## Packaging Notes

- **vcpkg**
  - Official port: `wirestead`
  - Recommended for most users
- **Containers**
  - [Wirestead container repository](https://github.com/wirestead/wirestead-container)

Other package managers (e.g., Conan) are not yet officially supported.

## Build Options (Source Builds)

| Option                             | Default | Description                          |
| ---------------------------------- | ------- | ------------------------------------ |
| `WIRESTEAD_BUILD_SHARED`             | `ON`    | Build shared library                 |
| `WIRESTEAD_BUILD_STATIC`             | `ON`    | Build static library                 |
| `WIRESTEAD_BUILD_TESTS`              | `ON`    | Build tests                          |
| `WIRESTEAD_BUILD_DOCS`               | `OFF`   | Legacy core option; documentation is generated from this documentation repository |
| `WIRESTEAD_ENABLE_CONFIG`            | `ON`    | Enable configuration management API  |
| `WIRESTEAD_ENABLE_INSTALL`           | `ON`    | Enable install targets               |
| `WIRESTEAD_ENABLE_PKGCONFIG`         | `ON`    | Install pkg-config file              |
| `WIRESTEAD_ENABLE_EXPORT_HEADER`     | `ON`    | Generate export header               |

Example:

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DWIRESTEAD_BUILD_SHARED=OFF \
  -DWIRESTEAD_BUILD_TESTS=OFF
```

## Next Steps

- [Quick Start Guide](quickstart.md)
- [API Reference](api_guide.md)
- [Migration from Unilink](https://github.com/wirestead/wirestead/blob/main/docs/migration-from-unilink.md)
- [Changelog](https://github.com/wirestead/wirestead/blob/main/CHANGELOG.md)
- [Examples](https://github.com/wirestead/wirestead-examples)
