## Multi-Game Extensibility (Game Modules)

The engine is designed as a **game-agnostic RTS framework** (D039) that ships with Red Alert (default) and Tiberian Dawn as built-in game modules. The same engine can run RA2, Dune 2000, or an original game as additional game modules — like OpenRA runs TD, RA, and D2K on one engine.

### Game Module Concept

A game module is a bundle of:

```rust
/// Each supported game implements this trait.
pub trait GameModule {
    /// Register ECS components (unit types, mechanics) into the world.
    fn register_components(&self, world: &mut World);

    /// Return the ordered system pipeline for this game's simulation tick.
    fn system_pipeline(&self) -> Vec<Box<dyn System>>;

    /// Provide the pathfinding implementation (selected by lobby/experience profile, D045).
    fn pathfinder(&self) -> Box<dyn Pathfinder>;

    /// Provide the spatial index implementation (spatial hash, BVH, etc.).
    fn spatial_index(&self) -> Box<dyn SpatialIndex>;

    /// Provide the fog of war implementation (radius, elevation LOS, etc.).
    fn fog_provider(&self) -> Box<dyn FogProvider>;

    /// Provide the damage resolution algorithm (standard, shield-first, etc.).
    fn damage_resolver(&self) -> Box<dyn DamageResolver>;

    /// Provide order validation logic (D041 — engine enforces this before apply_orders).
    fn order_validator(&self) -> Box<dyn OrderValidator>;

    /// Register format loaders (e.g., .vxl for RA2, .shp for RA1).
    fn register_format_loaders(&self, registry: &mut FormatRegistry);

    /// Register render backends (sprite renderer, voxel renderer, etc.).
    fn register_renderers(&self, registry: &mut RenderRegistry);

    /// List available render modes — Classic, HD, 3D, etc. (D048).
    fn render_modes(&self) -> Vec<RenderMode>;

    /// Register game-module-specific commands into the Brigadier command tree (D058).
    /// RA1 registers `/sell`, `/deploy`, `/stance`, etc. A total conversion registers
    /// its own novel commands. The engine's built-in commands (chat, help, cvars) are
    /// pre-registered before this method is called.
    fn register_commands(&self, dispatcher: &mut CommandDispatcher);

    /// YAML rule schema for this game's unit definitions.
    /// Used by the editor for validation/autocomplete and by `ic mod lint` for checking.
    fn rule_schema(&self) -> RuleSchema;
}

/// Describes the YAML rule structure for a game module.
/// Maps top-level YAML keys to expected field types, marks required vs optional fields,
/// and declares valid enum values. Used by the scenario editor (D038) for autocomplete,
/// `ic mod lint` for validation, and LLM generation (D016) for schema-aware output.
struct RuleSchema {
    /// Top-level actor categories (e.g., "infantry", "vehicle", "building", "aircraft").
    actor_categories: Vec<CategoryDef>,
    /// Weapon definition schema.
    weapon_schema: FieldMap,
    /// Projectile/warhead schemas.
    warhead_schema: FieldMap,
}
```

### Bevy App Construction (ic-game)

The `GameModule` trait provides the pieces; the `ic-game` binary assembles them into a Bevy `App`. Each IC crate exposes a Bevy `Plugin` that accepts game-module-provided configuration:

```rust
// ic-game/src/main.rs (simplified)
fn main() {
    let module: Box<dyn GameModule> = select_game_module(); // CLI flag, config, or lobby selection

    let mut app = App::new();
    app.add_plugins(DefaultPlugins);                    // Bevy: windowing, input, asset server
    app.add_plugins(SimPlugin::new(&*module));           // ic-sim: ECS components + system pipeline
    app.add_plugins(NetPlugin);                          // ic-net: NetworkModel, relay client
    app.add_plugins(RenderPlugin::new(&*module));        // ic-render: sprite/voxel backends
    app.add_plugins(AudioPlugin);                        // ic-audio: .aud, EVA, music
    app.add_plugins(UiPlugin);                           // ic-ui: sidebar, minimap, build queue
    app.add_plugins(ScriptPlugin);                       // ic-script: Lua + WASM runtimes
    app.add_plugins(AiPlugin);                           // ic-ai: skirmish AI
    app.run();
}
```

