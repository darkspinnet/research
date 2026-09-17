# Post-Quadra route ownership

## Scope and conclusion

This note traces the authoritative route from runtime object `42`
(`TutorialSpecialOne_Intro.Noun`, called Quadra here) through the boss-security
teleporter and into the horde arena. Evidence labels are intentionally strict:

- **Proven** means content linkage, decoded Lua 5.1 bytecode, recovered native
  code, an exact packet fixture, or a documented build-103 live observation.
- **Inferred** means the ordering is necessary or best-supported, but its retail
  caller, transition, or packet sender has not been recovered.
- **darkspin policy** describes the current playable reconstruction and is not
  evidence of retail ownership.

The main result is negative: **object `42` defeat does not directly own retail
teleporter activation**. Object `42` is at `(259.346,81.391,25.088)`, about
`151.48` units from the platform at `(260.833,232.784,20.168)`, and therefore
cannot be a member of the platform passive's radius-`20` security query. The
retail activation owner is `BossSecurityTeleporterPassive` (Lua chunk `144`),
which continuously counts a separate five-placement guard set around the
platform. The route edge connecting Quadra's defeat to those guards remains
unrecovered.

The recovered ownership chain is:

```text
object 42 defeat
  -> [missing retail route/director edge]
  -> five platform guards become countable and are defeated
  -> BossSecurityTeleporterPassive, chunk 144: zero-guard transition
  -> active trigger accepts a player-owned entrant
  -> TeleporterModifier, chunk 349: out / teleport / in sequence
  -> destination marker 174193625
  -> boss director marker 1730050752
  -> server-side DirectorTrigger_SpawnBoss
  -> "horde triggered02" fan-out to two HordeSpawner_Register listeners
```

No recovered objective update owns any edge in that chain.

## Proven ownership

### 1. Object `42` defeat is not the teleporter-clear predicate

Object `42` is darkspin's runtime identity for the colocated
`TutorialSpecialOne_Intro.Noun`. Its introduction ability is
`FirstAggro_SpecialOne` (chunk `502`). That ability owns only the cinematic,
visibility, `character_teleport_in`, and its `4.291667s` presentation lifetime.
It contains no objective operation, teleporter operation, director event, or
post-death callback.

The security platform's actual scan center is its own position, and chunk
`144` uses radius `20`. Object `42` is about `151.48` units away. This spatial
fact independently excludes it even before the scan's type/team/visibility
filters are applied.

The only fixed AI placements within radius `20` are a different Special One,
two Poison, and two Ranged enemies:

| AI ordinal | Noun | Distance from platform |
| ---: | --- | ---: |
| `0` | `TutorialSpecialOne.Noun` | `7.99` |
| `1` | `TutorialBasicPoison.Noun` | `7.99` |
| `14` | `TutorialBasicPoison.Noun` | `7.32` |
| `15` | `TutorialBasicRanged.Noun` | `7.62` |
| `21` | `TutorialBasicRanged.Noun` | `6.69` |

All other fixed placements are at least `88.11` units from the platform.
Direct `ai.markerset` decoding with the correct `0xc8` record stride supplies
these coordinates; the currently normalized rows after ordinal zero use an
incorrect `0xd0` stride and are not coordinate evidence.

**Ownership consequence:** a direct object-`42` death listener must not unlock
the retail platform. The authoritative condition belongs to the platform
passive and its local guard query.

### 2. `BossSecurityTeleporterPassive` owns activation

The content edge is exact:

```text
BossSecurityTeleporter.Noun
  -> BossSecurityTeleporter.AIDefinition
  -> BossSecurityTeleporterPassive
  -> chunk 144 (resource 13674)
```

The decoded chunk SHA-256 is
`c2e727c6cf4c13708fa315321fb51f0402a3a4a9763eaf2f8d782c1eca3267f0`.
It registers both generic `SecurityTeleporterPassive` and the boss-specific
`BossSecurityTeleporterPassive`.

Its server-authoritative `tick()` loop queries radius `20` for
`nSporeLabs.organicDamageableObjectTypes`, proven by chunk `659` to contain
exactly creature type GUID `0x06c27d00`. A candidate counts only when all of
these are true:

1. `IsAlive(object)` is true;
2. `GetTeam(object) == 0`;
3. `InvisibleToSecurityTeleporters == 0`.

The three focused noun families all deploy as creature type `0x06c27d00` and
team `0`. Their shared invisible behavior (chunk `865`) adds attribute `112`,
`InvisibleToSecurityTeleporters=1`, while hidden and removes it during behavior
deactivation. The behavior-scheduler condition that performs this deactivation
is not yet recovered, but ownership of the exclusion itself is proven.

The state changes and delays in chunk `144` are exact:

| Condition | Authoritative transition |
| --- | --- |
| Stable state | wait `0.5s`, then scan again |
| Nonzero count while active | remove active effect; add `powerDown`; wait `1s`; add inactive effect; set inactive; the replaced power-down handle is not explicitly removed |
| Zero count while inactive | remove inactive effect; add `powerUp`; wait `1s`; remove it; add active effect; set active |

The `1s` power-down delay actually reads the field named
`teleportActivationTime`; the authored `teleportDeactivationTime` field is not
read. The boss assets are:

- `zelem_boss_teleporter.ServerEventDef`;
- `zelem_boss_teleporter_inactive.ServerEventDef`;
- `zelem_boss_teleporter_powerup.ServerEventDef`;
- `zelem_boss_teleporter_powerdown.ServerEventDef`.

These are replicated presentation effects. They are not objective updates and
their asset-bearing `ServerEvent` shape is distinct from a field-`15`
`clientEventID` notification.

### 3. Chunk `144` accepts entry; chunk `349` owns teleport timing

On activation, chunk `144` creates a spherical trigger of radius `2` at the
platform. An entrant qualifies when it is player-controlled or when its owner
is player-controlled. Entry is accepted only while the private teleporter
state is active.

`CreateTriggerVolume(x,y,z,radius,callback1,callback2,callback3)` retains all
three closures and the creating Lua thread. Native physics contact mask bits
`1`, `2`, and `4` dispatch callback one as enter, callback two as exit, and
callback three as repeated stay/contact respectively. Every closure receives
exactly `(triggerID, entrantObjectID)`. Chunk `144` supplies native delay `0`
and repeat flag `0`, so stay is not latched; its cadence is each bit-`4` physics
notification, not a recovered fixed timer. `DestroyTriggerVolume` unreferences
the closures and removes the volume without synthesizing an exit callback.

The entry callback resolves the noun-linked destination with
`GetTeleporterDestination` and requests modifier GUID `0x502f1932`. Native
registration and hashing prove that GUID names `TeleporterModifier`, chunk
`349` (resource `13896`, SHA-256
`9417dfe9163dbb857dc1b423d4e127170a6ba9e128f583d648fe7f258a1c1cd1`).
It is not the separately packaged `HordeGateTeleporter` modifier.

Chunk `349` owns this exact sequence:

| Offset from accepted modifier activation | Action |
| ---: | --- |
| `0` | stop entrant; add `Immobilized=1`; set `character_teleport_out` |
| `+0.5s` | authoritative `TeleportObject` to the three destination properties |
| `+0.5s` | set `character_teleport_in` after the authored zero-duration wait |
| `+1.0s` | final wait completes |

The modifier has no positive authored duration and its empty Lua deactivate
callback does not explicitly remove immobilization. Native modifier teardown
does remove registered scoped handles. Unique replacement and phase/session
teardown are proven cleanup paths; ordinary first-instance release after the
final wait is still unresolved.

The entry closure's exact native call is
`RequestModifier(entrant, teleporter, 0x502f1932, kObjIDNone, 1, x, y, z)`;
chunk `144` discards the returned numeric instance/request ID. The native
definition is `Unique`. A request that reaches native creation first deactivates
and removes every existing same-GUID instance, then creates the replacement.
Normal callback-three traffic avoids that replacement path by calling
`GetFirstModifierByGUID` and suppressing the request while any matching
instance remains.

Creation emits packed logical message `36`, wire `0xa2`, `ModifierCreated`
before the modifier's Activate body runs. Its 37-byte body carries target,
GUID, instance ID, duration, overdrive, stack count, 64-bit start time, source,
and bind byte; here the target is the entrant, the source/initiator is the
teleporter, the GUID is `0x502f1932`, rank/stack input is `1`, and authored
duration is `0`. Chunk `349` then owns the server mutations and their separate
presentation: scoped `Immobilized`, wire `0xa5` teleport-out animation, wire
`0x90` authoritative teleport, and wire `0xa5` teleport-in animation.

Activate coroutine return only leaves dormant thread state `-1`; it neither
deletes the modifier nor emits replication. Explicit removal follows
`sub_9E12D0 -> sub_9E09D0 -> sub_9E11F0`: run the empty Deactivate callback,
release and clear the instance's retained Lua thread, remove remaining scoped
attribute handles and the owned instance, then emit logical message `38`, wire
`0xa4`, `ModifierDeleted` with `(target, instanceID)`. Full object teardown
reaches the same owned-modifier cleanup through
`sub_9D1B30 -> sub_9DEA80`. Logical message `37`, wire `0xa3`, is modifier
update and is not a deletion substitute. No inspected native path turns the
final Activate return or the zero authored duration into ordinary release.

