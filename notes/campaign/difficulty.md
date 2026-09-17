# Campaign difficulty runtime ownership

This note records build-103 client evidence. It distinguishes exact native
client behavior from policy that would have to be proved by a retail server
capture. The canonical sources are
`bin/game/GameBin/Game.c`,
`bin/game/GameBin/Game.exe`,
`bin/game/logs/difficulty-tuning.bin`, and
`bin/game/logs/labs-tuning.bin`.

## Result

Simulator `+8` is the client's current one-based difficulty selector. The
client initially sets it to `4`, later replaces it only from two inbound
message handlers, and accepts every signed input at or below `72`; inputs above
`72` become `72`. Simulator `+12` is a packed field. Only its low word is read
by the recovered difficulty consumers, and reflection-backed field identities
prove that this low word is the Star Mode increment/exponent: it is the
exponent of `StarModeHealthMult` and `StarModeDamageMult`, and a third consumer
multiplies it by `StarModeEliteChanceAdd`. It is not the party-size selector.

The constructor's reset to `+12 = 0x00040000` therefore proves only an initial
consumed low word of zero. It does not prove that the complete dword means four
players, difficulty four, or Star Mode four. No post-reset writer to `+12` was
recovered in the build-103 executable. The high word remains unidentified.

`sub_9CE960` and `sub_9CE9B0` are a separate party-size stage. They select
one of two four-element LabsTuning arrays with a one-based cached player count
or a positive debug override. Neither function receives or tests a captain or
elite rank. Captain/elite identity can already affect the authored class and
attribute inputs supplied to their callers, but these two selectors themselves
are only party-count selectors.

## Simulator ownership and lifecycle

The simulator is thread-local client state:

- `sub_9BCBE0` (`0x009BCBE0`) reads the simulator pointer from TLS slot `+8`.
- `sub_9BCC00` (`0x009BCC00`) publishes or clears that TLS pointer.
- `sub_4E8BB0` constructs the simulator with `sub_9C2320` and immediately
  publishes it with `sub_9BCC00(simulator)`.
- `sub_4E5170` tears the surrounding labs/game state down and calls
  `sub_9BCC00(0)` before releasing the remaining state.

The field transitions recovered from all references to these accessors and
the simulator initializer are:

| Transition | Simulator `+8` | Simulator `+12` | Evidence |
| --- | ---: | ---: | --- |
| allocation/setup in `sub_9C21D0` | `0` | `0` | `v4[2] = 0`, `v4[3] = 0` before reset |
| reset/init in `sub_9BEB10` | `4` | `0x00040000` | the only recovered reset call is at the end of `sub_9C21D0` |
| inbound action 48 in `sub_537E00` | fourth decoded dword, through `sub_9BD4F0` | unchanged | handler decodes 16 bytes and passes `v11` |
| inbound action 49 in `sub_537EC0` | sole decoded dword, through `sub_9BD4F0` | unchanged | handler decodes 4 bytes and passes `Dst` |
| TLS unpublish in `sub_4E5170` | object is no longer current | object is no longer current | pointer cleared; this is not a field write |

`sub_9BD4F0` is the only recovered post-reset `+8` setter:

```text
if signed(input) <= 72:
    simulator.+8 = input
else:
    simulator.+8 = 72
```

There is no lower clamp. Zero and negative values are stored unchanged. The
upper fallback is not a guessed limit: `sub_9BD4F0` loads the dword at virtual
address `0x0101D45C`; the executable bytes at file offset `0x00C1BA5C` are
`48 00 00 00`, exactly `72`.

The complete build-103 decompiler reference scan found no post-reset setter for
`+12`. The only writes tied to this simulator object are the setup zero and
`sub_9BEB10`'s `0x00040000`. Consequently the ordinary lifecycle visible in
this executable consumes Star Mode increment zero unless some unrecovered
external/raw-memory restoration path changes the dword. That last exclusion
cannot be proved from a static C listing, so the no-later-writer conclusion is
high rather than absolute confidence.

## DifficultyTuning identity and decoded layout

Names below come from build-103 reflection registration, not from the binary
payload's shape. The registered structure offsets relevant here are:

