# Strategic Campaign Layer — Research Study (Adopted Direction Support)

> **Purpose:** Study how successful games build strategic layers around tactical missions, then document the lessons that informed Iron Curtain's adopted phase-based strategic campaign model, where the original RA1 missions remain milestone battles and the operational layer provides connective tissue.
>
> **Date:** 2026-03-16
> **Status:** Confirmatory research (supports adopted D021 strategic-layer extension)

> **Adoption note:** The normative policy now lives in `src/decisions/09d/D021-branching-campaigns.md` and `src/modding/campaigns.md`. This note is supporting research and rationale, not the canonical spec.

---

## Sources Reviewed

| Source | What it is best at | What IC should take | What IC should reject |
|---|---|---|---|
| **XCOM: EU/EW** | Opportunity-cost decisions (abduction sites), panic system, research/engineering loop, Meld as aggression incentive | Pick-one-of-N with visible consequences; authored missions as narrative punctuation; enemy scaling tied to time not player power | Satellite-rush dominance; binary panic resolution; shallow air game |
| **XCOM 2** | Avatar Project doom clock, Dark Events as "pick your poison," procedural map assembly, mission timers forcing aggression | Doom clock that is manageable but never eliminable; Dark Events as enemy initiative cards; authored + generated mission mix at ~15%/85% ratio | Turn timers (not applicable to RTS); scanning-as-busywork; strategic layer that feels like overhead |
| **XCOM 2: WotC** | Covert Actions as passive missions, Chosen as personal nemeses, faction influence | Covert actions = IC's off-screen intel/sabotage ops; nemesis escalation = hero capture system; faction orders = theater-front bonuses | Three-faction influence grind; information overload from too many parallel systems |
| **FTL** | Rebel fleet as spatial time pressure, node-based route planning, scrap economy | Spatial/visible doom pressure rather than abstract counter; tight resource economy where every spend is a tradeoff | Pure roguelike reset; single-run structure |
| **Into the Breach** | Grid power as shared campaign health, perfect information + insufficient actions, island selection | Shared strategic resource that depletes across missions (war momentum); letting the player see all threats and choose which to absorb | Puzzle-like single-solution turns; minimal campaign layer |
| **Total War** | Campaign map giving context to every battle, persistent armies, multi-front management, emergent narratives | Every battle exists in a concrete strategic context; losing an army has permanent consequences; multi-front prioritization | Full sandbox scope; "painting the map" tedium; auto-resolve trivializing tactical layer |
| **Battle Brothers** | Financial pressure as organic time pressure, world simulation running independently, contract selection | Constant resource drain forcing engagement; the world moves whether the player acts or not; operation selection from a dynamic pool | Open-ended sandbox without authored milestones; mid-game pacing sag |
| **Jagged Alliance 2** | Sector control with economic consequences, militia training tradeoff, NPC roster management, counterattack pressure | Territory has economic meaning; best soldiers must choose between fighting and infrastructure; enemy counterattacks on held gains | Granular inventory management; overwhelming complexity for new players |
| **StarCraft 2: WoL** | Hub space with character interactions, mission board with flexible order, Armory/Lab permanent upgrades, mutually exclusive choices | Hub feel between missions; flexible mission order within authored constraints; permanent upgrade decisions that create replay value | Lack of real consequence for failure; cosmetic strategic layer |
| **Hearts of Iron IV** | National Focus trees as structured decisions, global simulation, front-line abstraction, production lag | Focus trees = IC's theater investment arcs; the "bigger picture" feeling from a world in motion; planning ahead because production has lead time | Spreadsheet complexity; micro-management fatigue; AI struggles |

---

## Key Patterns Extracted

### 1. Opportunity cost is the engine of strategic fun

Every source confirms: the core unit of strategic game design is "you can do A or B, but not both." XCOM's abduction sites, FTL's route planning, Battle Brothers' contract selection, JA2's fight-or-train dilemma — all produce interesting decisions through visible, permanent opportunity cost.

**IC application:** The player must choose which operations to pursue between main missions. Unchosen operations resolve with consequences. This extends the timed-choice mechanic into the continuous campaign experience rather than leaving it as a discrete event at only a few authored nodes.

### 2. The world must move without you

The strongest "commanding a war" feeling comes from a world that operates independently. Battle Brothers' simulated economy, Total War's AI factions taking turns, XCOM 2's Avatar Project advancing, FTL's rebel fleet sweeping the map. The player reacts to a world in motion, not a world that waits.

**IC application:** Enemy Initiatives advance between operations. Fronts degrade if ignored. The Soviet war machine doesn't pause while the player optimizes.

### 3. Authored milestones + dynamic operations

XCOM 2 proved the formula: ~15% authored story missions as narrative punctuation, ~85% generated/pool missions as connective tissue. SC2:WoL showed that even in a story-driven campaign, flexible mission order and mutually exclusive choices create replay value. The authored missions are the chorus; the operations are the verses.

**IC application:** The 14 original RA1 missions are the authored milestones (always present, same order). CS/AM expansion missions + IC originals + generated SpecOps fill the operational pool between them. Each playthrough's journey between milestones is different.

### 4. Enemy initiative creates reactive play

XCOM 2's Dark Events are the clearest model: each month, the enemy launches 2-3 initiatives. The player can counter one by choosing the right mission. The rest activate. This adds variety to the tactical game, creates narrative flavor (the enemy adapts), and forces reactive decision-making alongside proactive planning.

**IC application:** Each campaign phase reveals 1-2 Soviet Initiatives on the War Table. Some can be directly countered by a specific operation in the pool. Others resolve automatically. The player must weigh "counter this threat" against "pursue this opportunity."

### 5. A doom clock prevents turtling

The Avatar Project (XCOM 2) is widely considered one of the best doom clocks in strategy gaming. It works because: it is a single clear unified threat, it is always visible, it can be pushed back but never eliminated, and it creates dramatic reversals. FTL's rebel fleet achieves the same effect spatially.

**IC application:** War Momentum — a phase-level pressure that reflects the overall war trajectory. If the player ignores operations and enemy initiatives pile up, the final missions become dramatically harder. Unlike XCOM's Avatar bar, IC's doom pressure is distributed across the asset ledger: each missed opportunity makes M14 worse, and the accumulation is visible on the War Table.

### 6. Persistent resources bridge tactical and strategic

XCOM's alloys-to-plasma pipeline, JA2's mine-income-to-mercenary loop, Battle Brothers' loot-to-equipment chain — the best strategic layers create a bidirectional feedback loop where tactical performance affects strategic resources and strategic decisions affect tactical options.

**IC application:** Operations produce concrete assets consumed by main missions. Roster carryover means tactical performance (unit survival, veterancy) feeds forward. Hero progression bridges both layers. The asset ledger on the War Table makes this visible.

### 7. The strategic layer must respect the tactical layer's time

XCOM's tactical missions are 20-45 minutes; the Geoscape is fast. Total War's campaign turns take minutes; battles take 10-30 minutes. The strategic layer should enhance, not compete with, the core gameplay. Paradox games are a cautionary tale: when the strategic layer becomes the whole game, the tactical moments lose weight.

**IC application:** The War Table must be lean. Operations are the same RTS gameplay the player came for. The between-mission strategic UI (War Table) should take 2-5 minutes, not 20. The player should spend most of their time in missions, not managing spreadsheets.

---

## Narrative Foundation — From Einstein's Experiment to Mission 1

The original Red Alert opens with Einstein using a temporal prototype at Trinity, NM (1946) to travel back to 1924 and erase Hitler from history. The game never lingers on what happens next — the briefing jumps straight to "the Soviets invaded." IC must make this timeline coherent, because the Enhanced Edition asks the player to invest 25-29 missions in a war that needs to feel earned, not arbitrary.

### The Timeline

> This is a narrative summary for campaign context. For the full canonical period-by-period reference (the "show bible"), see § The 22-Year Vacuum Period Timeline below.

1. **1946 — Einstein's Experiment.** Albert Einstein, horrified by the destruction of World War II, activates a temporal displacement prototype at Trinity, New Mexico. He travels back to 1924 and erases Hitler from history. He succeeds. Hitler never rises. The Nazi party never takes power. Einstein returns to 1946 — but to a world he no longer recognizes.

2. **1924-1946 (new timeline) — The Vacuum.** Without Hitler, Germany never remilitarizes. The Western democracies never face the existential threat that united them. France, Britain, and the United States remain fractured, isolationist, and complacent. There is no NATO. There is no Marshall Plan. There is no shared memory of a common enemy. Einstein's 1946 is peaceful on the surface — no World War II happened — but the power vacuum he created has been quietly filling for 22 years.

3. **Meanwhile, the Soviet Union grows unchecked.** Stalin's industrialization program proceeds without the Eastern Front consuming Soviet manpower and materiel. By the mid-1940s, the USSR possesses the largest standing army in Europe, an advanced weapons program, and an ideology with no credible opposition.

4. **1945–1946 — Soviet expansion begins.** Stalin, facing no Western alliance and no rearmed Germany, pushes west. Eastern Europe falls — not through sudden invasion but through political subversion, border incidents, staged "liberation" campaigns, and economic coercion. Poland, Czechoslovakia, Hungary, Romania — absorbed one by one. The West protests but does not act. There is no Churchill "iron curtain" speech because there is no shared context for Western unity. (For the full period-by-period breakdown, see § The 22-Year Vacuum Period Timeline below.)

5. **1946 — Greece.** Soviet forces push into Greece. This is the point where the Western powers can no longer ignore the threat. Greece is a Mediterranean ally, a trade partner, a strategic position. The Allied coalition forms — late, improvised, and outgunned.

6. **The Prologue — Stavros and Tanya.** The first three missions take place in Greece as the Allied response crystallizes. Stavros is a local resistance leader trying to survive; Tanya is a Western special operative sent to support the Greek front. Their missions are the first Allied actions of the war.

