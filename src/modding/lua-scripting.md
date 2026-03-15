# Tier 2: Lua Scripting

### Decision: Lua over Python

**Why Lua:**
- Tiny runtime (~200KB)
- Designed for embedding — exists for this purpose
- Deterministic (provide fixed-point math bindings, no floats)
- Trivially sandboxable (control exactly what functions are available)
- Industry standard: Factorio, WoW, Garry's Mod, Dota 2, Roblox
- `mlua` or `rlua` crates are mature
- Any modder can learn in an afternoon

**Why NOT Python:**
- Floating-point non-determinism breaks lockstep multiplayer
- GC pauses (reintroduces the problem Rust solves)
- 50-100x slower than native (hot paths run every tick for every unit)
- Embedding CPython is heavy (~15-30MB)
- Sandboxing is basically unsolvable — security disaster for community mods
- `import os; os.system("rm -rf /")` is one mod away

### Lua API — Strict Superset of OpenRA (D024)

Iron Curtain's Lua API is a **strict superset** of OpenRA's 16 global objects. All OpenRA Lua missions run unmodified — same function names, same parameter signatures, same return types.

**OpenRA-compatible globals (all supported identically):**

| Global           | Purpose                                                         |
| ---------------- | --------------------------------------------------------------- |
| `Actor`          | Create, query actors; mutations via trigger context (see below) |
| `Map`            | Terrain, bounds, spatial queries                                |
| `Trigger`        | Event hooks (OnKilled, AfterDelay)                              |
| `Media`          | Audio, video, text display                                      |
| `Player`         | Player state, resources, diplomacy                              |
| `Reinforcements` | Spawn units at edges/drops                                      |
| `Camera`         | Pan, position, shake                                            |
| `DateTime`       | Game time queries                                               |
| `Objectives`     | Mission objective management                                    |
| `Lighting`       | Global lighting control                                         |
| `UserInterface`  | UI text, notifications                                          |
| `Utils`          | Math, random, table utilities                                   |
| `Beacon`         | Map beacon management                                           |
| `Radar`          | Radar ping control                                              |
| `HSLColor`       | Color construction                                              |
| `WDist`          | Distance unit conversion                                        |

**IC-exclusive extensions (additive, no conflicts):**

