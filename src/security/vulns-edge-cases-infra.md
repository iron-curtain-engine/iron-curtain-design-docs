## Vulnerability 53: Direct-Peer Replay Peer-Attestation Gap (Deferred Optional Mode)

### The Problem

**Severity: MEDIUM**

In deferred direct-peer modes (for example, explicit LAN/experimental variants without relay authority), replays are recorded locally by each client. There is no mutual attestation — a player can modify their local replay to remove evidence of cheating or alter match outcomes. Since there is no relay server to act as a neutral observer, replay integrity depends entirely on the local client.

### Mitigation

**Peer-attested frame hashes:** At the end of each sim tick, all peers exchange signed hashes of their current sim state (already required for desync detection). These signed hashes are recorded in each peer's replay file, creating a cross-attestation chain.

```rust
pub struct PeerAttestation {
    pub tick: SimTick,
    pub peer_id: PlayerId,
    pub state_hash: SimStateHash,
    pub peer_signature: Ed25519Signature,
}
```

**Replay reconciliation:** When a dispute arises, replays from all peers can be compared. Frames where peer-attested hashes diverge from the replay's recorded state indicate tampering. The attestation chain provides cryptographic proof of which peer's replay was modified.

**End-of-match summary signing:** At match end, all peers sign a match summary (final score, duration, player list, final state hash). This summary is embedded in all replays and can be independently verified.

**Phase:** This ships only if a direct-peer gameplay mode is explicitly enabled by future decision. Peer hash exchange is the corresponding exit criterion for that mode.

## Vulnerability 54: Anti-Cheat False-Positive Rate Targets

### The Problem

**Severity: MEDIUM**

The behavioral anti-cheat system (fog-of-war access patterns, APM anomaly detection, click accuracy outliers) has no defined false-positive rate targets. Without explicit thresholds, aggressive detection can alienate legitimate high-skill players while lenient detection misses actual cheaters.

### Mitigation

**Tiered confidence thresholds:**

| Detection Category       | Action          | Minimum Confidence | Max False-Positive Rate |
| ------------------------ | --------------- | ------------------ | ----------------------- |
| Fog oracle (maphack)     | Auto-flag       | 95%                | 1 in 10,000 matches     |
| APM anomaly (bot)        | Auto-flag       | 99%                | 1 in 100,000 matches    |
| Click precision (aimbot) | Review queue    | 90%                | 1 in 1,000 matches      |
| Desync pattern (exploit) | Auto-disconnect | 99.9%              | 1 in 1,000,000 matches  |

**Calibration dataset:** Before deployment, each detector is calibrated against a corpus of labeled replays: confirmed-cheating replays (from test accounts) and confirmed-legitimate replays (from high-skill tournament players). The false-positive rate is measured against the legitimate corpus.

**Graduated response:** No single detection event triggers a ban. The system uses a point-based accumulation model:
- Auto-flag: +1 point (decays after 30 days)
- Review-confirmed: +5 points (no decay)
- 10 points → temporary suspension (7 days) + manual review
- 25 points → permanent ban (appealable)

**Transparency report:** Aggregate anti-cheat statistics (total flags, false-positive rate, ban count) are published quarterly. Individual detection details are not disclosed (to avoid teaching cheaters to evade).

**Continuous calibration (post-deployment feedback loop):** The pre-deployment calibration corpus is a starting point, not a static artifact. Drawing from VACNet's continuous retraining model, IC maintains a living calibration pipeline:

1. **Confirmed-cheat ingestion:** When a human reviewer confirms a flagged player as cheating, the relevant replays are automatically added to the "confirmed-cheat" partition of the calibration corpus. When an appeal succeeds, the replays move to the "false-positive" partition.
2. **Threshold recalibration:** Population baselines (V12) and detection thresholds are recomputed weekly from the updated corpus. If the confirmed-cheat partition grows to include a new cheat pattern (e.g., a novel tool that evades the current entropy check), the recalibrated thresholds will detect it in subsequent matches.
3. **Model drift monitoring:** The ranking server tracks detection rates, false-positive rates, and appeal rates over rolling 90-day windows. A sustained increase in appeal success rate signals model drift — the thresholds are catching more legitimate players than they should. A sustained decrease in detection rate signals evasion evolution — cheat tools have adapted.
4. **Corpus hygiene:** The calibration corpus is versioned with timestamps. Replays older than 12 months are archived (not deleted) and excluded from active calibration to prevent stale patterns from anchoring thresholds.

