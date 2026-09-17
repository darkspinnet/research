# Hero basic decoder gaps

## Scope and result

This report captured the original `32/100` baseline and its 68 explicit
failures. Subsequent evidence-backed aliases, decoder shapes, Wraith operand,
Sprout runtime, ordinary Lua 5.1 `math.rad`, instruction-checked modifier
aliases, specialized event contracts, nested melee hit schedules, and the
scoped Lightspeed compatibility link now compile `100/100`.
Shadow and Time Ravager preserve their authored multi-hit rows and execute
their repeated campaign damage commits transactionally.
The recipe-40 end-to-end build proved Fire Ravager and Lightning Tempest
complete, while Missile Tempest and Time Ravager exposed secondary boundaries.
The original 68 failures below still
serve as the dependency inventory and historical reproduction baseline.
The dependency-identity pass is complete: every packaged dependency named in
this report now has an exact chunk identity. The one absent generic chain
module remains an external package boundary for Citadel, while Orion uses the
documented scoped compatibility link. The table below remains a historical
inventory and distinguishes recovered identities from the implementation
boundaries that followed them.

The inventory is from the runtime
`bin/darkspinner/darkspin/cache/content.db`. Package presence was checked with
`darkrun inspect bin/game/Data/ServerData.package`; the package has 1,101
resources. Authored noun names were decoded from the 100 player noun records
in `AssetData_Binary.package`, not guessed from the display names or hashes.

## Definition inventory

The noun lists are ordered Alpha, Beta, Gamma, Delta. Hex values are the noun
IDs used as the keys of `playerBasicUnsupported`.

