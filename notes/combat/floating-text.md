# Floating combat text audit

Date: 2026-07-26
Scope: Game build 103 client, current Dark Spin combat publication, and observational Fang diagnostics
Implementation status: audit only; no server, client, asset, or database behavior was changed

## Finding

Build 103 does not derive floating combat text from the authoritative hit-point update. It derives it from application packet `0xBA CombatEvent`.

The normal damage presentation chain is:

```text
0xBA CombatEvent
  -> sub_539F60 (decode reflected type 0x5F1CC727)
  -> sub_4E2CA0 (relationship/preference/flag dispatch)
  -> local ServerEvent record with a combattext_*.ServerEventDef asset
  -> sub_506D90 (resolve the target object and create the world-space presentation)
```

`0x97 CombatantDataUpdate` only applies current hit points or mana to the combatant component through `sub_539AA0`. A hit can therefore be fully authoritative and visible in the resource bar while producing no number.

Current server behavior is mixed:

- The shared ordinary and lethal damage encoders generally send the correct native `0xBA` shape before `0x97`. Those paths are not explained by a missing damage opcode.
- Tree of Life, Enrage healing, generic hero-resource projection, and the developer heal path send resource state without a healing `0xBA`, so they cannot produce native healing numbers.
- A remote co-op observer of a hero damaged by an NPC receives the projected resource update but not the source, amount, or critical combat event.
- Immune results are discarded before publication, Ghost Form dodge is represented only by an effect, and geometry/projectile misses generally have only an authored miss effect. None of those paths sends the corresponding zero-delta combat feedback flag.
- Projected companion damage omits critical state, so a peer can receive a normal number for a critical hit.
- A valid `0xBA` still produces no text if the controlled-player binding is absent, the source/target objects cannot be resolved, neither object is related to the local player, the relevant preference is disabled, or the target is torn down before local dispatch.

Consequently, an observation that *all* ordinary damage lacks text should first be reduced to one received `0xBA` and its client-side relationship decision. The static client and current encoder evidence says a received, resolvable native `BA 9B` damage event is sufficient; the recent trace does not contain application-message payloads and cannot prove that such an event reached the client.

## Evidence and confidence

Confidence labels used below:

- **Exact**: packet reflection, executable control flow, or current encoder establishes the field or behavior directly.
- **High**: multiple independent build-103 artifacts agree, with only naming or presentation details outside the code.
- **Medium**: branch purpose is clear but its content-authored visual treatment or producer is not fully recovered.
- **Low**: a field or flag is accepted but no build-103 semantic producer was identified.

Primary evidence:

- `notes/client/packet.md:572-579` documents the reflected `CombatEvent` body and the packet catalog maps GMS message 63 to wire `0xBA`.
- The requested `notes/stats.md` has moved to `notes/combat/stats.md`; its critical, damage, and healing formulas describe authoritative calculation, not presentation.
- `notes/design/architecture/raknet-gameplay-exchange.md:594-647` and `notes/tutorial/overview.md:1811-1844,4355-4388` independently reconstruct the native send and receive paths.
- `bin/game/GameBin/Game.c:380762-380774` is the `0xBA` decoder; `:316213-316957` is the central presentation handler; `:345416-345786` is the local `ServerEvent` dispatcher.
- `bin/game/GameBin/Game.c:1399978-1400020` is the native damage/healing producer and `:378703-378810` synthesizes status feedback from modifier creation.
- Existing `bin/game/logs/ida-gms-name-range.log` and `bin/game/logs/ida-resolved-inline-fields.log` corroborate the message and global identities.
- Read-only package inspection produced `bin/game/logs/floating-combat-text-assetdata.bin`. Its strings corroborate the combat-text asset names without modifying the shipped package.
- The only recent protocol trace, `bin/server/darkspin/logs/traces/client.jsonl`, records connection/client-state diagnostics but no decoded `0xBA`, `0x97`, or combat payload. This is insufficient capture coverage, not evidence that either packet was absent.