**Phase:** Continuous calibration pipeline ships with ranked matchmaking (Phase 5). Initial corpus creation is the Phase 5 exit criterion; the feedback loop activates post-launch once human review generates confirmed cases.

**Phase:** False-positive calibration ships with ranked matchmaking (Phase 5). Calibration dataset creation is a Phase 5 exit criterion.

## Vulnerability 55: Platform Bug vs Cheat Desync Classification

### The Problem

**Severity: MEDIUM**

Desync events (clients diverging from deterministic sim state) can be caused by either legitimate platform bugs (floating-point differences across CPUs, compiler optimizations, OS scheduling) or deliberate cheating (memory editing, modified binaries). The current desync detection treats all desyncs uniformly, which can lead to false cheat accusations from genuine bugs.

### Mitigation

**Desync fingerprinting:** When a desync is detected, the system captures a diagnostic fingerprint: divergence tick, diverging state components (which ECS resources differ), hardware/OS info, and recent order history. Platform bugs produce characteristic patterns (e.g., divergence in physics-adjacent systems on specific CPU architectures) that differ from cheat patterns (e.g., divergence in fog-of-war state or resource counts).

**Classification heuristic:**

| Signal                                        | Likely Platform Bug | Likely Cheat        |
| --------------------------------------------- | ------------------- | ------------------- |
| Divergence in position/pathfinding only       | ✓                   |                     |
| Divergence in fog/vision state                |                     | ✓                   |
| Divergence in resource/unit count             |                     | ✓                   |
| Affects multiple independent matches          | ✓                   |                     |
| Correlates with specific CPU/OS combination   | ✓                   |                     |
| Divergence immediately after suspicious order |                     | ✓                   |
| Both peers report same divergence point       | ✓                   |                     |
| Only one peer reports divergence              |                     | ✓ (modified client) |

**Bug report pipeline:** Desyncs classified as likely-platform-bug are automatically filed as bug reports with the diagnostic fingerprint. These do not count toward anti-cheat points (V54).

**Phase:** Desync classification ships with anti-cheat system (Phase 5). Classification heuristic is a Phase 5 exit criterion.

## Vulnerability 56: RTL/BiDi Override Character Injection in Non-Chat Contexts

### The Problem

**Severity: LOW**

D059 (09g-interaction.md) defines RTL/BiDi sanitization for chat messages and marker labels, but other text-rendering contexts — player display names (see V46), package descriptions, mod names, lobby titles, tournament names — may not pass through the same sanitization pipeline, allowing BiDi override characters to create misleading visual presentations.

### Mitigation

**Unified text sanitization pipeline:** All user-supplied text passes through a single sanitization function before rendering, regardless of context. The pipeline:

1. Strip dangerous BiDi overrides (U+202A–U+202E) except in contexts where explicit direction marks are legitimate (chat with mixed-direction text uses U+2066–U+2069 isolates instead)
2. Normalize Unicode to NFC form
3. Apply context-specific length/width limits
4. Validate against context-specific allowed script sets

**Context registry:** Each text-rendering context (chat, display name, package title, lobby name, etc.) registers its sanitization policy. The pipeline applies the correct policy based on context, preventing bypass through context confusion.

**Cross-reference:** V46 (display name confusables), D059 (chat/marker sanitization), RTL/BiDi QA corpus categories E and F.

**Phase:** Unified text pipeline ships with UI system (Phase 3). Pipeline coverage for all user-text contexts is a Phase 3 exit criterion.

## Vulnerability 57: ICRP Local WebSocket Cross-Site WebSocket Hijacking (CSWSH)

### The Problem

**Severity: HIGH**

D071 (IC Remote Protocol) exposes a JSON-RPC 2.0 API over WebSocket on localhost for local tool integration (MCP server for LLM coaching, LSP server for mod development, debug overlay). A malicious web page opened in the user's browser can initiate a WebSocket connection to `ws://localhost:<port>` — the browser sends the page's cookies and the WebSocket handshake does not enforce same-origin policy by default. This is Cross-Site WebSocket Hijacking (CSWSH).

If the ICRP server accepts any incoming WebSocket connection without origin validation, a malicious page can:
- Read game state (fog-filtered for observer tier, but still leaks match information)
- Issue commands at the user's permission tier (if the local session has admin/mod permissions)
- Exfiltrate replay data, player statistics, or configuration

**Real-world precedent:** Jupyter Notebook, VS Code Live Share, and Electron apps have all patched CSWSH vulnerabilities in local WebSocket servers. The attack requires only that the user visits a malicious page while the game is running.