| Basic definition; root | Affected hero nouns | Primary classification | Current failure and next proven boundary |
|---|---|---|---|
| `BeastSmash`; chunk `143`, `Abilities/0x12139728.lua` | Savage: `PC_LF_Tank2.Noun` (`DA338C06`), `PC_LF_Tank2_v1.Noun` (`E06819AE`), `PC_LF_Tank2_v2.Noun` (`4B61F3A7`), `PC_LF_Tank2_v3.Noun` (`E236BF9C`) | resolved linker alias | `Modifiers!modifier_energysentinel_healdebuff.lua` resolves to chunk `942`, `Modifiers/0x1E05F108.lua`; supplying that exact alias compiles the definition without another error. |
| `BinarySentinelBasic`; chunk `582`, `Abilities/0x953C39B8.lua` | Magnos: `PC_SP_Tank.Noun` (`B01A9CDF`), `PC_SP_Tank_v1.Noun` (`D8E75D15`), `PC_SP_Tank_v2.Noun` (`F7CDA488`), `PC_SP_Tank_v3.Noun` (`5804D543`) | resolved linker alias | `0x3681d755!GlobalDefinitions.lua` resolves to chunk `659`, `Lua/0x2E64AA9E.lua`; that exact alias compiles the definition. |
| `EnergySentinelBasic`; chunk `796`, `Abilities/0xC6CF21CF.lua` | Goliath: `PC_TE_Tank2.Noun` (`C940B9DF`), `PC_TE_Tank2_v1.Noun` (`EB58DC15`), `PC_TE_Tank2_v2.Noun` (`0A3F2388`), `PC_TE_Tank2_v3.Noun` (`6A765443`) | resolved linker alias | Same exact chunk-`942` alias and successful compile probe as `BeastSmash`. |
| `FireRavagerBasic`; chunk `88`, `Abilities/0x4921A499.lua` | Krel: `PC_EL_fireRavager.Noun` (`52FE4B89`), `PC_EL_fireRavager_v1.Noun` (`5082FF33`), `PC_EL_fireRavager_v2.Noun` (`B9431A7A`), `PC_EL_fireRavager_v3.Noun` (`226E4D85`) | resolved linker aliases | `Modifiers!modifier_fireravagerburn.lua` resolves to chunk `126`; its DOT template resolves to chunk `149` and the global bootstrap to chunk `659`. With those identities supplied and `Lua!Global.lua` treated as the already-seeded bootstrap, the root compiles. Chunk `555` is only a false-positive string-constant candidate and need not compile. |
| `FireTempestBasic`; chunk `459`, `Abilities/0xD882C2C9.lua` | Char: `PC_EL_Mage.Noun` (`A894C6CF`), `PC_EL_Mage_v1.Noun` (`FC38F905`), `PC_EL_Mage_v2.Noun` (`C9BC4F38`), `PC_EL_Mage_v3.Noun` (`2A4DAAB3`) | constrained-decoder limitation | Execution succeeds, but `abilityDefinitionFromLua` sees no `hitEffect`, assumes the projectile schema, and requires missing `distance`. The authored table instead has `alwaysUseCursorPos`, `radius`, `range`, `hitEvent`, `timetohit`, and `timetorelease`: a cursor/area basic shape. |
| `ForcePulse`; chunk `528`, `Abilities/0x77B70E70.lua` | Andromeda: `PC_SP_gravityTempest.Noun` (`E7227797`), `PC_SP_gravityTempest_v1.Noun` (`7007354D`), `PC_SP_gravityTempest_v2.Noun` (`FB4FD7C0`), `PC_SP_gravityTempest_v3.Noun` (`1E3F4C3B`) | resolved linker alias | `Modifiers!modifier_knockbackwithimmunity.lua` resolves semantically to packaged chunk `878`, `Modifiers/0x9B80AB0C.lua`. Supplying that exact alias compiles the definition. |
| `LightningTempest_Basic`; chunk `894`, `Abilities/0x519FCB9B.lua` | Lumin: `PC_EL_lightningTempest.Noun` (`397288E5`), `PC_EL_lightningTempest_v1.Noun` (`57EFED77`), `PC_EL_lightningTempest_v2.Noun` (`29BBE97E`), `PC_EL_lightningTempest_v3.Noun` (`A06CCD79`) | implemented event/global surface | The modifier body is exact chunk `897`, `Modifiers/0x768636B6.lua`. Instruction-level reinspection proves the earlier diagnostic misattributed its failed lookup: the required member is `nAbilityEventFlags.StackModifier = 32`, while the modifier separately uses `nModifierPriorities.Vulnerable = 600`. Both values are defined by GlobalDefinitions chunk `659`; no `nModifierPriorities.StackModifier` member exists or should be invented. |
| `LightspeedTempestBasic`; chunk `938`, `Abilities/0x55608FEA.lua` | Orion: `PC_SP_lightSpeedTempest.Noun` (`49F75386`), `PC_SP_lightSpeedTempest_v1.Noun` (`C0420C2E`), `PC_SP_lightSpeedTempest_v2.Noun` (`2B3BE627`), `PC_SP_lightSpeedTempest_v3.Noun` (`C210B21C`) | implemented scoped compatibility link | `Modifiers!modifier_chain_ability_counter.lua` remains absent, but only Orion substitutes exact counter chunk `339` plus template chunk `169`; haste chunk `201` and AttributeUtils chunk `87` link exactly. The unrelated Citadel consumer receives no alias. |
| `MissileTempestBasic`; chunk `441`, `Abilities/0x07DBB360.lua` | SRS-42: `PC_TE_Mage2.Noun` (`D2136801`), `PC_TE_Mage2_v1.Noun` (`91C9630B`), `PC_TE_Mage2_v2.Noun` (`C6BF5AD2`), `PC_TE_Mage2_v3.Noun` (`7981091D`) | implemented | Ordinary `math.rad` preserves the authored 45-degree lock angle. The projectile template's numeric `impactEvent = 0` is an absent-asset sentinel; this ability publishes its real `cyber_missile_homing_hit.ServerEventDef` through `explodeEvent`, which the typed decoder now retains. Recipe-40 verification compiles all four variants. |
| `PlasmaSentinelBasic`; chunk `409`, `Abilities/0x5CF2E729.lua` | Zrin: `PC_EL_Tank.Noun` (`2257E3E9`), `PC_EL_Tank_v1.Noun` (`96D97C53`), `PC_EL_Tank_v2.Noun` (`E64CE19A`), `PC_EL_Tank_v3.Noun` (`39588D25`) | implemented paired-hit shape | Recipe 41 links exact burn chunk `306`. The typed definition retains both `hitEffects`, `hitModifiers`, and ranked chances 10/100; runtime selects the zipped pair by animation index, emits its effect, rolls 1..100, and requests Shock or Burn on success. |
| `Pummel`; chunk `309`, `Abilities/0x2633EE6F.lua` | Wraith: `PC_SN_Tank.Noun` (`D6189D41`), `PC_SN_Tank_v1.Noun` (`FA748ACB`), `PC_SN_Tank_v2.Noun` (`2CEE8592`), `PC_SN_Tank_v3.Noun` (`F8FE7CDD`) | implemented global operand | Root instruction `268` evaluates `nBit.Or(nDebuffDescriptors.IsSilence)`. Packaged GlobalDefinitions chunk `659` proves `IsSilence = 256`; the constrained runtime now seeds that exact operand and all four Wraith variants compile. |
| `ShadowRavagerBasic`; chunk `674`, `Abilities/0x5D05F491.lua` | Skar: `PC_SN_Rogue.Noun` (`5DDD8EE5`), `PC_SN_Rogue_v1.Noun` (`3C7CAF77`), `PC_SN_Rogue_v2.Noun` (`0E48AB7E`), `PC_SN_Rogue_v3.Noun` (`84F98F79`) | compiled; runtime blocked | The melee template proves each ordered `hit` table entry is separately waited and runs the complete damage path. The compiler preserves all five two-hit rows, including row one at 0.26/0.31 seconds; campaign admission rejects the shape until repeated commits are supported. |
| `Sprout`; chunk `982`, `Abilities/0x009B7C57.lua` | Tork: `PC_LF_shroomTempest.Noun` (`74B4AB9A`), `PC_LF_shroomTempest_v1.Noun` (`2690DBAA`), `PC_LF_shroomTempest_v2.Noun` (`1260DF63`), `PC_LF_shroomTempest_v3.Noun` (`98DCF9A8`) | identities resolved; custom runtime shape | The root's exact dependencies are toss template chunk `630`, DOT template chunk `149`, and global bootstrap chunk `659`. With those identities supplied, the generic decoder reaches `distance: fieldKind: nil`; Sprout is a custom lob/projectile/cloud definition. Chunk `647` is a modifier false-positive candidate and its `PreloadAnimation` error is irrelevant to the matching root. |
| `TCShieldedSentinelBasic`; chunk `612`, `Abilities/0xB28755E4.lua` | Titan: `PC_TE_Tank.Noun` (`B6ECE7BF`), `PC_TE_Tank_v1.Noun` (`4732CE35`), `PC_TE_Tank_v2.Noun` (`4FFB2028`), `PC_TE_Tank_v3.Noun` (`C97F05E3`) | implemented | The proven point-blank template alias and typed decoder preserve caster-centered radius, one-third bonus damage, and the authored `impactEvent` (`ctd_minn_tc_4_bullet_hit.ServerEventDef`). Recipe-40 verification compiles all four variants. |
| `TimeRavagerBasic`; chunk `385`, `Abilities/0x5FB76978.lua` | Vex: `PC_SP_Rogue.Noun` (`D7AB4DD3`), `PC_SP_Rogue_v1.Noun` (`E98F78C1`), `PC_SP_Rogue_v2.Noun` (`F0058AD4`), `PC_SP_Rogue_v3.Noun` (`403CA1BF`) | implemented campaign multi-hit execution | Exact modifier chunk `893`, StackModifier flag, priorities, AttributeUtils bootstrap, and selectors execute. The catalog restores the omitted inherited melee discriminator, and the fifth animation preserves both authored damage commits. |
| `Trapper_Grenade`; chunk `909`, `Abilities/0x141BA061.lua` | Seraph-XS: `PC_TE_Rogue.Noun` (`30E47FB3`), `PC_TE_Rogue_v1.Noun` (`A2867361`), `PC_TE_Rogue_v2.Noun` (`92DE8FF4`), `PC_TE_Rogue_v3.Noun` (`FC625BDF`) | identity resolved; toss decoder shape | Toss template chunk `630`, `Abilities/0xCDD518BA.lua`, is exact. After aliasing it, the decoder rejects missing `animationstate`; the authored toss shape uses its own warmup/near/far animation fields. |
| `VoodooTempestBasic`; chunk `635`, `Abilities/0x493651EB.lua` | Jinx: `PC_SN_Mage2.Noun` (`E94FDC47`), `PC_SN_Mage2_v1.Noun` (`560124FD`), `PC_SN_Mage2_v2.Noun` (`E1048EF0`), `PC_SN_Mage2_v3.Noun` (`6E497EEB`) | identities resolved; toss decoder shape | Its exact dependencies are toss template chunk `630`, weaken modifier chunk `644` (`Modifiers/0x71564167.lua`), and global bootstrap chunk `659`. With those bodies supplied, the decoder reaches the same missing generic `animationstate` boundary as Trapper because Voodoo uses `nearAnimation`/`farAnimation`. |

