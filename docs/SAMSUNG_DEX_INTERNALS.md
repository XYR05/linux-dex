# Samsung DeX Internals

Research notes on Samsung DeX's internal architecture, services, and activation mechanisms.

## Samsung DeX System Packages

| Package | Role |
|---------|------|
| `com.sec.android.desktopmode` | Core desktop mode state machine |
| `com.sec.android.desktopmode.uiservice` | Window management, display handling, IPC |
| `com.sec.android.app.desktoplauncher` | Taskbar, app launcher, desktop shell |
| `com.samsung.desktopsystemui` | Status bar, quick settings in DeX mode |

## Querying DeX State

```bash
# Check if DeX is currently active
adb shell settings get global desktop_mode
# Returns: 1 (active) or 0 (inactive)

# Check forced desktop mode setting
adb shell settings get global force_desktop_mode_on_external_displays

# Dump desktop mode service state
adb shell dumpsys desktopmode

# Dump display topology (see virtual displays)
adb shell dumpsys display

# Dump window manager state (freeform tasks)
adb shell dumpsys window

# List DeX-related services
adb shell service list | grep -i dex
adb shell service list | grep -i desktop

# List running DeX activities
adb shell dumpsys activity services | grep -i dex
```

## Activating DeX

### Method 1: Settings Global
```bash
adb shell settings put global force_desktop_mode_on_external_displays 1
```

### Method 2: Samsung Broadcast Intent
```bash
adb shell am broadcast -a com.samsung.android.dex.action.START_DEX
```

### Method 3: UiModeManager (Programmatic)
```java
// Listen for DeX mode changes
UiModeManager uiModeManager = getSystemService(UiModeManager.class);
// UI_MODE_TYPE_DESK = 0x02 (indicates desktop/DeX mode)
```

### Method 4: Knox Desktop Intent
```bash
adb shell am broadcast -a android.app.action.ENTER_KNOX_DESKTOP_MODE
```

## Key Internal State Variables

From `dumpsys desktopmode`:

| Variable | Description |
|----------|-------------|
| `mDualModeEnabled` | Whether DeX is running in dual mode (separate phone + desktop screens) |
| `mDesktopDisplayId` | Display ID assigned to the DeX virtual display |
| `mConnectionState` | Connection type: WIRED, WIRELESS, or NONE |

## Relevant Logcat Tags

```bash
adb logcat -s DesktopModeService WindowManager ActivityTaskManager SecExternalDisplay DeXManager
```

## Samsung APK Extraction

To pull Samsung DeX APKs for analysis:

```bash
# Find the APK paths
adb shell pm path com.sec.android.desktopmode
adb shell pm path com.sec.android.desktopmode.uiservice
adb shell pm path com.sec.android.app.desktoplauncher

# Pull them
adb pull <path> ./apks/

# Decompile with JADX
jadx -d decompiled/ <apk_file>
```

## Key Classes to Investigate (from JADX decompilation)

In `com.sec.android.desktopmode.uiservice`:
- **IDesktopMode / Binder interfaces** — AIDL bridge between WindowManagerService and Samsung desktop services
- **WiredDesktopModeConnection** — state machine for USB/HDMI connections
- **WirelessDesktopModeConnection** — state machine for Miracast/WiFi connections
- **Display management classes** — secondary/virtual display creation and coordinate mapping

## Samsung DeX on PC (Windows) Structure

The now-discontinued Windows app was installed at:
```
C:\Program Files (x86)\Samsung\Samsung DeX\
```

Key files:
- `SamsungDeX.exe` — main entry point
- `DesktopExperience.exe` — desktop rendering and window management
- `DeXComms.dll` — **core protocol module** (USB/WiFi bridge, data pipe, state sync)
- Transport/USB redirection libraries
- Audio/video decoding libraries (H.264/H.265)
- Bundled USB drivers and ADB helpers

## What the Windows App Did Over USB

1. Detected phone via USB (VID `0x04e8`)
2. Established ADB connection (or custom AOA endpoint)
3. Opened control channel via `adb forward` to a local TCP port on the phone
4. Exchanged capability packets (resolution, codec, features) — JSON or binary
5. Negotiated session token / encryption key
6. Started multiplexed streams: video, audio, input, clipboard
7. Video: H.264/HEVC elementary stream with custom framing headers
8. Input: binary-packed keyboard scancodes + mouse coordinates
9. Cleanup on disconnect
