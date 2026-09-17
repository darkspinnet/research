# Build-103 concrete campaign pickups

## Scope and evidence standard

This note continues `campaign-drops.md` after the `Orb`, `Crystal`, and
`Loot` selectors succeed. It covers concrete world-object creation, component
state, launch, collection, recipient selection, capacity, and persistence. It
does not replace that note's selector and probability derivation.

Evidence labels used below are:

- **Exact**: packaged Lua bytecode or an instruction-complete native path;
- **High**: decompiler and content evidence agree, with only symbol names or
  the absent original sender unresolved;
- **Receiver-valid**: build 103 proves the client accepts the packet shape,
  but the authoritative server's changed-field mask, ordering, or reliability
  is absent;
- **Fallback**: recommended server policy, not recovered retail behavior.

The authoritative content source inspected here is the Darkspinner runtime
`content.db`, recipe `35`, source build `103`. The focused equipment pickup
chunk was extracted as
`bin/game/logs/campaign-pickup-loot-14174.luac`, SHA-256
`4ac582bd4a70c229f248c21bfde74a5d252b9d40cc72fcb4fad0d99932a38fd4`.

## Result

| Category | Concrete noun/item | World component | Collection | Durable result |
| --- | --- | --- | --- | --- |
| Orb | `HealthOrb.Noun` or `manaorb.Noun`, selected by roster-average resource need after the scaled budget accepts an attempt | No `cLootData`; ordinary networked object plus locomotion and the matching drop `ServerEvent` | Server-owned walk-over trigger; the client `Orb_Pickup` body is a no-op | Mission resource mutation only; dropped nouns expire after authored `30s` |
| Crystal | Weighted, inclusive level-bounded `crystal_definition.noun_reference`; the chosen crystal level is also written to `cLootData.crystalLevel` | Sparse `cLootData` field `0`, then lob locomotion | Authored range-`2` `PickUpCrystal` interaction; first free one of nine mission slots wins | Mission crystal slot and bonus recomputation; no account submission is present |
| Equipment loot | A generated rigblock, rarity, suffix, zero-to-two prefixes, and item level; rarity selects one of four pickup-container nouns | Full ten-field `cLootData` item image, including item ID and rigblock/affix/level/rarity fields | Authored range-`2` `PickUpLoot` interaction calls `PlayersRollForLoot`, then marks the pickup for deletion | Original server persistence is absent from the client; ship reload later obtains owned parts through HTTP `getPartList` |

The activating agent passed to `DropStuffForObject` is selection and chance
context. It is not copied into an exclusive world-owner field. Crystal and
equipment creation likewise do not set `sporelabsObject.ownerId`. A zero
equipment `mLootInstanceId` invokes the multiplayer roll path at collection
instead of binding the item to the obelisk activator. Confidence: **Exact** for
the relevant writes and branches; **High** for the absence of an exclusive
owner in the examined creation path.

## Selector handoff and count resolution

`campaign-drops.md` remains authoritative for the selector entry:

- `0x02` (`Orb`) treats the source amount as a difficulty-scaled budget. Each
  complete `100` guarantees one selection and the remainder is the percentage
  chance of one more. Challenge `100` produces one orb at unit scale. The
  difficulty-indexed scale itself remains unnamed.
- `0x04` (`Crystal`) performs one attempt per cached simulator player.
  Challenge `500` gives the base threshold `500 * 0.15 = 75`; activating-agent
  attribute `65` can multiply it.
- `0x08` (`Loot`) uses
  `cachedPlayerCount * sourceAmount * lootScalar * 0.01`, multiplied by
  activating-agent attribute `71` when present. Build-103 tuning overrides the
  compiled `0.25` scalar with `0.45`; a one-player challenge-`500` ordinary
  attempt therefore has operand `2.25` and beats every `[0,1)` draw, subject
  to the global gate and downstream item eligibility.

The `Loot|Crystal` obelisk branches are independent. One successful equipment
attempt neither consumes nor guarantees the crystal attempt. No selector
challenge chooses a fixed noun, fixed item, owner, destination, or lob.

## Concrete noun and item creation

### Orbs

For every accepted orb selection, `sub_9CE140` calls `sub_9D6A30` with the
selected noun and the source position. It then emits the matching
`health_orb_drop.ServerEventDef` or `mana_orb_drop.ServerEventDef` at that
position and initializes the shared lob. The native order is therefore object
construction, drop-event emission, and lob initialization. Network flush order
is not proven.

