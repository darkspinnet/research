# Tutorial Lua authority and reverse-engineering queue

This note classifies the Lua jobs and native commands associated with the
build-103 tutorial. Its purpose is to keep client presentation callbacks,
documented server-authority behavior, and unresolved server behavior separate.
The unresolved server rows are the immediate reverse-engineering homework.

The authoritative script inputs are the original Lua 5.1 bytecode resources in
`bin/darkspinner/darkspin/cache/content.db`. The stable DBPF resource groups are:

| Resource group | Count | Authority role |
| --- | ---: | --- |
| `0x24F78AA1` | 8 | Server-authority tutorial jobs |
| `0xA35BED24` | 8 | Client presentation tutorial jobs |

The server-group bytecode inventory is now losslessly extracted with
`darkrun db server_data bget decoded_payload ... --decode zlib`:

| Resource | Mapped job | Size | SHA-256 |
| --- | --- | ---: | --- |
| `0x724391F1.lua` | `Tutorial_IntroOverdriveActivate` | 847 | `909a2ff8a3494f6b9bcfd26461cbe72538d8161778771cc8bc5e11792fcf1ffa` |
| `0xBA0E5C06.lua` | `Tutorial_IntroAbilitySecond` | 889 | `07c3a923a272fc2313dd00876044398597fd1107e833037cd0e2cf2b7e57ec64` |
| `0xE6FA7F3F.lua` | `Tutorial_IntroSecondCreatureUnlock` | 1,033 | `e98b1e855e64904689451bd79635c3487a7a20e29f1b943885c9648b61f8e30a` |
| `0x5C4C5BBF.lua` | `Tutorial_RandomUnlock` | 1,051 | `b9bf494972e98b73e60fe527bbfcaec6b66434abea4545adacbdb343c6364878` |
| `0xC9B6860B.lua` | `Tutorial_SupportUnlock` | 1,053 | `2079148a63af96dd765857ae51298bc86a65a9a9ae4d7a0ca8145e204a400509` |
| `0x90CE5ECC.lua` | `Tutorial_SoloSupportUnlock` | 1,172 | `b67c0aace5d108e5f6f7b2432f5d8be1aaee3f00d1204fdd56c6b03abcd878c4` |
| `0xE137EF5C.lua` | `Tutorial_OverdriveUnlock` | 1,182 | `cbec31d39922671c12c67caa2482bfaa139d795d4dd4a96266b5ad235bd2fef4` |
| `0xB8716CFB.lua` | `Tutorial_CatalystUnlock` | 1,216 | `13f084e59a192f55a30e8285e8ebb651bfcaa90ae0ab16f5235c3ffa28fa09db` |

The authored names below come from the bytecode-to-source mapping. The DBPF
index retains synthetic resource identities rather than original filenames, so
every instruction-level result must retain both the resource key and mapped
name.

## Classification rules

- **Client only** means the retail client executes the job and its meaningful
  calls target presentation systems such as `nUIManager`. The server must make
  the prerequisite state true and activate the authored marker, but must not
  reproduce the UI call as a server command.
- **Server driven, documented** means the server-side Lua order and native
  mutation are recovered well enough to express as a typed director intent.
  This does not imply that every resulting packet, persistence write, or live
  ordering edge is confirmed.
- **Server driven, undocumented** means a server-authority job, native side
  effect, route dependency, or resulting wire transition is still missing.
  These rows must not be implemented by guessing from a script name.

`serverOnly` on a level marker is an authority boundary. It is not merely an
extraction flag. A client-only job may still require the server to expose an
ability, create an object, or advance an objective before the client callback
can present anything.

## Client-only tutorial jobs

These eight jobs belong to group `0xA35BED24`. None grants authority to mutate
combat, inventory, squad ownership, progression, or persistence. All eight
original chunks are now indexed and instruction-decoded from `content.db`:

| Resource | Mapped job | Size | SHA-256 |
| --- | --- | ---: | --- |
| `0x9E23DE1E.lua` | `Tutorial_IntroAbilities` | 851 | `8274e9d35a88c2697ca53c0f8583107cd58c7390f533fa4326302825023f7985` |
| `0xAC4FD6FE.lua` | `Tutorial_IntroHealth` | 441 | `788303c5a04b20db6df509630c35120209bdf99436935a48240188447d1be556` |
| `0xBCE26B84.lua` | `Tutorial_IntroHealthAndPower` | 1,167 | `f267ff878ef2ef61f913f38fb3471bbc5b6fbc768607d3188bebeb2fd0dd0b2a` |
| `0x40449A11.lua` | `Tutorial_OverdriveUnlockClient` | 1,036 | `d274bc0c751d36bfe4c5c0ed2b2f83fcd8cbbc53142f676454c7e636a0b54180` |
| `0x22EC11F2.lua` | `Tutorial_RandomUnlockClient` | 1,007 | `ff7b4677d624fb57c73dc9078d2d4f00fb71a7e6d03aed8d3315903aaab469e3` |
| `0xAFA67BFE.lua` | `Tutorial_SupportUnlockClient` | 1,094 | `ab5dac04256d1942e46f603f5935c735380d6c614c23f4df7b0d2d0df6181231` |
| `0xB005B821.lua` | `Tutorial_SoloSupportUnlockClient` | 1,081 | `afb48133fc171b8f9eca258893bceaa63e1721b609eddf81349decdde8249966` |
| `0xB907420E.lua` | `Tutorial_CatalystUnlockClient` | 899 | `672c517618407c6ba852958eb6be8e77dbeebaa077d32549ef357097f21951b5` |

