## D050: Workshop as Cross-Project Reusable Library

**Decision:** The Workshop core (registry, distribution, federation, P2P) is designed as a **standalone, engine-agnostic, game-agnostic Rust library** that Iron Curtain is the first consumer of, with the explicit intent that future game projects (XCOM-inspired tactics clone, Civilization-inspired 4X clone, Operation Flashpoint/ArmA-inspired military sim) will be additional consumers. These future projects may or may not use Bevy — the Workshop library must not depend on any specific game engine.

**Rationale:**
- The author plans to build multiple open-source game clones in the spirit of OpenRA, each targeting a different genre's community. Every one of these projects faces the same Workshop problem: mod distribution, versioning, dependencies, integrity, community hosting, P2P delivery
- Building Workshop infrastructure once and reusing it across projects amortizes the significant design and engineering investment over multiple games
- An XCOM clone needs soldier mods, ability packs, map presets, voice packs. A Civ clone needs civilization packs, map scripts, leader art, scenario bundles. An OFP/ArmA clone needs terrains (often 5–20 GB), vehicle models, weapon packs, mission scripts, campaign packages. All of these are "versioned packages with metadata, dependencies, and integrity verification" — the same core abstraction
- The P2P distribution layer is especially valuable for the ArmA-style project where mod sizes routinely exceed what any free CDN can sustain
- Making the library engine-agnostic also produces cleaner IC code — the Bevy integration layer is thinner, better tested, and easier to maintain

### Two-Layer Architecture

The Workshop is split into two layers with a clean boundary:

```
┌─────────────────────────────────────────────────────────┐
│  Game Integration Layer (per-project, engine-specific)  │
│                                                         │
│  IC: Bevy plugin, lobby auto-download, game_module,     │
│       .icpkg extension, `ic mod` CLI, ra-formats,       │
│       Bevy-native format recommendations (D049)         │
│                                                         │
│  XCOM clone: its engine plugin, mission-trigger          │
│       download, .xpkg, its CLI, its format prefs        │
│                                                         │
│  Civ clone: its engine plugin, scenario-load download,  │
│       .cpkg, its CLI, its format prefs                  │
│                                                         │
│  OFP clone: its engine plugin, server-join download,    │
│       .opkg, its CLI, its format prefs                  │
├─────────────────────────────────────────────────────────┤
│  Workshop Core Library (engine-agnostic, game-agnostic) │
│                                                         │
│  Registry: search, publish, version, depend, license    │
│  Distribution: BitTorrent/WebTorrent, HTTP fallback     │
│  Federation: multi-source, git-index, remote, local     │
│  Integrity: SHA-256, piece hashing, signed manifests    │
│  Identity: publisher/name@version                       │
│  P2P engine: peer scoring, piece selection, bandwidth   │
│  CLI core: auth, publish, install, update, resolve      │
│  Protocol: federation spec, manifest schema, APIs       │
└─────────────────────────────────────────────────────────┘
```

### Core Library Boundary — What's In and What's Out

| Concern                    | Core Library (game-agnostic)                                                                                                                 | Game Integration Layer (per-project)                                                                                                                                                                                 |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Package format**         | ZIP archive with `manifest.yaml`. Extension is configurable (default: `.pkg`)                                                                | IC uses `.icpkg`, other projects choose their own                                                                                                                                                                    |
| **Manifest schema**        | Core fields: `name`, `version`, `publisher`, `description`, `license`, `dependencies`, `platforms`, `sha256`, `tags`                         | Extension fields: `game_module`, `engine_version`, `category` (IC-specific). Each project defines its own extension fields                                                                                           |
| **Resource categories**    | Tags (free-form strings). Core provides no fixed category enum                                                                               | Each project defines a recommended tag vocabulary (IC: `sprites`, `music`, `map`; XCOM: `soldiers`, `abilities`, `missions`; Civ: `civilizations`, `leaders`, `scenarios`; OFP: `terrains`, `vehicles`, `campaigns`) |
| **Package identity**       | `publisher/name@version` — already game-agnostic                                                                                             | No change needed                                                                                                                                                                                                     |
| **Dependency resolution**  | semver resolution, lockfile, integrity verification                                                                                          | Per-project compatibility checks (e.g., IC checks `game_module` + `engine_version`)                                                                                                                                  |
| **P2P distribution**       | BitTorrent/WebTorrent protocol, tracker, peer scoring, piece selection, bandwidth limiting, HTTP fallback                                    | Per-project seed infrastructure (IC uses `ironcurtain.gg` tracker, OFP clone uses its own)                                                                                                                           |
| **P2P peer scoring**       | Weighted multi-dimensional: `Capacity × w1 + Locality × w2 + SeedStatus × w3 + ApplicationContext × w4`. Weights and dimensions configurable | Each project defines `ApplicationContext`: IC = same-lobby bonus, OFP = same-server bonus, Civ = same-matchmaking-pool bonus. Projects that have no context concept set weight to 0                                  |
| **Download priority**      | Three tiers: `critical` (blocking gameplay), `requested` (user-initiated), `background` (cache warming)                                      | Each project maps its triggers: IC's lobby-join → `critical`. OFP's server-join → `critical`. Civ's scenario-load → `requested`                                                                                      |
| **Auto-download trigger**  | Library provides `download_packages(list, priority)` API                                                                                     | Integration layer decides WHEN to call it: IC calls on lobby join, OFP calls on server connect, XCOM calls on mod browser click                                                                                      |
| **CLI operations**         | Core operations: `auth`, `publish`, `install`, `update`, `search`, `resolve`, `lock`, `audit`, `export-bundle`, `import-bundle`              | Each project wraps as its own CLI: `ic mod *`, `xcom mod *`, etc.                                                                                                                                                    |
| **Format recommendations** | None. The core library is format-agnostic — it distributes opaque files                                                                      | Each project recommends formats for its engine: IC recommends Bevy-native (D049). A Godot-based project recommends Godot-native formats. A custom-engine project recommends whatever it loads                        |
| **Federation**             | Multi-source registry, `sources.yaml`, git-index support, remote server API, local repository                                                | Per-project default sources: IC uses `ironcurtain.gg` + `iron-curtain/workshop-index`. Each project configures its own                                                                                               |
| **Config paths**           | Library accepts a config root path                                                                                                           | Each project sets its own: IC uses `~/.ic/`, XCOM clone uses `~/.xcom/`, etc.                                                                                                                                        |
| **Auth tokens**            | Token generation, storage, scoping (publish/admin/readonly), environment variable override                                                   | Per-project env var names: `IC_AUTH_TOKEN`, `XCOM_AUTH_TOKEN`, etc.                                                                                                                                                  |
| **Lockfile**               | Core lockfile format with package hashes                                                                                                     | Per-project lockfile name: `ic.lock`, `xcom.lock`, etc.                                                                                                                                                              |

