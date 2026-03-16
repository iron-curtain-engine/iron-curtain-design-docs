# Enhanced Edition Campaign Plan — Allied & Soviet

This is the concrete content plan for IC's first-party "Enhanced Edition" campaign extensions. It weaves the original 14+14 main campaign missions, 34 Counterstrike/Aftermath expansion missions, and IC-original missions into two unified branching campaign graphs.

**Source material:** 28 base game missions + 20 Counterstrike missions + 18 Aftermath missions + 4 Ant missions = 70 total existing missions. IC adds ~15-20 original missions showcasing platform features.

**Design principle:** The classic campaign is always available as a straight-line path through the graph. Enhanced Edition content branches off as optional/alternative paths. IC-original missions are toggleable per-mission; Counterstrike and Aftermath missions are toggleable per-chain (following D021's sub-feature toggle model in `modding/campaigns.md`).

---

## Campaign Design Rules

These rules govern how the Enhanced Edition missions are designed. They address problems in the original campaigns — awkward character introductions, unexplained difficulty spikes, disconnected side missions — and establish IC's standard for campaign quality.

### Rule 1: No Awkward Character Introductions

The original RA1 cutscenes often introduced characters with a "am I supposed to know this person?" feeling. Von Esling, Stavros, Tanya, Einstein — they all appear in briefing videos assuming the player knows who they are and why they matter.

**IC rule:** Every named character gets a **gameplay introduction before their cutscene introduction**. The player meets the character through gameplay (controlling them, fighting alongside them, or hearing about them) before seeing them in a briefing video.

| Character | Original Problem | IC Fix |
|---|---|---|
| **Stavros** | Appears in M1 briefing with no introduction. Who is this Greek officer? | Player fights alongside him in Fall of Greece prologue missions BEFORE M1 |
| **Tanya** | Appears in M1 briefing. Stavros objects but the player doesn't know either of them | Player plays Tanya's prologue mission (First Blood) BEFORE M1. Now the M1 confrontation between Stavros and Tanya is a clash between two known characters |
| **Einstein** | Appears as a rescue target in M1 but the player has no emotional connection to him | IC prologue includes a brief Einstein moment — his voice on a radio transmission during the Fall of Greece, explaining what the Soviets are building. The player knows his voice before rescuing him |
| **Volkov** (Soviet) | Appears in a CS expansion mission with no setup | Player meets Volkov in the Soviet prologue mission (Awakening) before the CS missions |
| **Kosygin** (M9) | Soviet defector — the player is told to rescue someone they've never heard of | IC adds intel references to Kosygin in earlier missions (intercepted communications, spy reports). By M9, the player knows who he is and why he matters |

### Rule 2: Progressive Difficulty and Mechanics Teaching

The original RA1 campaign has inconsistent difficulty — some early missions are harder than late ones, and new mechanics appear without introduction. IC's Enhanced Edition follows D065 (Tutorial) principles: each mission teaches one new thing, difficulty escalates smoothly, and the player is never overwhelmed.

**Difficulty curve for the Enhanced Edition prologue and Act 1:**

| Mission | Difficulty | New Mechanic Introduced | Why This Order |
|---|---|---|---|
| CS "Personal War" (prologue) | Very Easy | Basic unit control, movement, escort | First mission ever — the player learns to move units. Stavros must survive |
| CS "Evacuation" (prologue) | Easy | Time pressure, civilian escort, multiple objectives | Adds urgency and multi-tasking. Still small scale |
| IC "Tanya: First Blood" (prologue) | Easy-Medium | Hero unit, special abilities, skill points | Introduces hero mechanics with a powerful unit. Tanya can't die but it's okay to struggle |
| [M1] "In the Thick of It" | Medium | Base assault, combined arms (Tanya + support) | First "real" mission — but player already knows unit control, escort, and hero mechanics |
| [M2] "Five to One" | Medium | Time-limited defense, convoy mechanics | Adds time pressure at larger scale |
| [M3] "Dead End" | Medium | Bridge destruction, terrain objectives, stealth | Tanya-focused mission — player is now comfortable with hero gameplay |
| [M4] "Ten to One" | Medium-Hard | Full base defense, counterattack | First genuinely hard mission — but by now the player has 6+ missions of experience |

**Side missions are exempt from this curve.** Optional content (Counterstrike, Aftermath, IC side missions) can be any difficulty — the player chooses to take them. A warning label in the mission select indicates difficulty: "This is a challenging optional mission."

**New IC mechanics are introduced through gameplay, not text:**
- **Hero progression:** Introduced in prologue (Tanya: First Blood). The player earns skill points and sees the skill tree for the first time in a low-stakes mission
- **Spectrum outcomes:** First experienced in M3 (bridge destruction with partial success). Low consequence — the next mission is harder, not game-ending
- **Dynamic weather:** First appears in M8 (Chronosphere defense storm at minute 30). The storm is progressive — starts mild, gets worse — giving the player time to adapt before it's severe
- **Embedded task force:** First appears as an optional alternative in M10B ("Evidence: Enhanced"). Players who choose the classic M10B path never encounter it until M14: Enhanced (also optional)

### Rule 3: The World Moves Without You (XCOM-Style Timed Choices)

Between certain main missions, the Enhanced Edition presents **timed strategic choices** — multiple operations happening simultaneously, and the player can only address one before the window closes. The others resolve without the player, with consequences.

This is the XCOM: Enemy Unknown model: three abduction sites appear, you pick one, the other two have consequences. Applied to RA1:

**How it works:** Timed choices are implemented as **standard D021 decision nodes** with `state_effects` that set flags for the unchosen options. No new campaign primitive needed — the existing `decision` mechanism (D021 § Campaign Graph) handles the branching, and Lua `Events.on("mission_start")` reads the flags to apply consequences.

```yaml
# Between M4 and M5: a D021 decision node with 3 options
# Unchosen options have their flags set automatically by the decision node
missions:
  act1_strategic_choice:
    type: decision                            # standard D021 decision node
    prompt: "Command has identified three urgent situations. We can only respond to one."
    choices:
      - label: "Rescue Operation — Tanya"
        description: "Tanya's team has gone silent. Send a rescue force now."
        next: ic_behind_enemy_lines
        unchosen_effects:                     # applied if NOT chosen
          set_flag:
            tanya_captured: true             # Tanya captured off-screen
      - label: "Chemical Threat — Greece"
        description: "Soviet Sarin gas facilities detected. Neutralize before they go active."
        next: cs_sarin_gas_1
        unchosen_effects:
          set_flag:
            sarin_active: true               # chemical weapons in M8
      - label: "Flanking Opportunity — Siberia"
        description: "A window to establish a second front. It won't last."
        next: cs_siberian_1
        unchosen_effects:
          set_flag:
            siberian_window_closed: true    # arc permanently closed
```

The `unchosen_effects` field is the one extension to D021's existing decision node: when the player picks one option, all other options' `unchosen_effects` are applied automatically. This is a small addition to the campaign YAML schema (a few lines in the decision node handler), not a new primitive. The flags are read by subsequent mission Lua scripts via `Campaign.get_flag()` as documented in § Side Mission Rewards.

**Design rules for timed choices:**

1. **Never more than 3 options.** The player can process three choices; more creates decision paralysis
2. **All options are good; the loss is opportunity cost.** No "trap" choice. Each has clear value. The tension is what you DON'T get
3. **Consequences are visible and referenced.** If Tanya gets captured because you chose Siberia, the M5 briefing says so. If Sarin goes active because you chose Tanya, M8 is harder and the briefing explains why
4. **Main campaign is never blocked.** The timed choice affects difficulty and available content, never whether the player can continue
5. **Placed between acts, not mid-mission.** Timed choices appear in the campaign map/intermission screen, not during gameplay. The player has time to think
6. **2-3 per campaign maximum.** Too many and the mechanic loses weight. Each timed choice should feel momentous

**Timed choice placement in Allied Enhanced Edition:**

| Between | Options | Theme |
|---|---|---|
| M4 → M5 | Rescue Tanya / Sarin Gas / Siberia | "Three fires burning — which one do you fight?" |
| M9 → M10 | Poland liberation / Aftermath Italy / IC air campaign | "Where do we push next?" |
| M12 → M14 | Final prep choice — one last operation before Moscow | "One more shot before the endgame" |

### Rule 4: Optional Content Has Real Stakes

If the player doesn't get involved in Tanya's mission, she gets captured. If the player doesn't neutralize the Sarin facilities, chemical weapons appear in later missions. This is the core Enhanced Edition promise: **side content matters because skipping it has consequences.**

But the inverse must also be true: **completing side content must visibly improve the main campaign.** This is covered in § Side Mission Rewards — the briefing references what the player did or didn't do, the next mission is harder or easier, unique units and abilities appear or don't.

The worst design sin: optional content that exists in a vacuum. If a side mission has no effect on the main campaign — if completing it and skipping it produce identical experiences — it shouldn't be in the Enhanced Edition.

### Rule 5: Two Lanes — Commander (Mandatory) and Operative (Optional)

The Enhanced Edition campaign has two parallel gameplay lanes:

| Lane | Player Role | Gameplay | Mandatory? | Example |
|------|-----------|----------|------------|---------|
| **Commander Lane** | Strategic commander | Base building, army management, combined-arms warfare | **Always mandatory** — these ARE the campaign | M1, M2, M4, M7, M8, M12, M14 |
| **Operative Lane** | Tanya / Volkov | Commando missions, hero abilities, stealth, infiltration, task force ops | **Optional (recommended)** — side missions that reward the main campaign | Behind Enemy Lines, Evidence: Enhanced, Spy Network, Operation Skyfall |

**Why this separation matters:**

1. **Respects player preference.** Some players love commando missions. Others find them frustrating and just want to build bases and crush armies. Neither preference is wrong. The Commander lane always works alone.

2. **Creates a meaningful investment loop.** Tanya missions are optional, but doing them makes the main missions easier. Skip them, and you face full-strength enemies with no intel, no sabotage bonuses, no captured equipment. The choice is real.

3. **Hero progression becomes opt-in depth.** Tanya's skill tree, veterancy, and equipment only matter if you play her missions. A Commander-only player never sees the skill tree UI. An operative-focused player builds Tanya into a legend across 8+ side missions.

4. **Co-op maps naturally to lanes.** In D070 co-op mode, Player 1 takes the Commander lane and Player 2 takes the Operative lane simultaneously — the Commander runs the base while SpecOps runs the commando mission. The lane separation is already the co-op architecture.

**The exception:** Allied M1 ("In the Thick of It") is the one mission where Tanya is integral to the objective — Einstein's rescue requires her. This is the only mandatory operative-lane mission in either campaign. Every other commando/spy mission (M3, M5, M6, M9, M10B, M13) has a Commander-lane alternative. A player who dislikes commando gameplay can complete the entire campaign through base building and army warfare after M1.

**Canonical rule: every operative mission has a Commander alternative.** The alternatives are documented in the lane diagram below and in the full campaign graph. Commander alternatives achieve the same narrative result with reduced rewards (less intel, more casualties, cruder outcome) — making the operative route attractive without making it mandatory.

**How it looks in practice:**

```
Commander Lane (mandatory):        Operative Lane (optional):
─────────────────────────          ─────────────────────────
                                   [CS] Personal War (Stavros)
                                   [CS] Evacuation (Stavros)
                                   [IC] Tanya: First Blood
[M1] Rescue Einstein ──────────── (Tanya used IN this mission — exception*)
[M2] Supply Convoy
                                   [IC] Supply Line Security
                                   [M3] Destroy Bridges (Tanya commando)
                                   ↓ bridges destroyed → M4 easier
[M4] Hold the Pass ────────────── if bridges intact → full enemy force
                                   [IC] Behind Enemy Lines (Tanya spy)
                                   ↓ results affect M5/M6
                                   [M5] Rescue Tanya (conditional, spy+commando)
                                   [M6] Iron Curtain Infiltration (spy)†
                                   ↓ intel affects M7/M8
                                   [CS] Sarin Gas 1-3
[M7] Sunken Treasure ──────────── affected by intel + Sarin results
                                   [IC] Operation Skyfall (air)
[M8] Protect Chronosphere ─────── affected by Skyfall + Sarin results
                                   [M9] Extract Kosygin (spy+escort)†
                                   ↓ Kosygin intel affects M10-M14
                                   [AM] Poland missions
[M10A] Suspicion ──────────────── affected by Kosygin intel + Poland
                                   [M10B] Evidence (interior commando)†
[M11] Naval Supremacy
                                   [IC] Joint Operations (co-op)
[M12] Takedown ─────────────────── affected by all side missions
                                   [M13] Focused Blast (interior commando)†
[M14] Moscow ───────────────────── EVERYTHING accumulates

*  M1 is the one exception where Tanya is mandatory — she's integral
   to Einstein's rescue. In co-op (D070), Commander handles the base
   assault while SpecOps handles Tanya's infiltration path.

†  ORIGINAL COMMANDO MISSIONS MOVED TO OPERATIVE LANE:
   These four missions (M6, M9, M10B, M13) are spy/commando gameplay
   in the original with no base building. In Enhanced Edition, each
   gets a Commander-lane alternative for players who prefer army
   gameplay:

   M6 (spy infiltration) → Commander alternative: full assault on
      the Tech Center. You get the intel but the facility is destroyed
      (less intel quality than the spy route — a trade-off, not a freebie)

   M9 (spy+escort) → Commander alternative: armored extraction.
      Send a tank column to the compound, blast through, grab Kosygin.
      Louder, more casualties, but no spy required. Kosygin provides
      less intel (he was roughed up in the fighting)

   M10B (interior commando) → already has IC "Evidence: Enhanced"
      as an embedded task force alternative

   M13 (interior commando) → Commander alternative: full base assault
      on the facility exterior. Destroy the Iron Curtain facility with
      artillery and air strikes instead of planting charges inside.
      Same narrative result, different gameplay

   In each case: the Commander alternative achieves the same story
   outcome but with REDUCED rewards (less intel, more casualties,
   cruder result). The operative route is the "clean" option with
   better rewards. This makes the operative lane attractive without
   making it mandatory.
```

The arrows show influence: operative results flow INTO commander missions as difficulty modifiers, briefing context, and bonus resources. Commander missions never require operative missions to be completable — they just benefit from them.

**Soviet campaign equivalent:** Volkov replaces Tanya in the operative lane. The Soviet campaign has fewer commando missions than the Allied one, but the same principle applies:

```
Soviet Commander Lane (mandatory):   Soviet Operative Lane (optional):
──────────────────────────────────   ─────────────────────────────────
                                     [IC] Volkov: Awakening
                                     [AM] Testing Grounds
[M1] Lesson in Blood
[M2] Guard Duty
                                     [M3] Covert Cleanup (spy chase)†
                                     [IC] Mole Hunt (counter-intel)
[M4] Behind the Lines ─────────────  affected by Mole Hunt results
[M5] Distant Thunder
[M6] Bridge over Grotzny ──────────  roster + resources from M5 carry over
                                     [CS] Volkov & Chitzkoi (hero)
                                     [CS] Legacy of Tesla
[M7] Core of the Matter ──────────── Volkov vs Tanya if available
[M8] Elba Island
                                     [CS] Paradox Equation
                                     [CS] Mousetrap (hunt Stavros)
                                     [M9] Liability Elimination†
                                     [IC] Stalin's Shadow (spy)
[M10] Overseer (or Strike alt.)
[M11] Sunk Costs ──────────────────  affected by Legacy of Tesla
[M12] Capture Tech Centers
                                     [IC] Operation Tempest (air)
[M13] Capture Chronosphere ────────  Volkov commando approach if avail.
[M14] Soviet Supremacy ────────────  EVERYTHING accumulates

†  SOVIET COMMANDO MISSIONS MOVED TO OPERATIVE LANE:

   M3 (spy chase) → Commander alternative: cordon and sweep.
      Deploy infantry squads to lock down the district and
      flush the spy out with overwhelming force. Less elegant
      than a precise chase — spy may escape with partial intel
      (worse for you in future missions)

   M9 (assassination) → Commander alternative: arrest operation.
      Send an armored column to detain the target publicly.
      Same political result but cruder — creates more internal
      unrest (a flag that affects Act 3 stability)
```

### Rule 6: Hero Capture Creates Escalating Rescue Pressure

When a hero operative (Tanya, Volkov) is captured in a failed mission, the enemy doesn't just hold them — they **interrogate** them. The intended mechanic is **escalation**: the longer the hero remains captured, the more intel or technology the enemy extracts.

This is one of the few places where the first-party Enhanced Edition deliberately extends the base D021 campaign graph model. The rescue mission is a **bounded pending branch**: it can be taken immediately, taken later within a limited window, or ignored entirely. Each intervening mission increases the compromise level until the window closes.

The original M5 cutscene already implies this urgency: a Soviet interrogator presses Tanya for information. IC makes those stakes real and mechanical.

#### Tanya Capture Escalation (Allied Campaign)

Tanya knows Allied troop positions, the Chronosphere project, spy network identities, and battle plans. The Soviets want all of it.

| Rescue Outcome | What Soviets Extract | Main Campaign Consequence |
|---|---|---|
| **Rescued immediately** (M5 taken right away) | Nothing — she held out | No penalty. Tanya returns to roster at full strength. Briefing: *"Tanya didn't break. She never does."* |
| **Rescued after 1 intervening mission** | Partial troop positions, reconnaissance routes | Tanya returns wounded and unavailable for 1 mission. The next main mission gets enemy ambushes at likely approach routes. Briefing: *"Soviet forces seem to know our staging areas. Tanya may have been forced to reveal positions before we got her out."* |
| **Rescued after 2+ intervening missions** | Chronosphere project details, battle plans, spy contacts | **Severe compound penalty for Act 2+:** Enemy knows Chronosphere location (M8 defense starts under immediate attack, no setup time). Spy network is partially rolled up (operative M6 route is degraded or unavailable). Enemy AI gains strong intel advantages through the rest of the act. Briefing: *"We've intercepted Soviet communications referencing detailed knowledge of our Chronosphere project and spy network. They could only have gotten this from Tanya. We waited too long."* |
| **Never rescued** | Everything they can get before Tanya breaks or is transferred | Worst version of the severe penalty. Tanya is lost from the roster for the rest of the campaign unless a later special recovery beat is authored. Briefing: *"We failed Tanya. The Soviets are now acting on intelligence they could only have extracted from her."* |
| **Never rescued + never captured** (Behind Enemy Lines succeeded) | N/A — Tanya was never captured | No penalty, no rescue needed. Best outcome. |

**Rescue quality also matters:** If M5 is completed but with heavy casualties or a long mission time, Tanya returns wounded (hero status: `wounded`, unavailable for 1 mission). A fast, clean rescue returns her at full strength.

#### Volkov Capture Escalation (Soviet Campaign)

Volkov is a cybernetic super-soldier. He knows Soviet military secrets, but his real value to the Allies isn't intel — it's **technology**. If the Allies capture Volkov, they reverse-engineer his cybernetics.

| Rescue Outcome | What Allies Extract | Main Campaign Consequence |
|---|---|---|
| **Rescued immediately** (`Deus Ex Machina` taken right away) | Nothing — they couldn't crack his armor | No penalty. Volkov returns to roster. Briefing: *"Volkov's armor held. They couldn't access his systems."* |
| **Rescued after 1 intervening mission** | Armor composition data | Volkov returns damaged / downgraded for 1 mission. Allied units in future missions gain improved armor-piercing capability. Briefing: *"Allied intelligence reports suggest they've analyzed Volkov's armor plating."* |
| **Rescued after 2+ intervening missions** | Armor composition + subsystem schematics | **Major penalty for Act 3:** Allies field improved anti-armor weapons and limited cyborg prototypes in M12-M14. Volkov returns scarred and with reduced availability. Briefing: *"The Allies have learned too much from Volkov's systems. We'll face our own technology soon enough."* |
| **Never rescued** | Complete technology transfer | **Severe penalty for Act 3:** Allies have full cyborg infantry squads in M12-M14. Volkov permanently lost from roster. Briefing: *"The Allies have weaponized Volkov's technology against us. Our greatest soldier has become our greatest liability."* |
| **Never captured** (Volkov missions succeeded or not attempted) | N/A | No penalty, no rescue needed. |

#### Implementation

The first-party plan uses a bounded pending-rescue state with explicit escalation, rather than collapsing capture into a one-node yes/no:

```yaml
# CampaignState flags set when a hero is captured
hero_capture:
  tanya:
    captured: true
    pending_rescue: true
    rescue_mission_id: allied_05_rescue
    missions_since_capture: 0
    rescue_window_closes_after: 2         # after 2 intervening missions, severe/terminal state
    compromise_level: none                # none | partial | severe | terminal
  volkov:
    captured: false
```

```lua
-- Example controller logic run when the player commits to another
-- non-rescue mission while the rescue branch is still pending.
-- Campaigns can execute this from mission_start of the next mission
-- or from their intermission / mission-launch script.
local capture = Campaign.get_flag("hero_capture.tanya")
if capture and capture.pending_rescue then
  capture.missions_since_capture = capture.missions_since_capture + 1

  if capture.missions_since_capture == 1 then
    capture.compromise_level = "partial"
    Campaign.set_flag("spy_network_compromised", true)
  elseif capture.missions_since_capture >= 2 then
    capture.compromise_level = "severe"
    Campaign.set_flag("chronosphere_location_known", true)
    Campaign.set_flag("operative_m6_degraded", true)
  end

  if capture.missions_since_capture > capture.rescue_window_closes_after then
    capture.compromise_level = "terminal"
    capture.pending_rescue = false
    Campaign.set_flag("tanya_lost", true)
  end

  Campaign.set_flag("hero_capture.tanya", capture)
end
```

**Briefing integration:** The dynamic briefing system (§ Side Mission Rewards) assembles conditional lines based on the current compromise level. Immediate rescue, delayed rescue, and abandoned rescue each get distinct lines in the next briefing.

**Cross-campaign echo:** The Soviet M7 briefing always includes a line about an Allied commando operating in the facility — regardless of Allied campaign state. *"Our interrogators have extracted valuable information from the captured Allied commando. Use it well."* This is always-present authored flavor text, not conditional on cross-campaign data (see § Cross-Campaign Continuity).

### Rule 7: Teach Every Mechanic at the Right Moment

Campaign mechanics (capture consequences, side mission rewards, timed choices, spectrum outcomes, hero progression, two-lane structure) are powerful but only if the player understands them. The Enhanced Edition uses **progressive disclosure** — each mechanic is explained exactly when the player first encounters it, never before, never after. No front-loaded tutorials, no walls of text, no unexplained mechanics.

This follows D065 (Tutorial & New Player Experience) § progressive disclosure and hint systems.

**When each mechanic is explained:**

| Mechanic | First Encountered | How It's Explained | Example |
|---|---|---|---|
| **Hero progression** | End of prologue (Tanya: First Blood) — first skill point earned | Skill tree opens with a brief tooltip: *"Tanya has earned a skill point. Choose an ability to unlock. Skills persist across all missions."* | Player sees the skill tree for 10 seconds with one clear choice. No overwhelm |
| **Roster carryover** | M1 → M2 transition — surviving prologue units appear | Briefing note: *"Troops from your previous operation have been reassigned to this front."* A tooltip highlights the carried-over units on the map | Player sees familiar faces from the prologue. The connection is obvious |
| **Spectrum outcomes** | M3 (Destroy Bridges) — first mission with partial success | Debrief screen shows which bridges were destroyed and explicitly states the consequence: *"2 of 4 bridges destroyed. Enemy reinforcements will be partially reduced next mission."* | Not hidden — the debrief tells the player exactly what their performance means |
| **Side mission rewards** | First optional mission completed — reward shown in next briefing | Briefing line highlighted: *"Thanks to your supply line operation, Chrono reinforcements are available."* Tooltip: *"Side mission rewards affect main missions. Completed operations provide tactical advantages."* | The cause-and-effect is spelled out the first time. After that, the player understands the pattern |
| **Timed choices** | First timed choice (between M4 and M5) | A dedicated intermission screen explains: *"Multiple operations require your attention. You can only commit forces to one before the window closes. The others will resolve without you — with consequences."* Each option shows its reward AND the consequence of not choosing it | The stakes are visible before the player commits. No hidden information |
| **Two-lane structure** | First time an operative mission appears alongside a main mission | Campaign map shows both lanes visually. Tooltip on the operative mission: *"Optional special operation. Completing this will affect the next main mission. Skipping it is valid — the campaign continues either way."* | The word "optional" is explicit. The reward preview is explicit. No guessing |
| **Capture consequences** | First time a hero is captured (Behind Enemy Lines fails) | Immediate notification: *"Tanya has been captured. The longer she is held, the more intel the enemy may extract. Rescue her when possible."* A priority marker / urgency indicator appears on the campaign map or intermission mission list next to M5 | The escalation mechanic is explained at the moment it becomes relevant — not in a tutorial before it happens |
| **Dynamic weather** | First mission with weather (M8 storm or side mission) | Brief in-mission tooltip when weather changes: *"A storm is approaching. Visibility will decrease and vehicle movement will slow."* Gameplay effect is immediate and visible | Learn by experiencing. The tooltip explains what's happening; the gameplay proves it |
| **Commander alternatives** | First time a commando mission appears with a Commander alternative | Mission select shows both options side by side with labels: *"Operative approach: infiltrate with Tanya. Stealthier, better rewards."* / *"Commander approach: assault with your army. Direct, but fewer intelligence gains."* | Both options visible, both labeled with trade-offs. The player chooses their style |

**Explanation principles:**

1. **Explain at the moment of encounter, not before.** Don't explain timed choices in the prologue. Explain them when the first timed choice appears
2. **Show, don't tell.** The first spectrum outcome isn't explained in a tutorial — the debrief screen SHOWS the player what their partial success means for the next mission
3. **Make consequences visible before they happen.** Timed choices show both the reward of choosing AND the consequence of not choosing. No hidden penalties
4. **Explain once, then trust the player.** The first side mission reward gets a highlighted briefing line and tooltip. The second one just gets the briefing line. By the third, the player understands the pattern
5. **Never interrupt gameplay to explain.** All explanations happen in briefings, debriefs, intermission screens, or small in-game tooltips — never a pause-the-game tutorial popup during combat
6. **Use the briefing officer's voice.** Von Esling (Allied) and Nadia (Soviet) explain mechanics in character. *"Commander, we can only commit to one operation"* is both a narrative moment and a mechanics explanation. The UI reinforces with tooltips, but the character delivers the message

**D065 integration:** These explanations use the existing `feature_discovery` hint category from D065 § Feature Smart Tips. Each mechanic has a hint entry in `hints/campaign-mechanics.yaml` that triggers on first encounter and is dismissible. Players who've already learned the mechanic (from a previous playthrough or the tutorial) never see the hint again.

---

## Side-Campaign Thread: Einstein's Burden

A narrative thread that runs underneath the main war campaign, surfacing at key moments and paying off in Act 3. This is not a separate campaign — it's woven into the existing missions as optional discoverable content, briefing moments, and one pivotal reveal mission.

The thread answers the question every player eventually asks: *"Einstein used a time machine to remove Hitler and created this entire war. Does he feel guilty? Could he undo it? Why doesn't he?"*

> **Scope note:** The narrative thread itself is baseline Enhanced Edition content. The actual **time-machine gameplay unlocks** tied to `Einstein's Confession` / `Temporal Experiment` are **experimental D078 add-ons only** while [D078](/mnt/c/git/games/iron-curtain-design-docs/src/decisions/09d/D078-time-machine.md) remains Draft. If D078 never graduates, these missions remain lore reveals with no gameplay unlock.

### Thread Structure

**Prologue — The Voice**

Einstein is heard but not seen. During the Fall of Greece prologue missions, his voice comes through on a radio transmission to Stavros:

> *"Stavros, listen to me. The Soviets have developed weapons beyond anything we anticipated. I... I bear some responsibility for this. Get your people out. I will explain when we meet."*

The player doesn't know what "some responsibility" means yet. It's a hook — filed away, not explained.

**M1 — The Rescue**

The player rescues Einstein. In the debrief, Einstein thanks the Commander but is visibly troubled. A brief exchange (text in the briefing, not a cutscene — IC doesn't add FMV):

> *"Commander, thank you. I owe you my life. And... I owe the world an explanation. But not today. Today we have a war to win. A war that exists because of me."*

Von Esling cuts him off: *"Professor, that's enough. Commander, your next assignment."* The moment passes. The player notices Einstein said something strange but the campaign moves on.

**Acts 1-2 — Intel Fragments (Collectible Narrative)**

During operative-lane missions (Behind Enemy Lines, Spy Network, Sarin Gas facilities, Italy operations), the player can find **Einstein's Research Fragments** — optional collectibles hidden in mission maps. These are short text entries that gradually reveal the Trinity story. Finding them is never required — they're rewards for thorough exploration.

| Fragment | Found In | Content |
|---|---|---|
| **Fragment 1: Soviet Intelligence File** | Behind Enemy Lines (Tanya infiltrates Soviet facility) | A Soviet report: *"Subject: Allied Scientist — Priority capture target. His research predates the Chronosphere program by decades. Original experiments conducted in the American Southwest, 1946. Nature of experiments: CLASSIFIED — temporal displacement."* |
| **Fragment 2: Einstein's Personal Note** | Spy Network (recruiting agents in Soviet territory) | A handwritten note found in an Allied dead drop: *"I have seen what happens when one man decides to change history. I removed a monster and created a war. Millions dead because I played God. I will not make that mistake again."* |
| **Fragment 3: Captured Soviet Briefing** | Sarin Gas facility (CS mission) | Soviet briefing document: *"Priority directive from Moscow: locate and secure Einstein's pre-war research notes. The Chronosphere is useful but limited. His original work — the temporal displacement device — is the true prize."* |
| **Fragment 4: Einstein's Hidden Journal** | Italy operations (AM mission) — hidden in a captured Allied lab | Einstein's journal: *"I kept one copy of my Trinity notes. I told myself it was insurance — that if things became desperate enough, I would consider rebuilding the device. I pray that day never comes. But I see how this war is going, and I am losing faith in prayer."* |
| **Fragment 5: Trinity Test Report** | Poland operations (AM mission) — Soviet archive | Original 1946 test report: *"Test designation: TRINITY-TEMPORAL. Subject successfully displaced from 1946 to 1924. Target individual removed from timeline. Side effects: unknown. Researcher's note: 'It worked. God help us all. — A. Einstein'"* |

**Fragment discovery UX:** When the player finds a fragment, a brief notification appears: *"Intel discovered: Einstein's Research Fragment 3/5"*. The fragments are readable in the campaign journal/intermission screen. They're short (2-3 sentences each) — enough to build the picture without interrupting gameplay.

**Act 2 (after M8) — The Confession**

The `Einstein's Confession` mission (documented in the campaign graph) is always the narrative reveal. If the experimental D078 add-on is enabled, it also offers the rebuilt prototype gameplay unlock.

**If the player found fragments:** Einstein's briefing is richer. He references specific things the player already discovered:

> *"You've seen the Soviet files. You know what I did in Trinity. You've read my journal — you know I kept one copy of the notes. I cannot hide from you any longer. Here is what I built."*

The player feels rewarded for their exploration — they pieced the story together before the reveal.

**If the player found no fragments:** Einstein's confession is the first the player hears of Trinity. It still works — but it's a surprise rather than a confirmation. The fragments are enrichment, not prerequisites.

**Act 3 — The Weight**

If the experimental time-machine add-on is enabled and the player uses the capability, Einstein reacts. A brief briefing line after each use:

- First use: *"It worked. Commander... please use this power sparingly. I know what it costs."*
- Second use: *"Again? I... understand. The stakes are high. But each use destabilizes the temporal field. There may not be a third."*
- If the player uses all charges: *"It is done. The device is spent. Perhaps that is for the best."*

**Epilogue — Closure**

In the Allied epilogue mission ("Aftermath"), Einstein appears one final time. His dialogue depends on whether the player used the time machine:

- **Used it:** *"You used my device. I don't blame you — I used it too, once. The difference is that you used it to save lives. I used it to play God. Perhaps that makes you wiser than me."*
- **Never used it:** *"You won this war without my device. You have no idea how much that means to me. Perhaps humanity doesn't need to meddle with time to find its way."*
- **Never unlocked it (skipped the Confession mission):** *"Commander, there is something I never told you about my past. Perhaps someday. But today... today we celebrate. The war is over."*

### Soviet Campaign Mirror

The Soviet campaign has a parallel thread — seen from the other side:

- **M7 briefing:** Soviet intelligence references Einstein's "pre-war research" as a capture priority
- **Paradox Equation (CS):** Temporal anomalies foreshadow that Einstein's deeper research exists
- **Temporal Experiment (after M13):** Soviet scientists recover Einstein's notes and build their own prototype. Nadia's briefing: *"The Americans buried this knowledge. We will use it."*
- **Soviet epilogue:** If the experimental time-machine add-on was used, the ending references temporal instability — a hint that the Soviets' reckless experimentation will have consequences beyond this war (foreshadowing RA2/RA3 timeline)

### Implementation

The thread uses existing and one new UI surface:
- **Fragments:** Campaign flags (`einstein_fragment_1: true` through `einstein_fragment_5: true`) set when the player enters a trigger zone in the mission map — standard Lua trigger, no new system
- **Conditional briefing lines:** Same `conditional_lines` system documented in § Side Mission Rewards — no new system
- **Fragment reader (new UI):** A new "Intel" tab in the campaign intermission screen displays collected fragments as short readable entries. The current intermission screen (`single-player.md`) exposes roster/heroes/stats — the Intel tab is an addition, not an existing viewer. Small scope: a list of 2-3 sentence text entries, readable between missions. Requires a new tab in the intermission UI but no new campaign state infrastructure beyond the flags already defined

---

## Narrative Framework — Making the Lore Coherent

### The Questions the Original Never Answered

The original RA1 and its expansions leave significant narrative gaps. Some are worth filling; others are better left mysterious. The Enhanced Edition's job: fill the gaps that hurt the story, leave open the mysteries that enrich it.

#### Design Principle: Cutscene Alignment

The original FMV cutscenes (briefing videos with live actors) are preserved exactly as they are — IC doesn't re-film them. Instead, the Enhanced Edition **aligns gameplay to make the cutscenes land better**. Every cutscene moment that felt abrupt or unexplained in the original gets a gameplay setup so the video makes sense in context.

**Key example — M1 briefing video:**

The original M1 video shows Stavros objecting to Tanya's presence: *"General Von Esling, she is a civilian!"* In the original game, this is the player's first encounter with BOTH characters — neither introduction lands because the player has no context for either person.

The Enhanced Edition plays two prologue missions before M1: one with Stavros (Fall of Greece) and one with Tanya (First Blood). When the M1 briefing video plays, the player recognizes both characters. Stavros's objection becomes a meaningful character moment: the player fought alongside him in Greece and understands his military pride. Tanya's cocky reply ("That's why I don't get killed") resonates because the player just saw her rescue POWs. The same unmodified video, but now it has emotional weight.

**This principle applies throughout:** IC missions are designed so that original cutscenes feel like they were always meant to be watched at this point in the story. The videos are untouchable; the gameplay around them is what changes.

#### Questions We Answer (fills narrative gaps)

**Q: How did Einstein get captured by the Soviets?**
The original intro shows Einstein using a time machine in Trinity, New Mexico (1946) to remove Hitler. He reappears in the RA timeline as a prisoner of the Soviets in Allied Mission 1. The game never explains how he went from Trinity to a Soviet firing squad.

IC Enhanced Edition answer: Einstein is the same person — he remembers the original timeline. After removing Hitler, he found himself in a world where the Soviets expanded unchecked. Wracked with guilt, he tried to help the European Allies covertly, developing the Chronosphere from his time machine research. The Soviets discovered his work, invaded Switzerland (where he was hiding), and captured him. Allied Mission 1 picks up at the moment of his scheduled execution. The IC prologue mission can show Einstein's final days in Switzerland — the player helps evacuate his research before the Soviets close in, but Einstein himself is captured to protect the escaping data.

**Q: How did Tanya get captured between Allied M4 and M5?**
Addressed in the campaign graph: IC adds "Behind Enemy Lines" — the mission where Tanya infiltrates a Soviet facility for Iron Curtain intel. If Tanya succeeds or escapes, no rescue branch opens. If she is captured, the M5 rescue branch becomes available and the later penalties depend on how long the player waits.

**Q: Why does Tanya appear at the Soviet nuclear facility in Soviet M7?**
In the original, Tanya just shows up at "Core of the Matter" with no explanation. IC links this to the Allied campaign: if the Allied "Behind Enemy Lines" mission was played, Tanya's presence is the Allied side of that same operation. From the Soviet perspective, an enemy commando has infiltrated the facility.

**Q: What are the expansion missions about? They seem random.**
The Counterstrike/Aftermath missions were designed as standalone challenges with no campaign integration. IC gives them narrative purpose:

| Expansion Missions | Original Context | IC Narrative Integration |
|---|---|---|
| **Sarin Gas trilogy** (CS Allied) | Standalone: destroy chemical facilities in Greece | Einstein's intel from M1 reveals a parallel Soviet weapons program beyond the Iron Curtain — chemical weapons. The Sarin facilities are part of the same research complex that produced the Iron Curtain technology. Destroying them prevents chemical attacks during the Chronosphere defense (M8) |
| **Fall of Greece** (CS Allied) | Standalone: Stavros escapes occupied Greece | Stavros's personal story — his homeland falls, he barely escapes. This is Act 1 content that establishes Stavros's character and motivation (he's driven by the loss of Greece). His rescue contacts become the spy network used later in M5-M6 |
| **Siberian Conflict** (CS Allied) | Standalone: establish Allied presence in Siberia | Strategic flanking operation — the Allies open a second front to divide Soviet forces. A long-arc side campaign that spans Acts 1-3, with compound rewards (Siberian front collapse weakens Soviet reinforcements in the final Moscow assault) |
| **Volkov & Chitzkoi** (CS Soviet) | Standalone: use a cybernetic super-soldier | The Soviet answer to Tanya. Where the Allies have a skilled human operative, the Soviets build a machine. Volkov is the Soviet hero character with his own progression arc. His missions parallel Tanya's — a mirror narrative |
| **Legacy of Tesla** (CS Soviet) | Standalone: capture an Allied prototype | The arms race escalates — both sides steal each other's technology. This is the Soviet side of the tech war that the Allied spy missions represent |
| **Paradox Equation** (CS Soviet) | Standalone: Chronosphere causes weird effects | The Chronosphere's teleportation mechanics interact with residual temporal energy from Einstein's original 1946 time machine research (separate technology, same scientific lineage). Tanks behaving as different units is a "temporal bleed" — the Chronosphere accidentally tapping into time-displacement side effects. Foreshadows that Einstein's deeper research exists |
| **Giant Ants** (CS secret) | Standalone: giant mutant ants attack | Radiation from an abandoned Soviet nuclear facility mutated local ant colonies. The facility was part of the same nuclear program that powers the Iron Curtain. The ants are an unintended consequence of the superweapon arms race — nature fights back |
| **Aftermath Italy** (AM Allied) | Standalone: Mediterranean operations | The war expands south. Italy becomes a second theater as the Allies push to cut Soviet supply lines through the Mediterranean. Chrono Tank prototypes recovered here become available for the endgame |
| **Aftermath Poland** (AM Allied) | Standalone: Polish operations | The push toward Moscow goes through Poland. Liberating Poland secures the Allied flank and provides resistance fighters for the final assault |
| **Aftermath France/Spain** (AM Soviet) | Standalone: Western European operations | The Soviets consolidate control of Western Europe. These missions show the occupation from the Soviet side — suppressing resistance, testing new weapons, dealing with internal dissent |
| **Deus Ex Machina** (AM Soviet) | Standalone: rescue Volkov | Volkov was captured by the Allies after a failed operation. This becomes a narrative continuation of the Volkov arc — if Volkov is captured, this is the rescue mission. If he was never captured, this mission doesn't appear |

**Q: What's Kane doing in the Soviet ending?**
The original reveals Kane (from Tiberian Dawn) advising Nadia after she poisons Stalin. IC leaves this mystery intact — it's too important to the larger C&C universe to alter. But IC adds subtle foreshadowing: in the Soviet "Stalin's Shadow" side mission, the player encounters references to a mysterious advisor who has been influencing Soviet politics from behind the scenes. Players who recognize Kane from Tiberian Dawn get the connection; others experience it as a Cold War thriller twist.

#### Questions We Leave Open (enriches the mystery)

**Q: Is this the same Einstein, or an alternate version?**
The game's intro shows Einstein physically walking into the past and shaking Hitler's hand before Hitler vanishes. This is the *same* Einstein — he remembers both timelines. But IC doesn't over-explain this. Einstein's dialogue in briefings can hint at his burden ("I have seen what happens when we meddle with time...") without spelling out the metaphysics. The player should feel the weight, not read a physics lecture.

**Q: Could the Chronosphere undo all of this?**
Einstein knows the Chronosphere *could* theoretically reverse his original intervention — send someone back to prevent him from removing Hitler. But he also knows the consequences of another temporal intervention could be even worse. This tension — "we have the power to fix everything, but using it might break everything worse" — is the thematic core of the time machine campaign mechanic (D078). Some players will wonder "why not just use the Chronosphere to fix it all?" The answer is that Einstein refuses, and the Enhanced Edition campaign shows why.

**Q: What happened in the "original" timeline that Einstein left?**
Without Hitler, WWII as we know it never happened. But what filled the vacuum? IC leaves this deliberately unanswered — it's the road not taken. However, the "Paradox Equation" mission (CS Soviet) shows temporal anomalies where the "original" timeline bleeds through — ghostly echoes of a world that might have been. This is atmospheric, not explanatory.

### How IC Features Connect to the Story

The Enhanced Edition doesn't just bolt new features onto old missions — each IC feature has a **narrative reason** for existing within the RA1 story.

| IC Feature | Narrative Justification | Story Role |
|---|---|---|
| **Hero progression (Tanya)** | Tanya is a legendary commando whose skills develop across the war. Her reputation grows with each mission — soldiers talk about her, Soviets fear her | The player's investment in Tanya's skill tree mirrors the in-universe legend building. Losing her is emotionally devastating because you built her up |
| **Hero progression (Volkov)** | The Soviets' answer to Tanya — a machine built to match a legend. But is he still human? His progression represents the Soviets investing in technology over humanity | Volkov's upgrades are the Soviet tech tree made personal. Each skill point is another piece of his humanity replaced by machinery |
| **Time machine (D078, experimental add-on)** | Einstein's original 1946 time travel research is rediscovered. If D078 graduates from Draft, the narrative reveal can also unlock a rebuilt prototype that transmits information back in time | The time machine is Einstein's worst fear: his original Trinity research weaponized. The campaign explores "what if someone rebuilt the device he used to remove Hitler?" |
| **Branching decisions** | War is chaos. Plans change. The commander makes choices with incomplete information, and those choices have consequences | Decision points aren't meta-game options — they're the fog of war. The briefing presents two plans; the commander picks one. Neither is "right" |
| **Spectrum outcomes** | Missions don't end in binary victory/defeat. Partial success, pyrrhic victory, tactical retreat — all are valid outcomes | This is how real military operations work. "We took the objective but lost half our force" is a common outcome, and the campaign should reflect it |
| **Dynamic weather (D022)** | The Eastern Front was defined by weather — Russian winters destroyed Napoleon and nearly destroyed the Wehrmacht. Weather should matter in a war set in Europe | The Moscow blizzard in M14, the Channel storm in Soviet M14, the Mediterranean heat — weather isn't decoration, it's strategy |
| **Asymmetric co-op (D070)** | Special operations require coordination between a strategic commander and a tactical field team. This is how modern militaries actually operate | Co-op missions are the most realistic depiction of combined-arms warfare in an RTS. One player thinks strategically; the other thinks tactically |
| **Spy infiltration** | Intelligence won the war as much as firepower. The Allied spy network (MI6, OSS) was critical to defeating the Axis — and in the RA timeline, the Soviets | Spy missions aren't filler — they're the invisible war that determines whether the visible war is winnable |
| **Air campaigns** | Air superiority was the decisive factor in WWII. The side that controlled the skies controlled the battlefield | Air campaign missions are the player experiencing what it means to *not* have boots on the ground — you support, you coordinate, but you can't hold territory from the air |
| **Campaign menu scenes** | The war is always present. Even when the player is in the menu, the campaign's mood follows them | The evolving menu background isn't a feature — it's the war haunting you. Act 1's hopeful naval scene gives way to Act 3's grim ground assault. The menu tells you where the war is |
| **Failure as consequence** | In war, failure doesn't mean "retry." It means "deal with what happened and keep going" | The bridge isn't destroyed? The enemy crosses it. Tanya is captured? Go rescue her — or don't, and fight without your best soldier. The campaign adapts to your failures |

### Expansion Missions — The Unified Story

With narrative integration, the 34 expansion missions stop being random standalone challenges and become chapters in a larger war story:

**Act 1 — The War Begins (M1-M4 + Fall of Greece + early Aftermath)**
The war erupts across Europe. Greece falls (Stavros's personal tragedy). The Allies scramble to respond. Einstein is rescued. Bridges are destroyed. Lines are drawn.

**Act 2 — The Arms Race (M5-M9 + Sarin Gas + Volkov + Italy)**
Both sides escalate: the Allies develop the Chronosphere; the Soviets build the Iron Curtain, develop chemical weapons (Sarin), and create Volkov. The Mediterranean becomes a second theater. Intelligence operations (spies, infiltration) become as important as firepower.

**Act 3 — Total War (M10-M14 + Siberian + Poland + final operations)**
Everything converges. The Siberian front collapses. Poland is liberated. The Allied fleet clears the river. Every side mission's reward (or absence) shapes the final assault. The war ends in Moscow (Allied) or London (Soviet).

**Epilogue — Consequences**
The war is over, but what comes next? Allied: cleanup operations with moral decisions. Soviet: the internal power struggle (Stalin/Nadia/Kane). Both: the Ant crisis emerges from the war's radiation legacy.

---

## Allied Enhanced Edition Campaign

### Narrative Gaps Identified in the Original

| Gap | Between | Problem | IC Fix |
|-----|---------|---------|--------|
| **Tanya captured off-screen** | M4 → M5 | M4 ends with a defensive victory. M5 briefing says Tanya was "captured during an intelligence operation." The capture is never shown — the player goes from defending a pass to suddenly rescuing Tanya from prison | IC adds the mission where Tanya gets captured: a spec-ops intelligence mission where success means no rescue branch opens, while capture opens the M5 rescue branch and starts the escalation clock |
| **Supply convoy disconnected** | M2 → M3 | M2 clears Soviet forces for a convoy. M3 destroys bridges. No narrative link between them | IC links them: the supply convoy in M2 was carrying demolition equipment for the bridge mission in M3. Side mission to secure additional explosives affects M3's difficulty |
| **Spy arc appears from nowhere** | M5 → M6 | M5 uses a spy to rescue Tanya. M6 is a spy infiltration mission. The spy capability isn't introduced — it just appears | IC adds a spy recruitment side mission between M4-M5 where the player establishes the spy network |
| **Naval missions disconnected** | M7 | Submarine pen destruction feels disconnected from the Iron Curtain investigation arc | IC links it: intel from M6 reveals the Iron Curtain components are being shipped by submarine. Destroying the sub pens cuts the supply line |
| **Abrupt format change** | M10A → M10B | Jump from outdoor base-building to interior commando mission with no transition | IC adds a transition briefing and an optional prep mission (reconnoiter the facility exterior before the interior raid) |
| **Naval gap** | M11 | Naval supremacy mission appears without setup for why naval control matters now | IC links it: Kosygin's intel (M9) reveals the final Soviet defenses require a naval approach — the river must be cleared for the ground assault |
| **Final push too linear** | M12 → M13 → M14 | Three missions in a row with no branching, no decisions, just "destroy everything" | IC adds decision points and alternative approaches for the endgame |

### Full Campaign Graph

```
PROLOGUE — STAVROS'S WAR (IC Original + Counterstrike)
│  Character intro: STAVROS (before any cutscene appearance)
│  Character intro: TANYA (before M1 cutscene)
│  Character intro: EINSTEIN (voice on radio during Fall of Greece)
│
├─ [CS] "Personal War" (Fall of Greece 1) ★ VERY EASY
│  │  Stavros's intro. Parachutes into occupied Athens to rescue family.
│  │  Player commands small resistance force. Teaches: basic movement,
│  │  unit selection, named character escort.
│  │  Einstein heard on radio: "Stavros, the Soviets have a weapon...
│  │  I've seen what they're capable of. Get your people out."
│  │  Menu scene: Mediterranean coast at dawn
│  │
│  └─ [CS] "Evacuation" (Fall of Greece 2) ★ EASY
│     Stavros evacuates Greek civilians before Soviets close the ports.
│     Teaches: time pressure, multi-objective, civilian escort.
│     Reward: Greek resistance contacts (spy network foundation)
│
├─ [IC] "Tanya: First Blood" ★ EASY-MEDIUM
│  │  Tanya rescues Allied POWs from a Soviet outpost.
│  │  Teaches: hero abilities, skill points, C4 charges.
│  │  First skill tree selection at mission end.
│  │  Menu scene: naval convoy at dawn
│  │
│  CUTSCENE ALIGNMENT: M1 briefing video plays here.
│  Stavros (seated) → player recognizes him from Fall of Greece.
│  Tanya (walks in) → player recognizes her from First Blood.
│  "She is a civilian!" — now a clash between two known characters.
│
ACT 1: LIBERATION OF EUROPE ─────────────────────────────────────────
│
├─ [M1] "In the Thick of It" ★ MEDIUM — Rescue Einstein.
│  │  Teaches: combined arms (Tanya + support units), base assault.
│  │  Einstein becomes a character the player heard in the prologue.
│  │
│  └─ [IC] "Supply Line Security" ★ MEDIUM — OPTIONAL
│     Escort Chrono tech convoy. Teaches: escort + defend.
│     Reward: Chrono reinforcement available in M4
│
├─ [M2] "Five to One" ★ MEDIUM — Clear road for supply convoy.
│  │  IC link: convoy carries demolition equipment for M3.
│  │  Teaches: time-limited clearing, area control.
│  │  Spectrum: convoy intact → full explosives for M3.
│  │  Convoy damaged → M3 is harder (fewer charges).
│  │
│  └─ [CS] "Fresh Tracks" (Siberian 1) ★ MEDIUM — OPTIONAL
│     Opens Siberian flanking arc (long-term investment).
│     Reward: Siberian arc unlocked for Act 3
│
├─ [M3] "Dead End" ★ MEDIUM — Destroy bridges. Tanya must survive.
│  │  Teaches: stealth, terrain objectives, hero preservation.
│  │  Spectrum: all destroyed → M4 easy / some → M4 harder /
│  │  none → M4 very hard. First real spectrum outcome.
│
├─ [M4] "Ten to One" ★ MEDIUM-HARD — Defend the pass.
│  │  First genuinely hard mission — but player has 6+ missions'
│  │  experience by now. Difficulty scales with M3 bridge results.
│  │  If Greek resistance contacted: partial shroud reveal.
│  │
│  ═══ TIMED CHOICE 1 ═══════════════════════════════════════════
│  │  "Three fires burning — which one do you fight?"
│  │  The world moves without you. Pick ONE. The other two
│  │  resolve with consequences.
│  │
│  │  OPTION A: [IC] "Behind Enemy Lines" ★ HARD
│  │  │  Tanya infiltrates Soviet facility for Iron Curtain intel.
│  │  │  Type: spy_infiltration. Teaches: detection/alert system.
│  │  │  Spectrum outcomes:
│  │  │  ├─ Success + escape → No rescue branch opens. Full intel for M6.
│  │  │  ├─ Success + captured → M5 rescue branch opens. Partial intel for M6.
│  │  │  ├─ Failed + escaped → No intel. M6 blind. Tanya available.
│  │  │  └─ Failed + captured → M5 rescue branch opens + M6 blind. Worst case.
│  │  │
│  │  IF NOT CHOSEN → Tanya is captured off-screen (world moved).
│  │  │  M5 (Rescue Tanya) enters a pending rescue queue.
│  │  │  The player can take it now, after M6, or ignore it entirely.
│  │  │  Each intervening mission escalates the intel leak (Rule 6).
│  │  │  After the rescue window closes, Tanya is lost and the
│  │  │  severe compromise state becomes permanent.
│  │  │  No intel for M6. Briefing: "Tanya went behind enemy lines
│  │  │  without support. She's been captured."
│  │
│  │  OPTION B: [CS] "Crackdown" (Sarin Gas 1) ★ HARD
│  │  │  Neutralize Sarin gas facilities in Greece.
│  │  │  IF CHOSEN → No chemical attacks in M8.
│  │  │  IF NOT CHOSEN → Sarin goes active. M8 includes gas attacks.
│  │  │  Briefing: "Chemical weapons reports from the eastern front.
│  │  │  We didn't act in time."
│  │
│  │  OPTION C: [CS] "Fresh Tracks" (Siberian 1) ★ MEDIUM
│  │     Open second front in Siberia.
│  │     IF CHOSEN → Siberian arc available in Act 3.
│  │     IF NOT CHOSEN → Siberian window closes permanently.
│  │     Briefing: "The Siberian opportunity has passed."
│  │     (Only appears if not already completed as optional earlier)
│  │
│  [IC] "Spy Network" ★ MEDIUM — OPTIONAL (if Tanya escaped)
│     Recruit a spy network. Reward: spy for M6, intel.
│
├─ [M5] "Tanya's Tale" ★ MEDIUM — Rescue Tanya. OPERATIVE LANE.
│  │  AVAILABLE (not mandatory) if Tanya was captured.
│  │  May be taken immediately or delayed for a limited number of missions.
│  │  Each delay increases compromise level and later-mission penalties.
│  │  Ignoring it entirely leaves Tanya lost and the Soviet intel haul intact.
│  │  Commander alternative: send an armored assault to the prison
│  │  compound. Louder, Tanya is rescued but wounded (unavailable
│  │  for 1 mission). Operative approach: spy+commando, clean rescue.
│  │  If Tanya was never captured → M5 doesn't appear. Skip to M6.
│  │  Menu scene: if captured → dark war room / if free → confident HQ
│
ACT 2: THE IRON CURTAIN ─────────────────────────────────────────────
│
├─ [M6] "Iron Curtain Infiltration" ★ MEDIUM-HARD — OPERATIVE LANE.
│  │  Spy infiltrates Soviet Tech Center for Iron Curtain intel.
│  │  Difficulty varies: full intel → spy has access codes.
│  │  No intel → blind infiltration, tighter timers.
│  │  Commander alternative: full assault on the Tech Center. You
│  │  destroy it and recover partial intel from the wreckage — less
│  │  intelligence than the spy route, but no commando gameplay.
│  │  Character intro: KOSYGIN mentioned in intercepted comms.
│  │
│  ├─ [CS] "Down Under" (Sarin Gas 2) ★ HARD — OPTIONAL
│  │  │  Continue Sarin campaign. Only if Sarin 1 was chosen/completed.
│  │  │  Reward: Greek theater secured → bonus naval units for M7
│  │  │
│  │  └─ [CS] "Controlled Burn" (Sarin Gas 3) ★ HARD — OPTIONAL
│  │     Capture Sarin facilities. Requires Sarin 2.
│  │     Reward: chemical expertise → special weapon in Act 3
│  │
│  ├─ [AM] "Harbor Reclamation" (Italy 1) ★ MEDIUM — OPTIONAL
│  │  Secure Italian harbor. Reward: Mediterranean staging base
│  │
│  └─ [AM] "In the Nick of Time" + "Caught in the Act" +
│     "Production Disruption" (Italy 2-4) ★ MEDIUM-HARD — OPTIONAL
│     Italian theater chain. Compound reward: Chrono Tank prototypes
│     + southern flank secured for final assault
│
├─ [M7] "Sunken Treasure" ★ MEDIUM-HARD — Destroy sub pens.
│  │  IC link: M6 intel → Iron Curtain parts shipped by sub.
│  │  If Sarin missions done: bonus naval units.
│  │  If Italy done: Mediterranean staging base support.
│  │  Teaches: naval combat.
│  │
│  └─ [IC] "Operation Skyfall" ★ HARD — OPTIONAL air campaign
│     Coordinate air strikes on AA around Chronosphere perimeter.
│     Type: air_campaign. Teaches: sortie management, no base.
│     Reward: M8 starts with AA destroyed (fewer air attacks).
│
├─ [M8] "Protect the Chronosphere" ★ HARD — Defend 45 minutes.
│  │  If Sarin NOT neutralized → chemical attacks during assault.
│  │  If Skyfall completed → Soviet AA pre-destroyed.
│  │  Dynamic weather: storm at minute 30 (D022 showcase).
│  │
│  └─ [IC] "Einstein's Confession" ★ MEDIUM — OPTIONAL
│     After the Chronosphere defense, Einstein reveals something
│     he's kept hidden: his original time machine research from
│     1946 — the device he used to remove Hitler. His notes still
│     exist. He buried them after seeing what his intervention
│     created. Now, with the war reaching a crisis, he's reconsidered.
│
│     Einstein's briefing:
│     "Commander... my original research — the experiment I conducted
│     in Trinity, New Mexico — I destroyed my notes after I saw what
│     I had done to the world. But I kept one copy. I told myself it
│     was insurance. Now I am not sure if offering it to you is
│     courage or cowardice."
│
│     Einstein has rebuilt a limited prototype from his original
│     research. It can only transmit information — tactical intel,
│     warnings, exploration data.
│
│     EXPERIMENTAL D078 ADD-ON ONLY:
│     If D078 graduates from Draft, this mission can also unlock
│     the TIME MACHINE capability (Layer 2):
│     - 1-2 uses available for Act 3 (limited, not infinite)
│     - Can rewind to a previous Act 3 mission-start checkpoint
│     - Knowledge (exploration, intel flags) carries back
│     - Army does NOT carry back (knowledge > power)
│     - Butterfly effects: enemy AI adapts ("something has changed")
│
│     If D078 stays Draft / cut, the mission remains a pure narrative
│     reveal with no gameplay unlock. The classic campaign path works
│     either way.
│
│     Menu scene changes: Einstein's lab with prototype device
│
├─ [M9] "Extract Kosygin" ★ MEDIUM-HARD — OPERATIVE LANE.
│  │  Spy infiltrates Soviet compound, frees defector, escorts out.
│  │  Character payoff: heard Kosygin in M6 comms. Now rescue him.
│  │  Commander alternative: armored extraction. Tank column blasts
│  │  into the compound, grabs Kosygin. Louder, more casualties,
│  │  Kosygin provides less intel (roughed up in the fighting).
│  │  His intel quality drives the endgame.
│  │
│  ═══ TIMED CHOICE 2 ═══════════════════════════════════════════
│  │  "Where do we push next? Pick one front."
│  │
│  │  OPTION A: [AM] "Monster Tank Madness" (Poland 1) ★ HARD
│  │  │  Rescue Dr. Demitri + Super Tanks.
│  │  │  IF CHOSEN → 2 Super Tanks for M14. Unlocks Poland 2-5.
│  │  │  IF NOT CHOSEN → No Super Tanks. Poland arc closed.
│  │
│  │  OPTION B: [IC] "Air Superiority" ★ HARD
│  │  │  Establish air dominance over the eastern front.
│  │  │  Type: air_campaign.
│  │  │  IF CHOSEN → Air support available in M12 and M14.
│  │  │  IF NOT CHOSEN → No strategic air support in endgame.
│  │
│  │  OPTION C: [CS] "Trapped" (Siberian 2) ★ HARD
│  │     Continue Siberian arc (only if Fresh Tracks was done).
│  │     IF CHOSEN → Siberian front weakens. Unlocks Wasteland.
│  │     IF NOT CHOSEN → Siberian gains lost. No Act 3 benefit.
│
│  [AM] Poland 2-5 ★ HARD — OPTIONAL (if Poland 1 was chosen)
│  │  "Negotiations" / "Time Flies" / "Absolute M.A.D.ness" / "PAWN"
│  │  Compound reward: Poland liberated → flanking + resistance
│
│  [CS] "Wasteland" (Siberian 3) ★ HARD — OPTIONAL (if Siberian 2 done)
│     Final Siberian operation. Reward: Siberian front collapses →
│     Soviet M14 reinforcements halved.
│
ACT 3: ENDGAME ──────────────────────────────────────────────────────
│  Menu scene: ground assault — tanks, artillery, dark skies
│
├─ [M10A] "Suspicion" ★ HARD — Destroy Soviet nuclear silos.
│  │  COMMANDER LANE — base building, army assault. Always available.
│  │
│  │  After M10A, choose approach for the underground facility:
│  │  ├─ [OPERATIVE] [M10B] "Evidence" ★ HARD — Interior commando.
│  │  │  Tanya infiltrates underground facility. Better intel yield.
│  │  │
│  │  ├─ [OPERATIVE] [IC] "Evidence: Enhanced" ★ HARD
│  │  │  Embedded task force: Tanya + squad inside a live battle.
│  │  │
│  │  └─ [COMMANDER] [IC] "Evidence: Siege" ★ HARD
│  │     Bomb the facility from above. Artillery + air strikes
│  │     collapse the underground complex. Cruder but no commando
│  │     gameplay. Reduced intel yield (facility partially destroyed).
│
├─ [M11] "Naval Supremacy" ★ HARD — Clear river for final push.
│  │  IC link: Kosygin's intel → this is the only approach.
│  │
│  └─ [IC] "Joint Operations" ★ HARD — OPTIONAL (Phase 6b add-on, D070)
│     Co-op variant: Commander (naval base) + SpecOps (shore batteries).
│     Not part of the Phase 4 baseline — requires D070 co-op infrastructure.
│
├─ [M12] "Takedown" ★ VERY HARD — Destroy Iron Curtain bases.
│  │  If Poland done → flanking approach (attack from two sides).
│  │  If Italy done → no southern naval threat.
│  │  If air superiority → bombing runs available.
│  │
│  ═══ TIMED CHOICE 3 ═══════════════════════════════════════════
│  │  "One last operation before Moscow. Make it count."
│  │
│  │  OPTION A: [OPERATIVE] [M13] "Focused Blast" ★ VERY HARD
│  │  │  Interior commando. Plant charges in underground facility.
│  │  │  IF CHOSEN → Iron Curtain facility destroyed from inside.
│  │
│  │  OPTION B: [OPERATIVE] [IC] "Focused Blast: Enhanced" ★ VERY HARD
│  │  │  Tanya's full skill tree affects the mission.
│  │  │  Silent Step → easier stealth. Chain Detonation → fewer
│  │  │  charges needed. Hero progression payoff.
│  │  │  (Only available if Tanya is alive and level 3+)
│  │
│  │  OPTION C: [COMMANDER] [IC] "Iron Curtain: Siege" ★ VERY HARD
│  │     Full base assault + artillery bombardment of the facility.
│  │     No commando gameplay. Same narrative result (facility destroyed).
│  │     Trade-off: higher casualties, no precision destruction —
│  │     some Iron Curtain tech survives (enemy retains partial
│  │     Iron Curtain capability in M14).
│
├─ [M14] "No Remorse" ★ VERY HARD — Final assault on Moscow.
│  │  EVERYTHING ACCUMULATES HERE:
│  │  ├─ Siberian arc done → reduced reinforcements
│  │  ├─ Poland liberated → Polish resistance arrives
│  │  ├─ Super Tanks → available as heavy armor
│  │  ├─ Sarin expertise → chemical defense
│  │  ├─ Italy secured → no southern counterattack
│  │  ├─ Air superiority → bombing runs
│  │  ├─ Tanya max level → leads assault squad
│  │  ├─ Nothing done → MAXIMUM difficulty, no bonuses
│  │  Briefing dynamically references every choice.
│  │
│  └─ [IC] "Moscow: Enhanced" ★ VERY HARD — ALTERNATIVE
│     Dynamic weather (blizzard). Embedded task force (Kremlin
│     infiltration during siege). Co-op option (Phase 6b add-on, D070).
│
EPILOGUE ────────────────────────────────────────────────────────────
│
├─ [IC] "Aftermath" ★ MEDIUM — Post-victory moral decisions
│  Cleanup in Moscow. Choices affect ending.
│  Menu scene: sunrise over liberated city
│
└─ [ANT] "It Came From Red Alert!" ★ VARIES — Secret campaign
   Unlocked after Allied completion. 4 ant missions.
   Uses campaign roster from completion.
```

### Allied Campaign Summary

| Content Type | Count | Toggleable |
|---|---|---|
| Classic main missions | 14 | Always on |
| IC-original missions | ~8 (prologue, Behind Enemy Lines, Spy Network, Skyfall, Joint Ops, enhanced alternatives, epilogue) | Per-mission |
| Counterstrike missions | 8 (Sarin Gas 1-3, Fall of Greece 1-2, Siberian 1-3) | Per-chain |
| Aftermath missions | 9 (Italy 1-4, Poland 1-5) | Per-chain |
| Ant campaign | 4 | Post-completion unlock |
| **Total missions available** | **~43** | — |
| **Minimum path (classic only)** | **14** | — |

---

## Soviet Enhanced Edition Campaign

### Narrative Gaps Identified in the Original

| Gap | Between | Problem | IC Fix |
|-----|---------|---------|--------|
| **Abrupt spy chase** | M2 → M3 | Guard duty (infantry bridges) → suddenly chasing a spy. No narrative bridge | IC adds context: the spy was spotted during M2's operations. Optional mission to track the spy's contacts before the chase |
| **Berlin jump** | M3 → M4 | Spy chase in unknown location → assault on Berlin. Huge geographic/narrative leap | IC adds an optional advance-to-Berlin logistics mission |
| **Generic base building** | M5 | "Distant Thunder" is a standalone base-building exercise with no story purpose | IC links it: this base becomes the staging ground for M6's convoy escort. The base you build in M5 is the base you defend in M6 |
| **Tanya appears from nowhere** | M7 | "Core of the Matter" — Tanya is sabotaging a nuclear facility, but there's no setup for her presence | IC adds context in the Soviet M7 briefing: Nadia explains that Allied commando activity was detected near the facility. This is always present — no cross-campaign data needed. Tanya's appearance is framed as a known Allied threat, not a surprise |
| **Internal politics jarring** | M8 → M9 | Jump from destroying the Chronosphere (war front) to killing a political rival (internal intrigue) | IC adds a transition: Kosygin's defection (Allied M9) is referenced here — the "liability" in M9 is someone who helped Kosygin escape |
| **Two escort missions back-to-back** | M9 → M10 | Both are "escort something across the map" — repetitive | IC offers an alternative to M10 (different mission type, same narrative purpose) |
| **Repetitive endgame** | M12-14 | Three "capture/destroy everything" missions in a row | IC adds variety: alternative approaches, co-op options, and decision points for the final push |

### Full Campaign Graph

```
PROLOGUE — THE PARTY'S WILL (IC Original + Aftermath)
│  Character intro: STALIN (voice in propaganda broadcast)
│  Character intro: NADIA (briefing officer, gives first orders)
│  Character intro: VOLKOV (Soviet hero, playable in prologue)
│  Character intro: GRADENKO (mentioned as political rival — foreshadowing)
│
├─ [IC] "Volkov: Awakening" ★ VERY EASY
│  │  Volkov + Chitzkoi activated for a field test. Destroy a small
│  │  Allied outpost. Teaches: hero unit control, special abilities.
│  │  Nadia briefs: "Comrade, the Party has invested greatly in you.
│  │  Do not disappoint." Stalin's voice on radio: propaganda.
│  │  Gradenko mentioned: "General Gradenko questions the Volkov
│  │  program's cost. Prove him wrong."
│  │  Menu scene: Soviet war room, red lighting, propaganda posters
│  │
│  └─ [AM] "Testing Grounds" (France 1) ★ EASY — OPTIONAL
│     Test new Soviet weapons. Introduces Shock Troopers.
│     Teaches: new unit types, combined arms.
│     Reward: Shock Troopers available from Act 1
│
ACT 1: THE IRON FIST ───────────────────────────────────────────────
│
├─ [M1] "Lesson in Blood" ★ EASY — Village assault. Tutorial.
│  │  Classic, enhanced with Nadia briefing context.
│  │  Teaches: basic base assault, unit production.
│
├─ [M2] "Guard Duty" ★ EASY-MEDIUM — Infantry bridge assault.
│  │  IC: spy spotted during battle (foreshadows M3).
│  │  If player kills spy → M3 is easier.
│  │  Teaches: multi-unit coordination, bridge tactics.
│  │
│  └─ [AM] "Shock Therapy" (Spain 1) ★ MEDIUM — OPTIONAL
│     Punish a border town. Reward: Shock Trooper veterancy boost
│
├─ [M3] "Covert Cleanup" ★ MEDIUM — OPERATIVE LANE. Chase a spy.
│  │  If spy killed in M2 → timer extended to 20 min.
│  │  Commander alternative: cordon and sweep. Deploy infantry
│  │  squads to lock down the district. Less elegant — spy may
│  │  escape with partial intel (worse for future missions).
│  │  Teaches: time pressure, pursuit, area sweep.
│  │
│  └─ [IC] "Mole Hunt" ★ MEDIUM — OPTIONAL
│     Hunt Allied agents inside Soviet command. Reversed spy mission.
│     Reward: future missions — Allies don't know your composition.
│     Character development: Nadia assigns this personally —
│     she's watching everyone. Foreshadows her true nature.
│
├─ [IC] "Road to Berlin" ★ EASY-MEDIUM — OPTIONAL logistics mission.
│  │  Bridges the geographic leap from the spy-chase theater to Berlin.
│  │  Advance Soviet forces through contested territory, secure a
│  │  staging area, and establish supply lines for the Berlin assault.
│  │  No base building — move existing forces across a multi-zone map.
│  │  Reward: M4 starts with a forward position (shorter approach,
│  │  more time to complete the radar objective).
│  │  If skipped: M4 starts from further back (harder time limit).
│
├─ [M4] "Behind the Lines" ★ MEDIUM — Destroy Allied radar in Berlin.
│  │  If Mole Hunt done → Allies don't expect you (no pre-defenses).
│  │  If Road to Berlin done → forward position, more time.
│  │  Teaches: deep strike behind enemy lines.
│  │
│  ├─ [AM] "Let's Make a Steal" (France 2) ★ MEDIUM — OPTIONAL
│  │  Steal Allied tech. Reward: vehicle tech upgrade.
│  │
│  └─ [AM] "Test Drive" (France 3) ★ MEDIUM — OPTIONAL
│     Test stolen vehicles. Reward: captured units in roster.
│
├─ [M5] "Distant Thunder" ★ MEDIUM — Build and defend a base.
│  │  IC link: surviving units and unspent resources carry over to M6
│  │  via D021 roster carryover (not literal base layout — structures
│  │  are map-specific). Over-investing in static defenses here means
│  │  fewer mobile units for M6's convoy escort.
│  │  Teaches: base building, resource management, investment trade-offs.
│  │
│  └─ [AM] "Don't Drink the Water" (France 4) ★ MEDIUM-HARD — OPTIONAL
│     Poison water supply, capture Chronosphere.
│     Reward: Chronosphere intel for M8.
│
├─ [M6] "Bridge over the River Grotzny" ★ MEDIUM-HARD — Convoy escort.
│  │  IC: surviving units and unspent resources from M5 carry over
│  │  via D021 roster carryover. M6 starts with a pre-built base
│  │  (map-authored) but your M5 troops are available as reinforcements.
│  │  Spectrum: full convoy → max supplies / partial → less /
│  │  lost → M7 starts with minimal forces.
│  │  Teaches: escort + defend with limited resources.
│  │
│  ═══ TIMED CHOICE 1 ═══════════════════════════════════════════
│  │  "The front is wide. Where do we strike?"
│  │
│  │  OPTION A: [CS] "Soldier Volkov & Chitzkoi" ★ HARD
│  │  │  Volkov commando mission behind Allied lines.
│  │  │  IF CHOSEN → Volkov gains veteran status for Act 2+.
│  │  │  IF NOT CHOSEN → Volkov stays in reserve (no veterancy).
│  │
│  │  OPTION B: [CS] "Legacy of Tesla" ★ HARD
│  │  │  Capture Allied nuclear MiG prototype.
│  │  │  IF CHOSEN → Prototype MiG available in M11 naval battle.
│  │  │  IF NOT CHOSEN → No air prototype. M11 is naval-only.
│  │
│  │  OPTION C: [AM] "Situation Critical" (Spain) ★ MEDIUM-HARD
│  │     Secure Mediterranean sea lanes.
│  │     IF CHOSEN → Naval advantage in M11. Unlocks Spain arc.
│  │     IF NOT CHOSEN → Allied naval threat from south persists.
│
ACT 2: SUPERWEAPONS ────────────────────────────────────────────────
│
├─ [M7] "Core of the Matter" ★ HARD — Nuclear facility vs Tanya.
│  │  Tanya's presence is always explained in the Soviet briefing
│  │  (IC adds context regardless of whether Allied campaign was played —
│  │  cross-campaign references are flavor text in briefing lines only,
│  │  not gating state. No global/legacy persistence layer needed).
│  │  If Volkov is veteran → Volkov vs Tanya encounter (hero vs hero).
│  │  If Volkov not available → standard mission, Tanya is a boss unit.
│  │  Teaches: facility defense + counter-commando operations.
│  │
│  └─ [AM] "Brothers in Arms" (Spain 2) ★ HARD — OPTIONAL
│     Soviet traitors. Only if Spain arc opened in timed choice.
│     Reward: loyal tank crews → heavy armor veterancy bonus.
│
├─ [M8] "Elba Island" ★ HARD — Destroy Allied Chronosphere.
│  │  If "Don't Drink the Water" done → know weak points.
│  │  If Volkov veteran → solo insertion option (sabotage before assault).
│  │  Teaches: combined naval + ground assault.
│  │
│  ├─ [CS] "Paradox Equation" ★ HARD — OPTIONAL
│  │  Chronosphere anomalies — tanks behave as other units.
│  │  Reward: Chrono tech understanding → advantage in M13.
│  │
│  └─ [CS] "Mousetrap" ★ HARD — OPTIONAL
│     Hunt Stavros at a Chronosphere research center.
│     Reward: Stavros eliminated → intel on Allied command structure.
│
├─ [M9] "Liability Elimination" ★ MEDIUM-HARD — OPERATIVE LANE.
│  │  Assassination mission. The "liability" helped Kosygin escape.
│  │  Character shift: the war turns inward. Stalin's paranoia.
│  │  Nadia's briefing is colder — she's consolidating power.
│  │  Commander alternative: arrest operation. Send an armored
│  │  column to detain the target publicly. Same political result
│  │  but cruder — creates more internal unrest (affects Act 3).
│  │
│  └─ [IC] "Stalin's Shadow" ★ HARD — OPTIONAL
│     Nadia's secret mission: evidence against Gradenko.
│     Foreshadows the Stalin/Nadia/Kane ending.
│     Reward: Gradenko neutralized → simpler politics in Act 3.
│     Character reveal: Nadia is not who she seems.
│
├─ [M10] "Overseer" ★ MEDIUM-HARD — Escort supply trucks.
│  │
│  ├─ [CLASSIC] M10 as-is ★ MEDIUM-HARD
│  │
│  └─ [IC] "Overseer: Strike" ★ HARD — ALTERNATIVE
│     Armored assault instead of escort. Same narrative result.
│     Teaches: offensive operations as alternative to escort.
│
ACT 3: CONQUEST ────────────────────────────────────────────────────
│  Menu scene: naval fleet mobilizing, Channel crossing preparation
│
├─ [M11] "Sunk Costs" ★ HARD — Naval defense vs Allied cruisers.
│  │  If MiG prototype captured → air support available.
│  │  If Spain secured → no Allied southern reinforcements.
│  │
│  ├─ [AM] "Deus Ex Machina" (Spain 3) ★ HARD — OPTIONAL
│  │  Rescue Volkov (if he was captured).
│  │  Can be taken immediately or after delay, but Allied tech extraction
│  │  escalates each mission he remains in custody.
│  │  Reward quality depends on how late the rescue happens.
│  │
│  └─ [AM] "Grunyev Revolution" (Spain 4) ★ MEDIUM-HARD — OPTIONAL
│     Crush a revolution. Reward: rear secured → no uprisings
│     during England invasion.
│
├─ [M12] "Capture the Tech Centers" ★ VERY HARD — Capture 3 centers.
│  │  Compound rewards from all side missions affect difficulty.
│  │
│  ═══ TIMED CHOICE 2 ═══════════════════════════════════════════
│  │  "The invasion of England approaches. One final preparation."
│  │
│  │  OPTION A: [IC] "Operation Tempest" ★ VERY HARD
│  │  │  Air/naval pre-invasion bombardment.
│  │  │  Type: air_campaign.
│  │  │  IF CHOSEN → M14 starts with damaged coastal defenses.
│  │  │  IF NOT CHOSEN → Full Allied coastal defense in M14.
│  │
│  │  OPTION B: [CS] "Nuclear Escalation" ★ VERY HARD
│  │     Prevent Allied air-fuel bomb testing.
│  │     IF CHOSEN → No Allied superweapon in M14.
│  │     IF NOT CHOSEN → Allied air-fuel bombs in M14.
│
├─ [M13] "Capture the Chronosphere" ★ VERY HARD
│  │  If Paradox Equation done → faster capture.
│  │  If Volkov available → commando approach option.
│  │
│  ├─ [IC] "Chronosphere: Enhanced" ★ VERY HARD — ALTERNATIVE
│  │  Co-op (Phase 6b add-on, D070): Commander sieges, SpecOps infiltrates.
│  │
│  └─ [IC] "Temporal Experiment" ★ HARD — OPTIONAL
│     Soviet intelligence recovers fragments of Einstein's original
│     1946 time travel research — notes hidden in a captured Allied
│     research facility. Soviet scientists build a crude prototype
│     from these notes. Less refined than Einstein's version, more
│     powerful, and dangerously unstable.
│
│     Nadia's briefing: "Our scientists found Einstein's original
│     time travel research in a captured laboratory. His notes
│     describe something extraordinary. The Americans buried this
│     knowledge. We will use it."
│
│     EXPERIMENTAL D078 ADD-ON ONLY:
│     If D078 graduates from Draft, this mission can also unlock the
│     TIME MACHINE capability (Layer 2):
│     - 1-2 uses available for M14 (the invasion)
│     - Can rewind to M14's mission-start checkpoint with knowledge
│     - Soviet version is more aggressive with KNOWLEDGE carryover:
│       full enemy patrol routes, base layouts, weapon positions carried
│       back (the Soviets squeeze every drop of intel from the temporal
│       transmission, where Einstein is more cautious)
│     - Army does NOT carry back — D078 Layer 2 rule, same for both factions
│     - Chrono Vortex hazards are Layer 3 / Phase 5 content — NOT included
│       in this Phase 4 campaign mechanic
│     - If Volkov was the test subject: his briefing dialogue hints
│       at residual temporal effects ("Something is... different.
│       I remember things I should not know.") — narrative flavor only
│
│     If D078 stays Draft / cut, the mission remains a narrative
│     demonstration of Soviet interest in Einstein's research with
│     no gameplay unlock.
│
│     Showcases: D078 campaign time machine with faction-specific rules
│     If Paradox Equation was completed: the temporal experiment
│     is more stable (fewer vortices — prior Chrono understanding helps)
│
├─ [M14] "Soviet Supremacy" ★ VERY HARD — Invade England.
│  │  EVERYTHING ACCUMULATES:
│  │  ├─ Spain secured → no southern counterattack
│  │  ├─ Volkov available → leads armor spearhead
│  │  ├─ Pre-invasion bombardment → damaged coastal defenses
│  │  ├─ Nuclear Escalation done → no Allied superweapon
│  │  ├─ Chrono tech → counter Allied Chronoshifts
│  │  ├─ Grunyev done → no rear uprisings
│  │  ├─ Nothing done → MAXIMUM difficulty
│  │  Dynamic weather: Channel storm clears mid-invasion.
│  │  Briefing references every choice.
│  │
│  └─ [IC] "Supremacy: Enhanced" ★ VERY HARD — ALTERNATIVE
│     Dynamic weather + Volkov commando insertion on White Cliffs
│     + embedded task force + co-op option (Phase 6b add-on).
│
EPILOGUE ────────────────────────────────────────────────────────────
│
└─ [IC] "Red Dawn" ★ MEDIUM — Stalin's betrayal.
   Player witnesses the power struggle from inside — not just
   cutscenes. Nadia, Gradenko (if alive), Kane's shadow.
   Decisions affect ending variant. Kane reveal lands harder
   if "Stalin's Shadow" was completed (foreshadowing pays off).
   Menu scene: Kremlin at night, red star glowing
```

### Soviet Campaign Summary

| Content Type | Count | Toggleable |
|---|---|---|
| Classic main missions | 14 | Always on |
| IC-original missions | ~7 (prologue, Mole Hunt, Stalin's Shadow, Overseer: Strike, Operation Tempest, enhanced alternatives, epilogue) | Per-mission |
| Counterstrike missions | 6 (Volkov & Chitzkoi, Legacy of Tesla, Paradox Equation, Mousetrap, Testing Grounds, Nuclear Escalation) | Per-chain |
| Aftermath missions | 9 (France 1-4, Spain 1-4, Grunyev Revolution) | Per-chain |
| **Total missions available** | **~36** | — |
| **Minimum path (classic only)** | **14** | — |

---

## Cross-Campaign Continuity (Flavor Text Only)

If the player plays both campaigns, certain thematic echoes emerge. These are **briefing flavor text only** — not gated by any cross-campaign state. Neither campaign reads data from the other. No global/legacy persistence layer is needed. Each campaign's `CampaignState` (D021) is fully independent.

| Allied Event | Soviet Briefing Echo |
|---|---|
| Tanya infiltrates Soviet facility (IC "Behind Enemy Lines") | M7 briefing mentions "an enemy commando was detected" — always present, not conditional |
| Allied spy network established | M3 briefing mentions "increased Allied intelligence activity" — always present |
| Kosygin extracted by Allies (M9) | Soviet M9 briefing references "a high-ranking defection" — always present |
| Stavros rescued in Greece (CS) | CS "Mousetrap" briefing mentions "the Greek officer" — always present |

The echoes are authored as standard briefing text that both campaigns include regardless of play order. A player who plays both campaigns recognizes the connections; a player who plays only one experiences them as normal briefing context. No data flows between campaigns — each stands alone.

---

## IC Feature Showcase Summary

| IC Feature | Allied Showcase | Soviet Showcase |
|---|---|---|
| **Hero progression** | Tanya skill tree across 14+ missions | Volkov progression across 14+ missions |
| **Embedded task force** | Evidence: Enhanced (M10B alt), Moscow Enhanced | Core of the Matter (Volkov vs Tanya), Supremacy Enhanced |
| **Air campaign** | Operation Skyfall | Operation Tempest |
| **Spy infiltration** | Behind Enemy Lines, Spy Network | Mole Hunt, Stalin's Shadow |
| **Dynamic weather** | Chronosphere defense storm, Moscow blizzard | Channel storm during England invasion |
| **Asymmetric co-op** | Joint Operations (M11, Phase 6b add-on), Moscow Enhanced (Phase 6b add-on) | Chronosphere Enhanced (Phase 6b add-on), Supremacy Enhanced (Phase 6b add-on) |
| **Branching decisions** | 4+ decision points | 3+ decision points |
| **Spectrum outcomes** | Bridge destruction, intel gathering | Convoy escort, spy elimination |
| **Side mission rewards** | Compound bonuses affecting M14 | Compound bonuses affecting M14 |
| **Campaign menu scenes** | 4 acts with evolving backgrounds | 4 acts with evolving backgrounds |
| **Roster carryover** | Squad persistence, hero survival matters | Volkov + captured units persist |
| **Failure as consequence** | Tanya capture/rescue branch | Convoy loss affects Act 2 forces |
| **Time machine (D078, experimental add-on)** | "Einstein's Confession" — narrative reveal in the baseline plan; only becomes a gameplay unlock if D078 graduates from Draft. If enabled: cautious, knowledge-only carryover | "Temporal Experiment" — narrative reveal in the baseline plan; only becomes a gameplay unlock if D078 graduates from Draft. If enabled: same Layer 2 knowledge-only rules, but harsher Soviet-flavored intel extraction |
| **Capture escalation** | Tanya captured → rescue can happen now, later, or never; the longer she is held, the more Chronosphere / spy-network intel leaks | Volkov captured → each delay gives the Allies more cyborg-tech insight; late or failed rescue escalates Act 3 penalties |
