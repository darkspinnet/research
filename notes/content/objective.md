# Objective content

## Implementation status

The first runtime boundary from this report is implemented. Generic zone
objective construction now accepts an arbitrary ordered selection and builds
one retained Lua runtime and state seed per selection. Campaign loading invokes
an explicitly named build-103 compatibility selector, so the current
`TouchAllObelisks` fallback is no longer presented as authored level content.
Initialization projection and member activation no longer assume exactly one
objective.

Zone completion now evaluates each selected packaged `ObjectiveStatus`
callback once for every active player, commits callback token writes, maps the
authored result to build-103 medal values, and publishes the completed records.
Elapsed mission time is retained across safe restart checkpoints. While a
campaign is active, the shared zone clock also advances token zero of the
packaged `FinishLevelQuickly` objective once per second, allowing its
`~time1~ on planet` text to advance in solo and co-op without changing the
authored completion-time medal callback.

The normalized `content.db` tables, pool importer, full event-mask surface,
content-backed selector, and replacement of the compatibility policy remain
outstanding.

## Decision summary

Objective construction should move to `server/zone/objective`, but objective
selection must not be presented as fully recovered authored behavior.

The evidence supports this source chain:

```text
AssetData_Binary.package LevelObjectives resources
  -> named objective/affix eligibility pools
ServerData.package compiled Lua
  -> registered objective definition, ID, labels, flags, event mask,
     initialization, live counters, and ObjectiveStatus medal predicate
level/director realization
  -> runtime census and events consumed by the selected Lua definitions
server selection policy
  -> ordered selected slots and active players for this zone instance
server/zone/objective
  -> per-zone Lua runtimes, token/state ledger, event dispatch, final medals
server/zone/objective/raknet103
  -> 0xb7 / 0xb8 / 0xb9 / 0xca projection
build-103 client
  -> fixed five-row Objectives Log headings, localization detokenization,
     medal rendering, and optional transient notification
```

The tutorial level `Game_Tutorial_cryos_1` has no authored
`LevelObjectives` override or objective callback. The current five-objective
list in `server/zone/tutorial/objective` is therefore not tutorial-authored
content. It is a darkspin compatibility selection assembled from globally
registered objective scripts. The same fixed inputs are also reused by the
campaign path, where generic code then selects only `TouchAllObelisks`.

Implement the content model and generic runtime first. Preserve current
selection as an explicitly named compatibility policy until a retail
server-to-client trace or additional server-side selection evidence replaces
it. Do not import the fixed tutorial list as if it came from the tutorial
level.

## Evidence and confidence

Confidence labels in this report are:

- **H — high:** directly decoded from the authoritative runtime `content.db`,
  packaged bytes represented by it, `Game.c`, or an exact build-103
  packet receiver.
- **M — medium:** multiple sources agree, but an upstream selector, symbolic
  asset-to-resource mapping, or exact event-slot name is inferred.
- **L — low:** a useful implementation hypothesis that must remain an
  explicit policy or gap.

Primary evidence:

| Source | Result | Confidence |
| --- | --- | --- |
| `bin/darkspinner/darkspin/cache/content.db` through `darkrun db` | 13 Lua chunks contain `nObjective.RegisterObjective`; their source IDs, hashes, string constants, localization keys, and retained bytecode are authoritative. | H |
| DBPF type `0x68232164` in `AssetData_Binary.package` | Three `LevelObjectives` resources contain 10 ordinary objective entries, eight default affix entries, and one Juggernaut entry. | H |
| `bin/game/GameBin/Game.c` | Exact status values, token storage, Lua setter behavior, client vector readers, popup detokenization, and message receivers. | H |
| `notes/client/packet.md` and the native receivers | Exact `0xb7`, `0xb8`, `0xb9`, and `0xca` wire layouts. | H |
| Current Darkspinner traces and `notes/profile/objectives-log.md` | Darkspin currently publishes five fixed records, followed by a synthetic `FinishLevelQuickly` update, before scene objects. This proves client compatibility and timing, not retail selection. | H for observed darkspin behavior; L for retail policy |
| Tutorial level/director import | Level `id=9`, name `Game_Tutorial_cryos_1`, alias `Game_Tutorial_cryos_1_v2`, has no objective-pool binding. It does contain three loot and two health obelisks. | H |

The current runtime schema has no table whose name contains `objective`.
`notes/content/inventory.md` describes a former/intended
`level_objective_entry` projection with 19 rows, but neither the authoritative
runtime database nor retained recipe databases queried by the existing
diagnostic contain that table. The report below proposes a replacement rather
than relying on that stale inventory claim.

## Authored objective pools

### Resource identities

The binary resource type is `0x68232164`. DBPF supplies stable
type/group/instance identity but does not retain original authored paths.
Asset-catalog names and native defaults supply the symbolic layer.

| Symbolic asset | DBPF instance | Decoded SHA-256 | Contents | Confidence |
| --- | ---: | --- | --- | --- |
| `Default.LevelObjectives` | `0x2ea8fb98` | `c1b3e14a9f43efa6585a535af3cc3683eef16583ff90daccfd39bf797d95c79b` | Six ordinary objectives plus affix arrays of 3, 2, and 3 entries. Build 103 initializes its native `levelObjectives` field to this instance. | H |
| `Daily.LevelObjectives` | `0x71aec1dc` | `0e6841e7f70888dcbe27cc2ed815b48ab60b2def7586839c8fa409507d756965` | Four objective entries. The symbolic mapping is the remaining cataloged ordinary pool and is not independently named in the DBPF index. | M |
| `Juggernaut.LevelObjectives` | `0x99e410f1` | `5f43225a14b375decdd19d51e8c08864f2fe1c8a0f8282a66abe82ebd5a4bd17` | One `Juggernaut` entry whose FNV-1 ID is also `0x99e410f1`. | H |

