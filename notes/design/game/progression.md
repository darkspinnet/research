# Progression and campaign reconstruction

This is a working game-design document for rebuilding Game's persistent
progression. It deliberately separates facts from compatibility defaults. The
client build examined is Game `5.3.0.103`, SHA-256
`3C7248A4C6614450290A66C22AB7FCDA86052B29A7B555E834D7C23F9B73EE5B`.

## Evidence labels

| Label | Meaning |
| --- | --- |
| **Binary-confirmed** | Directly visible in build 103 strings or x86 control flow. Addresses are build-specific. |
| **Data-confirmed** | Present in extracted `bin/server/data` assets. Field names produced by the extractor can still be imperfect. |
| **Corroborated** | Contemporary coverage or community documentation agrees with local evidence. |
| **Inferred** | Best current interpretation of multiple facts; needs a client/server trace. |
| **darkspin current** | Existing emulator behavior, not a claim about the retail service. |
| **Unknown** | No trustworthy value has been recovered yet. |

## Executive model

Game has several distinct progression concepts that should not be folded
into one integer:

1. The client wire field `new_player_progress` is an onboarding state machine
   for unlocking ship rooms and tutorial UI. darkspin calls the domain and
   storage value `onboarding_progress`; the legacy name remains only at the
   protocol boundary.
2. `chain_progression` is permanent campaign advancement. It gates the next
   planet in a 72-stage chain.
3. A chain run has its own temporary progression. Retail starts with capacity
   two; legacy data suggests purchases extend it to five before cashout.
4. Account `level` and `xp` are player progression and drive hero reward
   milestones and suggested difficulty.
5. `creature_rewards` is a spendable count of hero unlock choices.
6. `dna` is the persistent currency spent on account upgrades and associated
   with parts/loot economy.
7. Creatures and inventory parts are persistent collections, not implied by
   the number of templates available to the server. Parts belong to the
   account collection and can be reassigned between compatible heroes.

The account's cap and `unlock_*` fields are independent too. The cap fields
appear alongside upsell/access-entitlement state in build 103 and are not
evidence of campaign advancement. The unlock fields represent individually
purchased account capabilities. Neither group should be derived from
onboarding progress or `chain_progression` without stronger client evidence.

The client account schema contains no scalar named “evolution points.” In this
build, the likely concepts meant by that phrase are XP, DNA, or
`creature_rewards`; they must remain separate in the domain model.

## New-account state: known and unknown

| Property | Retail finding | Confidence | Current darkspin behavior |
| --- | --- | --- | --- |
| Account level | Level 1 is the natural zero-state. Difficulty 1-1 expects avatar level 4, suggesting tutorial advancement precedes the first campaign map. | **Inferred** | `defaultAccount()` uses level 1. The local registration endpoint overrides it to level 100. |
| XP | No retail starter grant found. | **Unknown** | Domain default 0; local registration overrides to 10,000. |
| DNA/money | No retail starter grant found in server data or build 103. Zero is the safest compatibility default until a clean-account trace proves otherwise. | **Unknown** | Domain default 0; local registration overrides to 10,000,000. |
| Creature reward tokens | No retail initial balance found. Hero choices are shown during cashout/level milestones. | **Corroborated**, exact count **unknown** | Domain default 0. `UnlockCreature` consumes one token. |
| Permanent campaign progress | Zero initially; the first chain entry remains selectable because the client permits index `chain_progression + 1`. | **Binary-confirmed** | Domain default 0. |
| Onboarding progress (`new_player_progress` on the wire) | Starts at 0 and advances through numeric milestones described below. Tutorial completion is the `3000` threshold, while later room/editor/map onboarding continues toward `9000`. | **Binary-confirmed** | Domain default 0. |
| Heroes | Tutorial play begins with Blitz. Sage is the level-2 eligible hero; Goliath, Wraith, and Zrin become eligible at level 3. Differing contemporary tutorial accounts suggest the third hero was selected rather than fixed. | **Data-confirmed** eligibility; grant cadence **corroborated/inferred** | A normal domain user has no creatures. Old ReCap test registration unlocked all 100 templates; that is explicitly a test hack. |
| Parts | No trustworthy starter count or starter loadout found. The 2,408 records in the extracted JSON are a template catalog, not an inventory grant. | **Unknown** | A normal domain user has no parts. Old ReCap test registration copied the entire catalog into inventory; that is explicitly a test hack. |
| Squads | The release manual says the player starts with one three-hero squad and purchases additional squad slots with DNA. The account representation can still carry three records while two remain locked. | **Corroborated** retail behavior | `NewUser` creates three empty squads; business logic must derive locked state from `unlock_pve_decks`. |

