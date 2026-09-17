# Build-103 ordinary death live contract

## Scope and evidence labels

This note is limited to the ordinary, non-critical death and revival of a
replicated `TutorialBasicDiseased` NPC and to the boundary needed to connect
`server/sim.DeathBehavior` to gameplay. It does not cover the critical fast
branch, loot/rewards, or respawn ownership.

The labels below are intentional:

- **Direct native packet evidence** means the build-103 native called a GMS
  sender and the application-message bytes can be recovered from that sender
  and the registered reflection fields. The attached-effect recipes still lack
  a retail framed capture, so RakNet reliability/channel framing is not claimed.
- **Ordinary replication evidence** means the native mutates authoritative
  object/component state and the normal replication pass owns any packet. The
  Lua/native operation itself does not send one.
- **Inference/open** means the semantic consequence is required, but the exact
  build-103 delta sender or a live packet has not been recovered. Such a packet
  must not be invented in the death adapter.

## Contract summary

For ordinary `TutorialBasicDiseased`, the authored timeline is:

| Time | Authoritative action | Wire owner |
| --- | --- | --- |
| `t=0` | Apply the killing combat/HP-zero boundary; clear the target; choose and set the death animation; add scoped `Immobilized += 1`; stop locomotion; disable physics and navigation collision. | Killing combat, HP zero, and `SetAnimationState` are packets. Target, modifier, locomotion, and collision changes are authoritative state whose further deltas belong to ordinary replication. |
| `0 < t < 10s` | Wait for HP to rise above zero. | The wait itself emits nothing. Revival exits through cleanup. |
| `t=10s`, still at zero HP | Set corpse-fading state, allocate an attached-effect slot, publish `fadeaway_bio.ServerEventDef`, then wait five seconds. | The state flag is ordinary combatant state. `AddEffect` directly publishes `ServerEvent` when the object is replicated. |
| `10s < t < 15s`, interrupted/revived | Cancel the pending delete, remove the immobilization modifier, reset animation, remove the fade slot, clear corpse-fading, and restore physics/navigation collision. | Reset animation and effect removal are direct packets. The other changes remain ordinary state/replication. |
| `t=15s`, still dead | `MarkForDelete`; the object-manager deletion sweep performs full teardown and normal replication emits `ObjectDelete`. | `MarkForDelete` itself emits no packet. |

`WaitForHitpointsAbove` may also complete because the object vanished or was
already corpse-fading. Those are terminating/cancellation cases, not revival.

## Direct native packet evidence

### Force-attached Life fade

`TutorialBasicDiseased.NonPlayerClass.creatureType` is Life, selecting
`fadeaway_bio.ServerEventDef`. Its case-insensitive FNV-1 catalog ID is
`0xea273c08`.

Build-103 `AddEffect` is registered to `sub_A0B410`. It scans the 16 effect
records at object `+0xc8` in ascending order and chooses the first record whose
asset word is zero. The internal index and Lua handle are zero-based `0..15`.
The reflected packet field is one-based `1..16`.

For a replicated object, the exact application message is 16 bytes including
the `0x9b` opcode:

```text
9b 01 SS 04 01 06 08 3c 27 ea 07 OO OO OO OO ff
```

Where:

- `01 SS` is reflected field 1, the one-based slot `SS = internalIndex + 1`;
- `04 01` is field 4, `forceAttach=true`;
- `06 08 3c 27 ea` is field 6, asset `0xea273c08` little-endian;
- `07 OO OO OO OO` is field 7, the target object ID little-endian;
- `ff` ends the reflected field list.

Slot `01` is correct only when the corpse has no earlier attached effect. The
gameplay adapter needs a per-object 16-slot pool; hard-coding slot 1 is not the
recovered contract. If all 16 slots are occupied, `AddEffect` returns `-1` and
does not publish this packet; the five-second death deadline still continues.

### Effect removal on revival/interruption

