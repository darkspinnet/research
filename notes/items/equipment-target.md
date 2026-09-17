# Equipment pickup targetability

Catalysts intentionally use a different type-9 contract; see
`notes/items/catalyst-target.md`.

## Result

Build 103 has two different client-side pickup command paths. A loot object with
`cLootData` whose noun passes the crystal predicate stages type 9. Other loot
stages type 11 only when the object exposes an interactable ability. Darkspin's
equipment drop sends ObjectCreate (`0x8c`), full `cLootData` (`0x9a`), the spawn
effect, and lob locomotion (`0x94`), but sends no `cInteractableData` update
(`0x98`). That omission is the smallest packet/component mismatch supported by
the recovered client contracts.

The smallest evidence-backed **equipment** fix to test is therefore one
`InteractableDataUpdateMessage` for each equipment container, after ObjectCreate
and before it can be selected:

| Reflected field | Proposed value |
| --- | ---: |
| `mNumTimesUsed` | `0` |
| `mNumUsesAllowed` | `1` |
| `mInteractableAbility` | `SPID("PickUpLoot") = 0x546c9523` |

On darkspin's encoder this is packet ID `0x98`, then object ID, bitmap `07`,
little-endian `0`, `1`, and `23 95 6c 54`. This is a packet/component proposal,
not an implementation recommendation beyond that minimum. In particular, the
evidence does not support changing combat targetability, ObjectCreate's mask,
the full equipment `cLootData`, or lob fields first.

There is an important precondition: build 103's `0x98` receiver only decodes
into an already-present `cInteractableData` at object offset `+752`. A retail
capture or a breakpoint must confirm that all four equipment rarity nouns
instantiate that component. If a noun does not, a `0x98` update is ignored and
the packet alone cannot repair it. This is why confidence that the packet is the
smallest mismatch is high, but confidence that it fixes every rarity without a
noun change is only moderate.

The same change is **not proven for catalysts/crystals**. Their normal command
is type 9, selected from `cLootData` plus a noun-definition predicate, and the
current sparse crystal update supplies the one explicitly changed field. The
intermittent missing type-9 report still needs an action-state/reachability
capture. It should not be folded into the equipment type-11 fix without that
evidence.

## Current darkspin wire and admission

The current send path is exact in `server/campaign_loot.go:122-157` and its
encoders are in `server/raknet/application.go`:

| Concern | Current behavior | Contract comparison |
| --- | --- | --- |
| ObjectCreate | `EnemyObjectCreateMessage`, noun selected by rarity, source position, scale 1, collidable | Sufficient to construct the noun object, but ObjectCreate cannot carry `cLootData`, `cInteractableData`, or lob state. |
| Equipment `cLootData` | `0x9a`, bitmap `ff 03`, all ten build-103 fields; item ID is the object ID, instance ID and DNA are zero | Matches the recovered ten-field build-103 reflection layout. DNA zero passes the loot click gate. |
| Catalyst/crystal `cLootData` | `0x9a`, bitmap field 0 only, `crystalLevel` | Matches the native pattern of retaining construction baselines and updating crystal level. |
| Lob | `0x94`, reflected fields 0-2, with the complete 84-byte lob record | Receiver-valid. The server stores the lob destination as the pickup position. |
| Interactable | No `0x98` for campaign equipment or catalysts | Equipment's non-crystal type-11 branch requires an ability; this is the identified mismatch. |
| Server admission | Type 9 catalyst commands are normalized to type 11; type 11 is accepted when the actor is the deployed object, the pickup is live and unscheduled, and center distance to the stored destination is at most 2 | This code runs only after a client packet arrives. It cannot explain an observed animation followed by no inbound type 9/11. A rejected inbound action is a separate trace case. |

The command normalization is at `server/gameplay_udp.go:33`; equipment admission
is at `server/gameplay_udp.go:6695-6758`. The authored `PickUpLoot` range is also
2, but the recovered evidence does not yet establish whether the retail client
expands that range by noun footprint. Darkspin's strict center-to-destination
check should therefore be retained as a separate admission question rather
than used to explain a command that was never sent.

## Exact client evidence

All line references below are to the canonical
`bin/game/GameBin/Game.c`.

1. **Click eligibility (`sub_44A7C0`, line 196663).** The function resolves the
   object. If `object+744` (`cLootData`) exists, it returns eligible when the
   float at component `+64` (`mDNAAmount`) is `<= 0`. Only objects without
   `cLootData` use the generic `sub_9D8A40(object) && sub_9D8A70(object)` test.
   Equipment and crystals both have `cLootData`; equipment's DNA is explicitly
   zero. This rules out combat `IsTargetable` as the manual loot-click gate.

