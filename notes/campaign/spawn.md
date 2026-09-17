# Campaign enemy spawn packet contract (build 103)

## Result

Build 103 has no campaign-only enemy-spawn packet. A director-spawned enemy is
materialized with the generic `kGmsObjectCreate` application packet (`0x8c`),
then receives ordinary component snapshots and deltas. Director replication is
a separate `kGmsDirectorState` packet (`0x8b`); it reports director state but
does not contain a noun, marker, position, object ID, budget, or spawn command.

The strongest recoverable per-agent dependency order is:

```text
server-only director chooses noun, marker, count, and object ID
  -> 0x8c ObjectCreate
  -> component baselines needed by the agent (0x97 combatant, 0x96 attributes)
  -> optional 0x8d visibility/transform delta
  -> per-agent target/combat state (0x99) before an action that consumes it
  -> locomotion/turn command (0x91; add 0x95 only for an active walking goal)
  -> ability, effect, animation, and later component deltas
```

If the server attaches `SpawnModifier`, its own strict internal order is:

```text
0xa2 ModifierCreated
  -> Activate: stop locomotion; add scoped Immobilized=1
  -> Tick: 0x9b generic_spawn; 0xa5 horde_beam_in; wait 0.5 s
  -> Tick returns (no deletion implied)
  -> explicit removal: Deactivate effect/reset; scoped cleanup; 0xa4 ModifierDeleted
```

The relative placement of that modifier lifecycle against target injection,
first aggro, and director-state dirties is **not recovered**. In particular,
neither the half-second modifier wait nor the `1.291667s` tutorial first-aggro
wait is evidence for a universal campaign activation delay.

## Evidence boundary

The canonical decompile is `bin/game/GameBin/Game.c`, SHA-256
`1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb`.
The useful native entry points and decompile locations are:

| Boundary | Native evidence |
| --- | --- |
| gameplay dispatch | `sub_53ADC0` at `0x0053adc0`, C lines `381437-381572` |
| `ObjectCreate` receiver | `sub_53A760` at `0x0053a760`, C lines `381130-381187` |
| create-data decode/defaults | `sub_A1DB60` / `sub_A1DB90`, C lines `1446565-1446595` |
| simulator construction | `sub_9D6910`, C lines `1387307-1387394` |
| network-tail decode | `sub_A1E710`, C lines `1447101-1447169` |
| director receiver | `sub_538EE0` -> `sub_A1DF50`, C lines `379979-379989` and `1446709-1446724` |
| modifier create-before-activate | `sub_9E0FB0` -> `sub_A1FB20`, recovered in `notes/tutorial/teleporter-live.md` |
| ability turn publication | `sub_9E0AE0` -> `sub_A15610` -> `sub_A20210`, recovered in `notes/combat/melee-arc.md` |

The checked-in `bin/server/darkspin/logs/traces/client.jsonl` contains socket
lengths and digests, not gameplay payload bytes, so it cannot establish an
opcode sequence. The current Darkspinner runtime trace has the same limitation.
`bin/darkspinner/darkspin/logs/darkspinner.log` does retain response prefixes;
for example its local campaign setup shows `0x8b 00` before locally generated
player creation, and its tutorial setup shows repeated `0x8c -> 0x8d -> 0x91`
groups. Those are captures of **darkspin's current sender**, not a retail
campaign server. They validate that build 103 accepts the local shapes, but
they do not recover retail director policy or flush order.

## `0x8c ObjectCreate`

The receiver first reads a little-endian four-byte object ID. It rejects an ID
that already resolves, decodes `cGameObjectCreateData`, creates the simulator
object, and only then decodes the trailing `sporelabsObject` reflection into
the new object's network component. ObjectCreate must therefore precede every
packet that names the new ID.

The fixed create image has ten fields and an all-fields mask of `0x03ff`:

| Index | Field | Native offset | Wire width | Default |
| ---: | --- | ---: | ---: | --- |
| 0 | noun | `+0x00` | 4 | `0` |
| 1 | position XYZ | `+0x04` | 12 | zero vector |
| 2 | rotation X, degrees | `+0x10` | 4 | `0` |
| 3 | rotation Y, degrees | `+0x14` | 4 | `0` |
| 4 | rotation Z, degrees | `+0x18` | 4 | `0` |
| 5 | asset ID | `+0x20` | 8 | `0` |
| 6 | scale | `+0x28` | 4 | `1.0` |
| 7 | team | `+0x2c` | 1 | `0` |
| 8 | `hasCollision` | `+0x2d` | 1 | `true` |
| 9 | `playerControlled` | `+0x2e` | 1 | `false` |