7. **M1 — Rescue Einstein.** By the time the formal Allied command structure is in place, Einstein has been captured by Soviet forces who have learned of his Chronosphere research. The Soviets see Einstein not as a war criminal but as a strategic asset — his temporal technology could give them an unassailable advantage. M1 is the first operation under formal Allied command: rescue the one man who might give the Allies a technological edge.

### Why This Matters for the Campaign

- **The player enters a losing war.** The Allies are behind from the start. The strategic layer (War Table, operations, asset ledger) is about clawing back from a deficit — not managing an empire.
- **Einstein is compromised.** He created this timeline. He knows it. Einstein's Burden (the narrative thread) is not abstract guilt — it is the specific knowledge that every death in this war exists because of his choice.
- **The prologue earns the War Table.** The player has already fought three desperate missions in Greece before the strategic layer opens. They know the war is real. The War Table is not a menu screen — it is the first time the player has enough breathing room to make strategic choices instead of just reacting.
- **M1 has stakes.** Einstein is not a generic scientist to rescue — he is the person who caused this war AND the person who might end it. The player knows both facts by M1.

### What IC Changes From the Original

| Element | Original RA1 | IC Enhanced Edition |
|---|---|---|
| Einstein's motive | Vague/comedic | Explicit — he saw WWII's destruction and made a desperate choice |
| The vacuum period | Skipped entirely | 22-year gap shown through briefing materials and Stavros backstory |
| Why the Allies are losing | Never explained | Western complacency without a shared threat; Soviet industrialization unchecked |
| Why Greece first | Arbitrary mission order | Greece is the flashpoint where Western inaction becomes impossible |
| Who Stavros is | Briefing character | Playable prologue protagonist; the player's first connection to the war |
| Why Einstein matters | He invented things | He caused this timeline AND holds the key to ending it |
| The player's entry point | Cold open — "rescue Einstein" | Three missions of context → War Table → M1 with full understanding |

---

## IC Strategic Campaign Model — "The War Table"

### Design Goals

1. **The original 14 missions are the skeleton.** They always occur in roughly the same order and serve as narrative milestones. Their difficulty and available approaches depend on the operational layer.
2. **Between milestones, the player commands the war.** The War Table shows active fronts, available operations, enemy initiatives, and accumulated assets. The player picks which operations to pursue.
3. **Every playthrough is different.** Generated SpecOps fill the operational pool alongside authored missions. The combination of which operations are available, which enemy initiatives appear, and which the player chooses creates unique campaigns.
4. **The war moves without you.** Enemy Initiatives advance. Fronts degrade. Operations expire. The player cannot do everything.
5. **It must be fun, not homework.** The War Table is lean. Decisions are clear. The player spends most of their time in RTS missions, not in menus.
6. **Classic path is always available.** A player who ignores the War Table entirely can play the 14 main missions in sequence. Harder, but completable.

### Campaign Flow

```
PROLOGUE (linear, tutorial)
    │
    ▼
┌─────────────────────────────────────────┐
│           THE WAR TABLE                 │
│                                         │
│  Active Fronts: Greece, Mediterranean   │
│  Available Operations: [3-5 cards]      │
│  Enemy Initiatives: [1-2 cards]         │
│  Assets: [ledger]                       │
│  War Clock: ██░░░░░░░░ Phase 1          │
│                                         │
│  [Pick Operation]  [Launch Main Mission]│
└─────────────────────────────────────────┘
    │                    │
    ▼                    ▼
 Play operation    Play main mission
    │                    │
    ▼                    │
 Return to War Table     │
 (op budget -1)          │
    │                    │
    ▼                    ▼
 Budget remaining?   Phase complete
 Yes → pick again    Next phase opens
 No  → must launch   Return to War Table
       main mission
```

### Campaign Phases

Each phase is defined by:

1. **Main Mission(s)** — the authored RA1 milestone(s) that end the phase
2. **Operational Budget** — how many operations the player can run before the main mission becomes urgent (typically 1-3)
3. **Operation Pool** — which operations are available (authored CS/AM/IC missions + generated SpecOps)
4. **Enemy Initiatives** — what the Soviet side is doing this phase
5. **Phase Trigger** — what makes the main mission urgent (narrative event, war clock, enemy action)

Operations are always optional. The player can launch the main mission immediately with zero operations. The campaign is completable on the classic 14-mission path. Operations make it richer, not mandatory.

### Operational Turns

Each operational turn:

1. The War Table shows available operations as cards (same format as existing operation cards: role tag, reward preview, fail/skip consequence, time window)
2. The player picks one and plays it (a full RTS or SpecOps mission)
3. Results apply: assets gained, flags set, fronts shift, hero status updated
4. The War Table updates: some operations may expire, new ones may appear based on state
5. Enemy Initiative timers advance (one tick per operational turn)
6. If operational budget remains, the player can pick another operation or proceed to the main mission
7. If budget is exhausted, the main mission becomes URGENT and must be launched

### Enemy Initiatives

Each phase, the War Table reveals 1-2 Enemy Initiatives — things the Soviet war machine is doing independently. These are IC's version of XCOM 2's Dark Events.

```yaml
enemy_initiatives:
  chemical_weapons_deployment:
    phase: 3
    title: "Chemical Weapons Deployment"
    description: "Soviet forces are activating Sarin gas facilities in Greece."
    effect_if_uncountered:
      set_flag:
        sarin_active: true
    briefing_consequence: "M8 gains 2 gas barrages and 1 contaminated approach lane."
    countered_by_operation: cs_crackdown
    counter_text: "Neutralize the Sarin facilities before they go active."

  radar_network_expansion:
    phase: 3
    title: "Radar Network Expansion"
    description: "Soviet radar coverage is extending across the Mediterranean."
    effect_if_uncountered:
      set_flag:
        soviet_radar_expanded: true
    briefing_consequence: "Future missions start with less shroud revealed."
    countered_by_operation: null  # no direct counter — absorbed as cost

  super_soldier_program:
    phase: 3           # introduced Phase 3, escalates Phase 4
    title: "Project Lazarus — Soviet Super Soldier Program"
    description: >
      Allied intelligence uncovers evidence of a classified Soviet program:
      cybernetic augmentation of a single elite operative — codename "Volkov."
      Research facilities in Eastern Europe are producing prototype implants.
      If the program reaches completion, the Soviets will deploy an
      unkillable battlefield asset.
    discovery_text: >
      Intercepted transmissions reference a "Project Lazarus" — a Soviet
      super-soldier initiative. Our analysts believe they are weeks from
      producing a viable subject. We can act now, or face the consequences.
    multi_phase: true  # spans two phases — must be countered in either
    legs:
      - phase: 3
        countered_by_operation: ic_lab_rats
        counter_text: "Raid the research facility and destroy the prototype implants before mass production begins."
      - phase: 4
        countered_by_operation: ic_shut_it_down
        counter_text: "The program has moved to a hardened production site. Destroy it before the subject is activated."
    effect_if_uncountered:
      set_flag:
        volkov_deployed: true
    briefing_consequence: >
      Volkov — the completed super-soldier — is deployed to the front.
      Heavily armored, self-repairing, immune to conventional small arms.
      He appears as an enemy unit in M8 and subsequent main missions.
      The player must outmaneuver or contain him, not fight him head-on.
      If partially countered (one leg completed), Volkov is deployed but
      the rushed activation leaves him weakened (no self-repair, reduced armor).
    partial_counter_effect:
      set_flag:
        volkov_deployed: true
        volkov_weakened: true
```

**Design rules for Enemy Initiatives:**

1. **1-2 per phase, never more.** The player can process 1-2 threats; more creates noise.
2. **Some are counterable, some are not.** Counterable initiatives have a specific operation in the pool that neutralizes them. Non-counterable ones are costs the player absorbs — they create the "things got worse" feeling.
3. **Effects are concrete and quantified.** Not "enemy is stronger" but "M8 gains 2 gas barrages." Same standard as operation cards.
4. **Initiatives are visible before they activate.** The player sees them on the War Table and can plan around them.
5. **They advance on operational turns.** Each operation the player runs before the main mission gives initiatives one more tick toward activation. This creates tension: doing more operations gives you more assets but also lets more initiatives activate. (But most initiatives activate at phase end regardless, so this is a secondary pressure, not a punishment for engaging with the system.)

### War Momentum

Instead of a single XCOM-style progress bar, IC tracks war momentum through the **asset ledger** — the cumulative record of what the player has gained and lost. The War Table surfaces this as a simple readout:

```
WAR MOMENTUM
  Assets secured: 7/15 available this campaign
  Fronts stable: 3/5
  Enemy projects countered: 4/8
  M14 difficulty estimate: HARD (baseline: VERY HARD)
```

This is not a fail state — the player cannot "lose" from momentum alone. But a player who ignores the operational layer entirely faces M14 at maximum difficulty with no bonus assets, no allied reinforcements, no denied enemy tech, and no intel advantages. A player who engages fully faces a dramatically easier and more varied endgame.

The momentum readout replaces a doom clock with a **preparation gauge**: "how ready are you for the endgame?" This is closer to SC2:WoL's Armory/Lab progression than XCOM's Avatar bar, and it fits IC better because the campaign is not about preventing a loss condition — it is about preparing for a known final battle.

### How Main Missions Fit

Main missions are unchanged from the original RA1 missions. What changes is the context:

- **Before:** Fixed mission in a fixed graph position with fixed difficulty.
- **After:** Same mission, same narrative position, but difficulty and available approaches are shaped by the operational layer.

Example — **M8 "Protect the Chronosphere"** (baseline: HARD):