## Client-accepted messages and their roles

| Wire opcode | Build-103 callback | Role in combat presentation | Triggers floating text |
|---|---|---|---|
| `0xBA CombatEvent` | `sub_539F60` -> `sub_4E2CA0` | Damage, critical, healing, absorption, avoidance, immunity, resurrection, and several status notifications | **Yes** |
| `0x97 CombatantDataUpdate` | `sub_539AA0` | Sparse authoritative current HP/mana update on the combatant component | No |
| `0x9B ServerEvent` | `sub_539E90` | Plays authored effects and can directly display content-authored numeric pickup text | Not for ordinary combat numbers; only when an explicit text-bearing event is authored |
| `0xA2 ModifierCreated` | modifier callback including `sub_537780` | Applies a modifier, then locally synthesizes slowed/stunned/taunted/suppressed/rooted/terrified feedback for selected modifier classes | Indirectly |
| `0xA3 ModifierUpdated` | modifier update callback | Maintains modifier state | No newly dispatched status text found |
| `0xA4 ModifierDeleted` | modifier delete callback | Removes modifier state | No newly dispatched status text found |
| `0xA5 SetAnimationState` | animation callback | Hit reaction, death, and other animation state | No |
| `0x96 AttributeDataUpdate` | attribute callback | Attribute/component state | No |
| `0x8E ObjectDelete` | object deletion callback | Tears down one or more object identities | No; can prevent a pending number from anchoring |
| `0xA1 LabsPlayerUpdate` | player update callback | Among other fields, binds the locally controlled object | No; its binding is a gate for `0xBA` |
| `0x9D PlayerDamage` | no attached constructor/callback in the build-103 catalog | Named but unattached legacy/unused candidate | No evidence that build 103 accepts or uses it |

The implementation must use `0xBA`; adding `0x9D PlayerDamage` or replacing combat events with `0x9B ServerEvent` would be a protocol guess and would bypass the native shared dispatcher.

## Exact `0xBA CombatEvent` layout

The application payload is mask-reflected and contains one event, not a repeated list. Fields are serialized in ascending bit order:

| Mask bit | Field | Encoding | Meaning | Confidence |
|---:|---|---|---|---|
| `0x01` | `flags` | little-endian `u16` | Damage/heal/critical/result/status classification | Exact |
| `0x02` | `delta_health` | little-endian `f32` | Positive damage, negative requested healing, zero for word-only feedback | Exact |
| `0x04` | `absorbed_amount` | little-endian `f32` | Absorbed amount; can be presented when health delta is zero | Exact |
| `0x08` | `target_object_id` | little-endian `u32` | Live target object used for relationship and world-space anchoring | Exact |
| `0x10` | `source_object_id` | little-endian `u32` | Live source object used for relationship and combat-log classification | Exact |
| `0x20` | `ability_id` | little-endian `u32` | Accepted identity field; the recovered generic native damage/heal producer leaves it zero | Exact layout; High native-zero behavior |
| `0x40` | `damage_direction` | three little-endian `f32` | Optional direction used by auxiliary hit response | Exact layout; Medium presentation use |
| `0x80` | `integer_hp_change` | little-endian `i32` | Integer displayed for damage/healing | Exact |

All-fields form:

```text
BA FF
  flags:u16
  delta_health:f32
  absorbed_amount:f32
  target_object_id:u32
  source_object_id:u32
  ability_id:u32
  damage_direction_x:f32 damage_direction_y:f32 damage_direction_z:f32
  integer_hp_change:i32
```

The native ordinary shape uses mask `0x9B`, fields 0, 1, 3, 4, and 7:

```text
BA 9B
  flags:u16
  delta_health:f32
  target_object_id:u32
  source_object_id:u32
  integer_hp_change:i32
```

This is 20 bytes including the application opcode, or 19 bytes after it. `server/raknet/application.go:1334-1352` encodes this shape as `DamageCombatEventMessage`. `CombatEventMessage` at `:1310-1332` encodes the full `0xFF` form.