**What `SimPlugin::new()` does internally:**

1. Calls `module.register_components(world)` — inserts game-specific ECS components
2. Stores the system pipeline from `module.system_pipeline()` as a resource — `Simulation::apply_tick()` runs these in order
3. Inserts trait objects as Bevy resources: `module.pathfinder()` → `Res<Box<dyn Pathfinder>>`, `module.spatial_index()` → `Res<Box<dyn SpatialIndex>>`, etc.
4. Calls `module.register_format_loaders(registry)` — registers `.shp`, `.mix`, etc. with Bevy's `AssetServer`
5. Stores `module.rule_schema()` for validation and editor integration

**Plugin registration order matters:** `SimPlugin` must register before `RenderPlugin` (render reads sim state). `NetPlugin` provides the `NetworkModel` resource that `GameLoop` depends on. `ScriptPlugin` registers after `SimPlugin` because Lua/WASM scripts interact with already-registered components.

**Module selection:** `select_game_module()` returns a `Box<dyn GameModule>` based on launch context — defaulting to RA1, switchable via `--mod td` CLI flag or lobby selection. The module choice is fixed for the lifetime of a match; switching modules between matches uses the drop-and-recreate strategy (see `game-loop.md` § Match Cleanup).

**Validation from OpenRA mod ecosystem:** Analysis of six major OpenRA community mods (see `research/openra-mod-architecture-analysis.md`) confirms that every `GameModule` trait method addresses a real extension need:

- **`register_format_loaders()`** — OpenKrush (KKnD on OpenRA) required 15+ custom binary format decoders (`.blit`, `.mobd`, `.mapd`, `.lvl`, `.son`, `.vbc`) that bear no resemblance to C&C formats. TiberianDawnHD needed `RemasterSpriteSequence` for 128×128 HD tiles. Format extensibility is not optional for non-C&C games.
- **`system_pipeline()`** — OpenKrush replaced 16 complete mechanic modules (construction, production, oil economy, researching, bunkers, saboteurs, veterancy). OpenSA (Swarm Assault) added living-world systems (plant growth, creep spawners, colony capture). The pipeline cannot be fixed.
- **`render_modes()`** — TiberianDawnHD is a pure render-only mod (zero gameplay changes) that adds HD sprite rendering with content source detection (Steam AppId, Origin registry, GOG paths). Render mode extensibility enables this cleanly.
- **`pathfinder()`** — OpenSA needed `WaspLocomotor` (flying insect pathfinding); OpenRA/ra2 defines 8 locomotor types (Hover, Mech, Jumpjet, Teleport, etc). RA1's JPS + flowfield is not universal.
- **`fog_provider()` / `damage_resolver()`** — RA2 needs elevation-based LOS and shield-first damage; OpenHV needs a completely different resource flow model (Collector → Transporter → Receiver pipeline). Game-specific logic belongs in the module.
- **`register_commands()`** — RA1 registers `/sell`, `/deploy`, `/stance`, superweapon commands. A Tiberian Dawn module registers different superweapon commands. A total conversion registers entirely novel commands. The engine cannot predefine game-specific commands (D058).

### What the engine provides (game-agnostic)