The elevated values in [`Register`](../../server/sporenet/user.go) must not be
used as game-design evidence. The preceding C++ and C# ReCap servers contain
the same “unlock everything for testing” pattern, including every creature and
part template.

### Template catalog facts

[`creature_templates.json`](../../bin/server/data/creature/creature_templates.json)
contains 100 templates: 25 named heroes with Alpha, Beta, Gamma, and Delta
variants. Its first records are Blitz Alpha, Sage Alpha, Wraith Alpha, and
Goliath Alpha. This order is suggestive but does not by itself define a retail
grant order.

[`creature_parts_templates.json`](../../bin/server/data/creature/creature_parts_templates.json)
contains 2,408 part templates. All extracted records currently report level 5
and rarity 1, so this JSON is useful as an identifier catalog but not yet as
faithful progression tuning. “2,408 available templates” must never become
“2,408 starter parts.”

## Hero Editor, equipment, and appearance

A surviving Hero Editor video provides a useful player-facing description of
the collection and equipment loop. The observations below use the timestamps
supplied with the video summary. They are labeled **video-observed** until the
source URL is recorded and the corresponding client paths are traced.

| Time | Observed behavior | Local correlation |
| --- | --- | --- |
| 0:00-0:05, 2:16-2:24 | The player assembles and switches among squads of three heroes to support different play styles. | The client sends three creature IDs per PvE/PvP deck and build 103 has `api.deck.updateDecks`. |
| 0:48-0:53 | The collection contains unlockable heroes with distinct ability sets. | Build-103 player-class data and `api.creature.unlockCreature` confirm distinct templates and an explicit unlock operation. |
| 1:09-1:19 | Functional equipment is divided into Weapon, Hand, Foot, Offense, Defense, and Utility slots. | An official EA systems-designer interview independently describes six equipment slots. Player-class data also marks whether a hero supports hands and feet. |
| 1:26-1:32 | Equipping a part changes gameplay statistics, including Health, Power, Strength, and Dexterity. | Creature responses carry gear score, item points, stats, and ability-stat values; authored class attributes contain these base-stat concepts. |
| 1:35-1:47 | Hovering a candidate part previews its differences from the currently equipped item, including stat and ability changes. | This is client presentation behavior; the authoritative modifier calculation and exact comparison payload still need tracing. |
| 1:47-2:04 | A functional part can have its statistical properties stripped so its model can be used as a cosmetic Detail. | The build-103 part parser recognizes `is_flair`; the account has editor-flair-slot upgrades. The exact conversion request is not yet confirmed. |
| 2:06-2:14 | Skin mode has separate Skin Coat and Skin Detail appearance layers. | `api.creature.updateCreature` submits serialized parts plus thumbnail/large renders and CRCs, but the individual skin fields have not yet been decoded. |
| 2:24-2:28 | An earned part is available to the player's hero collection rather than permanently bound to the hero that acquired it. | A part carries a mutable `creature_id`, supporting reassignment between heroes. |

### Meaning of shared parts

The video does **not** establish that one inventory instance can be equipped by
multiple heroes simultaneously. Build 103 recognizes one `creature_id` on each
part, which instead supports an account-owned item with zero or one current
functional assignment. The safe server rule is therefore:

1. a loot roll creates one stable inventory-item instance owned by the account;
2. any compatible owned hero may equip that instance;
3. equipping it elsewhere atomically removes its previous assignment;
4. the item is never copied merely because another hero equips it.

