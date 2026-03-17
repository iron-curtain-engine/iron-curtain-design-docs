# Generated SpecOps Missions — Research Study (Confirmatory Research)

> **Purpose:** Extract practical generation lessons from games and open-source codebases that already solve "repeatable but authored-feeling tactical missions," then map the accepted takeaways into IC's campaign / SpecOps design.
>
> **Date:** 2026-03-16
> **Status:** Confirmatory research (implementation emphasis + authoring-pipeline hardening)

---

## Scope

This note answers one narrow question:

1. How should Iron Curtain build **generated SpecOps missions** that feel authored, remain deterministic, and fit the campaign world-screen model?

This is **not**:
- a recommendation to procedurally generate full macro campaign battles
- a recommendation to replace landmark handcrafted missions
- a claim that any one source game's generator can be copied directly into IC

IC's current direction remains:
- handcrafted missions for flagship story beats
- generated missions for repeatable or branch-variable optional SpecOps opportunities
- deterministic persistence once a generated operation appears on the campaign map

---

## Source Quality and Limits

- **XCOM 2 talks/articles/modding material** are strong sources for:
  - the authored-piece assembly model (`plot` + `parcel` + filler composition)
  - the production reality that procedural missions feel good only when built from handcrafted chunks
- **OpenXcom source** is a strong primary source for:
  - script-driven tactical map assembly
  - weighted block selection
  - hard validation / fail-loud generation discipline
- **Cataclysm-DDA docs** are strong sources for:
  - authored procedural composition
  - nested chunking, palette reuse, and explicit placement constraints
- These sources are **not** authoritative for:
  - IC's exact RTS combat pacing
  - stealth + loud-route hybrid mission grammar
  - commander-supported commando operations

So the goal here is not imitation. It is extracting patterns that survive contact with IC's campaign design.

---

## Sources Reviewed

### Primary Sources

- GDC Vault: "Plot and Parcel: Procedural Level Design in XCOM 2"
  - <https://www.gdcvault.com/play/1025213/Plot-and-Parcel-Procedural-Level>
- GDC presentation mirror: Brian Hess, "Plot and Parcel"
  - <https://media.gdcvault.com/gdc2018/presentations/Hess_Brian_PlotAndParcel.pdf>
- Modder-facing XCOM 2 level-editing overview
  - <https://steamsolo.com/guide/editing-xcom-2-levels-part-1-introduction-xcom-2/>
- Public interview: XCOM 2 procedural generation and mods
  - <https://hardcoregamer.com/features/interviews/e3-2015-talking-procedural-generation-and-mods-in-xcom-2/155373/>
- Public writeup on XCOM 2 procedural maps
  - <https://www.escapistmagazine.com/xcom-2-is-what-happens-when-you-lose-enemy-unknown/>
- OpenXcom `MapScript.cpp`
  - <https://github.com/OpenXcom/OpenXcom/blob/master/src/Mod/MapScript.cpp>
- OpenXcom `MapBlock.cpp`
  - <https://github.com/OpenXcom/OpenXcom/blob/master/src/Mod/MapBlock.cpp>
- OpenXcom `BattlescapeGenerator.cpp`
  - <https://github.com/OpenXcom/OpenXcom/blob/master/src/Battlescape/BattlescapeGenerator.cpp>
- OpenXcom ruleset reference
  - <https://www.ufopaedia.org/index.php/Ruleset_Reference_Nightly_(OpenXcom)>
- Cataclysm-DDA `MAPGEN.md`
  - <https://github.com/CleverRaven/Cataclysm-DDA/blob/master/doc/JSON/MAPGEN.md>
- Cataclysm-DDA `OVERMAP.md`
  - <https://github.com/CleverRaven/Cataclysm-DDA/blob/master/doc/JSON/OVERMAP.md>

### Why these sources were chosen

They directly cover the three implementation questions IC has to solve:

- How do you keep generated missions from feeling like random mush?
- How do you make authored procedural content scalable for creators?
- How do you validate generated tactical spaces so they are not silently broken?

---

## Quick Comparison Matrix