Runtime `noun_physics` rows reconcile the world contract:

| Noun | Lifetime | Authored bounds | Role |
| --- | ---: | --- | --- |
| `HealthOrb.Noun` | `30s` | `(-0.5,-0.5,-0.5)` to `(0.5,0.5,2.5)` | dropped health pickup |
| `manaorb.Noun` | `30s` | same | dropped power pickup |

Both dropped variants use the authored `2x2x4` pickup box after game-object
dimension substitution, are networked non-combatants with locomotion type `6`,
and install the non-server-only `Orb_Pickup` overlap trigger. The similarly
named placed tutorial variants have lifetime zero and a `1x1x1` pickup box;
they are not the drop nouns. Confidence: **Exact content/native**.

### Crystals

`sub_A184E0` first applies the chance gate, converts current difficulty to a
crystal level, applies the weighted level offset, and chooses a weighted
definition whose inclusive minimum/maximum range contains that level.
`content.db` contains 192 `crystal_definition` rows and one
`crystal_level_offset` row (`offset=0`, `weight=1`). The first family illustrates
the authored weighting: normal/rare/epic attack-speed nouns carry weights
`100/20/4`; all eligible rows participate.

`sub_A179E0` creates the chosen noun at the supplied source position. It
requires the object's `cLootData` component, writes the selected 16-bit value
into the component's reflected `int32 crystalLevel` field, and initializes the
lob. If the noun fails to create or lacks `cLootData`, the new object is
deleted and the attempt returns no pickup. Confidence: **Exact**.

### Equipment

`sub_9CDF40 -> sub_9CD990` constructs the item before a world object exists.
The generator resolves difficulty, rarity, eligible rigblock, suffix, and up
to two distinct prefixes, then validates the resulting item record. The
runtime projections contain 2,288 `loot_rigblock` rows, 666 `loot_affix` rows,
and one `loot_tuning` row. The tuning row includes rarity step `5`, level bands,
rarity topology, item point costs, and slot/stat scales. These tables are the
catalog and post-selection tuning inputs; they do not turn challenge `500`
into a named rigblock.

`sub_9C98C0` then performs these exact steps:

1. Emit `loot_spawn.ServerEventDef` at the source position.
2. Select a pickup-container noun from item rarity:
   common `loot_container_white.Noun`, uncommon
   `loot_container_green.Noun`, rarified `Loot_Drop.Noun`, or purified
   `loot_container_purple.Noun`.
3. Create that noun at the source position.
4. Copy the 40-byte item image into `cLootData` at component offset `+16`.
5. Write the two-word `mLootInstanceId` supplied by the caller.

The ordinary campaign caller supplies zero for both `mLootInstanceId` words.
The copied item image supplies `mLootItem.id`, `rigblockAsset`, suffix,
prefixes, item level, and rarity. No Electro Claws or other fixed rigblock is
present in the obelisk ability. Confidence: **Exact** for control flow and
rarity-to-container mapping; **High** for semantic item-field names, which are
independently fixed by `cLootData` reflection.

## Source-centered placement and lob

All three branches use the same sampler initialized by `sub_9CA360` around the
source object's current ground position:

- initial radius is half the source extent, clamped to at least `1.0`;
- each angular candidate adds a random radial component up to `0.625`;
- `sub_9CBDE0` snaps a candidate to the navigation surface and accepts it only
  when the planar snap error is below `1.0`;
- after exhausting a ring, the radius grows by `2.5`, for at most four rings;
- failure returns the exact source position.

Every pickup is created at the source position and launched toward the next
sampled navmesh destination. Orb, crystal, and equipment all call
`sub_A2EF00` with height `2.5`, duration `0.5s`, zero bounce parameters, and
the normal ground-collision setting. The initializer records authoritative
simulation-millisecond start time, zero previous speed modifier, and the full
84-byte `cLobParams` image. It also switches object movement type to `4` and
recomputes orientation. Confidence: **Exact** for constants and initializer;
**High** for the sampler; the field used for the source extent has no recovered
symbolic name.

## Object-create, loot-component, event, and lob packets

Build 103 has no pickup-specific create packet. Every category uses generic
`kGmsObjectCreate` (`0x8c`). The client requires object creation before a
component update can resolve its target ID.