Native `TeleportObject` (`sub_A09A90`) mutates authoritative position/facing
and reaches sender `sub_A20110`, which emits the 32-byte `ObjectTeleport`
payload (`0x90` after the application ID). Thus the position change is
server-owned; the animation messages are surrounding presentation.

### 4. The destination enters the arena director

The platform is marker `495984414`, authored as
`BossSecurityTeleporter.Noun` and deployed by the current fixture as runtime
object `28`, at `(260.83334,232.78407,20.16802)`. Its linked destination is
marker `174193625`, whose marker and noun identities are both
`TeleporterSpawnPoint.Noun`, at `(-347.57553,-224.60829,10.08803)`.

That destination lies inside two distinct radius-`30` server boundaries:

- tutorial marker `1359119906`, callback
  `nTutorial_IntroOverdriveActivate.main` (chunk `141`);
- boss director marker `1730050752`, callback
  `DirectorTrigger_SpawnBoss`.

Chunk `141` is a proven no-op job: it waits `3.75s`, enumerates players, runs
an empty numeric loop, deletes its job object, and emits no mutation, objective,
packet, or event. It must not be credited with arena or horde ownership.

The boss director publishes event `horde triggered02` to exactly two
`HordeSpawner_Register` listeners at the two authored spawn loci. Its component
also carries `waveOverride=4`. Neither callback exists in the 1,029 indexed Lua
chunks. Build 103 registers the client-side `DirectorTrigger_SpawnBoss` bridge
to `sub_9FACF0`, but that branch only validates the entrant, fetches the
director, and returns false without mutation or GMS. `HordeSpawner_Register`
is absent from the executable. The authoritative horde implementation is
therefore the missing **server-side director**, not chunk `141`, not
`ActivateHordeSpawn`, and not the inert client native.

The director's first-wave activation delay is `2s`. The current reconstructed
`1.5s` inter-wave delay and deterministic two-enemy wave composition are not
retail-proven; `waveOverride=4` supports four waves but does not reveal budget,
counts, or selection order.

## Objective and client-event audit

### Proven negatives

- Chunks `502`, `144`, `349`, and `141` contain no objective operation for this
  route.
- The recovered teleporter and entry chain does not emit `ObjectiveUpdated`
  (`0xB8`), `ObjectivesComplete` (`0xB9`), or `ObjectiveAdd` (`0xCA`).
- No objective ID has been content-linked to object `42` defeat, platform
  activation, accepted teleporter entry, or director entry.
- `TutorialGameMsgs` (`0xC8`) is not a post-Quadra or arena-entry message. Its
  positive subtype-`0` snapshot is a later tutorial-completion client snapshot.

### Client events and presentation

The active/inactive/power-up/power-down effects are `ServerEvent` (`0x9B`)
asset presentations. A standalone `ServerEvent.clientEventID` uses reflected
field `15` and has a different semantic role.

Two build-103 horde alert IDs are receiver- and live-proven in darkspin:

| Event | field-`15` ID | Client result |
| --- | ---: | --- |
| incoming | `0x1d42121d` | `Horde incoming!` |
| defeated | `0x8047eaf4` | `Horde defeated!` |

The build-103 `ServerEvent` handler passes nonzero field `15` directly to the
local alert dispatcher. A focused live run observed the incoming event in the
same response batch as successful `ObjectTeleport`, followed by wave one after
`2s`. This proves the client consumption and darkspin ordering. The exact
retail server constructor and whether retail emits the incoming alert before
or after the authoritative director transition remain unrecovered.

## Packet evidence

The following evidence must not be conflated with the missing retail caller.

### Exact wire fixtures and native sender

- Object `42` deletion has application bytes `8e 2a 00 00 00` in darkspin's
  current damage path. This proves object removal encoding, not that deletion
  directly activates the retail platform.
- The active platform presentation currently used by darkspin is exactly:

  ```text
  9b 06 87 55 56 6d 07 1c 00 00 00 0a
  ab 6a 82 43 b9 c8 68 43 1b 58 a1 41 ff
  ```

  Here field `6` is hash `0x6d565587` for
  `zelem_boss_teleporter.ServerEventDef`, field `7` is platform object `28`,
  and field `10` is the platform XYZ. No field `15` and no objective packet is
  present.
- For active hero object `1`, the arena position change is:

  ```text
  90 01 00 00 00 ab c9 ad c3 b9 9b 60 c3 92 68 21 41
  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  ```

  The object ID varies with the deployed hero. The destination and 16-byte
  zero orientation tail are fixed by the current fixture. Independently,
  recovered native `TeleportObject` reaches the build-103 `0x90` sender.