There is no repeated or dynamic body after the reflected fields, no terminator, and no combat-event timestamp.

### Adjacent packet layouts

These packets can establish state, effects, timing, or object lifetime around the text, but do not replace `0xBA`.

HP/mana state is an ordinary bit mask:

```text
97
  object_id:u32
  mask:u8
  if mask & 0x01: hit_points:f32
  if mask & 0x02: mana_points:f32
```

An HP-only update is 10 bytes including the opcode. There is no source, amount, damage type, critical flag, team, or timestamp, so none can be recovered from `0x97`.

`0x9B ServerEvent` is a terminated sparse reflection rather than the fixed bit-mask body used by `0xBA`. Each selected field is encoded by field index and field body, and `0xFF` ends the event:

```text
9B
  repeated selected fields:
     0 simple_swarm_effect:u32
     1 slot:u8
     2 is_removal:bool
     3 is_hard_stop:bool
     4 is_force_attached:bool
     5 is_critical:bool
     6 asset:u32
     7 primary_object_id:u32
     8 secondary_object_id:u32
     9 attacker_object_id:u32
    10 position:vec3
    11 facing:vec3
    12 orientation:quat
    13 target_point:vec3
    14 text:i32
    15 client_event_id:u32
    16 player_exclusion_mask:u32
    17..25 loot fields
  FF
```

Ordinary combat text does not arrive in this envelope. `sub_4E2CA0` creates a local equivalent containing the selected combat-text asset, primary target, secondary source, and integer text, then passes it to `sub_506D90`. Explicit pickup messages are the server-authored exception.

Modifier creation is fixed width:

```text
A2
  target_object_id:u32
  modifier_guid:u32
  instance_id:u32
  duration_ms:u32
  overdrive:u32
  stack_count:u32
  start_ms:u64
  source_object_id:u32
  is_bound:bool
```

The body is 37 bytes after the opcode. `0xA3` updates target, instance, signed start time, stack count, and bound state; `0xA4` deletes a target/instance pair. Only the recovered `0xA2` apply callback synthesizes the status words described below.

Animation state is also fixed width:

```text
A5
  object_id:u32
  animation_state:u32
  timestamp_ms:u64
  is_overlay:bool
  scale:f32
  source_echo_gate:u32
```

Its body is 25 bytes after the opcode. The timestamp controls animation, not combat-text age.

`0x8E ObjectDelete` is the relevant repeated body: every remaining four bytes is another `u32 object_id`, so the payload length must be divisible by four. It has no count or terminator.

The relationship prerequisites arrive elsewhere:

- Object creation's reflected tail fields include `team:u8` at field 0, `is_player_controlled:bool` at field 1, `player_index:u8` at field 3, and `owner_object_id:u32` at field 18.
- `0xA1 LabsPlayerUpdate` starts with `player_slot:u8` and `outer_mask:u16`; outer bit `0x1000` selects the terminated player reflection. Player fields 4, 5, and 9 are player index, team, and controlled object.
- `0xA7 PlayerCharacterDeploy` is `[player_index:u8][creature_index:u32][object_id:u32]`.

Those messages explain how a client knows local ownership and teams. They do not add fields to `0xBA`.

### Fields that are deliberately not on the packet

- **Team** is not transmitted. The client resolves source and target objects and compares their existing team/player data to the locally controlled player.
- **Player ownership** is not transmitted. Object creation/player state supplies player-controlled, player index, owner object, and controlled-object binding.
- **Damage type** is not an explicit field. The optional ability identity is not a damage-type field and the generic native producer leaves it zero. Combat-log noun/type data is recovered from resolved objects.
- **Presentation color, font, scale, motion, and anchor offset** are not transmitted. The selected `combattext_*.ServerEventDef` owns them.
- **Timestamp** is not transmitted. Delivery/scheduling time is the event time; only the healing target effect uses a local client clock.

## Flags and feedback subtypes