| Condition | Modifier |
|---|---|
| `sarin_denied` (Crackdown completed) | No chemical attacks |
| `skyfall_complete` (Skyfall completed) | 2 AA sites pre-destroyed, -1 Soviet air wave |
| `spy_network_active` (Spy Network completed) | 40% shroud revealed at start |
| `soviet_radar_expanded` (initiative uncountered) | Enemy detects your movements earlier |
| `einstein_confession_done` | Narrative context added |
| None of the above | Baseline HARD — no bonuses, no penalties |

This is already designed in the existing staged changes. The strategic layer model does not change what these missions are — it changes how the player earns the assets that shape them.

### How Generated SpecOps Fit

Generated SpecOps are first-class members of the operation pool. Each phase, the pool contains:

1. **Authored missions** — CS/AM expansion missions and IC-original missions, placed at specific phases
2. **Generated SpecOps** — drawn from the mission-family grammar (`intel_raid`, `tech_theft`, `tech_denial`, `rescue`, `faction_favor`, `counter_intel`) based on the current theater and campaign state

The generated missions use the existing pipeline: site kits, objective modules, ingress/egress, security, complications, validation, persistence. They appear as normal operation cards on the War Table with full reward/risk disclosure.

**Generation trigger:** At the start of each phase, the campaign system evaluates:
- Which theaters are active
- Which mission families are relevant to current campaign state
- Which site kits are available for those theaters
- How many generated operations to offer (typically 1-2 per phase, alongside authored ones)

**Diversity rule:** The pool should not offer two operations of the same family in the same phase. If the pool has an authored `intel_raid` (e.g., Behind Enemy Lines), the generator should not also create a generated `intel_raid` — it should pick a different family.

### How Existing Features Integrate

| Feature | Role in Strategic Model |
|---|---|
| **Timed choices** | Subsumed. Every phase IS a choice point. The current 3 authored timed choices become "crisis phases" with heightened stakes and more operations competing |
| **Generated SpecOps** | Fill the operation pool with per-playthrough variety |
| **Operation cards** | The War Table's primary UI element, unchanged |
| **Spectrum outcomes** | Operations and main missions both use spectrum outcomes; results feed the asset ledger |
| **Hero capture escalation** | Ticks forward with each phase. If Tanya is captured, the rescue operation appears in the pool and urgency increases each phase |
| **Asset ledger** | Central to the War Table; tracks everything the player has earned/lost |
| **Commander alternatives** | Available within operations (same as current design) |
| **Theater branches** | Multi-phase operational arcs; e.g., Poland chain spans Phases 5-7 |
| **Enemy Initiatives** | **New.** 1-2 per phase, counterable or absorbed |
| **Hero progression** | Tanya/Volkov gain XP from operations and main missions alike |
| **Embedded task force** | Specific operations and main mission variants use this archetype |
| **Air campaign** | Specific operations use this archetype (Skyfall, Tempest) |
| **Dynamic weather** | Main missions and some operations showcase weather |
| **Campaign menu scenes** | Change at phase boundaries (tied to act progression) |
| **D078 time machine** | Experimental add-on; Einstein's Confession appears as a Phase 5 operation |
| **D070 co-op** | Phase 6b add-on; specific operations gain co-op variants |

### Hero Survival Model

Heroes do not die in the traditional sense. Battlefield "death" is reframed as wounding, and permanent loss requires sustained strategic neglect — not a single bad engagement.

**Three tiers:**

| Tier | Trigger | Effect | Recovery |
|---|---|---|---|
| **Wounded** | Hero HP hits zero during a mission | Hero is unavailable for the next 1-2 operations (sits out, healing) | Automatic — returns to the roster after the recovery window |
| **Critical Injury** | Hero is wounded a 3rd time within the same act (cumulative) | Hero is unavailable for the remainder of the current phase; gains a permanent minor debuff (e.g., -5% speed, -1 ability charge) | Automatic return next phase, but the debuff persists for the rest of the campaign |
| **Permanent Loss** | Hero is captured AND the player fails to rescue them within the rescue window (2 phases); OR hero accumulates 3 critical injuries across the campaign | Hero is removed from the roster permanently. A replacement unit (less capable) may become available | No recovery — this is the consequence of sustained neglect |

**Design rationale:**

- **No single-mission permadeath.** A bad engagement costs tempo (hero unavailable), not a campaign-ending asset loss. This fits IC's "failure-forward" philosophy — the player always continues.
- **Cumulative wounds create narrative weight.** A hero who has been wounded twice feels fragile. The player starts protecting them more carefully. This is emergent storytelling through mechanics.
- **Permanent loss requires repeated failure.** The player must ignore a captured hero for two full phases (plenty of warning) OR accumulate three critical injuries (a pattern of recklessness, not a single mistake). This is a strategic failure, not a tactical one.
- **Capture is the real danger.** Battlefield wounding is recoverable. Capture starts a timer with escalating consequences (existing capture escalation system). The worst outcomes come from capture + neglect, not from combat damage alone.

**Interaction with the War Table:**

- A wounded hero appears on the War Table with a "RECOVERING" tag and a turn count until return.
- A captured hero appears with a "CAPTURED" tag and a rescue operation in the pool. The urgency marker increases each phase.
- The War Momentum readout includes hero status: "Tanya: ACTIVE / WOUNDED (1 turn) / CAPTURED (Phase 4, rescue available) / LOST."

**Modder contract:**

```yaml
hero_survival:
  wound_recovery_turns: 2          # operations until hero returns
  critical_injury_threshold: 3     # wounds-per-act before critical
  critical_injury_debuff: minor    # minor | major (modder choice)
  capture_rescue_window_phases: 2  # phases before permanent loss
  permanent_loss_threshold: 3      # critical injuries across campaign
  replacement_unit_available: true # whether a lesser replacement spawns
```

### Tech Acquisition & Denial Matrix — The Arms Race

In the original Red Alert campaigns, the player gets a fixed tech tree that unlocks linearly. Mission 1 gives infantry. Mission 5 gives tanks. Mission 10 gives everything. There is no choice, no trade-off, no consequence. Every playthrough has the same arsenal.

IC replaces this with a **dynamic arms race**. Every expansion-pack unit, every prototype, every support power, and every enemy capability is tied to the operational layer. What you have at M14 depends on what you did at the War Table. What the *enemy* has at M14 depends on what you *failed* to do.

This is the single biggest reason a player will never want to go back to the originals: in IC, the arsenal is *yours* — earned, denied, traded, and fought for.

#### Design Principles

1. **No free unlocks.** Expansion-pack units are not handed out by mission number. They are campaign rewards earned through operations, theater branches, or tech theft missions. A player who skips the Italy chain never sees a Chrono Tank.

2. **Three-outcome tech.** Every major technology has three possible states — not just "have it / don't have it":
   - **Acquired** — full capability, earned through the optimal operation path
   - **Partial** — degraded version, earned through a suboptimal path (commander assault instead of commando infiltration, or rushed prototype recovery)
   - **Denied** — unavailable this playthrough (operation skipped, failed, or never offered)

3. **Mirror arms race.** For every tech the player can acquire, there is an enemy tech they can deny — and vice versa. The Soviets are running their own arms race. If the player focuses entirely on acquiring tech and ignores denial operations, they'll face the enemy's full arsenal at M14 even if their own arsenal is stacked.

4. **Commander choice shapes tech quality.** The commando path on a tech operation typically yields the *full* prototype (intact schematics, complete research data). The commander alternative typically yields a *partial* result (captured but damaged, destroyed instead of captured, or denial without acquisition). Both are valid. Neither is wasted. But they produce different arsenals.

5. **Compound stacking.** Multiple tech acquisitions in the same domain compound. Air Superiority alone gives bombing runs. Skyfall alone gives AA denial. Both together give a **combined air package** that is more than the sum of its parts. This rewards breadth without punishing depth.

6. **The briefing tells you.** Every tech state is surfaced in mission briefings. Before M14, the player sees exactly what they have and what they're facing. No hidden variables. The War Table's asset ledger is the player's arms-race scoreboard.

#### Allied Tech Acquisition

| Tech / Unit | Source Operation | Phase | Commando Path Result | Commander Path Result | If Skipped / Failed | Consumed By |
|---|---|---|---|---|---|---|
| **Chrono Tank** (AM prototype) | Italy Chain: "In the Nick of Time" + follow-ups | 5-6 | 1 fully functional Chrono Tank with temporal shift ability | 1 damaged Chrono Tank — no temporal shift, but still a heavy hitter | No prototype; southern counterattack remains in M14 | **M12** (deploy), **M14** (deploy + no southern flank) |
| **Super Tanks** (AM heavy armor) | Poland Chain: "Monster Tank Madness" + follow-ups | 6-7 | 2 Super Tanks + full schematics (repair capability) | 1 Super Tank, damaged (no field repair) | No Super Tanks; Poland arc closed; M14 has no partisan armor | **M12** (west-flank entry), **M14** (heavy assault) |
| **Phase Transport** (AM stealth vehicle) | Generated SpecOps: `tech_theft` in Mediterranean, or Italy Chain bonus | 5-7 | 2 Phase Transports with full cloak duration | 1 Phase Transport with reduced cloak (50% duration) | Unavailable | **M9** (stealth insertion option), **M13** (infiltration) |
| **Field Medic** (IC support infantry) | SpecOps: "Rescue Dr. Hoffmann" (authored) or generated `rescue` | 3-5 | Field Medic joins roster permanently; can heal hero units | Medic rescued but injured — available every other mission | No field healing; hero wounds recover only between phases | **All subsequent missions** (roster unit) |
| **Partisan Squads** (resistance infantry) | Poland Chain liberation + Faction Favor ops | 6-8 | 3 veteran partisan squads with local-terrain bonuses | 2 green partisan squads, no terrain bonus | No partisans; M14 has no resistance reinforcements | **M14** (reinforcement wave) |
| **Naval Detachment** (bonus fleet) | Sarin Chain: "Down Under" completion + Mediterranean ops | 4-5 | Full naval detachment: 2 Destroyers + 1 Cruiser | Partial detachment: 1 Destroyer only | M7 faces full Soviet naval strength alone | **M7** (starting fleet), **M11** (reinforcements) |
| **Air Package** (strategic bombing) | "Operation Skyfall" + "Air Superiority" | 5-6 | Combined: 2 bombing runs in M12, 3 in M14, AA pre-destroyed in M8 | Individual: Skyfall gives AA denial only; Air Sup gives 1 bombing run only | No air support; M8 has full AA; M14 has no bombing | **M8**, **M12**, **M14** |
| **Chemical Expertise** (defense kit) | Sarin Chain complete: Crackdown → Down Under → Controlled Burn | 3-5 | Chemical defense package: gas masks for all infantry, decontamination vehicle | Partial: gas masks only (no decontamination) | No chemical defense; if Sarin active, infantry are vulnerable | **M8** (if Sarin active), **M14** (if enemy deploys chemical reserves) |
| **Einstein Prototype** (D078 experimental) | "Einstein's Confession" narrative mission | 5 | If D078 enabled: temporal intel device (knowledge carryover). If D078 disabled: lore only, no gameplay tech | N/A — narrative mission, no commander alternative | Narrative thread incomplete; Einstein's Burden unresolved | **Epilogue** (narrative), **D078** (if enabled) |
| **Forward Command Kit** (support structures) | Generated SpecOps or authored commander-support ops | 3-7 | Expanded support package: Field HQ + Repair Bay + Sensor Post + 4 structures | Basic package: Field HQ + Repair Bay only, 2 structures | Default commander-support package in SpecOps missions | **All commander-supported SpecOps** |
| **Spy Network** (intel infrastructure) | "Spy Network" operation (requires Tanya not captured) | 3 | Full network: 40% shroud reveal in M6, patrol routes in M10B, compound intel bonuses | N/A — SpecOps only, no commander path | No pre-reveal, no route markers, no compound intel | **M6**, **M10B**, **M14** (if compounded) |

