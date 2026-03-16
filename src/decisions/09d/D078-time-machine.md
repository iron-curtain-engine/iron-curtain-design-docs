## D078: Time-Machine Mechanics — Replay Takeover, Temporal Campaigns, and Multiplayer Time Modes

|                |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Status**     | Draft                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| **Phase**      | Phase 3 (replay takeover), Phase 4 (campaign time machine), Phase 5 (multiplayer time modes), Phase 7 (advanced co-op/adversarial timelines)                                                                                                                                                                                                                                                                                                                                                     |
| **Depends on** | D010 (snapshottable state), D012 (order validation), D021 (branching campaigns), D024 (Lua scripting), D033 (QoL presets), D043 (AI presets), D055 (ranked matchmaking), D077 (replay highlights)                                                                                                                                                                                                                                                                                                |
| **Driver**     | IC's deterministic simulation, snapshottable state, and keyframe-based replay system provide architectural primitives that no other RTS has leveraged for time-travel gameplay. StarCraft 2 shipped "Take Command" (replay takeover) to critical praise. Achron proved time-travel RTS is viable but failed on execution fundamentals. C&C Red Alert's lore is built on time travel (Einstein, Chronosphere). IC can be the first RTS to deliver time-travel as both a tool and a game mechanic. |

### Philosophy & Scope Note

> **This decision is Draft — experimental, not committed.** It requires community validation before moving to Accepted.
>
> **What is proven:** Layer 1 (replay takeover) is SC2-proven with clear coaching/practice/tournament-recovery value. The architectural primitives (snapshot restore, keyframe seeking) already exist for replay viewing and reconnection. Thematic fit with C&C Red Alert's time-travel lore (Einstein, Chronosphere) is natural.
>
> **What is experimental:** Layers 2–4 (campaign time machine, multiplayer time modes, temporal co-op) are driven by architectural opportunity — "IC's architecture provides primitives no other RTS has leveraged" — rather than documented community pain points. The 12 community pain points in `01-VISION.md` do not include "replay takeover" or "campaign rewind." The closest ("campaigns are incomplete") is addressed by D021.
>
> **Philosophy tensions (acknowledged):**
> - **Scope discipline (Principle 6):** D078 spans 4 phases and 6 crates. The layered design mitigates this (each layer ships independently), but the cross-cutting surface is real.
> - **Core loop relevance (Principle 15):** Time-machine mechanics are *on top of* Extract-Build-Amass-Crush, not *part of* it. They must not distract from the core loop.
> - **Community-driven design (Principle 10):** Layers 2–4 should be validated with community feedback before becoming permanent. If the community doesn't find them fun or useful, they should be cut rather than becoming "entrenched complexity."
>
> **Recommendation:** Ship Layer 1 with the replay system (Phase 3). Validate Layers 2–4 with playtesters before committing design and implementation effort. Do not update affected docs (roadmap, replays.md, campaigns.md, multiplayer.md, game-loop.md, save-replay-formats.md) until this decision moves from Draft to Accepted.

### Decision Capsule (LLM/RAG Summary)

- **Status:** Draft
- **Phase:** Phase 3 (replay takeover) → Phase 4 (campaign time machine) → Phase 5 (multiplayer time modes) → Phase 7 (advanced temporal campaigns)
- **Canonical for:** Replay-to-gameplay branching ("Take Command"), speculative branch preview, campaign time-machine weapon, campaign mission archetypes (convergence puzzles, information locks, causal loops, dual-state battlefields, retrocausal modification), multiplayer temporal game modes, temporal manipulation support powers, temporal pincer co-op
- **Scope:** `ic-game` (takeover orchestration via `GameRunner::restore_full()`, time-machine game mode), `ic-sim` (snapshot production — `snapshot()`, `delta_snapshot()`, `state_hash()`), `ic-script` (Lua `TimeMachine` API), `ic-ui` (takeover UX, timeline visualization), `ic-net` (multiplayer takeover lobby + time mode relay support — Phase 5+)
- **Decision:** IC implements time-machine mechanics in four layers: (1) replay takeover lets any player branch from any point in a replay into live gameplay; (2) a diegetic campaign time-machine weapon lets players rewind to earlier missions with controlled state carryover; (3) multiplayer time-machine game modes (capture/race, timeline peek); (4) advanced co-op/adversarial temporal campaigns. Each layer builds on the previous and can ship independently.
- **Why:**
  - The snapshot + deterministic-replay architecture already solves the hard engineering problem (restore state at tick T, continue with new input)
  - StarCraft 2's "Take Command" proved the concept for replay takeover; IC can ship a better version with richer state (campaign, Lua scripts, fog)
  - C&C Red Alert's entire lore is founded on time travel (Einstein, Chronosphere) — a time-machine mechanic is thematically native
  - Achron's failure was not the mechanic itself but the RTS fundamentals around it — IC has the foundation to succeed where Achron fell short
  - No RTS has explored time travel as a campaign narrative weapon with persistent state carryover
- **Non-goals:** Free-form multiplayer time travel during live matches (Achron model — too complex, too niche). Replacing the core RTS gameplay loop with time puzzles. Mandatory time-machine usage in any campaign.
- **Out of current scope:** LLM-generated alternate timeline missions (Phase 7+, depends on D016 maturity). Cross-match timeline persistence (career-spanning time travel).
- **Invariants preserved:** Deterministic sim (`ic-sim` remains pure — time machine operates through snapshot restore + new order streams, not by mutating sim internals). Replay integrity (original `.icrep` files are immutable; branched replays reference their parent). Campaign state serializable (time-machine state is part of `CampaignState`).
- **Defaults / UX behavior:** Replay takeover available from the replay viewer via "Take Command" button (native `.icrep` replays only — blocked for imported/foreign replays per D056 divergence risk). Campaign time machine is opt-in per campaign definition (YAML). Multiplayer time modes are custom game modes, not ranked by default.
- **Security / Trust impact:** Replay takeover from ranked replays does not affect the original match's rating. Time-machine multiplayer modes excluded from ranked queue unless explicitly whitelisted by D055 seasonal config.
- **Performance impact:** Snapshot restore + re-simulation to target tick is bounded by existing keyframe seeking performance (<100ms worst case per D010). No new per-tick cost in standard gameplay.
- **Public interfaces / types / commands:** `ReplayTakeover`, `TakeoverSource`, `PlayerAssignment` (Layer 1); `TimeMachineState`, `TimeMachineCarryover`, `CampaignProgressSnapshot`, `CampaignProgressCheckpoint`, `CheckpointId` (Layer 2); `GhostUnit` component (Layer 4); `TimeMachine` (Lua global); `/takeover`, `/timemachine` (console commands)
- **Affected docs:** `player-flow/replays.md` (takeover UX + InReplay → Loading → InGame transition), `modding/campaigns.md` (time-machine YAML schema + Lua API + `CampaignState` additions), `player-flow/multiplayer.md` (time-machine game modes), `formats/save-replay-formats.md` (branched replay metadata fields), `architecture/game-loop.md` (new state transition path for takeover)
- **Keywords:** time machine, replay takeover, take command, resume from replay, speculative branch preview, what-if ghost overlay, temporal campaign, timeline branch, chronosphere, time travel RTS, Achron, rewind, what-if, alternate timeline, time race, convergence puzzle, attractor field, information lock, causal loop, bootstrap paradox, dual-state battlefield, retrocausal, temporal pincer, ghost army, temporal support power, age structure, chrono stasis, GGPO rollback, Novikov self-consistency, many-worlds, Steins;Gate, Tenet, Singularity TMD, Zero Escape, Titanfall Effect and Cause

---

### Problem

IC's deterministic simulation and keyframe-based replay system already contain the architectural primitives for time-travel gameplay:

- **Snapshot restore:** `GameRunner::restore_full()` (the canonical end-to-end restore contract in `ic-game` per `02-ARCHITECTURE.md`) can reconstruct any prior state from a `SimSnapshot`
- **Arbitrary seeking:** Keyframe index enables jumping to any tick in <100ms
- **Campaign state serialization:** `CampaignState` (roster, veterancy, flags, equipment) is part of `SimSnapshot`
- **Lua/WASM script state:** `ScriptState` is snapshottable and restorable

These primitives are currently used only for replay viewing, reconnection, and save/load. They are not exposed as gameplay mechanics. This is a missed opportunity — especially given that:

1. C&C Red Alert's entire narrative premise is time travel (Einstein erasing Hitler, Chronosphere technology)
2. The RTS genre has never delivered a polished time-travel gameplay experience (Achron proved the concept but failed on RTS fundamentals)
3. StarCraft 2 shipped replay takeover ("Take Command") to enthusiastic reception, validating the replay-to-gameplay transition
4. Time-loop and time-travel mechanics are among the most praised innovations in modern game design (Outer Wilds, Braid, Prince of Persia: Sands of Time, Deathloop)

### Prior Art

| Game / Feature                                       | Mechanic                                                                                                                                                                  | Result                                                                                                           | Lesson for IC                                                                                                                                                        |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **StarCraft 2: Heart of the Swarm** — "Take Command" | Jump into any replay point, take over any player's control, continue playing                                                                                              | Enthusiastically received; used for coaching, practice, tournament recovery, "what if" exploration               | Proven concept. Ship as baseline. IC's richer state (campaign, Lua, fog) makes it more powerful                                                                      |
| **StarCraft 2: HotS** — "Resume from Replay"         | Recover dropped/crashed games by restoring replay state at a chosen point                                                                                                 | Essential for tournament integrity; used by referees to recover from disconnects                                 | IC's reconnection protocol already supports this via relay snapshot transfer. Expose it explicitly                                                                   |
| **Achron** (2011)                                    | Free-form multiplayer time travel. "Timewaves" propagate past changes to present. Energy cost for timeline modifications. Won Best Original Game Mechanic (GameSpot 2011) | Brilliant core idea, terrible RTS execution — primitive pathfinding, bad UI, steep learning curve. Metacritic 56 | The mechanic works, but the game underneath must be excellent first. IC has the RTS foundation Achron lacked. Don't let time travel overshadow core gameplay quality |
| **Braid** (2008)                                     | Six worlds, each with a unique time mechanic (global rewind, time-immune objects, time tied to movement). GOTY-level indie                                                | Made time manipulation *the puzzle*. Each mechanic deeply explored. Players felt clever, not confused            | Each time-machine use should feel like solving a puzzle. Design missions where foreknowledge creates new strategic possibilities                                     |
| **Prince of Persia: Sands of Time** (2003)           | Limited rewind charges. Reverse death, freeze enemies, slow time                                                                                                          | Forgiving but not free — limited charges create tension. Defined "time rewind as resource"                       | Limited uses create meaningful decisions. "Should I rewind now or save it?" is the fun                                                                               |
| **Outer Wilds** (2019)                               | 22-minute time loop. Knowledge is the only persistent upgrade. GOTY at Eurogamer, Polygon                                                                                 | "Knowledge is the upgrade." Players improve, not their avatar                                                    | "Knowledge carries back" is the most elegant carryover rule. Explored terrain (shroud) + enemy positions known, but army resets                                      |
| **Chrono Trigger** (1995)                            | Travel between eras. Past actions change the future. 13+ endings                                                                                                          | Cause-and-effect across time is deeply satisfying. "Plant a seed in 600 AD, find a forest in 1000 AD"            | Cross-mission cause-and-effect is IC's campaign branching. Time machine amplifies it by letting the player choose to go back and set different flags                 |
| **Deathloop** / **Returnal** (2021)                  | Time loops with knowledge persistence. Select upgrades carry between loops                                                                                                | "Time is on your side." The loop empowers, not punishes                                                          | Frame the time machine as empowerment. The player should feel like a genius for using foreknowledge                                                                  |
| **C&C Red Alert** (1996)                             | Einstein used a time machine in 1946 (Trinity, NM) to remove Hitler, creating the RA timeline. The Chronosphere is a separate teleportation device developed later | Einstein's original time machine research is the narrative foundation for D078. The campaign asks: "what if someone rebuilt the device he used in Trinity?" | The time machine is Einstein's 1946 research, rebuilt as a limited prototype |