## Packaged identity evidence

The following bodies are present in `ServerData.package`; therefore their
failures are identity gaps, not missing packaged Lua. Stable package ordinals
and decoded sizes independently match the `content.db` chunks.

| Authored dependency | Exact packaged target | Package evidence |
|---|---|---|
| `Modifiers!modifier_energysentinel_healdebuff.lua` | chunk `942`, SHA-256 `a9648e174cff6a5324ab4e469ca8b77b3adce0e04f7e6bd93e80e21a80877cb1` | ordinal `1005`, `001005_3681d755_fc0ff8f5_000000001e05f108.bin`, 1,440 decoded bytes |
| `0x3681d755!GlobalDefinitions.lua` / `Lua!GlobalDefinitions.lua` | chunk `659`, SHA-256 `25927418c849f13c1f45592387f0d9d0fd1e8b67b6537daf0ac8b57d663d57b3` | ordinal `711`, `000711_3681d755_3681d755_000000002e64aa9e.bin`, 14,305 decoded bytes |
| `Modifiers!modifier_fireravagerburn.lua` | chunk `126`, SHA-256 `b3d74b2e64fb7527dfff1387ee17e737135debc980fa001cc82309c263b4d92f` | ordinal `135`, `000135_3681d755_fc0ff8f5_0000000071216062.bin`, 1,977 bytes |
| `Modifiers!template_modifier_dot.lua` | chunk `149`, SHA-256 `01b9ed201b2c5bd0e72820d42f76532a1a229ea1ed4a3a134933fe058a84dcaf` | ordinal `163`, `000163_3681d755_fc0ff8f5_00000000d6c886b1.bin`, 4,026 bytes |
| `Modifiers!modifier_knockbackwithimmunity.lua` | chunk `878`, SHA-256 `2b701ae49a3dd53855cf7b2168ebb3c628e7dec2d68b833a9e757dbb6e9ba1cc` | ordinal `940`, `000940_3681d755_fc0ff8f5_000000009b80ab0c.bin`, 2,135 bytes |
| `Modifiers!modifier_LightningTempest_Basic.lua` | chunk `897`, SHA-256 `3f43665f94d756759e07efdc1017585f386d2556d440cde1b6a2029c0af971b4` | ordinal `959`, `000959_3681d755_fc0ff8f5_00000000768636b6.bin`, 2,402 bytes |
| `Modifiers!modifier_lightspeed_haste.lua` | chunk `201`, SHA-256 `878850cc5072c0cd2d3551763a07fc009afdeafad7a6bcc31de502eda158f9c6` | ordinal `223`, `000223_3681d755_fc0ff8f5_00000000cfdb9908.bin`, 3,231 bytes |
| `Modifiers!modifier_plasmasentinel_burn.lua` | chunk `306`, SHA-256 `938687b85638a3a6f7e875eb1aa00236f2708a459827c0f284278cea49057b0b` | ordinal `335`, `000335_3681d755_fc0ff8f5_00000000ecbd9819.bin`, 1,991 bytes |
| `Abilities!template_ability_pointblankaoe.lua` | chunk `324`, SHA-256 `554be7a0fa694b0abe3d96ff8258af147e40e85a843d76c7bcb4b776064152f0` | ordinal `355`, `000355_3681d755_7153bbb1_000000008c550d00.bin`, 3,337 bytes |
| `Modifiers!modifier_timeravager_basic.lua` | chunk `893`, SHA-256 `1e31483a329c374b339d96b8b3dc2b7432ba7813e6466a370b6e72b35bc2d62d` | ordinal `955`, `000955_3681d755_fc0ff8f5_00000000fbfdcb83.bin`, 2,997 bytes |
| `Abilities!template_ability_toss.lua` | chunk `630`, SHA-256 `c75b68fab1bd56c399894277954544a697d308b121d551b2e31ee5871b86259f` | ordinal `678`, `000678_3681d755_7153bbb1_00000000cdd518ba.bin`, 6,803 bytes |
| `Modifiers!modifier_voodootempest_weaken.lua` | chunk `644`, SHA-256 `b0dd46a7208c3b651b111cf1c853d7219a444b591937dcde469ed7a4454d7316` | ordinal `693`, `000693_3681d755_fc0ff8f5_0000000071564167.bin`, 1,995 bytes |