This distinction matters for trading, selling, concurrent editor sessions, and
preventing duplicated stats.

### Functional gear versus cosmetic detail

The editor exposes two related but separate representations:

- **functional equipment** occupies one of the six gear slots and contributes
  modifiers, gear score, item points, and the hero's equipment-derived level;
- **Detail/flair** preserves a part's visual rigblock but contributes no combat
  statistics and consumes cosmetic placement capacity instead;
- **Skin Coat/Skin Detail** changes texture/color presentation and should not
  alter combat state.

The Detail operation appears to be a state transition on an owned part, not a
second visual reference layered on top of the same functional instance. Before
implementing it, trace whether conversion is irreversible, whether it clears
prefix/suffix modifiers, whether it changes sale value, and which
`api.inventory.updatePartStatus` operator performs the transition.

### Server implications and current gaps

Equipment mutation belongs in guarded business logic, not in the XML adapter.
An equip operation must validate account ownership, stable part identity, slot
category, hero hand/foot compatibility, exclusivity, and editor-slot capacity;
then recalculate authoritative stats, gear score, item points, and displayed
hero level in the same persistence transaction.

The comparison preview should be a read-only domain query over the same
modifier calculator used by equip. The client may render the preview, but it
must not authoritatively submit the resulting stats. Appearance blobs and
renders can be accepted as presentation data only after the functional
loadout has been validated independently.

darkspin is not yet capable of enforcing this model:

- `Part` has no stable inventory-item ID; SQLite identifies parts by slice
  position even though build 103 sends `part_id` for status mutations;
- a part has no decoded six-slot category;
- `Creature` stores aggregate stats and gear score but no authoritative
  functional-slot loadout or separate cosmetic-detail placements;
- `api.inventory.updatePartStatus` persists the exact captured operator-free
  Create Detail status-4 batch; equip, unequip, sale, and other status/operator
  meanings remain unsupported;
- `api.creature.updateCreature` accepts client-computed aggregate fields in the
  protocol but its guarded server-side equipment semantics are not implemented;
- the part response currently omits recognized fields including stable `id`,
  `is_flair`, and `creature_id`.

These are prerequisites for persistent progression because equipment is what
levels an individual hero. Crogenitor XP progression can work independently,
but a faithful hero-level and multiplayer power model cannot.

## First-session onboarding

The consolidated milestone, scene, asset, launch-DLL, and capture reference is
[`../tutorial.md`](../tutorial.md). This section retains the progression-level
summary used by the wider campaign model.

Build 103 sends `new_player_progress` through the HTTP game endpoint. Static
call analysis recovers this milestone graph:

| State transition | Client event currently identified | Confidence |
| --- | --- | --- |
| 0 -> 1000 | Initial spaceship/room state | **Binary-confirmed** |
| 1000 -> 2000 | Early onboarding step | **Binary-confirmed**; exact presentation unknown |
| around 3000 | Emits `NewPlayerArrivedCollectionRoom` | **Binary-confirmed** |
| 3000 -> 4000 | Collection-room path | **Binary-confirmed** |
| 4000 -> 5000 | Editor-room transition path | **Binary-confirmed** |
| 5000 -> 6000 | Launches `SP_Editor` | **Binary-confirmed** |
| 6000 -> 6500 | Emits `NewPlayerUnlockedMapRoom` | **Binary-confirmed** |
| 6500 -> 6800 | Planet/map initialization | **Binary-confirmed** |
| 6800 -> 8000 | Planet/map initialization | **Binary-confirmed** |
| 8000 -> 9000 | First progressed campaign/planet-screen path | **Binary-confirmed**; exact account-level predicate needs a live trace |

The room controller also emits `NewPlayerUnlockedEditorRoom` around the
5000-stage flow. This proves collection, editor, and map access are onboarding
unlocks rather than simple account-level gates.

`new_player_inventory` is submitted alongside the progress field. The client
sets it to 1 after observing a non-default creature/collection condition, but
its exact retail meaning remains **unknown**.