2. **Command selection (`sub_44CAF0`, line 198612).** After resolving the target
   and local player, an object with `cLootData` whose noun passes
   `sub_9D9770(object[1])` stages the target/current target position/selector
   form used by type 9. Otherwise it stages type 11 only if
   `sub_9D8A40(object)` returns nonzero. Thus having `cLootData` makes an object
   clickable, but does not by itself give ordinary equipment a sendable
   type-11 action.

3. **Ability lookup (`sub_9D8A40`, line 1389016).** It first asks the noun for
   an intrinsic interactable ability through `sub_A190A0`. If that returns zero,
   it reads `object+752` and returns `cInteractableData+20`, the reflected
   interactable ability. Darkspin does not currently transmit this component
   for equipment.

4. **Use admission (`sub_9D8A70`, line 1389033).** A noun-special predicate can
   admit the object; otherwise the component path requires
   `mNumTimesUsed < mNumUsesAllowed` at `cInteractableData+8/+12`. The proposed
   `0 < 1` is the minimum one-use state and is consistent with a pickup.

5. **Manual click dispatch (`sub_44CFA0`, line 198901).** A non-combat target
   passing `sub_44A7C0` is handed to `sub_44CAF0`; failure goes to ground
   movement. The visible pickup animation is therefore not proof that
   `sub_44CAF0` successfully staged a network command.

6. **`0x98` receiver (`sub_539B70`, line 380598).** It reads the object ID,
   resolves the object, fetches `object+752`, and reflection-decodes only when
   that pointer is nonzero. It does not allocate the component. This is the
   pre-existing-component caveat on the proposed packet.

7. **Native equipment construction (`sub_9C98C0`, line 1376863).** It constructs
   the rarity container noun, requires `object+744`, copies the 40-byte item
   into `cLootData+16`, and writes the instance ID at `+56/+60`. It performs no
   explicit `cInteractableData` writes. The noun may still create intrinsic or
   component baseline state; only live inspection can decide that.

Darkspin's corresponding encoder facts are independently visible at
`server/raknet/application.go:793-877`: `cInteractableData` has exactly the three
fields above and bitmap `07`; equipment `cLootData` uses `ff 03`; crystal data
uses only field-zero bitmap `00` followed by the reflection terminator.

## `content.db` and noun/role evidence

The authoritative runtime database was queried with the Darkspinner config,
not the older loose Lua corpus.

- `darkrun db lua_chunk get 610 --config bin/darkspinner/darkspin.toml` resolves
  `Abilities/0x238E6F16.lua`, SHA-256
  `4ac582bd4a70c229f248c21bfde74a5d252b9d40cc72fcb4fad0d99932a38fd4`.
  Its recovered `nAbility_PickUpLoot` definition is an interaction,
  requires an agent, should pursue, uses the pickup animation, has cast time
  `0.1`, animation/release time `0.4`, and range 2 at every rank. Release first
  checks that the target remains valid and undeleted, then rolls loot and marks
  it for deletion.

- `darkrun db lua_chunk get 705 --config bin/darkspinner/darkspin.toml` resolves
  `LuaTestScripts/0x89404DF4.lua`, decoded size 18,845, SHA-256
  `7a1625acaf4229acfbea77fef0e5e6bd6be8bb0afd2b9357866731c21e4c26a7`.
  Prototype 0.7 scans sorted objects within 20 and requires
  `nLocomotion.FindGoodMeleePosition(actor, candidate)` before classifying the
  candidate. A `kType_Loot` object is accepted as `ClosestLoot` unless its asset
  is `DNA.Noun`. Prototype 0.15 directly calls
  `nAction.InteractWithObject(ClosestLoot)`. Unlike generic
  `ClosestInteractable`, the loot selection step does not call
  `HasInteractableUsesLeft`. This means noun role/type and navigable melee
  placement govern automatic discovery, while the later interaction-to-command
  conversion still encounters the client gates above.