| Publication | Receiver-proven application shape | Evidence boundary |
| --- | --- | --- |
| Generic pickup create | `0x8c`, object ID, `cGameObjectCreateData`, then `sporelabsObject` reflection. Noun and source position are required; defaults are rotation zero, asset zero, scale one, team zero, collision true, not player-controlled. | The ordinary 81-byte fields-`6/7` position/orientation envelope is receiver-valid. The original sender's exact pickup field mask and reliability are absent. |
| Crystal loot component | `9a <object:u32> 00 <crystalLevel:i32> ff` (11 bytes including opcode). | Exact legal sparse delta; batching/order is not sender-proven. |
| Equipment loot component | `0x9a`, object ID, mask `0x03ff`, then all ten fields: crystal level; item ID; rigblock, suffix, two prefixes; item level; rarity; instance ID; DNA amount (55 bytes including opcode). | Exact receiver reflection. A full initial component image is required semantically; original sender selection is absent. |
| Orb/equipment spawn event | `0x9b` with field `6` asset and field `10` position, terminated by `0xff` (20 bytes). | Exact event fields and native emission. Packet flush order remains absent. |
| Lob locomotion | `0x94` fields `0`, `1`, and `2`: start milliseconds, previous-speed modifier zero, and complete 84-byte lob image (105 bytes including opcode). | Receiver-valid for exactly the fields dirtied by the initializer; the original sender is absent. |

`ObjectCreate` cannot carry either `cLootData` or `cLocomotionData`; those are
separate `0x9a` and `0x94` messages. A complete client xref audit found no
authoritative factory call for logical object-create, loot-update, or
locomotion-update replication. Consequently a claim such as exact
`create -> loot -> lob` datagram order, channel, or batching would exceed the
available evidence even though create must precede a target-resolving component
on the receiver.

## Collection and recipient rules

### Equipment is an interact ability, then a multiplayer roll

Packaged chunk `610`, `Abilities/0x238E6F16.lua`, defines
`nAbility_PickUpLoot`. It is an `IsInteract` ability with
`requiresAgent=true`, `shouldPursue=true`, zero cooldown/mana cost, no global
cooldown, animation `pickup`, cast time `0.1s`, animation/release time `0.4s`,
and rank ranges all equal to `2`. Automated client chunk `705` selects
`ClosestLoot` and calls `nAction.InteractWithObject`, so the input is generic
type-`11` `ActionUseInteractable`, not a walk-over or a special loot packet.
The type-`11` tail supplies the target object ID; authenticated actor identity
must come from the common action header and peer binding.

At release, chunk `610`:

1. rechecks that the target still exists and is not marked for deletion;
2. emits `loot_acquiredFirstTime.ServerEventDef` only when the single-player
   simulator's first-loot flag was previously false;
3. calls `PlayersRollForLoot(targetID, playerID)`;
4. unconditionally calls `MarkForDelete(targetID)`;
5. waits the remaining animation time.

`PlayersRollForLoot -> sub_A04D60` resolves recipient policy:

- with exactly one simulator player, the interacting player's slot wins;
- with multiple players and nonzero `cLootData.mLootInstanceId`, the supplied
  player wins;
- with multiple players and zero instance ID, every simulator player draws an
  integer `1..100`; the highest wins and an exact tie is broken by a random
  bit. Each participant receives a roll presentation event. The winner alone
  reaches `sub_9C3740` with the generated item.

Because ordinary campaign equipment sets instance ID to zero, multiplayer
campaign loot is rolled across the player list. The obelisk activator is not
the exclusive owner and the player who clicks the pickup is not automatically
the winner. Confidence: **Exact**.

### Crystal collection

Chunk `134`, `Abilities/0xD71BDB7E.lua`, defines the distinct `PickUpCrystal`
interaction. Native generic ability admission owns request-time range and hit
checks. At its `1s` release, the Lua callback calls
`PickupCrystal(playerID,targetID)`; a target deleted during the cast becomes a
silent no-op.

Success fills the first free one of nine mission slots, copies noun and crystal
level, deletes the world object, emits subtype-zero `kGmsCrystalMessage`
(`0xc3`), and recomputes crystal bonuses, in that mutation order. The client
also locally checks `IsCrystalSlotAvailable` before requesting the action, but
the native collection body repeats the authoritative capacity decision.

Bonus recomputation evaluates three horizontal, three vertical, and, after the
diagonal upgrade, two diagonal lines. Prismatic color `1` matches either of the
other two catalysts' shared color. Every completed line increments each of its
three slots, and the packaged `CrystalTuning` scalar `0.5` adds half of that
catalyst's contribution per overlapping line. The player reflection carries
the same eight line flags to `MaxisHUDCatalysts`; its `updatelinkMCs` method
reveals the corresponding horizontal, vertical, or diagonal clip and plays its
`Intro` animation. Confidence: **Exact**.