Server implication: accept only legal forward transitions in business logic,
persist the two fields independently, and treat repeated client submissions as
idempotent. Do not let an HTTP payload write arbitrary account state directly.

## Hero acquisition

Build 103 contains a complete `api.creature.unlockCreature` request path. The
UI resolves the selected creature-template noun and submits it. The cashout UI
also populates `numCreatureUnlocks`, hero images, classes, genetic types, and
descriptions. Together with contemporary descriptions of new hero choices at
level milestones, this supports the following model:

1. Tutorial/onboarding grants or makes Blitz playable.
2. Progression awards one or more `creature_rewards` choices at specific level
   milestones.
3. The client presents an eligible hero pool.
4. Selecting a hero calls `api.creature.unlockCreature`.
5. The server validates eligibility, consumes one reward token, creates the
   persistent creature, and returns the updated state.

darkspin already guards step 5 in
[`server/sporenet/user_features.go`](../../server/sporenet/user_features.go),
but the milestone award schedule and eligible-pool rules are not recovered.
The server should not grant all 100 templates while those rules are unknown.

Contemporary sources identify Blitz as the tutorial/beginning hero and Sage as
the second hero. Local player-class assets make Sage the sole level-2 template,
then make Goliath, Wraith, and Zrin eligible at level 3. Contemporary tutorial
accounts exited with different third heroes, supporting a level-3 selection
from that pool. The exact ownership snapshots and reward-token changes still
need a clean trace.

The full player-facing cadence and all 100 authored eligibility levels are in
[`leveling-experience.md`](leveling-experience.md).

### Starter hero: Blitz Alpha

The first hero's authored combat data is recoverable even though the flattened
creature-template JSON contains zeroes for most of it. Blitz Alpha is template
noun `1667741389` (`0x6367B6CD`). Its noun resolves through
[`PC_EL_Rogue.Noun.xml`](../../bin/server/data/noun/PC_EL_Rogue.Noun.xml)
to [`PC_EL_Rogue.playerClass.xml`](../../bin/server/data/playerclass/PC_EL_Rogue.playerClass.xml)
and then
[`PC_EL_Rogue.ClassAttributes.xml`](../../bin/server/data/classattributes/PC_EL_Rogue.ClassAttributes.xml).

| Property | Authored value |
| --- | --- |
| Display name | Blitz, the Storm Striker |
| Variant | Alpha |
| Unlock level | 1 |
| Genesis / class | Plasma Ravager |
| Primary attribute | Dexterity |
| Weapon damage | 4–12 physical |
| Strength | 14 |
| Dexterity | 23 |
| Mind | 13 |
| Base health field | 200 |
| Base mana field | 100 |
| Base physical-defense field | 150 |
| Base energy-defense field | 50 |
| Base critical field | 100 |
| Combat speed | 8.5 |
| Non-combat speed | 8.5 |

The player-facing unmodified values are 220 Health, 113 Power, 288 Dodge
Rating, 128 Resist Rating, and 192 Critical Rating. These values are both
reported by the contemporary community stat table and reproduced from the
local fields plus
[`MagicNumbers.MagicNumbers.xml`](../../bin/server/data/magicnumbers/MagicNumbers.MagicNumbers.xml):

```text
health   = 200 + (14 - 10) * 5 = 220
power    = 100 + 13 * 1        = 113
dodge    = 150 + 23 * 6        = 288
resist   =  50 + 13 * 6        = 128
critical = 100 + 23 * 4        = 192
```

The ten-point Strength baseline in the health formula is corroborated across
all four Blitz variants: their authored Strength values reproduce the published
Health values exactly. Its source has not yet been located in the extracted
magic-number asset, so the formula is **strongly corroborated**, not yet
binary-confirmed.

The class-attribute block also contains `max*` fields—Health 400, Mana 125,
Strength 28, Dexterity 46, Mind 26, physical defense 375, energy defense 275,
and critical 500. These are not Blitz's starting UI stats. They appear to be
growth/interpolation limits, but their exact runtime use still needs executable
analysis.

