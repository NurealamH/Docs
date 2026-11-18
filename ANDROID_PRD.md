# Android Product Requirements Document (PRD): SmartConnect Companion

## 1. Executive Summary & Vision
SmartConnect-Android is a native Android companion application engineered to pair with the SmartConnect Desktop system over local networks (LAN, WLAN, and Wi-Fi Direct/Hotspot). The application eliminates reliance on cloud relays, third-party synchronization servers, or internet uplinks, delivering zero-configuration peer-to-peer (P2P) file transfers and bi-directional clipboard synchronization.

The system is built upon raw TCP/UDP network primitives using a hardened binary framing protocol. It achieves sustained wire-speed file transfer performance while adhering to Android platform constraints including Scoped Storage, App Standby buckets, Doze mode restrictions, and strict background execution limits.

---

## 2. Quantitative Performance & Resource Ceilings

### 2.1 Throughput Benchmarks
| Network Interface | Protocol | Target Sustained Throughput | Peak Target |
| :--- | :--- | :--- | :--- |
| 5GHz Wi-Fi (802.11ac, 80MHz channel) | Raw TCP Stream | **>40 MB/s** | **65 MB/s** |
| 5GHz Wi-Fi (802.11ax / Wi-Fi 6) | Raw TCP Stream | **>55 MB/s** | **85 MB/s** |
| Wi-Fi Direct / Local Hotspot | Raw TCP Stream | **>50 MB/s** | **75 MB/s** |
| 2.4GHz Wi-Fi (802.11n, 20/40MHz) | Raw TCP Stream | **>12 MB/s** | **18 MB/s** |

*Throughput conditions: Measured over transfers of files $\ge 1\text{ GiB}$ without thermal throttling, writing directly to internal UFS 2.2/3.0/3.1 storage via Android `MediaStore` or public `Download/` directories.*

### 2.2 Resource Ceilings & Energy Budget
- **Battery Drain Limit**: $\le 3.0\%$ battery capacity consumption per $10\text{ GB}$ of data transferred over 5GHz Wi-Fi on a 4,500–5,000 mAh battery cell at room temperature ($22^\circ\text{C}$).
- **Steady-State Memory Footprint**: $\le 80\text{ MB}$ Resident Set Size (RSS) during active $1\text{ MiB}$ chunk streaming. Dalvik/ART heap usage must remain flat ($\le 35\text{ MB}$); all chunk transmission buffers must reside in off-heap direct byte buffers managed via a fixed-capacity buffer pool.
- **Garbage Collection Overhead**: Zero concurrent or stop-the-world GC pauses triggered by buffer churn during active chunk transfer.
- **Latency Budgets**:
  - UDP Discovery roundtrip (beacon emit to peer UI representation): $\le 250\text{ ms}$.
  - TCP Handshake and authentication verification: $\le 300\text{ ms}$.
  - Single/Batch Transfer start latency: $\le 150\text{ ms}$ from user approval to first byte transferred.
  - Remote clipboard dispatch-to-paste availability: $\le 150\text{ ms}$.

---

## 3. Core Functional Scope

```
+-------------------------------------------------------------------------+
|                        SmartConnect Android Client                      |
+--------------------+---------------------+------------------------------+
| Discovery Engine   | Transfer Pipeline   | Sync Engine                  |
| - UDP 45454 Beacon | - TCP 46554 Engine  | - Real-time Clipboard        |
| - MulticastLock Mgmt- 1 MiB Chunks       | - SHA-256 Echo Filter        |
| - Subnet Filtering | - Dual SHA-256 Check| - Foreground Service Sync    |
+--------------------+---------------------+------------------------------+
| Storage Pipeline   | Lifecycle Engine    | Security Boundary            |
| - Scoped Storage   | - Sticky FGS        | - RFC 1918 Private LAN Only  |
| - MediaStore API   | - PARTIAL_WAKE_LOCK | - Path Traversal Sanitize    |
| - Checkpoint Meta  | - Network Isolation | - Token Handshake (OpCode 01)|
+--------------------+---------------------+------------------------------+
```

### 3.1 Zero-Cloud LAN Discovery
- Broadcasts presence beacons over UDP port `45454` to the local broadcast address (`255.255.255.255`) and configured subnet broadcast addresses.
- Listens continuously on UDP port `45454` with `SO_REUSEADDR` enabled.
- Heartbeat beacon period: $2,000\text{ ms}$.
- Peer stale timeout: $10,000\text{ ms}$. Peered devices not observed within this window are pruned from the active peer table.
- Acquires `WifiManager.MulticastLock` exclusively while discovery is active; releases the lock immediately when discovery stops or the application enters complete idle state.
- Strictly ignores self-generated beacons based on the unique instance device UUID.

