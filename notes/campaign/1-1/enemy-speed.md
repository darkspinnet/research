# 1-1 enemy effective movement speed (build 103)

## Result

For a freshly initialized, unmodified campaign object, the build-103 effective
combat movement speeds are:

| Director role | Noun | `CombatSpeed` (12) | `NonCombatSpeed` (11) | `MovementSpeedBuff` (48) | Effective combat | Effective noncombat |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| captain | `NomadWithDrone.Noun` | 6.5 | 5.0 | 0.0 | **6.5** | 5.0 |
| captain | `NomadSnipe.Noun` | 7.0 | 5.5 | 0.0 | **7.0** | 5.5 |
| minion | `ZelemBasicRanged.Noun` | 5.0 | 3.5 | 0.0 | **5.0** | 3.5 |
| captain | `ZelemSpecialHaster.Noun` | 5.5 | 4.0 | 0.0 | **5.5** | 4.0 |

The units are world units per second: the native movement update consumes a
frame delta converted from milliseconds to seconds, and the locomotion speed
operand is the value above. The effective-speed formula recovered from the
client is:

```text
effective = selectedBaseSpeed * (1 + MovementSpeedBuff)
selectedBaseSpeed = CombatSpeed when in combat, otherwise NonCombatSpeed
```

All four packaged `MovementSpeedBuff` defaults are zero, so each fresh effective
speed equals its selected noun-class base. The result is **exact/high confidence
for fresh objects without a subsequently applied attribute modifier**. It is
not a claim that `ZelemSpecialHaster` remains at 5.5 while a haste ability is
active, nor that a separately affixed creature cannot change selector 48.

No fixed retail-server simulation or replication interval was recovered. The
retail client advances locomotion with the current variable frame delta and
issues a change callback in the same mover update when position or orientation
changes. An incomplete community server uses a 50 ms object-simulation pass and
flushes dirty locomotion during that pass; this is useful comparative evidence,
not build-103 retail-server authority.

## Evidence labels

- **Exact native**: instruction/decompiler evidence from the build-103 client.
- **Exact content**: decoded values from the build-103 packaged class records or
  authoritative runtime content database.
- **Reference implementation**: behavior in the incomplete community
  reimplementation. It is not evidence that the retail server used the same
  timing or packet policy.
- **Inference**: the narrow conclusion compelled by the cited exact evidence,
  with its scope stated explicitly.

## Provenance

- Client version: `bin/game/GameBin/version_bin.txt` reports
  `5.3.0.103`.
- Executable: `bin/game/GameBin/Game.exe`, SHA-256
  `3c7248a4c6614450290a66c22ab7fcda86052b29a7b555e834d7c23f9b73ee5b`.
- Canonical decompiler output: `bin/game/GameBin/Game.c`, SHA-256
  `1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb`.
- Runtime content database:
  `bin/darkspinner/darkspin/cache/content.db`, SHA-256
  `13fd5043b5aa5fffa703b6b3457568e3ecffcb0349720c2343ea23d3c471bb04`.
- Packaged class source: `bin/darkspinner/Data/AssetData_Binary.package`,
  SHA-256
  `faf7b72a27b7f6b6f2c497c9aa5379bd2f20b1e8fe7b59b0d3798a2232774f2b`.
- Focused decoded records used below are retained under
  `bin/game/logs/1-1-enemy-speed`.

Native addresses below are image virtual addresses in the canonical build-103
decompiler output. Source line numbers are the current line numbers in that
file.

## Selector identities and effective formula

The packaged `GlobalDefinitions` enumeration identifies
`NonCombatSpeed = 11`, `CombatSpeed = 12`, and `MovementSpeedBuff = 48`.
The authoritative runtime row is `lua_chunk` ID `659`, source
`lua/0x2E64AA9E.lua`; its decoded payload SHA-256 is
`25927418c849f13c1f45592387f0d9d0fd1e8b67b6537daf0ac8b57d663d57b3`.
The selector recovery is also recorded in `notes/abilities/lightspeed-gaps.md`.

`sub_9E49D0` (`0x009E49D0`, `Game.c:1399788-1399803`) is the native
effective-speed getter:

1. It returns zero when the object or its attribute component is absent.
2. When the combat-state byte is set, or simulator state is `3`, it reads
   selector 12; otherwise it reads selector 11
   (`Game.c:1399797-1399801`).