| Flag | Client behavior | Producer/evidence | Confidence |
|---:|---|---|---|
| `0x0001` | Damage classification | Native producer sets it for positive delta | Exact |
| `0x0002` | Healing classification | Native producer sets it for negative delta | Exact |
| `0x0004` | Killing blow/death log branch | Native producer sets it when target HP is zero | Exact |
| `0x0008` | Critical; selects critical damage/absorb/heal asset | Native producer critical argument | Exact |
| `0x0010` | Deflect word | Central handler selects `combattext_deflect` | Exact dispatch |
| `0x0020` | Dodge word | Central handler selects `combattext_dodge` | Exact dispatch |
| `0x0040` | Immune word | Central handler selects `combattext_immune` | Exact dispatch |
| `0x0080` | Resist word | Central handler selects `combattext_resist` | Exact dispatch |
| `0x0100` | Resurrected word | `sub_9E5870` and central handler | Exact |
| `0x0200` | Suppresses selected auxiliary magnitude/hit-response effects | Native producer accepts it; numeric text is not suppressed | Exact behavior; Medium semantic name |
| `0x0400` | Accepted/pass-through bit | A recovered native wrapper supplies it, but no direct central-handler presentation branch was found | Low |
| `0x0800` | Slowed word | Modifier-created callback synthesizes a local event | Exact |
| `0x1000` | Stunned word | Modifier-created callback | Exact |
| `0x4000` | Taunted word | Modifier-created callback | Exact |
| `0x8000` | Suppressed word | Modifier-created callback | Exact |
| `0x10000` | Rooted word | Local in-memory modifier event only; cannot fit the wire `u16` | Exact local-only |
| `0x20000` | Terrified word | Local in-memory modifier event only; cannot fit the wire `u16` | Exact local-only |

The in-memory client structure can hold flags wider than the wire field. That explains the two local modifier flags; it does not make the server wire field wider than `u16`.

There is no single recovered `MISS` flag or `combattext_miss` asset. Build 103 distinguishes:

- dodge, deflect, resist, and immune outcomes through their explicit flags;
- geometry/projectile misses through an authored `missEvent` effect, which normally has no numeric or word combat text;
- status feedback synthesized after `0xA2 ModifierCreated`.

## What actually triggers text

`sub_4E2CA0` requires a locally controlled player. It resolves the target and source object IDs before choosing a branch.

### Damage and absorption

For positive `delta_health`, the client displays `integer_hp_change` exactly as supplied. For zero/non-positive delta with positive `absorbed_amount`, it converts the absorbed float to an integer using C truncation toward zero.

The event is accepted for display only when one of these local relationships is established and its preference is enabled:

| Relationship | Preference |
|---|---|
| Controlled object is the source | `ShowDamageDoneByMe` |
| Controlled object is the target | `ShowDamageDoneToMe` |
| Source is on the controlled player's team | `ShowDamageDoneByAllies` |
| Target is on the controlled player's team | `ShowDamageDoneToAllies` |

The source-controlled test is first. A diagnostic packet that incorrectly uses the same object as source and target is therefore classified as damage done by the player, not incoming/environmental damage.

An unrelated remote source attacking an unrelated remote target has no matching preference branch and produces no local text. A companion counts through its team/object relationship; it does not need a companion-specific opcode. A hostile NPC attacking the controlled hero matches the target-controlled branch. A hostile NPC attacking a co-op ally matches the target-team branch, provided both objects and team data are present.

### Healing

For negative `delta_health`, the client displays `-integer_hp_change`. The supported relationships are:

| Relationship | Preference |
|---|---|
| Controlled object is the target | `ShowHealingDoneToMe` |
| Target is on the controlled player's team | `ShowHealingDoneToAllies` |
| Target is on the opposing team | `ShowHealingDoneToEnemies` |

No separate `ShowHealingDoneByMe` option exists in the recovered build-103 option array at `Game.c:111650-111670`. The target relationship controls display.