#### Allied Tech Denial (What You Can Prevent the Enemy From Having)

| Enemy Tech | Denial Operation | Phase | Full Denial Result | Partial Denial Result | If Uncountered | Affects |
|---|---|---|---|---|---|---|
| **Sarin Gas** (chemical weapons) | Crackdown (Sarin 1) | 3 | No gas attacks anywhere in the campaign | N/A — binary denial | M8 gains 2 gas barrages + 1 contaminated lane; M14 chemical reserve | **M8**, **M14** |
| **Project Lazarus / Volkov** (super-soldier) | Lab Rats (Phase 3) + Shut It Down (Phase 4) | 3-4 | No Volkov encounters | Volkov deployed but weakened (no self-repair, reduced armor) | Volkov at full strength in M8, M12, M14 | **M8**, **M12**, **M14** |
| **Soviet Super Tanks** (heavy armor) | Generated SpecOps: `tech_denial` + Poland Chain (capturing their factories) | 6-7 | Enemy cannot field Super Tanks in Act 3 | Enemy fields 1 Super Tank instead of 3 | 3 Soviet Super Tanks in M14 defense line | **M14** |
| **Soviet Radar Network** (detection) | Radar station sabotage ops (generated `tech_denial`) | 3-5 | Future missions start with normal shroud; enemy loses early detection | Radar degraded — enemy detects 30% later than full coverage | Enemy detects player movements early; less setup time in missions | **M7**, **M8**, **M12** |
| **Soviet Air Force** (MiG strength) | "Operation Skyfall" (AA destruction) | 5 | M8 starts with 2 AA sites destroyed, 1 fewer air wave | 1 AA site destroyed only | Full Soviet air strength in M8 | **M8** |
| **Iron Curtain Effectiveness** | "Focused Blast" precision path (M13 variant) | 9 | Iron Curtain completely removed from M14 | Siege path: 1 emergency Iron Curtain pulse at minute 12 of M14 | Full Iron Curtain active in M14 | **M14** |
| **Soviet Fortifications** (static defenses) | Air Superiority bombing + Poland flanking route | 6-7 | M14 fortifications reduced; flanking route bypasses main defense line | Partial: bombing weakens some, but no flank | Full fortified defense in M14 | **M14** |
| **Soviet Naval Strength** (submarine fleet) | M7 "Sunken Treasure" performance + pre-ops | 5 | Sub pens destroyed; M11 faces reduced naval opposition | Sub pens damaged; M11 faces standard opposition | M11 faces reinforced submarine groups | **M11** |
| **Emergency Mobilization** (reserve waves) | Siberian Chain completion (reserves diverted) | 7-8 | M14 reserve waves halved (Soviets fighting on two fronts) | Reserve waves reduced by 25% (Siberian pressure but not collapse) | Full reserve waves in M14 | **M14** |

#### Soviet Tech Acquisition

| Tech / Unit | Source Operation | Phase | Commando Path Result | Commander Path Result | If Skipped / Failed | Consumed By |
|---|---|---|---|---|---|---|
| **Tesla Tank** (AM assault vehicle) | Research chain: "Legacy of Tesla" + follow-ups | 4-5 | 2 Tesla Tanks with overcharge ability | 1 Tesla Tank, standard power | No Tesla Tanks; rely on standard armor | **M11**, **M14** |
| **Demolition Truck** (AM suicide vehicle) | Engineering ops: sabotage-specialist training | 3-4 | 3 Demolition Trucks with enhanced blast radius | 2 Demolition Trucks, standard blast | Unavailable; must use conventional siege | **M8**, **M12** (siege options) |
| **M.A.D. Tank** (AM area-denial) | Tech theft from Allied prototype lab | 6-7 | 1 M.A.D. Tank with full seismic capability | Captured but unstable — single use only, then destroyed | No area-denial weapon | **M14** (breach option) |
| **Missile Sub** (AM naval unit) | Naval research chain + Mediterranean operations | 5-6 | 2 Missile Subs with extended range | 1 Missile Sub, standard range | Standard submarine fleet only | **M11**, **M14** (naval phase) |
| **Shock Trooper** (AM elite infantry) | Infantry enhancement program: "Iron Will" | 4-5 | Shock Trooper squad (4 units) with enhanced armor | 2 Shock Troopers, standard armor | Standard conscript infantry only | **All subsequent missions** (roster) |
| **Prototype MiG** (advanced air) | "Legacy of Tesla" chain + air research | 5-6 | 1 prototype MiG sortie in M11 + 1 emergency air strike in M14 | MiG prototype captured but damaged — M11 sortie only, no M14 strike | No prototype air support; M11 is naval-only | **M11**, **M14** |
| **Volkov Augmentation** (hero upgrade) | Project Lazarus (from Soviet perspective — the player IS the program) | 3-4 | Full augmentation: Volkov gains self-repair, heavy armor, enhanced abilities | Rushed augmentation: armor and abilities, but no self-repair | Volkov remains a skilled but human officer; no super-soldier | **All subsequent missions** (hero power) |
| **Iron Curtain Enhancement** (superweapon) | Research operations + Red Ledger fragments | 5-7 | Iron Curtain covers 3 units for 45 seconds | Standard: 2 units for 30 seconds | Baseline Iron Curtain (weakest version) | **M12**, **M14** |

#### Soviet Tech Denial (What Soviets Can Prevent Allies From Having)

| Allied Tech | Denial Operation | Phase | Full Denial Result | If Uncountered | Affects |
|---|---|---|---|---|---|
| **Chronosphere** (temporal weapon) | Counter-intel ops + Einstein containment | 5-7 | Chronosphere disabled or delayed in M14; Allies cannot teleport assault force | Allies have full Chronosphere capability | **M14** |
| **Allied Air Superiority** (bombing runs) | AA expansion + MiG dominance ops | 5-6 | No Allied bombing runs in endgame | Allied bombing runs weaken Soviet fortifications | **M12**, **M14** |
| **GPS Satellite** (full intel) | Counter-spy operations | 4-6 | Allied missions have standard fog of war | Allies start with partial/full shroud reveal | **M10**, **M12** |
| **Chrono Tank** (Allied prototype) | Intercept the Italy salvage convoy | 5-6 | Allies never recover the Chrono Tank prototype | Allies deploy Chrono Tank in endgame | **M14** |
| **Partisan Networks** (resistance fighters) | Suppress resistance operations + Security Sweeps | 6-8 | No partisan reinforcements in Allied endgame | Partisans reinforce Allied M14 assault | **M14** |

#### The Spectrum Model — Why Three States Matter