### Cross-Domain Inspirations

Beyond games, time-travel mechanics draw from netcode engineering, theoretical physics, and science fiction. These inform both the technical implementation and the design of fun, novel game mechanics.

#### Netcode Techniques as Game Mechanics

| Source                                          | Technique                                                                                                              | IC Application                                                                                                                                                                                                                                                  |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **GGPO / Rollback Netcode**                     | Snapshot full state every frame; on misprediction, roll back to divergence point and re-simulate with corrected inputs | IC's takeover is architecturally identical but triggered by the player, not mispredictions. The same snapshot+re-simulate primitive enables **speculative branch previews** — "what if?" ghost overlays showing two possible outcomes before the player commits |
| **Delta Rollback**                              | Instead of full snapshots, track only changed fields. Reduces rollback cost dramatically                               | IC already has `#[derive(TrackChanges)]` with per-field change masks. Makes speculative branching cheap enough for real-time "what if?" previews — the delta between two speculative branches is small                                                          |
| **Speculative Execution** (lockstep prediction) | Run the sim ahead with predicted inputs; correct when real inputs arrive                                               | A "timeline peek" ability: activate the time machine, the game speculatively simulates N seconds forward using current AI orders, shows a ghost overlay of "what will happen if nothing changes." Uses the same infrastructure as replay fast-forward           |

#### Physics Theories as Game Rules

| Theory                                 | Core Idea                                                                                                                                  | Game Rule                                                                                                                                                                                                                                                                                                          |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Novikov Self-Consistency Principle** | If time travel exists, paradoxes are impossible. Any action taken in the past was *always* part of history                                 | **Locked outcomes.** When the player rewinds, certain mission outcomes are immutable — they happened in the original timeline and can't be changed. The puzzle: figure out *which* outcomes are locked and *which* are mutable. "The bridge was always going to be destroyed. But *who* destroyed it is up to you" |
| **Many-Worlds Interpretation**         | Every quantum event branches into all possible outcomes. All timelines are equally real                                                    | **Parallel timeline reality.** Time-machine use doesn't "rewrite" the old timeline — it creates a new one alongside it. In co-op, one player continues in the old timeline while another plays the new branch. Both are "real" and can affect each other through shared campaign state                             |
| **Closed Timelike Curves**             | An object's worldline loops back on itself. Effect can precede cause                                                                       | **Causal loop missions.** The player receives mysterious reinforcements at mission start, then must send those exact reinforcements back in time at mission end. If they fail to close the loop, the mission retroactively becomes harder                                                                          |
| **Retrocausality**                     | Future events influence past events                                                                                                        | **Reverse cause-and-effect.** Actions in a *later* campaign mission retroactively modify an *earlier* mission's conditions. The player replays the earlier mission and finds it changed — different enemy positions, altered terrain — because of what they did in the future                                      |
| **Attractor Fields** (Steins;Gate)     | Worldlines within an attractor field converge on the same key outcomes. Escaping requires shifting to a different attractor field entirely | **Convergence puzzles.** Campaign missions have "convergence points" — outcomes that happen regardless of rewinds. To break convergence, the player must accumulate enough divergence across multiple rewinds + different choices to shift to a new attractor field and unlock a hidden campaign branch            |

#### Science Fiction Narrative Models

| Source                                                | Concept                                                                                                                                                  | IC Mechanic                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Tenet** (Nolan, 2020) — Temporal Pincer Movement    | Two teams attack the same objective from opposite temporal directions. Forward team reports; backward team uses that knowledge                           | **Temporal pincer co-op.** Player A plays a battle normally. Player B replays the *same battle* but with Player A's replay as ghost support fire — the failed timeline's actions become helpful cover. See Layer 4 Concept A                                                       |
| **Dark** (Netflix) — Bootstrap Paradox                | Objects exist in causal loops with no origin. The time machine's blueprints are given to the person who builds the time machine                          | **Self-causing technology.** A campaign mission where the player captures enemy temporal technology that turns out to be derived from research *they* sent back in time. The player's own time-travel journey contributed to the enemy's capabilities — fitting for C&C's Einstein narrative |
| **Everything Everywhere All at Once** — Verse-Jumping | Characters tap into alternate versions of themselves, gaining skills/knowledge from other timelines                                                      | **Timeline borrowing.** Instead of replaying a mission, briefly access an alternate timeline version and bring back one specific advantage (unit composition, position, intel). Requires "anchor" conditions true in both timelines — a strategic puzzle to set up                 |
| **Steins;Gate** — Worldline Convergence               | Certain events are fated across all worldlines in an attractor field. Small details change, but major beats are locked                                   | **Fated events + hidden branches.** Most mission outcomes are cosmetically different but narratively convergent. Only specific multi-rewind strategies break convergence and reveal secret campaign branches — massive replayability                                               |
| **Zero Escape: Virtue's Last Reward** — FLOW Chart    | Protagonist jumps between timeline branches using a flowchart. Information from one branch unlocks progress in another. Passwords carry across timelines | **Information locks.** Certain missions are impassable without intel from a different timeline branch. The player must rewind, play a different path, learn the enemy's codes/weaknesses, then jump back. Transforms the time machine from safety net into required puzzle tool    |

#### Additional Game Mechanics

| Game                                         | Mechanic                                                                                                                                                                                         | IC Application                                                                                                                                                                                                                                                                                                                                                                                    |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Titanfall 2 — "Effect and Cause"** (2016)  | Player shifts between past/present versions of the same map with a button press. Two vertically stacked, perfectly aligned maps. Universally praised as one of the best FPS levels ever designed | **Dual-state battlefield.** A mission type where the battlefield exists in two temporal states. Buildings that are ruins in the present are intact in the past. The player shifts units between states — build in the past (abundant resources, alert enemy), attack in the present (weakened enemy, devastated terrain). Achievable within single-sim using a temporal-state flag, not multi-sim |
| **Singularity — TMD** (2010, Raven Software) | A device that ages or restores individual objects/enemies. Broken bridges restored. Enemies aged to dust. Rusted safes crumble open                                                              | **Temporal manipulation support power.** Age an enemy bridge to collapse it (cut reinforcement routes). Restore a destroyed refinery for your own use. Age an enemy tank column (reduce veterancy/health). Extends the Chronosphere from teleportation to per-object time manipulation                                                                                                            |
| **Quantum Chess** (Akl)                      | Chess pieces exist in superposition of two types until moved, then collapse to one. The opponent doesn't know which until engagement                                                             | **Quantum units (experimental).** Units built at the Chronosphere exist in superposition of two unit types. They collapse to whichever type is advantageous upon combat contact. The opponent can't scout which type they'll face — information warfare meets quantum mechanics                                                                                                                   |

### Decision

Time-machine mechanics are implemented in four independent layers, each building on the previous:

---

#### Layer 1: Replay Takeover ("Take Command")

**Phase 3 — ships with the replay system.**

While watching a native `.icrep` replay, the player can stop at any point, select a faction, and transition into live gameplay from that moment. The original replay remains immutable; a new branched replay is recorded.

**Takeover is restricted to native `.icrep` replays only.** Imported/foreign replays (OpenRA `.orarep`, Remastered — per D056) are excluded because their playback may visibly diverge from IC's simulation, making the restored state unreliable for live gameplay. If a foreign replay has not yet diverged at the selected tick, takeover is still blocked — divergence is unpredictable and the state cannot be trusted.

##### App State Transition

The current game lifecycle state machine (`game-loop.md`) defines no `InReplay → InGame` path. Takeover adds a new transition:

```
InReplay → Loading → InGame
```

When the player confirms takeover:
1. The app transitions from `InReplay` to `Loading` (same `Loading` state used by `InMenus → Loading`)
2. During `Loading`: snapshot is restored, players/AI assigned, `LocalNetwork` adapter initialized (Phase 3 single-player; `EmbeddedRelayNetwork` or `RelayLockstepNetwork` in Phase 5 multiplayer takeover), new `.icrep` recording starts
3. `Loading → InGame`: game begins in paused state, identical to a normal match start

This reuses the existing `Loading` state — no new states are added to the state machine. `game-loop.md` must be updated to document this additional entry edge into `Loading`.

##### UX Flow

1. Player opens replay in the replay viewer (existing UX)
2. Player scrubs to desired tick T using existing timeline/transport controls
3. Player clicks **"Take Command"** button (visible in replay viewer toolbar)
4. **Takeover dialog** appears:
   - Faction selector — choose which player to control (with faction icon, color, current army preview)
   - AI assignment — select AI preset (D043) for each non-human faction
   - Difficulty — optional AI difficulty override
   - "Start from here" button
5. System performs snapshot restore (via `GameRunner::restore_full()`):
   - Find nearest keyframe ≤ T in the keyframe index
   - Restore the keyframe snapshot (full snapshot, or full + intervening deltas)
   - Re-simulate forward from the restored keyframe to exact tick T via `apply_tick()`
   - Assign human control to selected faction, AI control to others
