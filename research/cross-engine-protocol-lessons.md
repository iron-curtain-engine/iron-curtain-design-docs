# Cross-Engine Protocol Lessons — IRC, Steam, Hamachi, XLink Kai & Network Overlays

> **Status:** Research document. Gap analysis informing cross-engine discovery, social features, and transport resilience.
> **Date:** 2026-02

## 1. Executive Summary

Study of 7 real-world multiplayer/overlay systems reveals 11 gaps in IC's current cross-engine and community server design. The guiding principle from XLink Kai: **"be indistinguishable"** — IC should integrate so seamlessly with existing communities that players don't notice they're using a different engine.

## 2. Systems Studied

| System | Type | Key Lesson |
|---|---|---|
| **IRC (CnCNet)** | Chat lobby / game listing | Persistent social spaces; CnCNet proves IRC is still viable for game lobbies |
| **Matrix** | Federated messaging | Federated identity; room-based state; end-to-end encryption patterns |
| **Steam** | Platform overlay | Deep links, rich presence, voice chat, server browser UX |
| **Hamachi** | VPN tunneling | Zero-signup first experience; key-as-identity |
| **XLink Kai** | LAN broadcast tunneling | "Be indistinguishable from LAN"; transparent tunneling |
| **ZeroTier** | Software-defined networking | Mesh networking; peer-to-peer with relay fallback |
| **Nebula** | Overlay network (Slack) | Lightweight mesh; certificate-based identity |

## 3. Gap Analysis

### Gap 1: BrowseFilter Fields Unspecified

**Problem:** The `CommunityBridge::browse_games()` interface returns `Vec<GameListing>` but the filter criteria for the server browser are not specified.

**Recommendation:** Define `BrowseFilter` with concrete fields:

```rust
pub struct BrowseFilter {
    pub game_module: Option<GameModuleId>,
    pub engine: Option<EngineTag>,         // "ic", "openra", "cncnet"
    pub min_open_slots: Option<u8>,
    pub max_ping_ms: Option<u32>,
    pub status: Option<LobbyStatus>,       // Waiting, InProgress, Arena
    pub map_name: Option<String>,
    pub competitive: Option<bool>,         // ranked only
    pub has_friends: Option<bool>,         // games with friends
    pub text_search: Option<String>,       // fuzzy host/map name search
    pub cursor: Option<String>,            // pagination token
    pub limit: u16,
}
```

**Phase:** Phase 5 (multiplayer UI). Required before the game browser ships.

### Gap 2: CnCNet IRC Bridge

**Problem:** CnCNet's IRC-based lobby is the primary social space for the classic C&C competitive community. IC needs to appear in CnCNet's game listing.

**Recommendation:** Implement `CnCNetIrcBridge` as a `CommunityBridge` implementation. This is NOT a tracking server — CnCNet handles its own infrastructure. The bridge publishes IC games to CnCNet's IRC channels and reads CnCNet game listings for IC's browser.

Key implementation details:
- Connect to CnCNet's IRC server (well-known address)
- Publish game listings in CnCNet's expected format
- Parse CnCNet game listings into IC's `GameListing` type
- Handle CnCNet's nickserv-style authentication
- Respect CnCNet's channel conventions (per-game channels)

**Phase:** Phase 5 (cross-engine). Requires community coordination with CnCNet maintainers.

### Gap 3: Deep Link Sub-Schemes

**Problem:** IC's `ic://` URI scheme is specified but sub-schemes for joining specific community games are not defined.

**Recommendation:**
```
ic://cncnet/<channel>         → Join CnCNet game channel
ic://openra/<lobby-id>        → Join OpenRA lobby
ic://direct/<host:port>       → Direct connect
ic://relay/<server>/<game-id> → Join via IC relay
ic://workshop/<resource-id>   → Open Workshop resource
```

Deep links should work from web pages (browser → IC client), chat messages, and clipboard paste.

**Phase:** Phase 5. Standard URI registration.

### Gap 4: Cross-Community Identity Resolution

**Problem:** A player may have different identities across CnCNet (IRC nick), OpenRA (player name), and IC (Ed25519 identity key). No mechanism links them.

**Recommendation:** Optional identity linking — players can cryptographically associate their IC identity with external community identities. The linked identities are stored locally and optionally displayed in the player profile.

- **Not required:** Players can use IC without linking any external identity
- **Optional verification:** CnCNet nick verified by signing a message with the IC key and posting it to the CnCNet IRC channel
- **Behavioral history:** Cross-community behavioral reputation (if opt-in) — a player with good standing on CnCNet carries that reputation into IC cross-engine games

**Phase:** Phase 5+. Nice-to-have, not blocking.

### Gap 5: WebSocket/TCP 443 Fallback

**Problem:** ~5% of players are on networks that block UDP entirely (corporate, university, some mobile). IC's relay uses UDP.

**Recommendation:** Add `WebSocketFallback` to the transport fallback chain:

```rust
pub enum TransportFallbackChain {
    DirectUdp,           // Best: direct UDP to relay
    StunAssistedUdp,     // NAT traversal via STUN
    WebSocketFallback,   // TCP 443: bypasses most firewalls
}
```

The WebSocket transport carries the same relay protocol messages over a TCP WebSocket connection on port 443. The relay detects the transport type per-client — different players in the same game can use different transports.