The semantic targets are not name-only guesses. For example, chunk `942`
defines and registers `nModifier_EnergySentinel_HealDebuff` /
`EnergySentinelHealDebuff`; chunk `878` defines
`nModifier_Knockback_With_Immunity` / `KnockbackWithImmunityModifier`; chunk
`324` defines `nAbility_PointBlankAoE_Template`; chunk `630` defines
`nAbility_Toss_Template`; and chunk `644` defines and registers
`nModifier_VoodooTempest_Weaken` / `VoodooTempestWeaken`.

## Minimal fixes ranked by completed hero unlocks

An unlock is counted only when every currently proven blocker in the row is
addressed. Merely exposing the next error earns no unlock.

| Rank | Heroes unlocked | Minimal evidence-backed fix |
|---:|---:|---|
| 1 | 8 | Add one instruction-checked module alias from `Modifiers!modifier_energysentinel_healdebuff.lua` to chunk `942`. This completes both `BeastSmash` and `EnergySentinelBasic` in the diagnostic compiler. |
| 2 | 4 | Alias `0x3681d755!GlobalDefinitions.lua` to chunk `659` for `BinarySentinelBasic`. |
| 3 | 4 | Alias `Modifiers!modifier_knockbackwithimmunity.lua` to chunk `878` for `ForcePulse`. |
| 4 | 4 | Implemented: ordinary `math.rad(number)->number` and the template-zero-to-authored-`explodeEvent` contract compile all four `MissileTempestBasic` variants. |
| 5 | 4 | Implemented: seed numeric `nDebuffDescriptors.IsSilence = 256`, recovered exactly from chunk `659`, so `Pummel` evaluates its authored bit mask. |
| 6 | 4 | Add a cursor/area-basic decode branch for `FireTempestBasic`: consume its authored `range`, `radius`, `hitEvent`, `timetohit`, and `timetorelease` without requiring projectile `distance`, `speed`, `trail`, or noun fields. |
| 7 | 4 | Extend animation-sequence timing to retain the two-number `hit` intervals authored by `ShadowRavagerBasic`; do not collapse the interval to an arbitrary endpoint. |
| 8 | 4 | For `FireRavagerBasic`, alias modifier chunk `126`, DOT-template chunk `149`, and GlobalDefinitions chunk `659`, and treat `Lua!Global.lua` as the existing typed bootstrap rather than a fabricated payload. This exact probe compiles. |
| 9 | 4 | Implemented: `LightningTempest_Basic` aliases modifier chunk `897` and seeds the instruction-proven `nAbilityEventFlags.StackModifier = 32` event bit plus its distinct `nModifierPriorities.Vulnerable = 600` priority. The formerly requested `nModifierPriorities.StackModifier` was a diagnostic namespace error and does not exist in shipped content. |
| 10 | 4 | For `PlasmaSentinelBasic`, alias chunks `306`, `149`, and `659` plus the typed `Lua!Global.lua` bootstrap, then recognize its descriptor-driven melee/hit-modifier shape instead of demanding projectile distance. |
| 11 | 4 | Implemented: `TCShieldedSentinelBasic` links point-blank-AoE template chunk `324`, uses the typed point-blank branch, and reads the authored `impactEvent`; no projectile distance is required. |
| 12 | 4 | Implemented: `TimeRavagerBasic` aliases modifier chunk `893` and uses the same event bit `nAbilityEventFlags.StackModifier = 32` with its separately authored `nModifierPriorities.Slowed = 450`; there is no unresolved priority value. |
| 13 | 4 | For `Trapper_Grenade`, alias toss template chunk `630` and decode its toss warmup/near/far animation fields instead of requiring generic `animationstate`. |
| 14 | 4 | For `VoodooTempestBasic`, apply the toss fix, alias weaken chunk `644` and GlobalDefinitions chunk `659`, use the typed global bootstrap, and decode `nearAnimation`/`farAnimation`. |
| 15 | 4 | For `Sprout`, alias toss chunk `630`, DOT-template chunk `149`, and GlobalDefinitions chunk `659`, use the typed global bootstrap, and add a custom lob/cloud projection that does not invent generic projectile `distance`. |
| 16 | 4 | Implemented: scope the missing aggregator compatibility link to `LightspeedTempestBasic`, loading exact Orion counter chunk `339`, generic template chunk `169`, haste chunk `201`, and AttributeUtils chunk `87`; do not apply it to Citadel. |