6. App transitions `InReplay → Loading → InGame` (game starts in paused state, giving the player a moment to orient)
7. Player unpauses and plays from tick T onward
8. A new `.icrep` file is recorded with branch provenance in its metadata

##### Branched Replay Provenance

Branch provenance is stored in the **JSON metadata section** of the new `.icrep` file (the canonical extensibility surface per `save-replay-formats.md`). The fixed 108-byte binary header is not modified.

```json
{
  "replay_id": "b7e2f1a9-...",
  "branch_from": {
    "parent_replay_id": "a3f7c2d1-...",
    "parent_replay_file_sha256": "e4d9c8b7...",
    "branch_tick": 14320,
    "branch_state_hash": "a1b2c3d4...",
    "original_player_slot": 1,
    "ai_assignments": [
      { "slot": 0, "preset": "adaptive", "difficulty": "hard" }
    ]
  }
}
```

Field semantics:
- `parent_replay_id` — the `replay_id` from the source replay's metadata (UUID, cross-referenceable)
- `parent_replay_file_sha256` — hex-encoded SHA-256 digest of the parent `.icrep` file for integrity verification. This is a **file content hash**, not a `StateHash` (which is reserved for simulation state per `type-safety.md` § Hash Type Distinction)
- `branch_state_hash` — hex-encoded `StateHash` (full SHA-256) of the simulation state at `branch_tick`, computed via `full_state_hash()` during the takeover restore. **Not directly verifiable from stored replay data** — per-tick replay frames carry only `SyncHash` (u64 truncation per `save-replay-formats.md`), and full `StateHash` appears only in relay signatures at signing cadence (`TickSignature.state_hash`). Verification requires re-simulating the parent replay to `branch_tick` and recomputing `full_state_hash()`. The hash is stored for offline auditing and desync debugging, not fast lookup
- `original_player_slot` — which player slot the human took over
- `ai_assignments` — AI presets assigned to non-human factions at takeover

The replay browser shows branch provenance: "Branched from [parent replay name] at [timestamp]". The `branch_from` field is absent (not `null`) in non-branched replays.

##### Multiplayer Takeover (Phase 5 — requires lobby/relay infrastructure)

> **Phase dependency:** Multiplayer takeover requires the lobby system, relay infrastructure, and game browser delivered in Phase 5 (`08-ROADMAP.md` § Phase 5). Single-player takeover (with AI opponents) ships in Phase 3; multiplayer takeover is deferred to Phase 5.

Multiple players can join the takeover session:

1. Host initiates takeover from replay viewer
2. Takeover dialog becomes a lobby — host shares a join code (D052 pattern)
3. Each joining player selects a faction
4. Remaining factions assigned to AI
5. All players synchronize on the restored snapshot
6. Game proceeds as a normal multiplayer match from the branch point

##### Console Commands (D058)

```
/takeover                           -- open takeover dialog at current replay tick
/takeover <tick> <faction>          -- immediate takeover (scripting/testing)
/takeover list-branches <replay>    -- show all branches from a replay
```

##### Constraints

- **Native replays only:** Takeover is available only for native `.icrep` replays. Imported/foreign replays (D056) are blocked entirely — their simulation may diverge, making restored state unreliable for live gameplay
- **Ranked replays:** Takeover from ranked replays is allowed but the branched game is always unranked
- **Signed replays:** If the source replay is relay-signed (Ed25519), the signature chain verifies integrity up to the branch tick. The branched replay is unsigned in Phase 3 (single-player uses `LocalNetwork`, which has no relay to sign). In Phase 5+ multiplayer takeover, the branched replay starts a new signature chain from the relay hosting the session
- **Campaign replays:** If the replay is from a campaign mission, campaign state (`CampaignState`) is restored — the branched game can be played as a standalone mission but does not affect campaign progress
- **Mod compatibility:** Takeover requires matching engine version and mod fingerprint (same as replay playback)

##### Speculative Branch Preview ("What If?" Ghost Overlay)

Inspired by GGPO rollback netcode's speculative execution. While viewing a replay, the player can invoke a **speculative branch** that fast-forwards the simulation N seconds with current AI orders, showing the outcome as a ghost overlay *without committing*.

**Restricted to replay viewer and single-player vs-AI only.** The speculative preview relies on deterministic AI prediction to simulate forward — this is meaningful only when all non-human factions are AI-controlled. Against human opponents, future orders are unknowable, making the preview meaningless. Additionally, `DeveloperMode`-gated forward simulation is explicitly unavailable in ranked (`D058` § Ranked Mode Restrictions), and exposing speculative previews in competitive play would conflict with ranked tooling constraints.

```
1. Player pauses replay (or pauses single-player vs-AI game) at tick T
2. Selects "What If?" mode
3. Engine snapshots state at T
4. Engine speculatively simulates forward N ticks (default 600 = 30s)
   using current queued orders + AI continuation (AI computes its next
   orders normally; human player's queued orders are held constant)
5. Result rendered as translucent ghost overlay (ghost units, projected
   positions, resource projections)
6. Player reviews the preview, then either:
   a. Commits (take command from tick T with those orders — full takeover flow)
   b. Cancels (discard speculative state, return to tick T)
```

Uses the same snapshot + re-simulate infrastructure as takeover. The delta between two speculative branches is small enough (via `TrackChanges` delta tracking) that the preview is cheap to compute. No new per-tick cost — the speculative sim runs in a scratch buffer that is discarded on cancel.

**Phase 3 scope:** Available in replay viewer and single-player vs-AI only. **Not available in multiplayer or ranked play.**

---

#### Layer 2: Campaign Time Machine (Narrative Weapon)

**Phase 4 — ships with the campaign system.**

The time machine is a **diegetic device within the campaign narrative**. Under designer-controlled conditions, it allows the player to rewind to an earlier mission while retaining controlled state — creating a "what if I had done things differently?" mechanic with narrative consequences.

##### Design Philosophy

Lessons from the best time-travel games:

1. **Limited, not infinite** (Prince of Persia) — scarcity creates meaningful decisions
2. **Knowledge is the best carryover** (Outer Wilds) — explored terrain (shroud removal) and intel carry back; raw military power does not
3. **Butterfly effects** (Chrono Trigger) — replaying with foreknowledge changes the world's response
4. **Empowerment, not punishment** (Deathloop) — the player should feel clever for using the machine, not punished for needing it
5. **Narrative integration** (C&C Red Alert) — the time machine is part of the story, not a meta-game escape hatch

##### Campaign YAML Schema Extension (Authored Config)

Following D021's pattern: YAML defines authored config (structure, rules, thresholds); `CampaignState` holds runtime progress. Campaign rewinds target **mission-start checkpoints only**.

**Restore mechanism:** Campaign rewinds use **rule-based reconstruction**, not SimSnapshot restore. Rewinding to a mission means:

1. Load the target mission's map and initial conditions from the campaign YAML (same as starting a fresh mission)
2. **Detach** the current `TimeMachineState` from `CampaignState` and hold it aside (meta-progress is never rewound)
3. **Overwrite** `CampaignState`'s gameplay fields from the target `CampaignProgressCheckpoint.progress` (roster, flags, stats, equipment — the stripped snapshot that excludes `TimeMachineState`)
4. **Reattach** the held-aside `TimeMachineState` with: `uses_consumed` incremented, `current_branch` incremented (monotonic, never reused), new `RewindRecord` appended (with `resulting_branch = current_branch`), and new `BranchNode` added to `branch_tree` (with `parent_branch = previous branch index`)
5. Apply the time-machine carryover rules on top (veterancy, exploration, intel flags per `carryover_rules`)
6. Apply butterfly effects (AI modifiers, spawn changes, flag overrides)
7. Start the mission normally via the existing campaign mission-start flow

This preserves the "limited, not infinite" invariant: `uses_consumed`, `rewind_history`, `branch_tree`, and `checkpoints` are never restored to earlier values — they accumulate monotonically across all jumps. Only gameplay progress (what the player *had* at that mission start) is rewound.

No mission-start `SimSnapshot` retention is needed — the map + rules + `CampaignProgressSnapshot` fully determine the initial state.

```yaml
campaign:
  id: allied_campaign
  start_mission: allied_01

  # Authored config — immutable during play.
  # Runtime progress (uses consumed, branch history) lives in CampaignState.
  time_machine:
    enabled: true
    uses_total: 3                          # max uses across the campaign (runtime tracks consumed count)
    unlock_mission: allied_05              # mission where the machine becomes available
    narrative_name: "Einstein's Temporal Prototype" # rebuilt from his 1946 Trinity research

    # What state carries back through time.
    # Field names align with D041 vocabulary: "exploration" = shroud removal
    # (is_explored), NOT live fog-of-war visibility (is_visible).
    carryover_rules:
      veterancy: true          # unit experience carries back
      unit_roster: false       # army does NOT carry back (knowledge > power)
      exploration: true        # shroud/explored cells carry back — the player remembers the terrain
                               # (D041 § FogProvider: is_explored(), not is_visible())
      intel_flag_keys:          # authored allowlist: which flag keys from D021's
        - radar_codes           # generic flags: HashMap<String, Value> are considered
        - base_layout_north     # "intel" and carry back through time. All other flags
        - enemy_patrol_routes   # are treated as story flags and reset on rewind.
      story_flags: false       # non-intel flags reset (the story hasn't happened yet)
      resources: halved        # partial resource carryover (none | halved | full)
      equipment: selective     # only equipment tagged 'temporal_stable' (none | selective | all)
      hero_progression: true   # hero XP/skills persist (the hero remembers)

    # Which missions can be rewound to (mission-start checkpoints only)
    rewind_targets:
      - mission_id: allied_03
        label: "Before the Bridge Assault"
        butterfly_effects:
          - set_flag: { timeline_aware: true }
          - ai_modifier: paranoid           # enemy AI adapts (they sense something changed)
          - spawn_modifier: reinforced      # enemy gets reinforcements (timeline resistance)
      - mission_id: allied_05
        label: "Before the Chronosphere Theft"
        butterfly_effects:
          - set_flag: { timeline_aware: true }
          - trigger_event: temporal_anomaly  # new scripted event in the mission

    # Missions that cannot be reached by time travel
    time_locked: [allied_01, allied_02]    # too far back — narrative justification

    # Consequences of using the machine (indexed by use count from CampaignState)
    usage_effects:
      first_use:
        narrative: "The Chronosphere hums to life. Reality shimmers."
        gameplay: none
      second_use:
        narrative: "Cracks appear in the machine's casing. Chrono Vortices flicker."
        gameplay: chrono_vortex_hazard       # random hazard zones on the map
      third_use:
        narrative: "The machine is failing. This may be the last jump."
        gameplay: [chrono_vortex_hazard, temporal_instability]  # units randomly phase
```