### Defense

**Origin header validation (mandatory):**

```rust
fn validate_websocket_upgrade(request: &HttpRequest) -> Result<(), IcrpError> {
    let origin = request.header("Origin");
    match origin {
        // No Origin header — non-browser client (curl, MCP, LSP). Allow.
        None => Ok(()),
        // Localhost origins — same-machine tools. Allow.
        Some(o) if o.starts_with("http://localhost") 
               || o.starts_with("http://127.0.0.1")
               || o.starts_with("http://[::1]")
               || o == "null" => Ok(()),
        // Any other origin — browser page from a different site. Reject.
        Some(o) => {
            tracing::warn!("CSWSH attempt blocked: origin={}", o);
            Err(IcrpError::ForbiddenOrigin)
        }
    }
}
```

**Challenge secret file:**

For elevated permission tiers (admin, mod, debug), the ICRP server generates a random 256-bit challenge secret and writes it to `<data_dir>/icrp-secret` with restrictive file permissions (0600 on Unix, user-only ACL on Windows). Connecting clients must present this secret in the first JSON-RPC message. A browser-based CSWSH attack cannot read local files, so this blocks privilege escalation even if origin validation is bypassed.

**HTTP fallback CORS whitelist:**

The HTTP fallback endpoint (D071) applies standard CORS headers: `Access-Control-Allow-Origin: http://localhost:<port>` (not `*`). Pre-flight `OPTIONS` requests validate the origin before processing.

**Additional hardening:**

- ICRP binds to `127.0.0.1` only by default. Binding to `0.0.0.0` requires explicit `--icrp-bind-all` flag with a console warning.
- WebSocket connections from browser origins (any `Origin` header present) are limited to observer-tier permissions regardless of challenge secret. Full permissions require a non-browser client (no `Origin` header).
- Rate limit: max 5 failed challenge attempts per minute per IP. Exceeding triggers 60-second lockout.

**Phase:** Ships with ICRP implementation (Phase 5). Origin validation and challenge secret are Phase 5 exit criteria. CSWSH is added to the threat model checklist for any future localhost-listening service.

## Vulnerability 58: Lobby Host Configuration Manipulation

### The Problem

**Severity: MEDIUM**

The lobby host selects game configuration (map, game speed, starting units, crates, fog settings, balance preset). In ranked play, certain configurations are restricted to the ranked whitelist (D055). But in unranked lobbies, a malicious host could:
- Change settings silently after players have readied up (race condition between ready confirmation and game start)
- Set configurations that advantage the host (e.g., changing starting position after seeing the map)
- Modify settings that affect ranked eligibility without clear indication to other players

### Defense

**Settings change notification:**

- Every lobby setting change emits a `LobbySettingChanged { key, old_value, new_value, changed_by }` message to all connected clients. The client UI displays a visible notification (toast + chat-style log entry) for each change.
- If any setting changes after a player has readied up, that player's ready status is automatically reset with a notification: "Settings changed — please re-confirm ready."

**Ranked configuration whitelist:**

- Ranked lobbies enforce a strict whitelist of allowed configurations defined in the ranking authority's `server_config.toml` (D064). The host cannot modify restricted settings when the lobby is marked as ranked.
- Settings outside the whitelist are grayed out in the lobby UI with a tooltip: "Locked for ranked play."
- The whitelist is versioned and signed by the ranking authority. Clients validate the whitelist version on lobby join.

**Match metadata recording:**

- All lobby settings at the moment of game start are recorded in the `CertifiedMatchResult` (V13) by the relay server. The ranking authority validates that recorded settings match the ranked whitelist before accepting the match result.
- This provides an audit trail — if a host exploits a race condition to change settings between ready and start, the recorded settings reveal the discrepancy.

**Phase:** Lobby setting notifications ship with lobby system (Phase 3). Ranked whitelist enforcement ships with ranked system (Phase 5). Match metadata recording ships with relay certification (Phase 5).

## Vulnerability 59: Ranked Spectator Minimum Delay Enforcement

### The Problem

**Severity: MEDIUM**

Observers in lockstep RTS games receive the full game state (all player orders). Without a delay, a spectator colluding with a player could relay opponent positions and orders in real time — effectively a maphack via social channel. D060 mentions observer delay as a concept but does not specify a minimum floor for ranked play.

### Defense

**Mandatory minimum observer delay for ranked matches:**