| Job | Recovered client behavior | Server responsibility |
| --- | --- | --- |
| `Tutorial_IntroAbilities` | Wait `2s`, then `nUIManager.PlayAbilityBlink(Ability_Enrage1)`; delete the job. | Expose Ride at the correct ability boundary and activate the client marker. Clear the related objective after the first accepted cast. Do not synthesize the HUD blink. |
| `Tutorial_IntroHealth` | Wait `3s`, then call `PlayHealthBlink`. | Make the health lesson marker and resource state valid. |
| `Tutorial_IntroHealthAndPower` | Wait `1s`; blink health every `1.5s` for `7s`; then blink mana every `1.5s` for `8s`; delete the job. It reads `nGameSimulator.GetGameTime`. | Maintain the shared gameplay timeline and authoritative HP/mana state. The blink loop is local presentation. |
| `Tutorial_OverdriveUnlockClient` | If the local player has beaten the level, return immediately. Otherwise wait `3s`, then `5s`; blink overdrive; repeat wait `2.5s` and blink three more times; wait `1.5s`, then `4s`; return. It does not delete its created job object. | Perform a separately validated server unlock only when its owning route is identified. |
| `Tutorial_RandomUnlockClient` | If beaten, return immediately. Otherwise wait `2`, `4`, and `0.5s`; blink `Ability_Enrage2`; wait `5.5` and `5s`; return without deleting the job object. | Perform the paired server ability-boundary mutation. |
| `Tutorial_SupportUnlockClient` | If beaten, return immediately. Otherwise wait `2`, `4`, and `0.5s`; blink `Ability_Support1`, `Ability_Support2`, and `Ability_Support3` consecutively; wait `5.5` and `5s`; return without deleting the job object. | Perform the paired server support unlock. |
| `Tutorial_SoloSupportUnlockClient` | If beaten, return immediately. Otherwise wait `2`, `4`, and `0.5s`; blink the same three support slots consecutively; wait `6.5s`; return without deleting the job object. | Perform the paired server support unlock. |
| `Tutorial_CatalystUnlockClient` | If the local player is unbeaten, wait `2s` and then four `4s` intervals; return. If beaten, return immediately. Instruction decoding proves there are no calls between the waits. | This is authored no-op client timing. The server performs only the separately defined Catalyst server job; it must not invent missing client presentation. |

The Cryos level has two nearby but distinct ability markers. The client-visible
`Tutorial_IntroAbilities` marker is not the server ability unlock. The separate
server marker waits one second and changes the ability boundary. A movement
segment can cross both markers, but the director must still dispatch them as
two authority domains.

## Server-driven behavior documented well enough to model

### `Tutorial_IntroAbilitySecondUnlock`

Recovered order:

1. Create the object-scoped job and thread.
2. Wait `1s` on the simulation timeline.
3. Enumerate the players.
4. Call `UnlockNextAbility` twice for each player.
5. Delete the job.

The native registered at `sub_A05120` resolves the simulation player and
increments the signed field at `+0x1370`; values above `5` become `9`.
Build-103 `LabsPlayer` reflection field `21` maps to that offset. The Cryos
player starts at boundary `1`; the two calls advance `1 -> 2 -> 3`, exposing
Ride at HUD descriptor index `2`. This transition is live-confirmed.

Typed director interpretation: after the accepted server-only marker, schedule
one delayed ability-boundary operation and emit the two authored state
transitions in order. Ability use remains subject to ordinary server validation.

### `Tutorial_IntroOverdriveActivate`

Instruction-level decoding corrects the earlier decompiler caveat. The player
loop is genuinely empty in the original bytecode:

```text
LEN       R2 R1
LOADK     R3 0
SUB       R4 R2 1
LOADK     R5 1
FORPREP   R3 -> FORLOOP
FORLOOP   R3 -> FORLOOP
```

The complete behavior is: wait `3.75s`, call `GetPlayerIds`, execute an empty
numeric loop from `0` through `count-1`, mark the job object for deletion, and
return. There is no hidden native call, overdrive mutation, packet, or event.
This is an authored server-side no-op stub, not an incomplete decompilation.

### `Tutorial_IntroSecondCreatureUnlock`

Recovered order:

1. Wait `0.5s`.
2. Enumerate the players.
3. Call `UnlockSecondCreature`.
4. Create and notify an event whose `clientEventID` is
   `SPID("PlayerUnlockedSecondCreature")`.
5. Delete the job.

The native registered at `sub_A050C0` resolves the simulation player and
increments the field at `+0x1378`, capped at `2`. Build-103 `LabsPlayer`
reflection field `23` maps to this mission-local counter. This native does not
submit the HTTP creature-unlock request and therefore does not prove permanent
Sage ownership.

The client caches fixed squad identities from the initial snapshot. Sage must
already have valid hidden backing data, while the locked deck minimum keeps her
portrait hidden until the authored reveal. The exact relationship between
field `23`, the deck-minimum reveal, the player update, the client event, and
persistent ownership remains partially open; the Lua call order itself is
documented.

### Later server-job timelines recovered from bytecode

All five later jobs first obtain `GetPlayerIds()` and use its length, but pass
the numeric loop index `0..count-1` to player natives rather than indexing the
returned table. They begin with `areAllUnbeaten=true` and clear it if any
`HasBeatenThisLevel(index)` call succeeds. Their exact schedules are:

| Job | If every player is initially unbeaten | Always-executed tail |
| --- | --- | --- |
| `Tutorial_RandomUnlock` | wait `2s`, wait `4s`; for each still-unbeaten index call `UnlockNextAbility`; wait `6s`, then `5s` | wait `2s`; return |
| `Tutorial_SupportUnlock` | Byte-for-byte control flow equivalent to Random Unlock: wait `2s`, `4s`, unlock once per still-unbeaten index, then wait `6s`, `5s` | wait `2s`; return |
| `Tutorial_SoloSupportUnlock` | wait `2s`, wait `4s`; unlock once per still-unbeaten index; wait `2s`, then `5s` | wait `2s`; resolve the initiating player's controlled object; call `ActivateHordeSpawn(firstMainArgument, controlledObject)` |
| `Tutorial_OverdriveUnlock` | wait `3s`, wait `5s`; call `UnlockOverdrive` once per still-unbeaten index; wait `5s`, `4s`, `4s` | wait `3s`; resolve the initiating player's controlled object; call `ActivateHordeSpawn(firstMainArgument, controlledObject)` |
| `Tutorial_CatalystUnlock` | wait `2s`, wait `4s`; for each index, recheck beaten state, conditionally call `UnlockCrystals`, then call `DropCrystals` regardless of that recheck; wait `4s`, `4s`, `4s` | wait `2s`; resolve the initiating player's controlled object; call `ActivateHordeSpawn(firstMainArgument, controlledObject)` |

If any player is already beaten at the initial scan, each job skips its entire
conditional sequence and begins only the always-executed tail. Route ownership
is still required before any of these later onboarding jobs can be assigned to
the Cryos tutorial. The normalized `level_script` table currently contains zero
rows for all eight server tutorial chunks. This is an indexing limitation, not
proof that the callbacks are unused: the packaged chunk stores its module name
and `main` as separate constants, while the marker component can store the
combined callback reference. Route ownership therefore requires improved
component decoding/linking or direct marker-source inspection.

### Native inventory for chunks 735, 623, 62, 769, 222, and 655