Binary tech (have / don't have) creates a checklist. The player either did the mission or didn't. That's what the originals do.

Three-state tech (acquired / partial / denied) creates *stories*:

```
Scenario A: "The Chrono Tank Heist"
  Player sent Tanya on the Italy chain commando path.
  Full stealth extraction. Prototype intact.
  Result: 1 fully functional Chrono Tank with temporal shift.
  M14 briefing: "The prototype is operational. Temporal shift online."

Scenario B: "The Smash and Grab"
  Player used commander alternative on the Italy chain.
  Armored assault. Facility destroyed. Prototype damaged in crossfire.
  Result: 1 Chrono Tank, heavy chassis only. No temporal shift.
  M14 briefing: "Our engineers salvaged what they could. It's a tank,
  not a time machine — but it still hits hard."

Scenario C: "The One That Got Away"
  Player skipped Italy entirely to focus on Poland.
  No Chrono Tank. Southern counterattack remains.
  M14 briefing: "We never made it to Italy. The southern flank is
  unguarded, and the enemy knows it."
```

Each scenario is a different war. The player who replays the campaign and chooses differently gets a genuinely different M14 — not just harder or easier, but *different in kind*. Different units, different threats, different approach routes, different briefing text.

#### Commander Choice and Tech Quality

The commando/commander split is not just a playstyle preference — it is a **tech quality decision**:

| Operation Type | Commando Path Yields | Commander Path Yields |
|---|---|---|
| **Tech Theft** | Full prototype, intact schematics, complete research | Destroyed facility (denial), or damaged prototype (partial acquisition) |
| **Intel Raid** | Full intelligence package (codes, routes, composition) | Partial intel (some shroud reveal, no patrol routes) |
| **Tech Denial** | Clean sabotage + bonus intel from the facility | Loud destruction — denial achieved, but no bonus intel |
| **Rescue** | Hero recovered at full capability + facility intel | Hero recovered but wounded + no facility intel |
| **Faction Favor** | Full faction trust (max reinforcements, best equipment) | Faction grateful but cautious (reduced reinforcements) |

This means a commando-focused player builds a **deeper** arsenal (fewer techs, but at full quality). A commander-focused player builds a **broader** denial portfolio (more enemy techs prevented, but own acquisitions are partial). A mixed player gets the most interesting M14 — some full-quality own tech, some denied enemy tech, and some gaps on both sides.

**No path is wrong. Every path produces a unique war.**

#### Cross-Faction Tech Interaction

The arms race is not two independent tracks. Allied acquisitions and Soviet denials interact:

```
If Allied player captures Chrono Tank AND denies Soviet Super Tanks:
  → M14: Player has temporal-shift armor; enemy has no heavy counter.
  → Easiest endgame armor matchup.

If Allied player captures Chrono Tank but FAILS to deny Soviet Super Tanks:
  → M14: Chrono Tank vs Super Tanks. Both sides have elite armor.
  → The most spectacular endgame — tech titans clash.

If Allied player skips Italy but denies Soviet Super Tanks:
  → M14: No elite armor on either side. Infantry and standard armor war.
  → Classic RA1 feel, but earned through choice.

If Allied player skips Italy AND fails to deny Soviet Super Tanks:
  → M14: Enemy has Super Tanks, player has nothing special.
  → Hardest armor matchup. Requires tactical brilliance or other assets.
```

This is the arms race working as intended. The player is not just building their own army — they are shaping the *entire battlefield*. Every denial operation removes something from the enemy's M14 roster. Every acquisition adds something to the player's. The intersection creates the unique war.

#### War Table — Arms Race Readout

The War Table should surface the arms race as a clear, at-a-glance readout alongside the existing War Momentum display:

```
ARMS RACE STATUS
  Your Arsenal                    Enemy Arsenal
  ────────────                    ─────────────
  Chrono Tank: ACQUIRED (full)    Sarin Gas: DENIED
  Super Tanks: PARTIAL (1, dmg)   Volkov: WEAKENED
  Air Package: ACQUIRED (combined) Soviet Super Tanks: ACTIVE (3)
  Partisans: ACQUIRED (3 squads)  Radar Network: DEGRADED
  Phase Transport: NOT ACQUIRED   Iron Curtain: STANDARD
  Chemical Defense: PARTIAL       Fortifications: REDUCED

  M14 Estimated Matchup: ADVANTAGE (Allied favor)
  ──────────────────────────────────────────────
  Your edge: air superiority, temporal armor, chemical denial
  Their edge: heavy armor (3 Super Tanks), active Volkov (weakened)
  Neutral: standard fortifications, reduced radar
```

This is what makes the player never want to go back. In the originals, M14 is the same mission every time. In IC, M14 is the *culmination of every choice the player made across the entire campaign*. The briefing reads differently. The enemy composition is different. The player's own army is different. The approach routes depend on theater branches. The difficulty is a direct function of operational engagement.

The arms race readout makes this visible and tangible. The player sees their war taking shape — not as an abstract difficulty slider, but as a concrete roster of weapons they earned and threats they prevented.

#### Modder Contract — Custom Tech Chains

```yaml
# Modders can define custom tech acquisition chains
tech_chains:
  chrono_tank_acquisition:
    tech_id: chrono_tank
    display_name: "Chrono Tank Prototype"
    faction: allied

    acquisition_paths:
      - path_id: commando_full
        required_operations: [am_italy_1, am_italy_2, am_italy_3]
        result: full          # full | partial | denied
        roster_add:
          - chrono_tank_full
        briefing_line: "Temporal shift online. The prototype is fully operational."

      - path_id: commander_partial
        required_operations: [am_italy_1_commander, am_italy_2]
        result: partial
        roster_add:
          - chrono_tank_damaged
        briefing_line: "Heavy chassis recovered. No temporal systems — but it still fights."

    denial_state:
      result: denied
      briefing_line: "We never reached Italy. No prototype."

    consumed_by: [allied_12, allied_14]
    war_table_category: "Your Arsenal"

  sarin_denial:
    tech_id: sarin_gas
    display_name: "Chemical Weapons Program"
    faction: soviet   # this is enemy tech the player denies

    denial_paths:
      - path_id: full_denial
        required_operations: [cs_crackdown]
        result: denied
        set_flag: { sarin_denied: true }
        briefing_line: "Sarin facilities neutralized. No chemical threat."

    uncountered_state:
      result: active
      set_flag: { sarin_active: true }
      briefing_line: "Gas barrages inbound. Infantry, mask up."
      mission_effects:
        allied_08:
          add_hazard: [gas_barrage_1, gas_barrage_2, contaminated_lane]
        allied_14:
          add_hazard: [chemical_reserve]

    war_table_category: "Enemy Arsenal"

# Spectrum states are defined per-tech
tech_spectrum:
  states: [full, partial, denied, active, weakened, degraded]
  # 'full/partial/denied' for player acquisition
  # 'active/weakened/denied' for enemy tech
  # 'degraded' for partial denial (e.g., reduced effectiveness)
```

---

## Allied Enhanced Edition — Phase-by-Phase Design

### Phase 0: Prologue (Linear, Tutorial)

No War Table. The prologue serves as both story introduction and learn-to-play sequence, integrated with D065 (tutorial system).

#### The Tutorial Problem: Campaign-Independent Learning

The Stavros prologue missions are Allied storyline content. But a player might start with the Soviet campaign. IC cannot assume Allied-first play order.

**Solution: Each campaign has its own story prologue that doubles as tutorial.**

Both prologues teach the same core mechanics (movement, combat, building, hero abilities) but through their own narrative. A player who completes either prologue is ready for any campaign. A player who plays both gets extra story context but no mechanical advantage.

| Mechanic | Allied Prologue Teaches Via | Soviet Prologue Teaches Via |
|---|---|---|
| Movement, selection, escort | Stavros evacuating civilians | Soviet squad securing a border crossing |
| Attack-move, stances, basic building | Stavros defending evacuation point | Soviet engineers establishing a forward outpost |
| Hero abilities, skill points, hero-as-asset | Tanya arrives as Western support | Early Volkov prototype field test (pre-augmentation officer) |
| The War Table (overview) | Post-prologue tutorial overlay | Post-prologue tutorial overlay (same UI, different briefing text) |

This means the Soviet prologue also introduces Project Lazarus — but from the *Soviet* side. The player runs the early field tests. When an Allied player later discovers the program in Phase 3, both campaign perspectives connect.

#### Allied Prologue — Stavros Arc

Three mandatory missions, each teaching a specific layer through authored on-screen prompts, contextual hints, and forgiving difficulty:

1. **CS "Personal War"** — Stavros guides Greek civilians to safety. **Teaches:** movement, selection, escort, fog of war, basic camera. On-screen prompts explain each mechanic as it becomes relevant. Very Easy. No base building, no combat until the final scripted encounter (flee, don't fight). The player learns to move units and care about them before any shooting starts.

2. **CS "Evacuation"** — Stavros defends an evacuation point under time pressure. **Teaches:** attack-move, unit stances, basic building placement (sandbags, one turret), multi-objective tracking (defend here AND evacuate there), the mission timer as a visible UI element. Easy. First real combat, but enemies arrive in small, predictable waves. On-screen prompts introduce the attack command, stance toggle, and build menu.

3. **IC "Tanya: First Blood"** — Tanya arrives as Western support. **Teaches:** hero abilities, skill point allocation, stealth movement, the ability hotbar, and the concept of hero-as-strategic-asset (Tanya is powerful but losing her has consequences). Easy-Medium. The player sees the hero survival model in action: if Tanya takes heavy damage, the mission shows a "WOUNDED — will recover" message rather than a game-over, teaching the three-tier model before it matters.

**Why Stavros as Allied tutorial protagonist:** Stavros is a resistance leader, not a trained soldier. His perspective is the player's perspective — learning on the job, improvising, surviving. This makes the tutorial feel like story, not instruction. By the time Tanya arrives in mission 3, the player has context for why a professional operative is different from what they've been doing.

#### Soviet Prologue — Early Expansion Arc

Three mandatory missions mirroring the Allied prologue's mechanical teaching through Soviet narrative:

1. **IC "Border Incident"** — A Soviet rifle squad secures a contested border crossing in Eastern Europe. **Teaches:** movement, selection, escort (guarding a political commissar), fog of war. Very Easy. The player sees the Soviet expansion from ground level — not grand strategy, but boots on the ground.

2. **IC "Forward Base"** — Soviet engineers establish an outpost in newly "liberated" territory. **Teaches:** attack-move, stances, base building (barracks, power, basic defenses), resource collection, multi-objective (build AND defend against partisans). Easy. Introduces the Soviet building style and economy.

3. **IC "Test Subject"** — A pre-augmentation Soviet officer (the future Volkov) leads a commando raid as a field evaluation. **Teaches:** hero abilities, skill points, the ability hotbar, hero-as-strategic-asset. Easy-Medium. The officer is skilled but human — the cybernetic augmentation has not happened yet. The mission ends with a briefing note: "Subject approved for Project Lazarus." This plants the seed that pays off in both campaigns.

**Why this works narratively:** The Soviet prologue shows the early war from the aggressor's side — not cartoonish villainy, but soldiers following orders in a machine they don't fully understand. Mission 3's Project Lazarus tease creates cross-campaign dramatic irony: Soviet players saw its origin; Allied players will discover and try to stop it.

#### Transition to the War Table

After either prologue, the M1 briefing cutscene plays (Allied) or the Soviet M1 equivalent. The War Table appears for the first time with a brief tutorial overlay. The overlay text differs per campaign but teaches the same UI: operation cards, enemy initiatives, asset ledger, war momentum.

---

### Phase 1: First Contact

**Main Mission:** M1 "In the Thick of It" — Rescue Einstein. Medium.

**Operational Budget:** 1 turn (training wheels for the War Table)

**Operation Pool:**
- IC "Supply Line Security" — Tanya escorts Chrono-tech convoy. Asset: +1 Chrono support platoon in M4.
- Generated SpecOps: `intel_raid` in Greece — scout Soviet positions. Asset: partial shroud reveal in M2.

**Enemy Initiative:** None (Phase 1 is gentle)

**Phase Trigger:** M1 becomes available immediately. The player can do one operation first or launch M1 directly.

**Teaching moment:** The War Table tutorial explains: "You can pick an operation before the main mission. Operations give you assets for the war ahead."

---

### Phase 2: Securing the Bridgehead

**Main Missions:** M2 "Five to One" + M3 "Dead End" (played in sequence)

M2 clears the supply route; M3 destroys bridges. M3 has spectrum outcomes (all destroyed / some / none / Tanya captured-or-wounded). These two are tightly connected narratively and play back-to-back with a short debrief between them.

**Operational Budget:** 1 turn (before M2)

**Operation Pool:**
- CS "Fresh Tracks" (Siberian 1) — Open second front. Asset: Siberian theater unlocked.
- Generated SpecOps: `faction_favor` in Greece — Contact resistance cell. Asset: Greek resistance favor (partial shroud reveal in M4).

**Enemy Initiative:** *Troop Buildup* — Soviet forces reinforcing the pass. If uncountered: M4 gets +1 enemy wave. No direct counter available this phase (absorbed cost — teaches the player that some initiatives just happen).

**Phase Trigger:** M2 becomes urgent after 1 operational turn.

---

### Phase 3: Hold the Line — The Campaign Opens

**Main Mission:** M4 "Ten to One" — Defend the pass. Medium-Hard.

This is where the strategic layer opens up for real. M4's difficulty scales with M3 bridge results AND accumulated assets.

**Operational Budget:** 2 turns

**Operation Pool (3-5 available, pick 2):**
- IC "Behind Enemy Lines" — Tanya infiltrates Soviet facility. Spectrum outcomes: success+escape / success+captured / fail+escape / fail+captured. Asset: Iron Curtain intel for M6. Risk: Tanya capture.
- CS "Crackdown" (Sarin 1) — Neutralize chemical facilities. Asset: Sarin denied. **Counters:** *Chemical Weapons Deployment* initiative.
- CS "Fresh Tracks" (if not done in Phase 2) — Siberian front.
- IC "Spy Network" — Recruit agents behind enemy lines. Asset: spy network (shroud reveal + M6 access codes). Requires: Tanya not captured.
- IC "Lab Rats" (Project Lazarus 1) — Allied intelligence has uncovered a secret Soviet super-soldier program. Raid the research facility and destroy prototype implants. Asset: Project Lazarus delayed. **Counters:** *Project Lazarus* initiative (Phase 3 leg). Risk: high-security facility, Tanya recommended.
- Generated SpecOps: `tech_denial` in Mediterranean — Sabotage radar station. Asset: radar coverage gap for M7.

**Enemy Initiatives (2-3):**
1. *Chemical Weapons Deployment* — Sarin goes active if uncountered. Effect: M8 gains gas attacks. Counter: Crackdown.
2. *Security Sweep* — Soviet counter-intelligence tightens. Effect: future generated SpecOps in Greece have +1 security tier. No direct counter.
3. *Project Lazarus* — Allied intelligence uncovers a secret Soviet super-soldier program. Effect: if uncountered across Phase 3 AND Phase 4, the completed super-soldier (Volkov) is deployed as an enemy unit in M8+. Counter: "Lab Rats" operation.

**Phase Trigger:** M4 becomes urgent after 2 operational turns. Briefing: "Soviet armored division is approaching the pass. We must hold."

**Teaching moment:** First time the player sees Enemy Initiatives. War Table overlay explains: "The enemy is doing things too. Some can be countered by choosing the right operation."

**If Behind Enemy Lines chosen and Tanya captured:** The rescue operation appears in Phase 4's pool. Capture escalation begins.

---

### Phase 4: The Intelligence War

**Main Missions:** M5 "Tanya's Tale" (conditional — only if captured) + M6 "Iron Curtain Infiltration"

If Tanya was captured, M5 appears as a high-priority operation in the pool (not a mandatory main mission). The player can take it immediately or delay, with escalating compromise per the existing capture system.

M6 is always the phase's main mission. Its quality depends heavily on Phase 3 results (intel from Behind Enemy Lines, spy network, etc.).

**Operational Budget:** 2-3 turns

**Operation Pool (varies based on state):**
- M5 Rescue Tanya (if captured) — HIGH PRIORITY marker. Commander alternative: armored assault (Tanya rescued but wounded).
- CS "Down Under" (Sarin 2) — Continue chemical denial chain. Requires: Sarin 1 done. Asset: Greek-theater security, bonus naval detachment in M7.
- AM "Harbor Reclamation" (Italy 1) — Secure Italian harbor. Asset: Mediterranean staging base, opens Italy chain.
- IC "Shut It Down" (Project Lazarus 2) — The super-soldier program has moved to a hardened production site. Destroy it before the subject is activated. Requires: Lab Rats completed OR Project Lazarus still active. Asset: Project Lazarus fully denied. **Counters:** *Project Lazarus* initiative (Phase 4 leg). If Lab Rats was skipped, this is harder (facility is on alert) but still possible.
- Generated SpecOps: `counter_intel` in Eastern Europe — Hunt Soviet informants. Asset: reduce enemy intel advantage.
- Generated SpecOps: `tech_theft` in Mediterranean — Capture prototype equipment. Asset: equipment upgrade for roster.

**Enemy Initiatives:**
1. *Iron Curtain Acceleration* — Soviets advance their Iron Curtain program. Effect: M13/M14 Iron Curtain defenses are stronger if uncountered. Counter: none directly (addressed through later Focused Blast operation). This is a long-fuse initiative — its effect is felt in Act 3.
2. *Project Lazarus — Final Assembly* (only if Phase 3 leg uncountered) — The super-soldier program has entered production. Effect: if still uncountered at end of Phase 4, `volkov_deployed` flag is set and Volkov is activated as an enemy super-unit in M8 onward. Counter: "Shut It Down" operation.
3. *Interrogation Pressure* (only if Tanya captured) — Compromise escalation accelerates. Effect: Tanya's capture timer advances an extra tick. Counter: complete M5 this phase.

**Phase Trigger:** M6 becomes urgent after 2-3 operational turns.

---

### Phase 5: Turning the Tide

**Main Missions:** M7 "Sunken Treasure" + M8 "Protect the Chronosphere"

M7 destroys sub pens (naval combat showcase). M8 is the Act 2 climax — 45-minute Chronosphere defense with dynamic weather. These play in sequence.

**Operational Budget:** 2 turns (before M7)

**Operation Pool:**
- IC "Operation Skyfall" — Air campaign: coordinate strikes on AA. Asset: M8 starts with AA destroyed, fewer Soviet air waves. Type: air_campaign archetype.
- CS "Controlled Burn" (Sarin 3) — Capture Sarin facilities. Requires: Sarin 2. Asset: chemical expertise package for Act 3.
- AM Italy 2-4 chain — "In the Nick of Time" + follow-ups. Asset: Chrono Tank salvage + no southern counterattack. (Multi-operation: if started here, continues in Phase 6.)
- IC "Einstein's Confession" — Narrative reveal. Fragment payoffs. If D078 enabled: time machine unlock. Type: narrative mission.
- Generated SpecOps: `rescue` or `faction_favor` based on state — theater-appropriate operation.

**Enemy Initiatives:**
1. *Air Superiority Push* — Soviet MiG production accelerates. Effect: M8 gets an extra air wave. Counter: Skyfall.
2. *Submarine Resupply* — Sub pens receive reinforcements. Effect: M7 starts with +1 submarine group. No direct counter.

**Phase Trigger:** M7 becomes urgent after 2 operational turns. M8 follows M7 immediately.

---

### Phase 6: The Deep Push

**Main Mission:** M9 "Extract Kosygin"

Spy infiltration + escort. Kosygin's intel shapes M10B. Commander alternative: armored extraction.

**Operational Budget:** 2-3 turns

**Operation Pool (the largest pool — peak strategic complexity):**
- AM "Monster Tank Madness" (Poland 1) — Super Tanks + opens Poland chain. Asset: 2 Super Tanks for M14, Poland arc.
- IC "Air Superiority" — Air campaign: establish dominance. Asset: bombing runs in M12+M14.
- CS "Trapped" (Siberian 2) — Continue Siberian arc. Requires: Fresh Tracks. Asset: Siberian front weakening.
- AM Italy continuation (if started in Phase 5) — complete the Italy chain.
- Generated SpecOps: `intel_raid` or `tech_denial` — theater-appropriate.
- Generated SpecOps: `faction_favor` in Poland/Siberia — resistance contacts.

**Enemy Initiatives:**
1. *Fortification Program* — Soviets harden M12/M14 defenses. Effect: M14 gets additional static defenses. Counter: Air Superiority (bombing weakens fortifications).
2. *Technology Transfer* — Soviets advance their prototype program. Effect: enemy fields upgraded units in Act 3. Counter: tech_denial operation if available.

**Phase Trigger:** M9 becomes urgent after 2-3 operational turns.

**This phase is the peak.** The most operations, the most competing priorities, the most at stake. The player is choosing between Poland, Siberia, Italy, air superiority, and intelligence operations. This is where the strategic layer creates its deepest "commanding a war" moments.

---

### Phase 7: Act 3 Begins

**Main Missions:** M10A "Suspicion" + M10B "Evidence" (three variants: classic / enhanced / siege)

M10A is always the main operation (destroy nuclear silos). After M10A, the player chooses M10B approach. M10B showcases embedded task force (Enhanced variant) and commander alternatives.

**Operational Budget:** 2 turns

**Operation Pool:**
- AM Poland 2-5 continuation — "Negotiations" / "Time Flies" / etc. Requires: Poland 1. Asset: Poland liberated, partisan favor, west-flank entry in M12.
- CS "Wasteland" (Siberian 3) — Final Siberian operation. Requires: Siberian 2. Asset: Soviet M14 reinforcements halved.
- Generated SpecOps: `counter_intel` — Soviet mole hunt. Asset: reduce enemy intel for endgame.
- IC "Joint Operations" (D070, Phase 6b add-on) — Co-op variant of M11. Requires D070.

**Enemy Initiatives:**
1. *Final Preparations* — Soviets entrench around Moscow. Effect: M14 gets additional defensive lines. Counter: Poland chain completion (flanking route bypasses defenses).

**Phase Trigger:** M10A becomes urgent after 2 operational turns.

---

### Phase 8: Naval Supremacy and the Iron Curtain

**Main Missions:** M11 "Naval Supremacy" + M12 "Takedown"

M11 clears the river. M12 destroys Iron Curtain bases. These play in close sequence — the war is accelerating.

**Operational Budget:** 1-2 turns (between M11 and M12)

**Operation Pool:**
- AM "Deus Ex Machina" — Rescue Volkov equivalent / special recovery op.
- IC "Operation Tempest" (Soviet campaign parallel; Allied version) — Pre-invasion bombardment. Asset: M14 coastal defenses damaged.
- Generated SpecOps: last-chance `tech_denial` or `intel_raid` — final asset accumulation.

**Enemy Initiatives:**
1. *Emergency Mobilization* — Soviets call up reserves for Moscow defense. Effect: M14 gets +2 reserve waves. Counter: Siberian chain completion (reserves diverted to Siberian front).

**Phase Trigger:** M12 becomes urgent after 1-2 operational turns. The war waits for no one.

---

### Phase 9: Endgame

**Main Missions:** M13 "Focused Blast" (three variants) + M14 "No Remorse"

M13 has the commander alternative choice (precision / enhanced / siege). M14 is the final assault on Moscow where EVERYTHING ACCUMULATES.

**Operational Budget:** 1 turn (one last shot)

**Operation Pool:**
- The player's final operational choice. Pool depends entirely on campaign state — whatever threads remain open, whatever last-minute opportunities exist.
- Generated SpecOps: one final operation, theater-appropriate.

**Enemy Initiative:** None. The war has reached its climax. No more Soviet initiatives — everything is committed to Moscow's defense.

**M14 Difficulty Scaling (cumulative):**

| Condition | M14 Effect |
|---|---|
| Siberian front collapsed | Soviet reinforcements halved |
| Poland liberated | Polish partisans arrive (+3 squads, +2 Super Tanks) |
| Air superiority secured | Bombing runs available |
| Sarin expertise captured | Chemical defense package |
| Italy secured | No southern counterattack |
| Spy network active | Partial shroud reveal |
| Skyfall + Air Superiority | Combined air package |
| Project Lazarus denied (both legs) | No Volkov encounters in main missions |
| Project Lazarus partially denied | Volkov appears but weakened (no self-repair) |
| Project Lazarus uncountered | Volkov at full strength in M8, M12, M14 |
| Tanya max level | Leads assault squad |
| All Act 1 SpecOps done | Secret approach route |
| No operations done at all | MAXIMUM difficulty — no bonuses, full enemy strength, Volkov at full power |

The War Table shows the M14 difficulty estimate throughout the campaign, updating as assets accumulate. The player always knows how prepared they are.

---

### Epilogue

- IC "Aftermath" — Post-victory moral decisions. Menu scene: sunrise over liberated city.
- "It Came From Red Alert!" — Ant campaign. Uses completion roster. Post-completion unlock.

---

## Soviet Campaign — Outline

The Soviet campaign follows the same phase model with its own operation pools, enemy initiatives (Allied countermeasures), and authored missions. Key differences:

- **Prologue:** Three Soviet-specific tutorial missions (Border Incident → Forward Base → Test Subject) that teach the same mechanics as the Allied prologue but through Soviet narrative. Mission 3 introduces the pre-augmentation Volkov and plants the Project Lazarus seed.
- **Hero:** Volkov (cybernetic super-soldier) instead of Tanya. The player who completed the Soviet prologue has already met him as a human officer — his augmentation mid-campaign carries weight.
- **Narrative thread:** The Red Ledger (political intelligence fragments) instead of Einstein's Burden
- **Enemy Initiatives:** Allied countermeasures — spy infiltrations, resistance uprisings, naval maneuvers, Chronosphere experiments
- **Theater fronts:** France, Spain, Mediterranean, England (instead of Greece, Siberia, Poland, Italy)
- **Endgame accumulation:** M14 "Soviet Supremacy" — invade England, with difficulty shaped by operational choices
- **Cross-campaign irony:** A player who has completed both campaigns sees Project Lazarus from both sides — as the Soviets building their greatest weapon, and as the Allies desperately trying to stop it.

The phase structure mirrors the Allied campaign with 9 phases + prologue + epilogue, using the 14 original Soviet missions as milestones. Each campaign is fully self-contained — a player can start with either and get a complete tutorial and story experience.

---

## Campaign Pacing Analysis

**Total mission count per playthrough (Enhanced Edition):**

| Content | Count |
|---|---|
| Prologue missions | 3 |
| Main missions (original 14) | 14 |
| Operations played (across ~15 operational turns, player does ~8-12) | 8-12 |
| **Total per playthrough** | **25-29** |
| **Total available across all phases** | **~45** |
| **Classic path (main missions only)** | **14** |

At 15-35 minutes per mission, a full Enhanced Edition playthrough is roughly 8-15 hours. A classic-only run is 4-8 hours.

**Pacing curve:**
- Phases 0-2: Linear, tutorial, 1 op turn each. Learning the systems.
- Phases 3-6: Strategic layer fully open. 2-3 op turns. Peak decision density.
- Phases 7-9: War accelerates. 1-2 op turns. Tension builds toward M14.

This mirrors XCOM's emotional arc: desperation → the turn → mastery → climax.

---

## Moddability

The phase model is fully YAML-definable:

```yaml
campaign_phases:
  phase_3:
    title: "Hold the Line"
    main_missions:
      - allied_04
    operational_budget: 2
    operation_pool:
      authored:
        - ic_behind_enemy_lines
        - cs_crackdown
        - cs_fresh_tracks
        - ic_spy_network
      generated:
        families: [tech_denial, intel_raid]
        theaters: [greece, mediterranean]
        count: 1
    enemy_initiatives:
      - chemical_weapons_deployment
      - security_sweep
    phase_trigger:
      type: budget_exhausted
      urgency_text: "Soviet armored division approaching. We must hold the pass."
```

Community modders can:
- Define custom phases with custom pools
- Add authored operations to any phase
- Define custom enemy initiatives
- Adjust operational budgets
- Add generated SpecOps families and theaters
- Create entirely new campaigns using the phase model

---

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Strategic layer feels like homework between RTS missions | Keep the War Table lean (2-5 minute decisions). Operations ARE RTS missions. The strategic layer is about choosing which mission to play, not managing spreadsheets. |
| Too many operations dilute the experience | Tight operational budgets (1-3 per phase). The player always leaves operations on the table. Opportunity cost maintains tension. |
| Generated SpecOps feel like filler | Strict validation, theater-consistent visuals, concrete campaign-specific rewards. Generated ops are short (10-15 min target). Authored ops are the spotlight; generated ops are the variety. |
| Enemy Initiatives feel punishing | Most initiatives affect difficulty modifiers, not campaign-ending penalties. The player can always continue. "Harder" is the cost, not "impossible." |
| Classic-path players feel punished | Classic path is always completable. Difficulty is tuned so M14 at baseline is Hard, not Impossible. Operations make it easier, not operations-absent make it unfair. |
| New players overwhelmed by the War Table | Phase 0-2 are linear with minimal War Table interaction. Full strategic layer doesn't open until Phase 3. Progressive disclosure matches the existing Rule 7. |
| Pacing drags in mid-campaign | Phase 6 is the peak (most operations); Phases 7-9 compress. The war accelerates toward the endgame. No "painting the map" syndrome. |

---

## Narrative Design Lessons from Alternate-History Science Fiction

IC's premise — Einstein erases Hitler, creating an alternate timeline — places it squarely in the alternate-history genre. Research into how successful TV series and films handle this premise reveals patterns directly applicable to IC's campaign, lore, and D078 time machine design.

### Sources and Relevance

| Show / Work | Core Mechanic | IC Relevance |
|---|---|---|
| **Sliders** (1995–2000) | Each episode explores one "what if?" divergence to its logical conclusion | Each Enhanced Edition playthrough = a different "slide" through the war |
| **The Man in the High Castle** (2015–2019) | Grounded alternate history in real historical evidence (Speer's architectural plans) | IC's 22-year vacuum must extrapolate from real 1920s–1940s history, not invent freely |
| **For All Mankind** (2019–present) | One divergence cascades through decades; ripple effects woven into fabric, not exposition-dumped | Einstein's intervention should escalate consequences across campaign phases, not front-load them |
| **Counterpart** (2017–2019) | Two worlds diverge from same point; seven whiteboards of timeline history; small details sell the difference | IC needs a "show bible" for the vacuum period; briefings need specific alternate-timeline flavor |
| **11.22.63** (Stephen King, 2011 / Hulu 2016) | "The past fights back" — well-intentioned intervention creates catastrophe | Einstein's Burden IS this story. The Confession mission = the moment the protagonist sees what they caused |
| **Fringe** (2008–2013) | Parallel universe sold through throwaway details; alternate selves show how circumstances shape people | Briefings should include specific flavor details; dual campaigns = "same person, two lives" drama |

### Seven Lessons

**Lesson 1 — The divergence must be explored, not just stated.** Sliders worked when it pushed "what if?" to its limits. Man in the High Castle grounded its speculation in real evidence. IC's 22-year vacuum (1924–1946) needs specific, concrete events — not just "the Soviets grew stronger." See the Vacuum Period Timeline below.

**Lesson 2 — Small details sell the world.** Fringe's Martin Luther King on the $20 bill. Counterpart's flip phones vs smartphones. IC's briefings and EVA dialogue should include specific alternate-timeline flavor text — not lore dumps, but background texture:

- *"Turn on the radio — the Berlin Philharmonic is performing live from a city that's been at peace for twenty years."*
- *"French intelligence is still calling this a 'border incident.' Paris hasn't seen a military threat since before most of their officers were born."*
- *"The Soviets are fielding vehicles our analysts can't even classify. Twenty years of unchecked industrialization while the West slept."*

**Lesson 3 — Unintended consequences are the emotional engine.** 11.22.63: Jake saves Kennedy and returns to a dystopian future. For All Mankind: Soviets-land-first intensifies the rivalry rather than resolving it. Einstein removes Hitler and creates a worse war. The lesson: keep escalating consequences across campaign phases.

| Campaign Phase | Consequence Escalation |
|---|---|
| Prologue | "At least there was no WWII." The war seems local, manageable. |
| Phase 1-2 | "This is spreading faster than anyone expected." Eastern Europe falls. |
| Phase 3-4 | "They have weapons we never imagined." Superweapons, Project Lazarus. |
| Phase 5-6 | "This is worse than what Einstein tried to prevent." Full continental war. |
| M14 | "And now we end what he started" — shaped entirely by the player's choices. |

**Lesson 4 — Dual perspectives create "Counterpart" drama.** Playing Allied then Soviet (or vice versa) creates the Counterpart experience — the same war, two perspectives. Specific character parallels amplify this: Volkov is your enemy in one campaign and your comrade in the other. Stavros is your mentor in the Allied campaign and a name on an intel report in the Soviet campaign. The same village is liberated in one campaign and conquered in the other.

**Lesson 5 — Each playthrough is a "slide."** Sliders' loop: arrive on new Earth → discover what's different → deal with consequences. IC's Enhanced Edition: start campaign → choices create a unique war → M14 reflects those choices. The War Table IS a sliding machine. D078 Replay Takeover makes this explicit — the player literally "slides" back into a past mission and creates an alternate timeline.

**Lesson 6 — The "show bible" is non-negotiable.** Counterpart's creator used seven whiteboards of diverging timeline history, distributed to the entire cast and crew. IC needs a concise alternate-timeline reference for campaign writers and LLM content generators (D016). See the Vacuum Period Timeline below.

**Lesson 7 — Time travel should be empowering, not punishing.** Outer Wilds: "knowledge is the upgrade." Deathloop: "the loop empowers." Prince of Persia: "rewind is a resource." D078's campaign time machine should make the player feel like a genius for using foreknowledge, not like they're cheating.

### The 22-Year Vacuum — Period Timeline (Show Bible)

This timeline provides the canonical reference for what happened between Einstein's 1924 intervention and the start of the war. Campaign writers, briefing text, EVA lines, and LLM-generated content (D016) should consult this for consistency.

**1924–1928 — Immediate Aftermath.** In the new timeline, Germany's Weimar Republic continues without the Nazi movement. The economic instability of the mid-1920s persists — hyperinflation, political fragmentation — but without Hitler, no single demagogue unites the far right. Germany remains a fractured democracy, slowly stabilizing but never remilitarizing.

**1928–1933 — Stalin Consolidates.** The Soviet Union proceeds as in real history through this period — Stalin's rise, the first Five-Year Plan (1928), forced collectivization, the Ukrainian famine (1932–33). These events predate Einstein's intervention and are unchanged. Without a fascist Germany to compete with, the Western democracies pay even less attention to Soviet internal affairs.

**1933–1939 — The Complacency.** In real history, Hitler's rise forced Western rearmament. Without Hitler: no German rearmament, no Anschluss, no Munich Agreement, no urgent reason for Britain and France to build military capacity. The democracies remain isolationist and complacent. The United States is even more withdrawn — no Pearl Harbor looms on any horizon. Meanwhile, Stalin's Great Purge (1936–38) still decimates the Soviet officer corps, but this weakness is temporary. Without a German invasion to exploit it, the Red Army recovers fully by the early 1940s.

**1933–1939 — What Doesn't Happen.** No Mussolini-Hitler axis (without Hitler, Italian fascism remains regional and eventually fades). No Spanish Civil War as an ideological proxy war (Franco may never rise, or rises without Axis support and fails). No Sino-Japanese War escalation driven by European distraction. No Manhattan Project — nuclear weapons research proceeds as academic physics, not a crash military program. No crash radar development programs. Military technology advances incrementally, not exponentially.

**1939–1945 — The Quiet Years.** The years that "should have been" WWII are peaceful on the surface in Europe. Underneath: the Soviet Union has completed industrialization, rebuilt its officer corps, and faces no credible military opposition. Stalin's territorial ambitions — always present — now face a Europe with no military alliance, no shared identity forged by common struggle, and no recent experience of total war. Eastern European nations (Poland, Czechoslovakia, Hungary, Romania) exist as independent states but have no NATO, no Western security guarantees, and no awareness of the threat.

**1945–1946 — The Squeeze.** Soviet pressure on Eastern Europe intensifies. Not through sudden invasion — through political subversion, border incidents, staged "liberation" campaigns, and economic coercion. Poland and Czechoslovakia are absorbed first, quietly. The West protests diplomatically but has no military mechanism to respond. By early 1946, the new timeline's Einstein observes a world he created — and recognizes it.

**1946 — The Breaking Point.** Soviet forces push into Greece. Greece is Mediterranean, a trade partner, strategically vital. For the first time, Western nations face a direct military threat from the USSR with no buffer states, no prepared defenses, and no alliance structure. The Allied coalition forms — late, improvised, and outgunned. The war begins.

**Key Absences in This Timeline:**

- No nuclear weapons (no wartime crash program — Einstein's temporal/Chronosphere research fills the "miracle weapon" gap)
- No NATO (no shared threat forged the alliance)
- No United Nations (founded after WWII, which didn't happen)
- No Cold War political vocabulary ("iron curtain," "containment," "mutually assured destruction")
- No shared Western identity built through common sacrifice
- No Holocaust (no Nazi regime) — millions of people are alive who weren't in our timeline

**Key Presences:**

- Einstein is alive and working on temporal/Chronosphere research
- Tesla technology (which in RA1 lore is Soviet) developed without wartime diversion
- A Soviet military machine that is fully industrialized, politically unified, and has never been bled dry by an Eastern Front
- A Western Europe that hasn't fielded a major army in a generation

### How the Vacuum Informs Country Identities

Each Allied country's gameplay identity reflects this specific timeline, not generic military tropes:

- **England**: Island mentality amplified by 22 years of no continental threat. Strong navy, reluctant to commit to land wars.
- **France**: Decades of peace bred complacency. Strong industry, weak military readiness. The army is professional but small.
- **Germany**: Weimar democracy that never fell. Industrial powerhouse, politically fractured, never rearmed. Engineering expertise without military application — until now.
- **Greece**: Economically fragile, internal instability, guerrilla tradition from domestic conflicts. The first to face the Soviets; the most desperate.

---

## Related Research (Separate Documents)

Two topics that emerged during the campaign design process have their own dedicated research documents:

- **Character personality profiles:** MBTI-based behavioral constraint sheets for all 8 named characters (Einstein, Tanya, Stavros, Von Esling, Stalin, Nadia, Volkov, Kukov), with pair dynamics and scene applications. See `research/character-mbti-bible.md`.
- **Subfaction / country differentiation:** Allied nations vs. Soviet institutions model, industry research, balance preset integration, and campaign theater bonuses. See `research/subfaction-country-system-study.md`.

---

## Relationship to Existing Design Documents

This adopted direction builds on, rather than replaces, the existing staged campaign design:

- **campaigns.md:** All committed mechanics (graph model, spectrum outcomes, hero capture, generated SpecOps, operation cards, commander alternatives) are preserved and used. The phase model is an organizational layer on top of the existing graph.
- **enhanced-campaign-plan.md:** The design rules (Rules 1-7), narrative framework (Einstein's Burden, Red Ledger, cutscene alignment), and mission content (CS/AM integration, IC originals) are preserved. What changes is the campaign *structure* — from a static branching graph to a phase-based strategic layer.
- **generated-specops-prototype.md:** Unchanged. Generated SpecOps feed directly into the operation pool.
- **single-player.md:** The War Table is an evolution of the "campaign graph as strategic layer" design already described there.
- **character-mbti-bible.md** (new): Full character personality profiles. Informs AI commander personality design (D043) and LLM-generated briefings (D016) but is not a constraint on either.
- **subfaction-country-system-study.md** (new): Full subfaction research. The subfaction model is a D019 extension — Principle 5 already documents the YAML inheritance pattern.

The remaining follow-up is editorial, not architectural: campaign tree presentations such as `enhanced-campaign-plan.md` can be rewritten over time to foreground the phase model instead of older static ASCII graphs. The design rules, narrative content, and mission details carry forward.
