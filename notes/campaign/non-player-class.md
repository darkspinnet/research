# NonPlayerClass decoding and party-health input (build 103)

## Result

The requested type association is incorrect. In build 103:

- `0x30728CE7` is `Phase`, not `NonPlayerClass`.
- `0xD117AFCA` is `NonPlayerClass`.
- `0x474940A5` is the companion `ClassAttributes` resource.

This is not a cosmetic distinction. The four same-instance `0x30728CE7`
records cited in the task contain Phase/gambit data and no literal
little-endian float `1.0`. The corresponding four `0xD117AFCA` records have a
dense, 0x7c-byte hydrated `NonPlayerClass` prefix and all four contain
`00 00 80 3f` at prefix offset `+0x64`. Therefore a decoder must dispatch
`NonPlayerClass` on `0xD117AFCA` and must reject `0x30728CE7` as the wrong
resource type.

The build-103 health path reads `playerCountHealthScale` from
`NonPlayerClass +0x64` at `0x009E2D92`. It obtains a coefficient selected by
the stable match-membership cardinality at `0x009E2DB1 -> sub_9CE960`, then
computes the multiplier at `0x009E2DB6..0x009E2DC0`:

```text
scaledHealth = baseHealth * (1.0 + partyCoefficient[participantCount - 1] *
                             npc.playerCountHealthScale)
```

Confidence is **high** for the type correction, all scalar offsets/defaults,
the four fixture values, and the participant-count source. Confidence is
**medium** for the semantic names of opaque pointer-backed runtime blocks;
their reflected types are known, but this investigation did not recover the
package relocation/fixup algorithm needed to materialize their children.

## Evidence and resource identity

Canonical inputs:

- `bin/game/GameBin/Game.idb`
- `bin/game/GameBin/Game.c`
- `bin/game/Data/AssetData_Binary.package`
- `bin/game/logs/recap_server-reference/game_server/res/data/assets_catalog.xml`

The asset catalog lists `.NonPlayerClass`, `.ClassAttributes`, `.Phase`,
`.CharacterAnimation`, `.Noun`, and `.AIDefinition` as distinct assets. The
DBPF inventory groups them by the same instance ID. The `0x30728CE7` payloads
contain Phase strings such as combat gambits and ability names. Conversely,
the `0xD117AFCA` payload prefix matches every reflected `NonPlayerClass`
runtime offset, followed by its display name, `.ClassAttributes` reference,
and localized descriptions. This independently identifies the types without
depending on a filename guess.

Localized descriptions are optional in the compiled tail. Package ordinal
234 (`Stalagmite`, instance `0x5715388A`) ends its required display name,
locale reference, and `DEST_prefab_cryos3_stalagmite_large.ClassAttributes`
identity at byte `0xD2`; byte `0xD3` begins non-string serialized state. The
importer therefore decodes the description/reference pair only when a
printable description is present, while keeping every required identity and
present locale reference strict.

Package ordinal 321 (`Soul Crate`, instance `0x6D4D038C`) demonstrates that
both locale references are independently optional: its compiled tail contains
the display text, `DEST_prefab_spectra_object_stealthball.ClassAttributes`,
and the plain description `Holds the mined supernatural engery`, with no
`AssetStrings!` entry for either text. Tail decoding consequently recognizes
locale references by their `AssetStrings!` prefix instead of shifting every
later string when either reference is absent.

Package ordinal 1019 (`Fear Cage`, instance `0xC1A31C3B`) omits the display
string and its locale reference entirely. Its tail starts with
`DEST_prefab_tnx_object_silenceField.ClassAttributes`, followed by the plain
description `Fear cage` and its `AssetStrings!0x761EF791` reference. The
required `.ClassAttributes` identity is therefore the structural anchor;
display text and its locale reference are optional prefixes rather than fixed
positions.

Noun-family identity is not unique across class resources. Ordinal 189
(`DLS-1227 the Impenetrable Defender`, instance `0xD62729BD`) and ordinal
1475 (`Invincitron`, instance `0xB366BC74`) both inherit
`NomadWithDrone.ClassAttributes`, while retaining distinct display identity,
rank, descriptions, and affixes. Storage therefore preserves every distinct
resource instance. A noun-family fallback prefers the canonical row whose
instance equals the case-insensitive FNV-1 hash of the class stem;
`NomadWithDrone` hashes to `0xB366BC74`.