This ledger is limited to the bytecode stored in `content.db` and the registered
native bodies in build-103 `Game.c`. **Exact** means that both sources agree
on the call shape and effect. **High** means the instruction or native body is
unambiguous but the resulting reflection/wire consumer is outside this slice.
**Medium** is reserved for a derived identity or lifecycle edge whose owning
caller is not present in these six chunks.

| Chunk | Resource identity | Authored module | Size and SHA-256 | Evidence |
| ---: | --- | --- | --- | --- |
| 735 | server data `14307`, `0x24F78AA1/0x5C4C5BBF.lua` | `nTutorial_RandomUnlock` | 1,051; `b9bf494972e98b73e60fe527bbfcaec6b66434abea4545adacbdb343c6364878` | Exact |
| 623 | server data `14187`, `0x24F78AA1/0xC9B6860B.lua` | `nTutorial_SupportUnlock` | 1,053; `2079148a63af96dd765857ae51298bc86a65a9a9ae4d7a0ca8145e204a400509` | Exact |
| 62 | server data `13584`, `0x24F78AA1/0x90CE5ECC.lua` | `nTutorial_SoloSupportUnlock` | 1,172; `b67c0aace5d108e5f6f7b2432f5d8be1aaee3f00d1204fdd56c6b03abcd878c4` | Exact |
| 769 | server data `14346`, `0x24F78AA1/0xE137EF5C.lua` | `nTutorial_OverdriveUnlock` | 1,182; `cbec31d39922671c12c67caa2482bfaa139d795d4dd4a96266b5ad235bd2fef4` | Exact |
| 222 | server data `13762`, `0x24F78AA1/0xB8716CFB.lua` | `nTutorial_CatalystUnlock` | 1,216; `13f084e59a192f55a30e8285e8ebb651bfcaa90ae0ab16f5235c3ffa28fa09db` | Exact |
| 655 | server data `14222`, `Modifiers/0x9C2B314A.lua` | `nModifier_Spawn` / `SpawnModifier` | 1,482; `0f1ff2f4dfaf35491d91b31c894061af3e3cc293fbb05f97ae656e75ef8c47f1` | Exact |

The five job entry points have the same exact wrapper signature and lifetime:
`main(firstMainArgument, initiatingObject) -> nil` creates the module's
`LuaJobObject.Noun`, then calls
`CreateThreadForObject(jobObject, worker, jobObject, firstMainArgument,
initiatingObject)`. The worker signature is therefore
`worker(jobObject, firstMainArgument, initiatingObject) -> nil`. None of these
five chunks marks or deletes its job object after the worker returns. The worker
first calls `GetPlayerIdForObject(initiatingObject) -> playerID`, then
`GetPlayerIds() -> playerIDTable`, but all loops use the numeric index
`0..#playerIDTable-1`; they never read a value from that table.

Exact checkpoints, measured from worker start, are:

| Chunk | All players unbeaten at the initial scan | Any player beaten at the initial scan | Events |
| ---: | --- | --- | --- |
| 735 | `t=2` wait completes; `t=6` wait completes and each still-unbeaten index receives `UnlockNextAbility(index)`; waits end at `t=12`, `t=17`, and tail `t=19`; return. | Skip the mutation branch; tail wait ends at `t=2`; return. | No event table, event ID, or explicit send. |
| 623 | Byte-for-byte worker control flow equivalent to chunk 735, including the `t=2/6/12/17/19` checkpoints and one unlock per still-unbeaten index. | Same `t=2` tail-only return. | No event table, event ID, or explicit send. |
| 62 | Waits end at `t=2` and `t=6`; unlock each still-unbeaten index; waits end at `t=8`, `t=13`, and tail `t=15`; resolve `GetPlayerControlledObjectID(playerID)` and call `ActivateHordeSpawn(firstMainArgument, controlledObjectID)`; return. | Tail wait ends at `t=2`, then perform the same resolve and horde call. | No event table or event ID; the final registered native is a no-op. |
| 769 | Waits end at `t=3` and `t=8`; call `UnlockOverdrive(index)` for each still-unbeaten index; waits end at `t=13`, `t=17`, `t=21`, and tail `t=24`; resolve the controlled object and make the same horde call; return. | Tail wait ends at `t=3`, then resolve and call the no-op horde native. | No event table or event ID. |
| 222 | Waits end at `t=2` and `t=6`. For every index, recheck `HasBeatenThisLevel(index)`; call `UnlockCrystals(index)` only when false, then call `DropCrystals(index)` regardless of that per-index recheck. Waits end at `t=10`, `t=14`, `t=18`, and tail `t=20`; resolve the controlled object and make the no-op horde call; return. | Skip every unlock and drop; tail wait ends at `t=2`, then resolve and call the no-op horde native. | No authored event ID. Pickup creation/launch is a native consequence, not a Lua event. |

The exact native surface and mutations used by those jobs are:

| Lua signature | Registered body | Exact native effect | Evidence |
| --- | --- | --- | --- |
| `GetPlayerIdForObject(object) -> playerID` | `sub_A04910` | Resolve argument 1 as an object and return its player byte at `+85` when it has the required player link; otherwise return the native's null/empty value. | High |
| `GetPlayerIds() -> table` | `sub_A04980` | Enumerate live simulation players and return their player bytes. The scripts use only the table length. | High |
| `HasBeatenThisLevel(playerIndex) -> bool` | `sub_A054C0` | Resolve the simulation player and compare signed field `+4700` with the current level threshold returned through `sub_9BCC50`. No mutation or send. | Exact |
| `WaitForXSeconds(seconds) -> yield` | `sub_A079F0` | Clamp negative input to zero; zero does not yield. Positive input installs a timed Lua continuation. These chunks pass only the positive constants listed above. | High |
| `UnlockNextAbility(playerIndex) -> nil` | `sub_A05120` | Resolve the player; increment reflected `mLockedAbilityMin`, field `21`, signed field `+4976` (`+0x1370`); if the result is greater than `5`, replace it with `9`. No dedicated event constructor occurs in the native. | Exact |
| `UnlockOverdrive(playerIndex) -> nil` | `sub_A05180` | Resolve the player; clear reflected `mbLockedOverdrive`, field `19`, byte `+4972` (`+0x136c`) and copy configured maximum `dword_1164AE8` into reflected `mEnergyPoints`, field `10`, at `+4668` (`+0x123c`). The compiled/default maximum is exactly `100.0`, though tuning can replace it at startup. | Exact mutation and reflection fields |
| `UnlockCrystals(playerIndex) -> nil` | `sub_A051E0` | Resolve the player and clear reflected `mbLockedCrystals`, field `20`, byte `+4973` (`+0x136d`). | Exact mutation and reflection field |
| `DropCrystals(playerIndex) -> nil` | `sub_A0BB10` | Require simulator, player, and controlled object; choose `max(currentDifficulty, configuredMinimum)`. The compiled fallback is `2`, but runtime `LabsTuning` property `0x35820747` overrides it to `4`. Loop exactly cached player count times; choose a position around the controlled object and call `sub_9CA3D0(difficulty, 1000, sourceObjectID, controlledObjectID, position)`. | Exact call shape and runtime tuning; High later create/launch boundary |
| `GetPlayerControlledObjectID(playerID) -> objectID` | `sub_A04A50` | Return resolved player field `+4664` (`+0x1238`), or `0` when resolution fails. | Exact |
| `ActivateHordeSpawn(firstMainArgument, controlledObjectID) -> nil` | `sub_A05920` | Read only argument 1, resolve it as a game object, require a simulator/director and object pointer `+100`, then call `sub_A23560(pointer)`. Argument 2 is ignored and `sub_A23560` is literally `return 0`; there is no mutation, wait, event, or GMS send. | Exact |