##### CampaignState Additions (Runtime Progress)

The time-machine runtime state is split into two distinct structures to avoid a recursive data model and to ensure meta-progress (rewind charges, branch history) survives jumps:

1. **`TimeMachineState`** — meta-progress that is **never rewound**. Lives on `CampaignState` but is explicitly excluded from checkpoint snapshots and preserved across rewinds. Contains: uses consumed, rewind history, branch tree, and checkpoint storage.
2. **`CampaignProgressSnapshot`** — a stripped version of `CampaignState` that captures *only* campaign gameplay progress (roster, flags, stats, equipment, hero profiles). Does **not** contain `TimeMachineState`. This is what checkpoints store and what gets restored on rewind.

This separation ensures:
- No recursive data model (`CampaignProgressSnapshot` cannot contain `TimeMachineState`, which cannot contain `CampaignProgressSnapshot` — the cycle is broken)
- The "limited, not infinite" rule is enforced — `uses_consumed` is never restored to an earlier value
- Branch history accumulates monotonically across all rewinds

```rust
/// Time-machine meta-progress — part of CampaignState but NEVER rewound.
/// Preserved across all rewinds. Excluded from CampaignProgressSnapshot.
/// Authored config (uses_total, carryover_rules, rewind_targets) lives in campaign YAML.
#[derive(Serialize, Deserialize, Clone)]
pub struct TimeMachineState {
    /// How many rewind charges have been consumed (uses_total - uses_consumed = remaining).
    /// NEVER restored from a checkpoint — monotonically increasing.
    pub uses_consumed: u32,

    /// History of all rewinds performed, in order.
    /// NEVER restored — accumulates across all jumps.
    pub rewind_history: Vec<RewindRecord>,

    /// The current timeline branch index (0 = original, incremented on each rewind).
    pub current_branch: u32,

    /// Full branch tree for timeline visualization.
    /// Each entry records a branch point and the checkpoint it restored.
    pub branch_tree: Vec<BranchNode>,

    /// Saved campaign-progress snapshots at each mission start, keyed by CheckpointId.
    /// Stores CampaignProgressSnapshot (stripped type), NOT full CampaignState.
    /// Retained for all missions in YAML rewind_targets.
    pub checkpoints: HashMap<CheckpointId, CampaignProgressCheckpoint>,

    /// Cross-branch campaign metrics — monotonically accumulated, never rewound.
    /// These replace CampaignProgressSummary fields that would otherwise regress
    /// when CampaignProgressSnapshot is restored from a checkpoint (which resets
    /// completed_missions and path_taken to their earlier values).
    /// D021's unique_missions_completed, discovered_missions, and best_path_depth
    /// (campaigns.md § CampaignProgressSummary) must be derived from these
    /// cross-branch accumulators, not from the restorable CampaignProgressSnapshot.
    pub cross_branch_stats: CrossBranchStats,
}

/// Cumulative campaign metrics across ALL branches and rewinds.
/// Part of TimeMachineState — never restored from checkpoints.
/// D021's CampaignProgressSummary should derive its values from these
/// when time_machine.enabled is true, rather than from CampaignProgressSnapshot
/// fields (which reset on rewind).
#[derive(Serialize, Deserialize, Clone)]
pub struct CrossBranchStats {
    /// All unique mission IDs completed across any branch (union set).
    pub unique_missions_completed: HashSet<MissionId>,
    /// All mission IDs discovered/encountered across any branch (union set).
    pub discovered_missions: HashSet<MissionId>,
    /// Farthest path depth reached across any branch.
    pub best_path_depth: u32,
    /// All endings unlocked across any branch.
    pub endings_unlocked: HashSet<String>,
}

/// Stripped campaign progress — everything EXCEPT TimeMachineState.
/// This is what checkpoints store and what gets restored on rewind.
/// Breaking the recursive cycle: CampaignState → TimeMachineState →
/// CampaignProgressCheckpoint → CampaignProgressSnapshot (no TimeMachineState).
#[derive(Serialize, Deserialize, Clone)]
pub struct CampaignProgressSnapshot {
    pub campaign_id: CampaignId,
    pub current_mission: MissionId,
    pub completed_missions: Vec<CompletedMission>,
    pub unit_roster: Vec<RosterUnit>,
    pub equipment_pool: Vec<EquipmentId>,
    pub hero_profiles: HashMap<String, HeroProfileState>,
    pub resources: i64,
    pub flags: HashMap<String, Value>,    // story + intel flags
    pub stats: CampaignStats,
    pub path_taken: Vec<MissionId>,
    pub world_map: Option<WorldMapState>,
    // NOTE: No Option<TimeMachineState> here — that's the whole point.
}

/// Unique identifier for a mission-start checkpoint.
/// Generated at each mission start. Disambiguates the same mission_id
/// visited on different branches or multiple times.
#[derive(Serialize, Deserialize, Clone, PartialEq, Eq, Hash)]
pub struct CheckpointId(pub String);  // UUID or "branch_{n}_{mission_id}_{occurrence}"

/// A saved campaign-progress snapshot at mission start.
/// Contains CampaignProgressSnapshot (stripped), NOT full CampaignState.
#[derive(Serialize, Deserialize, Clone)]
pub struct CampaignProgressCheckpoint {
    pub checkpoint_id: CheckpointId,
    pub mission_id: MissionId,
    pub branch_index: u32,
    pub occurrence: u32,                          // how many times this mission has been started
    pub progress: CampaignProgressSnapshot,       // stripped — no TimeMachineState
}

/// A single rewind event recorded in meta-progress.
#[derive(Serialize, Deserialize, Clone)]
pub struct RewindRecord {
    /// Which mission the player rewound FROM (the "present" at time of rewind).
    pub source_mission: MissionId,
    /// Which checkpoint was restored.
    pub target_checkpoint: CheckpointId,
    /// Snapshot of carryover that was applied (for debrief/visualization).
    pub carryover_summary: CarryoverSummary,
    /// Which butterfly effects were triggered.
    pub butterfly_effects_applied: Vec<String>,
    /// The branch index this rewind created.
    pub resulting_branch: u32,
}

/// A node in the campaign timeline tree (for visualization).
///
/// Branch ID allocation rule: branch 0 is the original playthrough.
/// Each rewind increments TimeMachineState.current_branch and creates
/// a new BranchNode with branch_index = current_branch. Branch IDs
/// are monotonically increasing and never reused. The rewind flow
/// (detach/reattach pattern) sets current_branch = previous + 1
/// atomically with uses_consumed++.
#[derive(Serialize, Deserialize, Clone)]
pub struct BranchNode {
    pub branch_index: u32,
    pub parent_branch: Option<u32>,       // None for branch 0 (original)
    pub branch_point_mission: MissionId,  // where the branch diverged
    /// Missions completed on this branch, with per-visit context.
    /// Uses MissionVisit instead of bare MissionId to disambiguate
    /// the same mission replayed with different outcomes.
    pub missions_in_branch: Vec<MissionVisit>,
}

/// A single mission visit within a branch — tracks enough context
/// to distinguish repeated plays of the same mission with different outcomes.
#[derive(Serialize, Deserialize, Clone)]
pub struct MissionVisit {
    pub mission_id: MissionId,
    pub checkpoint_id: CheckpointId,       // the checkpoint created at this mission's start
    pub outcome: Option<String>,           // named outcome (D021 MissionOutcome) if completed; None if in-progress
    pub occurrence: u32,                   // 0-indexed: how many times this mission has been started on this branch
}

/// Summary of what state was carried back (for UI display, not the actual state).
#[derive(Serialize, Deserialize, Clone)]
pub struct CarryoverSummary {
    pub veterancy_units_carried: u32,
    pub roster_units_carried: u32,
    pub exploration_cells_carried: u32,  // shroud cells (D041 is_explored), not fog
    pub intel_flag_keys_carried: Vec<String>,
    pub resource_amount: i64,
    pub equipment_items_carried: u32,
    pub hero_profiles_carried: Vec<String>,
}
```

**Ownership model:** `CampaignState` has an `Option<TimeMachineState>` field. On rewind:
1. The current `TimeMachineState` is **detached** from `CampaignState` and held aside
2. `CampaignState`'s gameplay fields (roster, flags, stats, etc.) are **overwritten** from `CampaignProgressCheckpoint.progress`
3. The held-aside `TimeMachineState` is **reattached** with `uses_consumed` incremented
4. Result: gameplay progress is rewound, but meta-progress (charges used, branch history, all checkpoints) is preserved

##### Lua API Extension — `TimeMachine` Global

```lua
-- Query time machine state
local tm = TimeMachine.get_status()
-- Returns: { enabled = true, uses_remaining = 2, available_targets = {...}, ... }

-- Check if time machine is available
if TimeMachine.is_available() then
  -- Show time machine activation UI
  TimeMachine.show_activation_ui()
end

-- Programmatic activation (for scripted sequences)
-- Accepts checkpoint_id (unambiguous) OR target_mission (convenience).
-- If target_mission is used and multiple checkpoints exist for that mission,
-- the most recent checkpoint on the current branch is selected.
-- Use checkpoint_id when disambiguation is needed.
TimeMachine.activate({
  checkpoint_id = "branch_0_allied_03_0",  -- exact checkpoint (preferred)
  -- OR: target_mission = "allied_03",     -- convenience shorthand (selects latest)
  cinematic = "chrono_jump_cutscene",      -- play cinematic before rewind
  on_complete = function(result)
    -- result.success: boolean
    -- result.checkpoint_id: string
    -- result.target_mission: string
    -- result.carryover_summary: table (what was carried back)
  end
})

-- Set custom carryover for this specific jump (roster selectors, not live UnitIds)
TimeMachine.set_carryover_override({
  carry_roster_entries = { "tanya", "spy_01" },  -- matches RosterUnit.unit_type or .name
  bonus_intel = { "radar_codes", "base_layout_north" }
})

-- React to time machine usage in mission scripts
Events.on("time_machine_arrival", function(context)
  -- context.source_mission: where the player came from
  -- context.timeline_index: how many times this mission has been visited
  -- context.carryover: what state arrived
  if context.timeline_index > 1 then
    -- Player has been here before — adapt!
    AI.set_paranoia_level(context.timeline_index)
    Trigger.spawn_reinforcements("enemy_temporal_response")
  end
end)

-- Track timeline state
local timeline = TimeMachine.get_timeline()
-- Returns: { branches = [...], current_branch = 2, total_jumps = 1 }
```

##### Timeline Visualization

