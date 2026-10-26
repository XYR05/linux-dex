# Why LinuxDeX Exists

## The Problem

Samsung DeX is a feature in Samsung Galaxy phones that transforms your phone into a desktop computer when connected to a monitor. Samsung made a "DeX on PC" app for Windows and macOS that let you run this desktop environment inside a window on your computer.

**Samsung killed the DeX on PC app.** It was discontinued after One UI 5 (early 2024). It never existed for Linux.

## What Samsung DeX on PC Did

You plugged your Samsung phone into your Windows PC via USB. The app opened and your phone's screen appeared on your monitor — not as a phone screen, but as a **full desktop**:

- Taskbar at the bottom (like Windows/GNOME)
- Resizable app windows you could drag around
- Your PC's keyboard and mouse controlled the phone
- Copy-paste worked between phone and PC
- Phone audio played through PC speakers
- You could drag files between them

## What LinuxDeX Does

The exact same thing, but for Linux. And open source.

## Why Not Just Use scrcpy?

scrcpy is an excellent tool that mirrors your phone screen. But it's fundamentally different from what we're building:

| Feature | scrcpy | LinuxDeX |
|---------|--------|----------|
| Screen mirror | ✅ | ✅ |
| Auto-activates Samsung DeX mode | ❌ manual | ✅ automatic |
| Samsung taskbar + freeform windows | ❌ basic | ✅ full DeX shell |
| Audio forwarding | ✅ | ✅ |
| Clipboard sync | ✅ basic text | ✅ rich (images too) |
| File drag-and-drop | ❌ | ✅ |
| Phone notifications on Linux | ❌ | ✅ |
| Polished GTK4 desktop app | ❌ CLI only | ✅ |
| Custom ADB (no adb dependency) | ❌ needs adb | ✅ standalone |
| Purpose-built for Samsung DeX | ❌ generic | ✅ Samsung-optimized |

## Why Custom ADB Instead of Using System adb?

1. **Zero dependencies** — users don't need to install Android SDK or android-tools
2. **Full control** — we manage the USB connection lifecycle ourselves
3. **Reliability** — no conflicts with other adb servers running on the system
4. **Integration** — tight coupling between device detection and our UI

## Why Not Reverse-Engineer Samsung's Protocol?

We considered this (Option B) and analyzed it thoroughly. The problems:

1. **Encrypted sessions** — Samsung uses session tokens and encryption that change between One UI versions
2. **Discontinued app** — the Windows DeX app no longer exists as a reference
3. **Version fragility** — every Samsung firmware update could break everything
4. **Legal risk** — DMCA/EULA concerns with reverse-engineering proprietary protocols

Instead, we use **proven, documented Android internals** (the same APIs scrcpy uses) combined with **Samsung-specific DeX activation**. This gives us the real DeX experience without depending on Samsung's wire protocol.
