# LightspeedTempestBasic Lua dependency gaps

## Result

The two reported dependencies have different outcomes:

- `Lua!AttributeUtils.lua` is **present under its exact authored FNV-1
  identity**. Case-insensitive FNV-1 of `AttributeUtils` is `0x4C983F6A`.
  `ServerData.package` contains that resource at ordinal `95` as
  `3681D755/3681D755/4C983F6A`, and the authoritative `content.db` contains
  it as `server_data` resource `13611` and `lua_chunk.id=87`. Its bytecode
  disassembles successfully and defines the expected attribute helper globals.
  The importer now carries the instruction-checked alias and links the
  dependency to chunk `87`; this is no longer a content or linker gap.
- `Modifiers!modifier_chain_ability_counter.lua` is **absent as an exact
  packaged identity**. FNV-1 of `modifier_chain_ability_counter` is
  `0xA53AC836`; no resource with that instance exists in any local package or
  in `content_source_resource`. The exact dependency string occurs only in
  ability chunks `4` and `938`. Related generic and Orion-specific counter
  bodies do exist under other, internally consistent hashed identities, but
  neither is instruction-proven to be the single target of the missing
  `require`. The requested module may have been a legacy alias or an omitted
  aggregator. Its exact body is truly absent from this build's package set.
  Orion now uses a deliberately scoped compatibility link to its exact
  chunk-`339` counter plus generic chunk-`169` template. That closes the four
  Lightspeed Tempest basics without assigning the same guess to the unrelated
  Citadel consumer.

This corrects the narrower conclusion in `notes/abilities/basic-gaps.md` that
`AttributeUtils.lua` was unresolved without a packaged body. The body was
already in that report's package inventory; the case-sensitive source lookup
missed `lua/0x4C983F6A.lua`.

## Corpus and method

The canonical game package tree contains 41 packages under `bin/game/Data`.
Every package index was inspected with `darkrun inspect`, matching both Lua
resource type `0x3681D755` and the expected instances `0xA53AC836` and
`0x4C983F6A`. The duplicate Darkspinner tree also contains 41 packages; a
relative-path SHA-256 comparison found `mismatch_count=0`, so it is a
byte-identical mirror rather than a second content corpus.

The inspected game packages were the 26 root packages
`Arenas_RDX9`, `Arenas_Textures`, `AssetData_Binary`, `Audio`, `AudioProps`,
`BMDL_environments`, `CompiledAnimData`, `CompiledMaterials`, `Config_Pack`,
`Config_Ship`, `Creatures`, `Editors`, `Effects`, `FlashUI`, `Images_32bit`,
`Levels`, `Movies`, `Other`, `Pollination`, `PreBaked`, `RenderingConfig`,
`RenderingConfig_Lights`, `RenderingConfig_Lua`, `ServerData`, `UI`, and
`Web`, plus `Audio`, `Movies`, and `Text` for each of `de-de`, `en-us`,
`fr-fr`, `pl-pl`, and `ru-ru`.

Only `RenderingConfig_Lua.package` and `ServerData.package` contain resources
of Lua type `0x3681D755`. The former has five resources, none with either
candidate instance. `ServerData.package` has 1,101 resources and contains the
exact AttributeUtils instance but not the chain-counter instance. DBPF indexes
do not retain authored paths, so identities below combine FNV-1, package
type/group/instance, bytecode constants, and disassembly rather than treating
a synthetic filename as an authored filename.

Database queries used the authoritative
`bin/darkspinner/darkspin/cache/content.db` through `darkrun db --config
bin/darkspinner/darkspin.toml`. The complete 65,758-row
`lua_string_constant` table was searched in bounded ID windows for chain,
counter, Lightspeed, and stack-count terms. Extracted bytecode was
disassembled with `darkrun lua`. `bin/game/GameBin/Game.c` was
searched for the authored names and inspected at the native Lua registration
tables.

## Exact dependency graph

`LightspeedTempestBasic` is `lua_chunk.id=938`, source
`Abilities/0x55608FEA.lua`, server-data resource `14517`, package ordinal
`1001`, SHA-256
`82083fe2d37f13ca2a8035564c3e561fbe01dafed18847648dd284a000a54309`,
and 3,610 decoded bytes. Its top-level instructions require modules in this
order:

| PC | Dependency | Database target |
|---:|---|---|
| `0`-`2` | `Abilities!template_ability_projectile.lua` | chunk `38`, resolved |
| `3`-`5` | `Modifiers!modifier_chain_ability_counter.lua` | `NULL` |
| `6`-`8` | `Modifiers!modifier_lightspeed_haste.lua` | chunk `201`, resolved |