- Ranked matches enforce a **minimum 120-second (3,600-tick at 30 tps) observer delay**. This is a `ProtocolLimits`-style hard floor — not configurable below 120s by lobby settings, server config, or console commands.
- Implementation: The relay server buffers observer-bound state updates and releases them only after the delay window. The buffer is per-observer, not shared — each observer's view is independently delayed.
- The 120-second floor is chosen because it exceeds the tactical relevance window for most RTS engagements (build order scouting is revealed by 2 minutes anyway, and active combat decisions have ~5-10 second relevance).

**Tiered delay policy:**

| Context         | Minimum Delay | Configurable Above Floor |
| --------------- | ------------- | ------------------------ |
| Ranked match    | 120 seconds   | Yes (host can increase)  |
| Unranked match  | 0 seconds     | Yes (host sets freely)   |
| Tournament mode | 180 seconds   | Server operator sets     |
| Replay playback | N/A           | Full speed available     |

**Enforcement point:** The relay server is authoritative for observer delay — the client cannot bypass it by modifying local configuration. The delay value for each match is recorded in the `CertifiedMatchResult` metadata.

**Cross-reference:** D055 (ranked exit criteria — add observer delay to ranked integrity checklist), D060 (netcode parameter philosophy — observer delay is Tier 3, always-on for ranked), V13 (match certification metadata).

**Phase:** Observer delay enforcement ships with spectator system (Phase 5). The 120-second ranked floor is a Phase 5 exit criterion for D055.

## Vulnerability 60: Observer Mode RNG State Prediction

### The Problem

**Severity: LOW**

In lockstep multiplayer, all clients (including observers) process the same deterministic simulation. An observer therefore has access to the RNG state and can predict future random outcomes (e.g., damage rolls, crate spawns, scatter patterns). A colluding observer could inform a player of upcoming favorable/unfavorable RNG outcomes.

This is an inherent limitation of lockstep architecture — the RNG state is derivable from the simulation state that all participants share.

### Defense

**Acknowledged limitation with layered mitigations:**

This vulnerability is **not fully closable** in lockstep architecture without server-authoritative RNG (which would require a fundamentally different network model). Instead, layered mitigations reduce the practical impact:

1. **Ranked observer delay (V59):** The 120-second delay makes RNG prediction tactically irrelevant — by the time the observer sees the state, the predicted outcomes have already resolved.
2. **Fog-authoritative server (for high-stakes play):** In fog-authoritative mode, observers receive only visibility-filtered state, which limits (but does not fully eliminate) RNG state inference. This is the recommended mode for tournaments.
3. **RNG state opacity:** The sim's PRNG (deterministic, seedable) does not expose its internal state through any API observable to mods or scripts. Prediction requires reverse-engineering the PRNG sequence from observed outcomes — feasible but requires significant effort per-match.
4. **Post-match detection:** If a player consistently exploits predicted RNG outcomes (e.g., always attacking when the next damage roll is favorable), the behavioral model (V12) can detect the unnatural correlation between action timing and RNG outcomes over a sufficient sample of matches.

**Documentation note:** This is a known-and-accepted limitation common to all lockstep RTS games (SC2, AoE2, original C&C). No lockstep game has solved generic RNG prediction because the simulation state is, by design, shared. The fog-authoritative server eliminates this class entirely for any deployment willing to run server-side simulation.

**Phase:** Documentation only (no implementation change). Observer delay (V59) and fog-authoritative server provide the practical mitigations.

## Path Security Infrastructure

All path operations involving untrusted input — archive extraction, save game loading, mod file references, Workshop package installation, replay resource extraction, YAML asset paths — require boundary-enforced path handling that defends against more than `..` sequences.