Healing also starts `life_healing_target_effect.ServerEventDef` at most once per target per 1,000 ms using the client's local clock and a per-target map. That throttle applies to the target visual effect, not to the numeric events. No numeric aggregation or coalescing was found: each accepted combat event queues one number.

### Zero-delta result words

When there is no damage/healing number, the central handler can dispatch the deflect, dodge, immune, resist, resurrected, and status assets listed above. The recovered branch is gated by the local damage-done preference/context; the server should preserve the real source and target so the same relationship logic can operate.

### Combat logs and hit reactions

The handler can also dispatch local combat-log events such as `LABS_local_PC_DAMAGE`, `LABS_local_PC_DAMAGE_ENVIRONMENT`, and `LABS_PC_KILLING_BLOW`. These are consequences of the accepted combat event, not the number renderer.

Magnitude thresholds, damage direction, and flag `0x0200` can select auxiliary hit-response effects. `0xA5 SetAnimationState` can play a hit/death animation. A `0x9B ServerEvent` can create an impact effect. None substitutes for the numeric `0xBA`.

## Formatting recovered from build 103

| Concern | Recovered rule | Confidence |
|---|---|---|
| Damage amount | `integer_hp_change` is passed through as the displayed integer | Exact |
| Healing amount/sign | Negative event integer is negated before display | Exact |
| Absorbed amount | Positive `absorbed_amount` float is truncated to an integer | Exact |
| Fractional rounding | No client rounding of `delta_health`; the server/native producer supplies the integer | Exact |
| Critical emphasis | Flag `0x0008` selects a distinct critical asset | Exact |
| Numeric aggregation | None found; one number per accepted event | High |
| Throttling | Only the healing target effect is throttled, at 1 second per target | Exact |
| World anchor | Primary object is the target; source is secondary. `sub_506D90` resolves the target visual/root transform and content attachment rules | Exact |
| Colors/font/scale/motion | Owned by content assets, not wire fields | Exact ownership |
| Exact RGBA/font metrics | Not recovered from the selected executable/package inventory | Unresolved; unnecessary for server implementation |

Exact executable asset selection:

| Outcome | Non-critical asset | Critical asset |
|---|---|---|
| Damage, first relationship style | `combattext_damage.ServerEventDef` | `combattext_damage_critical.ServerEventDef` |
| Damage, alternate relationship style | `combattext_enemy_damage.ServerEventDef` | `combattext_enemy_damage_critical.ServerEventDef` |
| Absorb, first relationship style | `combattext_absorb.ServerEventDef` | `combattext_absorb_critical.ServerEventDef` |
| Absorb, alternate relationship style | `combattext_enemy_absorb.ServerEventDef` | `combattext_enemy_absorb_critical.ServerEventDef` |
| Healing | `combattext_heal.ServerEventDef` | `combattext_critical_heal.ServerEventDef` |

The two damage/absorb style families are selected by the resolved local relationship. Their exact authored colors should not be inferred from the names. The executable also names `combattext_dodge`, `combattext_deflect`, `combattext_resist`, `combattext_immune`, `combattext_resurrected`, `combattext_slowed`, `combattext_stunned`, `combattext_taunted`, `combattext_suppressed`, `combattext_rooted`, and `combattext_terrified`.

## Required ordering and lifetime

The numeric event contains its own displayed integer and does not need the subsequent `0x97` to calculate text. Native/current ordering should nevertheless be preserved:

1. The target object, source object, team data, and controlled-object binding must already exist on the receiving client.
2. Ability launch/projectile/impact events may establish authored visual timing, but the `0xBA` must be emitted at the actual hit/commit time.
3. Emit `0xBA CombatEvent`.
4. Emit `0x97 CombatantDataUpdate` with the committed current HP.
5. Emit modifier changes and hit/death state as semantically appropriate.
6. On lethal damage, keep `0xBA` before zero HP, death state/animation, and `0x8E ObjectDelete`. Do not tear down the target before `sub_506D90` can resolve its world anchor.