Chunk 655 is a modifier lifecycle, not a server job. Its exact authored
signatures and order are:

1. `Activate() -> nil`: obtain `GetMyAgentID()`, call `Stop(agent)`, then
   `AddAttributeModifier(agent, nAttributeType.Immobilized, 1)`. The returned
   modifier handle is discarded.
2. `Tick() -> nil`: obtain the agent, call
   `AddEffect(agent, generic_spawn.ServerEventDef)`, then
   `SetAnimationState(agent, horde_beam_in)`, then wait exactly `0.5s`.
3. `Deactivate() -> nil`: obtain the agent, call
   `RemoveEffect(agent, generic_spawn.ServerEventDef)`, then
   `ResetAnimationState(agent)`. It does **not** call
   `RemoveAttributeModifier`; removal of `Immobilized` must be owned by modifier
   scope cleanup or another caller and is not proved by this chunk.

The exact authored effect/event identity is the preloaded asset name
`generic_spawn.ServerEventDef`; this chunk does not construct an `nEvent` table
or a separate `clientEventID`, and the permitted evidence does not expose a
stable numeric event ID for that asset. `SetAnimationState` does case-fold and
hash its string argument through `sub_AD9B60(..., 0x811c9dc5, 1)`, giving exact
animation SPID `horde_beam_in = 0x7EE868F5`. The bytecode selects the typed
`nAttributeType.Immobilized` constant, but does not expose its numeric value.
`AddEffect`/`RemoveEffect` update one of 16 effect slots and, on the authoritative
path, use logical GMS message kind `28` (`ServerEvent`). `SetAnimationState` and
`ResetAnimationState` write the animation SPID (or zero) and a simulator
timestamp, then use logical GMS message kind `39`, the recovered 25-byte wire
opcode `0xA5`. `Stop` uses the locomotion snapshot sender, logical message kind
`18`. The authored identities, message kinds, mutations, and `0.5s` wait are
Exact; absence of a numeric event ID is an explicit evidence boundary, and the
external owner and deactivation instant remain Medium.

#### Next minimal parity fixture and required typed opcodes

The next minimal server-job parity fixture is the paired chunk-735/chunk-623
single-player unbeaten case. It proves two resource identities can drive the
same typed program without route-specific behavior: scan at `t=0`, wait to
`t=2`, wait to `t=6`, apply exactly one ability-boundary increment to player
index `0`, then resume at `t=12`, `t=17`, and `t=19`. A companion already-beaten
case must prove no mutation and a single `2s` tail wait. The fixture must not
delete the job object and must emit no tutorial event. This fixture is minimal
because it avoids crystal creation, modifier replication, and the authored
horde no-op while still testing branch selection, recheck, scheduling, and a
real player mutation.

All typed intents/opcodes required to represent the six chunks without a
stringly native escape hatch are:

| Domain | Required typed intent/opcode | Used by |
| --- | --- | --- |
| Lifecycle | `CreateJobObject(noun)`, `StartObjectThread(job, program, firstMainArgument, initiatingObject)`, `ReturnThread` | 735, 623, 62, 769, 222 |
| Scheduling | `WaitSimulation(seconds)` with object/phase cancellation | all six |
| Player queries | `ResolvePlayerForObject(object)`, `SnapshotPlayerCount`, `HasBeatenCurrentLevel(playerIndex)`, `ResolveControlledObject(playerID)` | job chunks; controlled-object lookup only 62, 769, 222 |
| Player mutations | `AdvanceAbilityBoundary(playerIndex)`, `UnlockOverdrive(playerIndex)`, `UnlockCrystals(playerIndex)`, `DropCrystals(playerIndex)` | 735/623/62, 769, 222 |
| Director compatibility | `InvokeBuild103HordeNoOp(firstMainArgument, ignoredControlledObject)` | 62, 769, 222 |
| Modifier lifecycle | `ModifierActivate`, `ModifierTick`, `ModifierDeactivate`, with the modifier instance owning cleanup scope | 655 |
| Agent mutation | `StopLocomotion(agent)`, `AddScopedAttribute(agent, Immobilized, 1)`, `AddEffect(agent, eventSPID)`, `RemoveEffect(agent, eventSPID)`, `SetAnimation(agent, animationSPID)`, `ResetAnimation(agent)` | 655 |
| Replication boundary | `LocomotionSnapshot` logical `18`; `ServerEvent` logical `28`; `AnimationState` logical `39` / wire `0xA5`; ordinary reflected player update; crystal object create/launch | 655 and the corresponding player/drop mutations |

No `NotifyTutorialEvent`, `DeleteJobObject`, real horde activation, or
`RemoveAttributeModifier` opcode is authorized by these chunks. The exact wire
opcode for the generic reflected player mutations and the crystal create/launch
messages remains outside this inventory and must not be guessed from the native
names.

The next three bytecode parity fixtures, in order, are:

1. Paired chunks `735/623`: `CALL×17`, `CLOSURE×2`, `FORLOOP×2`,
   `FORPREP×2`, `GETGLOBAL×21`, `GETTABLE×15`, `GETUPVAL×1`, `JMP×3`,
   `LEN×1`, `LOADBOOL×2`, `LOADK×13`, `MOVE×9`, `RETURN×3`,
   `SETGLOBAL×1`, `SETTABLE×3`, `SUB×2`, and `TEST×3`; bind
   `CreateJobObject`, closure-descriptor consumption, `StartObjectThread`,
   `SnapshotPlayerCount`, `HasBeatenCurrentLevel`, cancellable
   `WaitSimulation`, `AdvanceAbilityBoundary`, and `ReturnThread`.
2. Chunk `62`: `CALL×19`, `CLOSURE×2`, `FORLOOP×2`, `FORPREP×2`,
   `GETGLOBAL×23`, `GETTABLE×17`, `GETUPVAL×1`, `JMP×3`, `LEN×1`,
   `LOADBOOL×2`, `LOADK×13`, `MOVE×13`, `RETURN×3`, `SETGLOBAL×1`,
   `SETTABLE×3`, `SUB×2`, and `TEST×3`; add bindings
   `ResolvePlayerForObject`, `ResolveControlledObject`, and
   `InvokeBuild103HordeNoOp` to the paired-fixture bindings. Assert that the
   tail creates no real horde command and the job is not deleted.
3. Chunk `769`: `CALL×20`, `CLOSURE×2`, `FORLOOP×2`, `FORPREP×2`,
   `GETGLOBAL×24`, `GETTABLE×18`, `GETUPVAL×1`, `JMP×3`, `LEN×1`,
   `LOADBOOL×2`, `LOADK×14`, `MOVE×13`, `RETURN×3`, `SETGLOBAL×1`,
   `SETTABLE×3`, `SUB×2`, and `TEST×3`; reuse chunk `62`'s query, wait,
   closure, thread, return, and no-op bindings, replacing the player mutation
   with `UnlockOverdrive` (`mEnergyPoints` field `10` plus
   `mbLockedOverdrive` field `19`).

Chunk `222` follows these only after the authoritative ordering and field
selection for the required post-`ObjectCreate` `0x9a` and `0x94` component
updates are recovered; its Lua VM instructions are already supported, but
freezing a packet fixture now would turn receiver-valid shapes into an
unsupported sender claim.

### Server timeline primitives with recovered wire consumers

These primitives are not necessarily defined by the eight tutorial jobs, but
they are required by authored tutorial abilities and director sequences.