A **campaign timeline view** shows the player's journey through missions, with branches created by time-machine usage:

```
  allied_01 ─── allied_02 ─── allied_03 ─── allied_04 ─── allied_05 (DEFEAT)
                                  │
                                  └──── allied_03' ─── allied_04' ─── allied_05' (VICTORY)
                                        [Timeline 2]    [knowledge    [different
                                         rewind here]    carried]      outcome]
```

This view is accessible from the campaign menu and the intermission screen. Completed branches are greyed but visible — the player can see the "roads not taken."

##### Console Commands (D058)

```
/timemachine status                          -- show uses remaining, available targets, current branch
/timemachine activate <checkpoint_id>        -- activate by exact checkpoint ID (unambiguous)
/timemachine activate --mission <mission_id> -- activate by mission ID (selects latest checkpoint; errors if ambiguous)
/timemachine timeline                        -- show branch tree (text-based timeline visualization)
/timemachine checkpoints                     -- list all stored checkpoints with IDs, missions, and branches
```

These commands are available only during campaign play when `time_machine.enabled: true`. `/timemachine activate` triggers the same flow as `TimeMachine.activate()` in Lua, including cinematic and carryover application. When using `--mission`, if multiple checkpoints exist for that mission ID (from different branches or repeated visits), the command errors with a disambiguation prompt listing available checkpoint IDs.

##### Narrative Integration Ideas (Red Alert Context)

- **Allied campaign:** Einstein reveals his original 1946 time travel research — the science behind the device he used to remove Hitler in Trinity, NM. He rebuilt a limited prototype from his original notes that can transmit tactical intel back in time, but each transmission destabilizes the device
- **Soviet campaign:** Soviet scientists reverse-engineer captured Chronosphere technology. The player uses it to undo catastrophic defeats — but the timeline fights back (butterfly effects)
- **Final act:** Both sides have temporal capability, creating a "temporal arms race" where the campaign graph becomes a tree of competing timelines. The player must decide which timeline to "collapse" into reality

##### Campaign Mission Archetypes (Cross-Domain Inspired)

The time machine enables mission design patterns impossible in a linear campaign. Each archetype draws from the cross-domain inspirations in the Prior Art section.

**Archetype 1: Convergence Puzzle** *(Steins;Gate attractor fields + Novikov self-consistency)*

Certain mission outcomes are *convergent* — they happen regardless of what the player does. The bridge will be destroyed. The commander will be captured. The convoy will be lost. These "locked" outcomes resist direct intervention.

The puzzle: the player must rewind to *earlier* missions and accumulate enough divergence (different flag states, different surviving units, different intel) to shift the campaign to a different "attractor field" where the locked outcome is no longer convergent. This requires multiple rewinds and strategic experimentation across the campaign graph.

```yaml
# Convergence definition in campaign YAML
missions:
  allied_06:
    convergence:
      - event: bridge_destroyed
        locked_until_divergence: 3    # requires 3+ flag differences from original timeline
        unlock_flags: [explosives_intercepted, engineer_survived, radar_sabotaged]
        unlocked_outcome: bridge_intact  # new outcome edge, only available after convergence breaks
```

```lua
-- Mission script checks convergence
Events.on("bridge_assault_begins", function()
  if Campaign.convergence_locked("bridge_destroyed") then
    -- No matter what the player does, the bridge will be destroyed
    -- But HOW it's destroyed can vary (enemy demolishes vs. friendly fire vs. airstrike)
    Trigger.schedule_destruction("bridge", { delay = 120, method = "enemy_demo_charge" })
  else
    -- Convergence broken! The bridge CAN be saved
    -- Player must actively defend it (new gameplay, not just a cutscene)
  end
end)
```

**Archetype 2: Information Lock** *(Zero Escape: Virtue's Last Reward FLOW chart)*

A mission that is *literally impassable* without intel obtained from a different timeline branch. The player reaches a locked door, an encrypted comm channel, a minefield with no safe path — and the solution exists only in a mission they haven't played yet (or played on a different branch).

The time machine becomes a *required tool*, not a safety net. The player must:
1. Recognize they're stuck (the game hints, never blocks silently)
2. Rewind to a branch point
3. Play a different branch to discover the password / patrol routes / safe path
4. Rewind again, returning to the locked mission with the carried-back intel

```yaml
# Information lock in campaign YAML
missions:
  allied_07:
    information_locks:
      - lock_id: minefield_safe_path
        description: "Dense minefield blocks the northern approach"
        required_intel_flag: minefield_map_obtained
        hint_text: "A captured Soviet engineer might know the safe path..."
        obtainable_in: [allied_04b]  # only available on the alternate branch of mission 04
```

**Archetype 3: Causal Loop** *(Dark bootstrap paradox + closed timelike curves)*

A mission where the player must *create the conditions for their own past success*. The mission starts with unexplained help — reinforcements arrive from nowhere, enemy defenses are mysteriously weakened, a supply cache appears at a critical moment.

At the end of the mission, the player discovers they have the means to send that help back in time. They must choose *exactly* the right units/resources to send, matching what they received at the start. If the loop doesn't close (wrong units sent, or player keeps the resources), the mission replays *without* the mysterious help — harder, but still winnable on an alternate path.

```lua
-- Causal loop mission script
Events.on("mission_start", function()
  local loop = Campaign.get_flag("temporal_loop_07")
  if loop and loop.closed then
    -- Loop is closed from a future playthrough — reinforcements arrive
    local units = loop.units_sent
    for _, unit_type in ipairs(units) do
      Trigger.spawn_reinforcement(unit_type, "temporal_arrival_zone", {
        effect = "chrono_shimmer",
        label = "Temporal Reinforcement"
      })
    end
    UI.show_timeline_effect("Reinforcements have arrived from... the future?")
  else
    -- First playthrough or loop not closed — no help
    UI.show_hint("You're on your own. For now.")
  end
end)

Events.on("mission_end_victory", function()
  -- Player has captured the Chronosphere. Offer the loop-closing choice:
  TimeMachine.offer_causal_loop({
    prompt = "Send reinforcements to your past self?",
    unit_selector = true,   -- player picks which units to send back
    target_mission = "allied_07",  -- this mission
    on_send = function(units_sent)
      Campaign.set_flag("temporal_loop_07", {
        closed = true,
        units_sent = units_sent
      })
    end,
    on_decline = function()
      -- Player keeps the units. Loop stays open.
      -- Next playthrough of this mission will have no temporal help.
    end
  })
end)
```

**Archetype 4: Dual-State Battlefield** *(Titanfall 2 "Effect and Cause")*

A single mission where the battlefield exists in two temporal states: **past** (intact, resource-rich, enemy-alert) and **present** (ruined, resource-scarce, enemy-weakened). The player can shift their forces between states using a captured Chronosphere at their base.

Implementable within the single-sim architecture: the map contains both states as terrain layers toggled by a game flag. Units in the "wrong" temporal state are invisible/intangible to enemies in the other state. The Chronosphere shift moves selected units between layers.

```yaml
# Dual-state mission definition
mission:
  id: allied_09_dual_state
  temporal_states:
    past:
      terrain_layer: terrain_past       # intact buildings, bridges, forests
      resources: abundant               # ore fields are rich
      enemy_posture: full_alert         # full garrison, active patrols
      fog: full                         # no prior exploration
    present:
      terrain_layer: terrain_present    # ruins, craters, dead forests
      resources: scarce                 # depleted ore fields
      enemy_posture: skeleton_crew      # reduced garrison, damaged defenses
      fog: explored                     # terrain already scouted from past visits

  chrono_shift:
    cooldown_ticks: 600                 # 30 seconds between shifts
    max_units_per_shift: 10             # can't move entire army at once
    shift_effect: chrono_shimmer        # visual effect on shifting units
```

**Archetype 5: Retrocausal Modification** *(Retrocausality + Chrono Trigger)*

A later mission's outcome *retroactively changes* the conditions of an earlier mission. The player completes mission 08, where they destroy a Soviet weapons lab. On their next playthrough of mission 05 (via time machine), the enemy no longer has the advanced weapons that were produced in that lab — because in the future, the lab was destroyed before it could produce them.

This is the reverse of Chrono Trigger's "plant seed → find forest" pattern: "destroy the factory → the weapons were never built." Campaign flags set in later missions modify the `conditions` and `spawn_modifier` fields of earlier missions' YAML definitions.

```lua
-- Mission 08 script: destroying the weapons lab
Events.on("weapons_lab_destroyed", function()
  Campaign.set_flag("weapons_lab_destroyed_in_future", true)
end)

-- Mission 05 setup script: reads retrocausal flag
Events.on("mission_start", function()
  if Campaign.get_flag("weapons_lab_destroyed_in_future") then
    -- The lab was destroyed in a future mission — enemy tech is downgraded
    AI.override_unit_type("heavy_tank", "medium_tank")  -- no heavy tanks available
    AI.disable_superweapon("tesla_coil_mk2")
    UI.show_timeline_effect("Soviet advanced weapons program never materialized — the lab was destroyed before it could produce them.")
  end
end)
```

---

#### Layer 3: Multiplayer Time-Machine Game Modes

**Phase 5 — ships with multiplayer mode expansion.**

New custom game modes that use the time machine as a contested battlefield objective or tactical ability.

##### Mode A: Chrono Capture (King of the Hill Variant)

A **Chronosphere device** sits at a contested point on the map. Factions race to capture and activate it for powerful effects.

```yaml
# Game mode definition
game_mode:
  id: chrono_capture
  name: "Chrono Capture"
  description: "Capture and activate the Chronosphere for devastating temporal effects."

  chrono_device:
    capture_radius: 5          # cells
    charge_time_ticks: 1200    # 60 seconds at 20 tps to activate
    charge_requires: uncontested  # must hold without enemy units in radius
    respawn_time_ticks: 3600   # 3 minutes after activation before it can be used again

  activation_effects:          # the capturing player chooses one
    - id: tactical_rewind
      name: "Tactical Rewind"
      description: "Undo the last 60 seconds of your unit losses. Destroyed units respawn at death locations."
      effect: restore_destroyed_units
      window_ticks: 1200       # 60-second lookback

    - id: timeline_peek
      name: "Timeline Peek"
      description: "Reveal the enemy's production queue and army composition for 30 seconds."
      effect: reveal_production
      duration_ticks: 600

    - id: chrono_shift
      name: "Chrono Shift"
      description: "Teleport your entire visible army to any revealed map location."
      effect: army_teleport

    - id: economic_rewind
      name: "Economic Rewind"
      description: "Reset the opponent's economy to its state 2 minutes ago."
      effect: rewind_opponent_economy
      window_ticks: 2400

  contested_effects:
    chrono_vortex:
      trigger: "charge interrupted at >50%"
      effect: "hazard zone spawns at device location (20-cell radius, 30-second duration)"
      damage: "damages all units in zone regardless of faction"

  escalation:
    enabled: true
    description: "Each successive activation by the same faction is stronger but takes longer to charge."
    charge_multiplier_per_use: 1.5   # 60s → 90s → 135s
    effect_multiplier_per_use: 1.3   # rewind window grows, reveal duration grows, etc.
```

> **Design dependency — partial temporal effects:** `restore_destroyed_units` and `rewind_opponent_economy` are **not** whole-state snapshot restores — they are selective, per-player temporal rewrites within a live match. The existing restore machinery (`GameRunner::restore_full()`, `SimSnapshot`/`DeltaSnapshot`) operates on the entire simulation state and is designed for save-load and replay seeking, not partial rollback of individual players.
>
> These effects require a **per-player history buffer** — a rolling window of recent unit destruction events and economy snapshots (resource levels, harvester counts, building completions) retained for the lookback window (default 60–120 seconds). The buffer must define:
> - **Conflict resolution:** What happens when a restored unit's death position is now occupied? (Spawn at nearest valid position)
> - **Cascading effects:** If a restored unit was carrying resources, are those resources also restored? (Yes — the unit snapshot includes carried state)
> - **Opponent interaction:** `rewind_opponent_economy` must handle buildings built with the "rewound" resources (buildings remain; only liquid resource balance is reset)
>
> This buffer is a new system (~300 lines in `ic-game`) not covered by existing snapshot infrastructure. It is scoped to Chrono Capture mode only and does not affect the core sim's determinism (the buffer is observation-only; the rewind effects are applied as standard game-logic orders).
>
> **Reconnect semantics:** The per-player history buffer is **not part of `SimSnapshot`** and is not restored by `GameRunner::restore_full()` (which restores sim core, campaign state, and script state only — `02-ARCHITECTURE.md` § ic-game Integration). On reconnect, the buffer starts empty. Temporal rewind abilities (`restore_destroyed_units`, `rewind_opponent_economy`) are **unavailable until the buffer re-accumulates** enough history (60–120 seconds of live play after reconnect). This is the simplest correct behavior — no snapshot extension needed, and the gameplay impact is minor (a reconnecting player temporarily loses access to one optional ability).

##### Mode B: Time Race

Two Chronosphere devices, one near each faction's base. Players must **defend their own** while trying to **capture and destroy the opponent's**.

```yaml
game_mode:
  id: time_race
  name: "Time Race"
  description: "Each faction has a Chronosphere. Activate yours while preventing the enemy from activating theirs."

  victory_condition: "First to 3 activations OR destroy opponent's Chronosphere"

  devices:
    - location: near_player_1_base
      owner: player_1
      health: 5000             # can be attacked and destroyed
      repair_rate: 10          # self-repairs slowly
    - location: near_player_2_base
      owner: player_2
      health: 5000
      repair_rate: 10

  activation_effect: "all enemy units in a 30-cell radius around YOUR device are chronoshifted to a random map location (scattered, disoriented)"
```

##### Mode C: Timeline Duel (Background Headless Sim Pattern — M11/P007)

> **Architecturally feasible, pending M11 client driver decision.** The background headless sim pattern (running a second `Simulation` instance alongside the primary `GameLoop`) is technically sound — `ic-sim` is a library designed for headless multi-instance embedding, and bot harnesses already create multiple instances per process. D038's rejection targets "concurrent nested maps in the same mission timeline" (sub-scenario portals), not independent background sims.
>
> However, `game-loop.md` explicitly gates non-standard client loop variants to **M11 (pending decision P007)**: *"Deferred non-lockstep architectures require a different client-side loop variant... This is an M11 design concern."* The background headless sim pattern is a new client architecture variant — it must be formalized as part of P007, not declared unilaterally in D078.
>
> **Status:** Feasible. Not blocked by architectural impossibility. Requires P007 decision (M11/Phase 7) to formalize the background sim client driver pattern. D078 defines the game design; P007 will define the client architecture that enables it.

**Proposed architecture (subject to P007 approval):**

**Architecture:**

```
┌─────────────────────────────────────────────────────┐
│ Client Process                                       │
│                                                      │
│  ┌──────────────────────────────────────────┐        │
│  │ GameLoop<RelayLockstepNetwork>           │        │
│  │   sim: Simulation  ← player's timeline   │        │
│  │   renderer: Renderer  ← renders primary  │        │
│  │   network: N  ← relay orders for this TL │        │
│  └──────────────────────────────────────────┘        │
│                                                      │
│  ┌──────────────────────────────────────────┐        │
│  │ Background thread (no GameLoop)           │        │
│  │   shadow_sim: Simulation  ← opponent TL  │        │
│  │   No renderer (headless)                  │        │
│  │   Fed orders from relay (opponent's TL)   │        │
│  │   Produces state snapshots for overlay    │        │
│  └──────────────────────────────────────────┘        │
│                                                      │
│  Primary renderer reads shadow_sim snapshots         │
│  via cross-thread channel for ghost overlay           │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Relay Server                                         │
│   RelayCore (Timeline A) ← routes Player A's orders  │
│   RelayCore (Timeline B) ← routes Player B's orders  │
│   Timewave coordinator ← triggers merge events       │
└─────────────────────────────────────────────────────┘
```

**How it works:**

1. Both players start from the same initial state (same map, same seed)
2. The relay runs **two `RelayCore` instances** — one per timeline. Each client's primary `GameLoop` connects to their own timeline's relay. Each client's background `Simulation` is fed the opponent's timeline orders from the second relay
3. Each client sees their own timeline rendered normally. The opponent's timeline state is overlaid as a translucent ghost layer (using `CosmeticRng`-isolated rendering — structurally cannot contaminate the primary sim's determinism)
4. **Timewaves:** At merge intervals, the relay broadcasts a `TimewaveSignal` order to both timeline relays (the relay is a stateless order router — it never runs the sim and cannot evaluate game state per D074). **Merge is computed client-side, deterministically:** each client has identical state for both timelines (same lockstep inputs → same state), so both clients independently evaluate the merge rule (e.g., compare army value in the merge zone between their two local sims) and arrive at the same deterministic result. The merge is applied as a `TimewaveMergeOrder` injected into both sims simultaneously. No relay state evaluation needed — the relay only provides the timing signal