The focused fixtures extracted read-only from the package are:

| Asset/instance | DBPF ordinal and type | Size | SHA-256 |
|---|---:|---:|---|
| `NomadSnipe`, `0x1DDB0187` | 789, `0xD117AFCA` | 512 | `d652b69c65deee2efa5ccf316345fb6f93fd59a3bfa57dd778160f2f61bad87b` |
| `NomadWithDrone`, `0xB366BC74` | 1475, `0xD117AFCA` | 466 | `643b3b6d40fef04094e3afeacebe46d1fa3476d645bc695f0609f93e53aa95c7` |
| `ZelemSpecialHaster`, `0xA1FDCCD8` | 6404, `0xD117AFCA` | 439 | `5fa93423b4edefea579a3f13f621884dc3b3071adfe2ec6f66a7bd0563b721a1` |
| `ZelemBasicRanged`, `0x8F291AF3` | 9353, `0xD117AFCA` | 462 | `c33e81a9ea10cbc72c01ce42035ee4ea81c755f7e428a4ad2217670bcbec542f` |
| `NomadSnipe.Phase`, `0x1DDB0187` | 792, `0x30728CE7` | 546 | `0bac97e7e62537a729a643041e0bd0c33e5dc5cced644099fd0d2477c0b1baa5` |
| `NomadWithDrone.Phase`, `0xB366BC74` | 1472, `0x30728CE7` | 88 | `dee5e3f46153a9ed5dac9dc30bda648e834aa0b73d652638dd8b13543d24a78b` |
| `ZelemSpecialHaster.Phase`, `0xA1FDCCD8` | 6401, `0x30728CE7` | 359 | `034332692f30f2cb31b92d0baa88fd6769a8faf0307ff992e65eac4aaa928069` |
| `ZelemBasicRanged.Phase`, `0x8F291AF3` | 9350, `0x30728CE7` | 160 | `240d7a31dda9c8f789a5010720ef4f133959ffc200bb934292cd31cd78a47235` |

The extracted bytes are under
`bin/game/logs/non-player-party-health/ordinal-*.bin`. A complete scan finds
no `00 00 80 3f` sequence in any of the four Phase fixtures. Direct reads from
each actual `NonPlayerClass` fixture return `0x3F800000` at `+0x64`.

For an additional same-instance check, `ZelemBasicRanged.Noun` is ordinal
9352, type `0x76A8F7D8`; its payload names
`ZelemBasicRanged.NonPlayerClass`,
`ZelemBasicRanged.CharacterAnimation`, and
`ZelemBasicRanged.AIDefinition`. Its companions are ordinal 9349
`0x17BBCE29` (`CharacterAnimation`), 9350 `0x30728CE7` (`Phase`), 9351
`0x474940A5` (`ClassAttributes`), 9353 `0xD117AFCA`
(`NonPlayerClass`), and 9354 `0xEEEB0E31` (`AIDefinition`).

## Reflection and all 19 fields

The static field initialization occupies `0x00F55BC0..0x00F56BC1`.
Registration at `0x00F56BF0..0x00F56C2D` pushes runtime size `0x7c` at
`0x00F56BF5`, field count `0x13` at `0x00F56BF7`, field table
`off_1166280` at `0x00F56BF9`, and the string `NonPlayerClass` at
`0x00F56C05`. The repeated `0x811C9DC5` argument is the reflection name-hash
seed passed to `sub_AD9B60`; it should not be recorded as the computed hash of
`playerCountHealthScale`.

The table below is sorted by runtime offset. “Init” is the address at which
the field name/default/offset sequence is established.