| Global        | Purpose                                                                                                                                                                                                                                                                                                                    |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Campaign`    | Branching campaign state (D021)                                                                                                                                                                                                                                                                                            |
| `Weather`     | Dynamic weather control (D022)                                                                                                                                                                                                                                                                                             |
| `Layer`       | Map layer activation/deactivation — dynamic map expansion, phase reveals, camera bounds changes. Layers group terrain, entities, and triggers into named sets that can be activated/deactivated at runtime. See § Dynamic Mission Flow below for the full API.                                                             |
| `SubMap`      | Sub-map transitions — enter building interiors, underground sections, or alternate map views mid-mission. Main map state freezes while sub-map is active. See § Dynamic Mission Flow below for the full API.                                                                                                               |
| `Region`      | Named region queries                                                                                                                                                                                                                                                                                                       |
| `Var`         | Mission/campaign variable access                                                                                                                                                                                                                                                                                           |
| `Workshop`    | Mod metadata queries                                                                                                                                                                                                                                                                                                       |
| `LLM`         | LLM integration hooks (Phase 7)                                                                                                                                                                                                                                                                                            |
| `Achievement` | Achievement trigger/query API (D036)                                                                                                                                                                                                                                                                                       |
| `Tutorial`    | Tutorial step management, contextual hints, UI highlighting, camera focus, build/order restrictions for pedagogical pacing (D065). Available in all game modes — modders use it to build tutorial sequences in custom campaigns. See `decisions/09g/D065-tutorial.md` for the full API.                                    |
| `Ai`          | AI scripting primitives (Phase 4) — force composition, resource ratios, patrol/attack commands; inspired by Stratagus's proven Lua AI API (`AiForce`, `AiSetCollect`, `AiWait` pattern — see `research/stratagus-stargus-opencraft-analysis.md`). Enables Tier 2 modders to write custom AI behaviors without Tier 3 WASM. |

Each actor reference exposes **read-only properties** (`.Health`, `.Location`, `.Owner`) and **order-issuing methods** (`.Move()`, `.Attack()`, `.Stop()`, `.Guard()`, `.Deploy()`) — identical to OpenRA's actor property groups. Order-issuing methods enqueue orders into the sim's order pipeline for the current tick; they do not mutate state directly.

> **Two Lua write paths (both deterministic):**
>
> 1. **Order methods** (`.Move()`, `.Attack()`, `.Deploy()`, etc.) — enqueue `PlayerOrder`s processed by `apply_orders()`. Available in all Lua contexts. These are the standard write path.
> 2. **Trigger-context mutations** (`Actor.Create()`, `unit:Teleport()`, `Reinforcements.Spawn()`, `unit:AddAbility()`) — direct sim writes that execute **inside `trigger_system()` (step 19)**. These run at a fixed point in the pipeline on every client with identical state, making them deterministic. They are available in mission/map trigger callbacks and mod command handlers (`Commands.register` — see D058), but **not** in standalone mod scripts running outside the trigger pipeline. This is how OpenRA's Lua missions work — `Actor.Create` spawns an entity during the trigger step, not via the order queue.
>
> The critical guarantee: both paths produce identical results on every client because they execute at deterministic points in the system pipeline with identical inputs.

**In-game command system (inspired by Mojang's Brigadier):** Mojang's Brigadier parser (3,668★, MIT) defines commands as a typed tree where each node is an argument with a parser, suggestions, and permission checks. This architecture — tree-based, type-safe, permission-aware, with mod-injected commands — is the model for IC's in-game console and chat commands. Mods should be able to register custom commands (e.g., `/spawn`, `/weather`, `/teleport` for mission scripting) using the same tree-based architecture, with tab-completion suggestions generated from the command tree. See `research/mojang-wube-modding-analysis.md` § Brigadier and `decisions/09g/D058-command-console.md` for the full command console design.

### API Design Principle: Runtime-Independent API Surface

The Lua API is defined as an **engine-level abstraction**, independent of the Lua VM implementation. This lesson comes from Valve's Source Engine VScript architecture (see `research/valve-github-analysis.md` § 2.3): VScript defined a scripting API abstraction layer so the same mod scripts work across Squirrel, Lua, and Python backends — the *API surface* is the stable contract, not the VM runtime.

For IC, this means:

1. **The API specification is the contract.** The 16 OpenRA-compatible globals and IC extensions are defined by their function signatures, parameter types, return types, and side effects — not by `mlua` implementation details. A mod that calls `Actor.Create("tank", pos)` depends on the API spec, not on how `mlua` dispatches the call.

2. **`mlua` is an implementation detail, not an API boundary.** The `mlua` crate is deeply integrated and switching Lua VM implementations (LuaJIT, Luau, or a future alternative) would be a substantial engineering effort. But mod scripts should never need to change when the VM implementation changes — they interact with the API surface, which is stable.

3. **WASM mods use the same API.** Tier 3 WASM modules access the equivalent API through host functions (see WASM Host API below). The function names, parameters, and semantics are identical. A mission modder can prototype in Lua (Tier 2) and port to WASM (Tier 3) by translating syntax, not by learning a different API.

4. **The API surface is testable independently.** Integration tests define expected behavior per-function ("`Actor.Create` with valid parameters returns an actor reference; with invalid parameters returns nil and logs a warning"). These tests validate any VM backend — they test the specification, not `mlua` internals.

This principle ensures the modding ecosystem survives VM transitions, just as VScript mods survived Valve's backend switches. The API is the asset; the runtime is replaceable.

### Lua API Examples

```lua
-- Mission scripting
function OnPlayerEnterArea(player, area)
  if area == "bridge_crossing" then
    Reinforcements.Spawn("allies", {"Tank", "Tank"}, "north")
    PlayEVA("reinforcements_arrived")
  end
end

-- Custom unit behavior (Trigger.OnUnitCreated is an IC extension on the OpenRA Trigger global)
Trigger.OnUnitCreated("ChronoTank", function(unit)
  unit:AddAbility("chronoshift", {
    cooldown = 120,
    range = 15,
    onActivate = function(target_cell)
      PlayEffect("chrono_flash", unit.position)
      unit:Teleport(target_cell)
      PlayEffect("chrono_flash", target_cell)
    end
  })
end)