**Performance budget:** One headless `Simulation` adds ~2-4ms per tick on modern hardware (based on the same performance profile used for bot harnesses and AI training in `10-PERFORMANCE.md`). At 20 tps, this is <10% of the frame budget. Memory: ~20-40MB for a typical match state. Acceptable for a Phase 7 experimental mode.

**Determinism:** Both sims on each client must produce identical state to the corresponding sims on the other client. This is guaranteed by the same lockstep invariant that governs normal multiplayer — same initial state + same ordered inputs = identical outputs. The relay's sub-tick ordering ensures both clients process orders in the same sequence for each timeline.

```yaml
game_mode:
  id: timeline_duel
  name: "Timeline Duel"
  description: "Both players exist in parallel timelines. Timewaves periodically merge reality."

  timewave:
    interval_ticks: 2400       # every 2 minutes
    merge_rule: "strongest"    # the player with more army value in the merge zone gets to 'overwrite' that area
    merge_zone: "random 20x20 area announced 30 seconds before timewave"

  victory: "Standard (destroy opponent's base) — but the base exists in both timelines. Must win in YOUR timeline."

  background_sim:
    headless: true             # no renderer on the shadow sim
    overlay_mode: translucent  # ghost overlay of opponent's timeline
    snapshot_cadence: 10       # ticks between overlay refreshes (200ms at 20 tps)
```

##### Mode D: Temporal Manipulation Support Power (Singularity TMD-Inspired)

Available in any multiplayer game mode as an optional support power unlocked by controlling a Chronosphere structure. Inspired by Singularity's Time Manipulation Device — per-object time manipulation as a tactical tool.

| Ability                  | Effect                                                                                                                               | Cooldown | Target                |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ | -------- | --------------------- |
| **Age Structure**        | Target enemy building rapidly ages — health reduced by 50%, production speed halved, armor degraded. Visually crumbles/rusts         | 90s      | Single enemy building |
| **Restore Structure**    | Target friendly ruin is restored to working condition (must be a recently destroyed building, within 60s of destruction)             | 120s     | Single friendly ruin  |
| **Temporal Decay Field** | Area-of-effect zone (15-cell radius, 20s duration) where all enemy units lose 1 veterancy level and take 15% health damage over time | 180s     | Ground target         |
| **Chrono Stasis**        | Single enemy unit frozen in time for 10 seconds — invulnerable but unable to act, blocking pathing                                   | 45s      | Single enemy unit     |

These are implemented as standard support powers (existing `SupportPower` component from `gameplay-systems.md`) with temporal visual effects (chrono shimmer, aging particles, time-freeze crystal). They require controlling a Chronosphere structure on the map — losing it disables the abilities.

##### Ranked Eligibility

- **Chrono Capture:** Eligible for ranked after community playtesting (seasonal D055 config)
- **Time Race:** Eligible for ranked (symmetric, skill-expressive)
- **Temporal Support Powers:** Eligible for ranked when enabled via map/mode config (symmetric access)
- **Timeline Duel:** Unranked only (experimental, uses background headless sim pattern — needs extensive playtesting)

---

#### Layer 4: Multiplayer Temporal Campaigns (Advanced)

**Phase 7 — ships after campaign + multiplayer time modes are proven.**

##### Concept A: Temporal Pincer Co-op Campaign

Inspired by Tenet's temporal pincer movement — "one team moves forward, one moves backward, both attack the same objective from opposite temporal directions."

2-player co-op where players operate across timelines. Each player runs a single simulation (one mission each) — no concurrent multi-sim required. Effects propagate at **mission boundaries only**, consistent with D021's mission-outcome → next-mission flow and the single-sim-per-client constraint in `game-loop.md`.

**Basic flow (mission-boundary effects):**
- **Commander** plays in the "present timeline" — a standard campaign mission
- **Temporal Operative** is sent back to a "past mission" via the time machine
- The Operative completes their past mission first; the outcome sets campaign flags. The Commander's next mission (or a mission restart with updated initial conditions) reflects those flags
- Both players see a shared **timeline visualization** showing cause-and-effect chains

**Temporal Pincer variant (replay-as-ghost-army):**

The pincer takes the co-op concept further using IC's replay infrastructure:

