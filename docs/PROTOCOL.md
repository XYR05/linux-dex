# Wire Protocol Specification

Version: 1.0 (Draft)

## Overview

LinuxDeX uses a custom binary protocol multiplexed over a single ADB local abstract socket (`localabstract:linuxdex`). This protocol carries video, audio, input, clipboard, file transfer, and notification data over separate logical channels.

## Transport

The protocol runs inside an ADB stream. The ADB connection is established first (see ADB_PROTOCOL.md), then a stream is opened to the server's local abstract socket:

```
ADB OPEN "localabstract:linuxdex"
```

All subsequent data flows through this single ADB stream, demultiplexed by channel ID.

## Message Format

Every message has a 6-byte header followed by a variable-length payload:

```
Offset  Size    Field       Description
0       1       type        Message type (see table below)
1       1       channel     Channel ID (see table below)
2       4       length      Payload length in bytes (little-endian, max 4MB)
6       N       payload     Message-specific data
```

### Message Types

| Type | Value | Description |
|------|-------|-------------|
| DATA | 0x01 | Raw data (video/audio frames, file chunks) |
| CONTROL | 0x02 | Control command (JSON-encoded) |
| ACK | 0x03 | Acknowledgment |
| ERROR | 0x04 | Error report |
| PING | 0x05 | Keep-alive ping |
| PONG | 0x06 | Keep-alive response |

### Channel IDs

| Channel | ID | Direction | Description |
|---------|----|-----------|-------------|
| CONTROL | 0x00 | Bidirectional | Session management, capability negotiation |
| VIDEO | 0x01 | Server → Client | H.264/HEVC elementary stream frames |
| AUDIO | 0x02 | Server → Client | Audio frames (Opus) |
| INPUT | 0x03 | Client → Server | Keyboard and mouse events |
| CLIPBOARD | 0x04 | Bidirectional | Clipboard content |
| FILE | 0x05 | Bidirectional | File transfer data |
| NOTIFICATION | 0x06 | Server → Client | Phone notification data |

## Channel Specifications

### CONTROL Channel (0x00)

Control messages use JSON payloads for readability and extensibility.

#### Session Handshake

Client → Server (after ADB socket connected):
```json
{
    "type": "hello",
    "version": 1,
    "client": "LinuxDeX",
    "client_version": "0.1.0",
    "capabilities": ["video", "audio", "input", "clipboard", "file", "notification"]
}
```

Server → Client:
```json
{
    "type": "hello_ack",
    "version": 1,
    "server": "LinuxDeX-Server",
    "device_model": "SM-S928B",
    "android_version": 15,
    "one_ui_version": "7.0",
    "dex_supported": true,
    "capabilities": ["video", "audio", "input", "clipboard", "file", "notification"]
}
```

#### Start DeX

Client → Server:
```json
{
    "type": "start_dex",
    "resolution": {"width": 1920, "height": 1080},
    "dpi": 160,
    "video_codec": "h264",
    "video_bitrate": 8000000,
    "max_fps": 60,
    "audio_codec": "opus",
    "audio_sample_rate": 48000
}
```

Server → Client:
```json
{
    "type": "dex_started",
    "display_id": 2,
    "actual_resolution": {"width": 1920, "height": 1080},
    "actual_dpi": 160
}
```

#### Stop DeX

Client → Server:
```json
{
    "type": "stop_dex"
}
```

### VIDEO Channel (0x01)

Raw H.264/HEVC NAL units with a minimal frame header:

```
Offset  Size    Field           Description
0       8       pts             Presentation timestamp (microseconds, little-endian)
8       1       flags           Bit 0: keyframe, Bit 1: config frame (SPS/PPS)
9       N       nal_data        Raw H.264/HEVC NAL unit(s)
```

The first frame is always a config frame containing SPS and PPS NAL units.

### AUDIO Channel (0x02)

```
Offset  Size    Field           Description
0       8       pts             Presentation timestamp (microseconds, little-endian)
8       N       audio_data      Opus-encoded audio frame
```

### INPUT Channel (0x03)

Binary-packed input events:

#### Keyboard Event
```
Offset  Size    Field           Description
0       1       event_type      0x01 = keyboard
1       1       action          0x00 = DOWN, 0x01 = UP
2       4       keycode         Android keycode (AKEYCODE_*)
6       4       meta_state      Modifier keys bitmask
```

#### Mouse Move Event
```
Offset  Size    Field           Description
0       1       event_type      0x02 = mouse_move
1       4       x               X coordinate (float, little-endian)
5       4       y               Y coordinate (float, little-endian)
9       4       display_width   Current display width
13      4       display_height  Current display height
```

#### Mouse Button Event
```
Offset  Size    Field           Description
0       1       event_type      0x03 = mouse_button
1       1       action          0x00 = DOWN, 0x01 = UP
2       1       button          0x01 = left, 0x02 = right, 0x04 = middle
3       4       x               X coordinate (float)
7       4       y               Y coordinate (float)
```

#### Scroll Event
```
Offset  Size    Field           Description
0       1       event_type      0x04 = scroll
1       4       x               X coordinate (float)
5       4       y               Y coordinate (float)
9       4       h_scroll        Horizontal scroll (float)
13      4       v_scroll        Vertical scroll (float)
```

### CLIPBOARD Channel (0x04)

```json
{
    "type": "clipboard_set",
    "mime_type": "text/plain",
    "content": "copied text here"
}
```

For binary clipboard data (images):
```
type=DATA, channel=CLIPBOARD
Offset  Size    Field
0       4       mime_type_len
4       N       mime_type       (e.g., "image/png")
4+N     M       binary_data
```

### FILE Channel (0x05)

#### File Transfer Start
```json
{
    "type": "file_start",
    "transfer_id": 1,
    "filename": "photo.jpg",
    "size": 1048576,
    "mime_type": "image/jpeg"
}
```

#### File Data Chunk
```
type=DATA, channel=FILE
Offset  Size    Field
0       4       transfer_id
4       4       chunk_index
8       N       data
```

#### File Transfer Complete
```json
{
    "type": "file_complete",
    "transfer_id": 1,
    "checksum": "sha256:abcdef..."
}
```

### NOTIFICATION Channel (0x06)

```json
{
    "type": "notification",
    "id": 12345,
    "package": "com.whatsapp",
    "app_name": "WhatsApp",
    "title": "John Doe",
    "text": "Hey, how are you?",
    "timestamp": 1698300000000,
    "priority": "high",
    "icon_base64": "iVBORw0KGgo..."
}
```

## Flow Control

- The client must send ACK messages for every 10th video frame received
- If the server doesn't receive an ACK within 1 second, it drops quality
- PING/PONG every 5 seconds to detect dead connections
- Maximum message payload: 4MB (4,194,304 bytes)
- File transfers are chunked at 64KB per chunk

## Error Handling

Error messages include a code and description:
```json
{
    "type": "error",
    "code": 100,
    "message": "DeX not supported on this device"
}
```

| Code | Meaning |
|------|---------|
| 100 | DeX not supported |
| 101 | DeX failed to start |
| 102 | Video encoder failed |
| 103 | Audio capture denied |
| 200 | Protocol version mismatch |
| 201 | Invalid message format |
| 300 | File transfer failed |
| 301 | Clipboard access denied |