The authored ability keys are `LightningRogueBasic`,
`LightningRogueActive`, `PlasmaRandom_LightningBall`, and
`LightningRogueSupport`. The live build-103 tooltip names Blitz Alpha's basic
attack Voltic Slash (an older public source called it Voltic Strike); the kit is
Ride the Lightning, Electron Sphere, Plasma Wreath, and the Deadly Precision
passive. Mapping every internal key to its public name and exact coefficient
still requires the missing ability assets or client disassembly.

darkspin currently defines an in-memory `Attributes` type and packet ID `0x96`
(`AttributeDataUpdate`), but does not yet resolve the player-class chain above
or emit a hero's authored spawn attributes. These recovered values therefore
identify a concrete gameplay implementation gap rather than behavior already
active on the Go server.

## DNA and upgrade economy

DNA is persistent currency. darkspin currently carries a 52-ID legacy upgrade
cost table covering catalyst slots, stats, inventory, PvE/PvP deck access, fuel
tanks, and editor flair. The table includes costs from 200 DNA to 20,000,000
DNA. It is useful for compatibility testing, but it has not yet been
independently matched instruction-for-instruction against build 103.

The extracted
[`UnlocksTuning.UnlocksTuning.xml`](../../bin/server/data/unlockstuning/UnlocksTuning.UnlocksTuning.xml)
has pointer-like values in fields that should be costs or levels. Those numbers
are an extractor failure and are not valid economic tuning.

The current reconstruction therefore knows the kinds of DNA sinks but not yet:

- clean-account starting DNA;
- per-planet DNA rewards;
- cashout multipliers and loss rules;
- part sale/refund formulas;
- exact account-level XP thresholds;
- which upgrade IDs become visible at which levels.

## Campaign structure

### Indexing and unlock rule

[`ChainLevels.ChainLevels.xml`](../../bin/server/data/chainlevels/ChainLevels.ChainLevels.xml)
contains exactly 72 ordered entries. The corresponding
[`DifficultyTuning.DifficultyTuning.xml`](../../bin/server/data/difficultytuning/DifficultyTuning.DifficultyTuning.xml)
contains exactly 72 health multipliers, 72 damage multipliers, and 72 expected
avatar levels. This establishes 18 threat bands with four stages each.

The client formats a non-tutorial level index as:

```text
major = ((levelIndex - 1) / 4) + 1
minor = ((levelIndex - 1) % 4) + 1
label = major-minor
```

Static analysis of the planet screen (`sub_527DD0`, especially the branch near
`0x005296F4`) compares the candidate level index with permanent
`chain_progression + 1`. Unless an unlock-all/test flag is set, indexes greater
than that are marked locked. Therefore:

```text
chain_progression = N  =>  highest selectable level index = N + 1
```

With zero permanent progress, 1-1 is accessible. A successful completion must
advance permanent progress monotonically; it must not be confused with the
temporary current-run counter.

### First story pass: 1-1 through 6-4

The asset order and cinematic/voice cues are local data. Player-facing map
names are corroborating community labels for those threat slots.

