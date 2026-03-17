# Campaign Strategic Depth — Research Study

> **Purpose:** Evaluate targeted additions to IC's campaign strategic layer that increase player agency, investment, and replayability without transforming the game into a 4X or management sim. Draws from XCOM's layered depth model (tactical → strategic → progression) while respecting Red Alert's identity as a fast-paced RTS.
>
> **Date:** 2026-03-17
> **Status:** Adopted (integration into enhanced-campaign-plan.md complete)
> **Complements:** `research/strategic-campaign-layer-study.md` (phase model, arms race, War Table), `src/modding/campaigns.md` (graph engine, persistent state), `src/modding/enhanced-campaign-plan.md` (mission design rules)

---

## The Problem

IC's Enhanced Edition campaign delivers meaningful choices — timed opportunity cost, spectrum outcomes, unit persistence, commander/operative trade-offs. These are genuine and well-designed. But they are choices about *navigating authored paths*, not about *shaping your war*. The player selects from operation pools and mission branches. They don't invest in a strategy, build toward a vision, or feel ownership over the direction of the campaign.

XCOM solves this with a remarkably lean strategic layer: a base with ~12 rooms, a research tree with ~20 nodes, a doom clock, soldier customization, and two main resources (supplies + intel). None of these are complex individually. Together, they create "my campaign" — two players playing the same XCOM campaign build different bases, research different tech, lose different soldiers, and face different endgames.

This direction is not foreign to Command & Conquer — **Westwood were already heading here.** Tiberian Sun's GDI campaign introduced a world map where the player chose which territory to attack next. Red Alert 2's Soviet campaign let you pick which country to invade. These were deliberate experiments in giving the player strategic agency beyond a linear mission sequence. Westwood didn't take it further because scope and timeline constrained them — not because the idea didn't fit. Two decades later, games like XCOM, Total War, and Into the Breach have proven that a lean strategic layer between tactical missions is not just viable but *highly successful* commercially and critically. What we propose here is the natural completion of what Westwood started, informed by twenty years of refinement in other genres.

IC already has most of the raw material. The arms race ledger tracks tech state. Operations produce concrete rewards. Hero progression exists as an optional module. The War Table presents choices between phases. What's missing is **a resource backbone that makes choices feel earned**, a few compounding systems that let the player invest in a direction, and a feedback loop where tactical performance feeds strategic options.

This document proposes ten additions organized in four layers:

1. **Resource Layer** — Requisition, Intel, and Command Authority: the strategic currencies that connect tactical performance to strategic freedom
2. **Investment Layer** — Command Doctrine, Research Tree, and Forward Operating Base: the systems that let the player build a unique campaign identity
3. **Presentation Layer** — The Doomsday Clock, War Table as Living Map, and Ironman Mode: the systems that make the war feel alive and raise the stakes
4. **Narrative Layer** — Enemy Commanders, Intercepted Communications, Commander's War Diary, and Defector Recruitment: the systems that give the war personality and make every campaign a story worth telling — without adding a single second to the War Table

Two cross-cutting principles govern all four layers:

- **Narrative Coherence:** Every cutscene, briefing, dialogue line, and diary entry must be gated by the player's actual campaign state. If a technology was never researched, it does not exist in the narrative. The campaign the player *hears about* is the campaign they *actually played*.
- **Progressive Introduction:** These ten additions are introduced one at a time across Phases 0–3, not dumped on the player at once. Phase 1 shows only Requisition and a deployment slider. The full War Table doesn't appear until Phase 3 — by which point the player has had two phases to learn the vocabulary. Newcomers and returning RA fans alike build fluency through pacing, not tutorials.

---

## Sources

### Primary Sources

These are publicly available design postmortems, developer talks, and documentation that directly informed this proposal's resource and progression models.