`RemoveEffect` is registered to `sub_A01330`; `RemoveEffectIndex` is
`sub_A014D0`. Both clear the selected internal slot before publishing the
removal. The ordinary `Behavior_Death` cleanup calls the non-hard form, so the
exact application message is 11 bytes including `0x9b`:

```text
9b 01 SS 02 01 07 OO OO OO OO ff
```

Field 2 is `remove=true`; the same one-based slot and object ID identify the
attachment. Field 3 (`hard stop`) is absent/false. The optional hard form would
insert `03 01`, but ordinary death cleanup does not request it.

There is no fade-removal packet in either of these cases:

- revival before the ten-second timeout, because no fade slot exists yet; or
- the normal still-dead path at 15 seconds, because it goes directly from the
  attached fade to object teardown/`ObjectDelete`.

### Death and revival animation timestamps

Content recipe 54 additionally follows every imported non-player noun's `.CharacterAnimation` reference and stores its first ordinary (noncritical, non-knockback) death state in `npc_death_animation`. Campaign death publication uses that rig-specific state for ordinary and captain variants, without changing physics or damage. For example, Pterodyne's `NPC_Stealthy.CharacterAnimation` selects `gen_death_melee`, not the Zelem spider state. Boss presentation and detonation overrides remain authoritative. This supplies variation between rigs, not random cross-rig animations or new client assets.

The ordinary Basic Diseased and Basic Poison melee death state is
`zlm_minn_sp_3_death_melee`, ID `0x99d9fb45`. Plasma-backed Ranged and
Special One use `gen_death_melee`; Quad-backed Sloth uses
`gen_death_melee_quad`. These are the first noncritical death entries in their
four exact type-`0x17BBCE29` CharacterAnimation resources.
`SetAnimationStateToDeath`
(registered entry `loc_A018D0`) resolves the descriptor-selected
state, then samples the clock immediately before calling packet sender
`sub_A1FDD0`:

```text
sub_9BCBE0() -> simulator
sub_9D2920(simulator) -> uint64 simulation milliseconds
```

`sub_9D2920` follows simulator `+0x39e84` to the simulation-clock object and
returns the 64-bit word at clock `+0x08..+0x0f`. This is current game simulation
time, not wall time, callback arrival time, animation duration, or the killing
packet's timestamp copied blindly.

The death application message is 26 bytes including opcode (`25`-byte body):

```text
a5
OO OO OO OO                         object ID
45 fb d9 99                         death state 0x99d9fb45
TT TT TT TT TT TT TT TT             current simulation ms, uint64 LE
00                                  overlay=false
00 00 80 3f                         scale=1.0
00 00 00 00                         source/echo gate=0
```

`ResetAnimationState` (`sub_A02130`) clears the stored state and independently
samples the same current simulation clock immediately before the same sender.
Revival therefore sends the same shape with `state=0`, a fresh timestamp,
`overlay=false`, `scale=1`, and source/echo zero.

## Ordinary replication, not direct death packets

### Immobilization, stop, target, and collision

`Behavior_Death` adds a scoped `Immobilized += 1` modifier and stops ordinary
locomotion. On cleanup it removes that exact modifier handle. Immobilization is
authoritative attribute/modifier state; neither the Lua attribute call nor the
death behavior has a direct GMS `ServerEvent`-style sender. Build 103 has an
`AttributeDataUpdate` (`0x96`) family, but the version-local numeric
`Immobilized` field and the exact dirty flush for this mutation are not
packet-proven here. Do not fabricate a field number or an extra packet in the
death presentation adapter.

Target clearing is likewise an authoritative simulation mutation. Locomotion
stop must prevent new movement goals while dead; whether a particular stop
requires a movement/locomotion delta depends on the ordinary replication
state, not on `DeathBehavior` emitting a special death packet.

Physics and navigation collision are two separate authoritative mutations:

- `SetObjectAsCollidable` (`sub_A00310`) updates the physics manager;
- `SetNavCollision` (`sub_A03CC0`) stores the inverted enabled byte at object
  `+0x284` and rebuilds navigation through `sub_9E6930`.