For healing, emit the negative `0xBA` at the committed heal pulse and the new `0x97` immediately after it. A full-state restoration that is not a semantic heal need not invent a number. Health-orb pickup is a special case: the current path already sends an explicit numeric `0x9B ServerEvent`, then resource state, then delete. Adding a generic heal `0xBA` there would duplicate presentation without native evidence.

Packets should remain on the same reliable ordered application stream. There is no timestamp in `0xBA` with which the client could repair reordering. Modifier timestamps, animation timestamps, and ability scheduling timestamps belong to their own messages and should not be copied into the combat event.

## Current server publication audit

### Wire adapters

- `server/raknet/application.go:1310-1352` has both the full reflected event and exact native `0x9B` damage shape. Field width and mask are correct.
- No healing-specific recipe or feedback-only recipe exists. Callers therefore have no shared way to encode native negative healing or zero-delta immune/dodge/deflect/resist outcomes.
- The all-fields encoder exposes ability, absorbed amount, and direction, but current gameplay callers do not publish those fields.
- The absence of team, timestamp, and damage-type fields is correct. Those must not be fabricated.

### Damage paths

| Path | Current publication | Audit |
|---|---|---|
| Simulator damage adapter | `server/sim/raknet103/encoder.go:294-320` emits `0xBA`, then its HP intent emits `0x97` | Correct native shape and critical/killing flags |
| Melee, burst, poison, and shared simulator runs | Representative order at `server/sim/melee.go:122-140`, `burst.go:239-264`, `poison.go:270-290` | Damage event and state exist; authored impact effects vary around them |
| Shared NPC target damage | `server/zone/npc/raknet103/damage.go:27-61` and `server/gameplay/handler.go:1447-1480` emit event then HP | Correct ordinary publication |
| NPC attack against the action owner's hero | `server/zone/npc/raknet103/hit.go:27-48` and `action.go:145-160` emit event then HP | Correct local publication |
| Lethal target damage | `server/zone/death/raknet103/death.go:278-319` emits killing event, HP, then death presentation; deletion is later | Correct anchor-preserving order |
| Remote co-op hero hit by NPC | `server/zone/zone.go:721-759` publishes `HeroResource`; `server/gameplay/handler.go:1527-1552` emits only HP/Labs resources | **Missing combat event, source, amount, and critical state for peers** |
| Remote co-op companion hit by NPC | `server/zone/zone.go:766-793` publishes `CompanionDamage`; handler `:1553-1584` emits event and HP | Number can render, but projection omits critical state; defeat is followed by immediate delete |
| Shared area/basic duplicate suppression | `server/gameplay/ability_basic.go:1390-1399` and `server/zone/companion/raknet103/presentation.go:130-138` remove simulator `0xBA`/`0x97`, then shared publication replaces them | Intentional, but packet-ID filtering is brittle; every outcome must be replaced exactly once |
| Immune NPC result | Shared damage publication projects the rejected result to every member; the build-103 adapter emits feedback-only `0xBA` flag `0x0040` | **Correct: native `immune` text, no HP packet, and no damage-objective progress** |
| Ghost Form dodge | `server/zone/ability/raknet103/ghost_form.go` publishes state/effect only | **Missing zero-delta dodge `0xBA`** |
| Projectile/geometry miss | Burst and other authored paths send `missEvent` effects | Correct for a physical miss; do not display “MISS” unless the semantic outcome is one of the recovered feedback flags |
| Developer damage | `server/developer/raknet103/projection.go:29-43` uses target as both source and target | Diagnostic-only classification is wrong: the source-first client gate treats it as damage done by me |

The server commonly derives `integer_hp_change` with `-int32(damage)`. Go and the client both truncate toward zero, but native behavior is best represented by the committed integer HP difference, not an independently projected/requested float. A shared publisher should derive the integer from old and new authoritative HP so fractional hits, caps, immunity, and overkill cannot disagree with the resource update.

### Healing paths