| Structure offset | Reflected field | Decoded value/layout |
| ---: | --- | --- |
| `+0` / count `+4` | `HealthPercentIncrease` | 72 float32 entries; payload bytes `72..359` |
| `+8` / count `+12` | `DamagePercentIncrease` | 72 float32 entries; payload bytes `360..647` |
| `+56` | `StarModeHealthMult` | float32 `1.2` (`0x3f99999a`) |
| `+60` | `StarModeDamageMult` | float32 `1.05` (`0x3f866666`) |
| `+64` | `StarModeEliteChanceAdd` | float32 `0.05` (`0x3d4ccccd`) |
| `+68` | `StarModeSuggestedLevelAdd` | float32 `3` (`0x40400000`) |

The decoded resource is 2,120 bytes. Its pointer slots are serialized runtime
addresses and are not semantic evidence; its counts and payload values
cross-check the reflected layout. For a compact byte oracle, difficulty rows
1 through 8 are health `[0.60, 0.72, 0.86, 1.03, 1.28, 1.36, 1.44, 1.53]`
and damage `[1.00, 1.05, 1.10, 1.16, 1.39, 1.46, 1.53, 1.61]`; row 72 is
health `851.43` and damage `376.77`.

## `sub_9E2430`: health difficulty and Star Mode stage

`sub_9E2430` reads `difficulty = sub_9BCC50(simulator)` and
`starIncrement = sub_9BCCF0(simulator)`, where `sub_9BCCF0` returns
`simulator.+12 & 0xffff` or zero for a null simulator. Its exact operation is:

```text
if HealthPercentIncrease.count == 0:
    return 1.0

row = unsigned_min(HealthPercentIncrease.count - 1, difficulty - 1)
return HealthPercentIncrease[row]
       * pow(StarModeHealthMult, uint32(starIncrement))
```

There is no lower clamp and the comparison is unsigned. With the build-103
count of 72, difficulty `1..72` selects rows `0..71`; difficulty `0` makes
`difficulty - 1 == UINT_MAX` and therefore selects row 71. Negative stored
difficulties likewise do not receive a signed lower-bound repair and select
according to their unsigned `difficulty - 1`. The null simulator default from
`sub_9BCC50` is difficulty `1`, while `sub_9BCCF0` defaults to zero.

In `sub_9E2D20`, this operation is present only for attribute index `4`, the
health path. The order is:

```text
base = sub_9E2520(attributeData, 4)
classScale = 1.0
if attributeData.ownerClassResource != null:
    classScale = float32(ownerClassResource + 100)

partyHealth = base * (1 + sub_9CE960() * classScale)
difficultyHealth = partyHealth * sub_9E2430()
withFlat = difficultyHealth + attributeData.flat[4]
result = sub_9BFCE0(DifficultyTuning, ownerClass, 0, withFlat)
```

The `+100` class field identity is supported by the NonPlayerClass reflection
name `playerCountHealthScale`. The function defaults it to `1.0` if there is
no class resource. `sub_9BFCE0` is a later level/zone normalization path; it is
not selected by `sub_9CE960`, and this call passes zero for its third argument.

Build-103 reflection initialization at `0x00F5688A..0x00F568D5` strengthens
that identity: the field is registered as a `float`, has property hash
`0x811C9DC5`, runtime offset `0x64`, and default `1.0`. `NonPlayerClass`
registration at `0x00F56BF0` gives the complete object size as `0x7c` with 19
reflected fields.

This runtime layout must not be confused with type `0x474940A5`
`ClassAttributes`, currently projected by `content.db.non_player_class`.
The player-count operand belongs to the separate type-`0xD117AFCA`
`NonPlayerClass` record. Type `0x30728CE7` is `Phase`. The decoded
same-instance `NonPlayerClass` records for the four implemented 1-1 families
all contain the literal float `1.0` at dense prefix offset `+0x64`:

| Instance | Noun | Decoded size |
| ---: | --- | ---: |
| `0x8F291AF3` | `ZelemBasicRanged.Noun` | `462` |
| `0xA1FDCCD8` | `ZelemSpecialHaster.Noun` | `439` |
| `0x1DDB0187` | `NomadSnipe.Noun` | `512` |
| `0xB366BC74` | `NomadWithDrone.Noun` | `466` |

