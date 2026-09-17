# Build-103 1-1 result record fields

## Scope and evidence labels

This note maps the subtype-0 bodies after `A9 00` and `AB 00` in build
`5.3.0.103`. Offsets are relative to the first byte after the subtype byte,
not to the RakNet application packet ID.

- **Native exact** means the width, offset, stride, comparison, or UI property
  is directly read by `bin/game/GameBin/Game.c`.
- **Semantic inference** means the name follows from the consumer, packaged
  content, or the recorded 1-1 presentation, but is not a recovered C++ symbol.
- **Current reconstruction** means the value is authored by the existing Go
  encoder. It is not evidence that retail used that name or placement.

The principal native path is:

```text
A9 00 -> sub_449090 -> sub_A87BF0(..., state+20, 337, ...)
      -> sub_527D80(state+20, state+97)
      -> sub_527C80 -> cPlanetScreen+48..384
      -> sub_527DD0 -> PlanetScreen SetNPCData / InitFor*Planet

AB 00 -> sub_449410 -> sub_A87BF0(..., stack, 712, ...)
      -> sub_409F10 -> sub_409D60
      -> qmemcpy(cCashOutUI+8, stack, 0x2c8)
      -> sub_407540 -> MaxisCashOut properties / OnGameDataLoaded
```

`sub_527C80` preserves all 337 A9 bytes without a layout conversion: it copies
body bytes `0..75` to `cPlanetScreen+48..123`, byte `76` to `+124`, and bytes
`77..336` to `+125..384`. `sub_409D60` likewise preserves all 712 AB bytes at
`cCashOutUI+8..719`.

## `A9 00`: 337-byte chain-vote record

### Fields with downstream native reads

| Body offset | Width/type | Native use | Best semantic name | Evidence |
| ---: | --- | --- | --- | --- |
| `0x000` | `u32` asset hash | `sub_527DD0` calls `sub_9C5FF0` and resolves level presentation data | current/completed level resource, e.g. `zelems_1.Level` | Native exact use; name inferred |
| `0x004` | `u32` | passed to `sub_9BCC70`, indexed into chain/reward tuning, and exported as `LABS_local_LEVEL_INT_DIFFICULTY` | offered/next difficulty coordinate; after completing 1-1 this is `2`, which presents the 1-2 recommendation | Native exact use; reward math and visible tuning range corroborate the meaning |
| `0x008` | `u32` | selects the zero/nonzero header token and participates in reward-level calculations | star/reward tier | Native exact use; name inferred and matches the existing `StarLevel` label |
| `0x00c` | `f32` | passed to `InitForFirstPlanet` or `InitForNewPlanet` | level time remaining, in the UI's time unit | Native exact type/use; name inferred from the visible `02:12` timer |
| `0x010` | `u8` | zero selects `InitForFirstPlanet`; nonzero selects `InitForNewPlanet`; also bounds progression/reward calculations | number of planets already represented in this active chain result | Native exact flag/branch; name inferred |
| `0x011 + 4*i` | `u32[6]`, stride `4` | six `sub_9C5FF0` noun lookups are converted into `SetNPCData` rows | preview-planet enemy noun hashes | Native exact array; the 1-1 recording shows six 1-2 enemy cards |
| `0x039`, `0x03d`, `0x041` | three `u32` | loaded by `cChainVotingState` before `sub_52CB60` | first transition/presentation hash triplet | Native exact reads; individual movie/voice names are not proved |
| `0x049` | `u32` | compared with the `0x010` chain count and sentinel `5`; contributes to one `InitForNewPlanet` boolean | unlock/completion ordinal guard; the current reconstruction sends sentinel `5` so Continue Fight remains available | Native exact comparison; user-directed unlock policy |
| `0x0d9`, `0x0dd`, `0x0e1` | three `u32` | second transition/presentation triplet; nonzero `0x0d9` supersedes `0x039` in the state path | next transition/presentation hash triplet | Native exact reads; individual movie/voice names are not proved |
| `0x0e9` | `u32` asset hash | resolved through `sub_9C5FF0`; its localized level name feeds the new-planet presentation | offered/next level resource, e.g. `zelems_3.Level` | Native exact use; next-level meaning corroborated by the walkthrough |

The byte at `0x010` is a count/ordinal, not a C++ `bool`: the client performs
unsigned comparisons and increments it for next-planet reward calculations.
The `0x049` value is also not safely nameable as `isCompleted`; it is read as a
32-bit ordinal, and the native condition is approximately
`field <= chainCount && field != 5`.