The fixed prefix is:

```text
8c
<objectId:u32-le>
ff 03
<noun:u32-le>
<position:3xf32-le>
<rotationDegrees:3xf32-le>
<assetId:u64-le>
<scale:f32-le>
<team:u8> <hasCollision:u8> <playerControlled:u8>
```

The following `sporelabsObject` reflection has 23 registered fields:

| Indices | Fields |
| --- | --- |
| 0-3 | team, player-controlled, input stamp, player index |
| 4-7 | linear velocity, angular velocity, position, orientation |
| 8-12 | scale, marker scale, last animation state/time, movement-animation override |
| 13-17 | graphics state, two graphics-state times, visibility, collision |
| 18-22 | owner ID, movement type, disable repulsion, interactable state, source marker ID |

The current receiver-valid enemy codec selects tail field `6` (position), tail
field `7` (identity quaternion), then `0xff`. With the fixed fields above this
is an 81-byte application packet. This is a safe minimal materialization shape,
not proof that retail campaign creation dirtied only those two tail fields.
Campaign spawning may use nonzero rotation, team, owner, movement type, source
marker, or other tail fields when its authoritative state requires them.

`ObjectCreate` does **not** contain combatant, attributes, blackboard,
locomotion, modifier, or director state. Those are separate messages.

## Director-state messages

`0x8b` is a seven-field `cAIDirector` reflection with no object-ID prefix:

| Index | Field | Native offset | Wire value |
| ---: | --- | ---: | --- |
| 0 | `mbBossSpawned` | `+0x0d` | bool |
| 1 | `mbBossHorde` | `+0x0e` | bool |
| 2 | `mbCaptainSpawned` | `+0x0f` | bool |
| 3 | `mbBossComplete` | `+0x10` | bool |
| 4 | `mbHordeSpawned` | `+0x48c` | bool |
| 5 | `mBossId` | `+0x14` | four-byte object ID |
| 6 | `mActiveHordeWaves` | `+0x47c` | registered four-byte field/container handle |

`8b 00` is a receiver-valid idle snapshot. `8b 08 01` selects field 3 and sets
`mbBossComplete=true`. Field 4 is the reflected horde flag and field 6 is the
active-wave storage, but the server-side element format and the correct dirty
combination for wave start/finish have not been recovered. Do not manufacture
an `mbHordeSpawned` packet merely because an enemy is created.

Named Captain marker sets also preserve two distinct authored roles: one
`SpawnPoint_DirectorBoss` listener owns the Captain while sibling
`SpawnPoint_DirectorHorde` listeners register through `HordeSpawner_Register`.
Together with the separate `mbCaptainSpawned` and active-horde fields, this
supports a two-stage encounter instead of simultaneous publication. Darkspin
now admits the registered horde actors as wave one, retains the Captain plan
dormant until that marker-scoped actor set is terminal, and then publishes the
Captain with the reflected `mbCaptainSpawned` field and director-active
presentation to all peers. Final Destructor nouns retain `mbBossSpawned`,
`mbBossHorde`, and their immediate boss encounter. The missing retail server
still prevents proving the exact clear-to-Captain delay.

The client has no direct message-factory call for logical director message 55.
The campaign director sender, including whether a state delta precedes or
follows its agents, is absent from this client-shaped binary. Current campaign
setup emits only idle `8b 00`; current tutorial completion uses field 3. Neither
is a captured campaign wave-start contract.

## Component baselines and AI activation

An agent normally needs state beyond its noun:

| Packet | Role | Ordering constraint |
| --- | --- | --- |
| `0x97 CombatantDataUpdate` | initial HP and mana; mask `0x03` in the current full baseline | after `0x8c`, before combat depends on those resources |
| `0x96 AttributeDataUpdate` | sparse indexed attributes, including maxima and combat tuning | after `0x8c`; no retail campaign field set is recovered |
| `0x8d ObjectUpdate` | transform/visibility and other `sporelabsObject` deltas | after `0x8c`; redundant position is accepted but not universally required |
| `0x99 AgentBlackboardUpdate` | target ID, in-combat, stealth, targetable, attacker count | after `0x8c`; target must exist before an ability or facing command consumes it |