| Lua/native intent | Recovered server meaning | Known client boundary | Remaining limitation |
| --- | --- | --- | --- |
| `WaitForXSeconds` and object-thread waits | Schedule an object- and phase-scoped continuation on the simulation clock. | No packet by itself. | Cancellation must use object lifetime and phase generation. |
| `StartCinematic` | Begin a timed camera sequence around a world position and radius. | `CinematicMsgs` wire `0xC9`: opcode, signed LE `int64` duration-ms, and four LE `float32` values for XYZ/radius. | Lua rounding, RakNet send parameters, replacement, and cancellation remain open. |
| `SetIsVisible` | Change authoritative presentation visibility for a live phase-owned object. | Existing reflected object update path. | Exact ordering around cinematic and animation still needs the Quadra trace. |
| `SetAnimationState` | Select and timestamp an authored animation state. | Exact 25-byte `0xA5` receiver/encoder is recovered. | Several live animations are still invisible, so prerequisites and ordering remain open. |
| `nEvent.Notify` | Reflection-decode an allowlisted event table and send a `ServerEvent`. | Exact recipes are recovered for attached effects, position/facing effects, pickup feedback, and tutorial alerts. | Each authored event still needs its exact field allowlist and route owner. |
| `WaitForProjectile` | Suspend the authoritative ability thread until collision, invalidation, or range exhaustion. | No wait packet; projectile creation/motion and later impact/state packets are separate. | Generic projectile `ObjectCreate` and launch replication remain open. |
| `DropStuffForObject` | Queue recipient, signed selector, deadline, and processing state on the combatant. | Later creates/launches pickup objects and may emit `ServerEvent`. | Exact pickup create/launch bytes and recovery policy remain open. |
| `UnlockOverdrive` | Build-103 `sub_A05180` resolves the simulation player, clears `mbLockedOverdrive` field `19` at `+4972`, and writes the configured maximum (`100.0` in this build) to `mEnergyPoints` field `10` at `+4668`. | Exact minimal `LabsPlayerUpdate` body is `slot, 00 10, 0a, <float32 maximum>, 13, 00, ff`; no persistence request is made. | Recover the concrete route owner and initial-snapshot policy that sets the lock true. |
| `UnlockCrystals` | Build-103 `sub_A051E0` resolves the simulation player and clears `mbLockedCrystals` field `20` at `+4973`. | Exact minimal `LabsPlayerUpdate` body is `slot, 00 10, 14, 00, ff`. The HUD reads the cleared byte; no account submission is visible. | Recover the concrete route owner and initial/late-join snapshot policy. New missions must reconstruct the lock from server policy. |
| `DropCrystals` | Build-103 `sub_A0BB10` requires a live simulation player and controlled object. Its loop count is `sub_9BE350(simulator, 1)`, the simulator's cached player count, so it attempts exactly one drop per player. Each attempt chooses a position around the controlled object and calls `sub_9CA3D0(max(difficulty, 4), 1000, sourceObjectID, controlledObjectID, position)`. The `4` is runtime `LabsTuning` property `0x35820747`; the executable fallback is `2`. | Executable disassembly proves `sub_9CA3D0` is a thunk to `sub_A18600`. That path rejects input below the same configured minimum, applies `1000 * 0.15 = 150`, and compares it with uniform integer RNG `[0,100)`, so this tutorial call is guaranteed before any nonnegative attribute multiplier. It converts difficulty into four minor steps per major tier (`4 -> 14`, `5 -> 21`), consumes a weighted level-offset draw, and passes the resulting crystal level through the 192 inclusive level-bounded noun definitions. Build-103 `CrystalTuning` has exactly one offset, `+0` at weight `1.0`. It then creates the selected noun, writes the computed level to `cLootData` field `0` `crystalLevel` (`int32`, component `+8`) after its reflection baseline, and initializes lob movement toward the generated position with height `2.5` and duration `0.5s`; `planeDirLinearParam` is projected distance divided by that duration. `ObjectCreate` cannot decode either component, so required post-create messages are `kGmsLootDataUpdate` wire `0x9a` and locomotion wire `0x94`; their relative order, reliability, and exact `0x94` field selection are not sender-proven. The build-103 executable has their decoders but no message-factory constructor for logical `21` or `27`. Fresh runtime chunk `134` (`Abilities/0xD71BDB7E.lua`, SHA-256 `372fbd8cdfd921f6fd0979cbbabedb3d64a3bfa2c99501e266015cf0d5d4dbc8`) proves the caller is registered `PickUpCrystal`: an `IsInteract` ability with authored range `2`, cast time `0.3s`, animation time `0.7s`, release time `1s`, and `shouldPursue=true`. Out-of-range admission returns pursuit result `3` rather than creating the ability; collection occurs only after admission returns `1`. Its branchless callback resolves the acting agent's player and selected target, then invokes `PickupCrystal`; it performs no Lua-side range, liveness, ownership, or component recheck. Native admission calls the generic range and `IsAbleToHit` predicates once; neither is called again by this callback or the generic execution path. A target lost before release therefore makes `PickupCrystal` silently no-op. Test-only chunk `705` proves the client classifies `kType_Crystal` separately and locally checks slot capacity before `nAction.PickUpCrystal`, but this is not server authorization. On a successful slot, native `sub_A17F40` dereferences `*(target + 744) + 8` without a component guard; the full-slot branch may apply lob state to the supplied object. Typed authority must therefore require a live phase-owned crystal object with `cLootData` before either branch. Successful collection writes the first eligible mission slot, deletes the pickup, sends `0xc3`, then recomputes bonuses. With all nine slots unavailable, it retains the pickup, optionally relaunches it after the current `0.5s` lob with height `3.0` and `groundCollisionOnly=true`, and sends scoped client event `0x6ea4091e`. No account submission is visible, supporting mission-local state. | Recover the original authoritative sender's exact post-create `0x9a`/`0x94` ordering, reliability, and locomotion field selection; preserve pursuit-versus-accepted request state, requesting-player agent ownership, request-time range/hit validation, release-time missing-target no-op, and explicit crystal/cLoot allowlisting; recover where original server enforcement lived, plus the rejection event name/text and noun/phase lifetime. A safe typed encoder must zero all consumer-ignored bytes rather than copy native stack garbage. |
| crystal pickup action request | Build-103 `sub_44CAF0` selects type `9` only for a clicked target with component pointer `+744` and a non-null noun-definition field `+148`; otherwise a generic interactable uses type `11`. Its exact 20-byte tail is target local object ID, target current XYZ, signed byte `-1`, then three uninitialized padding bytes. `sub_5370F0` sends the resulting 60-byte logical `29` / wire `0x9c` body reliable ordered with literal arguments `1,3,0`; `sub_536D50` network-normalizes the target ID. | `sub_44A7C0` locally permits the component click only when component float `+64 <= 0`. Binding `nObject.IsDNALoot` and decoder telemetry identify `+64` exactly as the DNA-loot amount, so this excludes DNA loot before the crystal-specific noun predicate. `sub_4DFB40` stages the request and `sub_4DF5B0` sends it once at action start. The final four bytes are not a rank: deterministic encoding is `ff 00 00 00`. Incoming `sub_9C0F70` queues the decoded action, but no client-side vector consumer or type-`9` execution switch is recovered. | Treat only authenticated peer plus target ID as intent. Derive the deployed actor and current target position server-side; validate live phase-owned crystal/cLoot identity, range `2`, and `IsAbleToHit`, then preserve chunk `134`'s release-time missing-target no-op. Recover the original server action-to-ability mapping/rejection response; do not trust common-header actor/pose, tail XYZ, selector, or padding. |
| `ActivateHordeSpawn` | Build-103 `sub_A05920` resolves only Lua argument 1 as a game object, confirms a live simulator/director, reads the non-null pointer at object offset `+100`, and directly calls `sub_A23560(pointer)`. The scripts pass a controlled object as argument 2, but this native does not read it. | The build-103 executable body of `sub_A23560` is literally `xor al, al; ret 4`; the call has no mutation or GMS side effect. | Treat this registered native as an authored no-op in build 103. Horde progression, if used by these routes, must be owned by another marker/director path; recover that owner rather than inventing behavior for this stub. |
| director marker callbacks | Build-103 startup code registers `HordeTrigger_OnEnterPlayer -> sub_9FACB0`, `DirectorTrigger_SpawnBoss -> sub_9FACF0`, and `DirectorTrigger_SpawnSurvivorHorde -> sub_9FB0C0` through `sub_A173A0`. The first two handlers reject the remote/non-authoritative branch, require a valid player entrant, fetch the simulator/director pointer, then return false without changing director state or sending GMS. The survivor handler is different: it calls `sub_9FAEB0` to allocate a director record and sets director byte `+0x0e`. | `DirectorTrigger_SpawnBoss` and `HordeTrigger_OnEnterPlayer` are deliberate inert client registrations in this build. `HordeSpawner_Register` occurs in neither the executable string corpus nor any packaged Lua constant, despite being authored on both Cryos listener markers. | Treat the Cryos boss trigger and both listener registrations as an undocumented server-owned command path. The client names prove dispatch identity, not wave semantics. Recover the server event delivery, listener state, budgets, clear predicate, and replication contract independently. |
| `ActivateMinionSpawn` / `ActivateLieutenantSpawn` | Build-103 `sub_A059B0` and `sub_A05A30` read `(spawnPosition, npcList, shouldAggro)`. They differ only in the list selector passed to `sub_9FEBC0` (`0` for minion, `1` for lieutenant). The shared path filters the selected noun list by current difficulty, chooses one uniformly with the simulator RNG, creates one object at the supplied position, and returns its object handle to Lua. If `shouldAggro` is true, it adds `5.0` threat of type `2` (`"Spawn Aggro"`) against every live player-controlled object. | Object creation and the returned handle use the ordinary authoritative simulation path. No separate horde packet is constructed by these bindings. | Identify the exact Cryos callback or component that supplies `npcList`, prove whether the two arena listeners use this binding, and recover the resulting create/beam-in ordering. |
| director spawn mode `5` candidate budget | Build-103 `sub_9FE270` mode `5` draws repeatedly from its difficulty-eligible cached candidate pool. `sub_9FBA70` chooses candidates uniformly with replacement. For accepted candidate number `n` (zero based), the adjusted total is `(1 + 0.1*n) * (rawAcceptedCost + candidateCost)`. A candidate that would exceed the caller's budget is removed from the temporary pool and terminates selection; accepted candidates remain eligible. The result is capped at `15` nouns. | This constructs a noun list only; later object creation and presentation are separate. | Recover the caller that supplies the Cryos arena budget and prove that mode `5` is the retail boss/horde callback path before using the formula for wave composition. |
| `IsHordeActive` | Build-103 `sub_A05AB0` returns simulator/director byte `+0x48c` (`+1164`). IDA resolves its reflected retail field name as `mbHordeSpawned`. The adjacent reflected `mActiveHordeWaves` container begins at `+0x47c`. A complete executable displacement scan finds only their two constructor initializations plus the getter; no local gameplay writer or active-wave consumer exists. | The getter itself emits no packet. Because both fields are registered reflection properties, generic server-state decoding remains a plausible way for the client to receive them, but no concrete update route is yet proven. Two packaged predicate modifiers consult `IsHordeActive`. | Recover the server replication/update that populates these reflected fields, or establish that they remain dormant in build 103. Do not use the byte as the authoritative arena-clear predicate until its server writer and lifetime are known. |
| `SpawnModifier` / `horde_beam_in` | Packaged chunk `Modifiers/0x9C2B314A.lua` (resource `14222`, SHA-256 `0f1ff2f4dfaf35491d91b31c894061af3e3cc293fbb05f97ae656e75ef8c47f1`) defines the exact spawn presentation. Activation stops locomotion and adds `Immobilized=1`. Its tick adds `generic_spawn.ServerEventDef`, sets animation `horde_beam_in`, and waits `0.5s`. Deactivation removes the effect and resets animation. | Uses the already recovered effect and animation replication paths; the wait itself emits nothing. | Recover who attaches/deactivates this modifier, the ordering relative to `ObjectCreate`, and whether scoped modifier cleanup removes the immobilization or a separate native does so. |
| HP/mana mutation | Validate pickup overlap and update authoritative resources before presentation/deletion. | `CombatantDataUpdate` `0x97`, then pickup/full `ServerEvent`, then `ObjectDelete`. | Active-hero versus squad-wide ownership needs a post-Sage retail capture. |
| per-kill XP snapshot | Emit cumulative account XP/level after an authoritative award. | Wire `0xA1`, slot plus reflected Labs player fields `15` and `16`. | Two horde awards and persistence checkpoints remain unmapped. |
| tutorial completion snapshot | Supply positive cumulative XP and advance in-memory onboarding to `3000`. | `TutorialGameMsgs` `0xC8`, subtype `0`, signed LE `int32` cumulative XP. | Sender, reliability, persistence, teardown, and return-to-ship ordering are open. |