When all nine slots are full, the pickup remains. Event `0x6ea4091e` is scoped
to the attempting player. If the preceding lob has ended, the pickup is
relaunched toward its current position with height `3`, duration `0.5s`, and
`groundCollisionOnly=true`; attempts during an active lob emit feedback but do
not restart it. No account/profile submission occurs. Confidence: **Exact**.

### Orb collection

Orb collection is server-owned overlap. The client-installed `Orb_Pickup`
callback merely queries its role and returns false: it sends no request,
changes no resource, emits no event, and deletes no object. Accepted movement
and current authoritative collision shapes must therefore drive collection.

The native client does not contain the missing server-side restore amount,
modifier application, squad-resource owner, or exact pickup/full publication
order. It does prove the six presentation assets, sparse HP/mana component
vocabulary, and one-shot deletion/expiry boundary described in
`tutorial.md`. Darkspin's current restore-`15`, active-entrant, event then
resource delta then delete behavior is live-compatible, but it is not recovered
retail policy.

## Inventory capacity and persistence boundary

The three capacities are separate:

| Capacity | Proven behavior |
| --- | --- |
| Orb resource cap | HP/mana caps govern whether restoration changes state. Exact retail restore amount and full-pickup consumption policy are absent from the client server stub. |
| Crystal capacity | Exactly nine mission slots; checked both by client prefilter and authoritative native collection. A full inventory retains and may relaunch the pickup. |
| Equipment inventory | Build-103 account XML supplies the active `unlock_inventory` ceiling. Live ship evidence shows `180` as the base and a zero ceiling produces `Inventory Full!`. The pickup Lua and `PlayersRollForLoot` path perform no visible capacity query. |

`AddLootToPlayer -> sub_9C3740` normalizes the generated item level and converts
the item into the client's loot/profile representation. In this client-shaped
executable it makes no HTTP/Blaze submission and exposes no success/failure
result to chunk `610`; Lua deletes the pickup after `PlayersRollForLoot`
regardless. This is evidence that durable account mutation belonged to the
missing authoritative server, not evidence that equipment may be dropped from
persistence.

Return-to-ship inventory reload is separately proven to call
`api.account.getAccount` for capacity and `api.inventory.getPartList` for owned
rows. No RakNet inventory snapshot replaces that HTTP authority. The retail
transaction boundary between roll victory, capacity admission, item identity
allocation, persistence, pickup deletion, and retry is not present in the
client. Confidence: **High negative evidence**.

Orb resource changes and crystal slots are mission state and have no recovered
account commit. Equipment is the only examined pickup category that must
produce a durable part row for ship reload. Whether unclaimed mission loot is
cashed out, discarded, or carried across a chain boundary remains unresolved.

## Proven policy versus recommended fallbacks

### Proven policy

- Apply the recovered selector budgets/chances before allocating world IDs.
- Select orb and crystal nouns from the recovered weighted policies; select a
  complete equipment item from the active rigblock/affix/tuning catalog.
- Create at the source, sample a source-centered navmesh destination, and lob
  with height `2.5` for `0.5s`.
- Publish `cLootData` separately for crystal and equipment; never serialize it
  inside `ObjectCreate`.
- Do not copy the obelisk activator into a fabricated pickup owner field.
- Use range-`2` interaction for crystal and equipment; use authoritative
  overlap for orbs.
- For ordinary zero-instance equipment in multiplayer, use the native
  `1..100` all-player roll with random tie-breaking.
- Preserve full-crystal behavior: retain the object, target feedback to the
  attempting player, and relaunch only after the active lob ends.
- Treat dropped health/mana noun lifetime `30s` as an authoritative simulation
  deadline.

### Recommended fallbacks

These policies keep campaign play safe where the original server is absent.
They are not parity claims:

1. **Packet publication.** Send reliable ordered
   `ObjectCreate -> cLootData (when present) -> lob`; publish the spawn event in
   the same ordered batch before the lob. This obeys receiver dependencies
   while leaving the exact retail batch order replaceable by a capture.
2. **Equipment transaction.** Resolve the native winner first, reserve one
   slot against the account's computed capacity, allocate stable item and
   reference IDs, and commit the complete part before deleting the pickup. On
   capacity or persistence failure, release the reservation and retain the
   pickup for retry. Do not emit a successful acquisition event before commit.
