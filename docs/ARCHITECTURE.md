# Architecture

## System Overview

LinuxDeX is a client-server system connected over USB via ADB (Android Debug Bridge).

```
┌──────────────────────────────────────┐       ┌──────────────────────────────────────┐
│           LINUX CLIENT (C++)         │       │         PHONE SERVER (Java)           │
│                                      │       │                                      │
│  ┌──────────┐    ┌────────────────┐  │       │  ┌────────────────┐   ┌───────────┐  │
│  │  GTK4 UI │───►│ Connection Mgr │◄─┼─USB──►┼─►│ Protocol Mux   │──►│ DeX Ctrl  │  │
│  └──────────┘    └────────────────┘  │       │  └────────────────┘   └───────────┘  │
│       │               │              │       │       │    │    │                     │
│  ┌────▼─────┐   ┌─────▼──────┐      │       │  ┌────▼──┐ │  ┌─▼──────────┐         │
│  │ SDL2     │   │ ADB Client │      │       │  │Screen │ │  │ Input      │         │
│  │ Renderer │   │ (libusb)   │      │       │  │Capture│ │  │ Injector   │         │
│  └──────────┘   └────────────┘      │       │  └───────┘ │  └────────────┘         │
│       ▲                              │       │       ┌────▼────┐                    │
│  ┌────┴─────┐   ┌────────────┐      │       │       │ Audio   │                    │
│  │ FFmpeg   │   │ PipeWire   │      │       │       │ Capture │                    │
│  │ Decoder  │   │ Player     │      │       │       └─────────┘                    │
│  └──────────┘   └────────────┘      │       │                                      │
└──────────────────────────────────────┘       └──────────────────────────────────────┘
```

## Component Details

### Transport Layer (ADB Client)

Our custom ADB implementation over libusb. No dependency on system `adb`.

**Responsibilities:**
- USB device enumeration and Samsung phone detection (Vendor ID `0x04e8`)
- ADB protocol: CNXN → AUTH → OPEN → WRTE → CLSE
- RSA key generation and authentication
- Stream multiplexing (multiple logical channels over one USB connection)
- Server JAR push and `app_process` execution

**ADB Packet Format:**
```
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ command  │   arg0   │   arg1   │ data_len │ checksum │  magic   │
│ (4 bytes)│ (4 bytes)│ (4 bytes)│ (4 bytes)│ (4 bytes)│ (4 bytes)│
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
                         24-byte header
                    followed by `data_len` bytes of payload
```

**Commands:**
| Command | Value | Purpose |
|---------|-------|---------|
| CNXN | 0x4E58434E | Connection handshake |
| AUTH | 0x48545541 | RSA authentication |
| OPEN | 0x4E45504F | Open a stream |
| OKAY | 0x59414B4F | Acknowledgment |
| WRTE | 0x45545257 | Write data |
| CLSE | 0x45534C43 | Close stream |

### Wire Protocol (Custom)

Multiplexed binary protocol running inside an ADB stream. Carries multiple logical channels.

**Message Format:**
```
┌──────────┬──────────┬──────────┬──────────────────────┐
│  type    │ channel  │  length  │      payload         │
│ (1 byte) │ (1 byte) │ (4 bytes)│   (length bytes)     │
└──────────┴──────────┴──────────┴──────────────────────┘
                   6-byte header + payload
```

**Channel IDs:**
| Channel | ID | Direction | Content |
|---------|----|-----------|---------|
| CONTROL | 0x00 | Bidirectional | Session setup, capabilities, state |
| VIDEO | 0x01 | Phone → PC | H.264/HEVC elementary stream |
| AUDIO | 0x02 | Phone → PC | Opus/AAC audio frames |
| INPUT | 0x03 | PC → Phone | Keyboard/mouse events |
| CLIPBOARD | 0x04 | Bidirectional | Clipboard data |
| FILE | 0x05 | Bidirectional | File transfer chunks |
| NOTIFICATION | 0x06 | Phone → PC | Notification data |

### Video Pipeline

```
Phone: SurfaceControl → MediaCodec (H.264 HW encoder)
                              │
                         ADB stream
                              │
PC:   FFmpeg decoder (VAAPI HW accel) → SDL2 texture → Window
```

### Input Pipeline

```
PC:   X11/Wayland key/mouse events → Linux keycode
                              │
                    Keymap translation
                              │
                    Android KeyEvent/MotionEvent serialization
                              │
                         ADB stream
                              │
Phone: InputManager.injectInputEvent() → Android input system
```

### Server Lifecycle

```
          LinuxDeX Client                    Samsung Phone
               │                                  │
               │──── USB Detection ──────────────►│
               │                                  │
               │──── ADB CNXN + AUTH ────────────►│
               │◄─── ADB CNXN (accepted) ────────│
               │                                  │
               │──── Push server.jar ────────────►│ → /data/local/tmp/
               │                                  │
               │──── OPEN shell: app_process ────►│ → Server starts
               │◄─── Server ready ────────────────│
               │                                  │
               │──── OPEN localabstract:linuxdex─►│ → Data socket
               │◄─── OKAY ───────────────────────│
               │                                  │
               │──── Control: START_DEX ─────────►│ → DeX mode activates
               │◄─── Control: DEX_READY ─────────│
               │                                  │
               │◄─── Video stream begins ────────│
               │◄─── Audio stream begins ────────│
               │──── Input events ───────────────►│
               │                                  │
              ... (session active) ...            ...
               │                                  │
               │──── USB Disconnected ───────────X│
               │                                  │ → Server process dies
               │                                  │ → JAR auto-cleaned
               │                                  │ → DeX mode exits
```

## Samsung DeX Activation

The server activates DeX using Samsung-specific intents and services:

```java
// Method 1: Desktop mode setting
Settings.Global.putInt(resolver, "force_desktop_mode_on_external_displays", 1);

// Method 2: Samsung broadcast
sendBroadcast(new Intent("com.samsung.android.dex.action.START_DEX"));

// Method 3: Desktop mode service
dumpsys desktopmode (to query state)

// Key Samsung packages involved:
// - com.sec.android.desktopmode
// - com.sec.android.desktopmode.uiservice
// - com.sec.android.app.desktoplauncher
// - com.samsung.desktopsystemui
```

## Directory Structure

```
linux-dex/
├── README.md                        # Project overview
├── LICENSE                          # GPLv3
├── CMakeLists.txt                   # Top-level build
│
├── docs/                            # Documentation
│   ├── WHY.md                       # Why this project exists
│   ├── ARCHITECTURE.md              # This file
│   ├── PROTOCOL.md                  # Wire protocol specification
│   ├── SAMSUNG_DEX_INTERNALS.md     # Samsung DeX reverse engineering notes
│   ├── ADB_PROTOCOL.md             # ADB protocol reference
│   └── BUILDING.md                  # Build instructions
│
├── client/                          # Linux client (C++)
│   ├── CMakeLists.txt
│   ├── include/linuxdex/            # Public headers
│   │   ├── transport/               # ADB + USB
│   │   ├── protocol/                # Wire protocol
│   │   ├── video/                   # Video decode + render
│   │   ├── audio/                   # Audio decode + play
│   │   ├── input/                   # Keyboard + mouse
│   │   └── features/                # Clipboard, files, notifications
│   ├── src/                         # Implementation
│   └── tests/                       # Unit + integration tests
│
├── server/                          # Phone server (Java)
│   └── src/main/java/com/linuxdex/server/
│
├── tools/                           # Dev + RE tools
└── tests/integration/               # End-to-end tests
```