The complete 1,029-chunk bytecode corpus was extracted through the indexed
`content.db` `bget` path into `bin/game/logs/lua-corpus`. Only chunk
`Modifiers/0x9105F887.lua` (resource `13894`, SHA-256
`071de2f42edfe21e3a94d8a4d1bc9bd333f4c1a20d1e0ce24417d1104910e225`)
calls `ActivateMinionSpawn` or `ActivateLieutenantSpawn`. It defines
`nModifier_EnemyPortal_Passive`: after a `2.1s` startup effect and an additional
`8s` delay, it removes dead children and repeatedly spawns up to a configurable
local cap. Its defaults are a random `1..2` minions, zero lieutenants, and four
local children; every native spawn call requests immediate aggro. The modifier
accepts an `npcList` override, so it is a plausible reusable spawner primitive,
but neither `DirectorTrigger_SpawnBoss` nor `HordeSpawner_Register` occurs in
any packaged Lua constant. Consequently this call site is not yet proof that
the Cryos marker callbacks attach this modifier.

Build-103 IDA registration closes the client half of that callback gap. The
executable does contain and register `DirectorTrigger_SpawnBoss`, but its
handler validates the entrant, obtains the director pointer, and deliberately
returns without spawning or mutating anything. `HordeSpawner_Register` is
absent from the executable as well as the Lua corpus. The two authored listener
records therefore describe a server-owned fan-out that this client can name
but cannot execute locally. The director's reflected `mbHordeSpawned` and
`mActiveHordeWaves` fields likewise have no native gameplay writes in the
client executable; only initialization and the Lua getter are present.

The recovered `FirstAggro_SpecialOne` ability is the sequence-parity fixture for
the typed server timeline. It starts a `4.291667s` cinematic, hides its agent,
waits `2s`, shows it, the shared template repeats the idempotent visible state,
plays `character_teleport_in`, waits `1.291667s`, waits a
final `1s`, and deactivates. A restart, object death, or phase transition must
cancel later steps.

## Recovered later unlock route ownership

Indexed level-script and marker linkage resolves all five formerly ambiguous
server jobs. None participates in the Cryos tutorial route:

| Job | Recovered owner |
| --- | --- |
| `Tutorial_RandomUnlock` | Level `56`, `zelems_1`; once-only radius-`20` enter marker row `126382`, authored ID `164537526`. |
| `Tutorial_SupportUnlock` | Level `60`, `zelems_3`; once-only radius-`20` enter marker row `131724`, authored ID `2750774706`. |
| `Tutorial_SoloSupportUnlock` | Level `56`, `zelems_1`; object-lifetime listener marker row `126389`, authored ID `223774364`, on `boss triggered`. |
| `Tutorial_OverdriveUnlock` | Level `50`, `verdanth_1`; object-lifetime listener marker row `99848`, authored ID `409195075`, on `boss triggered`. |
| `Tutorial_CatalystUnlock` | Level `23`, `nocturna_4`; object-lifetime listener marker row `68601`, authored ID `2147048951`, on `lvl 4 boss arena`. |

The listener-owned rows have no inferred contact radius. Their trailing
`ActivateHordeSpawn` calls remain the separately proven build-103 no-op and do
not move horde ownership into these jobs.

## Server-driven behavior not yet documented

These are the primary reverse-engineering targets. Until each row is resolved,
its name is evidence of intent but not permission to invent state changes.

