# LinuxDeX

> A standalone Linux desktop app that turns your Samsung Galaxy phone into a full desktop computer on your monitor — with keyboard/mouse control, audio, clipboard, file sharing, and notifications — by activating Samsung DeX mode and streaming it over USB.

## What Is This?

Samsung DeX on PC was a Windows/macOS app that let you plug your Samsung phone into your computer and use it as a full desktop. Samsung killed it. It never existed for Linux.

**LinuxDeX** brings that experience to Linux.

## How It Works

Two programs talk to each other over USB:

```
┌─────────────────────┐         USB Cable          ┌─────────────────────┐
│    YOUR LINUX PC     │ ◄════════════════════════► │  YOUR SAMSUNG PHONE │
│                      │                            │                     │
│  "LinuxDeX Client"   │                            │  "LinuxDeX Server"  │
│  (C++ desktop app)   │                            │  (tiny Java program)│
│                      │                            │                     │
│  • Shows the phone's │  ◄── video stream ───────  │  • Captures screen  │
│    DeX desktop in    │                            │  • Activates DeX    │
│    a window          │  ◄── audio stream ───────  │    desktop mode     │
│  • Sends your        │                            │  • Encodes video    │
│    keyboard/mouse    │  ─── key/mouse events ──►  │  • Injects your     │
│    input to phone    │                            │    keyboard/mouse   │
│  • Syncs clipboard   │  ◄──► clipboard data ───►  │  • Reads clipboard  │
│  • Transfers files   │  ◄──► file data ────────►  │  • Sends files      │
│  • Shows phone       │  ◄── notifications ─────   │  • Reads notifs     │
│    notifications     │                            │                     │
└─────────────────────┘                            └─────────────────────┘
```

### The Phone Server
A small Java program (~100KB) that we **push to the phone via USB** and run automatically. Nothing is installed — no app, no APK, no root. It runs temporarily via `app_process` (Android's internal Java runtime).

### The Linux Client
A native GTK4 desktop app that detects your phone, pushes the server, and streams the DeX desktop to your monitor.

### Server Lifecycle
```
USB Plugged In                                          USB Disconnected
     │                                                        │
     ▼                                                        ▼
┌─────────┐   ┌──────────┐   ┌─────────┐   ┌──────────┐   ┌──────────┐
│  Detect  │──►│ Push JAR  │──►│  Run it  │──►│  DeX is  │──►│  Cleanup │
│  phone   │   │ to phone  │   │ via ADB  │   │  active  │   │          │
└─────────┘   └──────────┘   └─────────┘   └──────────┘   └──────────┘

• JAR pushed to /data/local/tmp/ (temporary storage)
• Executed with app_process (system shell privileges, no root)
• On disconnect: process auto-dies, JAR deleted, phone returns to normal
• No trace left on the phone
```

## Features

| Feature | Description |
|---------|-------------|
| Samsung DeX activation | Auto-activates DeX desktop mode (taskbar, freeform windows) |
| Video streaming | H.264/HEVC, hardware-encoded on phone, hardware-decoded on PC |
| Audio forwarding | Phone audio plays through PC speakers |
| Keyboard & mouse | Full input forwarding with proper keycode mapping |
| Clipboard sync | Copy on phone, paste on PC and vice versa |
| File transfer | Drag-and-drop files between phone and PC |
| Notifications | Phone notifications show as Linux desktop notifications |
| Auto-discovery | Detects Samsung phone on USB plug-in |
| Standalone | No `adb` installation needed — custom ADB client built-in |

## Tech Stack

| Component | Technology |
|-----------|------------|
| Linux Client | C++20 |
| Build System | CMake + Meson |
| UI | GTK4 + libadwaita |
| Video Decode | FFmpeg (libavcodec) + VAAPI hardware accel |
| Video Render | SDL2 |
| Audio | FFmpeg + PipeWire |
| USB | libusb-1.0 (custom ADB implementation) |
| Phone Server | Java (runs via app_process) |
| Test Framework | GoogleTest |

## Building

```bash
cmake -B build -DBUILD_TESTS=ON
cmake --build build
ctest --test-dir build
```

## License

GPLv3
