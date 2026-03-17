# Generated SpecOps Prototype

This is a proof-of-concept template showing how a generated SpecOps mission fits into IC's campaign system end to end.

Research companion: `research/generated-specops-missions-study.md`

Goal:

- prove the campaign node, generation profile, and briefing UX all align
- keep the example deterministic and modest
- demonstrate a single mission family clearly

Chosen vertical slice:

- family: `intel_raid`
- theater: `greece`
- player role: Tanya-led SpecOps
- commander support: optional, bounded

## 1. Campaign Node

This is the authored campaign-facing operation card. It defines the strategic meaning, not the final map layout.

```yaml
missions:
  allied_specops_greece_07:
    role: specops
    source: generated
    mission_family: intel_raid
    theater: greece
    generated_profile: allied_intel_raid_t1
    badge: "IC Generated"
    urgency: timed
    critical_failure: false

    briefing_risk:
      success_reward: "M8 gains AA blind spot and one patrol route reveal"
      failure_consequence: "Operative may be captured; M8 loses the blind spot"
      skip_consequence: "Soviet radar net remains active"
      time_window: "Available until the current Act 1 decision window closes"

    state_effects_on_success:
      set_flag:
        greek_radar_codes_stolen: true
    state_effects_on_failure:
      set_flag:
        greek_radar_raid_failed: true
    state_effects_if_skipped:
      set_flag:
        greek_radar_window_missed: true

    generation_fallback:
      mode: authored_backup
      mission: allied_specops_greece_07_backup

    consumed_by:
      - allied_08
```

## 2. Generation Profile

This chooses the allowed authored-piece pools and validation contract for the mission family.

```yaml
generated_specops_profiles:
  allied_intel_raid_t1:
    family: intel_raid
    allowed_heroes:
      - tanya
    allowed_theaters:
      - greece
      - poland

    site_kits:
      - id: soviet_coastal_radar_compound
        weight: 3
      - id: soviet_signal_outpost_hillside
        weight: 2

    objective_modules:
      - id: steal_radar_codes
        weight: 4
      - id: photograph_aa_layout
        weight: 2

    ingress_modules:
      - id: sewer_infiltration
        weight: 3
      - id: cliff_rope_entry
        weight: 2

    exfil_modules:
      - id: fishing_boat_extract
        weight: 2
      - id: cliff_signal_extract
        weight: 3

    security_profiles:
      - id: radar_tier_2
        weight: 3
      - id: radar_tier_3
        weight: 1

    complication_modules:
      - id: data_purge_timer
        weight: 2
      - id: weather_front_low_visibility
        weight: 1

    commander_support:
      enabled: true
      support_zone_pool:
        - id: offshore_fire_support
      package:
        allowed_structures:
          - field_hq
          - power_node
          - repair_bay
          - sensor_post
        allowed_support_powers:
          - recon_sweep
          - offmap_artillery
          - extraction_beacon
        allowed_unit_classes:
          - engineer
          - medic
          - light_apc
        max_structures: 4
        max_combat_vehicles: 4
        economy_enabled: false
        heavy_factory_enabled: false
        superweapon_enabled: false

    validation:
      require_stealth_route: true
      require_loud_route: true
      require_exfil_after_alarm: true
      max_alarm_to_exfil_distance_tiles: 160
      target_estimated_duration_minutes: 15
      max_estimated_duration_minutes: 20
```

## 3. Site Kit Template

This is the reusable authored tactical site.

```yaml
site_kits:
  soviet_coastal_radar_compound:
    theater: greece
    biome: mediterranean_coast
    footprint:
      width_cells: 8
      height_cells: 8

    sockets:
      objective:
        - id: command_bunker
        - id: radar_control_room
      ingress:
        - id: south_sewer
        - id: west_cliff
      exfil:
        - id: north_boat_cove
        - id: east_signal_ridge
      security:
        - id: yard_patrol_ring
        - id: aa_watchtower_slot
        - id: reserve_barracks_slot
      commander_support:
        - id: offshore_support_anchor

    route_promises:
      stealth_lane:
        from: south_sewer
        to: radar_control_room
      loud_lane:
        from: west_cliff
        to: command_bunker
```

## 4. Example Resolved Mission Instance

This is the deterministic materialized output once the operation appears on the campaign map. This object is what gets persisted in `CampaignState`.

```yaml
generated_operation_instances:
  allied_specops_greece_07:
    source_profile: allied_intel_raid_t1
    family: intel_raid
    seed: 1048849597263513533

    resolved_modules:
      site_kit: soviet_coastal_radar_compound
      objective_module: steal_radar_codes
      ingress_module: sewer_infiltration
      exfil_module: cliff_signal_extract
      security_profile: radar_tier_2
      complication_module: data_purge_timer
      commander_support_zone: offshore_fire_support

    validation_report:
      objective_reachable: true
      stealth_route_exists: true
      loud_route_exists: true
      exfil_after_alarm_exists: true
      commander_support_bounded: true
      estimated_duration_minutes: 14

    derived_briefing:
      reward_preview: "M8 gains one AA blind spot and one patrol route reveal"
      failure_consequence: "If Tanya is captured, M5 rescue branch opens and M8 loses the blind spot"
      skip_consequence: "Soviet radar net remains active"
      time_window: "Must be chosen during the current Act 1 decision window"
```

## 5. Briefing UI Proof

This is the player-facing form the above data should take.

```text
OPERATION: COASTAL SHADOW
Tags: [SPECOPS] [TIMED] [RECOVERABLE] [GENERATED]
Generated from: Greek coastal radar compound

On Success
- M8 west AA arc begins disabled
- One Soviet patrol route is pre-marked at mission start

On Failure
- Tanya may be captured
- No AA blind spot in M8

If Skipped
- Soviet radar net remains active

Time Window
- Expires when the current Act 1 operation window closes
```

## 6. Why this validates the whole design

This prototype proves the key joins in the system:

1. **Campaign graph integration**
   - The operation is a normal mission node with normal reward/risk state effects
2. **Deterministic generation**
   - The mission stores a concrete seed and resolved module list
3. **World-screen readability**
   - The operation card still shows exact reward / failure / skip state
4. **Mission-family reuse**
   - Another `intel_raid` node can reuse the same profile with a different site kit and seed
5. **Commander-support compatibility**
   - The optional support lane is bounded and does not become a main-operation economy

## 7. Prototype Expansion Path

If this first slice works, add content in this order:

1. second `intel_raid` site kit in Greece
2. same family in Poland
3. second family: `rescue`
4. second commander-support profile
5. per-theater art swaps and complication pools

Do not expand to five families before validating the first one in editor, save/load, and briefing UX.