| Threat | Level asset | Planet / map | Story cue in chain data |
| --- | --- | --- | --- |
| 1-1 | `zelems_1.Level` | Zelem — Floating Isles | `cam_fmv_02_zelems` |
| 1-2 | `zelems_3.Level` | Zelem — Gnarled Plateau | — |
| 1-3 | `nocturna_4.Level` | Nocturna — Twilight Summit | `cam_fmv_03_nocturna` |
| 1-4 | `nocturna_1.Level` | Nocturna — Shadowglades | Boss slot |
| 2-1 | `verdanth_1.Level` | Verdanth — Whispering Forest | `cam_fmv_04_verdanth` |
| 2-2 | `verdanth_3.Level` | Verdanth — Deathly Everglades | — |
| 2-3 | `zelems_2.Level` | Zelem — Outer Rings | Reinfection voice-over |
| 2-4 | `zelems_4.Level` | Zelem — Chaos Fields | Boss slot |
| 3-1 | `cryos_4.Level` | Cryos — Frozen Precipice | `cam_fmv_05_cryos` |
| 3-2 | `cryos_3.Level` | Cryos — Frigid Caverns | — |
| 3-3 | `verdanth_2.Level` | Verdanth — Fertile Strand | Reinfection voice-over |
| 3-4 | `verdanth_4.Level` | Verdanth — Shrouded Marsh | Boss slot |
| 4-1 | `infinity_2.Level` | Infinity — Uranium Heights | `cam_fmv_06_infinity` |
| 4-2 | `infinity_3.Level` | Infinity — Terminal Haven | — |
| 4-3 | `cryos_1.Level` | Cryos — Glacial Rift | Reinfection voice-over |
| 4-4 | `cryos_2.Level` | Cryos — Arctic Ridge | Boss slot |
| 5-1 | `nocturna_3.Level` | Nocturna — Spectral Forest | Reinfection voice-over |
| 5-2 | `nocturna_2.Level` | Nocturna — Ruby Gorge | — |
| 5-3 | `infinity_1.Level` | Infinity — Core Extractor | Reinfection voice-over |
| 5-4 | `infinity_4.Level` | Infinity — Fr'agmented Peak | Boss slot |
| 6-1 | `scaldron_2.Level` | Scaldron — Plains of Desolation | `cam_fmv_07_scaldron` |
| 6-2 | `scaldron_1.Level` | Scaldron — Desert Necropolis | — |
| 6-3 | `scaldron_3.Level` | Scaldron — Sunken Monastery | — |
| 6-4 | `scaldron_4.Level` | Scaldron — Perceptory | Boss slot; `cam_fmv_08_epilogue` |

The older C++ ReCap table swaps `scaldron_1` and `scaldron_2` at 6-1/6-2.
Current extracted chain data explicitly places `_2` first, so `_2`, `_1` is
the compatibility order unless a live trace disproves it.

### Reused-map pass: 7-1 through 12-4

| Threat | Asset | Threat | Asset | Threat | Asset | Threat | Asset |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 7-1 | `cryos_1` | 7-2 | `cryos_2` | 7-3 | `zelems_2` | 7-4 | `zelems_3` |
| 8-1 | `infinity_1` | 8-2 | `infinity_3` | 8-3 | `verdanth_2` | 8-4 | `verdanth_3` |
| 9-1 | `scaldron_2` | 9-2 | `scaldron_4` | 9-3 | `cryos_4` | 9-4 | `cryos_3` |
| 10-1 | `nocturna_1` | 10-2 | `nocturna_2` | 10-3 | `infinity_4` | 10-4 | `infinity_2` |
| 11-1 | `zelems_1` | 11-2 | `zelems_4` | 11-3 | `nocturna_4` | 11-4 | `nocturna_3` |
| 12-1 | `verdanth_1` | 12-2 | `verdanth_4` | 12-3 | `scaldron_3` | 12-4 | `scaldron_1` |

### Reused-map pass: 13-1 through 18-4

| Threat | Asset | Threat | Asset | Threat | Asset | Threat | Asset |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 13-1 | `nocturna_3` | 13-2 | `infinity_4` | 13-3 | `cryos_2` | 13-4 | `verdanth_1` |
| 14-1 | `scaldron_1` | 14-2 | `zelems_3` | 14-3 | `nocturna_4` | 14-4 | `infinity_3` |
| 15-1 | `zelems_2` | 15-2 | `verdanth_4` | 15-3 | `scaldron_4` | 15-4 | `cryos_1` |
| 16-1 | `verdanth_3` | 16-2 | `cryos_3` | 16-3 | `zelems_4` | 16-4 | `nocturna_2` |
| 17-1 | `infinity_2` | 17-2 | `scaldron_2` | 17-3 | `verdanth_2` | 17-4 | `zelems_1` |
| 18-1 | `cryos_4` | 18-2 | `nocturna_1` | 18-3 | `infinity_1` | 18-4 | `scaldron_3` |

The three 24-entry blocks each use every authored campaign asset exactly once.
That is deliberate campaign recycling, not missing map data.

### Difficulty bands