| Priority | Server job or boundary | What is known | What must be recovered |
| ---: | --- | --- | --- |
| 1 | Sage/Quadra introduction | `Tutorial_IntroSecondCreatureUnlock` order and `FirstAggro_SpecialOne` timeline are known independently. | Recover their owning director/marker ordering: Sage reveal, swap lesson, control/camera capture, Quadra create/focus, cinematic, visibility, animation, objective state, control return, and combat activation. |
| 3 | Crystal wire construction | `UnlockCrystals`, one guaranteed drop attempt per player using runtime minimum difficulty `4`, the `sub_9CA3D0 -> sub_A18600` thunk, weighted subtype selection, object creation, launch initialization, successful slot assignment, world-object deletion, bonus recomputation, and the fixed 29-byte `kGmsCrystalMessage` / wire `0xC3` slot update are build-103 confirmed. Pickup component hash `0x2adb076a` is exact `cLootData`; subtype is field `0` `crystalLevel`, and wire `0x9a` is its required receiver family. Lob field inventory and coefficient construction are exact, and wire `0x94` is the required locomotion family. `ObjectCreate` accepts neither component reflection. Client dispatch, deferred HUD queueing, acquisition subtype `0`, accepted move subtype `2`, rejected move subtype `3`, consumed offsets, and ignored bytes are exact. | The client-shaped executable contains no logical-`21` or logical-`27` constructor. Recover the original authoritative sender's exact post-create `0x9a`/`0x94` changed-field selection, ordering, and reliability from a server capture/source, then verify mission-only lifetime across teardown/late join. Encode ignored bytes as zero; never fixture uninitialized native stack contents. |
| 4 | Horde director fidelity | `ActivateHordeSpawn` ignores argument 2 and terminates in a literal no-op stub. Cryos instead has radius-30 director marker `1730050752`; its normalized callback is `DirectorTrigger_SpawnBoss` on event `horde triggered02`, consumed by exactly two `HordeSpawner_Register` listeners. IDA now proves the build-103 client registers `DirectorTrigger_SpawnBoss -> sub_9FACF0`, but that handler only validates the entrant and fetches the director pointer before returning false with no state change. `HordeSpawner_Register` exists in neither the executable nor packaged Lua. Arena teleport lands inside the director, and the first wave begins after the authored `2s` delay. The exact level payload has one ordinary Poison director noun plus five first-visit nouns; all six records are legal from difficulty `1` through `100` and have their horde-legal flag set. Build 103 also exposes real minion/lieutenant creation bindings and a mode-`5` budget selector with a `0.1` per-accepted-unit cost multiplier and a 15-noun cap, but their ownership by the server callbacks is not yet proven. The packaged spawn modifier fixes beam-in presentation at `0.5s` while immobilized. Four waves and terminal alert behavior are live-confirmed in darkspin. | The client bridge is recovered and intentionally inert; reconstruct the missing server callback implementation and caller-supplied retail budget, then derive exact per-wave composition/counts. Recover event fan-out to both listeners, `1.5s` inter-wave timing authority, creation/modifier ordering, aggro behavior after beam-in, cancellation, late-join reconstruction, replication of reflected `mbHordeSpawned`/`mActiveHordeWaves`, and the final exit transition. Do not assign these effects to either client no-op. |
| 5 | Tutorial success and Beam Out | The client accepts Beam Out status/director messages and `AF 00` reaches a scene-ready/UI-ready ship. Positive `0xC8` mutates the in-memory tutorial/account snapshot, but stacking it before `AF 00` is crash-proven because active gameplay treats both as state changes. | Trace the original sender-side constructor or generic broadcaster, RakNet reliability/channel, persistence write, departure, teardown, account refresh, and collection-room arrival without assuming the two transitions may be paired. |
| 6 | Game over and restart | The tutorial must have an authored failure overlay/audio and a restart path, but the relevant primitives are absent from the recovered named chunks. | Identify squad-death predicate, failure message/objective/director packets, restart-button request, server acknowledgement, teardown, phase-job cancellation, and complete clean-session reconstruction. |
| 7 | Projectile and dropped-pickup replication | Authority, collision waits, drop selectors, effects, damage, and deletion order are substantially known. | Recover the generic projectile and pickup `ObjectCreate`, launch/motion updates, target/collision fields, reliability, expiry, and late-join state. |

## Instruction-level workflow

For every undocumented server row:

1. Select the exact `content.db` resource by group and synthetic name and record
   its content-source resource ID, decoded size, and bytecode hash.
2. Decode the original Lua 5.1 instructions. Do not treat generated source as
   authoritative where it loses a loop body, call, jump, or table assignment.
3. Map constants, upvalues, registers, jump targets, closures, varargs, and
   native call arguments into a compact instruction ledger.
4. Resolve each native name through build-103 registration in IDA, then trace
   the implementation to concrete simulation fields, validation branches,
   scheduler operations, and GMS senders.
5. Correlate the job with its level marker, `serverOnly` flag, director, object
   role, prerequisites, once/repeat policy, and route phase.
6. Record the exact client-visible consequence separately from the server-local
   mutation. A local wait, collision predicate, or horde state change is not a
   packet merely because Lua initiated it.
7. Promote the result only after bytecode plus native evidence agree. Use a
   focused live trace when packet order, timing, reliability, or UI behavior is
   observable only at runtime.

Generated instruction and IDA diagnostics belong under `bin/game/logs`, with
automated IDA runs using `bin/game/ida_user` as `IDAUSR`.

## Immediate homework order

1. Recover the complete Sage/Quadra director ordering using the second-creature
   job, `FirstAggro_SpecialOne`, `0xC9`, visibility, animation, objective, and
   object-create consumers.
2. Finish the known Cryos director at marker `1730050752`: recover its retail
   random budget, exact wave composition/counts, aggro/cancellation policy, and
   final exit transition. The registered `ActivateHordeSpawn` native is a
   build-103 no-op and is not this director.
3. Trace the original authoritative publication of the created crystal pickup:
   `ObjectCreate` must be followed by separate `0x9a` and `0x94` component
   updates. Their receiver-valid sparse encodings are now byte-covered, including
   the complete 84-byte nested lob image, but order, reliability, movement type
   `4`/orientation snapshot ownership, and late-join reconstruction require a
   server capture/source because this executable contains only their decoders.
   Then finish full-slot feedback/relaunch and phase teardown.
4. Locate the owning level marker/director for every later unlock job; do not
   assign them to Cryos from their names alone.
5. Finish `0xC8` sender/reliability/teardown research, then recover game-over
   and restart as separate terminal state-machine paths.

## Related evidence

- `notes/tutorial/overview.md` contains the complete route, bytecode summaries, native
  addresses, live observations, and protocol/persistence discussion.
- `notes/tutorial/history.md` tracks implementation and live-verification gaps.
- `notes/design/architecture/raknet-gameplay-exchange.md` contains the build-103 GMS
  layouts and transport evidence.
- `notes/content/inventory.md` documents the `content.db` bytecode inventory and mapped
  research representations.
- `notes/design/architecture/research-roadmap.md` contains the
  wider IDA and evidence queue.
