# P2P Content Distribution — Design Study for IC Workshop

> **Purpose:** Study existing P2P and BitTorrent implementations to inform IC's own Workshop distribution engine.
> **Date:** 2026-02-26
> **Referenced by:** D074 (Community Server Bundle), D049 (Workshop Asset Formats & P2P Distribution)
> **Philosophy:** IC defines its own standard. Existing implementations are studied for protocol understanding, architectural patterns, and lessons learned — not as mandatory dependencies. If the best path is a purpose-built P2P engine, that's what gets built.

---

## 1. IC Workshop P2P Requirements

IC's Workshop distributes game content (mods, maps, asset packs, total conversions: 5 MB – 2 GB packages) to players. The distribution system must:

1. **Work across all IC platforms** — desktop (Windows, macOS, Linux), browser (WASM), Steam Deck, mobile (planned)
2. **Scale without infrastructure cost** — popular content should get faster to download, not more expensive to host
3. **Be BitTorrent-compatible** — leverage the proven, battle-tested BitTorrent wire protocol and ecosystem where it makes sense
4. **Include integrated tracking** — peer coordination built into the Workshop server, not a separate service
5. **Support browser↔desktop interop** — WASM builds must participate in the same swarm as desktop clients (via WebRTC)
6. **Provide bandwidth control** — configurable upload/download limits, seeding policies
7. **Use content-aware piece strategies** — rarest-first, endgame mode, lobby-priority seeding (D049)
8. **Be pure Rust** — no C/C++ FFI, no Boost, no OpenSSL dependency chain. WASM compilation must be possible.
9. **Be GPL v3 compatible** — all dependencies must be license-compatible

---

## 2. The BitTorrent Protocol — What IC Needs From It

BitTorrent is a well-specified, battle-tested protocol family. IC's P2P engine should speak standard BT wire protocol where possible (interoperability with existing tools, proven correctness) and extend it where IC's use case demands.

### BEPs (BitTorrent Enhancement Proposals) Relevant to IC

| BEP    | Name                                       | IC Relevance                                                                                                                             |
| ------ | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| BEP 3  | The BitTorrent Protocol Specification      | Core wire protocol. IC implements this.                                                                                                  |
| BEP 5  | DHT Protocol                               | Decentralized peer discovery. Enables trackerless operation.                                                                             |
| BEP 9  | Extension for Peers to Send Metadata Files | Magnet link support. Download torrent metadata from peers.                                                                               |
| BEP 10 | Extension Protocol                         | Extensibility handshake. IC can advertise custom extensions.                                                                             |
| BEP 17 | HTTP Seeding (GetRight-style)              | Web seeding — download pieces from HTTP mirrors alongside BT peers. `httpseeds` torrent key. See `p2p-distribute-crate-design.md` § 2.8. |
| BEP 19 | WebSeed — HTTP/FTP Seeding (Hoffman-style) | Web seeding — `url-list` torrent key maps files to web URLs. Complementary to BEP 17. See `p2p-distribute-crate-design.md` § 2.8.        |
| BEP 23 | Tracker Returns Compact Peer Lists         | Bandwidth-efficient tracker responses.                                                                                                   |
| BEP 29 | uTP — Micro Transport Protocol             | UDP-based transport that doesn't saturate home connections.                                                                              |
| BEP 52 | The BitTorrent Protocol Specification v2   | Merkle tree piece hashing. Better integrity, per-file deduplication. May inform IC's content-addressed store (D049).                     |

### Where IC May Diverge or Extend

- **Package-aware piece prioritization:** Standard BT treats all pieces equally. IC knows which `.icpkg` a piece belongs to, which lobby needs it, and which player requested it. Priority channels (lobby-urgent > user-requested > background) are an IC-specific scheduling layer on top of standard piece selection.
- **Authenticated announce:** IC's tracker requires per-session tokens tied to client identity (D052 Ed25519). Standard BT trackers are anonymous. IC's announce protocol extends the standard with signed authentication.
- **Workshop metadata integration:** Standard BT distributes raw bytes. IC's system integrates with the Workshop registry — manifest lookup, dependency resolution, and version checking happen before the BT transfer begins.
- **WebRTC transport:** Standard BT uses TCP/uTP. Browser builds need WebRTC data channels (the WebTorrent approach). IC implements BT wire protocol over WebRTC for browser↔desktop interop.

---

## 3. Existing Implementations — Study Reference