- [XCOM: Enemy Unknown — Postmortem (Jake Solomon, GDC 2013)](https://www.gdcvault.com/play/1018247/Everything-I-Learned-About-Level) — Solomon describes how the strategic layer (base, research, doom clock) was designed to be lean enough that players always return to tactical combat within minutes. Directly informed IC's "10 seconds, not 10 minutes" design test for strategic decisions.
- [Civilization V — Designer Notes (Jon Shafer, 2010)](https://www.gamedeveloper.com/design/the-design-of-civilization-v) — Shafer's account of how tech tree branching creates playstyle identity and the "one more turn" compulsion through visible incremental progress.
- [Into the Breach — Design Postmortem (Matthew Davis, GDC 2019)](https://www.gdcvault.com/play/1025766/Into-the-Breach-Design-Postmortem) — How lean between-mission choices with permanent consequences create run identity through small compounding decisions.
- [Jagged Alliance 2 — Source Code (Stracciatella project)](https://github.com/ja2-straea/ja2-straea) — Open-source continuation; mine income → mercenary hiring loop and sector control as strategic map resource model studied at code level.
- [Total War: Warhammer — Campaign Design (GDC 2017)](https://www.gdcvault.com/play/1024134/) — How the campaign map serves as a living strategic display where army composition reflects player investment.
- [FTL: Faster Than Light — Postmortem (Justin Ma & Matthew Davis, GDC 2013)](https://www.gdcvault.com/play/1019382/FTL-Postmortem-Launching-a-Roguelike) — Ship upgrades between encounters that compound into playstyle identity; resource scarcity forcing hard trade-offs.
- [Tiberian Sun — World Map (Westwood, 1999)](https://cnc.fandom.com/wiki/Tiberian_Sun) — GDI campaign's territory selection map as Westwood's first experiment with strategic agency in C&C.
- [Red Alert 2 — Country Selection (Westwood, 2000)](https://cnc.fandom.com/wiki/Red_Alert_2) — Soviet campaign's country invasion choice as Westwood's second strategic agency experiment.
- [Bulletin of the Atomic Scientists — Doomsday Clock (1947–present)](https://thebulletin.org/doomsday-clock/) — The original Cold War urgency symbol. Created to represent how close humanity is to catastrophe ("midnight"). Directly inspired the Doomsday Clock mechanic's name, visual language, and threshold model.
- [XCOM 2 — Avatar Project (Firaxis, 2016)](https://xcom.fandom.com/wiki/Avatar_Project) — The canonical implementation of a doom clock in a tactical-strategic game. Single visible bar, pushable but never eliminable, creates urgency without arbitrary time limits. Directly informed IC's threshold ring design and the "recoverable but never safe" philosophy.
- [Middle-earth: Shadow of Mordor — Nemesis System (Monolith, 2014)](https://en.wikipedia.org/wiki/Nemesis_system) — Persistent enemy commanders who remember encounters, adapt to player tactics, and create emergent rivalry narratives. Directly inspired IC's Enemy Commander system. IC adopts the memory/adaptation concept but uses authored personality profiles (MBTI bible) rather than procedural generation.
- [This War of Mine (11 bit studios, 2014)](https://en.wikipedia.org/wiki/This_War_of_Mine) — Auto-generated narrative diary that records campaign events in character voice. Informed IC's War Diary design — passive, template-driven, tone-shifting narrative that costs zero player time but creates lasting emotional investment.

### Source Quality and Limits

- **GDC postmortems** (XCOM, FTL, Into the Breach, Total War) are the strongest evidence — designers explaining their own systems with hindsight. These directly informed resource count, complexity budget, and pacing decisions.
- **Civilization design notes** are secondary — Shafer's account is public but focuses on 4X design. The "one more turn" compulsion model is transferable; the specific tech tree mechanics are not (too complex for an RTS between-mission layer).
- **Jagged Alliance 2 source code** is primary for the mine-income → hiring loop, but JA2's economy is far more granular than what IC proposes. IC extracts the pattern (territory → income → deployment budget), not the complexity.
- **Westwood precedents** (Tiberian Sun, Red Alert 2) are evidence that strategic agency was an intentional design direction for C&C, not a foreign graft. The games themselves are the primary evidence; the cited wiki pages are secondary reference material (no Westwood design docs survive publicly). Technically the weakest sources in this list, but the strongest for justifying IC's direction within the C&C lineage.
- **Bulletin of the Atomic Scientists** is the thematic source for the Doomsday Clock name and visual language — not a game design reference. The Doomsday Clock is a Cold War icon; using it in a Cold War game is nominative, not derivative. No game design claims are drawn from this source.
- **XCOM 2's Avatar Project** is the primary mechanical reference for the Doomsday Clock design. The cited wiki page is secondary; the game itself is the evidence. The Avatar Project's single-bar, threshold-based, recoverable-but-never-safe model directly informed IC's threshold ring design.
- **Shadow of Mordor's Nemesis System** is the mechanical reference for Enemy Commanders. The Wikipedia article is secondary; the game itself is the evidence. IC adopts the core concept (persistent enemies with memory and adaptation) but replaces Monolith's procedural personality generation with authored MBTI profiles — a better fit for a campaign with a fixed cast of characters.
- **This War of Mine's diary system** informed the War Diary's passive, template-driven approach. The game demonstrated that auto-generated narrative records create emotional investment without requiring player interaction. IC's implementation is simpler (campaign-state templates, not procedural prose).
- **No academic papers or controlled studies** support the specific resource, urgency, or narrative models proposed here. The evidence base is design postmortems and commercial success. This is normal for game design research but should be acknowledged.

### Adoption / Rejection Matrix

| Source | What IC learns from it | What IC does NOT adopt |
|---|---|---|
| **XCOM: Enemy Unknown / XCOM 2** | Lean strategic layer between tactical missions. Two main resources (supplies + intel). Research milestones trigger story missions. Avatar Project doom clock (XCOM 2) — single visible bar, threshold-based, recoverable but never eliminable → directly inspired IC's Doomsday Clock. Base as persistent home. Soldier bonds create attachment. | Complex base management (room adjacency bonuses, power/excavation). Strategic map panic system. Avatar bar's instant-loss countdown (IC's Clock affects difficulty, not game-over). Permadeath as default (IC offers it as a toggle). |
| **Civilization V/VI** | "One more turn" compulsion from visible incremental progress. Tech tree branching creates playstyle identity. Investment compounds over time. | Full sandbox, multiple victory conditions, diplomacy, city management. These don't fit an RTS campaign structure. |
| **Total War series** | Campaign map as a living strategic display. Army composition reflects player investment over many battles. Tech trees give faction identity within a campaign. War chest funding decisions. | Turn-based campaign map movement, province management, tax/trade economy. |
| **Jagged Alliance 2** | Persistent mercenary roster. Mine income → mercenary hiring loop. Sector control on a strategic map. Resource scarcity as the core tension. | Complex inventory/equipment management, hiring economy. |
| **Into the Breach** | Lean between-mission choices with permanent consequences. Each run feels different through small compounding decisions. | Roguelike structure. IC's campaign is authored, not procedural. |
| **FTL: Faster Than Light** | Ship upgrades between encounters compound into playstyle identity. Resource scarcity forces hard trade-offs. Visible sector map with branching paths. | Roguelike permadeath loop. Random encounters. |
| **Shadow of Mordor** | Persistent enemy commanders with memory, adaptation, and emergent rivalry narratives (Nemesis System). Directly inspired IC's Enemy Commanders. | Procedurally generated personalities (IC uses authored MBTI profiles). Orc social hierarchy simulation. Power struggles between enemies as independent system. |
| **This War of Mine** | Auto-generated narrative diary that records campaign events in character voice. Passive, zero player time, creates emotional investment. Informed IC's War Diary. | Survival resource management. Moral choice systems. Civilian perspective (IC is military command). |

---

## The Resource Layer

### Design Philosophy

Every good strategic layer boils down to one feeling: **"I want to do more than I can afford."** In the current IC design, the only constraint is operational budget (1-2 turns per phase) — an abstract designer-assigned number. It doesn't feel earned or managed.

> **Integration Note:** This study successfully replaced the older "operational budget" concept across the codebase (D021, `campaigns.md`, `README.md`). Command Authority now gates operation slots, and Requisition funds them.

The resource layer replaces this with three concrete currencies that the player earns through tactical performance and spends on strategic decisions. The currencies were selected by testing each candidate against two questions: *"Does a Supreme Allied Commander actually manage this?"* and *"Does it create a decision the player wants to think about for 10 seconds, not 10 minutes?"*

**Candidates evaluated and rejected:**
- **Manpower / Readiness** — tracking troop fatigue is HR paperwork, not commanding a war
- **Diplomatic relations** — negotiating with allies is Civilization, not Red Alert
- **Approval ratings** — overlaps with Command Authority; redundant as a separate system
- **Raw materials / supply chains** — too granular; Red Alert abstracts logistics into credits

### Resource 1: Requisition (War Funds)

The player's war chest. A persistent pool of military funding that bridges tactical and strategic decisions.

**How you earn it:**
- **Main mission success** — liberated territory, captured supply depots, spectrum bonus for clean victories (e.g., full victory: +3,000; partial: +1,500; pyrrhic: +500)
- **Successful operations** — seized enemy materiel, salvaged equipment
- **Resource-opportunity missions** — dedicated operations that exist specifically to bolster the war chest (see § Resource-Opportunity Operations below)
- **Efficiency bonus** — unspent in-mission credits partially convert to Requisition at mission end (e.g., 25% of remaining credits). This rewards tactical efficiency without punishing spending.

**How you spend it:**
- **Mission deployment budgets** — before each mission, the player requisitions a deployment amount via a slider. Higher requisition = more starting credits in-mission. Lower = harder start but more saved for strategic options. This is the core pre-mission decision.
- **Operations** — launching a SpecOps costs Requisition (deploying forces isn't free). Authored cost per operation. More ambitious operations cost more.
- **Research funding** — advancing research projects costs Requisition (replaces the abstract "Research Points" model). Einstein's lab needs money, not just willpower.
- **FOB upgrades** — building out command infrastructure costs Requisition.

**The tension it creates:**

> *You have 12,000 Requisition. M07 is a naval assault — you could deploy with 8,000 (comfortable) or 5,000 (tight but doable). The difference funds the Chronosphere research AND the "Skyfall" air operation next phase. But if you underfund M07 and lose ships, you'll be weaker for M11...*

This connects tactical play to strategic planning. Players who are efficient in missions (fewer losses, less spending) earn more back and have more strategic freedom. **Tactical skill feeds strategic options.**

**Why this works for Red Alert:** The credit counter ticking up is already the most satisfying sound in RA. Making credits matter *across* missions extends that dopamine hit to the campaign level.

**Pre-mission Requisition Screen:**

```
DEPLOYMENT REQUISITION — M07: "Sunken Treasure"
═══════════════════════════════════════════════

  War Funds:   14,200 REQ

  Deploy:      ████████████░░░░░░  8,000 REQ
               ◄──── drag ────►

  You get:     8,000 starting credits
               Standard naval detachment
               1 airstrike (FOB: Precision Strikes)

  You keep:    6,200 REQ  (operations, research, future missions)

  Von Esling: "Naval operations are expensive, Commander.
              But we cannot overcommit — the Chronosphere
              program needs funding."

  [ DEPLOY ]
```

### Resource 2: Intel

The Cold War resource. Represents the player's information advantage over the enemy.

**How you earn it:**
- **Spy and intelligence operations** — the primary source
- **Research milestones** (Signals Intelligence project)
- **FOB choice** (Signals Intercept wing generates +1 Intel per phase passively)
- **In-mission captures** — interrogated officers, seized documents, captured comm equipment (authored per mission as bonus objectives)
- **Resource-opportunity missions** — dedicated intel-gathering operations

**How you spend it:**
- **Pre-mission preparation** — spend Intel before a mission to:
  - Reveal enemy starting positions (2 Intel)
  - See enemy composition in the briefing (1 Intel)
  - Partial shroud reveal at mission start (3 Intel)
  - Reveal bonus objective locations (1 Intel)
- **Initiative awareness** — spend Intel to see enemy initiative details before they resolve (2 Intel per initiative)
- **Hidden operations** — some operations are only visible in the War Table pool if you have enough Intel. You can't run an operation you don't know exists.
- **Research acceleration** — certain research projects accept Intel as an alternative input (Signals Intelligence, Counter-Battery Systems)

**The tension it creates:**

> *You have 6 Intel. You could spend 3 to reveal the enemy composition for M08 — is it a chemical attack? An armor push? You don't know. Or you could spend 2 to uncover a hidden Mediterranean operation that might yield the naval detachment you need. Or save it all for Phase 6 when you know things get harder...*

Intel is about **choosing where to have clarity and where to accept fog.** You can't know everything. Where you invest your information advantage shapes your campaign.

**Why this works for Red Alert:** Fog of war is central to RA gameplay. Intel extends that dynamic to the strategic layer. And "intelligence" is peak Cold War — spy satellites, intercepted transmissions, stolen documents. It's thematically bulletproof.

### Resource 3: Command Authority (Passive Track)

Von Esling's trust in you. This is NOT a spendable resource — it's a **consequence meter** that gates operational freedom. It rises with success and falls with failure.

**How it rises:**
- Main mission victories (especially clean spectrum outcomes)
- Successful operations
- Denying enemy capabilities (tech denial)
- Research breakthroughs (milestone completions)
- Pushing the Doomsday Clock away from midnight

**How it falls:**
- Mission failures or heavy losses (pyrrhic victories)
- Ignoring enemy initiatives (the war deteriorating on your watch)
- Hero casualties or captures
- Repeated operation failures
- Doomsday Clock advancing toward midnight

**What it affects:**
- **Operation slots per phase:**
  - Low Authority (0-30): 1 operation per phase
  - Medium Authority (31-65): 2 operations per phase
  - High Authority (66-100): 3 operations per phase
  - This **replaces** the older abstract operational budget concept with something the player *earned*
- **Reinforcement quality** — High authority: Von Esling sends veteran units. Low: green conscripts.
- **Requisition income bonus** — High authority: +15% Requisition from missions (Allied Command channels more resources to a proven commander). Low: -10%.
- **Briefing tone** — Von Esling's attitude reflects authority. High: *"Commander, your track record speaks for itself. Choose your next operation."* Low: *"Your... recent results have not gone unnoticed. Allied Command has restricted your operational scope."*
- **Endgame access** — Very high authority (80+) unlocks an ambitious M14 approach variant with maximum Allied support. Low authority forces a conservative plan with minimal backup.

**The feedback loop:**

Authority creates a virtuous or vicious cycle:
- High authority → more operations → more resources → better missions → higher authority
- Low authority → fewer operations → fewer resources → harder missions → lower authority

But the cycle is recoverable. A player who is struggling can stabilize by succeeding at a main mission (which always grants authority regardless of operational performance). The system pressures without trapping.

**Why this works for Red Alert:** Von Esling is your commanding officer. He's already in the design, already in the briefings. His approval is a natural narrative frame for "how much rope does the player get." It's the XCOM Council satisfaction meter, but with a face and a voice the player already knows.

### Resource Flow Diagram

```
TACTICAL LAYER (RTS mission)
  ├─ Earn Requisition (credits captured, bases secured, efficiency bonus)
  ├─ Earn Intel (documents seized, officers interrogated, comm captured)
  ├─ Performance affects Command Authority (+/- based on outcome)
  └─ Unit survival feeds roster persistence
          │
          ▼
WAR TABLE (strategic layer, ~2 minutes)
  ├─ Spend Requisition on: research, operations, next deployment budget
  ├─ Spend Intel on: mission preparation, initiative awareness, hidden ops
  ├─ Command Authority gates: operation slots, reinforcement quality, income
  ├─ Research progress triggers: cutscenes, milestones, story missions
  ├─ Doomsday Clock shifts: threshold crossings trigger tone/map changes
  └─ Resource-opportunity operations available alongside tactical ops
          │
          ▼
NEXT MISSION (shaped by all of the above)
  ├─ Deployment budget chosen → starting credits
  ├─ Intel spent → shroud reveal, enemy visibility, bonus objectives
  ├─ Research milestones → new units, support powers, passive upgrades
  ├─ Doctrine + FOB → passive bonuses active
  └─ Arms race state → what you have, what they have
```

### Resource-Opportunity Operations

The existing SpecOps system grants tactical advantages (tech, flags, unit unlocks) but not strategic resources. This creates a gap: the player has no way to actively *improve their strategic position* except by performing well in missions they're already committed to.

**Resource-opportunity operations** fill this gap. They are operations on the War Table whose primary reward is Requisition, Intel, or both — rather than tech or tactical advantage. They represent the logistical, intelligence, and economic side of the war that exists alongside combat operations.

**Why these matter:** They give the player a genuine strategic choice — "Do I spend my operation slot on a combat SpecOps that yields a Chrono Tank prototype, or on a supply raid that funds three phases of research?" This is the XCOM "supply raid vs. story mission" tension.

#### Allied Resource Operations

| Operation | Type | Theater | Reward | Cost | When Available |
|---|---|---|---|---|---|
| **"Gold Reserve"** | Requisition | Central Europe | +4,000 REQ (liberate a Swiss bank vault holding Allied war funds frozen since 1924) | 1 operation slot | Phase 3+ |
| **"Convoy Interdiction"** | Requisition | Mediterranean | +2,500 REQ + captured naval supplies (reduce Soviet naval logistics) | 1 operation slot, 1,500 REQ to deploy | Phase 4+ |
| **"Lend-Lease Revival"** | Requisition | North Atlantic | +3,000 REQ + +1,000 REQ per phase for 3 phases (reopen Atlantic supply routes) | 1 operation slot, 2,000 REQ | Phase 5+ (high value but late) |
| **"Radio Free Europe"** | Intel | Central Europe | +4 Intel + hidden operation revealed (tap into resistance communication networks) | 1 operation slot | Phase 2+ |
| **"Defector Extraction"** | Intel | Eastern Front | +3 Intel + 1 enemy initiative fully revealed (extract a Soviet officer with operational knowledge) | 1 operation slot, 1 Intel (to locate the defector) | Phase 4+ |
| **"Satellite Recon"** | Intel | Global | +5 Intel (redirect a reconnaissance satellite for a full intelligence sweep) | 1 operation slot, 2,000 REQ (satellite repositioning costs) | Phase 6+ |
| **"War Bonds"** | Requisition + Authority | Home Front | +2,000 REQ + 5 Authority (successful propaganda campaign rallies public support) | 1 operation slot | Phase 3+ (once per campaign) |
| **"Archaeological Salvage"** | Requisition + Research | Mediterranean | +1,500 REQ + Chronosphere research accelerated 1 phase (recover pre-war Einstein research notes from a bombed university) | 1 operation slot, 2 Intel (to locate the archive) | Phase 3+, requires Chronosphere research started |
| **"Allied Armory"** | Requisition | Western Europe | +2,000 REQ + captured equipment pool refreshed (raid a Soviet-occupied Allied arms depot) | 1 operation slot | Phase 4+, generated variant available |

#### Soviet Resource Operations

| Operation | Type | Theater | Reward | Cost | When Available |
|---|---|---|---|---|---|
| **"Grain Requisition"** | Requisition | Ukraine | +3,500 REQ (redirect agricultural surplus to military production) | 1 operation slot | Phase 3+ |
| **"Factory Conversion"** | Requisition | Urals | +2,000 REQ + +1,000 REQ per phase for 2 phases (convert civilian factories to military production) | 1 operation slot | Phase 4+ |
| **"Gulag Labor"** | Requisition + Authority | Siberia | +3,000 REQ + 3 Authority (mobilize labor for military infrastructure) | 1 operation slot | Phase 3+ |
| **"Sleeper Network"** | Intel | Western Europe | +4 Intel + 1 hidden operation revealed (activate pre-war Soviet agents) | 1 operation slot | Phase 2+ |
| **"Cipher Bureau"** | Intel | Moscow | +3 Intel per phase for 2 phases (crack Allied encryption, ongoing intelligence stream) | 1 operation slot, 2,000 REQ | Phase 4+ |
| **"Politburo Appeal"** | Requisition + Authority | Moscow | +3,000 REQ + 5 Authority (present battlefield successes to secure increased military budget) | 1 operation slot, requires Authority ≥ 40 | Phase 5+ (once per campaign) |

#### Generated Resource Operations

The generated SpecOps system (`research/generated-specops-missions-study.md`) gains two new mission families alongside the existing six:

| Family | Primary Reward | Secondary Reward | Example |
|---|---|---|---|
| **`supply_raid`** | +1,500–3,000 REQ | Captured equipment | Raid enemy supply depot; destroy warehouses or capture trucks |
| **`intelligence_sweep`** | +2–4 Intel | Hidden operation revealed | Infiltrate communications post; tap signals or extract documents |

These use the same generated mission infrastructure (site kits, objective modules, complications) but produce strategic resources as their primary output. They appear in the operation pool alongside tactical SpecOps, giving the player an explicit choice: **combat advantage now, or strategic resources for later**.

#### Defector Operations (Narrative Layer)

Defector recruitment ops (see Addition 10: Defector Recruitment) also appear in the operation pool, competing for the same slots. Unlike resource operations that yield REQ or Intel, defector ops yield unique narrative rewards — a Soviet unit type added to the Allied roster, a scientist who accelerates research, or an intelligence source that reveals enemy initiatives for free. They are the highest-Intel-cost operations in the pool (2–4 Intel), making them a genuine trade-off against mission preparation and initiative reveals.

Defector availability is driven by campaign state: intercept counts, enemy commander morale, and phase thresholds. 1–2 defector ops appear per campaign — rare, high-impact moments. See the defector types table and YAML contract in Addition 10 for the full specification.

#### How Resource Operations Compose With Everything Else

Resource operations compete for the same operation slots as tactical SpecOps. This is deliberate — the slot is the scarcity. A player who runs "Gold Reserve" in Phase 3 is NOT running "Behind Enemy Lines" (Tanya intel). The Requisition funds future research and deployments, but the missed tactical opportunity creates a different M06.

Doctrine biases resource operation availability:
- **Fortress Europe** — "Lend-Lease Revival" and "Allied Armory" appear more frequently (logistical doctrine)
- **Intelligence War** — "Defector Extraction," "Satellite Recon," and defector recruitment ops (Addition 10) appear more frequently
- **Shock & Awe** / **Deep Strike** — fewer resource operations in pool (bias toward combat ops)

Command Authority gates some resource operations:
- "Politburo Appeal" requires Authority ≥ 40 (you need credibility to ask for more money)
- "Lend-Lease Revival" requires Authority ≥ 50 (Allied Command won't fund an underperforming commander)

---

## The Investment Layer

### Addition 1: Command Doctrine

When the War Table first opens (Phase 1, after the prologue), the player selects a command doctrine — a strategic philosophy that shapes their campaign identity. This is the one upfront choice before any resource decisions, framed as Von Esling asking: *"How do you intend to fight this war, Commander?"* The player picks an identity — not a trade-off — before the strategic layer introduces resource scarcity. Doctrine doesn't gate content, change the story, or lock out operations. It **filters operation pools, adjusts rewards, and compounds passive bonuses** so that two players with different doctrines play the same campaign differently.

#### Why It Fits Red Alert

Red Alert multiplayer already has implicit doctrines — an Allied player who opens with Longbows is playing a different game than one who rushes Light Tanks. Doctrine makes the campaign acknowledge what multiplayer players already do: choose a style and invest in it.

Doctrine also composes with the existing subfaction system. England + Air Doctrine is a different feel than Germany + Armor Doctrine. The combination of nation/institution + doctrine creates enough variety for meaningful replayability without exponential content requirements.

#### Allied Doctrines

| Doctrine | Identity | Operation Pool Bias | Passive Bonus | Endgame Shape |
|---|---|---|---|---|
| **Shock & Awe** | Armor-heavy, aggressive tempo | More direct-assault operations, fewer stealth ops | +1 veterancy rank for surviving armor units between missions | M14 with heavy armor spearhead, fewer intel advantages |
| **Deep Strike** | Commando/air, surgical precision | More infiltration and SpecOps operations | Tanya gains XP 25% faster; air assets have +1 sortie per mission | M14 with precision strikes, fewer ground forces |
| **Intelligence War** | Espionage, information advantage | More intel/spy/counter-intel operations | +1 Intel earned per phase passively; enemy initiative details visible 1 phase early | M14 with full battlefield awareness, standard forces |
| **Fortress Europe** | Defensive, attrition, survival | More defensive, denial, and resource operations | +15% Requisition from all sources; surviving structures persist between missions (where map allows) | M14 with fortified forward positions, strong denial portfolio |

#### Soviet Doctrines

| Doctrine | Identity | Operation Pool Bias | Passive Bonus | Endgame Shape |
|---|---|---|---|---|
| **Iron Tide** | Mass armor, overwhelming force | More conquest and assault operations | Conscript production -15% cost; +1 tank per reinforcement wave | M14 with massed armor assault |
| **Shadow Bureau** | NKVD-style espionage and subversion | More sabotage and intelligence operations | +1 Intel per phase passively; enemy defenses revealed on approach | M14 with weakened Allied defenses, information advantage |
| **Tesla Doctrine** | Technology-focused, experimental weapons | More tech research and prototype operations | Research costs -20% Requisition; experimental weapons have +10% effectiveness | M14 with advanced experimental arsenal |
| **People's War** | Quantity over quality, expendable assets | More mobilization, logistics, and resource operations | Reinforcement waves arrive 30s faster; unit replacement is free between missions | M14 with endless waves, expendable but numerous |

#### Design Constraints

- **No doctrine is strictly better.** Each trades depth in one area for weakness in another.
- **No content is doctrine-locked.** All operations remain available regardless of doctrine — doctrine adjusts the *probability* of an operation appearing in the pool and the *magnitude* of rewards, not whether it exists.
- **Doctrine is visible on the War Table.** The player sees their doctrine's name and current bonuses. It's part of their campaign identity.
- **Doctrine is moddable.** Defined in YAML. Community can add doctrines for custom campaigns.

#### YAML Contract

```yaml
doctrines:
  allied:
    shock_and_awe:
      display_name: "Shock & Awe"
      description: "Aggressive armor doctrine. Hit hard, hit fast, don't stop."
      icon: doctrine_shock_awe
      operation_pool_weights:
        direct_assault: 1.5
        infiltration: 0.7
        tech_theft: 1.0
        denial: 0.8
        supply_raid: 0.8          # fewer resource ops
        intelligence_sweep: 0.6
      passive_bonuses:
        - type: veterancy_bonus
          scope: armor
          value: 1              # +1 rank between missions
      resource_modifiers:
        requisition_income: 1.0   # no bonus
        intel_income: 1.0
      endgame_modifier:
        m14_reinforcement_type: heavy_armor_spearhead

    fortress_europe:
      display_name: "Fortress Europe"
      description: "Defensive doctrine. Attrition, denial, economic strength."
      icon: doctrine_fortress
      operation_pool_weights:
        direct_assault: 0.7
        infiltration: 0.8
        denial: 1.5
        supply_raid: 1.4          # more resource ops
        intelligence_sweep: 1.0
      passive_bonuses:
        - type: structure_persistence
          scope: all
      resource_modifiers:
        requisition_income: 1.15  # +15% from all sources
        intel_income: 1.0
      endgame_modifier:
        m14_reinforcement_type: fortified_forward_positions
```

---

### Addition 2: Research Tree — Tech Progression With Milestone Missions

Between missions, the player funds **research projects** using Requisition — representing Einstein's lab budget (Allies) or the Soviet Science Bureau's allocation (Soviets). Projects progress across phases. When a project reaches completion, it triggers a **milestone event**: a cutscene, a briefing reveal, a new unit unlocking, or a full story mission.

This is the single most impactful addition in this document. It transforms the arms race from a passive checklist ("did I grab the operation?") into an active investment system ("I chose to fund this, I watched it develop, and now the payoff mission is here"). The player doesn't just *receive* the Chronosphere — they invest in it across 3 phases, watch Einstein's progress notes between missions, and when it completes, the mission that deploys it for the first time is the reward for their investment.

#### Why It Fits Red Alert

The original Red Alert campaign unlocks tech linearly — mission number determines what you build. This was fine in 1996 but creates zero player agency. Research gives the player control over *which* tech they develop, creating the "my war, my army" feeling without adding the complexity of a full Civilization tech tree.

More importantly: Einstein's lab is canon. Soviet research programs are canon. Tesla's legacy, Chronosphere development, Project Lazarus — these are all research efforts *that already exist in the campaign narrative*. Making them mechanically playable doesn't add lore — it surfaces lore that's already there.

#### How It Works

Research is funded with Requisition at the War Table between phases. Each project has a per-phase cost in Requisition (e.g., 1,500 REQ per phase for a major project, 500 REQ for a minor one). Projects have 2-4 phases of development.

```
WAR TABLE — RESEARCH LAB
═══════════════════════════════════════════════════════

  War Funds:  14,200 REQ        Intel:  6

  ┌──────────────────────────────────────────────────┐
  │ CHRONOSPHERE DEVELOPMENT             Phase 2 / 4 │
  │ ████████████░░░░░░░░░░░░░░░░  Fund: 2,000 REQ   │
  │ "Temporal field stable at lab scale.              │
  │  Einstein requests captured Soviet crystals       │
  │  to extend range to battlefield distances."       │
  │                                                   │
  │ Phase 3 → Einstein briefing cutscene              │
  │ Phase 4 → "Chronosphere" deployment mission       │
  │                                       [ FUND ]    │
  ├──────────────────────────────────────────────────┤
  │ FIELD MEDIC TRAINING                 Phase 1 / 2 │
  │ ██████████████░░░░░░░░░░░░░░░  Fund: 500 REQ    │
  │ "Combat medic program active. First cadre         │
  │  graduates next phase."                           │
  │                                                   │
  │ Phase 2 → Field Medic unit unlocked               │
  │                                       [ FUND ]    │
  ├──────────────────────────────────────────────────┤
  │ ADVANCED ARMOR PLATING               Phase 0 / 3 │
  │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Cost: 1,000 REQ  │
  │ ⚠ Requires: Captured Soviet armor sample          │
  │                                                   │
  │ Phase 2 → +15% armor for all MBTs                 │
  │ Phase 3 → Heavy Tank variant                      │
  │                                       [ LOCKED ]  │
  └──────────────────────────────────────────────────┘

  Research budget this phase:  2,500 REQ allocated
  Remaining War Funds:         11,700 REQ
```

The player clicks "FUND" on the projects they want to advance. Each click deducts the cost from Requisition. They can fund multiple projects in the same phase if they can afford it — but every REQ spent on research is REQ not spent on operations or deployment. **The budget IS the decision.**

#### Research ↔ Arms Race Integration

Research doesn't replace the arms race — it **layers on top of it**. Captured prototypes from operations accelerate research. Denied enemy tech prevents them from researching counters. The two systems compound:

| Arms Race Event | Research Effect |
|---|---|
| Capture Chrono Tank prototype (Italy chain) | Chronosphere research gains +1 free phase of progress |
| Deny Soviet radar network | Soviet counter-research (anti-Chronosphere) is blocked |
| Fail to capture Soviet armor samples | Advanced Armor Plating research is unavailable |
| Complete Spy Network operation | Intelligence research branch unlocks (new projects visible) |
| Run "Archaeological Salvage" resource op | Chronosphere research gains +1 free phase |

This creates a feedback loop: **operations feed research, research unlocks capabilities, capabilities shape future operations.** The player who invests in the Chronosphere across 3 phases and then plays the Chronosphere deployment mission has a fundamentally different emotional relationship to that technology than one who just received it as a linear unlock.

#### Research ↔ Resource Integration

Research competes for Requisition with operations and deployments. This is the core strategic tension:

> *You have 14,200 REQ. Chronosphere Phase 3 costs 2,000. "Skyfall" air operation costs 1,500. M07 deployment needs at least 6,000. That's 9,500 — leaving 4,700 for next phase. But if you skip Chronosphere this phase, it won't be ready until Phase 7, which means no temporal assault in M12...*

Doctrine affects research costs:
- **Tesla Doctrine** (Soviet) reduces research costs by 20%
- **Fortress Europe** (Allied) earns 15% more Requisition, effectively making research relatively cheaper
- Doctrine-affinity projects cost 25% less (round down). A Deep Strike player funds Chronosphere research more efficiently.

Intel can substitute for Requisition on certain intel-flavored projects (Signals Intelligence, Counter-Battery Systems) at a rate of 1 Intel = 500 REQ equivalent. This gives Intel a secondary use and rewards intelligence-focused players.

#### Research Milestones and Story Triggers

**Research completion IS campaign pacing.**

Instead of the campaign graph dictating "Mission 10 introduces the Chronosphere," the research system says "when Chronosphere research reaches Phase 4, the Chronosphere deployment mission becomes available." This means:

- A player who prioritizes Chronosphere research gets it in Phase 6. Their M12 briefing includes temporal displacement as a tactical option.
- A player who deprioritizes Chronosphere gets it in Phase 8 — or never, if they invest elsewhere. Their M12 is a conventional assault.
- Both are valid. Both produce different campaigns. The player's research investment shaped the story.

**Milestone types:**

| Milestone Type | What Happens | Example |
|---|---|---|
| **Unit Unlock** | New unit added to roster | Field Medic training complete → Field Medics available in all future missions |
| **Passive Upgrade** | Existing units gain a bonus | Advanced Armor Plating Phase 2 → all MBTs gain +15% armor |
| **Briefing Reveal** | Einstein/Nadia delivers a narrative update | Chronosphere Phase 3 → cutscene: Einstein demonstrates lab-scale temporal displacement. *"The field holds. But to move a tank... I need more power."* |
| **Story Mission** | A full mission unlocks on the War Table | Chronosphere Phase 4 → "Chronosphere" mission: deploy the prototype in combat for the first time. The mission IS the payoff. |
| **Support Power** | New support ability available in future missions | Air Recon Phase 2 → one free reconnaissance flight per mission |
| **Enemy Intel** | Reveal an enemy capability or weakness | Signals Intercept Phase 2 → Soviet naval patrol routes visible in M7 briefing |

#### The Chronosphere Example (Full Walkthrough)

This illustrates how research milestones drive narrative pacing:

**Phase 3 — Research begins:**
> Player funds "Chronosphere Development" for 1,500 REQ. War Table note from Einstein: *"The temporal mathematics are sound. What I lack is a stable power source. The Soviet Tesla technology may hold the answer — if we could obtain samples."*

**Phase 4 — Operation feeds research:**
> Player runs "Legacy of Tesla" SpecOps (costs 1,500 REQ). Captures Tesla coil schematics. Research notification: *"Chronosphere Development accelerated. Einstein reports: 'The Tesla harmonics... yes. This changes everything. I can see the field geometry now.'"*
>
> Chronosphere advances to Phase 2 automatically (bonus from captured tech).

**Phase 5 — Research milestone (briefing):**
> Player funds Chronosphere for 2,000 REQ. Reaches Phase 3. Cutscene triggers:
>
> *Einstein's lab. A tank-sized platform crackles with blue energy. Einstein adjusts instruments. The air shimmers. A crate on the platform blinks out of existence — and reappears 20 meters away. Einstein stares at his readings. Long pause.*
>
> *"Twelve meters. Last week it was three. The field scales logarithmically with power input. To move a battalion... I would need the output of a small city."*
>
> *He looks at the player through the camera.*
>
> *"Or one Soviet nuclear reactor."*
>
> The War Table now shows a new operation: **"Power Surge"** — capture a Soviet reactor to power the Chronosphere.

**Phase 6 — Story mission unlocks:**
> Player runs "Power Surge" (costs 2,000 REQ — it's an ambitious operation). Captures the reactor. Chronosphere reaches Phase 4. A full deployment mission unlocks: **"Time Enough"** — the first battlefield use of the Chronosphere. The mission briefing is Einstein's most hopeful moment in the campaign:
>
> *"The Chronosphere is... operational. I confess I did not believe we would reach this point. The field will hold for approximately ninety seconds — enough to displace one assault group behind enemy lines. Use it wisely, Commander. I did not build this to fail twice."*
>
> The player deploys the Chronosphere in-mission. It's not a cutscene — they choose where to teleport, when, and which units. The research investment across 3 phases pays off as direct gameplay power.

**Phase 7+ — Compound returns:**
> If the player continues funding, Chronosphere upgrades become available:
> - Phase 5 (1,500 REQ): Extended duration (90s → 180s)
> - Phase 6 (2,000 REQ): Multi-group displacement (2 groups instead of 1)
>
> Each upgrade is a brief War Table notification, not a full mission. The big payoff already happened. Now it's refinement.

#### Allied Research Projects

| Project | Phases | Cost/Phase | Key Milestone | Requires | Doctrine Affinity |
|---|---|---|---|---|---|
| **Chronosphere Development** | 4 | 1,500–2,000 REQ | Phase 4: deployment mission | Tesla samples (operation) | Deep Strike |
| **Advanced Armor Plating** | 3 | 1,000 REQ | Phase 3: Heavy Tank variant | Captured Soviet armor | Shock & Awe |
| **Field Medic Program** | 2 | 500 REQ | Phase 2: Field Medic unit | None (baseline) | Fortress Europe |
| **Air Recon Network** | 2 | 500 REQ | Phase 2: recon flights | None (baseline) | Intelligence War |
| **Tanya Special Training** | 3 | 1,000 REQ | Phase 3: Tanya gains "Saboteur" ability | Tanya alive, Level 3+ | Deep Strike |
| **Chemical Defense Kit** | 2 | 750 REQ | Phase 2: gas masks for infantry | Sarin samples OR Crackdown intel | Fortress Europe |
| **Signals Intelligence** | 3 | 750 REQ or 1 Intel | Phase 3: initiative details visible 2 phases early | Spy Network operational | Intelligence War |
| **Prototype Tank Program** | 3 | 1,500 REQ | Phase 2: Super Tank chassis; Phase 3: full Super Tank | Polish factory access (operation) | Shock & Awe |
| **Naval Modernization** | 2 | 750 REQ | Phase 2: +1 Destroyer per naval mission | None (baseline) | Any |
| **Counter-Battery Systems** | 2 | 500 REQ or 1 Intel | Phase 2: enemy artillery positions auto-revealed | None (baseline) | Fortress Europe |

#### Soviet Research Projects

| Project | Phases | Cost/Phase | Key Milestone | Requires | Doctrine Affinity |
|---|---|---|---|---|---|
| **Project Lazarus (Volkov)** | 3 | 1,500 REQ | Phase 3: full augmentation mission | Test subjects (authored operation) | Iron Tide |
| **Iron Curtain Enhancement** | 4 | 1,500–2,000 REQ | Phase 4: enhanced Iron Curtain deployment mission | Red Ledger fragments (operations) | Tesla Doctrine |
| **Tesla Weapons Program** | 3 | 1,000 REQ | Phase 2: Tesla Tank prototype; Phase 3: deployment mission | Legacy of Tesla operation | Tesla Doctrine |
| **Mass Mobilization** | 2 | 500 REQ | Phase 2: reinforcement waves arrive faster | None (baseline) | People's War |
| **Volkov Augmentation** | 3 | 1,000 REQ | Phase 2: self-repair; Phase 3: heavy armor + mission | Lazarus Phase 2 complete | Shadow Bureau |
| **MiG Prototype** | 3 | 1,500 REQ | Phase 3: advanced air sortie mission | Air research + captured Allied radar | Tesla Doctrine |
| **Demolition Engineering** | 2 | 500 REQ | Phase 2: Demolition Truck unit | None (baseline) | Iron Tide |
| **Counter-Intelligence** | 3 | 750 REQ or 1 Intel | Phase 3: Allied spy network degraded | Security operations | Shadow Bureau |
| **Submarine Missile System** | 3 | 1,000 REQ | Phase 2: Missile Sub; Phase 3: extended range | Naval research chain | Any |
| **Propaganda Division** | 2 | 500 REQ | Phase 2: enemy units have -10% morale (surrender earlier) | None (baseline) | People's War |

#### Design Constraints

1. **Research competes for Requisition.** This is the whole point. Every REQ spent on Einstein's lab is REQ not spent on deploying forces or running operations. The budget IS the decision.

2. **Research never blocks the main campaign.** All 14 main missions per side are playable without any research. Research adds capabilities and story content — it doesn't gate progression. A player who ignores research entirely plays a harder, leaner campaign.

3. **Research milestones are authored content.** Each milestone (cutscene, briefing, mission) is hand-crafted. Generated SpecOps don't trigger research milestones — only authored content does. This keeps narrative quality high.

4. **Unfunded research does not exist in the narrative.** If the player never starts a project, no cutscene, briefing, or dialogue line references it. The Chronosphere is not "something you missed" — it was never built. See § "Narrative Coherence Principle" for the cross-cutting rule.

5. **Operations accelerate research, not replace it.** Capturing a prototype gives a bonus phase of progress — it doesn't complete the project instantly. The player still needs to fund it. This prevents operations from making research feel redundant.

6. **Doctrine-affinity projects cost 25% less.** This rewards specialization without punishing breadth.

7. **The UI is lean.** One panel at the War Table. Project list with progress bars, costs, and milestone descriptions. Click "FUND" or don't. 15-30 seconds to review and decide. The design rule from `strategic-campaign-layer-study.md` § "IC application" holds: *"The War Table should take 2-5 minutes, not 20."*

#### YAML Contract

```yaml
research:
  allied:
    chronosphere:
      display_name: "Chronosphere Development"
      description: "Einstein's temporal displacement project."
      icon: research_chronosphere
      phases: 4
      cost_per_phase: [1500, 1500, 2000, 2000]   # REQ per phase
      doctrine_affinity: deep_strike               # 25% discount

      prerequisites:
        phase_2:
          any_of:
            - flag: tesla_samples_captured
            - research: signals_intelligence >= 2

      milestones:
        phase_2:
          type: briefing_reveal
          cutscene: einstein_chronosphere_lab_demo
          war_table_note: >
            "The field scales logarithmically. To move a tank,
            I need the output of a small city."

        phase_3:
          type: operation_unlock
          operation: power_surge
          war_table_note: >
            Einstein requests a high-output power source.
            New operation available: "Power Surge."

        phase_4:
          type: story_mission
          mission: chronosphere_deployment
          roster_add: [chronosphere_support_power]
          briefing_line: >
            "The Chronosphere is operational. Use it wisely,
            Commander. I did not build this to fail twice."

      upgrades:
        phase_5:
          type: passive_upgrade
          cost: 1500
          effect: chronosphere_duration_extended
          description: "Temporal field duration: 90s → 180s"
        phase_6:
          type: passive_upgrade
          cost: 2000
          effect: chronosphere_multi_group
          description: "Multi-group displacement (2 groups)"

    field_medic_program:
      display_name: "Field Medic Training"
      description: "Combat medic program for frontline healing."
      phases: 2
      cost_per_phase: [500, 500]
      doctrine_affinity: fortress_europe
      prerequisites: {}
      milestones:
        phase_2:
          type: unit_unlock
          roster_add: [field_medic]
          briefing_line: "First medic cadre graduated. Deploying to your command."
```

---

### Addition 3: Forward Operating Base

The War Table includes a persistent **Forward Operating Base (FOB)** — a lean visual representation of the player's command infrastructure that upgrades between phases. 5 slots, each with a binary A/B choice, each costing Requisition to build. The FOB is the player's "home" across the campaign — the persistent element that reflects their strategic identity.

#### Why It Fits Red Alert

Base building is the most Red Alert thing there is. Every RA mission starts with "build a base." The FOB extends this into the strategic layer — between missions, you're also building your command infrastructure. It's the same instinct (invest in structures that pay off later) applied at campaign scale.

#### FOB Slots

| Slot | Option A | Option B | Build Cost |
|---|---|---|---|
| **Intelligence Wing** | *Signals Intercept* — +1 Intel per phase passively | *Field Agents* — One free recon pass per mission (15% shroud reveal at start) | 1,500 REQ |
| **Logistics Hub** | *Extended Supply Lines* — +15% deployment budget efficiency (spend 1,000, get 1,150 in-mission credits) | *Rapid Deployment* — Reinforcements arrive 30s faster in all missions | 1,000 REQ |
| **Air Support Facility** | *Standing CAP* — Air patrol in all missions (auto-intercept one enemy air attack) | *Precision Strikes* — One airstrike call-in per mission (player-directed) | 2,000 REQ |
| **Research & Development** | *Prototype Lab* — Captured tech yields +1 bonus research phase | *Counter-Tech Lab* — Denied enemy tech yields a defensive bonus (partial counter-capability) | 1,500 REQ |
| **Personnel Division** | *Field Hospital* — Wounded heroes recover in 1 mission instead of 2; +500 REQ per phase (reduced medical costs) | *Veteran Cadre* — All surviving units gain +1 veterancy rank between missions | 1,000 REQ |

Total cost to build all 5: 7,000 REQ. The player can't afford all five immediately — they build them across phases as budget allows, creating another layer of prioritization. **Which facility do you build first?**

#### Design Constraints

1. **Binary choices, not upgrades.** Each slot has exactly two options. You pick one and it's built. You can change it once mid-campaign (pays the build cost again). This prevents min-maxing and creates commitment.

2. **FOB costs Requisition to build.** Building facilities competes with research, operations, and deployment — another strand in the budget web. Building the Air Support Facility early (2,000 REQ) means less for research that phase.

3. **FOB is visible.** The War Table shows a simple visual of the FOB with your chosen facilities. It looks like a small base — hangars, comm towers, research labs. It's cosmetic but creates ownership.

4. **FOB compounds with doctrine and resources.** Deep Strike + Precision Strikes + Prototype Lab = a very different strategic profile than Fortress Europe + Extended Supply Lines + Field Hospital. Intelligence War + Signals Intercept generates more Intel passively, compounding the doctrine's information advantage.

5. **5 choices, each a 10-second decision.** Built across phases as budget allows. Not a management game.

#### YAML Contract

```yaml
forward_operating_base:
  slots:
    intelligence:
      display_name: "Intelligence Wing"
      build_cost: 1500
      options:
        signals_intercept:
          display_name: "Signals Intercept"
          description: "+1 Intel per phase."
          effect:
            type: intel_per_phase
            value: 1
        field_agents:
          display_name: "Field Agents"
          description: "15% shroud reveal at mission start."
          effect:
            type: mission_start_reveal
            value: 15

    logistics:
      display_name: "Logistics Hub"
      build_cost: 1000
      options:
        extended_supply:
          display_name: "Extended Supply Lines"
          description: "+15% deployment budget efficiency."
          effect:
            type: deployment_efficiency
            value: 1.15
        rapid_deployment:
          display_name: "Rapid Deployment"
          description: "Reinforcements arrive 30s faster."
          effect:
            type: reinforcement_speed
            value: -30
```

---

## The Presentation Layer

### Addition 4: The Doomsday Clock

The **Doomsday Clock** is IC's central urgency mechanic — a single, always-visible indicator of how close the Soviets are to winning the war. It is displayed prominently on the War Table as an analog clock face, with the minute hand advancing toward midnight. Midnight means Europe falls.

The name is not a metaphor. The [Doomsday Clock](https://thebulletin.org/doomsday-clock/) was created by the Bulletin of the Atomic Scientists in 1947 to represent how close humanity is to catastrophe during the Cold War. It is perhaps the most iconic symbol of Cold War tension — and it belongs in Red Alert's world as naturally as Tesla coils and Chronospheres.

#### Why It Matters

The existing design (strategic-campaign-layer-study.md § "War Momentum") chose a "preparation gauge" over a doom clock: *"the campaign is not about preventing a loss condition — it is about preparing for a known final battle."* That rationale is half right. The campaign IS about preparing for M14. But XCOM's Avatar Project proves that a doom clock's real value is not the loss condition — it's the **urgency**. The Avatar bar makes ignoring things feel dangerous. It makes "I'll deal with that next turn" into a genuine risk calculation. Without that pressure, a player can coast through phases, run the safe operations, and never feel the war closing in.

The Doomsday Clock gives IC's campaign its XCOM-style urgency while remaining true to the "preparation gauge" model. It advances toward midnight when the war deteriorates. It retreats when the player pushes back. It never triggers an instant game-over — but crossing threshold rings changes the war's character in ways the player can feel.

> **Relationship to adopted design:** `strategic-campaign-layer-study.md` § "War Momentum" defines momentum as a distributed readout across the asset ledger. This proposal replaces that with a single unified Clock value that absorbs momentum's effects while adding threshold-based escalation. The underlying data (asset ledger, initiative outcomes, operation results) is unchanged — the Clock is a new lens on the same state.

#### How It Works

The Clock runs from **23:48** (Allied advantage — the war is well in hand) to **00:00 / midnight** (Soviet strategic dominance — Europe is falling). It starts at **23:53** (7 minutes to midnight) — the war is contested, with slight Soviet advantage. The player's goal is never explicitly "stop the Clock" — it's "win the war" — but the Clock creates ambient pressure that makes every decision feel consequential.

```
THE DOOMSDAY CLOCK

            ┌───12───┐
          11           1
        10               2
       9        ●─────▶   3       ◄── 23:56 — Four Minutes to Midnight
        8               4
          7           5
            └───6───┘

    ██████████░░░░░░░░░░░░
    ALLIED          MIDNIGHT
    ASCENDANCY      (Soviet Dominance)
```

#### What Moves the Clock

**Toward midnight (the war deteriorates):**
- Enemy initiatives that resolve uncountered (+1–2 minutes)
- Main mission failures or pyrrhic outcomes (+1–2 minutes)
- Operations skipped when the enemy exploits the gap (+0.5–1 minute)
- Key tech denial failures — the enemy acquires something dangerous (+1 minute)
- Hero casualties or captures (+0.5 minute)

**Away from midnight (the player pushes back):**
- Main mission victories, especially clean spectrum outcomes (-1–2 minutes)
- Successful operations — tactical and resource (-0.5–1 minute)
- Tech denial successes — destroying enemy programs (-1 minute)
- Research milestones reached (-0.5 minute per milestone)
- Counter-initiative operations — directly addressing Soviet threats (-1–2 minutes)

**The Clock is always recoverable.** Unlike XCOM's Avatar Project, which can fill and trigger a countdown, IC's Clock can always be pushed back. A desperate player who pulls off a major victory or a critical denial op can claw back several minutes in a single phase. The feeling is not "inevitable doom" — it's "the tide of war."

#### Threshold Rings

The Clock has four named threshold rings. Crossing a threshold changes the war's character:

| Ring | Time Range | Name | Effects |
|---|---|---|---|
| **Green** | 23:48–23:52 | "Allied Ascendancy" | Ambitious offensive operations available. Requisition income +10%. Von Esling is confident. M14 approach variants include bold flanking options with maximum Allied support. |
| **Yellow** | 23:53–23:56 | "Contested" | Standard operation pool. Normal income. Von Esling is focused. The default state — where most campaigns spend most of their time. |
| **Orange** | 23:57–23:58 | "Darkening" | Desperate "last chance" operations appear (high risk, high reward). Requisition income -10%. Von Esling's tone sharpens. War Table map lighting dims. Enemy forces in upcoming missions receive reinforcement bonuses. |
| **Red** | 23:59–00:00 | "Minutes to Midnight" | Emergency operations only. Requisition income -20%. Von Esling is grim: *"Commander... Allied Command is discussing contingencies. I suggest you give them a reason not to."* M14 becomes maximum difficulty — full enemy arsenal, minimal Allied support, additional defensive lines. The War Table map is dark, front lines are deep into Allied territory, radio chatter is tense. |

**Crossing a threshold is an event.** When the Clock enters Orange, a brief War Table cutscene plays: Von Esling's expression hardens, the map lighting shifts, a radio broadcast crackles with worried Allied dispatches. When it retreats from Orange back to Yellow, relief is palpable — *"Commander, you've bought us time. Use it wisely."* These moments make the Clock feel alive, not just numerical.

**Midnight is not an instant loss.** If the Clock reaches 00:00, the campaign continues — but the player is in crisis. M14 at midnight is brutally difficult, all available forces are committed, the briefing text reflects desperation rather than triumph. It's the "bad ending" approach — winnable, but grueling. The campaign does not end early. The Clock measures *how hard your endgame will be*, not *whether you get one*.

#### Why This Works for Red Alert

The Doomsday Clock is a Cold War icon repurposed as a game mechanic. When Von Esling says *"The Clock stands at four minutes to midnight, Commander"* — every player understands. The metaphor needs no tutorial. It creates urgency without artificial timers, punishes passivity without punishing careful play, and gives the War Table a heartbeat. Every phase, the first thing the player sees is the Clock. Every choice is framed by it.

#### Interaction With Other Systems

- **Command Authority** reflects *Von Esling's trust in you*. The Clock reflects *the state of the war*. A player can have high Authority (Von Esling trusts them) while the Clock is in Orange (the war is going badly) — they're earning more operation slots to deal with a deteriorating situation. Or low Authority with a green Clock — the war is fine but Von Esling thinks someone else should be leading it.
- **Research milestones** push the Clock back. Completing the Chronosphere program is a strategic victory that buys time.
- **Resource-opportunity operations** have minor Clock impact — logistical wins stabilize the front slightly.
- **Doctrine** doesn't directly affect the Clock, but doctrines that emphasize denial (Intelligence War) naturally generate more counter-initiative operations, which push it back.
- **FOB facilities** don't move the Clock directly, but the passive bonuses they provide (Intel generation, reinforcement quality) make operations more successful, which moves the Clock indirectly.

#### YAML Contract

```yaml
doomsday_clock:
  initial_time: "23:53"        # 7 minutes to midnight
  min_time: "23:48"            # maximum Allied advantage
  max_time: "00:00"            # midnight — Soviet strategic dominance
  thresholds:
    green:  { range: ["23:48", "23:52"], label: "Allied Ascendancy" }
    yellow: { range: ["23:53", "23:56"], label: "Contested" }
    orange: { range: ["23:57", "23:58"], label: "Darkening" }
    red:    { range: ["23:59", "00:00"], label: "Minutes to Midnight" }

  # Per-event Clock shifts (in minutes, positive = toward midnight)
  shifts:
    mission_victory_full:      -2
    mission_victory_partial:   -1
    mission_victory_pyrrhic:   +1
    mission_failure:           +2
    operation_success:         -0.5
    operation_failure:         +0.5
    initiative_uncountered:    +1.5
    tech_denial_success:       -1
    tech_denial_failure:       +1
    research_milestone:        -0.5
    hero_captured:             +0.5
    hero_killed:               +1
    resource_op_success:       -0.25

  # Threshold crossing triggers
  on_enter_orange:
    cutscene: "war_table_darkening"
    briefing_tone: "urgent"
    map_lighting: "dim"
  on_enter_red:
    cutscene: "minutes_to_midnight"
    briefing_tone: "desperate"
    map_lighting: "dark"
    radio_chatter: true
  on_retreat_to_yellow:
    cutscene: "clock_pushed_back"
    briefing_tone: "focused"
  on_retreat_to_green:
    cutscene: "allied_ascendancy"
    briefing_tone: "confident"
```

#### Design Constraints

1. **Recoverable, never terminal.** The Clock can always be pushed back. Midnight is the hardest endgame, not a game-over screen.
2. **Transparent.** The player sees every shift and why. The post-phase summary shows: *"Clock moved from 23:55 to 23:57: Soviet Chemical Initiative uncountered (+1.5 min), Operation 'Convoy Interdiction' success (-0.5 min). Net: +1 minute toward midnight."*
3. **Authored, not simulated.** Clock shifts are per-event authored values, not an independent simulation. The designer controls the pace.
4. **Emotionally resonant.** Threshold crossings are events with cutscenes and tone shifts — not silent number changes.

---

### Addition 5: War Table as Living Map

The War Table is a **map**, not a menu. Fronts are visible. Theaters are geographic regions. When the player skips an operation in Poland, the Soviet front line there advances visually. When they succeed in Italy, the southern theater stabilizes. The **Doomsday Clock** is always visible at the top of the War Table. Resource and Intel indicators are always visible in the corner.

#### Why It Matters

This is the single change that transforms the strategic layer from "pick an operation from a list" to "look at where the war is going and decide where to fight." The data already exists — enemy initiatives, theater states, operation outcomes. Making it spatial and visual changes how the player *feels* about their choices. The Clock provides the heartbeat; the map provides the geography.

#### Front Lines

The War Table map shows approximate front lines across Europe:

- **Northern Front** (Scandinavia) — static unless specific operations address it
- **Central Front** (Germany/Poland) — primary theater, most operations affect this
- **Southern Front** (Mediterranean/Italy) — secondary theater, naval operations
- **Eastern Front** (Siberia) — optional theater, only opens if Siberian chain is pursued

Front lines advance or retreat based on phase outcomes. They're **cosmetic** — approximate visual indicators of the war state, not precise simulation. But they make the war *visible*.

#### Resource Display

The map header always shows:

```
  ⏱ DOOMSDAY CLOCK: 23:55 [CONTESTED]     REQ: 14,200 ▲+2,300     INTEL: 6 ▲+2
  AUTHORITY: ████████████░░░  72 / 100  (HIGH — 3 operations available)
```

The Clock is the first element — the player's eye goes there first, establishing the war's urgency before they review resources.

#### Design Constraints

1. **Cosmetic, not simulated.** Front lines are authored positions per phase/state combination, not a real-time wargame simulation. The map shows the *consequence* of choices, not an independent strategic simulation.
2. **Clock and resources are transparent.** The player sees every Clock shift and why. No hidden mechanics.
3. **The map is fast.** Glance at it for 5 seconds, understand the state of the war. Hover for details. Click a theater to see available operations in that region.

---

### Addition 6: Ironman Mode

A campaign mode toggle: `ironman: true`. One save slot, autosaved after every mission and War Table phase. Hero death is permanent — no rescue branches. Failed operations cannot be retried. Decisions are final.

#### Why It Matters

For zero design cost (it's a mode flag that disables specific campaign graph edges), Ironman mode creates XCOM-level stakes. Players who want the forgiving narrative campaign still have it. Players who want consequences get consequences.

#### What Changes in Ironman

| Normal Mode | Ironman Mode |
|---|---|
| Hero captured → rescue branch available | Hero captured → hero lost permanently |
| Hero wounded → recovers in 1-2 missions | Hero wounded → recovers in 2-3 missions |
| Failed operation → can retry next phase | Failed operation → permanently failed |
| Multiple save slots | One autosave slot |
| Reload after bad outcome | Outcome is final |

#### What Doesn't Change

- Campaign graph structure (same missions, same branches)
- Spectrum outcomes (still 3-4 per mission)
- Resources, research, FOB, doctrine (all work identically)
- Difficulty tuning (Ironman is about *permanence*, not *harder enemies*)

#### YAML Contract

```yaml
campaign:
  mode: ironman          # or 'standard' (default)

  ironman_rules:
    save_slots: 1
    autosave: after_every_phase
    hero_death: permanent
    hero_wound_recovery: 3
    operation_retry: false
    reload_on_defeat: false
```

---

## The Narrative Layer

The Resource, Investment, and Presentation layers create strategic depth. This layer creates **narrative depth** — the feeling that the war has personalities, that your campaign has a story, and that the intelligence game is espionage, not bookkeeping. These additions add zero time to the War Table and zero new screens. They work by making existing systems feel richer through authored text, persistent enemy state, and auto-generated narrative.

### Addition 7: Enemy Commanders (Nemesis System)

Soviet operations and initiatives are not abstract threats — they are led by **named commanders** with personalities, memories, and grudges. When the player encounters General Kukov's armored division in Phase 3 and fails to destroy it, Kukov returns in Phase 5 with heavier armor and adapted tactics. If the player humiliated him with a commando raid, he deploys counter-intelligence next time. If the player defeated him cleanly, he's replaced by a less experienced successor.

#### Why It Matters

An RA fan remembers Stavros, Nadia, Von Esling, Stalin. The characters made Red Alert memorable. Enemy Commanders extend that principle to the operational layer — the war has faces, not just force compositions. This creates the emergent "my campaign" stories that make XCOM playthroughs memorable: *"Kukov was the general who killed my Tanya in Hamburg. I chased him all the way to Moscow."*

#### How It Works

Each first-party Enhanced Edition campaign defines 4-6 Soviet commanders with:

- **Personality archetype** (from `character-mbti-bible.md`) — aggressive, cautious, cunning, fanatical
- **Specialty** — armor, infantry, naval, intelligence, chemical/experimental
- **Memory flags** — tracks what the player did to them (defeated, humiliated, ignored, escaped)
- **Adaptation rules** — simple conditional modifiers: if `defeated_by_commando`, then `deploys_counter_intel: true`; if `player_denied_chemical_tech`, then `switches_to_armor`

Commanders manifest through:
- **Enemy initiative cards** — *"General Kukov's 3rd Armored: Push toward Hamburg"* instead of *"Soviet armor initiative"*
- **Briefing text** — Von Esling names them: *"Kukov is back. He's brought T-80s this time. Intel says he's still angry about the bridge."*
- **Mission composition** — a commander's specialty and adaptation state bias the enemy force composition in missions they lead
- **Intercepted communications** (see Addition 8) — commanders have voice in their intercepts, informed by their MBTI profile

Commanders are NOT:
- An AI system — they don't make strategic decisions. They're authored personality profiles that modify existing initiative and mission generation parameters.
- A separate management screen — the player never "interacts" with commanders. They appear in briefings, initiative cards, and mission composition.
- Required — a modder creating a campaign can skip commanders entirely. They're a first-party narrative feature.

#### Commander Lifecycle

```
PHASE 2: Commander Kukov introduced (armor specialty, aggressive)
  → Leads "Push toward Hamburg" initiative
  → Player counters with Tanya commando raid → Kukov HUMILIATED

PHASE 4: Kukov returns (memory: humiliated by commando)
  → Leads "Armored Retaliation" initiative
  → Adaptation: deploys counter-intel patrols, anti-personnel mines
  → Player defeats Kukov conventionally → Kukov DEFEATED

PHASE 6: Kukov replaced by Colonel Volkov (less experienced)
  → Volkov leads weaker armor initiative
  → OR: if player FAILED to defeat Kukov → Kukov PROMOTED
  → Promoted Kukov leads elite initiative with veteran forces
```

#### YAML Contract

```yaml
enemy_commanders:
  kukov:
    name: "General Kukov"
    rank: general
    personality: ESTJ          # references character-mbti-bible.md
    specialty: armor
    introduced_phase: 2
    voice_style: "aggressive, direct, holds grudges"

    memory_flags:
      defeated_by_commando: false
      defeated_conventionally: false
      humiliated: false         # e.g., base destroyed, forced retreat
      ignored: false            # player never engaged his initiative
      promoted: false           # survived long enough to gain rank

    adaptations:
      - condition: "humiliated"
        effect: { deploys_counter_intel: true, patrols_doubled: true }
      - condition: "defeated_by_commando AND NOT defeated_conventionally"
        effect: { anti_personnel_priority: high, mines: true }
      - condition: "ignored"
        effect: { force_strength: "+25%", morale: high }

    lifecycle:
      on_defeated_twice: "replaced_by_successor"
      on_survived_to_phase_7: "promoted"
      successor: "volkov"

  volkov:
    name: "Colonel Volkov"
    rank: colonel
    personality: ISTP
    specialty: armor
    introduced_phase: null      # only appears as Kukov's successor
    voice_style: "quiet, methodical, underestimated"
```

#### Design Constraints

1. **4-6 commanders per campaign.** Enough to feel like a war with personalities, few enough to remember names.
2. **Commanders are authored, not generated.** Their personalities, specialties, and adaptation rules are hand-written. The procedural element is limited to which memory flags get set based on player actions.
3. **Zero War Table time.** Commanders appear in briefings and initiative cards. The player never manages or interacts with them directly.
4. **Mod-extensible.** Community campaigns can define their own commander rosters via YAML.

---

### Addition 8: Intercepted Communications

When the player spends Intel at the War Table to reveal enemy initiative details, they receive **intercepted enemy communications** — not stat blocks. The same information (enemy composition, initiative timing, threat level) is conveyed through in-character transmissions that reflect the sending commander's personality.

#### Why It Matters

Intel spending currently reveals data: *"Soviet Chemical Weapons Program: Phase 2, deploys Phase 5."* Intercepted communications reveal the same data as narrative: *"Intercepted transmission from Dr. Grigor, Science Bureau: 'The compound is stable. Field trials in three weeks. Hamburg is the preferred test site. Tell Kukov to hold the line until then.'"*

Same information. Different feeling. One is a spreadsheet. The other is espionage.

#### How It Works

Each enemy initiative, commander action, and arms race event has an associated **intercept template** — a short text block (2-4 sentences) that conveys the mechanical information through in-character voice. Templates reference:

- The **commander** leading the initiative (personality from MBTI bible)
- The **arms race state** (what tech they're developing or deploying)
- The **Doomsday Clock** position (tone shifts based on how the war is going)
- **Player memory flags** (if the player denied their chemical program, the intercept reflects frustration)

Intercepts appear as a small overlay on the War Table when the player clicks "REVEAL" on an initiative card (costing Intel). They're skippable — a quick reader absorbs them in 5 seconds. A lore fan savors them. The mechanical information is always visible separately as a tooltip or stat line.

#### Intercept Examples

**Standard reveal (1 Intel):**
> *Intercepted — General Kukov to Moscow Command:*
> *"The 3rd Armored is in position. We move on Hamburg at dawn, Tuesday. Tell Volkov his infantry better keep up this time."*
> **[Translates to: Armor initiative, Hamburg theater, activates next phase, combined-arms composition]**

**Deep reveal (2 Intel — shows exact composition):**
> *Intercepted — Dr. Grigor, Science Bureau, to General Kukov:*
> *"The Tesla prototypes are ready for field deployment. Four mobile coils. I suggest the northern approach — their defenses are thinnest there. Do NOT waste them on a frontal assault. These are irreplaceable."*
> **[Translates to: 4x Tesla Tanks in next mission, will approach from north, high-value targets — destroying them denies Tesla Tank tech]**

**Post-denial intercept (free, triggered by player action):**
> *Intercepted — General Kukov to Moscow Command:*
> *"The laboratory is gone. Their commando — the woman — she destroyed everything. Grigor is dead. We have nothing. I need reinforcements, not scientists."*
> **[Translates to: Chemical Weapons Program denied. Kukov switching to conventional forces.]**

#### Integration With Enemy Commanders

Intercepted communications are the primary way commanders express personality between missions. A cautious commander's intercepts are measured and professional. An aggressive commander's are blunt and impatient. A fanatical commander's are ideological and grandiose. The MBTI bible provides the voice constraints.

When a commander remembers a previous encounter with the player, intercepts reference it:
> *"...their commando destroyed our Hamburg facility. I will not make the same mistake. Triple the perimeter guard. Mine every approach."*

#### YAML Contract

```yaml
intercept_templates:
  kukov_armor_push:
    commander: kukov
    initiative: armor_push_hamburg
    intel_cost: 1
    text: |
      Intercepted — General {commander.name} to Moscow Command:
      "The {commander.specialty} division is in position. We move on
      {initiative.theater} at dawn. {adaptation_line}"
    adaptation_lines:
      default: "Tell Volkov his infantry better keep up this time."
      if_humiliated: "This time there will be no commando tricks. I have
        counter-intelligence teams on every approach."
      if_player_denied_chem: "Without Grigor's weapons, we do this the
        old way. More tanks. More shells."
    reveals:
      composition: "armor-heavy, combined-arms"
      timing: "next phase"
      theater: "hamburg"
```

#### Design Constraints

1. **Never blocks gameplay.** Intercepts are an overlay, not a modal dialog. Click to read, click to dismiss, or ignore.
2. **Mechanical info always available separately.** The intercept is narrative sugar. The stat line is always there.
3. **Authored per initiative.** Each first-party initiative has 1-3 intercept variants. Generated SpecOps don't need intercepts (they're tactical, not strategic).
4. **Modders can skip them.** If a community campaign doesn't define intercepts, initiative reveals show the standard stat-block format.

---

### Addition 9: Commander's War Diary

An auto-generated narrative record of the campaign that writes itself from campaign state. The player never creates or edits it — it appears as a readable document accessible from the War Table, updated after every phase.

#### Why It Matters

When an RA fan finishes a campaign, they have a story to tell. The War Diary writes that story down. *"Phase 3 — Lost Hamburg. Miller's 3rd Armored held the retreat. Two tanks survived, both veteran. Chronosphere project at 60%. The Clock stands at four minutes to midnight."*

This creates the "campfire story" effect — when players share their IC campaigns, they share their War Diary. Two players compare diaries and discover their wars were completely different. The diary costs nothing during gameplay (it's passive, auto-generated) but creates lasting emotional investment in the campaign.

#### How It Works

After each phase, the diary system reads campaign state and generates a brief entry (3-6 sentences) covering:

- **Mission outcomes** — what happened, who was lost, what was gained
- **Doomsday Clock** — current position and any threshold crossings
- **Key decisions** — which operations were chosen (and which were not)
- **Commander encounters** — any nemesis commander activity
- **Research milestones** — breakthroughs reached
- **Roster changes** — notable unit losses or veterancy achievements
- **Tone** — shifts based on Clock position (green entries are confident, red entries are desperate)

The diary is:
- **Viewable from the War Table** via a "War Diary" tab or button
- **Exportable** at campaign end as a text/markdown file — a unique campaign narrative
- **Purely passive** — the player never writes or edits it. Zero time cost.

#### Example Entries

**Phase 3 (Clock: Yellow — 23:55)**
> *Day 47. We held the line at Hamburg, but only barely. General Kukov's 3rd Armored hit us at dawn — heavier than intelligence predicted. Miller's tank platoon bought time for the evacuation. Two survived. Both crews received field commendations.*
>
> *At the War Table, I chose Radio Free Europe over the Sarin denial. Four new Intel sources. The Chemical Weapons Program continues unchecked. I hope I don't regret that.*
>
> *Einstein reports the Chronosphere prototype is 60% complete. He needs more funding. The Clock stands at five minutes to midnight.*

**Phase 6 (Clock: Orange — 23:57)**
> *Day 112. The Clock crossed into Darkening today. Von Esling's expression said everything his words didn't.*
>
> *Kukov is back — promoted to Theater Commander after his Hamburg victory. He's brought T-80s. Intel says he's consolidated the entire northern front under his command. I should have dealt with him when I had the chance.*
>
> *The Chronosphere is ready. Einstein's demonstration was... extraordinary. If we can deploy it, this war changes. Power Surge is greenlit. Everything depends on the next operation.*

**Phase 9 (Clock: Green — 23:50)**
> *Day 198. The final briefing. Von Esling shook my hand — first time in the entire campaign. "Whatever happens tomorrow," he said, "you gave us a chance."*
>
> *My order of battle: 3rd Armored (veteran), Chronosphere strike team, partisan diversions on the northern flank. Kukov's replacement, Colonel Volkov, commands Moscow's defenses. He's methodical but inexperienced. We have the initiative.*
>
> *The Clock reads ten minutes to midnight. The best it's been all war. Tomorrow, we end this.*

#### YAML Contract

```yaml
war_diary:
  enabled: true                # can be disabled by modders
  format: markdown             # or plaintext
  exportable: true             # player can save as file at campaign end
  tone_source: doomsday_clock  # diary tone follows Clock threshold

  # Template structure — one entry per phase
  entry_template:
    sections:
      - mission_outcomes        # what happened in combat
      - clock_status            # current time + any threshold crossings
      - key_decisions           # operations chosen / skipped
      - commander_encounters    # nemesis activity
      - research_progress       # milestone updates
      - roster_changes          # notable losses or promotions
    max_sentences: 8
    tone_by_threshold:
      green: "confident, forward-looking"
      yellow: "focused, businesslike"
      orange: "tense, second-guessing"
      red: "desperate, clinging to hope"
```

#### Design Constraints

1. **Zero time cost.** The diary generates itself. The player reads it when they want, or never.
2. **Template-driven.** Entries are filled from campaign state, not generated by LLM. Works without any AI. (An optional LLM enhancement could enrich the prose, but the base system is pure templates.)
3. **No gameplay impact.** The diary changes nothing. It's a record, not a mechanic.
4. **Exportable.** The campaign's unique story is a shareable artifact.

---

### Addition 10: Defector Recruitment

A special operation category where the player spends Intel and an operation slot to recruit disillusioned Soviet personnel. Defectors provide unique rewards — a Soviet unit type in the Allied roster, a scientist who accelerates research, or an intelligence source who reveals enemy initiatives for free.

#### Why It Matters

Defection is *the* Cold War narrative. Oleg Penkovsky, Vasili Mitrokhin, Arkady Shevchenko — the real Cold War was shaped by people who switched sides. In Red Alert's world, this means a Soviet tank commander who's seen enough, or a scientist who believes in Einstein's vision, or a GRU officer disgusted by Stalin's purges.

Defector ops slot into the existing operation framework. The player already picks operations at the War Table. A defector mission looks like every other op — one slot, one Intel cost, one reward. The reward is unique, but the interaction is identical. An RA fan wouldn't register this as a new mechanic.

#### How It Works

Defector opportunities appear on the War Table when conditions are met:

1. **Discovery** — the player's Intel level (or a specific intercept) reveals a potential defector
2. **Availability** — the defector op appears in the operation pool, competing for the same slots as everything else
3. **Execution** — the player runs the op (an RTS extraction mission — breach the safe house, extract the defector under fire, get to the exfil point)
4. **Outcome** — spectrum results:
   - **Full success:** Defector extracted safely → full reward
   - **Partial success:** Defector extracted but wounded/compromised → partial reward, Intel leak
   - **Failure:** Defector captured or killed → reward lost permanently, Doomsday Clock advances (Soviets tighten internal security)

#### Defector Types

| Type | Example | Reward | Intel Cost | Available |
|---|---|---|---|---|
| **Military defector** | Tank Commander Petrov | Soviet unit type added to roster (e.g., Mammoth Tank or Tesla Tank usable by Allies) | 3 Intel | Phase 4+, requires 2+ military intercepts |
| **Scientific defector** | Physicist Dr. Kovalev | Research project accelerated by 1 phase | 2 Intel | Phase 3+, requires active research project |
| **Intelligence defector** | GRU Officer Orlov | 1 enemy initiative revealed for free per phase for 3 phases | 4 Intel | Phase 5+, requires High authority |
| **Political defector** | Deputy Minister Sarin | +8 Authority, Doomsday Clock -1 minute (propaganda victory) | 3 Intel + 2,000 REQ (extraction costs) | Phase 6+, once per campaign |

#### Interaction With Other Systems

- **Enemy Commanders:** If a commander's forces are demoralized (defeated twice, or their tech program was denied), a defector from their unit may become available. Kukov's humiliation might trigger a defector op: *"One of Kukov's officers wants out. He says the general's recklessness will get them all killed."*
- **Doomsday Clock:** Successful defections push the Clock back slightly (-0.5 minute) — a propaganda and intelligence victory. Failed defections advance it (+0.5 minute) — the Soviets tighten security.
- **Arms race:** A military defector can shift the tech ledger — the player gains a Soviet unit type that the enemy still has, creating asymmetric compositions.
- **Intel economy:** Defectors are the highest-Intel-cost operations. Running a defector op means NOT spending that Intel on mission preparation or initiative reveals. The Intel scarcity makes defection a genuine trade-off.

#### YAML Contract

```yaml
defector_operations:
  petrov_extraction:
    type: military_defector
    name: "Operation LOOKING GLASS"
    description: "Extract Tank Commander Petrov from behind Soviet lines"
    intel_cost: 3
    req_cost: 1500
    operation_slots: 1
    available_phase: 4
    prerequisites:
      - "intercept_count >= 2"
      - "kukov.memory_flags.defeated_by_commando OR authority >= 60"

    outcomes:
      full_success:
        reward: { unit_unlock: "mammoth_tank_allied", clock_shift: -0.5 }
        diary_entry: "Petrov is ours. His knowledge of Soviet armor tactics
          is invaluable — and the Mammoth Tank blueprints he carried are
          already in production."
      partial_success:
        reward: { unit_unlock: "mammoth_tank_allied_damaged", clock_shift: 0 }
        diary_entry: "Petrov made it out, but barely. The blueprints are
          incomplete. Engineers say they can work with what we have."
      failure:
        consequence: { clock_shift: +0.5, soviet_security: tightened }
        diary_entry: "Petrov is dead. The KGB got to him first. Soviet
          internal security has doubled across all fronts."
```

#### Design Constraints

1. **1-2 defector ops per campaign.** Rare, high-impact moments — not a production line.
2. **Uses existing operation framework.** Same slot, same War Table UI, same RTS mission format. Zero new screens.
3. **Failure is permanent.** A dead defector doesn't come back. This raises the stakes without adding complexity.
4. **Interacts with commanders.** Defector availability can be triggered by nemesis commander state, creating narrative coherence.

---

## Narrative Coherence Principle

**Every cutscene, briefing, dialogue line, diary entry, and mission reference must be gated by the player's actual campaign state.** If the player never researched the Chronosphere, they never see the Einstein lab-demo cutscene, never receive a briefing mentioning temporal displacement, never play the Chronosphere deployment mission, and never hear a single line about temporal fields. The Chronosphere does not exist in their campaign. Their M14 is a conventional assault — and the briefing, the objectives, the enemy dialogue, and the War Diary all reflect that.

This principle applies universally:

| System | What Gets Gated | Example |
|---|---|---|
| **Research milestones** | Cutscenes, briefing text, story missions, unit unlocks, support powers | No Chronosphere research → no Einstein temporal dialogue, no "Time Enough" mission, no Chrono Tank in roster |
| **Briefings** | All tech references, tactical options, approach suggestions | M12 briefing only mentions temporal assault if Chronosphere is Phase 4+; otherwise presents conventional options |
| **Enemy commanders** | Adaptation lines, intercept references | Kukov doesn't taunt about "your stolen Tesla data" if the player never ran "Legacy of Tesla" |
| **Intercepted communications** | Intelligence content, adaptation lines | No intercept mentions the Chronosphere threat unless the player has active Chronosphere research |
| **War Diary** | Milestone entries, phase summaries, roster mentions | Diary never records "Chronosphere at 60%" if the project was never funded |
| **Defectors** | Roster composition, briefing references | M14 briefing doesn't mention "Petrov's Mammoth Tank" if Petrov was never extracted |
| **Doomsday Clock cutscenes** | Threshold events reference only the player's actual strategic position | Von Esling's Orange-threshold speech references the player's actual research and denial portfolio, not a generic script |
| **M14 (endgame)** | Briefing, objectives, enemy composition, approach routes, difficulty | Assembled entirely from campaign state — different every playthrough because different things actually happened |

### Implementation

Briefings, cutscenes, and dialogue use **conditional blocks** gated by persistent state flags:

```yaml
briefing:
  m12:
    base_text: "Commander, the final push on the eastern corridor begins tomorrow."
    conditional_blocks:
      - condition: "research.chronosphere.phase >= 4"
        insert: "Einstein's Chronosphere is operational. We can displace one
          assault group behind their lines — ninety seconds, Commander. Use it."
      - condition: "research.chronosphere.phase >= 2 AND research.chronosphere.phase < 4"
        insert: "Einstein reports the Chronosphere prototype is progressing, but
          it won't be ready for this operation. We proceed conventionally."
      # If chronosphere.phase == 0 or undefined: no mention at all.
      - condition: "defectors.petrov.status == 'full_success'"
        insert: "Petrov's intelligence on Soviet armor formations gives us
          a clear picture of their defensive positions."
      - condition: "enemy_commanders.kukov.status == 'defeated'"
        insert: "Kukov's replacement, Colonel Volkov, is less experienced.
          Our forward scouts report hesitation in his defensive arrangements."
      - condition: "enemy_commanders.kukov.status == 'active'"
        insert: "General Kukov commands the eastern defenses personally.
          Expect aggressive counterattacks."
```

The same pattern applies to cutscene selection (play cutscene A if state X, cutscene B if state Y, skip if neither), enemy commander dialogue (adaptation lines reference only observed player behavior), and War Diary templates (only emit lines for state that exists).

### Why This Matters

A player who never funded Einstein's lab should experience a campaign where the Chronosphere was never built — not a campaign where the Chronosphere exists but they missed it. The absence of a technology is a valid campaign identity. The player who went all-in on armor should hear about armor. The player who built a spy network should hear about intelligence. The narrative reflects what the player actually did, creating coherence between their choices and the story the game tells.

This is not optional polish. **Incoherent narrative breaks immersion harder than no narrative at all.** A cutscene showing a Chronosphere the player never built tells the player their choices didn't matter. A briefing mentioning a defector they never recruited breaks trust. Every such reference is a lie that undermines the campaign's core promise: *this is your war.*

---

## How Everything Composes

```
PHASE 0: PROLOGUE (pure RTS — no strategic systems)
  Three scripted missions. Player learns unit control, base building, combat.
  Resources accumulate invisibly in the background.

PHASE 1: WAR TABLE OPENS
  Choose Doctrine (identity — "How will you fight this war?")
  Choose Subfaction (flavor — nation or institution)
  │
  │  WAR TABLE (stripped down — ~30 seconds)
  │  ├─ View: front-line map + Requisition counter
  │  └─ Set: deployment budget for M02 (one slider, one decision)
  │
  │  POST-MISSION: one operation becomes available (single slot)
  │
  Systems active: Requisition, Deployment Budget, Operations (1 slot)

PHASE 2: CONTEXT LAYER (systems shown, not yet controlled)
  │  WAR TABLE (~1 minute)
  │  ├─ View: Requisition + Intel counter (NEW) + Doomsday Clock (NEW, visual)
  │  ├─ Enemy: General Kukov named in briefing (NEW — narrative context)
  │  ├─ Select: 1 operation
  │  └─ Set: deployment budget
  │
  │  POST-PHASE: Research panel opens (Field Medic, 500 REQ — one project)
  │              Authority counter appears (linked to operation slots)
  │
  Systems added: Intel (visible), Doomsday Clock (visual), Enemy Commander
               (briefing), Research (1 project), Authority (displayed)

PHASE 3: FULL WAR TABLE (campaign opens up)
  │  WAR TABLE (~2 minutes — full depth reached)
  │  ├─ View: Doomsday Clock + front lines + resource display
  │  ├─ Scan: enemy initiatives (spend Intel → intercepts)     ◄─ NEW
  │  │   └─ Intercepted comms reveal commander personality/plans
  │  ├─ Fund: research projects (spend Requisition)
  │  ├─ Build: FOB slot if affordable (spend Requisition)       ◄─ NEW
  │  ├─ Select: 1-3 operations (slots gated by Authority)
  │  │   ├─ Tactical SpecOps (tech, denial, story)
  │  │   └─ Resource Ops (supply raids, intel sweeps)           ◄─ NEW
  │  └─ Set: deployment budget for next mission (Requisition)
  │
  Systems added: FOB, Resource Ops, Intel spending, Timed Choices,
               Intercepted Communications

PHASE 4+ LOOP ◄────────────────────────────────────────────────────────┐
  │  All systems active. Full War Table (~2 minutes per visit).        │
  │                                                                    │
  │  WAR TABLE                                                         │
  │  ├─ View: Doomsday Clock + front lines + resource display          │
  │  ├─ Scan: enemy initiatives (spend Intel → intercepts)             │
  │  ├─ Fund: research projects (spend Requisition)                    │
  │  ├─ Build: FOB slot if affordable (spend Requisition)              │
  │  ├─ Select: 1-3 operations (slots gated by Authority)              │
  │  │   ├─ Tactical SpecOps (tech, denial, story)                     │
  │  │   ├─ Resource Ops (supply raids, intel sweeps)                  │
  │  │   └─ Defector Ops (rare, Phase 4+ — recruit Soviets)            │
  │  └─ Set: deployment budget for next mission (Requisition)          │
  │                                                                    │
  │  RTS MISSION (the actual game — 15-45 minutes)                     │
  │  ├─ Starting credits = deployment budget                           │
  │  ├─ Shroud reveal = Intel spent on preparation                     │
  │  ├─ Research milestones = new units / support powers active        │
  │  ├─ Doctrine + FOB = passive bonuses active                        │
  │  ├─ Arms race state = your roster vs. enemy roster                 │
  │  ├─ Enemy commander leads opposing forces (personality/adapt)      │
  │  └─ Performance → earn REQ, Intel, Authority ──────────────────────┘
  │
  │  OPERATIONS (same RTS gameplay, 10-20 minutes each)
  │  ├─ Tactical ops → tech, denial, story flags
  │  ├─ Resource ops → REQ, Intel, compound bonuses
  │  └─ Defector ops → unique unit/scientist/intel source
  │
  │  WAR DIARY auto-updates (passive — zero player time)
  │  └─ Records: outcomes, Clock, decisions, commanders, roster
  │
  REPEAT until M14 ────────────────────────────────────────────────────┘

ENDGAME (M14)
  ├─ Shaped by: doctrine, research milestones, FOB, arms race
  ├─ Difficulty by: Doomsday Clock threshold, Authority, denial portfolio
  ├─ Enemy commander(s) lead final defense (adapted to player history)
  ├─ Defectors in roster create asymmetric Allied force composition
  └─ Unique to this playthrough: different units, enemies, briefing

POST-CAMPAIGN
  └─ War Diary exportable as campaign narrative document
```

### Example Compound Scenario

> The player chooses **Deep Strike** doctrine and **England** subfaction. They build the Intelligence Wing (Signals Intercept, 1,500 REQ) first for passive Intel generation. They fund Chronosphere research early. In Phase 3, they spend Intel to reveal General Kukov's armor initiative — the intercepted transmission reads: *"Tell Volkov his infantry better keep up this time."* They run "Radio Free Europe" (+4 Intel) instead of a combat SpecOps. In Phase 4, an intercept reveals Dr. Grigor's chemical weapons are nearing completion; they deploy light (5,000 REQ) and save funds for "Legacy of Tesla." Captured Tesla schematics accelerate Chronosphere research. In Phase 5, Einstein's lab-demo cutscene triggers. The War Diary records: *"The Chronosphere is ready. If we can deploy it, this war changes."* In Phase 6, Kukov returns — promoted after Hamburg, and his intercepts reference the earlier commando humiliation. But Intel also reveals one of Kukov's officers wants out. They run Operation LOOKING GLASS — a defector extraction. Tank Commander Petrov switches sides, bringing Mammoth Tank blueprints. The diary entry: *"Petrov is ours. Kukov must be furious."* Chronosphere deployment mission triggers in Phase 7. Their M14 features a Chronosphere assault on Moscow's rear defenses while partisan diversions hold the front, with Petrov's Mammoth Tank in the vanguard. Colonel Volkov — Kukov's inexperienced replacement — commands the defense. Authority is 78 (High). The final diary entry: *"Von Esling shook my hand. Tomorrow, we end this."*
>
> A different player with **Shock & Awe** doctrine and **Germany** subfaction builds the Air Support Facility first (Standing CAP). They never fund Chronosphere research. They run "Gold Reserve" and "Convoy Interdiction" for Requisition, pouring it into Advanced Armor Plating and the Prototype Tank Program. They deploy heavy every mission (8,000+ REQ). They never intercept Kukov's communications — they just outfight him every time. By Phase 6, Kukov has been defeated twice and is replaced by General Zhukov, a more cautious commander whose intercepts (when finally read) reveal defensive preparations rather than aggressive pushes. No defector ops — the player spent Intel on mission preparation instead. Their M14 is a frontal armor assault with Super Tanks, veteran cadre, and pre-destroyed fortifications from bombing runs. Zhukov commands Moscow's defenses with careful layered positions. Authority is 65 (Medium) — they took losses but won every main mission. No Chronosphere. No temporal tricks. Just steel. Their diary reads like a military campaign log. The other player's reads like a spy thriller.
>
> Both players played the same 14 main missions. Their campaigns felt fundamentally different. Both felt like *their* war. Both have a War Diary to prove it.

---

## Progressive System Introduction

The strategic layer has ten major additions across four layers. Introducing them all at once would overwhelm any player — newcomers and returning RA fans alike. Even XCOM, which this study draws from, gates its strategic systems behind campaign progress: the Avenger doesn't show every facility slot on day one, the Avatar Project doesn't start ticking immediately, and the Resistance Ring isn't available until you've learned basic squad management.

**Core principle: one new system per War Table visit, maximum two.** Each system is introduced at the moment it becomes relevant, explained in Von Esling's voice (Allied) or Nadia's voice (Soviet), and accompanied by a single clear action the player can take. This follows `enhanced-campaign-plan.md` § "Progressive Disclosure of Campaign Mechanics" — the same design philosophy applied to the strategic layer.

### Onboarding Schedule

| Phase | New Systems Introduced | What the Player Sees | What's Hidden |
|---|---|---|---|
| **Phase 0** (Prologue) | None | Pure RTS missions. No War Table. Player learns (or relearns) unit control, base building, combat. Resources accumulate invisibly in the background | Everything strategic |
| **Phase 1** (M01) | **War Table**, **Requisition**, **Deployment Budget** | War Table opens for the first time. Simple layout: front-line map, Requisition counter, deployment slider. Von Esling: *"Welcome to the War Table, Commander. You have funds — decide how to deploy them."* One action: set deployment budget for M02 | Intel, Authority, Research, FOB, Doctrine, Operations, Doomsday Clock (ticking internally but not yet displayed), enemy commanders |
| **Phase 1→2** (post-M01) | **Operations** (single slot) | One operation appears. Von Esling: *"An opportunity has presented itself. We can commit a team — if you authorize it."* One action: run the operation or skip it | Multiple slots, resource ops, operation pool complexity |
| **Phase 2** (M02-M03) | **Intel**, **Doomsday Clock** (visual), **Enemy Commander** (Kukov) | Intel counter appears after M01 operation yields intelligence. Clock appears on War Table as a visual element — Von Esling references it once: *"The Clock stands at seven minutes to midnight, Commander. We have time — but not forever."* Kukov is named in the M02 briefing as the opposing commander. No new decisions required — these are context, not actions | Authority display, Research, FOB, Intel spending |
| **Phase 2→3** (post-M03) | **Research** (one project), **Authority** (displayed) | First research project available (Field Medic, 500 REQ — cheap, obvious benefit). Authority counter appears with Von Esling: *"Allied Command is watching. Your performance earns operational freedom."* Player sees authority linked to operation slots for the first time | FOB, multiple research projects, resource ops |
| **Phase 3** (M04) | **FOB** (first slot), **Resource Operations**, **Timed Choices** | FOB panel opens — one slot available. Von Esling: *"We've secured a forward position. You may establish a facility."* Resource operations appear alongside tactical ops for the first time. First timed choice arrives. The campaign opens up | Multiple FOB slots, advanced research, intercepted communications |
| **Phase 4+** | **Intercepted Communications**, **Full Research Tree**, **Defector Opportunities** | Intel spending on initiative reveals now returns narrative intercepts (commander personality, adaptation). Multiple research projects available simultaneously. First defector opportunity may appear. All systems active | Nothing — full War Table |

### Why This Order

1. **Requisition first** because it's the simplest resource — you earn it, you spend it. One number, one slider. A complete newcomer can understand "I have 8,000. I deploy with 5,000. I have 3,000 left."

2. **Operations second** because they're a single binary choice: run this mission or don't. The player already understands RTS missions from Phase 0. An operation is just another mission with a reward.

3. **Intel and the Clock third** because they're read-only at introduction — the player doesn't spend Intel or manage the Clock. They just see them. The Clock provides ambient tension. Intel is a number that goes up. Understanding comes through observation, not action.

4. **Research and Authority fourth** because they require understanding Requisition (research costs REQ) and Operations (authority gates slots). The prerequisites are now in place.

5. **FOB and resource ops fifth** because they're the first genuine three-way trade-off (build a facility OR fund research OR save for deployment). By Phase 3 the player has enough context to evaluate that trade-off.

6. **Narrative systems last** because they layer on top of everything else. Intercepted communications only make sense after the player has spent Intel on initiative reveals. Defector ops only make sense after the player understands operations, Intel costs, and the arms race.

### Design Constraints

1. **No system appears before its prerequisite is understood.** Research requires understanding Requisition. Intercepted communications require understanding Intel spending. Defector ops require understanding operations + Intel + the arms race. The schedule respects these dependencies.

2. **Each introduction has exactly one action.** Phase 1 War Table: set a slider. Phase 1→2 operation: run or skip. Phase 2→3 research: fund or don't. The player never faces two new decisions simultaneously.

3. **Context before control.** The Doomsday Clock and enemy commanders are shown before the player can affect them. Intel is earned before the player spends it. Authority is visible before the player understands operation slots. Seeing a system before controlling it builds intuition.

4. **Returning RA fans skip nothing.** The schedule is paced, not gated. A veteran who already understands RTS fundamentals still progresses through the same phases — but the low Phase 0-1 decision pressure lets them absorb the new strategic systems without competing with relearning tank micro.

5. **Second playthroughs retain the schedule.** The onboarding pacing is authored into the campaign structure (which phases offer which systems), not a dismissible tutorial overlay. This keeps the difficulty curve consistent across playthroughs. Feature-discovery hints (D065 § Feature Smart Tips) are dismissible; the system availability is not.

---

## Campaign Phase-by-Phase Resource Flow

This section shows how resources flow across the Allied Enhanced Edition campaign, demonstrating the scarcity curve and decision pressure at each phase.

### Phase 0: Prologue (Linear, Tutorial)

No War Table. No resources. No strategic systems. Three scripted missions introduce the world and let the player learn (or relearn) RTS fundamentals — unit control, base building, combat.

Resources accumulate passively in the background (lore justification: Allied Command is assembling war funds while you fight in Greece):
- **Starting War Funds for Phase 1:** 8,000 REQ
- **Starting Intel:** 2 (earned invisibly — revealed in Phase 2)
- **Starting Authority:** 40 (Medium — revealed in Phase 2→3)

### Phase 1: First Contact (M01 — Rescue Einstein)

| Resource | Start | Earned | Spent | End |
|---|---|---|---|---|
| **Requisition** | 8,000 | +2,500 (M01 success) | -5,000 (M01 deploy) | 5,500 |
| **Intel** | 2 | +1 (M01 document capture) | — (not yet visible) | 3 |
| **Authority** | 40 | +5 (M01 victory) | — (not yet visible) | 45 |

**War Table opens for the first time.** Stripped-down view: front-line map, Requisition counter, deployment slider. Von Esling walks the player through it in character. One operation becomes available (post-M01) — a single slot, a single choice: run it or skip it.

**Decision pressure:** Minimal. The player learns the War Table by doing one thing: setting a deployment budget. The operation is a bonus — a familiar RTS mission with a reward attached.

### Phase 2: Securing the Bridgehead (M02-M03)

| Resource | Start | Earned | Spent | End |
|---|---|---|---|---|
| **Requisition** | 5,500 | +4,000 (M02+M03) +2,000 (operation) | -7,000 (deployments) | 4,500 |
| **Intel** | 3 | +2 (operations, captures) | — (visible but not yet spendable) | 5 |
| **Authority** | 45 | +8 (two victories) | — | 53 |

**New this phase:** Intel counter appears. Doomsday Clock appears on the War Table (visual context — Von Esling references it once). General Kukov is named in the M02 briefing as the opposing commander. Post-M03: first research project available (Field Medic, 500 REQ). Authority counter appears with explanation of operation slots.

**Decision pressure:** Low-moderate. The only new *decision* is whether to fund Field Medic research (500 REQ — affordable, obvious benefit). Everything else introduced this phase is context, not choices.

### Phase 3: Hold the Line — The Campaign Opens (M04 + Timed Choice #1)

| Resource | Start | Earned | Spent | End |
|---|---|---|---|---|
| **Requisition** | 4,500 | +3,500 (M04) +2,500 (operation) +4,000 ("Gold Reserve" if chosen) | -6,000 (deploy) -1,500 (research) -1,000 (FOB) -1,500 (operation costs) | varies: 4,500–8,500 |
| **Intel** | 5 | +1–4 (depending on operation choice) | -2 (initiative awareness — first Intel spend) | 4–7 |
| **Authority** | 53 | +5–10 (depends on performance) | -3 (if initiative uncountered) | 55–60 |

**New this phase:** FOB panel opens (one slot available). Resource operations appear alongside tactical ops. First timed choice arrives. Intel becomes spendable (on initiative reveals). This is the phase where the War Table reaches full depth.

**Timed Choice #1:** Behind Enemy Lines / Crackdown / Fresh Tracks. Each has a Requisition cost and a different resource reward profile. The player ALSO sees "Gold Reserve" (resource op, +4,000 REQ) competing for the same slot.

> *Do I run the Sarin denial (tactical advantage) or the Swiss bank raid (war funds for research)? I can't do both.*

**Decision pressure:** HIGH — but the player has had two phases to learn the vocabulary. They understand Requisition (Phase 1), operations (Phase 1→2), Intel (Phase 2), and research (Phase 2→3). The new elements (FOB, resource ops, timed choices) layer onto a foundation that's already solid. By the time the campaign asks hard questions, the player has the tools to answer them.

### Phase 4-6: The War Intensifies

Resources scale up. Main missions yield +3,000–4,000 REQ each. Operations cost more but yield more. Research projects hit their middle phases. Authority climbs toward High (65+) for successful players, opening the third operation slot.

**Key tension:** Multiple research projects competing for the same budget. Operations competing with research. Resource operations tempting the player away from tactical ops. FOB slots still unbuilt if the player prioritized research early.

The player who invested in Intel early (Intelligence War doctrine, Signals Intercept FOB) is now seeing hidden operations and initiative details that other players miss. The player who invested in Requisition (Fortress Europe, "Gold Reserve," "Lend-Lease Revival") has a fat wallet but less battlefield information.

### Phase 7-8: Endgame Preparation

| Resource | Typical Range | Notes |
|---|---|---|
| **Requisition** | 8,000–15,000 | Players who ran resource ops are wealthy; combat-focused players are leaner |
| **Intel** | 4–10 | Intelligence-focused players have a deep well; others are scraping |
| **Authority** | 60–85 | Correlates with overall performance |

Research milestones are paying off. Story missions triggered by research completion are the Act 3 highlights. The Chronosphere player is deploying temporal tech. The Armor player has Super Tanks. The Intel player sees everything the enemy is doing.

**Decision pressure:** The pressure shifts from "can I afford this?" to "which endgame am I building?" The final research phases are expensive (2,000 REQ). The final operations are ambitious. M14 deployment will be the biggest Requisition draw of the campaign. Every allocation shapes the endgame.

### Phase 9: Endgame (M14)

M14 deployment budget is the culmination of the entire resource game.

A wealthy, Authority-rich player deploys with 10,000+ starting credits, full research tree payoffs, three completed FOB facilities, and a dense arms race portfolio. M14 is still hard — but it's their kind of hard.

A lean, combat-focused player deploys with 6,000 starting credits but has every tactical advantage from operations and denial. M14 is different-hard — fewer resources, but the enemy is weaker.

A struggling player (low Authority, thin wallet) deploys with 4,000 and faces the enemy's full arsenal. M14 is genuinely difficult — but the failure-forward design means the campaign doesn't end; it just asks more of the player's tactical skill.

---

## Persistent State Update

The `campaigns.md` persistent state model expands:

```yaml
persistent_state:
  unit_roster: true
  veterancy: true
  equipment: true
  hero_progression: true          # breaking change if adopted; currently optional in campaigns.md
  war_funds: true                 # persistent Requisition pool (REQ)
  intel: true                     # persistent Intel pool
  command_authority: true         # passive track (0-100)
  research: true                  # project progress per project
  fob: true                      # built facilities and choices
  doctrine: true                 # selected doctrine ID
  doomsday_clock: true            # current Clock time (minutes to midnight)
  enemy_commanders: true          # commander memory flags and lifecycle state
  war_diary: true                 # auto-generated narrative entries per phase
  defectors: true                 # recruited defectors and their rewards
  custom_flags: {}               # existing Lua-writable state
```

---

## Decision-Making Time Budget

The strategic layer must remain lean. The time budget scales with the onboarding schedule — the player isn't facing the full table until Phase 3.

**Phase 1 War Table (~1 minute including Doctrine):**

| Activity | Time | Frequency |
|---|---|---|
| Doctrine selection | 60 seconds | Once (Phase 1, first War Table visit) |
| Set deployment budget (drag slider) | 5-10 seconds | Phase 1 |
| **Phase 1 total** | **~1 minute** | |

**Phase 2 War Table (~1 minute):**

| Activity | Time | Frequency |
|---|---|---|
| Glance at Doomsday Clock + front lines + Requisition + Intel | 5-10 seconds | Phase 2+ |
| Select operation (1 slot) | 15-30 seconds | Phase 2 |
| Fund research project (1 project available) | 10-15 seconds | Phase 2 (post-M03) |
| Set deployment budget (drag slider) | 5-10 seconds | Every phase |
| **Phase 2 total** | **~1 minute** | |

**Phase 3+ War Table (full depth — ~2 minutes):**

| Activity | Time | Frequency |
|---|---|---|
| Glance at Doomsday Clock + front lines + resources | 5-10 seconds | Every phase |
| Scan enemy initiatives (spend Intel to reveal?) | 10-15 seconds | Phase 3+ |
| Fund research projects (click FUND, see REQ deducted) | 15-20 seconds | Phase 3+ |
| Select operations — tactical, resource, and/or defector ops | 30-60 seconds | Phase 3+ |
| Build a FOB slot (if affordable and desired) | 10 seconds | Phase 3+ (occasional) |
| Set deployment budget (drag slider) | 5-10 seconds | Every phase |
| FOB review (glance at current setup) | 5 seconds | Phase 3+ |
| **Phase 3+ total per visit** | **~1.5–2.5 minutes** | |

The time budget ramps from ~1 minute (Phase 1) to ~2 minutes (Phase 3+). By the time the player faces the full table, they've had two phases of progressively richer War Table visits. This is within the "2-5 minutes, not 20" rule at every stage. The player spends 90% of their time in RTS missions. The War Table is a sharp, focused interlude — not a second game.

**Narrative Layer elements add zero time to this budget.** Enemy commander presence surfaces through briefings (already in the flow). Intercepted communications appear when Intel is spent on initiative reveals (already budgeted above). The War Diary updates passively — the player reads it only if they want to. Defector ops, when available (1–2 per campaign), are selected through the same operation slot mechanism already budgeted at 30–60 seconds.

---

## What This Does NOT Add

To be explicit about scope — this proposal intentionally avoids:

- **Complex economy.** Requisition is one number. It goes up when you win, down when you spend. No income rates, no tax sliders, no trade routes. The pre-mission slider and the research/operation costs are the entire economy.
- **Diplomacy or negotiation.** No ally requests, no faction negotiation. The war has two sides. You command one.
- **Procedural campaign structure.** The phase model and mission graph remain authored. Research milestones trigger authored content, not generated content. Generated SpecOps (including resource ops) use the existing generated-specops infrastructure.
- **Multiple victory conditions.** The campaign has one goal: win the war. Resources, research, and doctrine change *how* you get there, not *whether* you get there.
- **Complex base management.** The FOB is 5 binary choices with Requisition costs. No adjacency bonuses, no power management, no room excavation.
- **Civilization-style sandbox.** The player navigates a designed campaign with meaningful choices — they don't design the campaign.

The Narrative Layer (Additions 7–10) adds no management complexity. Enemy commanders, intercepted communications, and the War Diary are passive — they surface through existing briefings, intel spending, and auto-generated log entries. Defector recruitment uses the existing operation framework (one slot, one Intel cost, one RTS mission). The player never opens a new screen or learns a new mechanic.

The goal is not to make IC into XCOM or Civilization. The goal is to add *just enough* strategic depth that the player feels ownership over their campaign without leaving Red Alert behind.

---

## Relationship to Existing Design Documents

This proposal builds on, rather than replaces, existing design:

- **`strategic-campaign-layer-study.md`:** The phase model, arms race ledger, War Table, operations, and enemy initiatives are all preserved. **Breaking change:** War Momentum (§ "War Momentum") is replaced by the Doomsday Clock — a single unified urgency mechanic with threshold rings and authored event triggers. The underlying data (asset ledger, initiative outcomes, operation results) is unchanged; the Clock is a new lens on the same state with added threshold-based escalation. Requisition, Intel, and Command Authority form the resource backbone that makes strategic choices concrete and earned. Research and Doctrine layer on top. The arms race ledger gains a research input channel. Front-line visualization is a visual upgrade to the existing War Table. Resource-opportunity operations extend the operation pool with a new reward category.
- **`campaigns.md`:** The campaign graph engine, persistent state model, spectrum outcomes, hero progression, and generated SpecOps are unchanged. Research milestones are a new trigger type for campaign graph transitions. FOB bonuses are passive modifiers applied through the `custom_flags` key-value state (campaigns.md § persistent_state) or a new dedicated FOB state field if adopted. The persistent state schema gains `war_funds`, `intel`, `command_authority`, `research`, `fob`, `doctrine`, `doomsday_clock`, `enemy_commanders`, `war_diary`, and `defectors` fields. The generated SpecOps system gains two new mission families (`supply_raid`, `intelligence_sweep`). Defector operations are a new operation category alongside tactical and resource ops.
- **`enhanced-campaign-plan.md`:** The 7 design rules, narrative framework, mission content, and timed-choice system are unchanged. Research milestones integrate into the existing cutscene and briefing system. Doctrine biases the operation pool weights that are already authored per phase. Resource operations appear alongside existing authored operations. The Progressive System Introduction schedule (§ "Progressive System Introduction") extends the existing § "Progressive Disclosure of Campaign Mechanics" — strategic systems follow the same one-at-a-time, explain-at-encounter, show-don't-tell principles. **Integration Note:** The older operational budget (turns per phase) has been completely replaced by Command Authority-gated operation slots — the number of operations per phase became an earned consequence of tactical performance rather than a designer-assigned number.
- **`generated-specops-missions-study.md`:** Gains two new mission families (`supply_raid` and `intelligence_sweep`) alongside the existing six (`intel_raid`, `tech_theft`, `tech_denial`, `rescue`, `faction_favor`, `counter_intel`). These use the same site kit / objective module / complication infrastructure.
- **`subfaction-country-system-study.md`:** Subfactions compose with Doctrine — nation/institution provides the flavor identity, doctrine provides the strategic identity.
- **`character-mbti-bible.md`:** Character personality profiles inform research milestone briefings, enemy commander voices, intercepted communications, and War Diary tone. Einstein's INTP voice drives Chronosphere research notes. Von Esling's ISTJ voice drives Command Authority feedback and Doomsday Clock threshold dialogue. Enemy commander MBTI profiles drive intercept voice and adaptation personality. Nadia's INTJ voice drives Soviet counter-research briefings.

### Follow-Up Work (If Adopted)

1. Add `war_funds`, `intel`, `command_authority`, `doctrine`, `research`, `fob`, `doomsday_clock`, `enemy_commanders`, `war_diary`, and `defectors` to `campaigns.md` persistent state schema
2. Add resource-opportunity and defector operation templates to `campaigns.md` operation card catalog
3. Add `supply_raid` and `intelligence_sweep` families to `generated-specops-missions-study.md`
4. Update `enhanced-campaign-plan.md` phase-by-phase design with resource flow tables and research allocation suggestions per phase
5. Author research milestone cutscenes and briefing text (narrative team)
6. Define War Table map visual spec (front lines, Doomsday Clock display, theater regions, resource header, intercept overlay)
7. Add FOB slot definitions and build-cost model to `campaigns.md` intermission screen catalog
8. Define Von Esling Command Authority dialogue variants for briefings
9. Author Doomsday Clock threshold-crossing cutscenes and Von Esling dialogue variants per ring (green/yellow/orange/red)
10. Define 4-6 Allied campaign enemy commanders with MBTI profiles, specialties, adaptation rules, and lifecycle definitions (narrative team)
11. Author intercepted communication templates per enemy initiative (2-3 variants per initiative, keyed to commander personality and player memory flags)
12. Define War Diary template structure and tone variants per Clock threshold
13. Author 2-4 defector operation scenarios with prerequisite conditions, extraction missions, and spectrum outcomes
14. Implement Narrative Coherence Principle: audit all authored briefings, cutscenes, and dialogue for conditional gating by research state, commander state, defector state, and arms race ledger — ensure no narrative element references campaign state the player hasn't reached
15. Integrate Progressive System Introduction schedule into `enhanced-campaign-plan.md` § "Progressive Disclosure" — add War Table system onboarding entries (Requisition, Operations, Intel, Clock, Research, Authority, FOB) to the existing mechanic-explanation table, with Von Esling/Nadia dialogue for each introduction moment