| Path | Current publication | Audit |
|---|---|---|
| Tree of Life pulses | `server/zone/ability/raknet103/tree_of_life.go:256-273` sends heal effects and `0x97` per healed target | **Missing negative `0xBA` per non-zero committed heal** |
| Enrage/self healing | `server/zone/ability/raknet103/enrage.go:115-148` commits healing and sends `0x97` | **Missing negative `0xBA`** |
| Generic hero resource projection | `server/zone/projection/session.go:199-218`, `server/zone/zone.go:853`, and handler `:1527-1552` carry current HP/mana only | **Cannot represent who healed, actual amount, or critical healing to co-op peers** |
| Generic hero/ability state adapters | `server/zone/hero/raknet103/resource.go:39-49`, `server/zone/ability/raknet103/state.go:81`, and targeted AOE resource encoding at `targeted_aoe.go:242` are state-only | Correct as low-level state adapters, but semantic heal callers need a paired event |
| Developer heal | `server/gameplay/status.go:806-893` mutates authority and publishes only resource updates | **No healing number** |
| Health orb | `server/zone/interact/raknet103/orb.go` uses an explicit numeric `0x9B ServerEvent`, resource delta, then object delete | Keep as an authored pickup exception; a second `0xBA` would duplicate text |

### Visibility and ownership

`server/zone/projection/session.go` fans projected NPC events to zone subscribers rather than filtering by camera distance. The action owner drains its direct/local response while peers drain projected events. The client then applies its own relationship preferences.

This division is sound only if every recipient receives:

- the source and target objects before the combat event;
- correct team and player-control metadata;
- the same semantic `0xBA` outcome, followed by the state update;
- no early target deletion.

The current `HeroResource` projection violates the third requirement. The current companion projection meets it for ordinary damage but loses the critical flag. No new “local player,” “companion,” or “remote player” wire flag is required; those distinctions are client-side object relationships.

## Fang

Fang does not currently alter the relevant presentation callback.

The diagnostic-only hook in `app/fang/fang.c:1349-1423` logs received application messages. For a `0xBA` mask-`0x9B` payload it reads target, source, integer change, controlled handle/object, player index, and a coarse relationship, then calls `original_message_construct` unchanged. The hook is installed only under `FANG_DIAGNOSTICS` at `:3461-3465`.

No hook was found that patches `sub_4E2CA0`, `sub_506D90`, the combat-text preferences, asset selection, or packaged UI defaults. The effect-preview diagnostic around `0x9B ServerEvent` is unrelated to normal `0xBA` combat text.

If runtime evidence is needed, proposed Fang work must remain observational:

- log the fully decoded `0xBA` fields and receive order relative to `0x97`/`0x8E`;
- log controlled-object, source/target resolution, teams, and the selected preference branch;
- log the selected combat-text asset and whether `sub_506D90` accepted or rejected the local event;
- never force a preference, substitute an asset, keep an object alive, suppress native presentation, or patch the callback result.

## Smallest shared server-side implementation plan

1. Add one build-103 health-change presentation helper beside the RakNet combat adapters. Give it source object, target object, requested amount, old HP, new HP, semantic kind, critical state, optional absorbed amount, and optional feedback result. It should return the native `0xBA` followed by `0x97` when a resource changed, or feedback-only `0xBA` when it did not.
2. Preserve native signs and the native requested/applied distinction: positive `delta_health` plus negative integer change for damage; negative requested `delta_health` plus the negative committed old/new integer difference for healing. Suppress zero-applied healing, matching the recovered native path.
3. Use the helper at the shared combat/resource commit boundaries, not in individual ability definitions. Migrate existing damage publishers without changing their ordering, then route Tree of Life, Enrage, and other semantic healing through the same boundary.
4. Extend projection semantics so co-op recipients receive the same source/target/amount/critical event as the action owner. A generic combat-presentation projection or focused additions to existing projections are both viable; do not attempt to reconstruct the event from `HeroResource`.
5. Publish immune and Ghost Form dodge results as zero-delta feedback events. Keep physical projectile misses effect-only unless client/content evidence classifies them as dodge, deflect, resist, or immune.
6. Preserve the health-orb authored numeric event as an explicit exception. Ensure packet filtering suppresses an original event only when the shared replacement is guaranteed, and assert exactly one combat event per semantic outcome.
7. Keep lethal targets alive through event dispatch and retain event-before-resource-before-death/delete order.