The repository's `0x99` full shape is object ID, mask `0x1f`, target ID,
`isInCombat`, stealth byte, `isTargetable`, and attacker count. It is a
replicated blackboard snapshot, not a director command. Native/content evidence
still assigns target insertion to the missing authoritative director/AI owner.
Campaign target policy, threat amount, attacker-count ownership, perception
timing, and multi-player target selection remain campaign-specific.

There is no separate "enable AI" opcode. Activation is the composition of a
live object, its combat/attribute state, an object-local target/blackboard, and
the authoritative behavior/ability scheduler. Packaged AI and ability content
then determine what the agent does; the packet stream only reflects resulting
state changes.

## Locomotion and first action

Build 103 distinguishes three useful NPC locomotion shapes:

| State | Packet contract |
| --- | --- |
| stopped | known `0x91 ObjectPlayerMove` with flags `0x20` and stationary goal; retail dirty-flush decision remains unproved |
| active walking goal | `0x91` flags `0x01`, immediately followed by `0x95 LocomotionDataUnreliableUpdate` with the same partial goal |
| turn/faced target for accepted ability | one 81-byte `0x91` with flags `0x42`, stationary goal, normalized facing, target position, and target object ID; no `0x95` |

The flags-`0x42` path is native sender-proven for `faceTarget=true` ability
activation. It is not itself an AI-start packet and should not be sent before
the referenced target exists. A campaign agent outside its ability envelope
instead needs an authoritative walking goal and later ability admission. There
is no recovered universal pursuit inset, retry period, or spawn-to-first-action
delay.

Projectile locomotion is a different contract: projectile startup uses
`0x8c ObjectCreate`, attached `0x9b` trail, then reliable reflected `0x94`.
It must not be generalized back onto creature activation.

## `SpawnModifier`

Packaged Lua chunk `655`, resource `14222`, registers `SpawnModifier` with GUID
`0xd2a08fed`. Its exact authored behavior is:

1. `Activate`: stop the agent and add a modifier-scoped `Immobilized=1`;
2. `Tick`: add `generic_spawn.ServerEventDef`, set `horde_beam_in`, wait
   `0.5s`, and return;
3. `Deactivate`: remove the effect and reset animation;
4. native teardown removes the retained attribute handle after `Deactivate`.

The generic native modifier path publishes packed `0xa2 ModifierCreated`
before creating/activating its Lua execution. Its 37-byte payload contains
target ID, GUID, unique generational instance ID, duration, overdrive/rank,
stack count, 64-bit start time, source ID, and bound byte. Explicit removal
runs Deactivate, scoped cleanup, collection removal, then publishes eight-byte
`0xa4 ModifierDeleted(target, instanceID)`.

No recovered Cryos level script, marker record, client native call, or Lua
chunk attaches `SpawnModifier` to a particular director spawn. The external
server owner and its request parameters are missing. Consequently the GUID and
lifecycle are reusable facts, while source ID, rank, bound state, instance ID,
attachment time, removal predicate, and placement relative to first aggro must
come from campaign authority or a retail capture.

## Elite boss modifier

The packaged `EliteModifier` is modifier chunk 650
(`Modifiers/0x6D6C743A.lua`, SHA-256
`a8e72677986648e5bdbfa29e334c0dbbff57981a38576559f1e976cb7ad4b2dc`).
It is a unique-irreplaceable construction modifier with a one-million-second
duration, deactivation on agent destruction, and dehydration persistence. Its
non-minion branch adds 75 percent maximum health, 50 percent damage, and 25
percent body scale; its separate minion branch adds 400 percent health, 150
percent damage, and the same scale increase. Elite enemies, Captains, and every
higher boss tier use the non-minion branch: their prepared profile owns the
health increase, attack resolution owns the damage increase, and spawn/rejoin
publication emits the body-scale attribute and stable noun-owned
`EliteModifier` instance. Build 103 uses that modifier to render the enemy title
lettering red; there is no separate title-color field in the object-create or
blackboard packet. Authored visible NPC affixes remain independent and retain
their class-defined order.

The 0.7.24 Mizod report (`bosses-dont-have-their-elite-affix`, captured 2026-09-04) places the player at the 1-3 boss anchor; the server log admits object 548 as `NocturnaSpecialLeech_Captain.Noun`. Its imported identity owns `Spiky.NPCAffix` and `Aura_Spiky.NPCAffix`, but the shared spawn adapter previously published only `EliteModifier`. Spawn and rejoin now also publish each nonempty authored affix modifier after object creation, preserving authored order and using distinct stable noun-owned handles. The ability allocator skips their reserved generation range so an ability cannot overwrite an affix handle. Dedicated boss identities with no authored affixes still receive only the generic elite modifier; no random affixes are invented.