The index-aligned tuning gives the following expected avatar levels and enemy
multiplier ranges. “Health” and “damage” retain the extracted field names; the
precise runtime formula still needs confirmation.

| Threat band | Expected avatar levels (stages 1-4) | Health, first -> fourth | Damage, first -> fourth |
| --- | --- | --- | --- |
| 1 | 4, 5, 5, 6 | 0.60 -> 1.03 | 1.00 -> 1.16 |
| 2 | 6, 7, 8, 8 | 1.28 -> 1.53 | 1.39 -> 1.61 |
| 3 | 9, 10, 10, 11 | 1.90 -> 2.26 | 1.93 -> 2.24 |
| 4 | 12, 12, 13, 14 | 2.80 -> 3.34 | 2.52 -> 2.92 |
| 5 | 14, 15, 16, 16 | 4.14 -> 4.93 | 3.50 -> 4.05 |
| 6 | 17, 18, 18, 19 | 6.11 -> 7.28 | 4.86 -> 5.63 |
| 7 | 20, 21, 22, 23 | 9.03 -> 10.75 | 6.76 -> 7.87 |
| 8 | 23, 24, 25, 26 | 13.33 -> 15.88 | 9.52 -> 11.10 |
| 9 | 26, 27, 28, 29 | 19.69 -> 23.45 | 13.43 -> 15.66 |
| 10 | 29, 30, 31, 32 | 29.08 -> 34.63 | 18.95 -> 22.09 |
| 11 | 32, 33, 34, 35 | 42.94 -> 51.15 | 26.73 -> 31.16 |
| 12 | 35, 36, 37, 38 | 63.43 -> 75.55 | 37.70 -> 43.95 |
| 13 | 39, 40, 41, 43 | 93.68 -> 112.37 | 53.18 -> 62.45 |
| 14 | 44, 45, 46, 47 | 140.46 -> 168.48 | 76.19 -> 89.46 |
| 15 | 48, 49, 50, 51 | 210.60 -> 252.61 | 109.14 -> 128.15 |
| 16 | 52, 53, 54, 55 | 315.76 -> 378.75 | 156.34 -> 183.58 |
| 17 | 56, 57, 58, 59 | 473.44 -> 567.87 | 223.97 -> 263.00 |
| 18 | 0, 0, 0, 0 | 709.84 -> 851.43 | 320.86 -> 376.77 |

The zero expected levels for band 18 are present in the asset and may be an
intentional endgame sentinel or unfinished tuning. They should not be silently
replaced without observing client behavior.

## Chain runs and cashout

Permanent campaign progression and current-run progression are different. The
legacy packet model matches the build 103 planet-screen object closely:

| Planet-screen field | Meaning |
| --- | --- |
| `+0x30` | level resource/noun |
| `+0x34` | permanent chain level index |
| `+0x38` | star level |
| `+0x3C` | time remaining |
| `+0x40` | progress within the current run |

The release manual confirms that a new player can chain two planets and must
purchase a chain upgrade to continue farther. The legacy account table has
three `unlock_fuel_tanks` purchases and the gameplay reference clamps current
run progress to five, supporting capacities 2 -> 3 -> 4 -> 5. One contemporary
review instead describes an eventual unlimited chain, so the build-103 maximum
still needs a live trace.

## Recommended provisional server rules

These are implementation choices for reaching an end-to-end prototype, not
claims about the lost retail service:

1. Create a normal account at level 1, XP 0, DNA 0, chain progression 0,
   creature rewards 0, onboarding 0, no parts, and no elevated access hacks.
2. Make tutorial completion a domain command that grants Blitz exactly once.
   Keep later hero grants configurable until their milestones are recovered.
3. Unlock 1-1 for progress 0. On authoritative successful completion of level
   index N, advance permanent progress to at most N, never backward and never
   from a client-provided account payload alone.
4. Keep current chain-run progress in game/session state. Start with capacity
   two and derive later capacity from purchased upgrades; use five as the
   provisional maximum until a trace resolves the conflicting public account.
5. Treat campaign order and difficulty tuning as loaded data, not hard-coded
   transport behavior.