- `darkrun db crystal_definition get 1 --config
  bin/darkspinner/darkspin.toml` identifies
  `crystal_attackspeed.Noun` (minimum 11, maximum 1000, weight 100). The asset
  catalog resource (`content_source_resource` ID 12308, package 1 ordinal
  12307, type `0x2699c284`) was extracted to
  `bin/game/logs/equipment-target/asset-catalog.bin`; decoded size 1,502,037,
  SHA-256
  `a1105c8870c30b29ff264d14af47396d804efab95327d1ce7bc4304d8755f2a5`.
  Its strings tag all four equipment container nouns as `obstacle`; the
  attack-speed crystal is tagged `crystal` and `red`. Those roles support the
  distinct equipment/type-11 and crystal/type-9 paths; they do not establish
  exact collision dimensions.

- The current `noun_physics` projection contains 13 rows for heroes, enemies,
  orbs, and a projectile, but no equipment container or crystal row. Therefore
  exact loot bounds, footprint radius, collision role, and melee-position
  expansion cannot be obtained from the projection. Absence from this derived
  table is not evidence that the packaged nouns have no physics.

The source-name hashes and complete resource digests above make the database
evidence repeatable.

## Why the other candidate changes are not first

- **Agent blackboard `IsTargetable`:** combat targeting reads it, but the
  recovered loot click and command-selection functions do not. Adding a combat
  component would be a larger, unsupported change.
- **ObjectCreate mask/fields:** the client visibly constructs and animates the
  object, and the missing state belongs to a separately reflected component.
- **Full versus sparse `cLootData`:** equipment already sends all ten build-103
  fields. Crystals deliberately send field zero while retaining noun baselines.
- **Lob duration or destination:** these can affect pursuit and automatic
  reachability, especially because the equipment destination is chosen one
  unit toward the player and the noun is tagged `obstacle`. They do not supply
  the missing type-11 ability and lack exact noun-footprint evidence.
- **Object reflection fields 21/22:** those fields represent authored
  interactable state/source marker. The direct command branch reads the
  interactable ability and use counts, so adding object state is not part of
  the minimum demonstrated contract.
- **Server pickup distance:** it can reject an action after receipt but cannot
  suppress the client's outgoing action. Diagnose it only in traces where a
  type 9/11 packet is present.

## Confidence

| Finding | Confidence | Basis |
| --- | --- | --- |
| Equipment's fallback command is type 11 and requires a nonzero interactable ability | Very high | Direct `sub_44CAF0` and `sub_9D8A40` control flow. |
| Darkspin currently omits the equipment `cInteractableData` update | Very high | Complete campaign equipment marshal path and packet encoders. |
| One `0x98` update with `0, 1, PickUpLoot` is the smallest evidence-backed server delta | High | Exact reflected fields plus the type-11 gate; no other current packet carries them. |
| The update alone works for every rarity noun | Moderate | `0x98` requires a component allocated by noun construction; this has not been observed live for all four nouns. |
| Catalyst no-send has the same cause | Low / not established | Catalyst takes the cLoot/noun type-9 branch; intermittent action-state and reachability evidence is missing. |
| Noun collision/placement contributes to automatic `ClosestLoot` misses | Moderate as a hypothesis | Automated Lua requires a good melee position and equipment is tagged `obstacle`, but exact packaged physics is not projected. |

## Remaining capture gaps

1. Capture a clean retail build-103 equipment drop and record `0x8c`, `0x98`,
   `0x9a`, and `0x94` ordering, RakNet reliability/channel, reflection bitmaps,
   and whether object fields 21/22 are also updated.
2. On a darkspin equipment object of each rarity, break at `sub_539B70`,
   `sub_9D8A40`, `sub_9D8A70`, `sub_44CAF0`, the action staging function, and
   the final sender. Record `object+752`, ability, use counts, intrinsic noun
   ability, chosen type-9/type-11 branch, and whether staging reaches send.
3. Repeat the same trace for representative catalysts. Record the result of
   the noun predicate used by the type-9 branch and distinguish target
   selection, pursuit/action start, cancellation, and actual packet send.
4. Capture both manual click and automated `ClosestLoot` attempts before,
   during, and after lob completion. Record client object transform,
   `FindGoodMeleePosition`, hero transform, queued action state, and overlapping
   loot.
5. Extract or inspect the real build-103 noun resources of type `0x76a8f7d8`
   for all equipment containers and representative crystals. Recover role,
   bounds, footprint, collision flags, intrinsic interaction ability, and
   whether construction allocates `cInteractableData`.
6. Correlate a server protocol trace with the client breakpoints. A visible
   animation with no type 9/11 must be distinguished from a packet received and
   silently rejected by darkspin's actor/duplicate/distance admission.

No server behavior was changed as part of this investigation.