This change supplies the live modifier collection used by the target HUD. It does not implement missing server-side affix callbacks, aura membership, or reflected damage, and does not apply elite health/damage bonuses a second time. Production client verification remains outstanding; no builds, compilation, or tests were run for this report.

## Comparison with current tutorial enemy replication

Current tutorial replication is useful as a build-103-compatible adapter, but
it combines recovered facts with local policy:

| Current path | Packet sequence | Assessment for campaign reuse |
| --- | --- | --- |
| `marshalTutorialEnemy` | `0x8c -> 0x97 -> 0x96` | Reuse the packet families and create-first dependency. Replace tutorial HP/mana/attribute constants with campaign noun/difficulty state. |
| opening stationary enemy | above, then `0x8d visible -> 0x91 flags 0x42 -> 0xa5 character_teleport_in` | Reusable presentation primitives; tutorial-specific composition. It omits `0x99` until the separate first-aggro path. |
| opening first aggro | `0x8d visible -> 0x99 target/combat -> 0x91 flags 0x42 -> 0xa5` | Best reusable target-before-action ordering. The trigger boundary and `1.291667s` presentation are tutorial content. |
| tutorial horde spawn | `0x8c -> 0x97 -> 0x96 -> 0x8d -> 0x91 flags 0x42`, then local spawn-modifier output `0x91 stop -> 0x9b -> 0xa5`; after `0.5s`, `0x9b stop -> 0xa5 reset` | Receiver-valid local composition, not retail campaign proof. It does not emit `0xa2`/`0xa4`, does not publish `0x99` at spawn, and schedules attacks using local timers. |
| campaign setup | `0x8a game state -> 0x8b idle -> 0xaf reset`, then player objects/state | Reuse only for session initialization. It is not an enemy-wave activation sequence. |

The current horde adapter executes chunk 655 server-side and translates its
visible consequences, but deliberately models immobilization internally. That
is why its packet stream lacks the native modifier-instance lifecycle. A
campaign implementation may reuse the semantic simulator and packet encoders,
but it must choose explicitly between full modifier replication and the
current presentation-only compatibility model; they are not byte-equivalent.

## Reusable versus campaign-specific ownership

| Reusable build-103 mechanism | Campaign-specific or unresolved policy |
| --- | --- |
| `0x8c` field schema and create-before-reference dependency | noun pool, affixes, count, budget, difficulty scaling, marker selection |
| `0x97`, `0x96`, `0x8d`, and `0x99` component families | exact initial values, dirty masks, and flush cadence |
| per-object blackboard and target ID | threat insertion, multiplayer target choice, retargeting, attacker-count changes |
| `0x91`/`0x95` walking and `0x91` flags-`0x42` turn | navigation goal, activation envelope, retry tick, first-action timing |
| `0x8b` director schema | wave-container encoding, flag transitions, boss ID timing, late-join replay |
| generic `0xa2`/`0xa4` modifier lifecycle | whether `SpawnModifier` is attached, its request parameters, and removal owner |
| chunk-655 spawn effect/animation behavior | its ordering against invisibility, first aggro, targeting, and locomotion release |
| per-agent behavior/ability concurrency | director wave composition, reinforcement cadence, clear predicate, next-wave delay |

## Safe campaign contract and remaining capture target

A campaign sender can safely reuse the generic codecs only after its feature
owner has selected and committed the complete agent state. It should publish
`ObjectCreate` first, then the initial components, then target/locomotion or
modifier consequences in causal order. It should not use `8b 00` as a spawn
command, infer AI activation from visibility, substitute a fixed tutorial
timer for ability admission, or assume that modifier Tick completion deletes
the modifier.

A retail/server-binary capture is still required to make the sequence byte
exact. The minimum useful capture must include the last director packet before
a wave, every application payload through the first accepted agent action,
RakNet reliability/channel and datagram grouping, and the corresponding wave
completion. That would resolve director dirty order, `mActiveHordeWaves`, exact
component masks, `SpawnModifier` attachment/removal, target injection, and
late-join replay without promoting current tutorial policy into a campaign
contract.
