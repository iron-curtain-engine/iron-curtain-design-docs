# Tier 3: WASM Modules

### Rationale

- Near-native performance for complex mods
- Perfectly sandboxed by design (WASM's memory model)
- Deterministic execution (critical for multiplayer)
- Modders write in Rust, C, Go, AssemblyScript, or even Python compiled to WASM
- `wasmtime` or `wasmer` crates

### Browser Build Limitation (WASM-on-WASM)

When IC is compiled to WASM for the browser target (Phase 7), Tier 3 WASM mods present a fundamental problem: `wasmtime` does not compile to `wasm32-unknown-unknown`. The game itself is running as WASM in the browser — it cannot embed a full WASM runtime to run mod WASM modules inside itself.

**Implications:**
- **Browser builds support Tier 1 (YAML) and Tier 2 (Lua) mods only.** Lua via `mlua` compiles to WASM and executes as interpreted bytecode within the browser build. YAML is pure data.
- **Tier 3 WASM mods are desktop/server-only** (native builds where `wasmtime` runs normally).
- **Multiplayer between browser and desktop clients** is not affected by this limitation *for the base game* — the sim, networking, and all built-in systems are native Rust compiled to WASM.

**Lobby enforcement — gameplay/load-required vs. presentation split:** The browser limitation only blocks lobbies that *require* a Tier 3 mod for gameplay or asset loading. Consistent with D068's gameplay/presentation fingerprint split:

- **Gameplay/load-required Tier 3 mods** (pathfinder, AI strategy — run during sim ticks; format loading — run at asset load time): part of the lobby gameplay fingerprint. If the lobby requires one, browser clients **cannot join** (they can't run the WASM binary, and the game would either desync or fail to load assets). Platform restriction badge shown. Note: format loaders don't affect sim ticks, but they are still required for the game to load correctly — a missing format loader means assets can't be decoded.
- **Presentation-only Tier 3 mods** (render overlays, UI mods): per-player optional, not part of the gameplay fingerprint (§ Multiplayer Capability Rules). Browser clients **can join** these lobbies — they simply don't run the presentation mod. Desktop players see the overlay; browser players don't. No desync because presentation mods never affect the sim or asset loading.

**Future mitigation:** A WASM interpreter written in pure Rust (e.g., `wasmi`) can itself compile to `wasm32-unknown-unknown`, enabling Tier 3 mods in the browser at reduced performance (~10-50x slower than native `wasmtime`). This is acceptable for lightweight WASM mods (AI strategies, format loaders) but likely too slow for complex pathfinder or render mods. When/if this becomes viable, the engine would use `wasmtime` on native builds and `wasmi` on browser builds — same mod binary, different execution speed. This is a Phase 7+ concern.

### WASM Host API (Security Boundary)

```rust
// The WASM host functions are the ONLY API mods can call.
// The API surface IS the security boundary.

#[wasm_host_fn]
// Note: WASM ABI passes entity IDs as opaque u32 handles.
// The host unwraps to UnitTag and validates. See type-safety.md
// § WASM ABI Boundary Policy for the two-layer convention.
fn get_unit_position(unit_handle: u32) -> Option<(i32, i32)> {
    let tag = UnitTag::from_wasm_handle(unit_handle)?;
    let unit = sim.resolve_unit(tag)?;
    // CHECK: is this unit visible to the mod's player?
    if !sim.is_visible_to(mod_player, unit.position) {
        return None;  // Mod cannot see fogged units
    }
    Some(unit.position)
}

// There is no get_all_units() function.
// There is no get_enemy_state() function.
```

### Mod Capabilities System

```rust
pub struct ModCapabilities {
    // --- Standard capabilities (sandbox-internal, no I/O) ---
    pub read_own_state: bool,
    pub read_visible_state: bool,     // Fog-filtered visible unit/building queries (ic_query_* API)
    // Can NEVER read fogged state (API doesn't exist)
    pub issue_orders: bool,           // For AI mods
    pub render: bool,                 // For render mods (ic_render_* API)
    pub pathfinding: bool,            // For pathfinder mods (ic_pathfind_* API)
    pub ai_strategy: bool,            // For AI mods (ic_ai_* API + AiStrategy trait)
    pub format_loading: bool,         // For format loader mods (ic_format_* API) — runs at asset load time, not during sim ticks

    // --- Elevated capabilities (extend beyond sandbox, require player review) ---
    // INVARIANT: filesystem and network are FORBIDDEN for sim-tick mods
    // (pathfinding, ai_strategy). ic-sim is pure and no-I/O
    // (crate-graph.md § ic-sim). These capabilities are available ONLY to
    // mods that never execute during sim ticks: presentation-layer mods
    // (render, UI overlay) and format loaders (which run at asset load
    // time, not during deterministic simulation).
    pub filesystem: FileAccess,       // Presentation + format loaders only. None for sim-tick mods
    pub network: NetworkAccess,       // Presentation mods only. None for sim-tick or format mods
}

pub enum FileAccess {
    None,                          // Most mods (and ALL sim-tick mods)
    ReadOnly(Vec<String>),         // Scoped read access to specific paths (format loaders, presentation mods)
    // NEVER write access. NEVER unrestricted path access.
    // Paths are relative to the mod's data directory or the game's asset directory.
    // Absolute paths and path traversal (../) are rejected at load time.
}

pub enum NetworkAccess {
    None,                          // Most mods (and ALL sim-tick mods, ALL format loaders)
    AllowList(Vec<String>),        // Presentation/UI mods fetching assets from specific domains
    // NEVER unrestricted
}
```

**Sim-tick capability exclusion rule:** WASM mods that declare `pathfinding` or `ai_strategy` capabilities **cannot** also declare `network` or `filesystem`. The Workshop rejects manifests that combine sim-tick and elevated capabilities. At load time, the runtime validates this exclusion — a mod binary requesting both is not loaded. This preserves `ic-sim`'s no-I/O invariant: code running during deterministic sim ticks never touches the network or filesystem.

**Format loaders** (`format_loading`) are a special case: they run at **asset load time** (during `Loading` state), not during sim ticks. By default, format loaders access file data through the engine-mediated `ic_format_read_bytes()` host function, which reads through the engine's archive abstraction (`.mix` files, mounted directories) — no elevated `filesystem` capability needed. The elevated `filesystem` capability is only required if a format loader needs to read files **outside** the engine's archive system (e.g., reading companion metadata files from the mod's own data directory). Format loaders **cannot** declare `network` (asset loading must be local/deterministic — network-fetched assets would create load-order non-determinism). The Workshop validates this: `format_loading + filesystem` is allowed; `format_loading + network` is rejected.

> **Security (V43):** Domain-based `AllowList` is vulnerable to DNS rebinding — an approved domain can be re-pointed to `127.0.0.1` or `192.168.x.x` after capability review. Mitigations: block RFC 1918/loopback/link-local IP ranges after DNS resolution, pin DNS at mod load time, validate resolved IP on every request. See `06-SECURITY.md` § Vulnerability 43.

#### Network Host API (Presentation Mods Only)

WASM mods with `NetworkAccess::AllowList` access the network **exclusively** through these host-provided functions. No WASI networking capabilities are granted — raw sockets, DNS resolution, and TCP/UDP are never available to WASM modules (`wasmtime::Config` explicitly denies all WASI networking proposals — `06-SECURITY.md` § F10).

```rust
// === Network Host API (ic_http_* namespace) ===
// Available only to mods with ModCapabilities.network = AllowList(...)
// Domain is validated against the AllowList BEFORE the request is made.
// Resolved IP is checked against blocked ranges (RFC 1918, loopback, link-local).

/// HTTP GET request. Domain must be in the mod's AllowList.
/// Returns response body as bytes, or Err if domain denied / request failed.
/// Runs asynchronously — the mod yields until the response arrives.
#[wasm_host_fn] fn ic_http_get(url: &str) -> Result<Vec<u8>, HttpError>;

/// HTTP POST request. Domain must be in the mod's AllowList.
/// Body is limited to 1 MB. Response limited to 4 MB.
#[wasm_host_fn] fn ic_http_post(url: &str, body: &[u8], content_type: &str) -> Result<Vec<u8>, HttpError>;

pub enum HttpError {
    DomainDenied,       // URL domain not in AllowList
    IpBlocked,          // Resolved IP is in blocked range (RFC 1918, loopback, etc.)
    Timeout,            // Request exceeded 10-second timeout
    NetworkError,       // Connection failed
    ResponseTooLarge,   // Response exceeded 4 MB limit
}
```

#### Filesystem Host API (Presentation + Format Loader Mods Only)

WASM mods with `FileAccess::ReadOnly` access files through these host-provided functions. No WASI filesystem capabilities are granted — `std::fs`, directory listing, and write access are never available.

```rust
// === Filesystem Host API (ic_fs_* namespace) ===
// Available only to mods with ModCapabilities.filesystem = ReadOnly(...)
// Path is validated against the mod's declared path scope BEFORE the read.
// Absolute paths and path traversal (../) are rejected.

/// Read a file's contents. Path must be within the mod's declared scope.
#[wasm_host_fn] fn ic_fs_read(path: &str) -> Result<Vec<u8>, FsError>;

/// Check if a file exists within the mod's declared scope.
#[wasm_host_fn] fn ic_fs_exists(path: &str) -> Result<bool, FsError>;

pub enum FsError {
    PathDenied,         // Path outside declared scope or contains traversal
    NotFound,           // File does not exist
    ReadError,          // I/O error
}
```

### Install-Time Capability Review & Player Control

The capability model above defines *what* a mod can access. This section defines *how the player sees and controls those capabilities*.

> **D074 policy revision:** D074-federated-moderation.md § Layer 3 states IC does not prompt players with capability permission dialogs because "mods cannot access files regardless of what the player clicks." This is true for Tier 1 (YAML) and Tier 2 (Lua) mods, which operate within fixed sandboxes. However, Tier 3 WASM mods **can** request `network` and `filesystem` capabilities that extend beyond the base sandbox — the `ModCapabilities` struct explicitly includes `filesystem: FileAccess` and `network: NetworkAccess` fields. For these elevated capabilities, player awareness is warranted.
>
> This section revises D074 Layer 3 specifically for **Tier 3 WASM mods that request network or filesystem access, or above-default resource limits**. The principle remains: the sandbox enforces limits regardless of player choice. But the player should know *which* elevated capabilities a WASM mod declares, and should be able to deny optional ones. This is a targeted extension, not a general permission dialog — Tier 1/2 mods are unaffected, and WASM mods that request only standard capabilities (`read_own_state`, `read_visible_state`, `issue_orders`, `render`, `pathfinding`, `ai_strategy`, `format_loading`) with default-or-below resource limits do not trigger a review prompt.
>
> **D074 Layer 4 is preserved:** Capability information on the Workshop listing page remains *informational* for standard capabilities. The review prompt triggers for network/filesystem capabilities (that reach outside the game sandbox) or above-default resource limits.

#### Modder Side — Declaring Capabilities

WASM mod authors declare capabilities in `mod.toml`. Capabilities that extend beyond the sandbox (network, filesystem) include a mandatory `reason` field:

```toml
# mod.toml — capability declaration
[mod]
id = "tactical-overlay"
title = "Tactical Overlay"           # canonical field is 'title', not 'name' (mod-sdk.md § mod.toml)
type = "render"
wasm_module = "tactical_overlay.wasm"

[capabilities]
render = true
read_visible_state = true
network = { essential = false, domains = ["api.example.com"], reason = "Fetches community overlay presets (optional — works offline with built-in presets)" }
filesystem = { essential = false, access = "read", paths = ["overlay-presets/"], reason = "Loads user-saved overlay configurations (optional)" }
# essential = true  → mod cannot function without this capability; denying it disables the mod
# essential = false → mod works with reduced functionality if denied (default)

[capabilities.limits]
fuel_per_tick = 500_000
max_memory_bytes = 8_388_608
```

The `reason` field is **mandatory for network and filesystem capabilities** — Workshop submission rejects manifests with missing reasons for these capabilities. Standard capabilities (`read_own_state`, `read_visible_state`, `issue_orders`, `render`, `pathfinding`, `ai_strategy`, `format_loading`) do not require reason fields — they are within the sandbox boundary.

#### Player Side — Capability Review

**When a review prompt is shown:** When a WASM mod declares `network` or `filesystem` capabilities, OR when it requests execution limits above defaults (§ WASM Execution Resource Limits). WASM mods with only standard capabilities AND default-or-below limits install silently (consistent with D074 Layer 3). A pure AI/pathfinder mod with no elevated capabilities but higher fuel/memory limits will trigger the review screen showing the resource usage section only.

```
┌──────────────────────────────────────────────────────────┐
│  Install: Tactical Overlay v1.2.0                         │
│  by MapMaster ✓  |  Tier 3 (WASM)  |  2.1 MB             │
│                                                           │
│  This mod requests elevated capabilities:                 │
│                                                           │
│  ┌────────────────────────────────────────────────────┐   │
│  │ 🔲 Network access                       [Toggle]  │   │
│  │    api.example.com                                 │   │
│  │    Fetches community overlay presets                │   │
│  │    (works offline with built-in presets)            │   │
│  │                                                    │   │
│  │ 🔲 File read access                     [Toggle]  │   │
│  │    overlay-presets/                                 │   │
│  │    Loads user-saved overlay configurations          │   │
│  └────────────────────────────────────────────────────┘   │
│                                                           │
│  Standard capabilities (render, read visible units):      │
│  granted automatically by the sandbox.                    │
│                                                           │
│  [Install]    [Cancel]                                    │
└──────────────────────────────────────────────────────────┘
```

**UX rules:**

| Rule | Behavior |
|------|----------|
| **Standard capabilities** (`read_own_state`, `read_visible_state`, `issue_orders`, `render`, `pathfinding`, `ai_strategy`, `format_loading`) | Granted automatically. No prompt. Listed as informational summary only |
| **Elevated capabilities** (network, filesystem) | Shown with toggle switches. Default: off. Player opts in per capability |
| **Reason text** | Mandatory for elevated capabilities, shown under each toggle |
| **Network domains** | Each allowed domain listed explicitly |
| **Filesystem paths** | Scoped paths listed explicitly |
| **No review needed** | WASM mods without network/filesystem AND with default-or-below limits install silently. Tier 1/2 mods always install silently |
| **Updates** | Re-prompt if elevated capabilities changed ("New: network access to api.example.com") OR if above-default resource limits changed ("CPU increased from 1M to 2M instructions/tick"). Keyed by `capability_manifest_hash` — unchanged manifests across version bumps skip re-review |

#### Managing Capabilities After Install

Elevated capabilities (network, filesystem) are adjustable via the mod management UI. Changes take effect at **next match start** — never mid-match.

> **Why not immediate:** D066 establishes install/update-time review as the safe pattern for extension permissions. Elevated capabilities are only available to presentation-layer mods and format loaders (never sim-tick mods — see § Sim-tick capability exclusion rule), but a uniform "next match" rule avoids edge cases and is simpler to implement.

Revoking an elevated capability that the mod declared as `essential = true` disables the mod entirely (with confirmation: "This mod requires [network access] to function. Disable the mod?"). Revoking an `essential = false` capability keeps the mod active with reduced functionality — the mod's host function calls for that capability return `Err(CapabilityDenied)`.

#### Capability Storage

Granted capabilities are stored in local SQLite (D034), keyed by mod ID + capability manifest hash:

```rust
/// Per-mod granted capabilities. Stored in local SQLite (D034).
pub struct GrantedCapabilities {
    /// Mod identity — uses mod.id from mod.toml (not WorkshopPackageId,
    /// since the same flow covers Workshop and local file installs).
    pub mod_id: ModId,
    /// Hash of the mod's capability declaration section.
    /// When capabilities change across versions, the hash changes and
    /// triggers re-review. When capabilities are unchanged, the hash
    /// stays the same — no re-prompt even on version bumps.
    pub capability_manifest_hash: Sha256Digest,
    /// Which elevated capabilities the player granted.
    pub granted_elevated: ElevatedCapabilities,
    pub granted_at: i64,
    pub last_reviewed_at: i64,
}

pub struct ElevatedCapabilities {
    pub network: Option<NetworkAccess>,   // None = denied, Some = granted with domain list
    pub filesystem: Option<FileAccess>,   // None = denied, Some = granted with path scope
}
```

#### Workshop Publication Flow

1. `ic mod publish` reads `mod.toml` capabilities and includes them in the package manifest
2. Workshop validates: every `network` and `filesystem` capability has a non-empty `reason` field
3. **Capability change detection:** If a mod update adds new elevated capabilities, the Workshop flags the update for moderation review (V25). Players with the mod installed see: "Update requests new capability: network access. Review before updating."
4. The Workshop listing shows the capability summary on the mod page (informational — consistent with D074 Layer 4)

#### Multiplayer Capability Rules

**Sim-tick mods** (pathfinder, AI strategy) **cannot declare elevated capabilities** (network, filesystem) — see § Sim-tick capability exclusion rule. Their capability sets are always identical across clients (standard capabilities are granted automatically). **Format loaders** may declare `filesystem` but not `network`; since format loading runs at asset load time (not sim ticks), filesystem access doesn't affect deterministic sim state — but the granted filesystem capability must still match across clients in the lobby fingerprint (D062) because it affects which assets load successfully. The lobby fingerprint includes the mod's execution limit grants, which must also match across clients for all sim-affecting and format-loading mods.

**Presentation-only mods** (render, UI overlay) allow per-player capability differences. These mods never affect the deterministic sim — Player A can have network-enabled overlays while Player B does not. No fingerprint impact.

> **Settings IA note:** The current settings panel (`settings.md`) does not have a "Mods" tab. Mod capability management should be accessible from the Workshop → Installed panel or a new Mods section in Settings. This is a `settings.md` / `player-flow/workshop.md` update deferred until this section moves from design to implementation.

### WASM Execution Resource Limits

Capability-based API controls *what* a mod can do. Execution resource limits control *how much*. Without them, a mod could consume unbounded CPU or spawn unbounded entities — degrading performance for all players and potentially overwhelming the network layer (Bryant & Saiedian 2021 documented this in Risk of Rain 2: "procedurally generated effects combined to produce unintended chain-reactive behavior which may ultimately overwhelm the ability for game clients to render objects or handle sending/receiving of game update messages").

```rust
/// Per-tick execution budget enforced by the WASM runtime (wasmtime fuel metering).
/// Exceeding any limit terminates the mod's tick callback early — the sim continues
/// without the mod's remaining contributions for that tick.
pub struct WasmExecutionLimits {
    pub fuel_per_tick: u64,              // wasmtime fuel units (~1 per wasm instruction)
    pub pathfinder_fuel_per_tick: u64,   // aggregate budget across ALL path requests in one tick (not per-request)
    pub max_memory_bytes: usize,         // WASM linear memory cap (default: 16 MB)
    pub max_entity_spawns_per_tick: u32, // Prevents chain-reactive entity explosions (default: 32)
    pub max_orders_per_tick: u32,        // AI mods can't flood the order pipeline (default: 64)
    pub max_host_calls_per_tick: u32,    // Bounds API call volume (default: 1024)
}

impl Default for WasmExecutionLimits {
    fn default() -> Self {
        Self {
            fuel_per_tick: 1_000_000,       // ~1M instructions — generous for most mods
            pathfinder_fuel_per_tick: 5_000_000, // aggregate per-tick budget (5M — 5× standard, reflects many path requests per tick)
            max_memory_bytes: 16 * 1024 * 1024,  // 16 MB
            max_entity_spawns_per_tick: 32,
            max_orders_per_tick: 64,
            max_host_calls_per_tick: 1024,
        }
    }
}
```

**Why this matters for multiplayer:** In deterministic lockstep, all clients run the same mods. A mod that consumes excessive CPU causes tick overruns on slower machines, triggering adaptive run-ahead increases for everyone. A mod that spawns hundreds of entities per tick inflates state size and network traffic. The execution limits prevent a single mod from degrading the experience — and because the limits are deterministic (same fuel budget, same cutoff point), all clients agree on when a mod is throttled.

**Mod authors can request higher limits** via their mod manifest's `[capabilities.limits]` section. Resource limit requests above defaults are handled via the same review surface as elevated capabilities:

- **Install-time:** If a mod requests limits above defaults, the capability review screen (§ Install-Time Capability Review) shows a "Resource usage" section with the requested values and how they compare to defaults (e.g., "CPU: 2M instructions/tick — 2× default"). The player can accept or cancel the install.
- **Lobby:** Sim-affecting mods with above-default limits require all players to accept the same limits (deterministic execution requires identical fuel budgets). The lobby fingerprint (D062) includes the granted limit values.
- **Storage:** Granted limits are stored alongside `GrantedCapabilities` in local SQLite (D034), keyed by `ModId + capability_manifest_hash` — same key as elevated capability grants.
- **Ranked/tournament:** Stricter defaults enforced. Mods requesting above-default limits are rejected in ranked queues unless whitelisted by the competitive committee (D037).

### WASM Float Determinism

WASM's IEEE 754 float arithmetic (`+`, `-`, `*`, `/`, `sqrt`) is bit-exact per spec. The determinism risks are **NaN bit patterns** (different hosts may produce different NaN payloads) and **FMA fusion** (a JIT may fuse `f32.mul` + `f32.add` into a single FMA instruction on hardware that supports it, changing the rounding result).

**Engine-level enforcement (sim-context WASM only):**

1. **NaN canonicalization:** `Config::cranelift_nan_canonicalization(true)` — all NaN outputs are rewritten to the canonical bit pattern, ensuring identical results across x86_64, aarch64, and any future target.
2. **FMA prevention:** `Config::cranelift_enable_nan_canonicalization` implicitly prevents FMA fusion because the inserted canonicalization barriers between float operations block instruction combining. This is a property of Cranelift's current compilation pipeline — if a future Cranelift version changes this behavior, IC will pin the Cranelift version or add an explicit FMA-disable flag. The `ic mod test --determinism` cross-platform gate (see below) catches any regression.
3. **Render/audio-only WASM** does not run in sim context and has neither enforcement applied — floats are permitted freely.

**Practical guidance for mod authors:** Sim-affecting WASM mods should use integer/fixed-point arithmetic (the `fixed-game-math` crate from D076 is available as a WASM dependency). Float operations are permitted but the mod must pass `ic mod test --determinism` on the CI cross-platform matrix (x86_64 + aarch64) before Workshop publication. This test runs the mod's registered test scenarios on both platforms and compares sim hashes tick-by-tick — any divergence fails the check.

### WASM Rendering API Surface

Tier 3 WASM mods that replace the visual presentation (e.g., a 3D render mod) need a well-defined rendering API surface. These are the WASM host functions exposed for render mods — they are the *only* way a WASM mod can draw to the screen.

```rust
// === Render Host API (ic_render_* namespace) ===
// Available only to mods with ModCapabilities.render = true
// Render APIs use u32 for resource handles (sprite_id, mesh_handle, etc.)
// These are render-side opaque handles, not sim domain IDs — newtypes are
// not required here (type-safety.md § WASM ABI Boundary Policy applies
// to sim-affecting domain IDs only). f32 is permitted for presentation.

/// Register a custom Renderable implementation for an actor type.
#[wasm_host_fn] fn ic_render_register(actor_type: &str, renderable_id: u32);

/// Draw a sprite at a world position (default renderer).
#[wasm_host_fn] fn ic_render_draw_sprite(
    sprite_id: u32, frame: u32, position: WorldPos, facing: u8, palette: u32
);

/// Draw a 3D mesh at a world position (Bevy 3D pipeline).
#[wasm_host_fn] fn ic_render_draw_mesh(
    mesh_handle: u32, position: WorldPos, rotation: [i32; 4], scale: [i32; 3]
);

/// Draw a line (debug overlays, targeting lines).
#[wasm_host_fn] fn ic_render_draw_line(
    start: WorldPos, end: WorldPos, color: u32, width: f32
);

/// Play a skeletal animation on a mesh entity.
#[wasm_host_fn] fn ic_render_play_animation(
    mesh_handle: u32, animation_name: &str, speed: f32, looping: bool
);

/// Set camera position and mode.
#[wasm_host_fn] fn ic_render_set_camera(
    position: WorldPos, mode: CameraMode, fov: Option<f32>
);

/// Screen-to-world conversion (for input mapping).
#[wasm_host_fn] fn ic_render_screen_to_world(
    screen_x: f32, screen_y: f32
) -> Option<WorldPos>;

/// Load an asset (sprite sheet, mesh, texture) by path.
/// Returns a handle ID for use in draw calls.
#[wasm_host_fn] fn ic_render_load_asset(path: &str) -> Option<u32>;

/// Spawn a particle effect at a position.
#[wasm_host_fn] fn ic_render_spawn_particles(
    effect_id: u32, position: WorldPos, duration: u32
);

pub enum CameraMode {
    Isometric,          // fixed angle, zoom via OrthographicProjection.scale
    FreeLook,           // full 3D rotation, zoom via camera distance
    Orbital { target: WorldPos },  // orbit a point, zoom via distance
}
// Zoom behavior is controlled by the GameCamera resource (02-ARCHITECTURE.md § Camera).
// WASM render mods that provide a custom ScreenToWorld impl interpret the zoom value
// appropriately for their camera type (orthographic scale vs. dolly distance vs. FOV).

// === Render-Side State Query API (ic_query_* namespace) ===
// Available only to mods with ModCapabilities.read_visible_state = true
// AND ModCapabilities.render = true.
// These are read-only, fog-filtered queries for rendering purposes —
// the same visibility rules as the AI query API, but returning
// presentation-friendly data (positions, health bars, unit types).
// This API runs OUTSIDE the sim context (render thread, not sim thread)
// and reads from the post-tick render snapshot, not live sim state.

/// Get all visible units for the local player (fog-filtered).
/// Returns presentation data (position, type, health fraction, facing).
#[wasm_host_fn] fn ic_query_visible_units() -> Vec<RenderUnitInfo>;

/// Get all visible buildings for the local player (fog-filtered).
#[wasm_host_fn] fn ic_query_visible_buildings() -> Vec<RenderBuildingInfo>;

/// Get the local player's resource state (for UI overlays).
#[wasm_host_fn] fn ic_query_resources() -> PlayerResources;

/// Check if a world position is currently visible (not fogged).
#[wasm_host_fn] fn ic_query_is_visible(pos: WorldPos) -> bool;

/// Check if a world position has been explored (shroud removed).
#[wasm_host_fn] fn ic_query_is_explored(pos: WorldPos) -> bool;

pub struct RenderUnitInfo {
    pub handle: u32,          // opaque render handle (not sim EntityId)
    pub unit_type: u32,       // unit type enum
    pub position: WorldPos,
    pub facing: u8,
    pub health_fraction: f32, // 0.0 – 1.0
    pub owner_slot: u8,       // player slot
    pub is_selected: bool,
}

pub struct RenderBuildingInfo {
    pub handle: u32,
    pub building_type: u32,
    pub position: WorldPos,
    pub health_fraction: f32,
    pub owner_slot: u8,
    pub build_progress: f32,  // 0.0 – 1.0 (1.0 = complete)
}
```

**Render mod registration:** A render mod implements the `Renderable` and `ScreenToWorld` traits (see `02-ARCHITECTURE.md` § "3D Rendering as a Mod"). It registers via `ic_render_register()` for each actor type it handles. Unregistered actor types fall through to the default sprite renderer. This allows **partial** render overrides — a mod can replace tank rendering with 3D meshes while leaving infantry as sprites.

**Security:** Render host functions are gated by `ModCapabilities.render`. A gameplay mod (AI, scripting) cannot access `ic_render_*` functions. Render mods cannot access `ic_host_issue_order()` — they draw, they don't command. These capabilities are declared in the mod manifest and verified at load time.

### WASM Pathfinding API Surface

Tier 3 WASM mods can provide custom `Pathfinder` trait implementations (D013, D045). This follows the same pattern as render mods — a well-defined host API surface, capability-gated, with the WASM module implementing the trait through exported functions that the engine calls.

**Why modders want this:** Different games need different pathfinding. A Generals-style total conversion needs layered grid pathfinding with bridge and surface bitmask support. A naval mod needs flow-based routing. A tower defense mod needs waypoint pathfinding. The three built-in presets (Remastered, OpenRA, IC Default) cover the Red Alert family — community pathfinders cover everything else.

```rust
// === Pathfinding Host API (ic_pathfind_* namespace) ===
// Available only to mods with ModCapabilities.pathfinding = true

/// Register this WASM module as a Pathfinder implementation.
/// Called once at load time. The engine calls the exported trait methods below.
#[wasm_host_fn] fn ic_pathfind_register(pathfinder_id: &str);

/// Query terrain passability at a position for a given locomotor.
/// Pathfinder mods need to read terrain but not modify it.
#[wasm_host_fn] fn ic_pathfind_get_terrain(pos: WorldPos) -> TerrainType;

/// Query the terrain height at a position (for 3D-aware pathfinding).
#[wasm_host_fn] fn ic_pathfind_get_height(pos: WorldPos) -> SimCoord;

/// Query entities in a radius (for dynamic obstacle avoidance).
/// Returns entity positions and radii — no gameplay data exposed.
#[wasm_host_fn] fn ic_pathfind_query_obstacles(
    center: WorldPos, radius: SimCoord
) -> Vec<(WorldPos, SimCoord)>;

/// Read the current map dimensions.
#[wasm_host_fn] fn ic_pathfind_map_bounds() -> (WorldPos, WorldPos);

/// Allocate scratch memory from the engine's pre-allocated pool.
/// Pathfinding is hot-path — no per-tick heap allocation allowed.
/// Returns a u32 offset into the guest's linear memory where the
/// engine has reserved `bytes` of scratch space. The host writes
/// into guest memory; the guest accesses it via this offset.
#[wasm_host_fn] fn ic_pathfind_scratch_alloc(bytes: u32) -> u32;

/// Return scratch memory to the pool.
#[wasm_host_fn] fn ic_pathfind_scratch_free(offset: u32, bytes: u32);
```

**WASM-exported trait functions** (the engine *calls* these on the mod):

```rust
// Exported by the WASM pathfinder mod — these map to the Pathfinder trait

/// Called by the engine when a unit requests a path.
#[wasm_export] fn pathfinder_request_path(
    origin: WorldPos, dest: WorldPos, locomotor: LocomotorType
) -> PathId;

/// Called by the engine to retrieve computed waypoints.
#[wasm_export] fn pathfinder_get_path(id: PathId) -> Option<Vec<WorldPos>>;

/// Called by the engine to check passability (e.g., building placement).
#[wasm_export] fn pathfinder_is_passable(
    pos: WorldPos, locomotor: LocomotorType
) -> bool;

/// Called by the engine when terrain changes (building placed/destroyed).
#[wasm_export] fn pathfinder_invalidate_area(
    center: WorldPos, radius: SimCoord
);
```

**Example: Generals-style layered grid pathfinder as a WASM mod**

The C&C Generals source code (GPL v3, `electronicarts/CnC_Generals_Zero_Hour`) uses a layered grid system with 10-unit cells, surface bitmasks, and bridge layers. A community mod can reimplement this as a WASM pathfinder — see `research/pathfinding-ic-default-design.md` § "C&C Generals / Zero Hour" for the `LayeredGridPathfinder` design sketch.

```toml
# generals_pathfinder/mod.toml
[mod]
title = "Generals Pathfinder"
type = "pathfinder"
pathfinder_id = "layered-grid-generals"
display_name = "Generals (Layered Grid)"
description = "Grid pathfinding with bridge layers and surface bitmasks, inspired by C&C Generals"
wasm_module = "generals_pathfinder.wasm"

[capabilities]
pathfinding = true

[config]
zone_block_size = 10
bridge_clearance = 10.0
surface_types = ["ground", "water", "cliff", "air", "rubble"]
```

**Security:** Pathfinding host functions are gated by `ModCapabilities.pathfinding`. A pathfinder mod can read terrain and obstacle positions but cannot issue orders, read gameplay state (health, resources, fog), or access render functions. This is a narrower capability than gameplay mods — pathfinders compute routes, nothing else.

**Determinism:** WASM pathfinder mods execute in the deterministic sim context. All clients run the same WASM binary (verified by SHA-256 hash in the lobby) with the same inputs, producing identical path results/deferred requests. Pathfinding uses a dedicated `pathfinder_fuel_per_tick` budget (see below) because its many-calls-per-tick workload differs from one-shot-per-tick WASM systems.

**Pathfinder fuel budget concern:** Pathfinding has fundamentally different call patterns from other WASM mod types. An AI mod calls `ai_decide()` once per tick — one large computation. A pathfinder mod handles `pathfinder_request_path()` potentially hundreds of times per tick (once per moving unit). The flat `WasmExecutionLimits.fuel_per_tick` budget doesn't distinguish between these patterns: a pathfinder that spends 5,000 fuel per path request × 200 moving units = 1,000,000 fuel, consuming the entire default budget on pathfinding alone.

**Mitigation — scaled fuel allocation for pathfinder mods:**
- Pathfinder WASM modules receive a **separate, larger fuel allocation** (`pathfinder_fuel_per_tick`) that defaults to 5× the standard budget (5,000,000 fuel). This reflects the many-calls-per-tick reality of pathfinding workloads.
- The per-request fuel is not individually capped — the total fuel across all path requests in a tick is bounded. This allows some paths to be expensive (complex terrain) as long as the aggregate stays within budget.
- If the pathfinder exhausts its fuel mid-tick, remaining path requests for that tick return `PathResult::Deferred` — the engine queues them for the next tick(s). This is deterministic (all clients defer the same requests) and gracefully degrades under load rather than truncating individual paths.
- The pathfinder fuel budget is separate from the mod's general `fuel_per_tick` (used for initialization, event handlers, etc.). A pathfinder mod that also handles events gets two budgets.
- Mod manifests can request a custom `pathfinder_fuel_per_tick` value. The lobby displays this alongside other requested limits.

**Multiplayer sync:** Because pathfinding is sim-affecting, all players must use the same pathfinder. The lobby validates that all clients have the same pathfinder WASM module (hash + version + config). A modded pathfinder is treated identically to a built-in preset for sync purposes.

**Ranked policy (D045):** Community pathfinders are allowed in single-player/skirmish/custom lobbies by default, but ranked/community competitive queues reject them unless the exact module hash/version/config profile has been certified and whitelisted (conformance + performance checks).

**Phase:** WASM pathfinding API ships in Phase 6a alongside the mod testing framework and Workshop. Built-in pathfinder presets (D045) ship in Phase 2 as native Rust implementations.

### WASM AI Strategy API Surface

Tier 3 WASM mods can provide custom `AiStrategy` trait implementations (D041, D043). This follows the same pattern as render and pathfinder mods — a well-defined host API surface, capability-gated, with the WASM module implementing the trait through exported functions that the engine calls.

**Why modders want this:** Different games call for different AI approaches. A competitive mod wants a GOAP planner that reads influence maps. An academic project wants a Monte Carlo tree search AI. A Generals-clone needs AI that understands bridge layers and surface types. A novelty mod wants a neural-net AI that learns from replays. The three built-in behavior presets (Classic RA, OpenRA, IC Default) use `PersonalityDrivenAi` — community AIs can use fundamentally different algorithms.

```rust
// === AI Host API (ic_ai_* namespace) ===
// Available only to mods with ModCapabilities.read_visible_state = true
// AND ModCapabilities.issue_orders = true

/// Query own units visible to this AI player.
/// Returns (entity_id, unit_type, position, health, max_health) tuples.
#[wasm_host_fn] fn ic_ai_get_own_units() -> Vec<AiUnitInfo>;

/// Query enemy units visible to this AI player (fog-filtered).
/// Only returns units in line of sight — no maphack.
#[wasm_host_fn] fn ic_ai_get_visible_enemies() -> Vec<AiUnitInfo>;

/// Query neutral/capturable entities visible to this AI player.
#[wasm_host_fn] fn ic_ai_get_visible_neutrals() -> Vec<AiUnitInfo>;

/// Get current resource state for this AI player.
#[wasm_host_fn] fn ic_ai_get_resources() -> AiResourceInfo;

/// Get current power state (production, drain, surplus).
#[wasm_host_fn] fn ic_ai_get_power() -> AiPowerInfo;

/// Get current production queue state.
#[wasm_host_fn] fn ic_ai_get_production_queues() -> Vec<AiProductionQueue>;

These host function signatures are the **logical API** — what mod authors see in documentation and use in their Rust/C/AssemblyScript code. The `#[wasm_host_fn]` attribute generates primitive ABI glue (same `(ptr: i32, len: i32)` MessagePack bridge as exports). Complex types (`WorldPos`, `Vec<T>`, `&str`) are serialized into guest linear memory by the host before the call. See `type-safety.md` § WASM ABI Boundary Policy.

/// Check if a unit type can be built (prerequisites, cost, factory available).
#[wasm_host_fn] fn ic_ai_can_build(unit_type: &str) -> bool;

/// Check if a building can be placed at a position.
#[wasm_host_fn] fn ic_ai_can_place_building(
    building_type: &str, pos: WorldPos
) -> bool;

/// Get terrain type at a position (for strategic planning).
#[wasm_host_fn] fn ic_ai_get_terrain(pos: WorldPos) -> TerrainType;

/// Get map dimensions.
#[wasm_host_fn] fn ic_ai_map_bounds() -> (WorldPos, WorldPos);

/// Get current tick number.
/// Host-side returns SimTick; WASM ABI serializes as u64.
#[wasm_host_fn] fn ic_ai_current_tick() -> SimTick;

/// Get fog-filtered event narrative since a given tick (D041 AiEventLog).
/// Returns a natural-language chronological account of game events.
/// This is the "inner game event log / action story / context" that LLM-based
/// AI (D044) and any WASM AI can use for temporal awareness.
#[wasm_host_fn] fn ic_ai_get_event_narrative(since_tick: SimTick) -> String;

/// Get structured event log since a given tick (D041 AiEventLog).
/// Returns fog-filtered events as typed entries for programmatic consumption.
#[wasm_host_fn] fn ic_ai_get_events(since_tick: SimTick) -> Vec<AiEventEntry>;

/// Issue an order for an owned unit. Returns false if order is invalid.
/// Orders go through the same OrderValidator (D012/D041) as human orders.
#[wasm_host_fn] fn ic_ai_issue_order(order: &PlayerOrder) -> bool;

/// Allocate scratch memory from the engine's pre-allocated pool.
/// Returns a u32 offset into the guest's linear memory (see pathfinder API for details).
#[wasm_host_fn] fn ic_ai_scratch_alloc(bytes: u32) -> u32;
#[wasm_host_fn] fn ic_ai_scratch_free(offset: u32, bytes: u32);

/// String table lookups — resolve interned IDs to human-readable names.
/// These use u32 because they are interned string table indices, not
/// domain entity IDs. Called once at game start; results cached WASM-side.
/// This avoids per-tick String allocation across the WASM boundary.
#[wasm_host_fn] fn ic_ai_get_type_name(type_id: u32) -> String;
#[wasm_host_fn] fn ic_ai_get_event_description(event_code: u32) -> String;
#[wasm_host_fn] fn ic_ai_get_type_count() -> u32;  // total registered unit types

// Host-side structs use newtypes (type-safety.md § WASM ABI Boundary Policy).
// The WASM ABI layer converts to/from primitives at the boundary.
pub struct AiUnitInfo {
    pub tag: UnitTag,             // opaque handle — guest cannot forge
    pub unit_type_id: UnitTypeId, // interned type ID (see ic_ai_get_type_name() for string lookup)
    pub position: WorldPos,
    pub health: SimCoord,
    pub max_health: SimCoord,
    pub is_idle: bool,
    pub is_moving: bool,
}

pub struct AiEventEntry {
    pub tick: SimTick,
    pub event_type: AiEventType,  // typed enum, not raw u32
    pub event_code: u32,          // interned event description ID (see ic_ai_get_event_description())
    pub entity: Option<UnitTag>,
    pub related_entity: Option<UnitTag>,
}
```

**WASM-exported trait functions** (the engine *calls* these on the mod):

These signatures are the **logical API** — the Rust-idiomatic interface that maps to the `AiStrategy` trait. They are NOT the raw WASM ABI. The actual WASM ABI uses only primitives (`i32`/`i64`/`f32`/`f64`). Complex types (`Vec<T>`, `&str`, `Option<T>`, structs) cross the boundary via a **serde bridge**: the host serializes them as MessagePack into the guest's linear memory and passes a `(ptr: i32, len: i32)` pair. The guest deserializes on its side. The `#[wasm_export]` attribute generates this glue code automatically — mod authors write the logical signatures; the toolchain produces the primitive ABI wrappers. See `type-safety.md` § WASM ABI Boundary Policy.

```rust
// LOGICAL API — what mod authors write (Rust-idiomatic).
// The #[wasm_export] macro generates primitive ABI wrappers.
// See "Raw ABI shape" note below.

/// Called once per tick. Returns serialized Vec<PlayerOrder>.
#[wasm_export] fn ai_decide(player_id: u32, tick: u64) -> Vec<PlayerOrder>;

/// Event callbacks — called before ai_decide() in the same tick.
/// Entity IDs are opaque u32 handles (see api-misuse-patterns.md § U4).
#[wasm_export] fn ai_on_unit_created(unit_handle: u32, unit_type: &str);
#[wasm_export] fn ai_on_unit_destroyed(unit_handle: u32, attacker_handle: Option<u32>);
#[wasm_export] fn ai_on_unit_idle(unit_handle: u32);
#[wasm_export] fn ai_on_enemy_spotted(unit_handle: u32, unit_type: &str);
#[wasm_export] fn ai_on_enemy_destroyed(unit_handle: u32);
#[wasm_export] fn ai_on_under_attack(unit_handle: u32, attacker_handle: u32);
#[wasm_export] fn ai_on_building_complete(building_handle: u32);
#[wasm_export] fn ai_on_research_complete(tech: &str);

/// Parameter introspection — called by lobby UI for "Advanced AI Settings."
#[wasm_export] fn ai_get_parameters() -> Vec<ParameterSpec>;
#[wasm_export] fn ai_set_parameter(name: &str, value: i32);

/// Engine scaling opt-out.
#[wasm_export] fn ai_uses_engine_difficulty_scaling() -> bool;

/// Tick budget hint — how many fuel units this AI expects per decide() call.
/// Called once at load time by the engine to tune fuel allocation.
/// Maps to the AiStrategy::tick_budget_hint() seam in crate-graph.md.
/// If not exported, the engine uses WasmExecutionLimits.fuel_per_tick default.
#[wasm_export] fn ai_tick_budget_hint() -> u64;
```

**Raw ABI shape:** The generated WASM ABI for a function like `ai_decide(player_id: u32, tick: u64) -> Vec<PlayerOrder>` is:
```rust
// Generated primitive ABI (what actually crosses the WASM boundary).
// Mod authors never see or write this — it's macro-generated.
#[no_mangle] pub extern "C" fn ai_decide(player_id: i32, tick_lo: i32, tick_hi: i32) -> i64 {
    // tick reconstructed from two i32s (WASM has no native u64 params)
    // Return value is (ptr << 32 | len) packed into i64, pointing to
    // MessagePack-serialized Vec<PlayerOrder> in guest linear memory.
    // The host reads (ptr, len), copies the bytes, deserializes to
    // Vec<PlayerOrder>, then converts to Vec<Verified<PlayerOrder>>
    // via the standard order validation path (D012).
}
```
For `&str` parameters: the host writes the UTF-8 bytes into guest memory and passes `(ptr: i32, len: i32)`. For `Option<u32>`: encoded as `(tag: i32, value: i32)` where `tag=0` is None.

**Security:** AI mods can read visible game state (`ic_ai_get_own_units`, `ic_ai_get_visible_enemies`) and issue orders (`ic_ai_issue_order`). They CANNOT read fogged state — `ic_ai_get_visible_enemies()` returns only units in the AI player's line of sight. They cannot access render functions, pathfinder internals, or other players' private data. Orders go through the same `OrderValidator` as human orders — an AI mod cannot issue impossible commands.

**Determinism:** WASM AI mods execute in the deterministic sim context. Events fire in a fixed order (same order on all clients). `decide()` is called at the same pipeline point on all clients with the same `FogFilteredView`. All clients run the same WASM binary (verified by SHA-256 hash per AI player slot) with the same inputs, producing identical orders.

**Performance:** AI mods share the `WasmExecutionLimits` fuel budget. The `tick_budget_hint()` return value is advisory — the engine uses it for scheduling but enforces the fuel limit regardless. A community AI that exceeds its budget mid-tick gets truncated deterministically.

**Phase:** WASM AI API ships in Phase 6a. Built-in AI (`PersonalityDrivenAi` + behavior presets from D043) ships in Phase 4 as native Rust.

### WASM Format Loader API Surface

Tier 3 WASM mods can register custom asset format loaders via the `FormatRegistry`. This is critical for total conversions that use non-C&C asset formats — analysis of OpenRA mods (see `research/openra-mod-architecture-analysis.md`) shows that non-C&C games on the engine require extensive custom format support:

- **OpenKrush (KKnD):** 15+ custom binary format decoders — `.blit` (sprites), `.mobd` (animations), `.mapd` (terrain), `.lvl` (levels), `.son`/`.soun` (audio), `.vbc` (video). None of these resemble C&C formats.
- **d2 (Dune II):** 6 distinct sprite formats (`.icn`, `.cps`, `.shp` variant), custom map format. Dune II reuses `.wsa` (same format as C&C — handled by `cnc-formats`). `.adl` music also handled by `cnc-formats` (behind `adl` feature flag).
- **OpenHV:** Uses standard PNG/WAV/OGG — no proprietary binary formats at all.

The engine provides a `FormatLoader` WASM API surface that lets mods register custom decoders:

```rust
// === Format Loader Host API (ic_format_* namespace) ===
// Available only to mods with ModCapabilities.format_loading = true

/// Register a custom format loader for a file extension.
/// When the engine encounters a file with this extension, it calls
/// the mod's exported decode function instead of the built-in loader.
#[wasm_host_fn] fn ic_format_register_loader(
    extension: &str, loader_id: &str
);

/// Report decoded sprite data back to the engine.
#[wasm_host_fn] fn ic_format_emit_sprite(
    sprite_id: u32, width: u32, height: u32,
    pixel_data: &[u8], palette: Option<&[u8]>
);

/// Report decoded audio data back to the engine.
#[wasm_host_fn] fn ic_format_emit_audio(
    audio_id: u32, sample_rate: u32, channels: u8,
    pcm_data: &[u8]
);

/// Read raw bytes from an archive or file (engine handles archive mounting).
#[wasm_host_fn] fn ic_format_read_bytes(
    path: &str, offset: u32, length: u32
) -> Option<Vec<u8>>;
```

**Security:** Format loading occurs at asset load time, not during simulation ticks. Format loader mods have file read access (through the engine's archive abstraction) but cannot issue orders, access game state, or call render functions. They decode bytes into engine-standard pixel/audio/mesh data — nothing else.

**Phase:** WASM format loader API ships in Phase 6a alongside the broader mod testing framework. Built-in C&C format loaders (`ra-formats`) ship in Phase 0.

### Mod Testing Framework

`ic mod test` is referenced throughout this document but needs a concrete assertion API and test runner design.

#### Test File Structure

```yaml
# tests/my_mod_tests.yaml
tests:
  - name: "Tank costs 800 credits"
    setup:
      map: test_maps/flat_8x8.oramap
      players: [{ faction: allies, credits: 10000 }]
    actions:
      - build: { actor: medium_tank, player: 0 }
      - wait_ticks: 500
    assertions:
      - entity_exists: { type: medium_tank, owner: 0 }
      - player_credits: { player: 0, less_than: 9300 }

  - name: "Tesla coil requires power"
    setup:
      map: test_maps/flat_8x8.oramap
      players: [{ faction: soviet, credits: 10000 }]
      buildings: [{ type: tesla_coil, player: 0, pos: [4, 4] }]
    actions:
      - destroy: { type: power_plant, player: 0 }
      - wait_ticks: 30
    assertions:
      - condition_active: { entity_type: tesla_coil, condition: "disabled" }
```

#### Lua Test API

For more complex test scenarios, Lua scripts can use test assertion functions:

```lua
-- tests/combat_test.lua
function TestTankDamage()
    local tank = Actor.Create("medium_tank", { Owner = Player.GetPlayer(0), Location = CellPos(4, 4) })
    local target = Actor.Create("light_tank", { Owner = Player.GetPlayer(1), Location = CellPos(5, 4) })

    -- Force attack
    tank.Attack(target)
    Trigger.AfterDelay(100, function()
        Test.Assert(target.Health < target.MaxHealth, "Target should take damage")
        Test.AssertRange(target.Health, 100, 350, "Damage should be in expected range")
        Test.Pass("Tank combat works correctly")
    end)
end

-- Test API globals (available only in test mode)
-- Test.Assert(condition, message)
-- Test.AssertEqual(actual, expected, message)
-- Test.AssertRange(value, min, max, message)
-- Test.AssertNear(actual, expected, tolerance, message)
-- Test.Pass(message)
-- Test.Fail(message)
-- Test.Log(message)
```

#### Test Runner (`ic mod test`)

```
$ ic mod test
Running 12 tests from tests/*.yaml and tests/*.lua...
  ✓ Tank costs 800 credits (0.3s)
  ✓ Tesla coil requires power (0.2s)
  ✓ Tank combat works correctly (0.8s)
  ✗ Harvester delivery rate (expected 100, got 0) (1.2s)
  ...
Results: 11 passed, 1 failed (2.5s total)
```

**Features:**
- `ic mod test` — run all tests in `tests/` directory
- `ic mod test --filter "combat"` — run matching tests
- `ic mod test --headless` — no rendering (CI/CD mode, used by modpack validation)
- `ic mod test --verbose` — show per-tick sim state for failing tests
- `ic mod test --coverage` — report which YAML rules are exercised by tests

**Headless mode:** The engine initializes `ic-sim` without `ic-render` or `ic-audio`. Orders are injected programmatically. This is the same `LocalNetwork` model used for automated testing of the engine itself. Tests run at maximum speed (no frame rate limit).

#### Deterministic Conformance Suites (Pathfinder / SpatialIndex)

Community pathfinders are one of the highest-risk Tier 3 extension points: they are **sim-affecting**, performance-sensitive, and easy to get subtly wrong (nondeterministic ordering, stale invalidation, cache bugs, path output drift across runs). D013/D045 therefore require a built-in conformance layer on top of ordinary scenario tests.

`ic mod test` includes two engine-provided conformance suites: **`PathfinderConformanceTest`** and **`SpatialIndexConformanceTest`**. These are contract tests for "does your implementation satisfy the engine seam safely and deterministically?" — not gameplay-balance tests. They verify deterministic repeatability, output validity, invalidation correctness, snapshot/restore equivalence, and (for spatial) ordering and coherence contracts. Specific test vectors are defined at implementation time.

```bash
ic mod test --conformance pathfinder
ic mod test --conformance spatial-index
ic mod test --conformance all
```

**Ranked / certification linkage (D045):** Passing conformance is the minimum requirement for community pathfinder certification. Ranked queues may additionally require `ic mod perf-test --conformance pathfinder` on the baseline hardware tier. Uncertified pathfinders remain available in single-player/skirmish/custom by default.

This makes D013's open `Pathfinder` seam practical: experimentation stays easy while deterministic multiplayer and ranked integrity remain protected.

**Phase:** Conformance suites ship in Phase 6a (with WASM pathfinder support); performance conformance hooks integrate with `ic mod perf-test` in Phase 6b.

---

For extended Tier 3 mod examples — 3D rendering, custom pathfinding, and custom AI mods — see [Tier 3 WASM Mod Showcases](wasm-showcases.md).