| # | Field | Runtime offset | Reflected kind | Default | Init evidence |
|---:|---|---:|---|---|---|
| 1 | `testingOnly` | `+0x00` | bool | `false` | `0x00F55BE5..0x00F55C28` |
| 2 | `creatureType` | `+0x04` | enum | `0` | name `0x00F55D8D`, offset `0x00F55DB2` |
| 3 | `mpClassEffect` | `+0x08` | asset reference to `ServerEventDef` | null | name `0x00F5649C`, offset `0x00F564C4` |
| 4 | `mpClassAttributes` | `+0x0C` | asset reference to `ClassAttributes` | `Default.ClassAttributes` | name `0x00F563D2`, default `0x00F563F1`, offset `0x00F563FB` |
| 5 | `name` | `+0x10` | `cLocalizedAssetString` | empty | name `0x00F55CBB`, offset `0x00F55CEB` |
| 6 | `challengeValue` | `+0x24` | int32 | `0` | name `0x00F5611B`, offset `0x00F56140` |
| 7 | `dropType` | `+0x28` | array of enum | empty | name `0x00F5663A`, offset `0x00F5665F` |
| 8 | `dropDelay` | `+0x34` | float32 | `0.0` | `fldz` `0x00F56727`, offset `0x00F56735` |
| 9 | `aggroRange` | `+0x38` | float32 | `0.0` | name `0x00F55EB0`, `fldz` `0x00F55ECF`, offset `0x00F55EDC` |
| 10 | `alertRange` | `+0x3C` | float32 | `0.0` | name `0x00F55F7D`, `fldz` `0x00F55F9C`, offset `0x00F55FA9` |
| 11 | `dropAggroRange` | `+0x40` | float32 | `FLT_MAX` (`0x7F7FFFFF`) | name `0x00F5604A`, load from `0x01020920` at `0x00F56069`, offset `0x00F5607A` |
| 12 | `mNPCType` | `+0x44` | enum | `0` | name `0x00F561E4`, offset `0x00F56209` |
| 13 | `npcRank` | `+0x48` | int32 | `1` | name/hash `0x00F562B2..0x00F562CC`, default/offset `0x00F562D4..0x00F562DA` |
| 14 | `targetable` | `+0x4C` | bool | `true` | name `0x00F567D6`, default `0x00F567F5`, offset `0x00F567FB` |
| 15 | `description` | `+0x50` | `cLocalizedAssetString` | empty | name `0x00F56565`, default `0x00F56588`, offset `0x00F56592` |
| 16 | `playerCountHealthScale` | `+0x64` | float32 | `1.0` | type/name/default/offset sequence `0x00F5688A..0x00F568D5`; `fld1` at `0x00F568BE`, offset write at `0x00F568CB` |
| 17 | `longDescription` | `+0x68` | array of `cLongDescription` | empty | name `0x00F5696C`, offset `0x00F56991` |
| 18 | `eliteAffix` | `+0x70` | array of `cEliteAffix` | empty | name `0x00F56A39`, offset `0x00F56A5D` |
| 19 | `playerPet` | `+0x78` | bool | `false` | name `0x00F56B03`, offset `0x00F56B2B` |