- The incoming alert is `0x9B` with reflected field `15 = 0x1d42121d`; codec
  tests pin the field at the packet tail and live testing confirms the client
  banner. It is not `SystemMessage` (`0xA0`).
- The surrounding modifier presentation is `SetAnimationState` (`0xA5`) for
  `character_teleport_out` and `character_teleport_in`, separated from `0x90`
  by the `0.5s`, `0s`, and `0.5s` waits in chunk `349`. A clean retail packet capture
  of this complete platform sequence is not retained.

### Current darkspin policy, not retail proof

The current handler uses object `42` defeat as the still-inferred route edge
that deploys the five content-proven platform guards. It assigns them runtime
objects `43..47` at the decoded authored coordinates and keeps chunk `144`
inactive until all five are dead. This preserves the proven security predicate
without claiming recovery of the missing reveal/director owner. Cryos already
constructs platform marker `495984414` as runtime object `28`, so darkspin no
longer sends a second noun creation for it; it publishes only the passive
inactive/active effect and owns the radius-2 trigger.

Likewise, current teleporter use is detected with a swept radius `6.67`, emits
`0x90` immediately, appends the two arena obelisks and incoming field-`15`
event, schedules wave one after `2s`, and separately schedules chunk `141`'s
`3.75s` no-op. That is a useful compatibility harness, but it does not reproduce
chunk `349`'s `out -> 0.5s -> teleport -> 0s -> in -> 0.5s` sequence.

## Inferred retail route

The smallest route consistent with all proven evidence is:

1. Object `42` dies, its introduction run is cancelled/retired, it is removed,
   and its XP is awarded. No teleporter or objective transition is assigned to
   this defeat.
2. An unrecovered encounter/director edge advances the player toward the five
   fixed enemies around the security platform and makes each guard countable by
   deactivating `nBehavior_Invisible` before or during first aggro.
3. As guards die, chunk `144` continues its `0.5s` stable-state scans. The last countable
   guard's removal produces zero on a subsequent scan; if the platform is
   inactive, it runs the `1s` power-up transition and becomes active.
4. A qualifying entrant crosses the active radius-`2` trigger. Chunk `144`
   resolves destination marker `174193625` and requests chunk `349`.
5. Chunk `349` immobilizes the entrant, plays teleport-out, waits `0.5s`,
   teleports authoritatively, performs a zero-duration authored wait, plays
   teleport-in, and waits another `0.5s`.
6. Arrival enters boss director `1730050752`. The server implementation of
   `DirectorTrigger_SpawnBoss` publishes `horde triggered02` to both registered
   listeners and begins the first wave after `2s`.
7. Marker `1359119906` may run concurrently at arrival, but its only effect is
   a cancellable `3.75s` wait and job deletion.

Steps 2 and the exact ordering within step 6 are inferred. Steps 3-5 are
bytecode/native-proven once their prerequisites occur.

## Remaining recovery targets

1. The server/director transition after Intro Quadra's defeat: how it exposes,
   spawns, or advances to the five platform guards.
2. The behavior-scheduler edge that deactivates `nBehavior_Invisible`, removing
   attribute `112` and making each living guard countable.
3. A retail packet capture spanning last platform-guard death, power-up,
   teleport-out, `0x90`, teleport-in, incoming alert, and first wave.
4. The retail sender/order for the incoming field-`15` alert and the exact
   server-side `DirectorTrigger_SpawnBoss` wave budget and fan-out.
5. Confirmation that the absence of an objective update is intentional, or an
   objective packet/content dependency not represented by the recovered chunks.

Until those targets are recovered, implementation should model explicit
`quadraDefeated`, `teleporterGuardsCleared`, `teleporterEntered`, and
`arenaDirectorEntered` facts rather than allowing object `42` deletion to set
the platform active directly.

## Evidence locations

- `notes/tutorial/route.md`: route topology, guard set, bytecode/native findings,
  and live observations.
- `notes/tutorial/overview.md`: full chunk inventories, native ABI work, packet/live
  ledger, and current reconstruction caveats.
- `notes/content/script-linking.md`: exact level-marker-to-callback links.
- `bin/game/logs/tutorial-priority-bytecode/chunk-144.luac`: security
  teleporter passive.
- `bin/game/logs/lua-corpus/0349-13896.luac`: `TeleporterModifier`.
- `bin/game/logs/cryos-route/design.markerset` and `ai.markerset`: platform,
  destination, director/listeners, and fixed guard placements.
- `server/gameplay_udp.go` and `server/gameplay_udp_test.go`: current darkspin
  policy and exact compatibility packet fixtures.
- `server/raknet/application.go` and `server/raknet/application_test.go`: build-
  103 application encoders and exact movement/teleport shapes.