**Phase:** Phase 5 (multiplayer). ~5% of players is significant enough to warrant the fallback.

### Gap 6: Persistent Social Arenas

**Problem:** IC's lobby model is transient — you create a game, play it, it disappears. There's no persistent social space where players hang out.

**Recommendation:** Add `LobbyStatus::Arena` for persistent social spaces (inspired by XLink Kai's persistent game rooms). Arenas are named lobbies that persist between games — players join an arena, chat, and launch games from within it. After a game ends, players return to the arena.

```rust
pub enum LobbyStatus {
    Waiting,     // Standard: waiting for players
    InProgress,  // Standard: game running
    Arena,       // Persistent: social space with game launching
}
```

**Phase:** Phase 5+. Social feature, not blocking for core multiplayer.

### Gap 7: Zero-Signup First Game

**Problem:** IC requires Ed25519 identity creation before play. This is a barrier for first-time players who just want to try the game.

**Recommendation:** Adopt Hamachi's key-as-identity principle — IC already generates an Ed25519 keypair during first-run setup (D069). Extend this to be completely automatic: no manual step needed. The player's first identity is anonymous but functional. Display name defaults to a generated callsign (e.g., "Commander_7K3"). Players can later claim a display name, link external identities, etc.

The key insight: **the first game should require zero explicit setup** beyond launching the game. Identity creation happens silently in the background.

**Phase:** Phase 3–5 (first-run wizard). Ensure D069 implements this.

### Gap 8: Transport-Level Reconnection Resilience

**Problem:** Brief network interruptions (Wi-Fi handoff, mobile signal loss, ISP blip) currently cause disconnection and game loss.

**Recommendation:** Add session resumption at the transport level. A reconnecting client presents its session token and resumes from the last acknowledged frame sequence number. The relay buffers unacknowledged frames for a configurable window (default: 10 seconds).

Key requirements:
- Session token survives transport disconnect
- Relay maintains per-player frame buffer for reconnection window
- Reconnected client receives buffered frames and catches up
- If reconnection window expires, standard disconnect/AI-takeover flow applies
- The sim doesn't know about the disconnect (orders queue, idle order injected by relay)

**Phase:** Phase 5 (multiplayer resilience). Important for mobile/WASM targets.

### Gap 9: Voice P2P Verification

**Problem:** The D059 communication design specifies Opus VoIP but doesn't clarify whether voice is relay-routed or P2P.

**Recommendation:** Confirm that D059's voice design supports P2P audio (direct peer-to-peer UDP) with relay fallback. P2P voice avoids relay bandwidth costs and reduces latency. The relay provides STUN/TURN for NAT traversal and fallback for peers that can't establish direct connections.

**Phase:** N/A (design clarification). Verify against D059 spec.

### Gap 10: LAN Mesh Assist for Embedded Relay

**Problem:** When a player hosts an embedded relay (listen server) on LAN, other LAN players should connect directly rather than going through the host's relay.

**Recommendation:** Add mDNS-based LAN mesh discovery. IC clients on the same LAN broadcast their presence via mDNS. The embedded relay detects LAN peers and facilitates direct LAN connections between them, reducing relay load and latency.

This extends the existing D072 portable server mDNS discovery to client-to-client mesh on LAN.

**Phase:** Phase 5. Nice-to-have for LAN party UX.

### Gap 11: Geographic Relay Coverage Targets

**Problem:** No explicit geographic coverage targets for IC's relay infrastructure.

**Recommendation:** Define minimum coverage targets per phase:
- **Phase 5 launch:** NA-East, EU-West (covers ~60% of expected player base)
- **Phase 5+:** NA-West, EU-Central, Oceania (covers ~85%)
- **Community-operated:** Community servers (D074) fill remaining regions

Use relay latency measurement (ping) to automatically route players to the lowest-latency relay.

**Phase:** Phase 5 (infrastructure planning).

## 4. The "Be Indistinguishable" Integration Principle

XLink Kai's design philosophy: **the tunneled game should be indistinguishable from a LAN game.** Applied to IC:

- IC appearing in CnCNet's lobby should feel identical to a native CnCNet game listing
- IC appearing in OpenRA's browser should feel identical to a native OpenRA game listing
- A CnCNet player joining an IC-hosted game should have the same experience as joining a CnCNet game (same protocol, same gameplay feel with classic-ea balance preset)

This principle means the `CommunityBridge` implementations must go beyond basic listing — they should support the full social interaction pattern of each community (chat, player info, game settings display).

## 5. Relationship to Design Documents

| Document | Gap | Change |
|---|---|---|
| `07-CROSS-ENGINE.md` | Gap 1, 2, 3, 6 | Add BrowseFilter, CnCNet bridge, deep links, arenas |
| `cross-engine/relay-security.md` | Gap 2, 10 | Add CnCNet trust tier, LAN mesh |
| `netcode/connection-establishment.md` | Gap 5, 8 | Add WebSocket fallback, session resumption |
| `decisions/09g/D059-communication.md` | Gap 9 | Clarify P2P voice model |
| `decisions/09g/D069-install-wizard.md` | Gap 7 | Ensure zero-signup first game |
| `decisions/09b/D074-community-server-bundle.md` | Gap 10, 11 | LAN mesh, geographic targets |
| `player-flow/multiplayer.md` | Gap 1, 6 | Browser filter UI, arena concept |