### 3.2 Bi-directional Single & Batch Directory Streaming
- Handles transmission and reception of single files, multiple selected files, and entire directory hierarchies.
- Directory structures are flattened into a batch transfer manifest (`OpCode 0x02`) encoding relative paths, file lengths, and whole-file SHA-256 hashes.
- Data payload is packetized into fixed $1\text{ MiB}$ ($1,048,576\text{ bytes}$) chunks with a $53\text{-byte}$ header (`OpCode 0x03`).
- Supports receiver-driven flow control and immediate per-chunk hash acknowledgment (`OpCode 0x04`).

### 3.3 Background Transfer Foreground Service
- Transfers run within an Android Foreground Service declared with `android:foregroundServiceType="dataSync"`.
- Uses a sticky service model (`START_STICKY`) backed by an ongoing, high-priority notification (`NotificationManager.IMPORTANCE_LOW` to avoid alert spam while preserving persistence).
- Real-time notification displays:
  - Current filename and index (e.g., `Receiving 3 of 12: project_archive.iso`).
  - Transfer speed in $\text{MB/s}$ and overall progress percentage.
  - Action buttons: `Cancel` and `Pause`.
- Automatically releases resources and revokes foreground status upon batch completion or cancellation.

### 3.4 Automatic Bi-Directional Clipboard Synchronization
- Monitors Android system clipboard changes via `ClipboardManager.OnPrimaryClipChangedListener`.
- When local clipboard text changes, the application calculates an SHA-256 digest of the text.
- If the text digest does not match the most recently received remote digest, it encodes the string into a clipboard packet (`OpCode 0x05`) and transmits it across the established TCP connection.
- Incoming clipboard packets are ingested, recorded in the hash exclusion cache, and applied to the Android clipboard via `ClipboardManager.setPrimaryClip()`.
- Clipboard synchronization is strictly opt-in and toggleable in user settings.

---

## 4. Resource Ceilings & Technical Constraints

### 4.1 Memory Allocation & Buffer Architecture
- Dalvik/ART garbage collection must not be subjected to transient allocations during transfer streaming.
- Standard `ByteArray(1024 * 1024)` allocations inside transfer loops are strictly forbidden.
- The network engine must utilize off-heap `ByteBuffer.allocateDirect(1024 * 1024)` instances drawn from a pre-allocated pool with a maximum ceiling of 6 buffers ($6\text{ MiB}$ direct memory overhead).
- Sockets must be configured with `SO_RCVBUF` and `SO_SNDBUF` sizes set to $1,048,576\text{ bytes}$ ($1\text{ MiB}$) to optimize TCP window scaling on high-bandwidth wireless interfaces.

### 4.2 Power & WakeLock Ceilings
- A `PowerManager.WakeLock` with tag `SmartConnect:TransferWakeLock` and flag `PARTIAL_WAKE_LOCK` must be held exclusively while active file streams are actively transmitting or receiving bytes.
- The WakeLock must be released within a `finally` block or lifecycle callback immediately upon:
  - Normal stream completion.
  - User-initiated cancellation.
  - Network disconnection or unrecoverable socket timeout.
- Discovery-only states must never hold a wake lock; discovery relies on standard Android alarm/coroutine timing and foreground service execution if scanning is actively commanded by the user.

### 4.3 Network Interface Guardrails
- The application must strictly operate over Wi-Fi, Ethernet, and local Wi-Fi Hotspot network interfaces.
- Transmission over cellular data (`TRANSPORT_CELLULAR`) is strictly prohibited. If the default route switches to mobile data, active sockets must be immediately terminated with a network error state to protect the user's data plan.

---

## 5. Non-Functional & Platform Compatibility Requirements

### 5.1 Android OS Target Support
- **Minimum Supported API**: Android 8.0 (API Level 26, `Oreo`).
- **Target SDK**: Android 14 / 15 (API Level 34 / 35).
- **Architecture**: 64-bit ARM (`arm64-v8a`) and x86_64 emulator compatibility.

### 5.2 Scoped Storage & Security Guardrails
- 100% compliance with Android Scoped Storage rules (API 29+).
- Inbound files are streamed directly to `MediaStore.Downloads` using `ContentResolver.openOutputStream()` with `IS_PENDING = 1`.
- Upon successful reception and full SHA-256 validation, `IS_PENDING` is updated to `0`, making the file instantly accessible to other applications.
- Inbound filenames are strictly sanitized to prevent directory traversal attacks (rejection of `/`, `\`, and `..`).

### 5.3 Wire Compatibility Commitment
- The Android client must maintain 100% wire-compatibility with the SmartConnect Desktop specification.
- Framing format: 4-byte Big-Endian integers for packet lengths and chunk headers.
- Magic signature: `0x53 0x4D 0x43 0x01` (`SMC\x01`).
- Supported OpCodes: `0x01` (Auth), `0x02` (Batch Manifest), `0x03` (Chunk Data), `0x04` (Chunk ACK), `0x05` (Clipboard Sync), `0x06` (Session Control).