1. **Forward pass:** Player A plays a mission and loses (or wins with heavy losses). The `.icrep` replay is recorded
2. **Backward pass:** Player B receives Player A's replay. Player B plays the *same mission*, but Player A's recorded actions play out simultaneously as **ghost units** — translucent allied forces executing Player A's original orders
3. **Pincer effect:** Player A's "failed" timeline becomes Player B's support fire. The replay ghost army draws enemy attention, creates diversions, and softens defenses — exactly as it happened in the original timeline
4. **Closure:** Player B's victory (aided by the ghost army) retroactively "justifies" Player A's sacrifice. The campaign narrative frames it as: "Your future self sent help back through time"

```lua
-- Temporal Pincer mission setup
Events.on("mission_start", function()
  local pincer = Campaign.get_flag("temporal_pincer_data")
  if pincer then
    -- Load the forward-pass replay as a ghost army
    local ghost_replay = Replay.load_by_id(pincer.replay_id)
    Replay.play_as_ghost_army(ghost_replay, {
      faction = pincer.original_faction,
      render_mode = "translucent",          -- ghost shimmer effect
      label = "Temporal Echo",
      controllable = false,                 -- plays autonomously from replay
      takes_damage = true,                  -- can be destroyed (not invincible)
      deals_damage = true,                  -- but deals real damage
      damage_multiplier = 0.7,             -- slightly weaker than real units
    })
    UI.show_timeline_effect("Temporal echoes of a parallel timeline have arrived to support you.")
  end
end)

-- After Player A's forward pass, the campaign stores the replay reference.
-- Uses replay_id (UUID from .icrep metadata), NOT a filesystem path —
-- raw paths break across portable mode, moved data dirs, cloud-restored
-- saves, and other machines (platform-portability.md § Path Rules).
-- The replay catalog (SQLite, D034) resolves replay_id to the current path.
Events.on("mission_end", function(ctx)
  if ctx.is_forward_pass then
    Campaign.set_flag("temporal_pincer_data", {
      replay_id = ctx.replay_id,             -- UUID, not filesystem path
      original_faction = ctx.player_faction,
    })
  end
end)
```

**Architectural note:** This does *not* reuse `ReplayPlayback` as a `NetworkModel` — `GameLoop` owns exactly one `NetworkModel` instance (`network: N` per `game-loop.md`), and the live match already uses `LocalNetwork` or a relay model. Instead, the ghost army is implemented as a **replay-fed ghost entity spawner**: a Lua/engine-level system that reads orders from the stored `.icrep` file and injects them as AI-like commands for specially tagged ghost entities within the single active sim. Ghost entities are standard ECS entities with a `GhostUnit` component (translucent rendering, reduced damage multiplier, non-controllable by the player). The replay reader runs as a background system alongside the primary `NetworkModel`, not as a replacement for it. No multi-sim needed.

> **Known limitation — replay order drift:** Once Player B's actions diverge the battlefield from Player A's recording, the replay-sourced orders operate against a different state than they were recorded in. This is the same class of divergence D056 describes for foreign replay import. Ghost units may attempt to move to occupied positions, attack destroyed targets, or path through changed terrain.
>
> **Mitigation strategy — graceful degradation, not literal replay:**
> - Ghost orders that fail validation (invalid target, blocked path, unaffordable) are **silently dropped** — the ghost unit idles rather than executing an impossible action
> - Ghost units fall back to **waypoint-following** derived from the replay's `UnitPositionSample` events (analysis stream, sampled every ~10 ticks) rather than literal order injection. They move toward the positions they occupied in the original replay, engaging enemies they encounter along the way using standard AI combat logic
> - Ghost units are explicitly **expendable and imperfect** — the narrative framing ("temporal echoes") sets the expectation that they are unreliable reflections, not precise duplicates
> - As drift accumulates, ghost effectiveness naturally degrades — this is a feature, not a bug. The earlier in the mission the ghost army is most helpful (when divergence is lowest), creating a natural tempo curve
>
> This is an acceptable trade-off for a Phase 7 co-op campaign feature. The ghost army provides thematic atmosphere and approximate tactical support, not frame-perfect replay reproduction.
>
> **Save/resume semantics:** The ghost replay reader (cursor position, spawned ghost entity state) is **not part of `SimSnapshot`** — `.icsave` files contain sim core + campaign + script state only (`02-ARCHITECTURE.md` § ic-game Integration, `D016` § multiplayer save). On save/resume, the ghost replay reader restarts from tick 0 of the source replay and fast-forwards to the current mission tick. Ghost entities that were already destroyed in the prior session are re-destroyed during fast-forward. This is imperfect (ghost behavior may differ on replay due to drift) but acceptable for the same "temporal echoes are unreliable" framing. Alternatively, the ghost cursor tick can be stored in script state (which IS saved) via `ScriptState` serialization — a single `u64` field.
>
> **Source replay requirement:** Temporal-pincer source replays **must have the `HAS_EVENTS` flag set** (analysis event stream present). The waypoint fallback depends on `UnitPositionSample` events, which are part of the optional analysis stream (`save-replay-formats.md` § flags). If the source replay lacks `HAS_EVENTS`, the ghost army falls back to **order-only mode** (literal order injection without waypoint fallback) — which degrades faster under drift but still provides early-mission support. The campaign Lua should validate the replay before storing it:
>
> ```lua
> if not Replay.has_events(ctx.replay_id) then
>   UI.show_warning("Source replay has no analysis events. Ghost army support will be limited.")
> end
> ```

**Standard co-op flow (without pincer):**

```lua
-- Co-op temporal campaign Lua example
-- Operative's past mission completes BEFORE Commander's next mission loads.
-- Flags set by the Operative's outcome alter the Commander's mission setup.
Events.on("operative_completed_past_mission", function(ctx)
  if ctx.outcome == "sabotaged_radar" then
    -- Flag is set in CampaignState; Commander's next mission reads it during setup
    Campaign.set_flag("radar_sabotaged_in_past", true)
  end
end)

-- Commander's mission setup script reads the flag
Events.on("mission_start", function()
  if Campaign.get_flag("radar_sabotaged_in_past") then
    Trigger.disable_structure("enemy_radar_tower")
    UI.show_timeline_effect("Temporal Operative sabotaged the radar in 1943. Enemy radar is offline!")
  end
end)
```

##### Concept B: Adversarial Timeline Campaign

Two players (or teams) each have a time machine with limited uses in a PvP campaign:

- When one side uses their time machine, the campaign "rewinds" for both — but the time-traveling side keeps knowledge/state per carryover rules
- Creates a meta-game of timing: use your rewind now on a minor loss, or save it for a catastrophic one?
- Could leverage LLM campaign generation (D016) to dynamically generate the "altered timeline" missions

##### Concept C: Temporal Commander (Background Headless Sim Pattern — M11/P007)

> **Same dependency as Timeline Duel.** Architecturally feasible via background headless sims, pending M11 (P007) client driver decision. The TC's client runs N background `Simulation` instances (one per Field Commander's branch), producing state snapshots for the TC's overlay UI.

Extends D070 asymmetric co-op with a new role:

**Architecture:**

```
┌──────────────────────────────────────────────────────┐
│ Temporal Commander Client                             │
│                                                       │
│  ┌─────────────────────────────────────────┐          │
│  │ GameLoop (lightweight command view)      │          │
│  │   sim: Simulation ← TC's own UI state   │          │
│  │   renderer: Renderer ← aggregate view   │          │
│  └─────────────────────────────────────────┘          │
│                                                       │
│  ┌─────────────────────────────────────────┐          │
│  │ Background threads (N branches)          │          │
│  │   branch_sim[0]: Simulation (headless)   │          │
│  │   branch_sim[1]: Simulation (headless)   │          │
│  │   branch_sim[2]: Simulation (headless)   │          │
│  │   Fed orders from relay per branch       │          │
│  │   Produce state snapshots for TC overlay  │          │
│  └─────────────────────────────────────────┘          │
│                                                       │
│  TC renderer composites all branch snapshots          │
│  into split-screen / tab view / aggregate minimap     │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│ Field Commander Clients (2-3 players)                 │
│   Each runs standard GameLoop with one Simulation     │
│   Connected to their own branch's RelayCore           │
└──────────────────────────────────────────────────────┘
```

