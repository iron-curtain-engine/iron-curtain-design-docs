# p2p-distribute — Concrete Implementation Plan

> **Status:** Implementation plan. Complements `research/p2p-distribute-crate-design.md` (high-level design).
> **Date:** 2026-02

## 1. Identity

- **Crate:** `p2p-distribute`
- **License:** MIT OR Apache-2.0 (per D076)
- **Repo:** Standalone Git repository (separate from IC engine)
- **MSRV:** 1.80+
- **Async runtime:** tokio
- **Zero IC dependencies:** This crate has no dependency on any `ic-*` crate

## 2. Feature Flags

```toml
[features]
default = []
http-seeds    = ["reqwest"]         # BEP 17/19 web seeding
dht           = []                  # Mainline DHT (BEP 5)
pex           = []                  # Peer exchange (BEP 11)
lsd           = []                  # Local service discovery
udp-tracker   = []                  # UDP tracker protocol (BEP 15)
utp           = []                  # uTP transport (BEP 29)
channels      = []                  # Mutable content channels
embedded-tracker = []               # In-process HTTP tracker
webrtc        = ["str0m"]           # WebRTC data channel transport
v2            = []                  # BitTorrent v2 (BEP 52) merkle trees
```

## 3. Architecture Overview

```
Session
├── TorrentHandle (per-content)
│   ├── Unified Piece Scheduler (rarest-first + priority hints)
│   ├── Virtual Peer Pool
│   │   ├── BT Peers (TCP + uTP)
│   │   ├── HTTP Seeds (web seeding)
│   │   └── WebRTC Peers (browser)
│   ├── StorageBackend (trait)
│   └── PieceValidator (SHA-1/SHA-256)
├── DiscoveryBackend (trait)
│   ├── HTTP Tracker
│   ├── UDP Tracker
│   ├── DHT
│   ├── PEX
│   └── LSD
├── AuthPolicy (trait)
└── RevocationPolicy (trait)
```

## 4. Milestone Sequence

### Milestone 1 — Bencode & Wire Protocol (2 weeks)

**Goal:** Parse and generate all BitTorrent protocol messages.

**Deliverables:**
- `bencode.rs`: Encoder/decoder (BEP 3)
- `metainfo.rs`: `.torrent` file parsing, `InfoHash` computation
- `wire.rs`: Peer wire protocol messages — handshake, bitfield, have, request, piece, choke/unchoke, interested/not-interested, cancel
- BEP 10 extension protocol framework (message ID negotiation)

**Tests:** Round-trip bencode, metainfo from real `.torrent` files, wire message serialization.

### Milestone 2 — Storage Backend (2 weeks)

**Goal:** Pluggable content storage with a file-system default.

**Deliverables:**
- `StorageBackend` trait: `read_piece`, `write_piece`, `have_bitfield`, `verify_piece`
- `FsStorage`: Files-on-disk backend with `mmap` for read-heavy access
- `MemoryStorage`: In-memory backend for testing and small content
- Piece hash verification (SHA-1 for v1, SHA-256 for v2)

**Tests:** Write + read-back, hash verification, corrupt piece detection.

### Milestone 3 — Peer Connections & Choking (3 weeks)

**Goal:** Establish TCP connections with peers, implement BT choking algorithm.

**Deliverables:**
- TCP listener + connector
- Peer state machine (connecting → handshaking → active → closing)
- Traditional choking algorithm (4 upload slots, optimistic unchoke every 30s)
- Connection limits (default: 50 per torrent, 200 global)
- Extension handshake (BEP 10)

**Tests:** Two-peer transfer, choking behavior under load.

### Milestone 4 — Piece Scheduler (2 weeks)

**Goal:** Rarest-first piece selection with priority hints.

**Deliverables:**
- Rarity tracking across peer bitfields
- Rarest-first selection with random tie-breaking
- Priority override: `ic_priority` extension for urgent pieces
- Endgame mode: request final pieces from all peers

**Tests:** Rarest-first distribution in 10-peer simulation.

### Milestone 5 — HTTP Tracker & Single Complete Transfer (3 weeks)

**Goal:** Announce to HTTP tracker, complete a full download.

**Deliverables:**
- HTTP tracker client (`/announce`, `/scrape`)
- Compact peer format (BEP 23)
- Integration: tracker → peer discovery → piece download → verification → completion
- CLI smoke test: `p2p-distribute download <torrent> <output>`

**Tests:** End-to-end download from a seeding peer via tracker.

### Milestone 6 — HTTP Web Seeding (3 weeks)

**Goal:** BEP 17/19 web seed support for CDN-backed initial distribution.

**Deliverables:**
- GetRight-style web seeding (BEP 19): Range requests to HTTP URLs
- Hoffman-style web seeding (BEP 17): URL per file
- Hybrid swarm: mix BT peers and HTTP seeds in same download
- Retry/fallback on HTTP errors

**Tests:** Download from pure HTTP seed, mixed HTTP+BT swarm.

### Milestone 7 — Peer Scoring & Bandwidth (2 weeks)

**Goal:** Adaptive peer quality scoring and bandwidth management.

**Deliverables:**
- Per-peer scoring: download speed, upload speed, latency, piece availability, failure rate
- Bandwidth limiter (global upload/download caps)
- Peer rotation: drop low-scoring peers, reconnect to better ones
- Upload slot allocation based on reciprocation (tit-for-tat)

**Tests:** Peer scoring convergence, bandwidth cap enforcement.

### Milestone 8 — DHT, UDP Tracker, PEX, LSD (4 weeks)