-- Idle unit automation (Trigger.OnUnitIdle is an IC extension — inspired by
-- SC2's OnUnitIdle callback, see research/blizzard-github-analysis.md § Part 6)
Trigger.OnUnitIdle("Harvester", function(unit)
  -- Automatically send idle harvesters back to the nearest ore field
  local ore = Map.FindClosestResource(unit.position, "ore")
  if ore then
    unit:Harvest(ore)
  end
end)
```

### Lua Sandbox Rules

- Only engine-provided functions available (no `io`, `os`, `require` from filesystem)
- `os.time()`, `os.clock()`, `os.date()` are removed entirely — Lua scripts read game time via `Trigger.GetTick()` and `DateTime.GameTime`
- Fixed-point math provided via engine bindings (no raw floats)
- Execution resource limits per tick (see `LuaExecutionLimits` below)
- Memory limits per mod

**Lua standard library inclusion policy** (precedent: Stratagus selectively loads stdlib modules, excluding `io` and `package` in release builds — see `research/stratagus-stargus-opencraft-analysis.md` §6). IC is stricter:

| Lua stdlib  | Loaded      | Notes                                                                                                                                                                                        |
| ----------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `base`      | ✅ selective | `print` redirected to engine log; `dofile`, `loadfile`, `load` **removed** (arbitrary code execution vectors)                                                                                |
| `table`     | ✅           | Safe — table manipulation only                                                                                                                                                               |
| `string`    | ✅           | Safe — string operations only                                                                                                                                                                |
| `math`      | ✅ modified  | `math.random` **redirected** to the engine's deterministic PRNG (same sequence as `Utils.RandomInteger()`). Not removed — existing OpenRA scripts that call `math.random()` work unmodified. |
| `coroutine` | ✅           | Useful for mission scripting flow control                                                                                                                                                    |
| `utf8`      | ✅           | Safe — Unicode string handling (Lua 5.4)                                                                                                                                                     |
| `io`        | ❌           | Filesystem access — never loaded in sandbox                                                                                                                                                  |
| `os`        | ❌           | `os.execute()`, `os.remove()`, `os.rename()` are dangerous; entire module excluded                                                                                                           |
| `package`   | ❌           | Module loading from filesystem — never loaded in sandbox                                                                                                                                     |
| `debug`     | ❌           | Can inspect/modify internals, bypass sandboxing; development-only if needed                                                                                                                  |

**Determinism note:** Lua's internal number type is `f64`, but this does not affect sim determinism. Lua has two write paths, both deterministic: (1) **order methods** (`.Move()`, `.Attack()`, etc.) enqueue `PlayerOrder`s processed by the sim's order pipeline, and (2) **trigger-context mutations** (`Actor.Create()`, `unit:Teleport()`, `Reinforcements.Spawn()`) execute direct sim writes inside `trigger_system()` (step 19) — available in mission/map trigger callbacks and mod command handlers (D058), but not in standalone mod scripts outside the trigger pipeline. Campaign state writes (`Campaign.set_flag()`) are also trigger-context mutations. Lua evaluation produces identical results across all clients because it runs at the same point in the system pipeline (the `triggers` step, see system execution order in `02-ARCHITECTURE.md`), with the same game state as input, on every tick. All Lua-driven mutations — orders, entity spawns, campaign state — are applied deterministically within this step, ensuring save/load and replay consistency.

**Additional determinism safeguards:**

- **String hashing → deterministic `pairs()`:** Lua's internal string hash uses a randomized seed by default (since Lua 5.3.3). The sandbox initializes `mlua` with a fixed seed, making hash table slot ordering identical across all clients. Combined with our deterministic pipeline (same code, same state, same insertion order on every client), this makes `pairs()` iteration order deterministic without modification. No sorted wrapper is needed — `pairs()` runs at native speed (zero overhead). For mod authors who want *explicit* ordering for gameplay clarity (e.g., "process units alphabetically"), the engine provides `Utils.SortedPairs(t)` — but this is a convenience for readability, not a determinism requirement. `ipairs()` is already deterministic (sequential integer keys) and should be preferred for array-style tables.
- **Garbage collection timing:** Lua's GC is configured with a fixed-step incremental mode (`LUA_GCINC`) with identical parameters on all clients. Finalizers (`__gc` metamethods) are disabled in the sandbox — mods cannot register them. This eliminates GC-timing-dependent side effects.
- **`math.random()`:** Redirected to the sim's deterministic PRNG (not removed — OpenRA compat requires it). `math.random()` returns a deterministic fixed-point number; `math.random(m)` and `math.random(m, n)` return deterministic integers. `Utils.RandomInteger(min, max)` is the preferred IC API but both draw from the same PRNG and produce identical sequences.

### Lua Execution Resource Limits

WASM mods have `WasmExecutionLimits` (see Tier 3 below). Lua scripts need equivalent protection — without execution budgets, a Lua `while true do end` would block the deterministic tick indefinitely, freezing all clients in lockstep.

The `mlua` crate supports instruction count hooks via `Lua::set_hook(HookTriggers::every_nth_instruction(N), callback)`. The engine uses this to enforce per-tick execution budgets:

```rust
/// Per-tick execution budget for Lua scripts, enforced via mlua instruction hooks.
/// Exceeding the instruction limit terminates the script's current callback —
/// the sim continues without the script's remaining contributions for that tick.
/// A warning is logged and the mod is flagged for the host.
pub struct LuaExecutionLimits {
    pub max_instructions_per_tick: u32,    // mlua instruction hook fires at this count
    pub max_memory_bytes: usize,           // mlua memory limit callback
    pub max_entity_spawns_per_tick: u32,   // Mirrors WASM limit — prevents chain-reactive spawns
    pub max_orders_per_tick: u32,          // Prevents order pipeline flooding
    pub max_host_calls_per_tick: u32,      // Bounds engine API call volume
}

impl Default for LuaExecutionLimits {
    fn default() -> Self {
        Self {
            max_instructions_per_tick: 1_000_000,  // ~1M Lua instructions — generous for missions
            max_memory_bytes: 8 * 1024 * 1024,     // 8 MB (Lua is lighter than WASM)
            max_entity_spawns_per_tick: 32,
            max_orders_per_tick: 64,
            max_host_calls_per_tick: 1024,
        }
    }
}
```

**Why this matters:** The same reasoning as WASM limits applies. In deterministic lockstep, a runaway Lua script on one client blocks the tick for all players (everyone waits for the slowest client). The instruction limit ensures Lua callbacks complete in bounded time. Because the limit is deterministic (same instruction budget, same cutoff point), all clients agree on when a script is terminated — no desync.

**Mod authors can request higher limits** via their mod manifest, same as WASM mods. The lobby displays requested limits and players can accept or reject. Campaign/mission scripts bundled with the game use elevated limits since they are trusted first-party content.

> **Security (V39):** Three edge cases in Lua limit enforcement: `string.rep` memory amplification (allocates before limit fires), coroutine instruction counter resets at yield/resume, and `pcall` suppressing limit violation errors. Mitigations: intercept `string.rep` with pre-allocation size check, verify instruction counting spans coroutines, make limit violations non-catchable (fatal to script context, not Lua errors). See `06-SECURITY.md` § Vulnerability 39.

### Lua Callback-Driven Engine Extensions (Bridging Tier 2 and Tier 3)

The gap between Tier 2 (Lua scripting) and Tier 3 (WASM algorithm replacement) is wide. Most modders who want to customize pathfinding, AI targeting, or damage resolution don't need to replace the *entire algorithm* — they need to change the *rules* the algorithm uses. Writing a full A* pathfinder in WASM is overkill for "I want hovercraft to cross water" or "forests should cost more to traverse."

The solution: **callback-driven APIs** where the algorithm runs in native Rust at full speed, but the modder supplies Lua functions that define the rules. The engine calls the Lua function per-cell/per-unit/per-event, keeping the hot loop native.

This follows the Factorio model: Lua defines *what* (rules, costs, conditions); native code handles *how* (algorithms, data structures, search).

#### Pathfinding Customization

```lua
-- Register a custom locomotor with Lua-defined passability and cost rules.
-- The pathfinding ALGORITHM (flowfield, A*, etc.) runs natively.
-- Your Lua functions are called per-cell to evaluate passability and cost.

Pathfinder.register_locomotor("hovercraft", {
    -- Called per-cell during path search. Return true if this cell is passable.
    -- The native pathfinder skips cells where this returns false.
    passable = function(terrain, cell)
        -- Hovercraft can cross water and land, but not cliffs
        return terrain ~= "cliff"
    end,

    -- Called per-cell during path search. Return the movement cost (integer).
    -- Higher cost = pathfinder prefers other routes. 100 = default land cost.
    cost = function(terrain, cell)
        if terrain == "water" then return 80 end    -- slightly cheaper on water
        if terrain == "road" then return 50 end     -- fast on roads
        if terrain == "forest" then return 200 end  -- slow through forests
        return 100                                  -- default
    end,

    -- Optional: speed multiplier on this terrain (affects movement animation, not pathfinding)
    speed_multiplier = function(terrain)
        if terrain == "road" then return 1.5 end
        if terrain == "water" then return 1.2 end
        return 1.0
    end,
})
```

**How it works under the hood:**

1. Modder registers a locomotor with Lua callbacks
2. When a unit with this locomotor requests a path, the native pathfinder runs its algorithm (flowfield/A*)
3. For each candidate cell, the native code calls the Lua `passable()` and `cost()` callbacks
4. The pathfinder uses the Lua-returned values in its native data structures and search logic
5. The final path is computed entirely in native Rust — Lua only answered "is this cell OK?" and "how expensive is it?"

**Performance characteristic:** The Lua callbacks are called O(cells_searched) times per path request — typically 100-1000 calls for a medium-length path. At ~0.1μs per Lua call, that's 0.01-0.1ms per path. Acceptable for 50-100 path requests per tick. For 500+ concurrent paths, the per-call overhead accumulates — that's when WASM (Tier 3) makes sense.

**Caching optimization:** The engine caches Lua passability/cost results per-cell in a grid. The cache is invalidated when `Pathfinder.invalidate_area()` is called (terrain change). This reduces Lua calls from "per-cell-per-search" to "per-cell-once-until-invalidated" — dramatically improving performance for repeated searches over the same terrain.

#### AI Targeting Customization

```lua
-- Customize how the built-in AI evaluates targets.
-- The targeting ALGORITHM (priority queue, threat assessment) runs natively.
-- Your Lua function scores each potential target.

Ai.register_targeting_rule("plasma-turret", {
    -- Called per-visible-enemy when this unit type is selecting a target.
    -- Return a priority score (integer). Highest score = preferred target.
    -- Return 0 to skip this target entirely.
    score = function(shooter, target)
        -- Plasma turrets prioritize armored vehicles
        if target.armor_class == "heavy" then return 200 end
        if target.armor_class == "medium" then return 150 end
        if target.armor_class == "light" then return 50 end
        -- Don't waste plasma on infantry
        if target.armor_class == "infantry" then return 10 end
        return 100
    end,
})
```

#### Damage Resolution Customization

```lua
-- Customize how damage is calculated for a specific weapon or warhead.
-- The damage PIPELINE (validation, armor lookup, health deduction) runs natively.
-- Your Lua function modifies the damage value before it's applied.

Combat.register_damage_modifier("cryo-warhead", {
    -- Called when this warhead hits a target. Modify the damage before application.
    -- 'context' contains attacker info, target info, terrain, distance.
    modify = function(base_damage, context)
        -- Cryo warhead: double damage vs. vehicles, half damage vs. buildings
        if context.target.category == "vehicle" then
            return base_damage * 2
        end
        if context.target.category == "building" then
            return math.floor(base_damage / 2)
        end
        return base_damage
    end,

    -- Optional: apply a status effect after damage
    on_hit = function(context)
        if context.target.category == "vehicle" then
            -- Slow the target for 5 seconds
            context.target:apply_condition("slowed", { duration = 100, speed_mult = 50 })
        end
    end,
})
```

#### When to Use Lua Callbacks vs. WASM

| Scenario | Use | Why |
|----------|-----|-----|
| "My hex game needs different passability rules" | Lua `passable()` callback | You're customizing *rules*, not the algorithm. Native pathfinder handles the search |
| "My game uses portal-based pathfinding instead of flowfield" | WASM pathfinder | You're replacing the *algorithm*. Lua callbacks can't change how the search works |
| "I want AI to prioritize healers" | Lua `score()` callback | You're customizing targeting *priorities*. The targeting system is still the engine's |
| "I want AI that plays like Starcraft's build-order optimizer" | WASM AI strategy | You're replacing the *entire decision-making algorithm* |
| "Cryo weapons should slow targets" | Lua `on_hit()` callback | You're adding an *effect* to the existing damage pipeline |
| "My game uses a completely different health/armor/shield system" | WASM damage resolver | You're replacing the *entire damage model* |

**The practical rule:** If your sentence starts with "I want X to behave differently" → Lua callbacks. If it starts with "I want to replace the entire X system" → WASM.

#### Determinism Guarantee

Lua callbacks used by the pathfinder/AI/damage systems are **sim-affecting** and must be deterministic:

- No `math.random()` — use `SimRng` via the Lua API (seeded, deterministic)
- No `os.time()` or `os.clock()` — these are removed from the Lua sandbox
- Integer math only (IC's Lua has no float library — `math.floor` etc. operate on fixed-point via `fixed-game-math` bindings)
- Callbacks are called in deterministic order (same cell order, same unit order) across all clients
- The instruction limit (`LuaExecutionLimits.max_instructions_per_tick`) applies — a callback that loops forever is terminated, and all clients terminate at the same point (deterministic cutoff)