| Layer           | Game-Agnostic                                                                        | Game-Module-Specific                                                                                                                                                                                                                                                                                    |
| --------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sim core**    | `Simulation`, `apply_tick()`, `snapshot()`, state hashing, order validation pipeline | Components, systems, rules, resource types                                                                                                                                                                                                                                                              |
| **Positions**   | `WorldPos { x, y, z }`                                                               | `CellPos` (grid-based modules), coordinate mapping, z usage                                                                                                                                                                                                                                             |
| **Pathfinding** | `Pathfinder` trait, `SpatialIndex` trait                                             | Remastered/OpenRA/IC flowfield (RA1, D045), navmesh (future), spatial hash vs BVH                                                                                                                                                                                                                       |
| **Fog of war**  | `FogProvider` trait                                                                  | Radius fog (RA1), elevation LOS (RA2/TS), no fog (sandbox)                                                                                                                                                                                                                                              |
| **Damage**      | `DamageResolver` trait                                                               | Standard pipeline (RA1), shield-first (RA2), sub-object (Generals)                                                                                                                                                                                                                                      |
| **Validation**  | `OrderValidator` trait (engine-enforced)                                             | Per-module validation rules (ownership, affordability, placement, etc.)                                                                                                                                                                                                                                 |
| **Networking**  | `NetworkModel` trait, `RelayCore` library, relay server binary, lockstep, replays    | `PlayerOrder` variants (game-specific commands)                                                                                                                                                                                                                                                         |
| **Rendering**   | Camera, sprite batching, UI framework; post-FX pipeline available to modders         | Sprite renderer (RA1), voxel renderer (RA2), mesh renderer (3D mod/future)                                                                                                                                                                                                                              |
| **Modding**     | YAML loader, Lua runtime, WASM sandbox, workshop                                     | Rule schemas, API surface exposed to scripts                                                                                                                                                                                                                                                            |
| **Formats**     | Archive loading, format registry                                                     | classic Westwood 2D family (`.mix`, `.shp`, `.tmp`, `.pal`, `.aud`, `.vqa`, `.vqp`, `.wsa`, `.fnt`, `.cps`, `.eng`, mission `.bin` / `.mpr`), RA2 families (`.vxl`, `.hva`, `.bag` / `.idx`, `.csf`, `.map`), and SAGE families (`.big`, `.w3d`, `.wnd`, `.str`, `.apt` bundle family, texture formats) |

### RA2 Extension Points

RA2 / Tiberian Sun would add these to the existing engine without modifying the core:

| Extension                     | What It Adds                                           | Engine Change Required                                |
| ----------------------------- | ------------------------------------------------------ | ----------------------------------------------------- |
| Voxel models (`.vxl`, `.hva`) | New format parsers                                     | None — additive to `ic-cnc-content`                   |
| Terrain elevation             | Z-axis in pathfinding, ramps, cliffs                   | None — `WorldPos.z` and `CellPos.z` are already there |
| Voxel rendering               | GPU voxel-to-sprite at runtime                         | New render backend in `RenderRegistry`                |
| Garrison mechanic             | `Garrisonable`, `Garrisoned` components + system       | New components + system in pipeline                   |
| Mind control                  | `MindController`, `MindControlled` components + system | New components + system in pipeline                   |
| IFV weapon swap               | `WeaponOverride` component                             | New component                                         |
| Prism forwarding              | `PrismForwarder` component + chain calculation system  | New component + system                                |
| Bridges / tunnels             | Layered pathing with Z transitions                     | Uses existing `CellPos.z`                             |

### Current Target: The Isometric C&C Family

The **first-party game modules** target the **isometric C&C family**: Red Alert, Red Alert 2, Tiberian Sun, Tiberian Dawn, and Dune 2000 (plus expansions and total conversions in the same visual paradigm). These games share:

- Fixed isometric camera
- Grid-based terrain (with optional elevation for TS/RA2)
- Sprite and/or voxel-to-sprite rendering
- `.mix` archives and related format lineage
- Discrete cell-based pathfinding (flowfields, hierarchical A*)

### Architectural Openness: Beyond Isometric

C&C Generals and later 3D titles (C&C3, RA3) are **not current targets** — we build only grid-based pathfinding and isometric rendering today. But the architecture deliberately avoids closing doors:

| Engine Concern     | Grid Assumption?   | Trait-Abstracted?             | 3D/Continuous Game Needs...                                                                                                                          |
| ------------------ | ------------------ | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Coordinates        | No (`WorldPos`)    | N/A — universal               | Nothing. `WorldPos` works for any spatial model.                                                                                                     |
| Pathfinding        | Implementation     | Yes (`Pathfinder` trait)      | A `NavmeshPathfinder` impl. Zero sim changes.                                                                                                        |
| Spatial queries    | Implementation     | Yes (`SpatialIndex` trait)    | A `BvhSpatialIndex` impl. Zero combat/targeting changes.                                                                                             |
| Fog of war         | Implementation     | Yes (`FogProvider` trait)     | An `ElevationFogProvider` impl. Zero sim changes.                                                                                                    |
| Damage resolution  | Implementation     | Yes (`DamageResolver` trait)  | A `SubObjectDamageResolver` impl. Zero projectile changes.                                                                                           |
| Order validation   | Implementation     | Yes (`OrderValidator` trait)  | Module-specific rules. Engine still enforces the contract.                                                                                           |
| AI strategy        | Implementation     | Yes (`AiStrategy` trait)      | Module-specific AI. Same lobby selection UI.                                                                                                         |
| Rendering          | Implementation     | Yes (`Renderable` trait)      | A mesh renderer impl. Already documented ("3D Rendering as a Mod").                                                                                  |
| Camera             | Implementation     | Yes (`ScreenToWorld` trait)   | A perspective camera impl. Already documented.                                                                                                       |
| Input              | No (`InputSource`) | Yes                           | Nothing. Orders are orders.                                                                                                                          |
| Networking         | No                 | Yes (`NetworkModel` trait)    | Nothing. Lockstep works regardless of spatial model.                                                                                                 |
| Format loaders     | Implementation     | Yes (`FormatRegistry`)        | New parsers for `.cps`, `.eng`, `.vxl`, `.hva`, `.bag` / `.idx`, `.big`, `.w3d`, `.wnd`, `.str`, `.apt`, and other game-family formats are additive. |
| Building placement | Data-driven        | N/A — YAML rules + components | Different components (no `RequiresBuildableArea`). YAML change.                                                                                      |

The key insight: the engine core (`Simulation`, `apply_tick()`, `GameLoop`, `NetworkModel`, `Pathfinder`, `SpatialIndex`, `FogProvider`, `DamageResolver`, `OrderValidator`) is spatial-model-agnostic. Grid-based pathfinding is a *game module implementation*, not an engine assumption — the same way `LocalNetwork` is a network implementation, not the only possible one.

A Generals-class game module would provide its own `Pathfinder` (navmesh), `SpatialIndex` (BVH), `FogProvider` (elevation LOS), `DamageResolver` (sub-object targeting), `AiStrategy` (custom AI), `Renderable` (mesh), and format loaders — while reusing the sim core, networking, modding infrastructure, workshop, competitive infrastructure, and all shared systems (production, veterancy, replays, save games). See D041 in `decisions/09d-gameplay.md` for the full trait-abstraction strategy.

This is not a current development target. We build only the grid implementations. But the trait seams exist from day one, so the door stays open — for us or for the community.

### 3D Rendering as a Mod (Not a Game Module)

While 3D C&C titles are not current development targets, the architecture explicitly supports **3D rendering mods** for any game module. A "3D Red Alert" mod replaces the visual presentation while the simulation, networking, pathfinding, and rules are completely unchanged.

This works because the sim/render split is absolute — the sim has no concept of camera, sprites, or visual style. Bevy already ships a full 3D pipeline (PBR materials, GLTF loading, skeletal animation, dynamic lighting, shadows), so a 3D render mod leverages existing infrastructure.

**What changes vs. what doesn't:**