The transition fields are intentionally described as opaque hashes. The state
loads the two triplets 160 bytes apart and passes the selected presentation ID
into the room/state transition path, but this decompilation does not establish
which member is movie, voice, or fallback. The exact repeated offsets are the
safe contract.

### Bytes not semantically consumed on this path

The complete body is retained, but no downstream read was found in the voting
state, `cPlanetScreen`, Planet Screen callbacks, or its shutdown path for:

```text
0x029..0x038  (16 bytes)
0x045..0x048  (4 bytes)
0x04d..0x0d8  (140 bytes)
0x0e5..0x0e8  (4 bytes)
0x0ed..0x150  (100 bytes)
```

These ranges must remain opaque/reserved. Copy length alone does not make them
zero-fill fields, and the apparent 160-byte spacing of the transition triplets
does not justify inventing a complete repeated C struct.

### Native hashes and the recorded 1-1 meaning

The six values at `0x011` are 32-bit noun/asset identifiers, not row numbers or
strings. In the 1-1 walkthrough they drive the six visible creature cards for
Gnarled Plateau 1-2. The level hashes at `0x000` and `0x0e9` resolve to the
completed Floating Isles and offered Gnarled Plateau resources respectively.
The chain resource independently places `zelems_1.Level` at ordinal 0 and
`zelems_3.Level` at ordinal 1.

Planet Screen localization uses these exact locale IDs while building the
presentation:

| ID | Native branch/use |
| ---: | --- |
| `0x0b2f2a3d` | zero-tier header |
| `0x0bbbc8af` | nonzero-tier header |
| `0x0b2f2a40` | reward-range/chance text |
| `0x0b2f2a43` | reward-count text, used for current and next chain counts |
| `0x0b2f2a45` | squad/enemy descriptive text |

The locale IDs identify presentation templates; they are not body fields.

### Difference from the current server reconstruction

`server/raknet/application.go` currently emits a compatibility envelope with:

| Offset(s) | Current Go label/value | Native result of this pass |
| --- | --- | --- |
| `0x000` | `Level` | consistent with the current/completed level resource use |
| `0x00c` | fixed `30*60*1000` float | correct width, but retail 1-1 visibly presents the actual remaining time (`02:12` in the recording), not proof of that fixed value |
| `0x011..0x028` | `EnemyNoun[6]` | correct shape; these are the six preview-planet enemy cards |
| `0x029`, `0x02d` | `LevelNoun[2]` | no downstream native read found; names remain reconstruction only |
| `0x03d`, `0x041`, `0x045` | duplicated `IntroMovie`, then `IntroVoice` | native state reads `0x039`, `0x03d`, `0x041`; `0x045` is not read on the traced path |
| `0x0dd`, `0x0e1` | duplicated `NextMovie` | these are the second and third members of the native triplet beginning at `0x0d9` |
| `0x0e9` | the same `Level` as `0x000` | native use is the offered/next level; for the recorded first result this must resolve as `zelems_3.Level`, not `zelems_1.Level` |

The older `game.ChainVoteData` type is also not a wire contract. Its two
`PlanetData[4]` arrays do not match the offsets read by build 103 and it is not
used by the encoder.

## `AB 00`: 712-byte cashout record

### Top-level layout

| Body offset | Width/type | Native/UI meaning | Evidence |
| ---: | --- | --- | --- |
| `0x000` | `i32` | `numPlanetsCompleted` | Exact SWF property name |
| `0x004` | `i32` | `dnaAmount` | Exact SWF property name and locale token `0x09e832b3` |
| `0x008..0x013` | 12 bytes | opaque | No downstream read found |
| `0x014 + 4*p` | `i32[4]`, stride `4` | starting total XP for player `p` | Exact level/fill computation; name inferred from before/after role |
| `0x024 + 4*p` | `i32[4]`, stride `4` | final total XP for player `p` | Exact level/fill computation; name inferred from before/after role |
| `0x034 + 4*p` | `i32[4]`, stride `4` | gold medal count for player `p` | Exact `numGoldMedals` source |
| `0x044 + 4*p` | `i32[4]`, stride `4` | silver medal count for player `p` | Exact `numSilverMedals` source |
| `0x054 + 4*p` | `i32[4]`, stride `4` | bronze medal count for player `p` | Exact `numBronzeMedals` source |
| `0x064 + 4*p` | `i32[4]`, stride `4` | upper rarity split `A[p]` | Exact arithmetic; descriptive name inferred |
| `0x074 + 4*p` | `i32[4]`, stride `4` | lower rarity split `B[p]` | Exact arithmetic; descriptive name inferred |
| `0x084 + p` | `u8[4]`, stride `1` | `cashoutBonusGranted[p]` | Exact SWF property name; presentation flag |
| `0x088 + 0x90*p` | four player reward blocks, stride `0x90` | four candidate loot records per player | Native exact matrix shape |