The haste body is present as chunk `201`,
`Modifiers/0xCFDB9908.lua`, server-data resource `13739`, package ordinal
`223`, SHA-256
`878850cc5072c0cd2d3551763a07fc009afdeafad7a6bcc31de502eda158f9c6`,
and 3,231 decoded bytes. `0xCFDB9908` is the exact FNV-1 of
`modifier_lightspeed_haste`. Its sole dependency is
`Lua!AttributeUtils.lua`, now linked to exact chunk `87`.

The root bytecode uses `nModifier_LightspeedHaste`, `LightSpeedHaste`, and the
haste table's `attackSpeedScale` to derive its haste token. It does **not**
contain `LightspeedBasicStackCounter` or
`nModifier_Lightspeed_Basic_Counter`. That absence is important when judging
the chain-counter candidates.

## AttributeUtils: exact hashed body

| Evidence | Exact result |
|---|---|
| Authored stem | `AttributeUtils` |
| Case-insensitive FNV-1 | `0x4C983F6A` |
| Package identity | ordinal `95`, `3681D755/3681D755/4C983F6A` |
| Package sizes | 1,186 stored, 2,784 decoded bytes |
| `content.db` identity | `server_data`/resource `13611`; `lua_chunk.id=87`; source `lua/0x4C983F6A.lua` |
| Decoded SHA-256 | `861450587408da51e6d0a937d7fbe32d7b3a17dffff04f5bfbfded987f228820` |
| Direct dependency | `Lua!GlobalDefinitions.lua`; exact body chunk `659`, supplied by the typed bootstrap |

The chunk has an empty embedded source name, so the importer correctly falls
back to the package group and synthetic resource name. A lookup using
uppercase `Lua/` returns no row because `source_name` comparison is
case-sensitive; the stored source is lowercase `lua/0x4C983F6A.lua`.

Top-level disassembly requires GlobalDefinitions and installs four globals by
side effect; it does not return a module table:

- `CreateStackingAttributeFunctions(attributeType, rankedProperty)` returns
  initialization and update closures. Initialization creates/uses the current
  ability's private table and stores an `nAttribute.AddAttributeModifier`
  handle under `modifier_..attributeType`. Update removes the previous handle,
  multiplies `nModifier.GetMyStackCount()` by the ranked value, and installs a
  replacement handle.
- `CreateStackingSpeedFunctions(rankedProperty)` does the same for both
  `nAttributeType.CombatSpeed` and `NonCombatSpeed`, using private keys
  `modifier_combatspeed` and `modifier_noncombatspeed`.
- `AutoDestructSpeedModifier(rankedProperty)` adds a ranked
  `nAttributeType.MovementSpeedBuff` modifier to the ability agent.
- `StandardBossSetup(agent)` adds the authored boss immunities and requests
  `BossStealthDetect` through `nModifier.RequestModifier`.

The semantic match is exact: chunk `201` calls
`CreateStackingAttributeFunctions` with
`nAttributeType.AttackSpeedScale`, one of the globals defined only by chunk
`87` in the packaged corpus. Across all constants, seven consumer chunks in
addition to chunk `87` mention that helper, while `CreateStackingSpeedFunctions`
occurs only in chunk `87`. This is not a name-only guess.

The database now records the instruction-checked `lua_module_alias` row:

`Lua!AttributeUtils.lua` -> chunk `87` / `lua/0x4C983F6A.lua`.

Its own `Lua!GlobalDefinitions.lua` dependency retains the authored-name versus
synthetic-identity mismatch in `lua_dependency`, but execution is not blocked.
The packaged semantic body is chunk `659`, `lua/0x2E64AA9E.lua`, SHA-256
`25927418c849f13c1f45592387f0d9d0fd1e8b67b6537daf0ac8b57d663d57b3`.
The constrained runtime preloads the exact GlobalDefinitions bootstrap before
executing AttributeUtils, so chunk `87` reaches its helper registrations.

## Chain counter: absent exact module and packaged candidates

Case-insensitive FNV-1 gives these independently checkable identities:

| Authored stem | FNV-1 | Package result |
|---|---:|---|
| `modifier_chain_ability_counter` | `0xA53AC836` | absent from all 41 packages and `content_source_resource` |
| `template_chain_counter` | `0x184258A8` | chunk `169`, present |
| `modifier_lightspeedtempest_basic_counter` | `0x82C8A3E8` | chunk `339`, present |
| `modifier_lightspeedtempest_support_counter` | `0x47E583B7` | chunk `504`, present |
| `modifier_lightningtempest_basic_counter` | `0x55A2D8D3` | chunk `166`, present |
| `modifier_beast_smash_counter` | `0x9CB7B2D6` | chunk `1023`, present |