**Goal:** Decentralized peer discovery.

**Deliverables:**
- UDP tracker protocol (BEP 15): announce, scrape, connection ID
- Mainline DHT (BEP 5): routing table, `get_peers`, `announce_peer`
- Peer Exchange (BEP 11): ut_pex extension messages
- Local Service Discovery (BEP 14): multicast announce on LAN
- DHT bootstrap from well-known nodes

**Tests:** DHT routing table stability, PEX peer propagation, LSD discovery.

### Milestone 9 — NAT Traversal & uTP (3 weeks)

**Goal:** Connectivity through NATs and firewalls.

**Deliverables:**
- uTP transport (BEP 29): congestion control, LEDBAT
- NAT-PMP / PCP port mapping
- UPnP IGD port mapping
- Hole punching via DHT (BEP 55 — if feasible)
- Fallback chain: direct → uTP → relay (signaling only)

**Tests:** Transfer through simulated NAT (Docker network), uTP congestion control.

### Milestone 10 — IC Extensions & Embedded Tracker (3 weeks)

**Goal:** IC-specific protocol extensions and in-process tracker.

**Deliverables:**
- IC extension messages via BEP 10 (message ID 20):
  - `ic_auth`: Ed25519 peer authentication
  - `ic_priority`: piece urgency hints
- `AuthPolicy` trait for peer authentication
- `RevocationPolicy` trait for content revocation (stop upload, de-announce, callback)
- Embedded HTTP tracker (in-process announce/scrape, peer bucketing, popularity classification)

**Tests:** Auth token validation, revocation enforcement, embedded tracker announce/scrape.

### Milestone 11 — Content Channels (2 weeks)

**Goal:** Mutable append-only content streams for balance patches, server config.

**Deliverables:**
- `SnapshotInfo` with sequence, hash chain, publisher signature
- Channel subscription: background-priority download of new snapshots
- Manual activation (no auto-apply — safety guarantee)
- Retention enforcement (garbage-collect old snapshots per `ChannelRetention` policy)
- Lobby fingerprint integration: expose snapshot IDs for callers to include in match fingerprints

**Tests:** Snapshot chain integrity, retention policy enforcement, subscription lifecycle.

### Milestone 12 — WebRTC Transport & Browser Support (4 weeks)

**Goal:** Desktop ↔ browser peer interop via WebRTC data channels.

**Deliverables:**
- WebRTC data channel transport (`str0m` crate, `"bittorrent"` protocol label)
- WebSocket signaling to workshop server
- ICE candidate exchange, STUN/TURN support
- Hybrid BT + WebRTC swarm (desktop seeds to browser leeches)
- Browser constraints: no persistent seeding, no DHT/PEX, IndexedDB storage (500 MB LRU), CORS-compliant HTTP seeds
- `wasm32-unknown-unknown` build target with feature gates for non-WASM code
- `BrowserWasm` config profile

**Tests:** Signaling protocol, data channel framing, hybrid swarm integration.

### Milestone 13 — Config System & Profiles (2 weeks)

**Goal:** 10-group "all knobs" configuration with built-in profiles.

**Deliverables:**
- Config groups: session, torrent_defaults, storage, network, tracker, peer_selection, bandwidth, cache, security, platform
- 4 built-in profiles: `embedded_minimal`, `desktop_balanced`, `server_seedbox`, `lan_party`
- Config builder with serde (TOML/JSON deserialization)
- Runtime config override API

**Tests:** Profile application, config validation, override precedence.

### Milestone 14 — Hardening & Publication (3 weeks)

**Goal:** Production-ready crate publication.

**Deliverables:**
- Fuzz testing (bencode parsing, wire protocol, metainfo)
- Property-based testing (piece selection fairness, bandwidth allocation)
- API documentation (every public type and method)
- `README.md` with usage examples
- `CHANGELOG.md`
- CI: Linux/macOS/Windows/WASM, clippy, rustfmt, cargo-deny
- `crates.io` publication

**Tests:** Fuzz corpus, stress test (100 peers, 1GB content).

## 5. Total Effort Estimate

| Phase | Milestones | Weeks |
|---|---|---|
| Foundation (wire + storage) | M1–M2 | 4 |
| Core transfer | M3–M5 | 8 |
| Enhanced distribution | M6–M7 | 5 |
| Decentralized discovery | M8–M9 | 7 |
| IC extensions | M10–M11 | 5 |
| Browser + polish | M12–M14 | 9 |
| **Total** | | **~38 weeks** |

## 6. Extensibility Traits

These traits allow IC (and other consumers) to customize behavior without forking:

| Trait | Purpose |
|---|---|
| `StorageBackend` | Pluggable content storage (filesystem, memory, IndexedDB) |
| `AuthPolicy` | Peer authentication (Ed25519, OAuth, anonymous) |
| `PeerFilter` | Custom peer selection/rejection logic |
| `RevocationPolicy` | Content takedown enforcement |
| `DiscoveryBackend` | Custom peer discovery sources |
| `LogSink` | Structured logging integration |

## 7. Relationship to Design Documents

| Document | Relevance |
|---|---|
| `research/p2p-distribute-crate-design.md` | High-level design this plan implements |
| `decisions/09a/D076-standalone-crates.md` | D076 Tier 3 extraction |
| `decisions/09e/D049-workshop-assets.md` | Primary consumer: Workshop P2P distribution |
| `decisions/09b/D074-community-server-bundle.md` | Server-side seeder capability |
| `research/p2p-engine-protocol-design.md` | Wire protocol reference |
```