| Layer         | 3D Mod Changes? | Details                                                                                                                                                               |
| ------------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Simulation    | No              | Same tick, same rules, same grid                                                                                                                                      |
| Pathfinding   | No              | Grid-based flowfields still work (SC2 is 3D but uses grid pathing). A future game module could provide a `NavmeshPathfinder` instead — independent of the render mod. |
| Networking    | No              | Orders are orders                                                                                                                                                     |
| Rules / YAML  | No              | Tank still costs 800, has 400 HP                                                                                                                                      |
| Rendering     | Yes             | Sprites → GLTF meshes, isometric camera → free 3D camera                                                                                                              |
| Input mapping | Yes             | Click-to-world changes from isometric transform to 3D raycast                                                                                                         |

**Architectural requirements to enable this:**

1. **`Renderable` trait is mod-swappable.** A WASM Tier 3 mod can register a 3D render backend that replaces the default sprite renderer.
2. **Camera system is configurable.** Default is fixed isometric; a 3D mod substitutes a free-rotating perspective camera. The camera is purely a render concern — the sim has no camera concept.
3. **Asset pipeline accepts 3D models.** Bevy natively loads GLTF/GLB. The mod maps unit IDs to 3D model paths in YAML:

```yaml
# Classic 2D (default)
rifle_infantry:
  render:
    type: sprite
    sequences: e1

# 3D mod override
rifle_infantry:
  render:
    type: mesh
    model: models/infantry/rifle.glb
    animations:
      idle: Idle
      move: Run
      attack: Shoot
```

4. **Click-to-world abstracted behind trait.** Isometric screen→world is a linear transform. 3D perspective screen→world is a raycast. Both produce a `WorldPos`. Grid-based game modules convert to `CellPos` as needed.
5. **Terrain rendering decoupled from terrain data.** The sim's spatial representation is authoritative. A 3D mod provides visual terrain geometry that matches it.

**Key benefits:**
- **Cross-view multiplayer.** A player running 3D can play against a player running classic isometric — the sim is identical. Like StarCraft Remastered's graphics toggle, but more radical.
- **Cross-view replays.** Watch any replay in 2D or 3D.
- **Orthogonal to gameplay mods.** A balance mod works in both views. A 3D graphics mod stacks with a gameplay mod.
- **Toggleable, not permanent.** D048 (Switchable Render Modes) formalizes this: a 3D render mod adds a render mode alongside the default 2D modes. F1 cycles between classic, HD, and 3D — the player isn't locked into one view. See `decisions/09d/D048-render-modes.md`.

This is a **Tier 3 (WASM) mod** — it replaces a rendering backend, which is too deep for YAML or Lua. See `04-MODDING.md` for details.

### Design Rules for Multi-Game Safety

1. **No game-specific enums in engine core.** Don't put `enum ResourceType { Ore, Gems }` in `ic-sim`. Resource types come from YAML rules / game module registration.
2. **Position is always 3D.** `WorldPos` carries Z. RA1 sets it to 0. The cost is one extra `i32` per position — negligible. `CellPos` is a grid-game-module convenience type, not an engine-core requirement.
3. **Pathfinding and spatial queries are behind traits.** `Pathfinder` and `SpatialIndex` — like `NetworkModel`. Grid implementations are the default; the engine core never calls grid-specific functions directly.
4. **System pipeline is data, not code.** The game module returns its system list; the engine executes it. No hardcoded `harvester_system()` call in engine core.
5. **Render through `Renderable` trait.** Sprites and voxels implement the same trait. The renderer doesn't know what it's drawing.
6. **Format loaders are pluggable.** `ic-cnc-content` provides parsers; the game module tells the asset pipeline which ones to use.
7. **`PlayerOrder` is extensible.** Use an enum with a `Custom(GameSpecificOrder)` variant, or make orders generic over the game module.
8. **Fog, damage, and validation are behind traits (D041).** `FogProvider`, `DamageResolver`, and `OrderValidator` — each game module supplies its own implementation. The engine core calls trait methods, never game-specific fog/damage/validation logic directly.
9. **AI strategy is behind a trait (D041).** `AiStrategy` lets each game module (or difficulty preset) supply different decision-making logic. The engine schedules AI ticks; the strategy decides what to do.