3. It reads selector 48 and returns
   `(selector48 + 1.0) * selectedBase` (`Game.c:1399802-1399803`).

This proves both selection and combination order. In particular, selector 48
is a fractional multiplier: `0.25` would mean 125% of base, not an additive
0.25 world units per second.

## Noun-class defaults

The four speed operands are 32-bit little-endian floats in decoded resource
type `0x474940A5`, group `0`, decoded size 88 bytes. Native
`sub_9E2180` (`0x009E2180`, `Game.c:1397495-1397559`) independently maps
the selectors to the same decoded float slots:

- selector 11 reads float slot 10, byte offset `0x28`
  (`Game.c:1397539-1397541`);
- selector 12 reads float slot 9, byte offset `0x24`
  (`Game.c:1397542-1397544`);
- selector 48 reads float slot 12, byte offset `0x30`
  (`Game.c:1397548-1397550`).

The exact content identities and decoded record hashes are:

| Noun/class instance | Decimal instance | `content_source_resource` ID | Package ordinal | Decoded SHA-256 | `0x24` | `0x28` | `0x30` |
| --- | ---: | ---: | ---: | --- | ---: | ---: | ---: |
| `NomadWithDrone` (`0xB366BC74`) | 3009854580 | 1474 | 1473 | `1f9450bd0536408f146739794a3edd30baad4af925c827a90ae9fa5ce708d6af` | 6.5 | 5.0 | 0.0 |
| `NomadSnipe` (`0x1DDB0187`) | 500892039 | 792 | 791 | `8abe5d0becfc50e569583140e542f1bed624644332117f393760bdc603eaa905` | 7.0 | 5.5 | 0.0 |
| `ZelemBasicRanged` (`0x8F291AF3`) | 2401835763 | 9352 | 9351 | `029b418d9f0ca9995f43098155d7b89ecd6450efe1ead247f36fdc08faf4280b` | 5.0 | 3.5 | 0.0 |
| `ZelemSpecialHaster` (`0xA1FDCCD8`) | 2717764824 | 6403 | 6402 | `fc12ae13cc2d6cecf87ebec87c8cb5dc91608d44a5b2f094f9fdf321b948fef8` | 5.5 | 4.0 | 0.0 |

These identities were resolved with `darkrun db non_player_class get
instance_id=<decimal>` and `darkrun db content_source_resource get id=<id>`
using `--config bin/darkspinner/darkspin.toml`. The latter rows report type ID
`1195983013` (`0x474940A5`), group zero, decoded size 88, package ordinal, and
the hashes above. The four resources were then decoded with
`darkrun inspect bin/darkspinner/Data/AssetData_Binary.package --ordinal
<ordinal> --output <path>` and their three native-mapped float offsets read
directly. This joins content identity, package bytes, and the native selector
reader without relying on the current server's projected stat model.

Confidence for every raw operand in the table is **exact content/high**. The
agreement with the native slot reader makes the selector interpretation
**exact native/high**.

## Attribute initialization and transforms

The non-player initialization path is:

1. Combatant setup `sub_9D8E60` (`0x009D8E60`) invokes `sub_9E3300` on the
   object's attribute component (`Game.c:1389254-1389285`).
2. `sub_9E3300` (`0x009E3300`) iterates all selectors `0..114`, computes each
   through `sub_9E2E70`, and stores its initialized cache value
   (`Game.c:1398420-1398440`).
3. For an ordinary non-player component without an inherited/template
   component, `sub_9E2E70` dispatches to `sub_9E2D20`
   (`Game.c:1398160-1398189`).
4. `sub_9E2D20` starts with the class default from `sub_9E2520`, adds the
   component's per-selector base delta, and folds its live modifier list through
   `sub_9E2680` (`Game.c:1398113-1398157`).

### Difficulty and captain handling

The only special difficulty/captain-looking scalar branch in this fresh
non-player initializer is guarded by `selector == 4`:

```text
if selector == MaxHealth (4):
    scaled = (difficultyScalar * captainOrClassScalar + 1) * classBase
    scaled = secondScalar * scaled
```