Focused automated coverage should live at the reusable owner:

- byte fixtures for `BA 9B` damage, critical damage, healing, critical healing, and zero-delta feedback;
- table-driven health-change sign/integer/cap/overkill/zero-heal invariants;
- one representative shared damage, heal, immune, lethal, and co-op projection path;
- duplicate-count and event-before-resource assertions;
- no dedicated unit suite for every named ability.

## Human verification matrix

Use a build-103 client with all recovered damage/healing display preferences enabled. Capture application packet order and, if needed, enable only the observational Fang trace described above.

| Scenario | Recipients and expected text | Required packet/ordering checks |
|---|---|---|
| Blitz ordinary hit on hostile NPC | Blitz player sees ordinary damage; co-op ally sees it only if `ShowDamageDoneByAllies` is enabled | One `0xBA` with Blitz source/NPC target, then NPC `0x97` |
| Blitz critical hit | Same recipients; visibly distinct critical asset | `flags & 0x0008`, exact committed integer, no duplicate normal event |
| Sage Tree of Life heals self | Sage sees healing; one number per non-zero pulse | Negative `0xBA`, then self `0x97`; target effect may throttle but numbers must not |
| Sage Tree of Life heals ally/companion | Sage and co-op recipients follow their healing-to-allies preferences | Same source/target/amount projected to every recipient before resource state |
| Wraith Ghost Form avoids a hit | Controlled Wraith receives dodge feedback, no damage number, no HP loss | One zero-delta `0xBA` with `0x0020`; authored effect may accompany it |
| Companion attacks hostile NPC | Owner sees damage through source-team relation; co-op ally follows damage-by-allies preference | Companion source/NPC target remain resolvable; event then HP |
| Hostile NPC attacks controlled hero | Hero player sees incoming style; no “done by me” classification | NPC source/hero target, event then hero HP |
| Hostile NPC attacks co-op ally | Observer sees text only with damage-to-allies preference | Peer projection includes full event, not resource-only |
| Hostile NPC attacks companion | Owner/peers follow target-team preference | Critical flag survives projection; event and HP precede any defeat delete |
| Immune hostile NPC | Attacker sees `immune`, no number, no HP change | Zero-delta `0xBA` with `0x0040`; no `0x97` required |
| Absorbed hit | Absorb amount and critical variant display as appropriate | Positive absorbed field, resolvable identities, optional damage delta handled once |
| Lethal damage | Killing number appears at victim before death presentation | `0xBA` with `0x0004`, zero-HP `0x97`, death state/animation, delayed delete |
| Enrage or another self-heal | Healing number equals actual capped gain | Negative `0xBA`, then `0x97`; zero gain emits neither number nor duplicate effect |
| Health orb | Exactly one authored pickup number | Existing `0x9B` numeric event, resource delta, orb delete; no generic heal `0xBA` |

Acceptance requires matching authoritative HP, displayed integer, critical style, relationship preference, and recipient set in every row. A bar change alone is a failure for semantic damage/healing; an authored pickup or physical miss must not gain an extra generic number.

## Remaining evidence to collect during implementation

The server-side contract is implementation-ready. These observations would close presentation-only uncertainty without changing the plan:

- capture one valid ordinary `BA 9B` and its following `0x97` in `bin/server/darkspin/logs/traces`;
- observe the selected asset for each relationship branch to label the exact authored color families;
- confirm the target remains resolvable through immediate companion defeat on a real client;
- determine the native semantic name/producer for accepted flag `0x0400`.

Do not block the shared server fix on exact RGBA recovery: build 103 selects its own content assets, and the packet neither carries nor needs color data.
