# SmartConnect Product Requirements Document (PRD)

Document Version: 1.1.0  
Status: Production Specification  
Target Runtime: C# .NET 8 / WPF (Windows Desktop)  
Workspace Root: D:\Code\SmartConnect  
Network Boundary: Local Area Network (Subnet Unicast / Broadcast)  
Cloud Dependencies: Zero (Air-Gapped Operation)  

---

## 1. Executive Summary & Objective

SmartConnect is an offline-first, high-throughput desktop application built for local area peer-to-peer data transport and real-time bidirectional clipboard synchronization. The application establishes direct socket connections between Windows workstations and compatible local nodes without relying on cloud mediation, relay servers, external internet access, or third-party telemetry.

The system targets high-speed local network interfaces (Wi-Fi 5/6/6E and Gigabit Ethernet) to deliver instant local file synchronization, multi-file and directory streaming, and clipboard bridging across adjacent workstations.

---

## 2. Quantitative Performance Targets & Resource Budgets

| Performance Dimension | Target Metric | Engineering Enforcement |
|---|---|---|
| **5GHz Wi-Fi Throughput** | **> 40.0 MB/s** sustained (802.11ac/ax) | 1 MiB chunk streaming, disabled Nagle algorithm (`NoDelay = true`), and 1 MiB socket kernel buffers. |
| **Gigabit Ethernet Throughput** | **> 100.0 MB/s** sustained | Direct I/O completion ports (`FileOptions.Asynchronous`), zero-copy memory buffers via `ArrayPool<byte>`. |
| **Idle CPU Utilization** | **< 1.0%** total CPU core usage | Event-driven architecture; Win32 `WM_CLIPBOARDUPDATE` hook, passive socket listen loops, 2.0s periodic timer. |
| **Idle Memory Footprint** | **< 45.0 MB** working set | Minimal working set retention, lazy service initialization, avoidance of heavy runtime frameworks. |
| **Active Streaming CPU** | **< 12.0%** single-core ceiling | Vectorized SHA-256 calculation (`IncrementalHash`), zero intermediate byte allocations, non-allocating spans. |
| **Discovery Latency** | **< 2.5 seconds** | 2000 ms UDP broadcast heartbeat combined with immediate solicitation probe on client launch. |
| **Clipboard Bridging Latency** | **< 150 milliseconds** | Direct point-to-point TCP unicast triggered synchronously upon OS clipboard mutation. |
| **Data Integrity Guarantee** | **100% (Zero silent corruption)** | Per-chunk 32-byte SHA-256 digest checks plus composite whole-file SHA-256 manifest validation. |

---

## 3. Functional Scope

### 3.1 Zero-Cloud Peer Discovery
- **UDP Subnet Broadcast:** Binds to `0.0.0.0:45454` with `SO_REUSEADDR` and `SO_BROADCAST`. Emits framed beacons every 2000 ms to `255.255.255.255`.
- **Peer Life Cycle:** Discovers active nodes, tracks keep-alive timestamps, and flags peers as offline after 6000 ms of beacon silence.
- **Graceful Retirement:** Broadcasts `BEACON_GOODBYE` packet upon process exit to immediately clear remote registries without waiting for timeout expiration.

### 3.2 Single and Batch Directory Streaming
- **Single File Transfers:** Direct transmission of individual files of arbitrary size (tested up to 50 GB) with deterministic 1 MiB chunking.
- **Recursive Directory Streaming:** Traverses nested directories, flattens paths into normalized relative POSIX-style paths, and builds a composite `BATCH_MANIFEST`.
- **Chunked Wire Pipeline:** Streams payload data in 1 MiB blocks prefixed with file index, chunk index, byte offset, chunk length, and raw SHA-256 digest.
- **Verification & Acceptance:** Receiver validates chunk digests in real time and computes cumulative file hashes using `IncrementalHash` before renaming temporary staging files into final downloads.

### 3.3 Bidirectional Shared Clipboard
- **Win32 Hook Listener:** Registers `AddClipboardFormatListener` to receive OS-level `WM_CLIPBOARDUPDATE` messages.
- **Deduplication & Anti-Echo:** Hashes clipboard contents with SHA-256. Suppresses transmission when inbound hash matches the previous local state, eliminating infinite feedback loops.
- **Multi-Format Support:** Synchronizes UTF-8 plain text, HTML/rich text fragments, and file URI listings up to an 8 MiB payload ceiling.

---

## 4. Non-Functional & Reliability Constraints

### 4.1 Zero Internet Connectivity Invariant
- The application executes completely isolated from external networks.
- No DNS resolutions for external hosts, no telemetry endpoints, no STUN/TURN relays, and no HTTP/HTTPS cloud requests.
- All network operations are strictly constrained to IPv4 broadcast and unicast within the local subnet.

### 4.2 Network Interface Dynamics & Crash-Free Lifecycle
- **NIC Flapping Resilience:** Dynamically handles transitions between Ethernet, Wi-Fi 2.4GHz, and Wi-Fi 5GHz/6GHz without throwing unhandled socket exceptions or crashing the UI.
- **Adapter Change Detection:** Subscribes to `NetworkChange.NetworkAddressChanged` and `NetworkChange.NetworkAvailabilityChanged` to rebind UDP discovery sockets and reset cached local IP endpoints.
- **Sleep & Resume Tolerance:** Closes broken TCP streams gracefully upon system sleep/suspend, clears incomplete transient buffers, and resumes UDP discovery upon workstation wake.

### 4.3 Storage and Filesystem Invariants
- **Destination Isolation:** Files are written strictly to `%USERPROFILE%\Downloads\SmartConnect` or user-configured directory.
- **Atomic File Finalization:** Incoming files are written to `.part` temporary files and atomically renamed to the target filename only after successful composite SHA-256 verification.
- **Collision Mitigation:** If a file with an identical name exists at destination, appends numeric suffixes (`file (1).ext`) without overwriting existing data.