6. Persist DNA, XP, hero rewards, creatures, parts, squads, onboarding, and
   permanent chain progress transactionally.
7. Expose only client DTO fields through XML/TDF/RakNet adapters. Internal
   eligibility, reward rolls, anti-cheat state, and transaction metadata stay
   in the server model.

## Highest-value research gaps

The next clean-account capture should record HTTP and RakNet from first login
through tutorial completion, first hero selection, first part grant, 1-1
completion, and cashout. Specifically capture before/after snapshots of:

1. DNA, XP, level, `creature_rewards`, and all `unlock_*` values;
2. creature list and squad membership after every onboarding milestone;
3. part inventory after tutorial and first cashout;
4. `new_player_progress` and `new_player_inventory` submissions;
5. level index, current-run progress, permanent `chain_progression`, and reward
   payloads;
6. the first request/response that awards a hero choice;
7. 6-1 selection, to settle the Scaldron `_1`/`_2` discrepancy;
8. 7-1 and 18-1 selection, to confirm post-story threat labels and band-18
   handling;
9. an equip, unequip, and cross-hero part reassignment, including the part's
   stable ID, `creature_id`, status, creature gear score, and stats;
10. comparison-preview inputs and whether the client computes the delta locally;
11. functional-part to Detail conversion, including its operator, modifier
    fields, cost, reversibility, and flair-slot usage;
12. a Skin Coat and Skin Detail save, including the serialized creature fields
    and whether either layer affects functional stats.

## Sources

Primary local evidence:

- [`ChainLevels.ChainLevels.xml`](../../bin/server/data/chainlevels/ChainLevels.ChainLevels.xml)
- [`DifficultyTuning.DifficultyTuning.xml`](../../bin/server/data/difficultytuning/DifficultyTuning.DifficultyTuning.xml)
- [`creature_templates.json`](../../bin/server/data/creature/creature_templates.json)
- [`creature_parts_templates.json`](../../bin/server/data/creature/creature_parts_templates.json)
- [`PC_EL_Rogue.playerClass.xml`](../../bin/server/data/playerclass/PC_EL_Rogue.playerClass.xml)
- [`PC_EL_Rogue.ClassAttributes.xml`](../../bin/server/data/classattributes/PC_EL_Rogue.ClassAttributes.xml)
- [`MagicNumbers.MagicNumbers.xml`](../../bin/server/data/magicnumbers/MagicNumbers.MagicNumbers.xml)
- [`game-5.3.0.103-first-pass.md`](../confirmed/game-5.3.0.103-first-pass.md)
- [`game-5.3.0.103-progression.md`](../confirmed/game-5.3.0.103-progression.md)
- IDA evidence logs under `bin/game/logs/ida-progress-*.log`
- darkspin account defaults in [`account.go`](../../server/sporenet/account.go)
- darkspin registration overrides in [`user.go`](../../server/sporenet/user.go)
- darkspin hero and upgrade guards in [`user_features.go`](../../server/sporenet/user_features.go)

Corroborating public sources:

- [Game Wiki: Planets & Places](https://gamegame.fandom.com/wiki/Planets_%26_Places)
- [GameSpot: Game review](https://www.gamespot.com/reviews/game-review/1900-6310899/)
- [GameSpot: review in progress](https://www.gamespot.com/articles/review-in-progress-game/1100-6310415/)
- [Game Wiki: Sage strategy](https://gamegame.fandom.com/wiki/Hero_Strategy%3A_Sage)
- [Game Wiki: Blitz strategy and base stats](https://gamegame.fandom.com/wiki/Hero_Strategy%3A_Blitz)
- [EA: Game in stores](https://www.ea.com/en-ca/news/game-in-stores-now)
- [EA: interview with Game senior systems designer](https://www.ea.com/news/game-beams-into-stores-today)
- [PC Gamer: Game preview](https://www.pcgamer.com/game-preview/)

Additional observational evidence:

- User-supplied Hero Editor video summary with observations at 0:00-2:28;
  source URL not yet recorded.