The [`strict-path`](https://github.com/DK26/strict-path-rs) crate (MIT/Apache-2.0, compatible with GPL v3 per D051) provides compile-time path boundary enforcement with protection against 19+ real-world CVEs:

- **Symlink escapes** — resolves symlinks before boundary check
- **Windows 8.3 short names** — `PROGRA~1` resolving outside boundary
- **NTFS Alternate Data Streams** — `file.txt:hidden` accessing hidden streams
- **Unicode normalization bypasses** — equivalent but differently-encoded paths
- **Null byte injection** — `file.txt\0.png` truncating at null
- **Mixed path separator tricks** — forward/backslash confusion
- **UNC path escapes** — `\\server\share` breaking out of local scope
- **TOCTOU race conditions** — time-of-check vs. time-of-use via built-in I/O

**Integration points across Iron Curtain:**

| Component                            | Use Case                                            | `strict-path` Type |
| ------------------------------------ | --------------------------------------------------- | ------------------ |
| `ra-formats` (`.oramap` extraction)  | Sandbox extracted map files to map directory        | `PathBoundary`     |
| Workshop (`.icpkg` extraction)       | Prevent Zip Slip during package installation (D030) | `PathBoundary`     |
| Save game loading                    | Restrict save file access to save directory         | `PathBoundary`     |
| Replay resource extraction           | Sandbox embedded resources to cache (V41)           | `PathBoundary`     |
| WASM `ic_format_read_bytes`          | Enforce mod's allowed file read scope               | `PathBoundary`     |
| Mod file references (`mod.yaml`)     | Ensure mod paths don't escape mod root              | `PathBoundary`     |
| YAML asset paths (icon, sprite refs) | Validate asset paths within content directory (V33) | `PathBoundary`     |

This supersedes naive string-based checks like `path.contains("..")` (see V33) which miss symlinks, Windows 8.3 short names, NTFS ADS, encoding tricks, and race conditions. `strict-path`'s compile-time marker types (`PathBoundary` vs `VirtualRoot`) provide domain separation — a path validated for one boundary cannot be accidentally used for another.

**Adoption strategy:** `strict-path` is integrated as a dependency of `ra-formats` (archive extraction), `ic-game` (save/load, replay extraction), and `ic-script` (WASM file access scope). All public APIs that accept filesystem paths from untrusted sources take `StrictPath<PathBoundary>` instead of `std::path::Path`.

## Competitive Integrity Summary

Iron Curtain's anti-cheat is **architectural, not bolted on.** Every defense emerges from design decisions made for other reasons:

| Threat                                     | Defense                                           | Source                                   |
| ------------------------------------------ | ------------------------------------------------- | ---------------------------------------- |
| Maphack                                    | Fog-authoritative server                          | Network model architecture               |
| Order injection                            | Deterministic validation in sim                   | Sim purity (invariant #1)                |
| Order forgery (direct-peer optional mode)  | Ed25519 per-order signing                         | Session auth design                      |
| Lag switch                                 | Relay server owns the clock                       | Relay architecture (D007)                |
| Speed hack                                 | Relay tick authority                              | Same as above                            |
| State saturation                           | Time-budget pool + EWMA scoring + hard caps       | OrderBudget + EwmaTrafficMonitor + relay |
| Eavesdropping                              | AEAD / TLS transport encryption                   | Transport security design                |
| Packet forgery                             | Authenticated encryption (AEAD)                   | Transport security design                |
| Protocol DoS                               | BoundedReader + size caps + rate limits           | Protocol hardening                       |
| Replay tampering                           | Ed25519 signed hash chain                         | Replay system design                     |
| Automation                                 | Dual-model detection (behavioral + statistical)   | Relay-side + post-hoc replay analysis    |
| Result fraud                               | Relay-certified match results                     | Relay architecture                       |
| Seed manipulation                          | Commit-reveal seed protocol                       | Connection establishment (03-NETCODE.md) |
| Version mismatch                           | Protocol handshake                                | Lobby system                             |
| WASM mod abuse                             | Capability-based sandbox                          | Modding architecture (D005)              |
| Desync exploit                             | Server-side only analysis                         | Security by design                       |
| Supply chain attack                        | Anomaly detection + provenance + 2FA + lockfile   | Workshop security (D030)                 |
| Typosquatting                              | Publisher-scoped naming + similarity detection    | Workshop naming (D030)                   |
| Manifest confusion                         | Canonical-inside-package + manifest_hash          | Workshop integrity (D030/D049)           |
| Index poisoning                            | Path-scoped PR validation + signed index          | Git-index security (D049)                |
| Dependency confusion                       | Source-pinned lockfiles + shadow warnings         | Workshop federation (D050)               |
| Version mutation                           | Immutability rule + CI enforcement                | Workshop integrity (D030)                |
| Relay exhaustion                           | Connection limits + per-IP caps + idle timeout    | Relay architecture (D007)                |
| Desync-as-DoS                              | Per-player attribution + strike system            | Desync detection                         |
| Win-trading                                | Diminishing returns + distinct-opponent req       | Ranked integrity (D055)                  |
| Queue dodging                              | Anonymous veto + escalating dodge penalty         | Matchmaking fairness (D055)              |
| Tracking phishing                          | Protocol handshake + trust indicators + HTTPS     | CommunityBridge security                 |
| Cross-community rep                        | Community-scoped display + local-only ratings     | SCR portability (D052)                   |
| Placement carnage                          | Hidden matchmaking rating + min match quality     | Season transition (D055)                 |
| Desperation exploit                        | Reduced info content + min queue population       | Matchmaking fairness (D055)              |
| Relay ranked SPOF                          | Checkpoint hashes + degraded cert + monitoring    | Relay architecture (D007)                |
| Tier config inject                         | Monotonic validation + path sandboxing            | YAML loading defense                     |
| EWMA NaN                                   | Finite guard + reset-to-safe + alpha validation   | Traffic monitor hardening                |
| Reconciler drift                           | Capped ticks_since_sync + defined MAX_DELTA       | Cross-engine security (D011)             |
| Anti-cheat trust                           | Relay ≠ judge + defined thresholds + appeal       | Dual-model integrity (V12)               |
| Behavioral mmk pool                        | Continuous trust score + tiered consequences      | Behavioral matchmaking (V12/D055)        |
| Detection evasion                          | Population baselines + continuous recalibration   | Population-baseline comparison (V12/V54) |
| Enforcement timing                         | Wave ban cadence + intelligence gathering         | Enforcement timing strategy (V12)        |
| Protocol fingerprint                       | Opt-in sources + proxy routing + minimal ident    | CommunityBridge privacy                  |
| Format parser DoS                          | Decompression caps + fuzzing + iteration limits   | `ra-formats` defensive parsing (V38)     |
| Lua sandbox bypass                         | `string.rep` cap + coroutine check + fatal limits | Modding sandbox hardening (V39)          |
| LLM content inject                         | Validation pipeline + cumulative limits + filter  | LLM safety gate (V40)                    |
| Replay resource skip                       | Consent prompt + content-type restriction         | Replay security model (V41)              |
| Save game bomb                             | Decompression cap + schema validation + size cap  | Format safety (V42)                      |
| DNS rebinding/SSRF                         | IP range block + DNS pinning + post-resolve val   | WASM network hardening (V43)             |
| Dev mode exploit                           | Sim-state flag + lobby-only + ranked disabled     | Multiplayer integrity (V44)              |
| Replay frame loss                          | Frame loss counter + `send_timeout` + gap mark    | Replay integrity (V45)                   |
| Path traversal                             | `strict-path` boundary enforcement                | Path security infrastructure             |
| Name impersonation                         | UTS #39 skeleton + mixed-script ban + BiDi strip  | Display name validation (V46)            |
| Key compromise                             | Dual-signed rotation + BIP-39 emergency recovery  | Identity key rotation (V47)              |
| Server impersonation                       | Cert chain + CRL + OCSP-style fast revocation     | Community server auth (V48)              |
| Package forgery                            | Author Ed25519 signing + registry counter-sign    | Workshop package integrity (V49)         |
| Mod cross-probing                          | Namespace isolation + capability-gated IPC        | WASM module isolation (V50)              |
| Supply chain update                        | Popularity quarantine + diff review + rollback    | Workshop package quarantine (V51)        |
| Star-jacking                               | Rate limit + anomaly detection + fork detection   | Workshop reputation defense (V52)        |
| Direct-peer replay forgery (optional mode) | Peer-attested frame hashes + end-match signing    | Direct-peer replay attestation (V53)     |
| False accusations                          | Tiered thresholds + calibration + graduated resp  | Anti-cheat false-positive control (V54)  |
| Bug-as-cheat                               | Desync fingerprint + classification heuristic     | Desync classification (V55)              |
| BiDi text injection                        | Unified sanitization pipeline + context registry  | Text safety (V56)                        |
| ICRP local CSWSH                           | Origin validation + challenge secret + bind local | ICRP WebSocket hardening (V57)           |
| Lobby host manipulation                    | Change notification + ranked whitelist + metadata | Lobby integrity (V58)                    |
| Ranked spectator ghosting                  | 120s minimum delay floor + relay-enforced buffer  | Observer delay enforcement (V59)         |
| Observer RNG prediction                    | Delay + fog-auth + behavioral detection           | Lockstep limitation (V60, acknowledged)  |

**No kernel-level anti-cheat.** Open-source, cross-platform, no ring-0 drivers. We accept that lockstep RTS will always have a maphack risk in client-sim modes — the fog-authoritative server is the real answer for high-stakes play.

**Performance as anti-cheat.** Our tick-time targets (< 10ms on 8-core desktop) mean the relay server can run games at full speed with headroom for behavioral analysis. Stuttery servers with 40ms ticks can't afford real-time order analysis — we can.
