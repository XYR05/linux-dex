# Building LinuxDeX

## Prerequisites

### Required (Arch Linux)

```bash
# Build tools
sudo pacman -S base-devel cmake meson ninja pkg-config

# Core libraries
sudo pacman -S libusb ffmpeg sdl2

# UI
sudo pacman -S gtk4 libadwaita

# Audio
sudo pacman -S pipewire

# Testing
sudo pacman -S gtest

# ADB (for development/testing only — not needed at runtime)
sudo pacman -S android-tools

# OpenSSL (for RSA key generation in ADB auth)
sudo pacman -S openssl
```

### Required (Ubuntu/Debian)

```bash
sudo apt install build-essential cmake meson ninja-build pkg-config \
    libusb-1.0-0-dev libavcodec-dev libavformat-dev libavutil-dev libswscale-dev \
    libsdl2-dev libgtk-4-dev libadwaita-1-dev libpipewire-0.3-dev \
    libgtest-dev libssl-dev android-tools-adb
```

### Optional (Reverse Engineering)

```bash
# For analyzing Samsung APKs and Windows binaries
# Install separately — not needed for building
# - Ghidra: https://ghidra-sre.org/
# - JADX: https://github.com/skylot/jadx
# - Frida: pip install frida-tools
```

## Building

### With CMake

```bash
cd linux-dex
cmake -B build -DBUILD_TESTS=ON
cmake --build build -j$(nproc)
```

### With Meson

```bash
cd linux-dex
meson setup build
meson compile -C build
```

### Running Tests

```bash
# CMake
ctest --test-dir build --output-on-failure

# Meson
meson test -C build
```

### Running

```bash
./build/client/linuxdex
```

## Development

### Code Style
- C++20
- clang-format with Google style
- Header guards: `#pragma once`
- Namespaces: `linuxdex::transport`, `linuxdex::protocol`, etc.

### Test-First Development
Every component has tests written BEFORE implementation. Tests live in `client/tests/`.
Implementation is done by making tests pass one at a time.