These are studied for protocol understanding, architectural patterns, and lessons — not as hard dependencies. IC may use components from these where they fit without compromise, but the default stance is to implement what IC needs.

### 3.1 Client Libraries

#### librqbit (Rust)
- **Repository:** github.com/ikatson/rqbit
- **License:** Apache-2.0
- **What to study:** Session-based API design, tokio async architecture, uTP implementation (`librqbit-utp`), DHT implementation, piece selection logic, resume/persistence model. The best reference for how a modern Rust BT client structures its internals.
- **Could IC use it directly?** Possibly as a starting point or dependency for the core BT wire protocol on desktop, if it doesn't constrain IC's requirements. It lacks WebRTC transport, tracker functionality, bandwidth throttling API, and WASM support — all of which IC needs. If IC builds its own P2P engine, librqbit's source is the best Rust-native reference implementation to study.

#### libtorrent-rasterbar (C++)
- **Repository:** github.com/arvidn/libtorrent
- **License:** BSD-3-Clause
- **What to study:** The gold standard for BT implementation. 20+ years of protocol knowledge baked in. Study its piece picker algorithms, choking/unchoking strategies, DHT implementation, uTP congestion control, and bandwidth management. Arvid Norberg's blog posts and design documents are the best available literature on practical BT implementation.
- **Could IC use it directly?** No. C++ dependency chain (Boost, OpenSSL) prevents WASM. All Rust bindings are abandoned. But its source code and documentation are invaluable reference material for anyone implementing BT from scratch.

#### webtorrent-rs (Rust)
- **Repository:** crates.io/crates/webtorrent-rs
- **License:** Unknown
- **What to study:** WebRTC transport implementation over BT wire protocol. The only Rust attempt at WebTorrent. Maintainer warns it's experimental ("vibe-coded"). Study for architectural ideas on bridging WebRTC and BT, not as a dependency.

#### WebTorrent (JavaScript)
- **Repository:** github.com/webtorrent/webtorrent
- **License:** MIT
- **What to study:** The canonical WebTorrent implementation. How it bridges standard BT peers and WebRTC peers. Signaling via WebSocket trackers. The hybrid swarm model where desktop and browser clients interoperate. This is the reference for IC's browser↔desktop interop design.

### 3.2 Tracker Implementations

#### aquatic (Rust)
- **Repository:** github.com/greatest-ape/aquatic
- **License:** Apache-2.0
- **What to study:** High-performance pure-Rust tracker. `aquatic_udp` handles standard UDP tracker protocol. `aquatic_ws` handles WebTorrent signaling via WebSocket — the critical piece for browser peer discovery. Production-proven at ~80K req/s. Study its protocol crates (`aquatic_udp_protocol`, `aquatic_ws_protocol`) for the tracker protocol implementation details.
- **Could IC use it?** The protocol crates could potentially be used to implement IC's embedded tracker. The standalone server binaries require Linux 5.8+ (io_uring) and are designed as separate processes, not embeddable libraries. IC may implement its own tracker using aquatic's protocol crates as a foundation, or implement the (relatively simple) tracker protocol from scratch to avoid the io_uring constraint.

#### Torrust Tracker (Rust)
- **Repository:** github.com/torrust/torrust-tracker
- **License:** AGPL-3.0-only
- **What to study:** More portable than aquatic (Axum-based, no io_uring). SQLite/MySQL persistence. Management API. Study for tracker administration patterns. **Cannot be embedded** due to AGPL-3.0 license — would require making the entire engine source available to network users.

#### chihaya (Go)
- **Repository:** github.com/chihaya/chihaya
- **License:** BSD-2-Clause
- **What to study:** Pluggable middleware architecture. Used at scale by Facebook. Study for tracker extensibility patterns (pre-hook, post-hook middleware for rate limiting, authentication, metrics).

### 3.3 Related P2P Systems (Non-BitTorrent)

#### IPFS / libp2p
- **What to study:** Content-addressed storage (CAS) model — IC already uses CAS for Workshop blobs (D049). libp2p's modular transport architecture (TCP, WebRTC, QUIC as swappable transports). Rust implementation (`rust-libp2p`) is mature.
- **Relevance:** IC's CAS blob store (SHA-256 addressed) is conceptually similar to IPFS. The transport modularity pattern (trait-based transport selection) aligns with IC's `NetworkModel` trait philosophy.