- **Temporal Commander** observes multiple timeline branches simultaneously (split-screen or tab view) via background headless sims, and allocates shared resources across them (resource transfers submitted as orders to the appropriate branch's relay)
- **Field Commanders** (2-3 players) each play in their own timeline branch using standard `GameLoop` — they see only their own branch
- The Temporal Commander decides which timeline to "collapse" (make canonical) based on which branch is winning — submitted as a `CollapseTimelineOrder` to all branch relays
- Collapsed timeline's state becomes the starting point for the next campaign mission

**Performance:** N headless sims at ~2-4ms/tick each. For 3 branches at 20 tps: ~6-12ms per tick on a background thread, well within budget on a multi-core system. Memory: ~60-120MB total for 3 branch states.

---

### Implementation Architecture

#### Snapshot Branch Operation

The core primitive shared across all layers:

```rust
/// Branch from a replay or saved state into live gameplay.
pub struct ReplayTakeover {
    /// Source of the state to branch from
    pub source: TakeoverSource,
    /// Tick at which to branch
    pub branch_tick: SimTick,
    /// Player assignments (human or AI)
    pub player_assignments: Vec<PlayerAssignment>,
    /// Campaign state overrides (for time-machine carryover rules)
    pub state_overrides: Option<TimeMachineCarryover>,
}

pub enum TakeoverSource {
    /// Branch from a replay file. Uses replay_id (UUID) resolved via the
    /// replay catalog (SQLite, D034), not a raw filesystem path — raw paths
    /// break across portable mode, cloud saves, and moved data dirs
    /// (platform-portability.md § Path Rules).
    Replay {
        replay_id: ReplayId,
    },
    /// Branch from a campaign mission-start checkpoint (time machine).
    /// Rewinds use rule-based reconstruction: load target mission map from YAML,
    /// restore CampaignState from the checkpoint, apply carryover rules.
    /// CheckpointId disambiguates the same mission visited on different branches.
    CampaignCheckpoint {
        campaign_id: CampaignId,
        checkpoint_id: CheckpointId,
    },
}

pub enum PlayerAssignment {
    Human { slot: PlayerSlot, player_id: PlayerId },
    Ai { slot: PlayerSlot, preset: AiPresetId, difficulty: AiDifficulty },
    Spectator { player_id: PlayerId },
}

/// Carryover rules for campaign time machine.
/// Models the full authored YAML `carryover_rules` section.
pub struct TimeMachineCarryover {
    pub veterancy: bool,
    pub unit_roster: bool,                          // carry surviving units (usually false — knowledge > power)
    pub exploration: bool,                          // carry explored/shroud state (D041 is_explored, NOT is_visible)
    pub intel_flag_keys: Vec<String>,               // authored allowlist: which keys from D021's
                                                    // flags: HashMap<String, Value> carry back.
                                                    // All other flags are story flags and reset.
    pub story_flags: bool,                          // if true, ALL flags carry back (overrides allowlist)
    pub resource_multiplier: FixedPoint,            // e.g., 0.5 for halved, 0 for none, 1 for full
    pub equipment_mode: EquipmentCarryoverMode,     // none | selective (tagged 'temporal_stable') | all
    pub hero_progression: bool,
    /// Named character or roster-entry selectors for specific units that persist.
    /// Uses RosterUnit identifiers (UnitTypeId + optional name), NOT live UnitId —
    /// UnitId is an in-match entity handle, not stable across mission boundaries.
    /// Matches RosterUnit entries from CampaignState.unit_roster (D021, campaigns.md).
    pub carry_roster_entries: Vec<RosterSelector>,
    pub butterfly_effects: Vec<ButterflyEffect>,
}

/// How to select roster entries for carryover.
pub enum RosterSelector {
    /// By unit type (e.g., "tanya", "spy") — carries all matching roster entries
    ByType(UnitTypeId),
    /// By custom name assigned to a roster unit
    ByName(String),
    /// By hero character ID (for hero progression system)
    ByHeroId(String),
}

pub enum EquipmentCarryoverMode {
    None,
    Selective,  // only equipment tagged 'temporal_stable' in YAML
    All,
}

pub struct ButterflyEffect {
    pub flag: String,
    pub value: Value,
    pub ai_modifier: Option<AiModifier>,
    pub spawn_modifier: Option<SpawnModifier>,
}
```

#### Takeover Execution Flow

**Layer 1 path (replay takeover):**
```
1. Resolve source replay file
2. Load keyframe index, find nearest keyframe ≤ branch_tick
3. Restore keyframe snapshot via GameRunner::restore_full()
   (full snapshot, or full + intervening deltas — same path as replay seeking)
4. Re-simulate forward from restored keyframe to exact branch_tick via apply_tick()
5. Assign player control (human/AI per PlayerAssignment)
6. Transition InReplay → Loading → InGame (game starts paused)
7. Begin recording new branched .icrep
8. Player unpauses → normal gameplay from branch_tick
```

Steps 1-4 reuse existing replay seeking infrastructure (`GameRunner::restore_full()` is the canonical restore contract per `02-ARCHITECTURE.md`). Steps 5-8 reuse existing match start infrastructure.

**Layer 2 path (campaign time machine — rule-based reconstruction):**
```
1. Resolve target checkpoint via CheckpointId from TimeMachineState.checkpoints
2. Detach current TimeMachineState from CampaignState (meta-progress preserved)
3. Load target mission's map and initial conditions from campaign YAML
4. Overwrite CampaignState gameplay fields from CampaignProgressCheckpoint.progress
   (stripped snapshot — roster, flags, stats, equipment; no TimeMachineState)
5. Reattach TimeMachineState with uses_consumed++ and new RewindRecord appended
6. Apply time-machine carryover rules (veterancy, exploration, intel flags per YAML)
7. Apply butterfly effects (AI modifiers, spawn changes, flag overrides)
8. Start mission via normal campaign mission-start flow (Loading → InGame)
9. Begin recording new .icrep
10. Player plays the mission with foreknowledge + carryover advantage
```

Steps 2-5 implement the detach/reattach pattern: gameplay progress is rewound from the stripped checkpoint, but meta-progress (charges used, branch history, all checkpoints) is preserved and never restored to earlier values. The mission state is reconstructed from map + `CampaignProgressSnapshot` + carryover rules, identical to how campaigns normally start missions.

### Implementation Estimate

| Component                                                                           | Crate(s)               | Est. Lines | Layer   | Phase      |
| ----------------------------------------------------------------------------------- | ---------------------- | ---------- | ------- | ---------- |
| Replay takeover orchestration                                                       | `ic-game`              | ~400       | Layer 1 | Phase 3    |
| Takeover UI (dialog, faction picker)                                                | `ic-ui`                | ~300       | Layer 1 | Phase 3    |
| Branched replay metadata (provenance)                                               | `ic-game`              | ~100       | Layer 1 | Phase 3    |
| Multiplayer takeover lobby                                                          | `ic-net`, `ic-ui`      | ~250       | Layer 1 | Phase 5    |
| Console commands (`/takeover`)                                                      | `ic-game`              | ~80        | Layer 1 | Phase 3    |
| Speculative branch preview (ghost)                                                  | `ic-game`, `ic-render` | ~400       | Layer 1 | Phase 3    |
| Campaign time-machine state machine                                                 | `ic-script`            | ~500       | Layer 2 | Phase 4    |
| `TimeMachine` Lua global                                                            | `ic-script`            | ~350       | Layer 2 | Phase 4    |
| Carryover rules engine                                                              | `ic-game`              | ~300       | Layer 2 | Phase 4    |
| Butterfly effects system                                                            | `ic-script`            | ~200       | Layer 2 | Phase 4    |
| Timeline visualization UI                                                           | `ic-ui`                | ~400       | Layer 2 | Phase 4    |
| Time-machine activation cinematic                                                   | `ic-render`, `ic-ui`   | ~200       | Layer 2 | Phase 4    |
| Campaign YAML schema extension                                                      | `ic-script`            | ~150       | Layer 2 | Phase 4    |
| Mission archetypes (convergence, info locks, causal loops, dual-state, retrocausal) | `ic-script`, `ic-game` | ~800       | Layer 2 | Phase 4    |
| Chrono Capture game mode                                                            | `ic-game`              | ~450       | Layer 3 | Phase 5    |
| Time Race game mode                                                                 | `ic-game`              | ~350       | Layer 3 | Phase 5    |
| Timeline Duel (background headless sim)                                             | `ic-game`, `ic-sim`    | ~600       | Layer 3 | M11 (P007) |
| Chrono Vortex hazard system                                                         | `ic-sim`               | ~200       | Layer 3 | Phase 5    |
| Game mode YAML definitions                                                          | data                   | ~150       | Layer 3 | Phase 5    |
| Temporal support powers (age/restore/stasis/decay)                                  | `ic-game`, `ic-sim`    | ~350       | Layer 3 | Phase 5    |
| Temporal pincer ghost army replay                                                   | `ic-game`, `ic-render` | ~400       | Layer 4 | Phase 7    |
| Co-op temporal campaign support                                                     | `ic-script`, `ic-net`  | ~500       | Layer 4 | Phase 7    |
| Temporal Commander (background headless sim)                                        | `ic-game`, `ic-ui`     | ~600       | Layer 4 | M11 (P007) |
| **Total**                                                                           |                        | **~9,030** |         |            |

### Alternatives Considered

| Alternative                                                                        | Verdict  | Reason                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ---------------------------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Free-form multiplayer time travel (Achron model)                                   | Rejected | Too complex for mainstream players. Achron proved this is a niche within a niche. IC's background headless sim pattern could technically support parallel timelines, but Achron's "any player can rewrite any moment in the past at any time" model creates exponential state complexity and is deeply confusing for non-expert players. The layered approach (takeover → campaign → modes → structured dual-timeline) provides time-travel gameplay without the cognitive overload |
| Time travel as cosmetic only (Chronosphere teleport, no actual timeline branching) | Rejected | Misses the opportunity. IC already has Chronosphere as a superweapon (gameplay-systems.md). The time-machine mechanic is about narrative and strategic depth beyond unit teleportation                                                                                                                                                                                                                                                                                              |
| Unlimited time-machine uses in campaigns                                           | Rejected | Removes tension. Limited uses (Prince of Persia pattern) create meaningful "should I use it now?" decisions. Unlimited rewind trivializes the campaign                                                                                                                                                                                                                                                                                                                              |
| Full state carryover through time machine (army + resources + everything)          | Rejected | Makes the game too easy on rewind. Outer Wilds's lesson: knowledge is the best carryover. Partial carryover (veterancy, exploration/shroud, intel) creates the right balance of advantage without trivializing the replayed mission                                                                                                                                                                                                                                                 |
| Time machine only in singleplayer                                                  | Rejected | Multiplayer time modes (Chrono Capture, Time Race) are unique differentiators. No other RTS has them. The multiplayer modes can exist independently of the campaign time machine                                                                                                                                                                                                                                                                                                    |
| Implementing all layers simultaneously                                             | Rejected | Risk is too high. Layer 1 (replay takeover) is proven by SC2 and architecturally trivial. Each subsequent layer adds complexity. Ship layers independently, iterate based on feedback                                                                                                                                                                                                                                                                                               |

### Cross-References

- **Replay system:** [`formats/save-replay-formats.md`](../../formats/save-replay-formats.md) (`.icrep` format), [`formats/replay-keyframes-analysis.md`](../../formats/replay-keyframes-analysis.md) (keyframe seeking), [`player-flow/replays.md`](../../player-flow/replays.md) (viewer UX)
- **Snapshottable state:** [D010](../09a/D010-snapshottable-state.md) (`SimSnapshot`, `SimCoreSnapshot`); restore contract: `GameRunner::restore_full()` in `ic-game` (`02-ARCHITECTURE.md` § ic-game Integration)
- **Campaign system:** [D021](D021-branching-campaigns.md) (campaign graph, `CampaignState`, Lua `Campaign` global), [`modding/campaigns.md`](../../modding/campaigns.md) (full campaign specification)
- **AI presets:** [D043](D043-ai-presets.md) (AI assignment for non-human factions in takeover)
- **Replay highlights:** [D077](D077-replay-highlights.md) (highlight moments as natural takeover entry points — "Take Command from this highlight")
- **Asymmetric co-op:** [D070](D070-asymmetric-coop.md) (Temporal Commander role extends Commander/SpecOps framework)
- **LLM missions:** [D016](../09f/D016-llm-missions.md) (LLM-generated alternate timeline missions in Phase 7)
- **Multiplayer:** [`player-flow/multiplayer.md`](../../player-flow/multiplayer.md) (game mode registration, lobby configuration)
- **Gameplay systems:** [`architecture/gameplay-systems.md`](../../architecture/gameplay-systems.md) (Chronosphere superweapon, support powers)
- **Console commands:** [D058](../09g/D058-command-console.md) (`/takeover`, `/timemachine` commands)