The word at `+0x30` is inside the runtime storage occupied by `dropType`
(between that field's `+0x28` start and `dropDelay` at `+0x34`). It is
`0xFFFFFFFF` in all four fixtures but is not a twentieth reflected field.
Do not expose it as an independent property.

### Dense defaults, not a sparse `+0x64` stream offset

`+0x64` is a runtime-structure offset, and in the actual `0xD117AFCA`
records it is also observable in the dense hydrated prefix. This does not
mean every reflected runtime offset is generally a stand-alone wire offset:
the prefix includes pointer-backed localized strings, asset references, and
arrays whose words are serialized runtime/fixup state. Those words cannot
safely be interpreted as offsets into the payload without recovering the
relocation scheme.

There is no evidence in these eight fixtures for a sparse field-tag/override
stream, nor for an inherited-default omission at `+0x64`. Instead:

1. Reflection defines constructor defaults, including `1.0` at
   `0x00F568BE`.
2. Asset compilation/hydration materializes a dense 0x7c-byte
   `NonPlayerClass` prefix.
3. The four actual NPC records retain the default `1.0` literally at
   `+0x64`.
4. The four records that omit it are a different reflected type (`Phase`),
   so their omission says nothing about `NonPlayerClass` default encoding.

Inheritance or authored overrides may have been resolved before the compiled
record was emitted. The dense output alone cannot distinguish “explicitly
authored equal to the default” from “inherited/defaulted.” Consequently the
safe wording is **effective compiled value**, not authored provenance, unless
source assets are recovered.

## Effective non-default values in the four NPC fixtures

All four have `npcRank=1`, `targetable=true`,
`dropAggroRange=FLT_MAX`, and `playerCountHealthScale=1.0`, matching
reflection defaults. All have empty `dropType`, empty `eliteAffix`, null
`mpClassEffect`, and two `longDescription` entries.

| Fixture | Name / class attributes | `creatureType` | `challengeValue` | `aggroRange` | `alertRange` | `mNPCType` |
|---|---|---:|---:|---:|---:|---:|
| ordinal 789 | `Decelerator` / `NomadSnipe.ClassAttributes` | 1 | 27 | 15.0 | 17.0 | 1 |
| ordinal 1475 | `Invincitron` / `NomadWithDrone.ClassAttributes` | 0 (default) | 30 | 15.0 | 17.0 | 1 |
| ordinal 6404 | `Haster` / `ZelemSpecialHaster.ClassAttributes` | 1 | 30 | 17.0 | 16.0 | 1 |
| ordinal 9353 | `Space Barracuda` / `ZelemBasicRanged.ClassAttributes` | 1 | 6 | 22.0 | 16.0 | 0 (default) |

Thus the compiled non-default scalars are every nonzero
`challengeValue`, `aggroRange`, and `alertRange`; `creatureType=1` in
ordinals 789/6404/9353; and `mNPCType=1` in ordinals 789/1475/6404.
The nonempty name, class-attributes reference, description, and
long-description blocks are also effective non-default data.

## Implementation-ready Go decoder contract

Dispatch and bounds:

```go
const (
	phaseTypeID          uint32 = 0x30728CE7
	nonPlayerClassTypeID uint32 = 0xD117AFCA
	nonPlayerClassSize          = 0x7C
)
```

- Accept only `typeID == nonPlayerClassTypeID`.
- Return a typed `wrongType` error for `0x30728CE7`, explicitly identifying
  it as `Phase`.
- Require `len(payload) >= 0x7c`; retain any tail for the later
  relocation/string decoder.
- Read all fixed words little-endian.
- Decode bools as uint32 at `+0x00`, `+0x4c`, and `+0x78`; reject values
  other than 0 or 1.
- Decode enums as uint32 and signed integer properties as int32.
- Decode float32 values from their raw uint32 bits. Permit exact
  `0x7F7FFFFF` for `dropAggroRange`; reject NaN for scalar gameplay fields.
  Domain limits beyond that require separate gameplay evidence.
- Return the pointer-backed spans as opaque data in this first-stage decoder:
  `mpClassEffect[0x08:0x0c]`,
  `mpClassAttributes[0x0c:0x10]`, `name[0x10:0x24]`,
  `dropType[0x28:0x34]`, `description[0x50:0x64]`,
  `longDescription[0x68:0x70]`, and `eliteAffix[0x70:0x78]`.
  Never use their embedded 32-bit words as direct slice offsets or process
  pointers.
- Do not synthesize a missing `+0x64` value for a short record. A short
  `0xD117AFCA` record is malformed for this decoder. Reflection defaults
  belong in a separate constructor/override layer if that format is later
  recovered.

The stage-one result should contain the 19 reflected properties, with the
seven pointer-backed properties represented by typed opaque blocks plus these
direct scalars:

```go
TestingOnly           bool    // +0x00
CreatureType          uint32  // +0x04
ChallengeValue        int32   // +0x24
DropDelay             float32 // +0x34
AggroRange            float32 // +0x38
AlertRange            float32 // +0x3c
DropAggroRange        float32 // +0x40
NPCType               uint32  // +0x44
NPCRank               int32   // +0x48
Targetable            bool    // +0x4c
PlayerCountHealthScale float32 // +0x64
PlayerPet             bool    // +0x78
```

This split is implementation-ready for party-health scaling without
pretending the unresolved reference/fixup encoding is known.

## Stable participant count used by `sub_9CE960`

`sub_9CE960` is `0x009CE960..0x009CE9A8`.

- `0x009CE960` first reads debug/forced selector
  `dword_14CAFF0`.
- With no valid override, `0x009CE983` pushes `1` and
  `0x009CE985` calls `sub_9BE3D0(1)`.
- It bounds-checks the returned one-based participant count against the
  coefficient vector at `off_1164DB4..off_1164DB8` and loads element
  `count-1` at `0x009CE97E`.
- Invalid/empty counts return `0.0` at `0x009CE9A6`.

`sub_9BE3D0` at `0x009BE3D0` is a TLS/current-simulation wrapper around
`sub_9BE350`. Because the selector passes `true`, `sub_9BE350` takes its
fast path at `0x009BE35B..0x009BE36B` and reads the cached dword at
`simulation +0x82c8` at exactly `0x009BE362`. Its false path instead walks
the membership container at `simulation +0x82bc` and excludes objects whose
flags at object `+0x0c` contain `0x40`; that filtered path is not used by
`sub_9CE960`.

The cache is the element count of the membership hash container:

- `sub_9C2340` at `0x009C2340` addresses the container as
  `simulation +0x82bc` at `0x009C234B` and calls lookup-or-insert
  `sub_9C20E0` at `0x009C2351`.
- The absent-key path calls `sub_9C1940` at `0x009C217F`.
  `sub_9C1940` detects an existing one-byte key at
  `0x009C1970..0x009C1978`; only a new key increments container
  `+0x0c` at `0x009C19E2`. Because the container base is
  `simulation +0x82bc`, this is exactly `simulation +0x82c8`.
- `sub_9BF360` at `0x009BF360` addresses the same container at
  `0x009BF377` and calls `sub_9BE970` at `0x009BF3B5`.
  Each matching removal decrements container `+0x0c` at
  `0x009BE9EF`.
- `sub_9BF3E0` clears the container and explicitly writes zero to its
  `+0x0c` count at `0x009BF401`.

The event adapters show what the one-byte key means. “Client Session Player
Connect” `sub_5362D0` calls `sub_9C2340` at `0x0053630C`; “Client Session
Player Disconnect” `sub_536340` calls `sub_9BF360` at `0x0053637A`.
Replay/history processing in `sub_9ECEC0` applies event kind 0 as membership
insertion at `0x009ED014` and kind 1 as removal at `0x009ECFFE`, both keyed
by the stored byte at event `+1`.

Therefore the selector input is the cardinality of distinct persistent
match/session-player membership keys. “Persistent” here means roster state
that is maintained across callbacks and reconstructible from match history;
it is removed by the matching membership-disconnect event. It is not a
socket count, peer arrival ordinal, or current iteration position:

- duplicate insertion of the same stable key does not increment the cache;
- removal is by key and decrements it;
- replayed membership events reconstruct the same cardinality;
- hash-bucket/peer iteration order never enters `sub_9CE960`;
- the connect adapter consumes the stable player key and does not feed a
  transient peer-order number into the counter.

## Focused tests

1. **Type dispatch:** all four `0xD117AFCA` fixtures decode; all four
   `0x30728CE7` fixtures fail with `wrongType: Phase`.
2. **Priority field:** each NPC fixture yields raw bits `0x3F800000` and
   `PlayerCountHealthScale == 1.0` from `+0x64`.
3. **Fixture table:** assert every scalar shown in the effective-value table,
   including `FLT_MAX`, `npcRank=1`, `targetable=true`, and the two enum
   exceptions that remain at default zero.
4. **Bounds:** payloads of 0x7b bytes or less fail; a 0x7c-byte prefix
   succeeds; an appended tail does not change scalar results.
5. **Opaque references:** mutate embedded pointer/fixup words while keeping
   scalar offsets fixed and confirm stage-one scalar decoding does not
   dereference or index through them.
6. **Validation:** invalid bool words and NaN gameplay floats fail;
   `dropAggroRange` raw `0x7F7FFFFF` succeeds.
7. **Reflection-default fixture:** construct a 0x7c-byte prefix with
   `dropAggroRange=FLT_MAX`, `npcRank=1`, `targetable=1`,
   `playerCountHealthScale=1.0`, and other scalar defaults; assert the exact
   decoded defaults.
8. **Membership cardinality:** insert stable keys A/B, repeat A, remove B,
   and assert cached counts 1/2/2/1. Clear must produce zero.
9. **Order independence:** replay A-then-B and B-then-A and assert the same
   count and coefficient index.
10. **Health formula:** with count `n`, coefficient vector entry `n-1`,
    NPC scale `s`, and base health `h`, assert
    `h * (1 + coefficient[n-1] * s)`; verify the forced-selector override is
    a separate test path.

## Confidence and remaining boundary

- **High:** `0xD117AFCA`/`0x30728CE7` identities; 19-field count and 0x7c
  runtime size; every scalar offset/default; literal `1.0` in all four NPC
  fixtures; health formula addresses; cached count offset and
  insertion/removal behavior.
- **Medium-high:** semantic interpretation as stable match membership. It is
  supported by connect/disconnect event names, duplicate-key behavior, and
  replay reconstruction.
- **Medium:** opaque block layout beyond its reflected type and byte span.
  A second-stage decoder should not be implemented until the package
  relocation/fixup mechanism is independently recovered.