#### Dat / Hypercore Protocol
- **What to study:** Append-only merkle tree structure. Version-aware content distribution. Relevant to Workshop's versioned packages — a new version of a mod shares unchanged pieces with the previous version.

### 3.4 Multi-Source Download & Web Seeding References

These projects are studied specifically for their HTTP+P2P hybrid download architecture, where HTTP mirrors and BT peers serve pieces concurrently through a unified scheduler. IC's web seeding design (`p2p-distribute-crate-design.md` § 2.8) draws from this work.

#### aria2 (C++)
- **Repository:** github.com/aria2/aria2
- **License:** GPL-2.0-or-later
- **What to study:** The gold standard for multi-protocol, multi-source downloading. Downloads from HTTP/HTTPS/FTP alongside BitTorrent simultaneously — the same file fetched from mirrors and BT peers at once. Metalink support (RFC 5854, RFC 6249) for structured mirror lists. Adaptive chunk scheduling across heterogeneous sources. The key insight: **model HTTP mirrors and BT peers as different transports for the same piece scheduler**, not as separate download mechanisms. Study its `RequestGroup` → `PieceStorage` → `DownloadCommand` pipeline for how it unifies the sources.
- **Could IC use it directly?** No. GPL, C++, not embeddable. But its architecture is the primary inspiration for `HttpSeedPeer`'s virtual peer model.

#### axel (C)
- **Repository:** github.com/axel-download-accelerator/axel
- **License:** GPL-2.0-or-later
- **What to study:** Lightweight HTTP-only multi-connection download accelerator. Opens N connections to the same URL (or multiple mirrors) and downloads different byte ranges in parallel. Simpler than aria2 but demonstrates the core Range-request splitting logic cleanly. Useful reference for IC's HTTP Range handling in `HttpSeedPeer`.

#### Dragonfly2 (Go)
- **Repository:** github.com/dragonflyoss/dragonfly
- **License:** Apache-2.0
- **What to study:** CNCF P2P-based container image and file distribution system. Uses "supernode" architecture where a CDN-like node (similar to web seed) provides initial pieces while a P2P mesh distributes laterally. Relevant to IC's Workshop server model where the server is both HTTP mirror and BT seeder. Study for CDN↔P2P hybrid piece scheduling and adaptive source selection.