3. **Orb authority.** Until a retail capture supersedes it, restore 7.5% of
   each affected hero's maximum resource, preserving the live-compatible
   tutorial amount `15` at the base maximum of `200`; consume the orb even when
   already full and publish the matching full event without a resource delta.
4. **Transient ownership and teardown.** Admit crystal/equipment interaction
   only from the authenticated peer's deployed in-range actor and a live
   phase-owned pickup. Keep crystals and equipment until collection or phase
   teardown; do not invent a lifetime for nouns whose content lifetime has not
   been projected.
5. **Failure atomicity.** Roll, capacity reservation, durable mutation, world
   deletion, and success presentation must be one idempotent authority
   operation. A packet-encoding or storage failure must restore the pickup and
   avoid a duplicate item on retry.

## Reconciliation with existing notes

- `campaign-drops.md` is confirmed on selector meanings, independent
  Loot/Crystal branches, count/chance formulas, weighted noun generation,
  source-centered sampling, and shared `2.5`/`0.5s` lob. This note closes the
  concrete creation and collection paths without changing those results.
- The fixed Electro Claws pickup in `tutorial.md` remains a tutorial-route
  compatibility result. It is not the general campaign obelisk item selector.
- The percentages and named family pools in `loot.md` remain theoretical.
  Build-103 concrete equipment generation uses the 2,288-rigblock/666-affix
  active catalog and native eligibility filters.
- The current non-combatant 81-byte create envelope and current packet order
  are receiver-valid/live-compatible, not recovered authoritative sender
  goldens.

## Remaining capture targets

1. The orb difficulty-scale table entry used by each campaign difficulty.
2. Original-server `0x8c/0x9a/0x94/0x9b` reliability, masks, batching, and order
   for all three pickup categories.
3. Retail orb restore percentage, modifiers, full behavior, and active-versus-squad
   resource ownership after a second hero unlock.
4. Equipment-full feedback and whether retail retains, deletes, or redirects a
   pickup when the roll winner has no capacity.
5. The authoritative item persistence/idempotency transaction and chain-end
   handling for unclaimed loot.
6. Crystal/equipment natural lifetime, if any, plus late-join reconstruction
   of live pickups and the nine mission crystal slots.

## Implemented equipment award boundary

Campaign equipment collection now commits the generated part before publishing
its success presentation. With multiple connected participants, the server
draws `1..100` once per stable player slot, consumes a random bit for each exact
tie, and freezes the winner on the pickup before scheduling persistence. Only
that account receives the durable item and `LootAwarded`; every connected peer
first receives every participant's sparse roll event with field `7` controlled
object ID, field `14` integer roll, and field `15` client event `0x502A2D7E`,
then receives the shared world deletion. Capacity or persistence failure releases
the pickup reservation without selecting another winner. The accepted winner
response order is:

`action response -> durable GrantPart -> roll events -> loot_acquired ServerEvent
-> LootAwarded -> ObjectDelete`

`LootAwarded` carries the final persisted item and reference IDs rather than
the temporary world-object identity. Build 103's receiver at `0x004E17E0`
updates the local inventory and then owns its localized "received ..."
notification. Fang does not alter this client behavior.

Focused tests assert that the complete persisted part is present in
`LootAwarded`, that the award precedes world deletion, and that a failed
durable grant cannot publish success or delete the pickup.

## Reproduction commands

```text
darkrun db marker get marker_id=2655870385 --config bin/darkspinner/darkspin.toml
darkrun db marker get marker_id=785296806 --config bin/darkspinner/darkspin.toml
darkrun db crystal_definition get id>=0 --limit 8 --config bin/darkspinner/darkspin.toml
darkrun db crystal_level_offset get id>=0 --config bin/darkspinner/darkspin.toml
darkrun db loot_rigblock get id>=0 --limit 4 --config bin/darkspinner/darkspin.toml
darkrun db loot_tuning get id=1 --config bin/darkspinner/darkspin.toml
darkrun db lua_chunk get 610 --config bin/darkspinner/darkspin.toml
darkrun db lua_string_constant get lua_chunk_id=610 --limit 100 --config bin/darkspinner/darkspin.toml
darkrun db server_data bget decoded_payload where content_source_resource_id=14174 --decode zlib --output bin/game/logs/campaign-pickup-loot-14174.luac --config bin/darkspinner/darkspin.toml
darkrun lua bin/game/logs/campaign-pickup-loot-14174.luac
```