That guard and transform are at `Game.c:1398130-1398145`. Selectors 11,
12, and 48 skip it. The inherited/template multiplier predicate
`sub_9E2120` (`0x009E2120`, `Game.c:1397471-1397491`) includes selector
48 but not 11 or 12. For these four records selector 48 is zero, so even such a
multiplication leaves it zero before any later live modifier.

The campaign director rows provide these exact role-to-base-noun assignments at
difficulty `1-24`:

| `level_director_entry` ID | Configuration | Noun | Min/max difficulty |
| ---: | --- | --- | --- |
| 830 | `minion` | `ZelemBasicRanged.Noun` | 1 / 24 |
| 845 | `captain` | `ZelemSpecialHaster.noun` | 1 / 24 |
| 846 | `captain` | `NomadSnipe.Noun` | 1 / 24 |
| 847 | `captain` | `NomadWithDrone.Noun` | 1 / 24 |

The role label therefore selects these base nouns, not similarly named
`*_Captain.Noun` records. No recovered director row authors a speed modifier,
and the native fresh initializer does not transform speed merely because the
entry is in the captain pool. The conclusion that director difficulty/captain
classification does not change these *fresh* speed operands is **high
confidence**. A later ability, affix, or other runtime attribute modifier can
still change selector 48 through `sub_9E2680`; that dynamic state is outside
the fresh baseline and must be simulated when its separate content contract is
known.

## Tick and locomotion dirty-flush evidence

### Retail build-103 client

`sub_9D2960` (`0x009D2960`, `Game.c:1384042-1384044`) reads the current
simulation-frame duration in milliseconds and multiplies it by `0.001`. It does
not return a fixed step.

The main mover update `sub_9EA1A0` (`0x009EA1A0`) samples that delta at
`Game.c:1404011-1404027`, advances the mover collection, then copies the
resulting position and orientation back to runtime objects. It compares both
against their previous values (`Game.c:1404291-1404343`) and invokes the
virtual callback at vtable offset `+152` only when either changed
(`Game.c:1404342-1404344`). This is **exact native/high** evidence of a
change-driven locomotion notification in the same client update pass. The
callback's eventual network-dirty meaning and the frequency of the server's
corresponding pass are not proved by this client body.

### Community server comparison

The retained reference tree at `bin/game/logs/recap_server-reference` is the
incomplete Resurrection Capsule server, remote
`https://github.com/vitor251093/recap_server.git`, commit
`2851559d098a447ccdabd00353f57bab7fd00997` (2025-10-08). In that tree:

- `game_server/source/game/instance.cpp:443-458` allows the outer server
  poll after 10 ms but runs object simulation only when elapsed time reaches
  50 ms, passing the actual elapsed seconds to `ObjectManager::Update`;
- `game_server/source/game/objectmanager.cpp:289-295` ticks every active
  object and immediately calls `SendObjectUpdate` for a dirty object;
- `game_server/source/game/instance.cpp:919-982` flushes and clears dirty
  flags, including `UpdateLocomotion`, in that call;
- its non-player locomotion branch has unreliable `0x95` sending disabled by
  `#if 0` and instead sends the full reliable locomotion update
  (`instance.cpp:962-977`).

Thus **50 ms (20 Hz)** is exact for this reference implementation's normal
object-simulation/dirty-flush pass, but confidence that retail used it is
**low/unsupported**. The commented packet choice is additional evidence that
this code is reconstruction policy rather than a retained retail-server body.

## Confidence summary and remaining gap

| Finding | Confidence |
| --- | --- |
| Four selector 11/12/48 noun-class defaults | Exact content, high |
| Native selector mapping and `(1 + buff) * base` formula | Exact native, high |
| Fresh effective combat speeds 6.5 / 7.0 / 5.0 / 5.5 | High, scoped to unmodified fresh objects |
| Difficulty/captain initializer does not scale selectors 11/12/48 | Exact native, high |
| Director roles use the four base nouns at difficulty 1-24 | Exact content, high |
| Retail client uses variable delta and change-driven mover callback | Exact native, high |
| Retail server used a 50 ms simulation or dirty-flush cadence | Unresolved; reference-only, low |

The remaining research gap is the original retail server's normal simulation
step, locomotion dirty batching interval, and `0x94`/`0x95` selection policy.
The previously documented 100 ms local fallback must not be reclassified as
recovered evidence from this investigation.