#### Uber Kraken (Go)
- **Repository:** github.com/uber/kraken
- **License:** Apache-2.0
- **What to study:** Docker registry with P2P distribution. Uses modified BT protocol for container layer distribution. Origin servers act as permanent "seed peers" (analogous to IC's web seed model). Study for its tracker-agent architecture and how it handles mixed HTTP+P2P transfers at scale (~50k hosts simultaneously).

#### Nydus (Rust)
- **Repository:** github.com/dragonflyoss/nydus
- **License:** Apache-2.0
- **What to study:** Rust container image distribution with on-demand piece fetching from multiple backends (registry/HTTP/P2P). Relevant to IC's streaming piece selection mode. Study for how Rust async code handles concurrent fetch from heterogeneous sources.

#### Transmission (C)
- **Repository:** github.com/transmission/transmission
- **License:** GPL-2.0-or-later
- **What to study:** Mature BT client with BEP 17/19 web seeding support. The `tr_webseed` implementation shows how to model HTTP web seeds as peers in a standard BT client — same approach IC's `HttpSeedPeer` uses. Study for per-seed request throttling, failure handling, and byte-range-to-piece mapping.

---

## 4. Architectural Decisions for IC's P2P Engine

Based on studying the above, here are the key architectural questions and the IC-appropriate answers:

### Build vs. Adopt

| Component               | Decision                                          | Reasoning                                                                                                                                                                                         |
| ----------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **BT wire protocol**    | Build (or adapt from librqbit if it fits)         | Core to IC's functionality. Must support IC-specific extensions (auth, priority channels, WebRTC transport). Too important to be constrained by an external library's API.                        |
| **BT tracker protocol** | Build (potentially using aquatic protocol crates) | Simple protocol. IC needs it embedded in `ic-server`, not as a separate process. Must integrate with IC's authenticated announce.                                                                 |
| **WebRTC transport**    | Build                                             | No production Rust implementation exists. IC implements BT wire protocol over WebRTC data channels using `web-sys` (browser) and a native WebRTC stack (desktop bridge).                          |
| **DHT**                 | Adopt or build                                    | DHT (BEP 5) is complex but well-specified. Could use librqbit's DHT implementation if available as a standalone crate, or implement from the BEP spec.                                            |
| **uTP**                 | Adopt or build                                    | uTP (BEP 29) is a UDP congestion control protocol. `librqbit-utp` is a standalone crate that could be used directly. Alternatively, QUIC (via `quinn`) provides similar benefits with modern TLS. |
| **Bencode**             | Adopt                                             | Trivial format. Multiple Rust crates exist (`serde_bencode`, `bt_bencode`). No reason to rewrite.                                                                                                 |

### Transport Strategy

```
Desktop (Windows/macOS/Linux)
├── TCP     — standard BT, always available
├── uTP     — UDP-based, doesn't saturate connections
└── WebRTC  — for bridging with browser peers (Workshop server acts as bridge)

Browser (WASM)
└── WebRTC  — only option, via web-sys / WebRTC data channels

Workshop Server (ic-server with workshop capability)
├── TCP     — seeds to desktop peers
├── uTP     — seeds to desktop peers
└── WebRTC  — seeds to browser peers, bridges the two swarms
```

The Workshop server is the **bridge node** — it speaks all transports simultaneously, allowing desktop and browser clients to participate in the same logical swarm even though they can't connect directly.

### Peer Scoring & Piece Selection

IC's piece selection is more sophisticated than standard BT because IC has domain knowledge:

1. **Lobby-urgent priority:** When a player joins a lobby and needs a mod, that mod's pieces get maximum priority across all peers in the lobby. Peers who already have the content seed directly to the joining player.
2. **Rarest-first within priority tier:** Standard BT rarest-first within each priority level.
3. **Endgame mode:** For the last ~5 pieces, duplicate requests to multiple peers to prevent stall.
4. **Background pre-fetch:** Popular/trending content can be pre-fetched during idle time.

Peer scoring (D049 § peer scoring) uses a weighted multi-dimensional score:
```
PeerScore = Capacity(0.4) + Locality(0.3) + SeedStatus(0.2) + LobbyContext(0.1)
```

### The Workshop Server as Permanent Seed

The Workshop capability in `ic-server` is not a web server that also seeds — it is a **BitTorrent seeder that also serves metadata**:

1. Permanently seeds all hosted content via BT wire protocol
2. Runs an embedded tracker for peer coordination
3. Serves a thin REST API for package manifests, search, and dependency resolution
4. Bridges desktop and browser swarms via dual TCP+WebRTC transport

"Hosting a Workshop" = running a dedicated P2P seeder. The metadata API is secondary. The bytes flow over BitTorrent.

---

## 5. Existing Ecosystem Landscape Summary

| Project              | Language   | License    | IC Use                                                                                                                                          |
| -------------------- | ---------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| librqbit             | Rust       | Apache-2.0 | **Study reference.** Best Rust BT implementation. Possible component if it doesn't constrain IC.                                                |
| libtorrent-rasterbar | C++        | BSD-3      | **Study reference only.** Best BT docs and algorithms. Cannot embed (C++, no WASM).                                                             |
| aquatic              | Rust       | Apache-2.0 | **Study reference.** Protocol crates may be usable for tracker.                                                                                 |
| WebTorrent (JS)      | JavaScript | MIT        | **Study reference.** Canonical WebRTC↔BT bridge design.                                                                                         |
| webtorrent-rs        | Rust       | Unknown    | **Study reference.** Only Rust WebTorrent attempt. Experimental.                                                                                |
| Torrust              | Rust       | AGPL-3.0   | **Study reference only.** License prevents embedding.                                                                                           |
| chihaya              | Go         | BSD-2      | **Study reference only.** Middleware architecture patterns.                                                                                     |
| libp2p (Rust)        | Rust       | MIT        | **Study reference.** Transport modularity, CAS model.                                                                                           |
| aria2                | C++        | GPL-2.0    | **Study reference only.** Gold standard for multi-source HTTP+BT hybrid downloading. Primary inspiration for `HttpSeedPeer` virtual peer model. |
| axel                 | C          | GPL-2.0    | **Study reference only.** Clean HTTP Range-request splitting implementation.                                                                    |
| Dragonfly2           | Go         | Apache-2.0 | **Study reference.** CDN↔P2P hybrid scheduling, supernode architecture.                                                                         |
| Uber Kraken          | Go         | Apache-2.0 | **Study reference.** Origin-server-as-seed model at scale.                                                                                      |
| Nydus                | Rust       | Apache-2.0 | **Study reference.** Rust async multi-backend fetching patterns.                                                                                |
| Transmission         | C          | GPL-2.0    | **Study reference only.** Mature BEP 17/19 web seed implementation (`tr_webseed`).                                                              |

---

## 6. Key Lessons from Studying the Ecosystem

1. **The BT wire protocol is simple.** The hard parts are DHT, uTP congestion control, and WebRTC signaling — not the core peer-to-peer transfer. IC can implement the wire protocol straightforwardly.

2. **WebRTC↔BT bridging is an unsolved problem in Rust.** No production library exists. WebTorrent (JS) proves the concept works. IC will need to build this — it's the hardest piece, but also the most valuable (enables browser builds to participate in Workshop P2P).

3. **Trackers are trivially simple.** The tracker protocol is a thin announce/scrape API. IC should embed this directly in `ic-server` rather than depending on an external tracker. The only complexity is WebSocket signaling for WebTorrent peers, which aquatic's protocol crates document well.

4. **Content-addressed storage is the right model.** Both IPFS and IC's Workshop (D049) use SHA-256 content addressing. This enables cross-version deduplication — when a mod updates, only changed pieces need re-downloading.

5. **uTP vs. QUIC is an open question.** Standard BT uses uTP (BEP 29) for UDP transport. QUIC (`quinn` crate, pure Rust, mature) provides similar congestion control with modern TLS and multiplexing. IC could speak uTP for BT compatibility and QUIC for IC-to-IC optimized transfers. This is a future optimization, not a Phase 4–5 requirement.

6. **Peer scoring with domain knowledge is IC's advantage.** Standard BT clients are generic. IC knows which lobby a player is in, which mod they need, how popular content is, and where peers are geographically. This domain knowledge produces better piece selection than any generic BT client can achieve.

7. **HTTP mirrors and BT peers are the same thing to a piece scheduler.** The aria2 insight: don't build separate HTTP and BT download paths — model HTTP mirrors as virtual peers in the same connection pool, scored and scheduled alongside BT peers. The scheduler requests pieces from the fastest available source regardless of transport. IC implements this via `HttpSeedPeer` (BEP 17/19) in `p2p-distribute`'s `webseed` feature. This is particularly valuable during initial swarm bootstrapping and for long-tail content with few seeders.

---

## 7. Extended Protocol Survey — What Else Could IC Support?

This section evaluates protocols beyond BT+HTTP that have been or could be asked about. Each gets a clear verdict and rationale so we don't revisit settled ground.

**Evaluation axis:** The only question that matters is whether a protocol can act as a **virtual piece contributor** — a source that concurrently feeds pieces into the same piece scheduler that BT peers use. This is how `HttpSeedPeer` works: the scheduler asks it for bytes X–Y of file F, it issues a Range request, returns the block, the scheduler verifies it against the piece hash, and moves on. For a protocol to be a virtual peer in this model it must support **random-access byte-range reads** — the ability to fetch an arbitrary slice of a file on demand, not the entire file.

Protocols that can't do random-access reads are not virtual peers. They may still be useful at a different layer (pre-download delta tools, descriptor formats, CLI import helpers), but they do not belong in `p2p-distribute`.

### 7.1 SCP (Secure Copy Protocol)

**Verdict: No — fails the byte-range test at the protocol level.**

SCP provides no random-access API. It copies a file start-to-finish over an SSH channel. There is no "give me bytes 262144–524288" operation — SCP doesn't know what a byte offset is. As a virtual piece contributor it cannot work at all.

Beyond the byte-range failure: SCP requires SSH credentials (not appropriate for public Workshop downloads) and transfers one file per connection with no multiplexing.

**Only tangential relevance:** Server operators bootstrapping a Workshop server might use `scp` to push initial content from their machine. This is a one-time admin workflow for `ic` CLI (`ic server seed-from scp://...`), not a transfer protocol — and even then HTTPS is cleaner.

### 7.2 SFTP (SSH File Transfer Protocol)

**Verdict: No — passes the byte-range test but fails on credentials.**

SFTP does support random-access reads via `SSH_FXP_READ` (offset + length). Technically, you could model an `SftpSeedPeer` — connect to an SSH daemon, authenticate, then issue `SSH_FXP_READ` requests for arbitrary byte ranges, mapping them to BT piece indices. The scheduler would see it like any other peer.

But credentials are the fatal issue. Workshop downloads are public and anonymous. SFTP requires SSH daemon access and an SSH key or password on every client connecting. No Workshop server would grant SSH access to arbitrary downloaders, and no player should need SSH credentials to download a mod.

Conclusion: the virtual-peer model is technically achievable but the deployment model makes it irrelevant for public content distribution. Admin-only tool at most.

### 7.3 rsync

**Verdict: No for `p2p-distribute` — wrong protocol model entirely.**

rsync is not a random-access byte-range protocol. It computes a whole-file delta: the client sends the server a list of block checksums for the file it already has; the server replies with only the changed blocks. The protocol is inherently two-pass and stateful — both sides must exchange the full checksum list before any data transfer begins.

This makes rsync fundamentally incompatible with the virtual-peer model. The piece scheduler cannot ask rsync for piece #42 independently — rsync doesn't work that way. It also requires a running `rsyncd` daemon or SSH access, so Workshop servers wouldn't expose it to arbitrary downloaders.

**What rsync does well** is computing update deltas — relevant to the *pre-download* phase of a mod update, not concurrent piece contribution. The HTTP-native equivalent (zsync, § 7.4) achieves the same delta without a daemon and with byte-range reads the scheduler can use.

### 7.4 zsync (Delta Transfer over HTTP)

**Verdict: Not a virtual peer — a pre-download deduplication tool. Worth designing for `workshop-core` update logic.**

zsync uses standard HTTP `Range:` headers underneath, so it *is* byte-range-capable. But it doesn't function as a concurrent piece contributor — it's a file-update tool that runs **before** a download starts, not alongside a swarm.

zsync (Colin Phipps, 2004) is rsync over HTTP. The server publishes a `.zsync` file alongside each package:

```
SHA-1: <full-file-SHA1>
Length: 524288000
Blocksize: 4096
Hash-Lengths: 1,4,5
URL: awesome-tanks-2.1.0.icpkg
<concatenated per-block checksums (Adler-32 + MD4)>
```

The client:
1. Downloads the `.zsync` file (~200 KB for a 500 MB package)
2. Computes which 4 KB blocks it already has locally (from v2.0.0 of the same mod)
3. Requests only missing blocks via standard `Range: bytes=<start>-<end>` headers
4. Updates the local file in place

No special server daemon. Any HTTP server. The server doesn't even need to know zsync exists — it just serves the file and the `.zsync` descriptor.

**IC fit:** Workshop mod updates. When a user goes from v2.0.0 to v2.1.0 of a 500 MB total conversion, zsync means downloading megabytes, not gigabytes. This is especially impactful for large asset packs where the content changes incrementally.

**Scope placement:**
- `p2p-distribute` does NOT need to implement zsync — it operates on whole BT pieces, not sub-piece block diffs.
- `workshop-core` (the IC integration layer above `p2p-distribute`) is the right place: detect version delta scenario → use zsync HTTP to produce an updated local file → hand the verified file to `p2p-distribute` for seeding onward to peers.
- `ic-server` Workshop capability generates `.zsync` files alongside `.icpkg` packages at upload time.

**Reference implementations:**
- Original C: https://github.com/cph6/zsync (Artistic License + GPL-2.0)
- Rust: No production crate exists. IC would need to implement the `.zsync` file format parser and the rolling checksum matching. ~500–800 lines.

**Why not a virtual peer:** zsync's block checksums are computed against the *existing local file*, not against BT piece boundaries. The `.zsync` descriptor uses 4 KB blocks while BT pieces are 256 KB. Feeding zsync block responses into the BT piece picker would require cross-referencing two different block-size grids, verifying partial pieces, and reassembling them — net complexity far exceeds just downloading the missing BT pieces normally.

The correct model: zsync runs as a pre-step in `workshop-core` to construct the updated local file, then hands the verified `.icpkg` to `p2p-distribute` for seeding onward. The download is complete before `p2p-distribute` sees it.

**BEP 52 relationship:** BEP 52 (BT v2) provides per-file Merkle trees that enable similar deduplication at the BT layer — unchanged files don't transfer at all in a multi-file v2 torrent. zsync and BEP 52 are complementary: zsync handles the HTTP/update-delta path, BEP 52 handles the P2P swarm deduplication path.

### 7.5 Metalink (RFC 5854 / RFC 6249)

**Verdict: Not a transport — a mirror-list descriptor. Generate in `ic-server` for interoperability with external tools.**

Metalink is not a protocol that transfers bytes — it has no byte-range read operation. It is a descriptor format: an XML file (or HTTP `Link:` headers) that lists all the places you can download a file from:
- Multiple mirror URLs (HTTP, FTP, BT magnet links)
- Per-segment checksums for integrity verification
- File metadata (size, hash, signature)

As a virtual peer model, Metalink itself cannot contribute pieces — the underlying sources it describes (HTTP mirrors, BT) are the actual transports. IC's `HttpSeedPeer` already consumes HTTP mirror URLs from the Workshop API directly, so `p2p-distribute` gets no benefit from parsing Metalink.

**Value is interoperability with external tools.** A user who downloads with aria2, uGet, or JDownloader gets the same multi-source effect (BT + all HTTP mirrors simultaneously) if `ic-server` generates a `.metalink` per package and emits RFC 6249 `Link:` headers. This is ~100–200 lines of XML serialization in `ic-server` at publish time — the Workshop API already has all the mirror URLs.

### 7.6 HTTP/3 (QUIC-based)

**Verdict: Yes, virtual-peer-capable — same model as HTTP/1.1 web seeding, better transport. Monitor reqwest maturity.**

HTTP/3 runs on QUIC, providing:
- No head-of-line blocking across concurrent piece requests (unlike HTTP/1.1 pipelining and HTTP/2)
- Connection migration (IP address changes don't kill the download — relevant for mobile/Steam Deck)
- 0-RTT reconnection after network interruption
- Built-in TLS 1.3

HTTP/3 fully supports `Range: bytes=` headers — byte-range reads work identically to HTTP/1.1 from the piece scheduler's perspective. An `HttpSeedPeer` using HTTP/3 is the same virtual peer, just with a better transport underneath (no head-of-line blocking across concurrent piece requests, connection migration for mobile/Steam Deck, 0-RTT reconnection).

IC's web seeding uses `reqwest`. HTTP/3 is already available behind `reqwest = { features = ["http3"] }` (backed by `h3` + `quinn`). Enabling it for web seed requests is a feature-flag toggle in `p2p-distribute`'s `Cargo.toml` — no architecture change.

**Action:** Track reqwest HTTP/3 stability. The correct Cargo feature definition is: declare `reqwest-http3` as a separate optional dependency (`reqwest-http3 = { package = "reqwest", version = "...", features = ["http3"], optional = true }`) and define `webseed-http3 = ["webseed", "dep:reqwest-http3"]`. Note: `dep:reqwest/http3` is not valid Cargo syntax — you cannot activate features of a dependency inline in the feature table. The gated flag is disabled by default; enable it when reqwest's HTTP/3 support is stable.

### 7.7 IPFS / Content IDs

**Verdict: Two modes — gateway (already works as HTTP), native bitswap (out of scope).**

**IPFS HTTP gateway mode:** An IPFS gateway URL (`https://ipfs.io/ipfs/<CID>/file.icpkg`) is just HTTPS with Range support. An `HttpSeedPeer` can point at it today with zero changes — it's byte-range-capable and participates in the piece scheduler exactly like any other HTTP mirror. If a Workshop package is pinned to IPFS, its gateway URL can be listed in `url-list` (BEP 19) and IC downloads from it concurrently with BT peers.

**IPFS native bitswap protocol:** The native IPFS peer protocol (bitswap) sends 256 KB blocks (close to IC's 256 KB BT piece size), is content-addressed by CID, and could theoretically be modeled as an `IpfsSeedPeer`. But implementing bitswap + libp2p is a full protocol stack (~20 crates via `rust-libp2p`) for marginal benefit over the gateway path.

**Action:** Reserve an `ipfs_cid` field in the Workshop package registry schema. The gateway URL (derived trivially from the CID) feeds into `url-list` as a normal HTTP seed — no extra code in `p2p-distribute`. Native bitswap is out of scope.

### 7.8 WebDAV

**Verdict: Already a virtual peer — works today with zero changes.**

WebDAV is HTTP with extra verbs. The `Range:` header works identically against WebDAV servers — the server ignores the extra verbs and honors Range requests like any HTTP server. A NextCloud, ownCloud, Box, or Seafile instance serving a Workshop package is a valid `HttpSeedPeer` today just by listing its URL in `url-list` (BEP 19).

**Action:** None. Document in user-facing Workshop setup guides so operators know their NextCloud works as a web seed mirror out of the box.

### 7.9 FTP / FTPS

**Verdict: Technically byte-range capable, but dead protocol — no.**

FTP does support byte-range reads: the `REST <offset>` command followed by `RETR` resumes a download from a specific offset. aria2 uses this to implement FTP as a multi-source contributor alongside HTTP and BT. So FTP *could* be an `FtpSeedPeer` in the virtual-peer model.

But no Workshop server will expose FTP in 2025+, FTP sends credentials in plaintext (FTPS adds TLS but the active/passive mode negotiation remains NAT-hostile), and the implementation complexity of an `FtpSeedPeer` (parsing FTP's stateful command protocol, handling active vs. passive mode, connection multiplexing) is non-trivial. The benefit is zero since no mirror will use it. Skipped entirely.

### 7.10 Multicast UDP (NORM / FLUTE)

**Verdict: No — P2P swarm handles LAN parties naturally.**

Reliable multicast (NORM: RFC 5740, FLUTE: RFC 3926) allows a server to transmit data once on a multicast address and N clients receive it simultaneously. This could reduce server bandwidth at LAN parties.

**Why it doesn't matter for IC:**
- Consumer routers frequently disable IP multicast. A protocol that works only on managed networks is not suitable as IC's primary LAN distribution path.
- IC's P2P swarm already handles LAN parties efficiently via LSD (Local Service Discovery, BEP 14). Once one client has content, it seeds to nearby LAN peers. The server transmits once; LAN peers redistribute locally at LAN speeds (1 GbE+).
- Infrastructure complexity (NORM/FLUTE are underspecified in practice, with few production Rust implementations) far outweighs the marginal benefit over LSD+BT.

### 7.11 Summary Table

The **Byte-range?** column is the primary filter: if a protocol can't fetch arbitrary byte slices on demand, it cannot be a virtual piece contributor in `p2p-distribute`.

| Protocol              | Byte-range?        | Virtual peer in `p2p-distribute`?   | Other layer                       | Verdict                 |
| --------------------- | ------------------ | ----------------------------------- | --------------------------------- | ----------------------- |
| BT wire (BEP 3/5/…)   | ✅                  | ✅ Core                              | —                                 | Core                    |
| HTTP/1.1 + BEP 17/19  | ✅                  | ✅ `webseed` feature                 | —                                 | Shipped                 |
| HTTP/3 (QUIC)         | ✅                  | ✅ `webseed-http3` flag              | —                                 | Monitor reqwest         |
| WebDAV                | ✅ (it's HTTP)      | ✅ implicit via `webseed`            | —                                 | Already works           |
| IPFS gateway          | ✅ (it's HTTPS)     | ✅ list URL in `url-list`            | `ipfs_cid` registry field         | Already works           |
| IPFS native (bitswap) | ✅ (256 KB blocks)  | ❌ out of scope                      | —                                 | Too heavy               |
| SFTP                  | ✅ (`SSH_FXP_READ`) | ❌ requires SSH credentials          | admin CLI only                    | No                      |
| FTP/FTPS              | ✅ (`REST`+`RETR`)  | ❌ dead, no mirrors                  | —                                 | No                      |
| SCP                   | ❌ no random access | ❌ protocol fails                    | admin CLI only                    | No                      |
| rsync                 | ❌ whole-file diff  | ❌ wrong model                       | —                                 | No                      |
| zsync                 | ✅ (HTTP Range)     | ❌ pre-download tool, not concurrent | `workshop-core` update delta      | Design in workshop-core |
| Metalink              | ❌ descriptor only  | ❌ not a transport                   | `ic-server` generates `.metalink` | Design in ic-server     |
| Multicast/NORM        | ❌ push broadcast   | ❌ wrong model                       | —                                 | No                      |

**Priority order for action (highest to lowest):**
1. **zsync** — Highest impact for large mod update experience. Design in `workshop-core` Phase 5–6a.
2. **Metalink** — Low effort, unlocks multi-source for power users with aria2. Add to `ic-server` Workshop capability Phase 5.
3. **HTTP/3** — Zero architecture cost, add forward-compatible feature flag now.
4. **IPFS CID field** — Reserve schema field, no implementation, Phase 4 (registry design time).
5. Everything else — No.
