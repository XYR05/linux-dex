# ADB Protocol Reference

Reference for LinuxDeX's custom ADB client implementation.

## USB Detection

Samsung devices use Vendor ID `0x04e8`. The ADB interface is identified by:
- Class: `0xFF` (vendor-specific)
- Subclass: `0x42`
- Protocol: `0x01`

The interface has two bulk endpoints:
- **Bulk OUT**: Send packets to the phone
- **Bulk IN**: Receive packets from the phone

## Packet Structure

Every ADB message is a 24-byte header, optionally followed by a payload:

```
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ command  │   arg0   │   arg1   │ data_len │ checksum │  magic   │
│ uint32   │ uint32   │ uint32   │ uint32   │ uint32   │ uint32   │
│ (4 bytes)│ (4 bytes)│ (4 bytes)│ (4 bytes)│ (4 bytes)│ (4 bytes)│
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

All fields are **little-endian**.

- `magic` = `command ^ 0xFFFFFFFF`
- `checksum` = sum of all payload bytes (mod 2³²), or 0 if no payload

## Commands

| Name | Value (hex) | Value (ASCII) |
|------|-------------|---------------|
| CNXN | 0x4E58434E | 'NXCN' |
| AUTH | 0x48545541 | 'HTUA' |
| OPEN | 0x4E45504F | 'NEPO' |
| OKAY | 0x59414B4F | 'YAKO' |
| WRTE | 0x45545257 | 'ETRW' |
| CLSE | 0x45534C43 | 'ESLC' |

## Connection Flow

### Step 1: CNXN (Connect)

Client sends:
```
command  = CNXN (0x4E58434E)
arg0     = 0x01000000 (ADB protocol version 1)
arg1     = 1048576 (max payload size = 1MB)
payload  = "host::features=stat_v2,shell_v2\0"
```

### Step 2: AUTH (Authentication)

If the device doesn't trust our key, it responds with AUTH:
```
command  = AUTH
arg0     = 1 (TOKEN — device is sending a challenge)
payload  = 20-byte random token
```

We sign the token with our RSA private key (PKCS#1 v1.5, SHA-1) and reply:
```
command  = AUTH
arg0     = 2 (SIGNATURE — we're sending the signed response)
payload  = 256-byte RSA signature
```

If the device has never seen our key, we also send our public key:
```
command  = AUTH
arg0     = 3 (RSAPUBLICKEY — sending our public key for user approval)
payload  = Base64-encoded RSA public key + " linuxdex@hostname\0"
```

This triggers the "Allow USB Debugging?" dialog on the phone.

On success, device replies with CNXN:
```
command  = CNXN
arg0     = 0x01000000
arg1     = max_payload
payload  = "device::features=...\0"
```

### Step 3: OPEN (Open Stream)

To open a stream to a service:
```
command  = OPEN
arg0     = local_id (our stream ID, any non-zero uint32)
arg1     = 0
payload  = "service_string\0"
```

Services we use:
- `shell:...` — execute commands
- `sync:` — file push/pull
- `localabstract:linuxdex` — our main data socket

Device responds with OKAY:
```
command  = OKAY
arg0     = remote_id (device's stream ID)
arg1     = local_id (echoed back)
```

### Step 4: WRTE / OKAY (Data Transfer)

To send data:
```
command  = WRTE
arg0     = local_id
arg1     = remote_id
payload  = data
```

Device acknowledges with OKAY. We must also acknowledge device's WRTE with OKAY.

### Step 5: CLSE (Close)

```
command  = CLSE
arg0     = local_id
arg1     = remote_id
```

## RSA Key Format

ADB uses 2048-bit RSA keys. The private key is stored in PKCS#8 PEM format.
The public key is sent as Android's custom base64 format during AUTH.

Key storage location: `~/.linuxdex/adb_key` and `~/.linuxdex/adb_key.pub`

## Server Push & Execute

To run our Java server on the phone:

```
1. OPEN "sync:" → push server.jar to /data/local/tmp/linuxdex-server.jar
2. OPEN "shell:CLASSPATH=/data/local/tmp/linuxdex-server.jar app_process / com.linuxdex.server.Server"
3. Server starts, creates localabstract:linuxdex socket
4. OPEN "localabstract:linuxdex" → main data channel
```