`Default.LevelObjectives` contains these objective entries in authored order:

| Ordinal | Name | Objective ID | Confidence |
| ---: | --- | ---: | --- |
| 0 | `DestroyAllDestructables` | `0xc5980573` | H |
| 1 | `DefeatAllMonsters` | `0xa28485cc` | H |
| 2 | `LootCrystals` | `0x53449566` | H |
| 3 | `TouchAllObelisks` | `0x61c07561` | H |
| 4 | `DontUseHealthObelisks` | `0x79278139` | H |
| 5 | `StayAlive` | `0x6d26f97d` | H |

Its three affix arrays contain:

| Array/ordinal | Affix name | ID | Confidence |
| --- | --- | ---: | --- |
| 0/0 | `EnemyPeriodicStun` | `0x7425b79c` | H |
| 0/1 | `PlayerBasicMana` | `0xc3f3d509` | H |
| 0/2 | `EnemyProjectilesSlowed` | `0x69c3bf71` | H |
| 1/0 | `EnemyHealthRegen` | `0x0f3e7324` | H |
| 1/1 | `EnemyIncreasedSpeed` | `0xc8077e1a` | H |
| 2/0 | `PlayerHealthDrain` | `0xec532a92` | H |
| 2/1 | `VoidZones` | `0x0d76d778` | H |
| 2/2 | `MeteorShower` | `0xa5800dc3` | H |

The names of the three arrays—historically described as
positive/minor/major—are not proven by the binary itself. Preserve
`array_ordinal` and do not assign gameplay severity names in the database
until the authored XML/property mapping is recovered. Confidence in the counts
and order is H; confidence in positive/minor/major labels is L.

`Daily.LevelObjectives` contains:

| Ordinal | Name | Objective ID | Confidence |
| ---: | --- | ---: | --- |
| 0 | `DestroyAllDestructables` | `0xc5980573` | H |
| 1 | `HugeDamage` | `0x0478facb` | H |
| 2 | `DoDamageOften` | `0xac4273f3` | H |
| 3 | `LootCrystals` | `0x53449566` | H |

This proves eligibility, not the number drawn, slot assignment, weighting, or
roll seed. Duplicates across Default and Daily are authored and must be
retained.

### Stable objective IDs

`nObjective.RegisterObjective(name, definition)` supplies the canonical
symbolic name. Build 103 uses darkspin's case-insensitive 32-bit FNV-1
algorithm, not FNV-1a:

```text
hash = 0x811c9dc5
for each lowercased ASCII byte:
    hash *= 0x01000193
    hash ^= byte
```

All five IDs already used by darkspin and all pool IDs reproduce exactly with
this algorithm. The stable definition key is therefore `(objective_id, name)`;
the Lua chunk ID is provenance and may change across recipes.

## All packaged Lua objective definitions

The following table maps every registered objective in the runtime database.
“Shared” means authored `groupObjective=true`; “timed eligibility” means the
literal, awkwardly named authored flag `timeObjectiveRequires=true`. Neither
flag means “required to finish the zone.”