### Impact on Existing D030/D049 Design

The existing Workshop design requires only **architectural clarification**, not redesign. The core abstractions (packages, manifests, publishers, dependencies, federation, P2P) are already game-agnostic in concept. The changes are:

1. **Naming**: Where the design says `.icpkg`, the implementation will have a configurable extension with `.icpkg` as IC's default. Where it says `ic mod *`, the core library provides operations and IC wraps them as `ic mod *` subcommands.

2. **Categories**: Where D030 lists a fixed `ResourceCategory` enum (Music, Sprites, Maps...), the core library uses free-form tags. IC's integration layer provides a recommended tag vocabulary and UI groupings. Other projects provide their own.

3. **Manifest**: The `manifest.yaml` schema splits into core fields (in the library) and extension fields (per-project). `game_module: ra1` is an IC extension field, not a core manifest requirement.

4. **Format recommendations**: D049's Bevy-native format table is IC-specific guidance, not a core Workshop concern. The core library is format-agnostic. Each consuming project publishes its own format recommendations based on its engine's capabilities.

5. **P2P scoring**: The `LobbyContext` dimension in peer scoring becomes `ApplicationContext` — a generic callback where any project can inject context-aware peer prioritization. IC implements it as "same lobby = bonus." An ArmA-style project implements it as "same server = bonus."

6. **Infrastructure**: Domain names (`ironcurtain.gg`), GitHub org (`iron-curtain/`), tracker URLs — these are IC deployment configuration. The core library is configured via `sources.yaml` with no hardcoded URLs.

### Cross-Project Infrastructure Sharing

While each project has its own Workshop deployment, sharing is possible:

- **Shared tracker**: A single BitTorrent tracker can serve multiple game projects. The info-hash namespace is naturally disjoint (different packages = different hashes).
- **Shared git-index hosting**: One GitHub org could host workshop-index repos for multiple projects.
- **Shared seed boxes**: Seed infrastructure can serve packages from multiple games simultaneously — BitTorrent doesn't care about content semantics.
- **Cross-project dependencies**: A music pack or shader effect could be published once and depended on by packages from multiple games. The identity system (`publisher/name@version`) is globally unique.
- **Shared federation network**: Community-hosted Workshop servers could participate in multiple games' federation networks simultaneously.

> **Also shared with IC's netcode infrastructure.** The tracking server, relay server, and Workshop server share deep structural parallels within IC itself — federation, heartbeats, rate control, connection management, observability, deployment principles. The cross-pollination analysis (`research/p2p-federated-registry-analysis.md` § "Netcode ↔ Workshop Cross-Pollination") identifies four shared infrastructure opportunities: a unified `ic-server` binary (tracking + relay + workshop in one process for small community operators), a shared federation library (multi-source aggregation used by both tracking and Workshop), a shared auth/identity layer (one Ed25519 keypair for multiplayer + publishing + profile), and shared scoring infrastructure (EWMA time-decaying reputation used by both P2P peer scoring and relay player quality tracking). The federation library and scoring infrastructure belong in the Workshop core library (D050) since they're already game-agnostic.

### Engine-Agnostic P2P and Netcode