Ranks 2-16 all unlock four heroes and are ordered from smallest verified change
to changes with additional evidence dependencies, not by a fabricated
sub-ranking of equal unlock counts.

## Native and decoder evidence

- Build-103 `Game.c` registers `nAbility.RegisterAbility`,
  `PreloadAsset`, `PreloadModifier`, and `PreloadAnimation` in the binding table
  around source lines `1512132`-`1512207`. Native `sub_A680C0` at
  `0xA680C0` accepts a numeric/integer/string first animation identity and a
  second string namespace, hashes strings, and returns the normalized first
  identity. This is why decoder-side animation shape support must not assume
  every authored timing field is a scalar string/number pair.
- Bootstrap `sub_A11990` around lines `1435968`-`1435976` augments the Lua
  `math` namespace with the game RNG and continues into the ordinary runtime
  initialization. The constrained compiler instead constructs a fresh `math`
  table containing only `floor`; chunk `441`'s ordinary Lua 5.1 `math.rad`
  call therefore fails before registration.
- The executable contains the native registration/reflection boundary but no
  authored strings for the missing `modifier_chain_ability_counter.lua` body,
  `StackModifier` policy, or module-path-to-hashed-resource aliases. Those
  identities must come from packaged bytecode/instruction evidence, not from
  the native binder.
- `abilityDefinitionFromLua` currently selects melee only when `hitEffect` is
  a string and otherwise falls through to a generic projectile schema. That
  explains the post-alias `distance: fieldKind: nil` failures for the authored
  cursor-area, point-blank, hit-modifier melee, and custom lob families.

## Reproduction artifacts

Generated diagnostics are kept under `bin/game/logs` as required by the
workspace conventions:

- `ability-basic-gaps-test.log`: the exact 68 noun-keyed loader errors.
- `ability-basic-gaps-serverdata-inspect.log`: the complete 1,101-resource
  `ServerData.package` inventory.
- `ability-basic-gaps-disassembly.log`: disassembly used for the `math.rad`,
  `IsSilence`, interval-animation, and false-positive-candidate findings.
- `ability-basic-gaps-alias-probe.log`: read-only compile probes using exact
  packaged chunk identities.
- `ability-basic-gaps-nouns.log`: the 100 decoded authored player noun names
  and hashes used for the 68-noun mapping.

The temporary diagnostic tests used to print these artifacts were removed.
No server, decoder, importer, package, database, or staging change was made.