| Objective | ID | Lua source / chunk | Labels | Flags and event | Initial tokens | Final medal predicate | Confidence |
| --- | ---: | --- | --- | --- | --- | --- | --- |
| `TouchAllObelisks` | `0x61c07561` | `lua/0x409C9F2B.lua`, chunk 5, SHA `e168a19cf64bddaff0b483b7d1b2546011d14cc4a72b5528331105c4b2133047` | title `0x097562ee`; progress `0x098cd6b8`; error `0x0990d32e`; completed VO `vo_ship_obelisk_accessed` | Shared; timed eligibility; `TouchedObelisk` | `(0,total,total)` for all players; total is the count of `InteractWithObelisk` objects | Gold/Silver/Bronze at at least 3/2/1 unique accepted uses; otherwise Failed | H |
| `DontUseHealthObelisks` | `0x79278139` | `lua/0x12410255.lua`, chunk 163, SHA `ce065a01ba48775543cbbcb24807465398c2a3a8a1a94af8866c6f2dde99beda` | title `0x0b556ee7`; progress `0x0b556f18` | Shared; `TouchedHealthObelisk` | `(0,total,total)`; total is `InteractHealthObelisk` count | Gold/Silver/Bronze for at most 0/1/2 accepted uses | H |
| `EnemyHealthRegen` | `0x0f3e7324` | `lua/0x31F89624.lua`, chunk 173, SHA `c702ffcdae3ee116eb62f71a786b801a0b935cbd9c34813ecf5ab10823738b86` | title `0x098cd815`; no progress key | `Heal`; writes all players despite no group flag | private regen count and amount start at zero; no Init token write | Gold/Silver/Bronze for fewer than 50/100/150 qualifying NPC heals; token 0=count, token 1=total healed amount | H for thresholds/counters; M for exact GUID-slot qualification |
| `PlayerHealthDrain` | `0xec532a92` | `lua/0x97584392.lua`, chunk 280, SHA `3f56609d4ff248916f2d4f51ad98b937da7ce2c8c75b9ffad88bace200127673` | title `0x098cd816`; no progress key | Definition subscribes to `Damage`, but `HandleEvent` tests `Death` | no Init token write | Per player, Gold/Silver/Bronze for at most 0/1/2 player-controlled deaths attributed to no source | H; the event-mask mismatch is an authored defect, not an importer correction |
| `DestroyAllDestructables` | `0xc5980573` | `lua/0x55AC5673.lua`, chunk 377, SHA `543949cb5562798bbe07fa0c03fae425867bb22c39fadfb3ca1db537049121c7` | title `0x09e6d558`; progress `0x09e6d559`; invalid completed VO | Shared; timed eligibility; `Death` | token 1 = 90; private denominator is registered marked destructibles | Gold/Silver/Bronze at at least 90%/70%/50%; token 0 reports percentage with 25-point notification boundaries | H |
| `DefeatAllMonsters` | `0xa28485cc` | `lua/0x4D899ECC.lua`, chunk 384, SHA `fda2e11e8da8f72c7415c061ee35b98d62a2d8ebbc71d0ce2ebaaf583a96f70a` | title `0x097562ef`; progress `0x098cd6b9` | Shared; timed eligibility; `Death | FullClear` | token 1 receives low 32 bits of `0.95*100`, which build 103 truncates to 94 | Gold/Silver/Bronze at kill fraction at least 95%/75%/50%; full clear writes 100; token 0 otherwise reports percentage and 25-point boundaries | H |
| `StayAlive` | `0x6d26f97d` | `lua/0x68E5F27D.lua`, chunk 506, SHA `1547e9e5dcda912fc166202042e9d09ac04c54b73fc727e2de37ff7834d8a4ae` | title `0x097562f1`; progress `0x097562f2` | Per-player `Death` | no Init token write | Gold/Silver/Bronze for at most 0/1/2 player-controlled hero deaths; token 0 is deaths for that player | H |
| `LootCrystals` | `0x53449566` | `lua/0x52528866.lua`, chunk 631, SHA `bb8cf84a351727e30a37596b334850862e9bb040df8cff0137e279585ab9741a` | title `0x09e6e562`; progress `0x09e6e563` | Shared; timed eligibility; `Death`; custom `CrystalFound` callback | tokens `(0,7,0)` | Drops `loot_questCrystal.Noun` when director kill fraction passes successive 0.135 boundaries; `CrystalFound` increments token 0; Gold/Silver/Bronze at at least 7/5/3 found | H; this proves objectives may emit world commands, not just counters |
| `HugeDamage` | `0x0478facb` | `lua/0x521AF5CB.lua`, chunk 708, SHA `0031aa46c8b68375d6b8976a0655a5c41bc8e1dbef70d1bb4703e30fa1877bfc` | title `0x09e6d259`; progress `0x09e6d260` | Timed eligibility; per-player `Damage` | token 1 = difficulty health multiplier × 35; token 2 = 30 | Per-player qualifying hits above token-1 threshold; Gold/Silver/Bronze at at least 30/15/5; token 0=count | H |
| `FinishLevelQuickly` | `0xff9733ee` | `lua/0x389D8EEE.lua`, chunk 754, SHA `cb9020a40265832803e62418ca1141fff829373d57b2f3ef8b644ee77bdc875c` | title `0x097562f0`; progress `0x09d16768` | Shared; no event subscription | no Init token write | On status evaluation, token 0=floor(completion seconds); Gold/Silver/Bronze at no more than 600/900/1200 seconds | H |
| `NoSlowing` | `0xe2745fb5` | `lua/0xDE3358B5.lua`, chunk 833, SHA `42f1ae9e55c14a58fe5578400f0505d78723b45f8e05c6cc36e084e1f02aab0a` | title `0x09e6d038`; no progress key | Per-player `ModifierCreated` | no Init token write and no live token write | Counts root/slow descriptors attributed to each player; Gold/Silver/Bronze for at most 0/10/20 | H for mask/threshold; M for the two GUID roles |
| `DefeatSameType` | `0xc0fbb81e` | `lua/0xE982631E.lua`, chunk 849, SHA `8f15ed5adb0e0de1becb91218d36b19f515224eebe9e5dc8fbf2c971564ecec0` | title `0x09e6d288`; progress `0x09e6d289` | Timed eligibility; per-player `Death` | tokens `(0,15,0)` | Tracks current same-NPC-asset streak and retained maximum; Gold/Silver/Bronze at at least 15/10/5; token 0 updates and notifies at multiples of 5 | H |
| `DoDamageOften` | `0xac4273f3` | `lua/0x2F37FCF3.lua`, chunk 902, SHA `8441e30aa93c5e61cb09cb2817bfc40a6a4fca856aea24645c673dd3d6721db4` | title `0x09e6d276`; progress `0x09e6d277` | Timed eligibility; per-player `Damage`; difficulty scale 7 | token 1 = scaled Gold target, 150 at the current baseline | Qualifying hostile-NPC hits; difficulty-scaled Gold 150..500, Silver 100..350, Bronze 50..200; token 0=count | H |

The English strings in `localization_text` resolve exactly:

| Key | Text | Confidence |
| --- | --- | --- |
| `0x097562ee` | `Access All Crogenitor Archive Obelisks` | H |
| `0x098cd6b8` | `~int1~ of ~int2~ Crogenitor Archive Obelisks Accessed` | H |
| `0x0990d32e` | `Access ~int3~ more Crogenitor Archive Obelisks` | H |
| `0x0b556ee7` / `0x0b556f18` | `Don't Use Health Obelisks` / `Medal downgraded! ~int1~ of ~int2~ Health Obelisks used up` | H |
| `0x098cd815` | `Don't let Game regen health` | H |
| `0x098cd816` | `Don't perish from health drain` | H |
| `0x09e6d558` / `0x09e6d559` | `Destroy All Destructible Objects` / `~int1~% of destructible objects destroyed` | H |
| `0x097562ef` / `0x098cd6b9` | `Defeat All Game On The Planet` / `Approximately ~int1~% Game defeated` | H |
| `0x097562f1` / `0x097562f2` | `Don't Let Your Heroes Perish` / `~int1~ Heroes perished` | H |
| `0x09e6e562` / `0x09e6e563` | `Collect ~int2~ Dropped Crystals` / `~int1~ of ~int2~ crystals found` | H |
| `0x09e6d259` / `0x09e6d260` | `Deal Over ~int2~ Damage ~int3~ Times` / `Dealt more than ~int2~ damage ~int1~ times` | H |
| `0x097562f0` / `0x09d16768` | `Finish Game Threat Quickly` / `~time1~ on planet` | H |
| `0x09e6d038` | `Don't root or slow Game` | H |
| `0x09e6d288` / `0x09e6d289` | `Defeat ~int2~ Of The Same Species In A Row` / `~int1~ of the same species defeated in a row` | H; the package text contains unusual whitespace before `a Row` |
| `0x09e6d276` / `0x09e6d277` | `Deal Damage To Game ~int2~ Times` / `Damage dealt to Game ~int1~ times` | H |

### Status, scope, type, and optionality

Packaged `GlobalDefinitions.lua` proves:

```text
InProgress = 0
Failed     = 1
Bronze     = 2
Silver     = 3
Gold       = 4
```

The current tutorial seeds selected objectives with state `1`. That is
`Failed`, not `InProgress`, even though the current client presentation can
look like an unearned grey medal. A content-driven implementation should seed
state `0` and retain a compatibility switch only if a clean client test proves
that build 103 requires `1` before final evaluation. Confidence in the enum is
H; confidence in the best compatibility seed without a retail packet is M.

There is no authored scalar “objective type.” Preserve the dimensions that
actually exist:

- scope: shared when `groupObjective=true`, otherwise per-player;
- selection constraint: `timeObjectiveRequires`;
- event mask: the `nObjectiveEvents` bitmask;
- slot role: Daily, Standard, or Hidden Bonus, assigned by selection;
- medal predicate: executable `ObjectiveStatus`;
- presentation templates and token roles.

There is no `is_required` or `is_optional` property. No recovered objective
callback completes a level, enables Beam Out, or grants a permanent reward.
These are medal challenges and are optional relative to zone completion.
`groupObjective` does not mean required. Confidence is H for the absence of a
completion call in the 13 scripts and M for the global retail design statement
because the unrecovered server selector could impose additional policy.

### Event mask and payload boundary

The packaged enum is:

| Event | Mask | Script consumers | Required semantic payload | Confidence |
| --- | ---: | --- | --- | --- |
| `TouchedObelisk` | `1` | `TouchAllObelisks` | accepted interactable/object identity | H |
| `Damage` | `2` | `DoDamageOften`, `HugeDamage`; declared by `PlayerHealthDrain` | target, source, float damage, integer metadata, teams/player ownership | H for indices used by scripts; M for names of every native event slot |
| `Death` | `4` | defeat/destructible/crystal/survival/streak definitions | dead object, source where used, player attribution, NPC/asset/team facts | H/M |
| `FullClear` | `8` | `DefeatAllMonsters` | authoritative kill fraction 1.0 | H |
| `ModifierCreated` | `16` | `NoSlowing` | two object identities and debuff descriptor mask | H/M |
| `ActivateAffix` | `32` | no recovered objective definition | affix identity/payload unrecovered | H for enum; L for payload |
| `Heal` | `64` | `EnemyHealthRegen` | healed/source identities and amount | H/M |
| `TouchedHealthObelisk` | `128` | `DontUseHealthObelisks` | accepted interactable/object identity | H |

The generic feature currently models equipment and health obelisk use, damage,
NPC healing, player/NPC/destructible death, hostile Root/Slow creation, and
full clear. NPC direct healing, life drain, repair, Ooze growth, and corpse
consumption publish the exact `Heal` GUID slots 1/2 and float slot 3 recovered
from `EnemyHealthRegen`. `ModifierCreated` uses its distinct GUID slots 0/1 and
integer slot 3; death retains victim/source GUID slots 1/2, credited player in
integer slot 3, stable victim asset identity, team, NPC type, and authored
marker identity where present. Content-driven loading still requires affix and
quest-crystal event producers.
Event production remains owned by combat, modifier, interaction, director, and
loot features; they publish transport-neutral semantic events to the objective
session.

## Selection and lifecycle

### What is authored

- Default, Daily, and Juggernaut pool membership and order.
- Definition metadata and behavior in Lua.
- The native `levelObjectives` default resource `0x2ea8fb98`.
- Tutorial level objects and director/population inputs used by objective
  initialization and progress.
- Five fixed UI row positions in the Objectives Log.

### What is a native client default

`MaxisObjectivePopup` (`0x0041D2E0`; population in `sub_41D300`) loops five
rows. For each present record it looks up the registered definition, selects
the current player's state, detokenizes the title/progress keys from the three
tokens, and calls `SetMedal` and `SetObjectiveText`.

If a row is absent/unregistered, it uses:

- `0x099532aa`: `Locked Objective`
- `0x099532ab`: `Complete other objectives to unlock.`

The SWF owns the fixed headings:

- row 0: Daily;
- rows 1–3: Standard;
- row 4: Hidden Bonus.

No category value travels in an objective record. A selected record is visible
because it is present in the client vector after `0xb7` or `0xca`.
`0xb8.is_shown` controls a transient notification path and is not Hidden Bonus
visibility. Confidence is H.

### What remains server authority

No recovered content or client receiver proves:

- which Daily entry is selected;
- which three Default entries are selected;
- how affixes choose or reveal the Hidden Bonus objective;
- whether pool order is weighted or merely authored order;
- reroll seed, difficulty gates, duplicate rejection, or party reconciliation;
- whether `FinishLevelQuickly`, absent from both ordinary pools, is automatic;
- retail initial state bytes;
- final packet ordering and reward conversion.

The current darkspin selection
`FinishLevelQuickly, DoDamageOften, TouchAllObelisks, DefeatAllMonsters,
HugeDamage` is a compatibility policy, not a recovered tutorial or campaign
definition. The trace reproducing it was generated against darkspin and must
not be cited as retail selection evidence.

### Required load order

Several `Init` functions query realized zone state:

- loot and health obelisk counts;
- registered destructible count;
- difficulty/health multiplier;
- director kill fraction;
- selected affix/objective relationships.

Therefore:

1. Load the level, marker sets, director definition, difficulty, and selected
   affixes.
2. Realize the authoritative interactable/destructible/population census, but
   do not accept gameplay events yet.
3. Select the ordered objective slots using an injected server policy.
4. Load one Lua definition/runtime per selected objective.
5. Run every selected `Init` against a read-only zone census and apply emitted
   token mutations.
6. Activate player rows with `InProgress`.
7. Publish `0xb7` before ordinary scene objects.
8. Open gameplay event admission and route semantic events to subscribed
   runtimes.
9. At accepted level completion, freeze the event stream, evaluate every
   `ObjectiveStatus`, apply status-time token writes, and freeze one immutable
   result snapshot.
10. Project any final `0xb8` updates and `0xb9` from that snapshot. Exact retail
    ordering remains a gap.

`LootCrystals` proves that the runtime host must also support semantic world
commands. Its Lua death callback can request a quest-crystal drop, while its
custom `CrystalFound` callback updates progress. Do not constrain the generic
feature to `SetObjectiveIntData` alone.

The retained runtime now exposes custom callback invocation and implements the
exact six-argument `SetObjectiveIntDataFromSPID` shape used by `CrystalFound`.
The zone exposes the corresponding transport-neutral committed-pickup hook.
Quest-crystal world creation and pickup classification remain deliberately
separate so ordinary nine-slot catalyst collection cannot be mistaken for an
objective crystal.

## Build-103 opcodes `0xb7` through `0xca`

Only four messages in this range are objective-vector messages. `0xba`,
combat, can be an objective event source; `0xc8` is tutorial completion but is
not an objective packet.

| Opcode | Name | Direction | Objective relevance | Confidence |
| ---: | --- | --- | --- | --- |
| `0xb7` | `ObjectivesInit` | S2C | Replace the selected vector. | H |
| `0xb8` | `ObjectiveUpdated` | S2C | Replace one objective/player status and all three tokens; optionally show notification/voice. | H |
| `0xb9` | `ObjectivesComplete` | S2C | Final vector plus 16 separately stored result bytes. | H for layout; L for result-byte semantics |
| `0xba` | `CombatEvent` | S2C | Combat presentation only; authoritative combat also produces semantic Damage/Heal/Death events. | H |
| `0xbb` | `JuggernautPlayerMsgs` | C2S | Mode command, not ordinary objectives. | H |
| `0xbc` | `JuggernautLobbyMsgs` | unconsumed | No build-103 receiver/constructor. | H |
| `0xbd` | `JuggernautGameMsgs` | S2C | Juggernaut mode state, separate from `Juggernaut.LevelObjectives`. | H |
| `0xbe` | `JuggernautResultsMsgs` | S2C | Juggernaut result state; no proven ordinary-objective coupling. | H |
| `0xbf` | `ReloadLevel` | S2C | Teardown/reload must replace the objective session and vector. | H for packet; M for required server ordering |
| `0xc0` | `GravityForceUpdate` | S2C | None. | H |
| `0xc1` | `CooldownUpdate` | S2C | None. | H |
| `0xc2` | `CrystalDragMessage` | C2S | Editor/inventory crystal drag, not `LootCrystals` objective collection. | H |
| `0xc3` | `CrystalMessage` | S2C | Crystal HUD queue; distinct from the quest-crystal objective's semantic `CrystalFound`. | H |
| `0xc4` | `KillRacePlayerMsgs` | C2S | Mode command. | H |
| `0xc5` | `KillRaceLobbyMsgs` | unconsumed | No build-103 receiver/constructor. | H |
| `0xc6` | `KillRaceGameMsgs` | S2C | Mode state. | H |
| `0xc7` | `KillRaceResultsMsgs` | S2C | Mode results. | H |
| `0xc8` | `TutorialGameMsgs` | S2C | Tutorial completion/progression transition; not an objective update or predicate. | H |
| `0xc9` | `CinematicMsgs` | S2C | None. | H |
| `0xca` | `ObjectiveAdd` | S2C | Append one selected record without replacing the vector. | H |

Exact objective payloads, excluding the one-byte application opcode:

```text
0xb7:
  u8 count
  repeat count:
    u32le objective_id
    u8 state[4]
    u32le token[4][3]

0xb8:
  u32le objective_id
  u8 player_index          // 0xff means all four in the client receiver
  u8 state
  u32le voiceover_guid
  bool is_shown
  u32le token[3]

0xb9:
  u8 count
  repeat count:
    ObjectiveRecord
  repeat four players:
    u8 result_field_0
    u8 result_field_1
    u8 result_field_2
    u8 result_field_3

0xca:
  ObjectiveRecord
```

`ObjectiveRecord` is exactly 56 bytes. The client expands each state byte to a
DWORD and retains all 48 token bytes in a 68-byte local record.