The compiled records retain the reflected default, although the dense output
cannot prove whether it was explicitly authored or inherited. A typed
`0xD117AFCA` decoder can read the direct scalar prefix; its pointer-backed
fields remain opaque until the package relocation/fixup scheme is recovered.
The focused evidence is consolidated in `notes/campaign/non-player-class.md`.

No captain/elite discriminator occurs in this sequence. Such rank differences
may already be present in `base`, `ownerClassResource`, its `+100` scale, or
the later normalization inputs. The recovered code does not justify a new
captain/elite multiplier after the party and difficulty terms.

## `sub_9E5A20`: non-player damage/healing difficulty stage

`sub_9E5A20(source)` returns `1.0` immediately for a source whose byte `+16`
is not `0xff`; the scaling branch is only for a null source or a source marked
non-player by that byte. Its exact scaling branch is:

```text
if DamagePercentIncrease.count == 0 or difficulty == 0:
    return 1.0

row = unsigned_min(DamagePercentIncrease.count - 1, difficulty - 1)
return DamagePercentIncrease[row]
       * pow(StarModeDamageMult, uint32(starIncrement))
```

This has the same upper-only unsigned row selection as the health helper, but
unlike `sub_9E2430` it explicitly treats difficulty zero as unscaled. A null
simulator supplies difficulty one and Star Mode increment zero.

`sub_9E5D10` is a healing application path. When its `a12` scaling flag is
set, it applies source modifier `sub_9E53D0`, target/source modifier
`sub_9E5480`, and then, only for a null/non-player source:

```text
heal *= sub_9CE9B0()
heal *= sub_9E5A20(source)
```

If `a13` is clear it subsequently performs the critical roll and applies the
critical multiplier on success. Recipient healing reduction and hit-point
application follow. Again, no captain/elite selector is passed to
`sub_9CE9B0`; rank-specific inputs can be embodied in the source's authored
attributes and the preceding modifier calls, but the selected table is party
count only.

## Party-count selectors

`sub_9BE3D0(1)` calls `sub_9BE350(currentSimulator, 1)`. With argument one,
`sub_9BE350` returns cached simulator dword index `8370` (`+33480`); a null
simulator defaults to one. With argument zero it instead enumerates the object
set and excludes objects whose `+12` flags contain `0x40`. The two tuning
selectors deliberately request the cached form.

Both selectors first try global `dword_14CAFF0`, loaded by `sub_9CF190` from
LabsTuning integer property hash `1169398165` with default zero. A positive,
in-range override wins; otherwise they try the cached player count. There is
no clamp:

```text
count = override if 0 < override <= arrayLength else cachedPlayerCount
if not (0 < count <= arrayLength):
    return selector-specific default
return array[count - 1]
```

`sub_9CF190` loads the first array from property hash `-370436399`
(`0xe9eb96d1`) and the second from `-585902736` (`0xdd13d570`). No recovered
reflection/string evidence assigns safe authored names to those hashes, so this
note identifies them by hash and use instead of naming them from their bytes.
An exhaustive string audit strengthens that boundary: hashing all `5,067`
distinct identifier-shaped quoted strings retained in canonical
`Game.c` with the game's case-folded FNV-1 routine produces no match for
`0xE9EB96D1`, `0xDD13D570`, or the override hash `0x45B39995`
(`1169398165`). The unrelated retained editor/property string
`ExpectedPlayerCount` hashes to `0x8377510F`, so it is not an alias for the
runtime override. The authored names are therefore stripped from the allowed
client evidence; only a different build, symbols, or external server content
can name them exactly.
The raw LabsTuning resource independently cross-checks their exact payloads:

| Selector | Property entry/payload | Runtime array | Invalid-count default | Caller use |
| --- | --- | --- | ---: | --- |
| `sub_9CE960` | hash at `0x0f58`, values at `0x0f68` | `[0.0, 1.0, 2.0, 3.0]` | `0.0` | additive count beyond the first player: `1 + selected * classScale` |
| `sub_9CE9B0` | hash at `0x0ec0`, values at `0x0ed0` | `[1.0, 1.4, 1.8, 2.2]` | `1.0` | direct non-player heal/damage-family scalar |