Neither wrapper calls a GMS sender. Build-103 object reflection proves collision
field 17 in object snapshots, but the runtime collision-delta sender and
whether physics additionally uses `PhysicsChanged` (`0x93`) remain open.
Consequently the gameplay connection updates physics and navigation authority
immediately. Darkspin's compatibility adapter also publishes the minimal
`ObjectUpdate` field-17 delta so the build-103 client stops treating the corpse
as a movement obstacle; the exact retail dirty-flush owner remains open.
Revival/interruption restores both only while the object has not been marked
for deletion.

The simulator represents these as two ordered `PhysicsStateIntent` events,
tagged `physics` and `navigation`. Only the physical event crosses the
build-103 packet encoder; the navigation event is retained in the semantic
trace and applied by authoritative world state without inventing a second
wire message.

Critical non-player, non-boss deaths now retain the descriptor-bearing combat
event and use the recovered fast corpse branch: immediate corpse-fading state,
a 0.1-second settle, locomotion stop, and deletion after another three seconds.
Bosses retain their ordinary revival/fade lifetime, and stationary fixtures
retain their dedicated short destruction delay. The adapter still uses each
noun's indexed rig-compatible death state rather than fabricating an animation
name absent from the projected `CharacterAnimation` resource.

The live death-run adapter retains the complete reversible authority snapshot:
target-cleared, immobilized, locomotion-stopped, corpse-fading, marked-delete,
physical collision, and navigation collision. Encounter and horde target maps
remove a defeated actor immediately, while this run owns its remaining corpse
lifetime. Revival or cancellation removes immobilization and restores both
collision states before that owner is retired.

### From `MarkForDelete` to `ObjectDelete`

`MarkForDelete` (`sub_A02910`) only sets simulated-object byte `+0x5d` to one.
It does not call the `ObjectDelete` sender. The authoritative simulator update
advances Lua threads near its start, runs the intervening systems, and executes
deletion sweep `sub_9BD430` at the tail. Therefore:

- a mark made by the death continuation during that update is eligible for
  teardown later in the same simulation update;
- a mark made after the sweep waits for a later update;
- packet publication occurs only after the sweep hands removal to normal
  replication, so `t=15s` is the authored mark deadline, not a promise that a
  datagram is written at that exact instant.

For one object, the resulting application message is the ordinary five-byte
delete form:

```text
8e OO OO OO OO
```

The payload is a packed list of little-endian object IDs, so one `0x8e` may
batch multiple deletions. Full teardown also owns attached-effect-slot cleanup;
the normal timeout path must not precede it with the soft fade-removal packet.

## Gameplay connection requirements

The adapter between `server/sim.DeathBehavior` and gameplay should preserve
these ownership boundaries:

1. Resolve the target role to the live replicated object and keep it alive
   through the death and fade presentations.
2. Stamp death/reset animation from the session's authoritative simulation
   millisecond clock at the semantic event time.
3. Maintain the per-object 16-slot attached-effect pool and return/reuse the
   zero-based handle while encoding the one-based wire slot.
4. On revival, cancel the pending continuation before cleanup; remove only a
   slot that was actually allocated; reject revival after mark-for-delete.
5. Apply immobilization, target, locomotion, corpse-fading, and both collision
   mutations to authoritative state without turning them into invented direct
   presentation packets.
6. Treat `MarkForDelete` as a state transition. Let the deletion sweep and
   replication layer produce/batch `ObjectDelete` and perform final slot and
   object-owned cancellation cleanup.

## Remaining packet evidence targets

The semantic contract above is sufficient to connect the behavior without an
immediate-delete shortcut. These wire details remain deliberately open:

- a retail framed golden for the 16-byte attach and 11-byte soft removal;
- the numeric/delta form, if any, used when `Immobilized` becomes dirty;
- the exact runtime collision delta(s) after physics/nav disable and restore;
- the precise sweep-to-network-flush latency and RakNet batching/framing for
  the eventual `ObjectDelete`.

Until those are captured or their build-103 senders are recovered, they belong
to ordinary replication rather than `DeathBehavior`'s direct packet outbox.