| Source | What it is best at | What IC should take | What IC should reject |
| --- | --- | --- | --- |
| **XCOM 2** | authored procedural assembly at production scale | plots/parcels/filler as a mental model; theater-aware authored chunks; repeatable mission variety without pure randomness | direct copy of turn-based cover assumptions |
| **OpenXcom** | script-driven tactical map assembly with strict validation | weighted selection, max-use limits, generation commands, fail-loud validation, persisted assembled result | heavy reliance on one uniform block/grid mindset for every mission family |
| **Cataclysm-DDA** | composable authored procgen libraries with explicit constraints | nested chunks, palettes/kits, explicit placement rules, reusable mission/site libraries | overly broad sandbox-style worldgen scope for IC's narrower mission needs |

**Converging pattern:** the best generated missions are not "random maps." They are **authored composition systems** with deterministic inputs and strict validation.

---

## High-Value Lessons for IC (Fit / Risk / IC Action)

## 1. Assemble from authored pieces, not raw procedural noise

**Observed across sources:** the successful systems all compose handcrafted content chunks rather than inventing geometry from scratch.

**Fit with IC:** **High**
- IC wants generated SpecOps to feel like believable military sites in known theaters
- This matches the current campaign/world-screen model, where the strategic layer knows *why* the mission exists and the generator decides *where/how* it is realized

**Risk if ignored:**
- missions become tactically mushy
- generated sites feel interchangeable and lose theater identity
- players read the missions as filler instead of deliberate operations

**IC action (accepted):**
- build generated SpecOps from **site kits**, **objective modules**, **ingress modules**, **egress modules**, **security modules**, and **complication modules**
- keep the reward/risk logic in the campaign node and the tactical realization in the generator

---

## 2. Persist the generated result when the operation appears

**Observed across sources:** good systems separate the appearance of a mission opportunity from the player's later replay/view/load actions.

**Fit with IC:** **High**
- IC requires deterministic save/load, replay safety, and stable campaign-state references
- the world screen now carries urgency, risk, and consequence previews; rerolling the map on each open would break trust

**Risk if ignored:**
- players can fish for easier maps by reopening briefings
- campaign state, replay commentary, and mission previews drift apart
- debugging generated content becomes much harder

**IC action (accepted):**
- assign the seed and resolved module set when the operation appears on the campaign map
- persist those values in `CampaignState`
- same campaign seed + same operation state = same generated mission instance

---

## 3. Treat generation as an authored pipeline with hard validation

**Observed across OpenXcom and Cataclysm-DDA:** generation is a pipeline with commands, constraints, retries, and failure handling.

**Fit with IC:** **High**
- IC needs generated missions to advertise real mission promises on the world screen ("stealth route exists", "commander support available", "must exfil under alarm")

**Risk if ignored:**
- the briefing lies to the player
- mission families become unreliable
- broken seeds ship silently

**IC action (accepted):**
- validate at minimum:
  - objective is reachable
  - exfil remains reachable after the alarm state
  - stealth route exists when promised
  - loud contingency route exists when promised
  - start positions are valid
  - commander-support zone cannot bloom into a full macro base
- retry from remaining candidates during development
- fail loudly when the authored pool cannot satisfy the contract

---

## 4. Keep the strategic layer responsible for mission meaning

**Observed as a gap in many procgen discussions:** generation quality collapses when the generator also invents the campaign consequences.

**Fit with IC:** **High**
- IC's campaign model already tracks:
  - success reward
  - failure consequence
  - skip consequence
  - time window
  - downstream consumer missions

**Risk if ignored:**
- generated missions become disconnected minigames
- the player sees random content instead of war-state-driven operations

**IC action (accepted):**
- campaign nodes author:
  - mission family
  - theater
  - urgency
  - reward/failure/skip consequences
  - consuming future missions
- the generator authors:
  - site realization
  - route topology
  - security composition
  - complication

---

## 5. Use handcrafted official missions once; generate the repeatable network around them

**Observed from the campaign-design side:** repeating the same landmark mission weakens both the story beat and the player's sense of place.

**Fit with IC:** **High**
- IC wants the original missions to remain recognizable anchor beats
- optional SpecOps volume should come from unique generated operations, not reusing the same showcase map

**Risk if ignored:**
- the campaign starts feeling padded
- classic missions lose their special status

**IC action (accepted):**
- use official / handcrafted missions once where they fit best
- use generated SpecOps for additional prison breaks, radar raids, scientist extractions, counter-intel sweeps, tech intercepts, and similar opportunities

---