Thus valid cached counts 1, 2, 3, and 4 choose array indices 0, 1, 2, and 3.
Counts above four do not saturate at four; they use the neutral selector
default. The same is true of zero and negative counts. The override also falls
back to the cached count when it is invalid rather than forcing the default.

## Separate elite-chance use of the low word

`sub_9FE7D0` is the third recovered caller of `sub_9BCCF0`. It uses
`24 * difficulty` to address two float fields at row offsets `+8` and `+20`
in another tuning array, adds
`StarModeEliteChanceAdd * float32(starIncrement)` to each, and clamps each
result to at most `1.0`:

```text
chanceA = min(1, row[difficulty].floatAt8
                 + StarModeEliteChanceAdd * starIncrement)
chanceB = min(1, row[difficulty].floatAt20
                 + StarModeEliteChanceAdd * starIncrement)
```

This consumer does not subtract one from difficulty and performs no visible
difficulty bounds check before the `24 * difficulty` address calculation. The
reflected additive field establishes the Star Mode/elite connection, but the
two destination probabilities are not named by the recovered evidence here;
calling one "captain" and the other "elite" from their positions alone would
violate the evidence boundary.

## Runtime authority classification

| Stage | Proven owner in this evidence | Authority conclusion |
| --- | --- | --- |
| simulator `+8` selection | client TLS simulator, populated by inbound handlers 48/49 | remotely supplied client state; the sender chooses the presented difficulty, but this does not recover the retail server's validation policy |
| simulator `+12` low word | client TLS simulator | client-consumed Star Mode increment; no setter or retail-server transport field was recovered |
| DifficultyTuning row/exponent arithmetic | build-103 client simulation code and client content resource | exact client presentation/prediction semantics, not by itself authoritative server policy |
| LabsTuning party count arrays | build-103 client simulation code, cached local simulator count, optional client tuning override | exact client presentation/prediction semantics, not proof of server party-count policy |
| captain/elite effects | authored class/attribute inputs outside the two selectors; separate Star Mode elite-chance consumer | no standalone captain/elite multiplier or server policy is proved by `sub_9CE960`/`sub_9CE9B0` |

For a server implementation, health, damage, healing, party size, Star Mode,
and elite selection are gameplay-authoritative mutations and must be owned by
the server. Reproducing these client formulas may be a compatibility decision,
but it should not be described as recovered retail server authority without a
server trace or protocol evidence that fixes the corresponding inputs and
validation rules.

## Confidence and open evidence

| Finding | Confidence | Basis / remaining gap |
| --- | --- | --- |
| `+8` is the one-based difficulty selector and upper fallback is exactly 72 | high | direct getter, setter, inbound callers, executable constant, and 72-entry arrays |
| `+12 & 0xffff` is the Star Mode increment/exponent | high | two reflected `StarMode*Mult` exponent uses plus reflected `StarModeEliteChanceAdd` linear use |
| reset leaves the consumed exponent at zero | exact | `0x00040000 & 0xffff == 0` |
| high word of `+12` means four of some named quantity | unsupported | it is not read by any recovered accessor/consumer; no name is assigned |
| no normal post-reset `+12` setter exists | high | exhaustive canonical decompiler references found only setup/reset writes; raw or externally restored memory remains a static-analysis caveat |
| the two LabsTuning arrays are selected strictly by player count | high | exact selector operands, cached-count implementation, four-entry payloads, and callers |
| either LabsTuning hash has a specific authored property name | unavailable in build 103 | neither reflection nor the exhaustive 5,067-identifier executable-string hash audit contains `0xE9EB96D1`, `0xDD13D570`, or override `0x45B39995`; byte shape is deliberately insufficient |
| `playerCountHealthScale` is runtime offset `0x64` with default `1.0` | exact | reflected field registration and `sub_9E2D20` consumer agree |
| the four implemented 1-1 enemy families use the reflected default | high | none of their decoded `0x30728CE7` records contains a literal float `1.0`; exact sparse/default application still needs a typed decoder |
| `sub_9CE960` or `sub_9CE9B0` selects captain/elite rank | disproved for these functions | neither has a rank operand or branch; each uses only override/count and array length |
| these formulas are retail server-authoritative policy | unresolved | all recovered execution and resources are client-side; inbound difficulty proves replication, not server validation semantics |