The exact missing string occurs only twice in all 65,758 constants:

- chunk `4`, `Abilities/0x90C547DA.lua`, which registers
  `CitadelSpecificFour_Bolt`;
- chunk `938`, the Lightspeed root.

No chunk registers that module name, and no `lua_module_alias` row names it.
Its use by an unrelated Citadel ability is evidence against simply mapping
the generic-looking missing name to Orion's specific counter.

### Generic template candidate

Chunk `169`, `Modifiers/0x184258A8.lua`, server-data resource `13702`, package
ordinal `186`, SHA-256
`e42a37bad3d4216978945320a8223244ccd60c213e0ef5f371ff2f5b409ec8d5`,
and 948 decoded bytes is the exact body of `template_chain_counter.lua`.
It defines `nModifier_Chain_Counter_Template` with:

- `requiresAgent=false`;
- `handledEvents=nAbilityEventFlags.StackModifier`;
- `activationType=nActivationType.Stacks`;
- `deactivationType=nDeactivationType.OnAgentDeath`;
- ranked `maxStackCount=3` and `duration=0.6000000238418579`;
- a forever-waiting tick and a handler that increments the modifier stack on a
  `StackModifier` ability event.

This is the strongest generic semantic candidate, but its exact authored hash
and dependency name are `template_chain_counter`, not
`modifier_chain_ability_counter`. Treating the latter as an alias would be a
policy decision, not recovered identity evidence.

### Orion-specific candidate

Chunk `339`, `Modifiers/0x82C8A3E8.lua`, server-data resource `13886`, package
ordinal `370`, SHA-256
`cdf28a8bdbc32813189acb780079c5aa2704dbc207742b85225b11188f68511d`,
and 734 decoded bytes is exactly consistent with the authored stem
`modifier_lightspeedtempest_basic_counter`.

It requires `Modifiers!template_chain_counter.lua`, derives
`nModifier_Lightspeed_Basic_Counter` from
`nModifier_Chain_Counter_Template`, sets ranked `maxStackCount=2` and
`duration=1`, registers `LightspeedBasicStackCounter`, and forwards its tick
and event handler to the template. Parallel chunks `166`, `504`, and `1023`
use the same inheritance shape for Lightning Tempest basic, Lightspeed
support, and Beast Smash, respectively.

Chunk `339` is the clear packaged Orion basic-counter body, but chunk `938`
does not name its registration or table and asks for a differently named
module. A missing aggregator could have loaded the template and/or one of
these concrete counters; the corpus does not prove which side effects the
missing require was intended to provide. Consequently:

- chunk `169` is a **candidate alias** for generic chain behavior;
- chunk `339` is a **candidate companion module** for Orion's concrete basic
  counter;
- neither is a safe, instruction-proven direct alias for
  `Modifiers!modifier_chain_ability_counter.lua`.

## Haste body and runtime globals

Chunk `201` defines and registers `nModifier_LightspeedHaste` /
`LightSpeedHaste`. Static and instruction evidence gives:

| Field or behavior | Authored value |
|---|---:|
| `activationType` | `nActivationType.Stacks` = `4` |
| `modifierPriority` | `nModifierPriorities.Haste` = `350` |
| `handledEvents` | `nAbilityEventFlags.StackModifier` = `32` |
| `maxStackCount` | ranked constant `5` |
| `duration` | `20` seconds |
| `descriptors` | `nDescriptors.IsHaste | IsBuff` = `512 | 16` = `528` |
| `attackSpeedScale` | ranked constant `0.07999999821186066` |
| `speedAdjustment` | ranked constant `0.07999999821186066` |

Activation initializes the attack-speed and movement-speed attribute handles.
The forever-waiting tick preserves the modifier. A `StackModifier` event calls
`nModifier.IncrementStackCount()` and updates both handles. The movement buff
also reads `nClient.GetStackCount(nUtil.SPID("LightSpeedHaste"))` for the local
player before applying its authored update path.

The numeric enum values above are not guesses. GlobalDefinitions chunk `659`
constructs the tables in its top-level instructions: `StackModifier=32`,
`Stacks=4`, `Haste=350`, `IsBuff=16`, `IsHaste=512`,
`AttackSpeedScale=23`, `NonCombatSpeed=11`, `CombatSpeed=12`, and
`MovementSpeedBuff=48`. The last four also agree with their ordered constants
in the `nAttributeType` table.

The complete runtime surface crossed by the recovered modules is:

- Lua/bootstrap globals: `require`, `Class`, `nBit.Or`, the global
  `GetRankedValueHelper`, `nAbility.GetRankedValue`, `TranslateToken`, the enum
  tables above, and the `nAbilityFns` callback table. The disassembly proves
  these call sites, but `Game.c` does not expose either ranked-value
  helper in the native registration tables inspected below;
- ability/modifier natives: `nAbility.GetAgentID`, `GetAbilityEventType`,
  `PreloadAsset`, and `RegisterAbility`; `nModifier.GetMyAgentID`,
  `GetMyStackCount`, `IncrementStackCount`, `ResetDuration`,
  `RegisterModifier`, and `RequestModifier`;
- state/attribute natives: `nThreadData.CreatePrivateTable` and
  `GetPrivateTable`; `nAttribute.AddAttributeModifier` and
  `RemoveAttributeModifier`; `nGameObject.AddEffect`; `nThread.WaitForever`;
- client/utility natives: `nClient.GetMyPlayerObjectId` and `GetStackCount`,
  plus `nUtil.SPID`.

`Game.c` confirms the native boundary but contains none of the authored
dependency names, helper names, or concrete Lightspeed registration strings:

- lines `325161`-`325177` register `GetMyPlayerObjectId` and `GetStackCount`
  under `nClient`;
- lines `1422822`-`1422838` register private-table operations under
  `nThreadData`;
- lines `1426221`-`1426239` register add/remove operations under `nAttribute`;
- lines `1428260`-`1428274` register `SPID` under `nUtil`;
- lines `1434758`-`1434804` register `WaitForever` under `nThread`;
- lines `1512120`-`1512212` register the `nAbility` operations, including
  `GetAgentID`, `RegisterAbility`, `GetAbilityEventType`, and `PreloadAsset`;
- lines `1512213`-`1512249` register `nModifier` operations, including
  `GetMyAgentID`, `RegisterModifier`, `IncrementStackCount`, and both stack
  getters.

Therefore `Game.c` can validate callable globals and their native
ownership, but it cannot supply the missing module-to-resource mapping or the
omitted aggregator body.

## Implemented boundary

1. `Lua!AttributeUtils.lua` maps only to chunk `87`, guarded by its exact
   identity, and executes after the chunk-`659` GlobalDefinitions bootstrap.
2. Exact authored family names map `template_chain_counter` to chunk `169`,
   `modifier_lightspeedtempest_basic_counter` to chunk `339`, and
   `modifier_lightspeed_haste` to chunk `201`.
3. Only `LightspeedTempestBasic` substitutes the shipped Orion counter for the
   absent aggregator require. This is an explicit compatibility fallback, not
   a claim that chunk `339` has the missing aggregator's identity.
4. `CitadelSpecificFour_Bolt` receives no such substitution. Its use of the
   absent generic module remains an external package/source boundary.
5. The four Orion basics now compile with their authored projectile, counter,
   stacking-haste, timing, and attribute definitions. Further work is live
   stack/haste parity verification, not dependency reverse engineering.

The captured 0.6.10 regression also proved that the projectile template's
callbacks are behavior, not decorative metadata. `GetProjectilesToLaunch`
returns one for animation sequence indexes zero and one, and two for index two.
`GetShotOffset` selects `(0.65, 0, 0.4)` for animation A,
`(-0.65, 0, 0.4)` for animation B, and both offsets for animation C. The
server now retains that per-animation launch shape and gives each projectile
its own object identity, trajectory, collision, and damage resolution.

The same capture showed `LightspeedTempestActive` was rejected before cast as
an invalid cursor-area definition because its required
`spacetime_AOE_slow_effect.ServerEventDef` impact event was absent. That event
is now present in the recovered Temporal Siphon definition, allowing the
existing radius-seven damage and five-second `LightSpeedSlow` path to run.
The declared eight-second cooldown and 18 base power with coefficient 0.09
remain nonpaying because the custom Lua never invokes the payment operation.

## Reproduction artifacts

Read-only extracted bytecode used for this investigation is under
`bin/game/logs`:

- `ability-lightspeed-root.luac` (chunk `938`);
- `ability-lightspeed-haste.luac` (chunk `201`);
- `ability-lightspeed-attribute-utils.luac` (chunk `87`);
- `ability-lightspeed-global-definitions.luac` (chunk `659`);
- `ability-lightspeed-candidate-{166,169,339,504,806,1023}.luac`;
- `ability-lightspeed-other-root.luac` (chunk `4`).

These are generated diagnostics in the workspace-designated log directory.
The research extraction modified no shipped package or game asset; subsequent
server-side aliases and compiler/runtime support are documented above.