## 6. Expose the generator as a creator-facing registry, not a first-party black box

**Observed across the strongest source patterns:** reusable procgen systems only scale when content authors can add new chunks without engine surgery.

**Fit with IC:** **High**
- this is a modding system, not just a first-party campaign feature
- IC's YAML/Lua/WASM layering is already designed for this kind of escalation path

**Risk if ignored:**
- generated SpecOps becomes a special-case feature only engine developers can extend
- community campaigns fall back to duplicated handcrafted maps instead of authored procedural composition

**IC action (accepted):**
- expose site kits / objective modules / ingress-egress modules / security profiles / complication modules as merged YAML registries
- allow Lua hooks for deterministic filtering, reweighting, and post-generation acceptance
- reserve WASM for custom scoring/validation backends in total conversions, still under a pure deterministic no-I/O contract

---

## 7. Runtime exhaustion needs a campaign-safe fallback policy

**Observed as a missing policy rather than a borrowed lesson:** validation and retries are not enough unless the shipped runtime also knows what to do when the pool is exhausted.

**Fit with IC:** **High**
- campaign operations are part of the strategic layer; they cannot disappear or dead-end the campaign because one authored pool is too small

**Risk if ignored:**
- players can hit impossible generated ops
- optional content can soft-break campaign progress

**IC action (accepted):**
- development/editor path: hard error
- shipped runtime path: fall back to an authored backup mission or resolve the operation as skipped, depending on how the campaign author tagged the node

---

## Recommended IC Mission Grammar

Generated SpecOps should stay inside a deliberately small grammar:

| Family | Core shape | Typical reward | Typical failure/skip effect |
| --- | --- | --- | --- |
| `intel_raid` | infiltrate -> acquire intel -> exfil | patrol routes, access codes, map reveal, branch unlock | enemy alertness preserved; intel route absent |
| `tech_theft` | infiltrate/strike -> secure prototype/scientist -> exfil | unit unlock, prototype roster asset, support power | asset lost or only partially recovered |
| `tech_denial` | sabotage site/convoy/lab -> exfil or survive response | future enemy asset denied | enemy keeps or accelerates the tech |
| `rescue` | breach -> free captive -> extract | hero/VIP returns, compromise reduced | captive moved or further compromised |
| `faction_favor` | save/contact/escort ally network -> defend or extract | resistance support, safe route, partisan aid | no allied package gained |
| `counter_intel` | seize records / kill mole / collapse network -> leave unseen or survive | enemy loses scouting/preparation edge | enemy keeps the preparation advantage |

This is enough to support campaign variety without turning the system into a kitchen sink.

---

## Recommended First-Party Authoring Model

IC should maintain authored libraries for generated SpecOps such as:

- `prison_compound`
- `research_campus`
- `rail_yard`
- `coastal_radar`
- `port_warehouse`
- `safe_house_block`
- `mountain_listening_post`
- `prototype_factory`

Each kit should declare:

- legal theaters
- legal mission families
- socket/attachment points
- stealth affordances
- loud-route affordances
- commander-support affordances

This is the direct bridge between campaign authorship and reusable tactical generation.

---

## Prototype Validation Criteria

A first proof-of-concept is successful if it demonstrates:

1. a campaign node can point at a generated SpecOps profile
2. the generator can resolve:
   - site
   - objective
   - ingress
   - exfil
   - security
   - complication
3. the resolved mission can be serialized into campaign state
4. the mission briefing can display:
   - `AUTHORED` / `GENERATED`
   - success reward
   - failure consequence
   - skip consequence
   - time window
5. the generated mission still pays out a normal campaign-state reward on completion

---

## Recommended Next Step

Do not build a giant mission library first.

Build one vertical slice:

- mission family: `intel_raid`
- theaters: `greece` and `poland`
- 2 site kits
- 2 ingress modules
- 2 exfil modules
- 2 complications
- 2 security tiers

That is enough to validate the whole pipeline:
- campaign node
- generator profile
- tactical realization
- persistence
- briefing UX
- downstream campaign consequence

---

## Companion Prototype

The design companion for this note is:

- `src/modding/generated-specops-prototype.md`

That file is intentionally not a research note. It is the authored proof-of-concept schema showing how the accepted research takeaways map into an IC campaign node, generator profile, and resolved generated operation.