The P2P distribution protocol (BitTorrent/WebTorrent) and all the patterns adopted from Kraken, Dragonfly, and IPFS (see D049 competitive landscape and `research/p2p-federated-registry-analysis.md`) are **already engine-agnostic**. The protocol operates at the TCP/UDP level — it doesn't know or care whether the consuming application uses Bevy, Godot, Unreal, or a custom engine. The Rust implementation (`ic-workshop` core library) has no engine dependency.

For projects that use a non-Rust engine (unlikely given the author's preferences, but architecturally supported): the Workshop core library exposes a C FFI or can be compiled as a standalone process that the game communicates with via IPC/localhost HTTP. The CLI itself serves as a non-Rust integration path — any game engine can shell out to the Workshop CLI for install/update operations.

### Non-RTS Game Considerations

Each future genre introduces patterns the current design doesn't explicitly address:

| Genre                         | Key Workshop Differences                                                                                            | Already Handled                                                               | Needs Attention                                                                                                                                                                                                                                   |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Turn-based tactics** (XCOM) | Smaller mod sizes, more code-heavy mods (abilities, AI), procedural map parameters                                  | Package format, dependencies, P2P                                             | Ability/behavior mods may need a scripting sandbox equivalent to IC's Lua/WASM — but that's a game concern, not a Workshop concern                                                                                                                |
| **Turn-based 4X** (Civ)       | Very large mod variety (civilizations, maps, scenarios, art), DLC-like mod structure, long-lived save compatibility | Package format, dependencies, versioning, P2P                                 | Save-game compatibility metadata (a Civ mod that changes game rules may break existing saves). Workshop manifest could include `breaks_saves: true` as an extension field                                                                         |
| **Military sim** (OFP/ArmA)   | Very large packages (terrains 5–20 GB), server-mandated mod lists, many simultaneous mods active                    | P2P (critical for large packages), dependencies, auto-download on server join | Partial downloads (download terrain mesh now, HD textures later) could benefit from sub-package granularity. Workshop packages already support dependencies — a terrain could be split into `base` + `hd-textures` + `satellite-imagery` packages |
| **Any**                       | Different scripting languages, different asset formats, different mod structures                                    | Core library is content-agnostic                                              | Nothing — this is the point of the two-layer design                                                                                                                                                                                               |

### Phase

D050 is an architectural principle, not a deliverable with its own phase. It shapes HOW D030 and D049 are implemented:

- **IC Phase 3–4**: Implement Workshop core as a separate Rust library crate within the IC monorepo. The crate has zero Bevy dependencies. IC's Bevy plugin wraps the core library. The API boundary enforces the two-layer split from the start.
- **IC Phase 5–6**: If a second game project begins, the core library can be extracted to its own repo with minimal effort because the boundary was enforced from day one.
- **Post-IC-launch**: Each new game project creates its own integration layer and deployment configuration. The core library, P2P protocol, and federation specification are shared.

---

| ID   | Topic                                                                                                                               | Needs Resolution By |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| P001 | ~~ECS crate choice~~ — RESOLVED: Bevy's built-in ECS                                                                                | Resolved            |
| P002 | Fixed-point scale (256? 1024? match OpenRA's 1024?)                                                                                 | Phase 2 start       |
| P003 | Audio library choice + music integration design (see note below)                                                                    | Phase 3 start       |
| P004 | Lobby/matchmaking protocol specifics — PARTIALLY RESOLVED: architecture + lobby protocol defined (D052), wire format details remain | Phase 5 start       |
| P005 | ~~Map editor architecture~~ — RESOLVED: Scenario editor in SDK (D038+D040)                                                          | Resolved            |
| P006 | ~~License choice~~ — RESOLVED: GPL v3 with modding exception (D051)                                                                 | Resolved            |
| P007 | ~~Workshop: single source vs multi-source~~ — RESOLVED: Federated multi-source (D030)                                               | Resolved            |

### P003 — Audio System Design Notes

The audio system is the least-designed critical subsystem. Beyond the library choice, Phase 3 needs to resolve:

- **Original `.aud` playback and encoding:** Decoding and encoding Westwood's `.aud` format (IMA ADPCM, mono/stereo, 8/16-bit, varying sample rates). Full codec implementation based on EA GPL source — `AUDHeaderType` header, `IndexTable`/`DiffTable` lookup tables, 4-bit nibble processing. See `05-FORMATS.md` § AUD Audio Format for complete struct definitions and algorithm details. Encoding support enables the Asset Studio (D040) audio converter for .aud ↔ .wav/.ogg conversion
- **Music loading from Remastered Collection:** If the player owns the Remastered Collection, can IC load the remastered soundtrack? Licensing allows personal use of purchased files, but the integration path needs design
- **Dynamic music states:** Combat/build/idle transitions (original RA had this — "Act on Instinct" during combat, ambient during base building). State machine driven by sim events
- **Music as Workshop resources:** Swappable soundtrack packs via D030 — architecture supports this, but audio pipeline needs to be resource-pack-aware
- **Frank Klepacki's music is integral to C&C identity.** The audio system should treat music as a first-class system, not an afterthought. See `13-PHILOSOPHY.md` § "Audio Drives Tempo"

### P006 — RESOLVED: See D051