`p` ranges from 0 through 3. The local current-player index is **not** in the
712-byte record: `sub_409D60` obtains it from client state and stores it at
`cCashOutUI+720`, immediately after the copied body.

The XP arrays are totals, not levels or percentages. For each valid player the
client calls `sub_9CEA00(totalXp)` to derive a level and `sub_9CEA40(level)` to
derive the lower XP boundary. It exports:

```text
startingPlayerLevel[p]   = level(startXp[p])
startingXpPercentFill[p] = percent within level(startXp[p])
playerLevel[p]           = level(finalXp[p])
xpPercentFill[p]         = percent within level(finalXp[p])
xpAmount                 = finalXp[0] - startXp[0]
```

The last scalar is hard-wired to array element zero in the native function;
the four-element arrays remain per-player.

The two rarity arrays are boundaries, not three independent probabilities.
The SWF receives:

```text
uniqueChances[p] = max(100 - A[p], 0)
rareChances[p]   = max(100 - B[p], 0)
epicChances[p]   = B[p]
```

For the per-player tooltip the client formats the ranges from those same two
values. The walkthrough's `Roll 1-65 for a Special Item` and `Roll 66-100 for
a Rarified Item` is therefore an observed boundary example, not evidence for a
universal 65/35 formula.

The 2026-08-23 paired reward screenshots cross-check an `A=35`, `B=0`
presentation: roll 47 belongs to the yellow Special band and roll 85
belongs to the orange Rarified band. Combined with the reward record's rarity
enum and the separately recovered yellow/orange/red cash-out family, this maps
Special to `PartUnique` (`4`), Rarified to `PartRareUnique` (`5`), and the
optional third band to `PartEpicUnique` (`6`). These are cash-out tiers, not
the ordinary white/green/blue/purple in-level drop tiers.

The same native publisher proves the medal tooltip arithmetic. It divides each
medal count by planets completed and reports a Rare Item Chance increase of
`1 * average Bronze + 3 * average Silver + 6 * average Gold`. The client does
not calculate the headline; the server supplies the final integer boundary.
The observed one-planet, one-Gold screen therefore supplies an exact six-percent
medal-derived chance, while its 35-percent headline came from the server's old
fixed boundary and is the mismatch being corrected.

### Player/loot matrix

Each player block is exactly `0x90` bytes and contains four records at a
`0x24` stride:

```text
loot(p, r) = 0x088 + 0x090*p + 0x024*r,  p=0..3, r=0..3
```

| Loot-relative offset | Width/type | Native meaning | Evidence |
| ---: | --- | --- | --- |
| `+0x00` | 8 bytes | opaque/unconsumed prefix | No cashout UI/controller read found |
| `+0x08` | `u32` asset hash | rigblock/item-base resource; zero means the slot is absent | Exact resource lookup and presence test |
| `+0x0c` | `u32` asset hash | suffix affix | Exact order passed into the native loot presentation object; name matched to build-103 `cLootData` |
| `+0x10` | `u32` asset hash | first prefix affix | Same |
| `+0x14` | `u32` asset hash | second prefix affix | Same |
| `+0x18` | `i32` | item level | Exact order and visible tooltip meaning |
| `+0x1c` | `i32` rarity enum | rarity | Exact order; equality with `6` drives `playerNEpicLoot[r]` |
| `+0x20` | `i32` | displayed roll | Exact `playerNRolls[r]` source |

Rarity `6` is build 103's epic-unique tier (`PartEpicUnique` in the current
repository). The SWF property's historical name `playerNEpicLoot` is narrower
than a generic "epic or better" test: the native client sets it only when the
recorded rarity equals exactly `6`.

`validPlayers[p]` is not transmitted as a separate flag. The client derives it
as `loot(p,0).rigblockHash != 0`. Consequently a player with no first reward
record is hidden even if later records contain data.

The native item-card callbacks consume the seven dwords at `+0x08..+0x20` to
build and inspect the loot presentation. The first eight bytes of every
`0x24` record survive the copy but are not read by the traced UI/controller
code. They may be server bookkeeping or identifiers, but assigning grant,
inventory, reference, or creation-time semantics would be speculation.

### Exact SWF output names and locale hashes

`sub_407540` publishes these exact property names before invoking
`OnGameDataLoaded`:

```text
currentPlayerIdx                 (client-local, not in AB)
numPlanetsCompleted
numGoldMedals / numSilverMedals / numBronzeMedals
rarityText / rarityChance
uniqueChances / rareChances / epicChances
dnaText / dnaAmount
xpText / xpAmount
player1Rolls .. player4Rolls
player1EpicLoot .. player4EpicLoot
player1RarityChances .. player4RarityChances
player1MedalTooltips .. player4MedalTooltips
validPlayers
playerLevel / xpPercentFill
startingPlayerLevel / startingXpPercentFill
cashoutBonusGranted
creatureUnlockText / creatureUnlockTypes
creatureUnlockClasses / creatureUnlockImages
numCreatureUnlocks
```

The directly embedded locale/token IDs are:

| ID | Native output |
| ---: | --- |
| `0x09e832b0` | `headerText` |
| `0x09e832b1` | `otherPlayerButtonText` |
| `0x09e94915` | `rarityChance` text |
| `0x09e832b3` | `dnaAmount` text |
| `0x09e95945` | `xpAmount` text |
| `0x0b7b3bdb`, `0x0b7b3bdc`, `0x0b7b3bdd` | three rarity-range tooltip labels |
| `0x0b7d29ce`, `0x0b7d29cf`, `0x0b7d29d0` | gold/silver/bronze medal tooltip labels |

These hashes select localized presentation strings. The item, affix, and level
hashes inside the records are separately resolved asset identifiers.

## Grant versus presentation semantics

The native evidence establishes a presentation record, not a grant command:

- `AB 00` is copied into `cCashOutUI`, converted into ActionScript properties,
  and followed by `OnGameDataLoaded`.
- Loot records carry enough hashes, item level, rarity, and roll to render the
  exact card and tooltip seen in the walkthrough. The client performs no
  inventory write or grant request from those fields in this path.
- `cashoutBonusGranted[p]` is an exact SWF name, but its only recovered effect
  here is presentation. It does not prove when the server consumed or persisted
  the bonus.
- Creature unlock choices are not transmitted as creature IDs in AB. For the
  current player, the client compares starting and final XP-derived levels,
  searches its local creature-unlock table for thresholds crossed, and exposes
  at most three choices. A later `api.creature.unlockCreature` request submits
  the selected template. Thus AB presents eligibility; the later account
  operation authorizes the chosen creature.
- The opaque eight-byte prefix of each loot record is not a safe durable-item ID.
  No read in the cashout UI or its callbacks gives it grant meaning.

Accordingly, replaying an identical AB record is naturally presentation-safe
on the client side, but idempotent persistence, inventory insertion, DNA/XP
commit, medal progression, and bonus consumption remain server-authority
operations outside this record's proven client contract.

## Walkthrough cross-check

`bin/video/walkthrough/1-1/frame-900.png` shows the fields supplied by A9:

- Floating Isles marked `MISSION COMPLETE` on the left;
- Gnarled Plateau 1-2 and six enemy noun cards on the right;
- `02:12` remaining;
- one level-11 collect reward versus two level-13 continue rewards.

This agrees with separate current and offered level hashes, the six-element
enemy array, the chain count branch, time field, and reward-tier calculations.
It also disproves treating the `0x0e9` preview level as necessarily identical
to the completed level. The visible medal totals `0`, `1`, and `4` are derived
by `sub_527DD0` from retained local objective state, not read from A9; they must
not be invented as additional fields in this record.

`bin/video/walkthrough/1-1/frame-915.png` shows the AB-driven presentation:
roll `39`, one rendered item card, item level `11`, rarity-range text, and the
Leto's Pale Guard tooltip. This agrees with the `0x24` loot record's roll,
rigblock/affix hashes, item level, and rarity, and with first-slot-derived
`validPlayers`. The video does not reveal the 712 raw bytes, the opaque loot
prefixes, a persistence acknowledgement, or the instant at which the item became
durable.

## Implementation boundary

This recovery is sufficient to describe every byte read by the build-103
Planet Screen and cashout UI/controllers. It is not evidence for values in the
opaque ranges, reward-generation formulas, or commit timing. An implementation
can use the exact offsets above, but should keep inferred labels and retail
authority policy separate and must not copy the current compatibility envelope
as if it were a recovered retail record.