The Lua setter
`SetObjectiveIntData(player,index,integer,bool,bool)` accepts player `0..3` or
`0xff`, token index `0..2`, truncates the Lua number toward zero through a
signed integer conversion, and expands `0xff` to all four rows. In
`sub_A1ED60`, the first boolean causes the objective thread's exit/update path;
the second parameter is not read by that setter body. The relationship between
those two authored booleans and `0xb8.is_shown` is not proven and must remain
opaque in the transport-neutral mutation. Confidence is H.

## Rewards and persistence

No objective pool, objective Lua definition, tutorial level, campaign level,
director resource, or localization row contains a permanent item, DNA, XP, or
account reward for an objective medal. `LootCrystals` creates mission-local
quest drops as part of its own counter loop; that is not a cash-out reward.

`0xb9`'s trailing 16 bytes have unknown semantics. Later `0xab` cash-out data
contains server-supplied per-player Gold/Silver/Bronze aggregate counts and
reward cards, but the client does not calculate those counts from the
objective vector and does not select the rewards. The conversion from final
objective states to aggregate medals, rarity, DNA, or loot is unrecovered
server policy.

Consequences:

- keep live objective state session-local and non-persistent;
- freeze final objective medals into the zone result;
- do not add reward columns to `objective_definition`;
- expose a separate result/reward operation that may consume final medals when
  an evidence-backed policy exists;
- keep the 16 `0xb9` bytes explicitly named `ResultField[4][4]` until traced;
- do not reuse account `cashout_bonus_time` for objective state.

Confidence is H for the negative content result and L for retail reward policy.

## Proposed normalized `content.db` schema

The schema separates immutable authored definition, selection pools, token
presentation, and optional level bindings. It does not store transient
selection/progress.

```sql
CREATE TABLE objective_definition (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE COLLATE NOCASE,
    lua_chunk_id INTEGER NOT NULL UNIQUE
        REFERENCES lua_chunk(id) ON DELETE CASCADE,
    namespace TEXT NOT NULL,
    description_locale_key TEXT NOT NULL,
    progress_locale_key TEXT,
    error_locale_key TEXT,
    completed_voiceover_id INTEGER,
    progress_voiceover_id INTEGER,
    event_mask INTEGER NOT NULL CHECK (event_mask >= 0),
    is_group INTEGER NOT NULL CHECK (is_group IN (0, 1)),
    is_time_required INTEGER NOT NULL CHECK (is_time_required IN (0, 1))
);

CREATE TABLE objective_token (
    objective_id INTEGER NOT NULL
        REFERENCES objective_definition(id) ON DELETE CASCADE,
    token_index INTEGER NOT NULL CHECK (token_index BETWEEN 0 AND 2),
    token_role TEXT NOT NULL,
    PRIMARY KEY (objective_id, token_index)
);

CREATE TABLE objective_property (
    objective_id INTEGER NOT NULL
        REFERENCES objective_definition(id) ON DELETE CASCADE,
    property_name TEXT NOT NULL,
    property_kind TEXT NOT NULL
        CHECK (property_kind IN ('integer', 'real', 'boolean', 'guid', 'asset', 'text')),
    integer_data INTEGER,
    real_data REAL,
    text_data TEXT,
    evidence TEXT NOT NULL,
    PRIMARY KEY (objective_id, property_name),
    CHECK (
        (property_kind='integer' AND integer_data IS NOT NULL AND real_data IS NULL AND text_data IS NULL)
        OR (property_kind='real' AND integer_data IS NULL AND real_data IS NOT NULL AND text_data IS NULL)
        OR (property_kind='boolean' AND integer_data IN (0, 1) AND real_data IS NULL AND text_data IS NULL)
        OR (property_kind IN ('guid', 'asset', 'text') AND integer_data IS NULL AND real_data IS NULL AND text_data IS NOT NULL)
    )
);

CREATE TABLE objective_pool (
    id INTEGER PRIMARY KEY,
    asset_name TEXT NOT NULL UNIQUE COLLATE NOCASE,
    asset_hash INTEGER NOT NULL UNIQUE,
    content_source_resource_id INTEGER NOT NULL UNIQUE
        REFERENCES content_source_resource(id) ON DELETE CASCADE,
    pool_kind TEXT NOT NULL CHECK (pool_kind IN ('default', 'daily', 'mode'))
);

CREATE TABLE objective_pool_entry (
    id INTEGER PRIMARY KEY,
    objective_pool_id INTEGER NOT NULL
        REFERENCES objective_pool(id) ON DELETE CASCADE,
    entry_kind TEXT NOT NULL CHECK (entry_kind IN ('objective', 'affix')),
    array_ordinal INTEGER NOT NULL CHECK (array_ordinal >= 0),
    entry_ordinal INTEGER NOT NULL CHECK (entry_ordinal >= 0),
    entry_name TEXT NOT NULL COLLATE NOCASE,
    entry_hash INTEGER NOT NULL,
    objective_id INTEGER REFERENCES objective_definition(id) ON DELETE RESTRICT,
    UNIQUE (objective_pool_id, array_ordinal, entry_ordinal),
    CHECK (
        (entry_kind='objective' AND objective_id IS NOT NULL)
        OR (entry_kind='affix' AND objective_id IS NULL)
    )
);

CREATE TABLE level_objective_pool (
    level_id INTEGER NOT NULL REFERENCES level(id) ON DELETE CASCADE,
    objective_pool_id INTEGER NOT NULL
        REFERENCES objective_pool(id) ON DELETE RESTRICT,
    binding_ordinal INTEGER NOT NULL CHECK (binding_ordinal >= 0),
    binding_role TEXT NOT NULL,
    PRIMARY KEY (level_id, binding_ordinal)
);
```

Field confidence:

| Field | Source | Confidence |
| --- | --- | --- |
| `objective_definition.id` | FNV-1 registered name; pool IDs and five live IDs agree | H |
| `name`, `namespace`, `lua_chunk_id` | Lua registration/table and runtime provenance | H |
| locale/voice fields | direct static Lua assignments | H |
| `event_mask` | direct `nObjectiveEvents` assignment | H |
| `is_group`, `is_time_required` | direct static Lua booleans, default false when absent | H |
| token index | native limit and localization placeholders | H |
| `token_role` | semantic interpretation of each callback/template | M; retain `unknown` where not used |
| `objective_property` typed value | direct static constants | H |
| pool source resource and entry order | decoded DBPF bytes | H |
| pool symbolic name/kind | native default plus asset catalog; Daily mapping partly by elimination | H for Default/Juggernaut, M for Daily |
| `array_ordinal` | binary array order | H |
| affix array semantic role | not encoded in recovered binary names | L; keep numeric ordinal |
| `level_objective_pool` row | only if a source reference is decoded | H when present; current tutorial row count must be zero |
| `binding_role` | authored field name if recovered | L today; do not populate from UI slot assumptions |

`objective_property` is useful for inspection and validation, but executable
Lua remains authoritative for derived thresholds and predicates. Do not turn
every callback into a handwritten rule table.

### Importer sources and validation

1. Scan `server_data.is_compiled_lua=1`.
2. Parse Lua 5.1 bytecode and identify top-level
   `nObjective.RegisterObjective`.
3. Execute each candidate in the constrained compiler to capture the
   registration name and definition table. Do not infer a definition merely
   from the string `RegisterObjective`.
4. Compute FNV-1 ID and reject zero/collision/name mismatch.
5. Extract static labels, flags, event mask, voice IDs, assets, and scalar
   properties. Link `lua_chunk_id` and preserve bytecode SHA-256.
6. Validate every locale key against `localization_text` in every available
   locale; absence is an importer error for `descriptionText` and a nullable
   result for optional progress/error keys.
7. Decode all type-`0x68232164` resources; retain exact arrays and ordinals.
8. Link objective entries by name and ID. Keep affix entries by name/hash and
   later link them to the affix definition table when that table exists.
9. Import level/director bindings only from an actual source reference.
   Absence is meaningful; never synthesize a tutorial binding.
10. Verify expected build-103 invariants: 13 definitions, three pools, 10
    ordinary pool entries, nine affix/mode entries, no dangling objective
    entry, and exact source hashes.

Stable keys are:

- objective: computed ID plus registration name;
- Lua provenance: `lua_chunk.id` plus SHA-256;
- pool: symbolic asset hash/name plus DBPF type/group/instance and decoded
  SHA-256;
- pool entry: `(pool asset hash, array ordinal, entry ordinal)`;
- level binding: `(level.id, binding ordinal)`.

## Transport-neutral Go model

The content adapter should return immutable definitions; the zone owns
selection and mutable instances.

```go
package objective

type Scope uint8

const (
    ScopePlayer Scope = iota
    ScopeGroup
)

type SlotRole uint8

const (
    SlotDaily SlotRole = iota
    SlotStandard
    SlotHiddenBonus
)

type Localization struct {
    DescriptionKey string
    ProgressKey    string
    ErrorKey       string
}

type TokenDefinition struct {
    Index uint8
    Role  string
}

type Definition struct {
    ID                  uint32
    Name                string
    Lua                 sim.LuaBytecode
    Namespace           string
    Localization        Localization
    CompletedVoiceover  uint32
    ProgressVoiceover   uint32
    EventMask           EventMask
    Scope               Scope
    IsTimeRequired      bool
    Token               [3]TokenDefinition
}

type Selection struct {
    Slot       uint8
    Role       SlotRole
    Definition Definition
}

type Census interface {
    InteractableCount(abilityID uint32) int
    RegisteredDestructibleCount() int
    Difficulty() uint32
    HealthMultiplier() float64
    KillFraction() float64
}

type DefinitionSource interface {
    Definitions(ctx context.Context) ([]Definition, error)
    Pools(ctx context.Context) ([]Pool, error)
    LevelPools(ctx context.Context, levelID int64) ([]PoolBinding, error)
}

type Selector interface {
    Select(ctx context.Context, req SelectionRequest) ([]Selection, error)
}

type Command interface {
    isObjectiveCommand()
}

type TokenChanged struct {
    ObjectiveID uint32
    PlayerIndex uint8
    TokenIndex  uint8
    Integer     int32
    Flag        [2]bool
}

type DropRequested struct {
    ObjectiveID uint32
    NounID      uint32
    SourceID    uint32
}
```

The actual code should follow repository naming and error-wrapping
conventions. The sketch shows ownership, not a mandate for this exact number of
files or interfaces.

`Session` should own:

- ordered selected definitions;
- one retained Lua runtime/private table per selected objective;
- `[4]Status` and `[4][3]int32` per objective;
- active player mask;
- event subscription index;
- accepted mutation/command log;
- start/completion clock;
- immutable final snapshot once completed.

The generic runtime should expose:

- `Initialize(ctx, census)`;
- `Join(playerIndex)` / `Leave(playerIndex)`;
- `Apply(ctx, SemanticEvent) []Publication`;
- `Invoke(ctx, objectiveID, callback, payload)` for proven custom callbacks
  such as `CrystalFound`;
- `Complete(ctx, CompletionInput) FinalSnapshot`;
- `Snapshot()`.

Selection, evaluation, state mutation, and world commands remain
transport-neutral. `server/zone/objective/raknet103` alone maps publications
to build-103 records.

## Packet projection boundary

Keep these domain publications:

```text
Initialized { records }
Updated     { objective ID, player selector, status, tokens,
              voiceover intent, notification intent }
Added       { record }
Completed   { records, opaque result fields }
```

The RakNet adapter:

- validates count `<=255`;
- converts signed token bits to `uint32` without changing the low 32 bits;
- expands domain status to the exact wire byte;
- emits `0xb7`, `0xb8`, `0xb9`, or `0xca`;
- never selects objectives or evaluates medals;
- never interprets Lua setter flags as notification intent without a proven
  mapping;
- never generates rewards.

The domain should not expose `raknet.ObjectiveRecord`, packet IDs, reliability,
ordering channels, or encoded byte slices.

## Tutorial versus generic ownership

### Move to `server/zone/objective`

- definition discovery/loading and FNV-1 validation;
- arbitrary ordered selected sets rather than the fixed five IDs;
- Lua runtime creation and Init execution;
- initial state and active-player handling;
- token/state ledger and snapshots;
- all-player expansion;
- event subscription and dispatch;
- status evaluation at completion;
- initialization/update/add/completion publications;
- interaction/destructible/enemy census queries;
- damage, heal, death, modifier, obelisk, full-clear, and custom callback
  support;
- rejoin snapshot behavior;
- content-backed selection inputs and injected compatibility policy.

Delete the generic special cases `ObeliskID`, `NewObelisk`, the assumption that
a valid snapshot has exactly one record, and selection based only on the
presence of loot obelisks.

### Remain tutorial-owned

- the Cryos route and its accepted-event timing;
- tutorial enemy/object/marker census and checkpoint reconstruction until the
  whole tutorial uses the shared `zone.Zone`;
- the Ride lesson's compatibility presentation;
- the decision to use a compatibility objective profile when the tutorial
  level has no authored binding;
- tutorial completion through `0xc8`;
- tutorial-specific restart, XP, unlock, voice, and cinematic behavior.

The current Ride lesson sends an `0xb8` for `DoDamageOften` with Gold,
`(1,2,3)`, and `is_shown=true`, then sends a silent Gold update after the first
accepted Ride to clear the hint. Live testing proves this clears the build-103
arrow, but packaged `DoDamageOften` does not define a Ride lesson. Treat it as
a tutorial compatibility presentation referencing an objective publication;
do not bake it into the generic objective definition or infer it from Lua.

Tutorial enemy deaths and obelisk uses are tutorial event sources today, but
their effect on selected objectives is generic. The current 30-enemy
denominator is darkspin's tutorial population census, not an objective rule
and not a retail director constant.

## Implementation sequence

1. Add importer tests for the 13 definitions, three pools, source hashes,
   locale links, and `PlayerHealthDrain` mismatch.
2. Add the normalized tables and a read-only content adapter.
3. Introduce immutable `Definition`, `Pool`, and `Selection` domain types.
4. Generalize `Session` from one obelisk objective to an arbitrary ordered
   selected set.
5. Move the current tutorial Lua runtime/state creation into generic
   `Session.Initialize`.
6. Expand semantic events and host operations required by all selectable
   definitions. Fail selection before zone publication when a selected
   definition requires an unsupported native.
7. Add a named `Build103CompatibilitySelector` that reproduces the current
   five-record behavior without claiming authored provenance.
8. Load both campaign and tutorial through the same zone objective factory.
9. Keep the Ride lesson as a tutorial publication overlay.
10. Add final status evaluation and immutable result snapshots; continue to
    zero the 16 unknown `0xb9` result bytes.
11. Replace the compatibility selector only after a retail trace or stronger
    server-selection source is recovered.

Minimum tests:

- all importer counts, keys, hashes, flags, labels, pool duplicates, and
  ordinals;
- `InProgress=0`, `Failed=1`, Bronze/Silver/Gold `2/3/4`;
- Init after census and before `0xb7`;
- byte-exact `0xb7`, `0xb8`, `0xb9`, and `0xca`;
- shared versus per-player counters;
- all eight event-mask bits and unsupported-event rejection;
- each generic predicate pattern, including lower-is-better objectives;
- `LootCrystals` semantic drop and `CrystalFound`;
- reload/rejoin replacement with no cross-zone Lua private state;
- tutorial compatibility selection and Ride overlay kept separate;
- no account repository write during ordinary initialization/progress.

## Explicit gaps

1. Retail server objective-selection algorithm and RNG.
2. Exact mapping of affix arrays 0/1/2 to authored severity names.
3. Per-level override fields beyond the native default; the tutorial has none.
4. Why `FinishLevelQuickly` is registered but absent from both ordinary pools.
5. Retail initial state byte and whether the client should receive
   `InProgress=0` before evaluation.
6. Exact meanings of both Lua setter booleans and how notification/voice intent
   reaches `0xb8`.
7. Exact native payload names for every event slot, especially Heal,
   ModifierCreated, and Death attribution.
8. Whether the `PlayerHealthDrain` Damage/Death mismatch was dead content,
   relied on non-mask dispatch, or was an authored bug.
9. Selection-time meaning of `timeObjectiveRequires`.
10. Hidden Bonus selection/reveal rules and its relationship to active affixes.
11. `0xca` retail use cases and ordering relative to the five fixed UI rows.
12. Final `0xb8` versus `0xb9` ordering.
13. Semantics of the 16 trailing `0xb9` result bytes.
14. Conversion of objective medals into cash-out aggregate medal counts,
    rarity, DNA, XP, or reward cards.
15. A retained retail tutorial or campaign objective trace. Current traces
    establish darkspin compatibility only.

These gaps do not block content-driven definitions and generic instantiation.
They require conservative, named server policies rather than invented content
columns.
