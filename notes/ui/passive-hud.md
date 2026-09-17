# Deployed passive HUD contract (build 103)

## Result

Build 103 does not have a passive-specific HUD packet. The right-side passive
icon is an ordinary icon-bearing modifier in the deployed agent's replicated
modifier collection. Logical application messages `36`, `37`, and `38` (wire
`0xa2`, `0xa3`, and `0xa4`) create, update, and erase that collection entry.
`MaxisHUDBuffs` then polls the currently controlled agent, resolves the entry's
modifier GUID to the locally registered modifier definition, and copies that
definition's `icon` token into `HUD_Modifiers.swf`.

The tutorial is the sole activation exception. Its initial dungeon baseline
withholds Blitz's `0xa2` entry and masks his critical rating, AutoCrit, and
critical-damage increase. Crossing the authored
`vo_ship_tut_abilities_passive` box publishes the retained passive instance and
enables the complete critical profile together. Ordinary campaign Blitz
sessions publish the passive and use criticals from entry as before.

For Blitz the three relevant identities are:

| Role | Exact identity | Meaning |
|---|---:|---|
| profile passive ability | `4022963036` / `0xEFC98B5C` | Value stored in every Blitz variant's `ability_passive` slot. |
| modifier registration | `LightningRoguePassive`, GUID `0xEFC98B5C` | The modifier definition registered by packaged Lua and placed in `0xa2.ModifierGUID`. It is numerically equal to the profile ID because both are the case-insensitive FNV-1 hash of the same asset name; it is a different semantic field, not a second use of the profile record. |
| HUD-only icon token | `forkedLightning`, SPID `0x190B0DC0` / `420154816` | The Lua definition's `icon`. It is resolved from client content and is never present in `0xa2`, `0xa3`, or `0xa4`. |

The synthetic packaged source name `Modifiers/0xDDD108BF.lua` is also not any
of those identities. It is only the indexed source/resource identity for the
bytecode container.

## Authoritative content evidence

The runtime content database is
`bin/darkspinner/darkspin/cache/content.db`, read through
`bin/darkspinner/darkspin.toml`.

`creature_template get ability_passive=4022963036` returns all four Blitz
variants (Beta `454054015`, Alpha `1667741389`, Gamma `1839725446`, and Delta
`2911848321`). `creature_template_ability get
asset_name=LightningRoguePassive` independently returns the same four records
with `slot="passive"`.

`lua_string_constant get string_constant=LightningRoguePassive` resolves Lua
chunk `845`. Its authoritative row and extracted payload are:

| Item | Exact value |
|---|---|
| source | `Modifiers/0xDDD108BF.lua` |
| `server_data_resource_id` | `14422` |
| decoded size | `1937` bytes |
| SHA-256 | `d797193ce6fdb9c7833ed3356913fe30e3c3ba1f871d3e81fbc8003e82c276d0` |
| extracted evidence | `bin/game/logs/passive-hud-lightning-14422.luac` |

The decoded Lua 5.1 chunk constructs
`nModifier_LightningRogue_Passive` with these relevant fields:

```text
activationType   = nActivationType.Unique
icon             = nUtil.SPID("forkedLightning")
localizedGroup   = nUtil.SPID("Blitz")
localizedName    = nUtil.ToGUID("0x09506544")
localizedDescription = nUtil.ToGUID("0x09506545")
descriptors      = 0
deactivationType = nModifierDeactivationType.OnAgentDeath
primaryStat      = nModifierPrimaryStat.Percent
criticalDamageIncrease = { 0.5, 0.5, 0.5, 0.5 }
criticalPercentIncrease = { 0, 0, 20, 20 }
nModifier.RegisterModifier("LightningRoguePassive", definition)
```

`Activate` gets the agent and adds the ranked
`CriticalDamageIncrease`. In overdrive it also converts the ranked critical
percentage through `nTuning.CriticalRatingPerCritPercent` and adds
`CriticalRating`. It then calls `nThread.WaitForever`. `Deactivate` is an empty
return. Therefore the definition has no finite authored duration and does not
delete itself. `descriptors=0` is also direct proof that a status descriptor is
not required for the HUD icon.

The repository's recovered identifier function (`server/utils/hash.go`) is
case-insensitive 32-bit FNV-1. Applying it gives:

```text
LightningRoguePassive = 0xEFC98B5C = 4022963036
forkedLightning        = 0x190B0DC0 = 420154816
SoulLink               = 0x16C5C6B0 = 382060208
```

Thus the equality between Blitz's profile ability ID and modifier GUID is
explained, while the icon remains a separate UI token.

## Passive slot and squad cross-check

`Game.c` function `sub_9C25C0` (`0x009C25C0`, near source line
`1370540`) maps ability index `1` to definition dword `37`; indices `0`, `2`,
`3`, and `4` use the basic and active fields instead. Lua binding
`sub_A05710` (`0x00A05710`, near line `1426993`) implements
`nPlayer.GetSquadPassiveGUIDs`: it loops deck indices `0..2`, calls
`sub_9C25C0(player, 1, deckIndex)`, and returns the three results at Lua table
indices `1..3`. This establishes that `4022963036` originates as Blitz's
passive-slot asset identity.

Chunk `417`, source `Modifiers/0x1481CD77.lua`, resource `13968`, decoded size
`2818`, SHA-256
`7bfd28c75c9c3f0ab18851021e900fa0c5fe9e063d609a1d3a199034abb4a5ab`,
registers `SoulLink`. Its extracted payload is
`bin/game/logs/passive-hud-squad-root-13968.luac`.

SoulLink is a useful type-boundary cross-check, not Blitz's HUD icon token. Its
activation helper obtains `GetSquadPassiveGUIDs(playerID)` and
`GetCurrentDeckIndex(playerID)`, excludes `currentDeckIndex + 1`, and calls
`nModifier.RequestModifier(agent, agent, guid)` for the other two entries.
Its deletion helper calls `nModifier.MarkForDelete(agent, instance)` for every
stored request. This proves that the numeric passive-slot identities are
accepted as modifier GUIDs by the modifier API. It does not make `SoulLink`
(`0x16C5C6B0`) the GUID or icon for `LightningRoguePassive`.

## Exact modifier wire contract

The application dispatcher in `Game.c` calls:

```text
logical 36 -> sub_539FE0 -> sub_537780   // wire 0xa2, create
logical 37 -> sub_53A090 -> sub_5379E0   // wire 0xa3, update
logical 38 -> sub_53A140 -> sub_537510   // wire 0xa4, delete
```

The three parsers demand exactly `37`, `21`, and `8` payload bytes. All
multibyte values are little-endian.

### `0xa2` ModifierCreated

| Offset | Width | Field | HUD significance |
|---:|---:|---|---|
| `0x00` | 4 | target object ID | Selects the agent modifier collection. The target must already resolve. |
| `0x04` | 4 | modifier GUID | For Blitz, `0xEFC98B5C`. Used to resolve the local definition and its icon. |
| `0x08` | 4 | runtime instance ID | The HUD's stable identity for this displayed entry. |
| `0x0c` | 4 | duration milliseconds | Used with start time for cooldown presentation; Blitz authors no finite duration. |
| `0x10` | 4 | overdrive/rank state | Stored on the modifier entry; not used to choose the icon. |
| `0x14` | 4 | stack count | Displayed as `stackCount`. |
| `0x18` | 8 | start milliseconds | Used with duration for cooldown percent/remaining. |
| `0x20` | 4 | source object ID | Resolved into the entry; not used to choose the icon. |
| `0x24` | 1 | bound flag | Stored on the entry; not used to choose the icon. |

`sub_539FE0` first resolves the target from the first dword and calls
`sub_537780` only if it exists. There is no deferred retry. `sub_537780`
inserts or replaces the entry in the target's modifier component at agent
offset `+692`, keyed by instance ID.

Native locally requested modifiers use the same contract. In `sub_9E0FB0`,
the instance is filled and `sub_A1FB20` publishes logical message `36` before
the Lua thread is created and before `Activate` is invoked. Consequently an
authoritative Blitz request publishes `0xa2` before its attribute additions
and before its `WaitForever` begins.

### `0xa3` ModifierUpdated

| Offset | Width | Field | Behavior |
|---:|---:|---|---|
| `0x00` | 4 | target object ID | Selects the collection. |
| `0x04` | 4 | runtime instance ID | Finds or creates the collection entry. |
| `0x08` | 8 | new start milliseconds | Updated unless both dwords are `0xffffffff`. |
| `0x10` | 4 | stack count | Replaces the stored count. |
| `0x14` | 1 | bound flag | Replaces the stored flag. |

`sub_5379E0` sets modifier-component dirty byte `+1156`. It does not carry a
GUID, duration, source, overdrive field, or icon and cannot independently
identify Blitz's passive. It refreshes timing/stack presentation for an
existing identity.

### `0xa4` ModifierDeleted

| Offset | Width | Field |
|---:|---:|---|
| `0x00` | 4 | target object ID |
| `0x04` | 4 | runtime instance ID |

`sub_537510` erases that instance from the target's modifier map. The next HUD
scan removes its cached entry and rebuilds the ActionScript buff array. No GUID
or icon is required on deletion.

## Exact HUD consumption

`sub_421EE0` (`0x00421EE0`, near line `167983`) names the controller
`MaxisHUDBuffs`. `sub_421EF0` loads `HUD_Modifiers.swf` and registers
`Buffs.GetModifierTooltipInfo`.

The controller update `sub_4247C0` (`0x004247C0`, near line `170055`) does the
following:

1. Resolve the current/deployed agent. In the local-player branch it reads the
   controlled-object handle at player offset `+4664` / `+0x1238`.
2. Read that agent's modifier component at `+692` and iterate its modifier map.
3. Resolve each entry's modifier GUID (`i[3]`) with `sub_9DAC00`.
4. Admit the entry only when the definition exists and definition dword
   `+120` is nonzero. For `LightningRoguePassive`, packaged Lua supplied this
   field with `SPID("forkedLightning")`.
5. Cache the entry by runtime instance ID (`i[2]`). The cached timing/count
   fields come from the modifier entry; the cached icon is copied from
   definition `+120`.
6. On a rebuild, construct ActionScript objects containing exactly
   `stackCount`, `icon`, `cooldownPercent`, and `cooldownRemaining`, then call
   `SetBuffData`. Incremental timing/count changes call
   `SetModifierCooldownPercent`.
7. Remove cached instance IDs no longer present in the currently scanned map,
   rebuild, and clear component dirty byte `+1156`.

The decisive filter is the nonzero locally registered `icon`, not the passive
profile slot, `descriptors`, `source`, or a passive-specific event. Neither
`0xA7 PlayerCharacterDeploy` nor a `ServerEventDef` creates this HUD entry.

## Deployment, squad swap, and the two lifetimes

There are two different lifetimes which must not be collapsed.

### Modifier lifetime

The underlying `LightningRoguePassive` instance begins when its `0xa2` is
accepted. The packaged definition is `Unique`, declares `OnAgentDeath`, and
waits forever; it has no natural timeout and its empty `Deactivate` does not
self-delete. Explicit native removal follows:

```text
sub_9E12D0(instanceID)
  -> sub_9E11F0(instance)
     -> sub_9E09D0(instance)       // Deactivate / owned resources
     -> sub_9DDD40(instance)       // attached attribute cleanup
     -> sub_9DFBA0(target, ID)
        -> sub_A1FD10(target, ID)  // logical 38 / wire 0xa4
```

Agent destruction removes bound instances through the same lifecycle.
Unique replacement removes the old same-GUID instance before creating the new
one. A squad selection change by itself is not an `0xa4` and does not prove
that a modifier attached to the reserve hero was destroyed.

### HUD visibility lifetime

The HUD shows the instance only while both conditions are true:

- the instance is present on the agent currently resolved as controlled; and
- its registered definition has a nonzero icon.

Changing the controlled object makes `sub_4247C0` compare its cache against a
different modifier map. It removes the old hero's cached icons and adds the new
hero's icon-bearing instances in the same rebuild. Therefore an old passive
can cease to be visible on squad swap without an `0xa4`; its underlying
instance may still live on the reserve object. Conversely, deleting an
instance on the still-deployed hero removes it on the next HUD update.

### Ordering constraints and the unproven sender edge

The client establishes these hard ordering constraints:

```text
ObjectCreate(target) must precede 0xa2(target, ...)
0xa2 must be accepted before that instance can appear in MaxisHUDBuffs
controlled-object change makes the HUD switch which modifier map it scans
0xa4 ends the underlying instance; the following HUD scan removes it
```

The already recovered build-103 deployment presentation updates reflected
`LabsPlayer` field `9` (the `+0x1238` controlled object) before
`0xA7 PlayerCharacterDeploy`; a newly introduced squad object must itself be
created before that binding can resolve. For a gap-free new deployment, its
passive `0xa2` must already be installed before the first HUD scan after the
field-9/`0xA7` selection change. If it was installed earlier on the reserve
object, no new modifier packet is needed at swap time. If authority chooses a
deployment-scoped instance instead, the receiver-safe sequence is old `0xa4`,
new `0xa2`, field `9`, then `0xA7`, after both objects exist.

No retail packet capture in the inspected workspace shows the original
server's `0xa2` position relative to field `9` and `0xA7`. The client binary
proves the dependencies and view behavior above, but it cannot prove whether
the retail server kept passives alive on reserve objects or recreated them on
every swap. That sender-policy edge must remain distinct from the recovered
HUD contract; inventing an alternate passive event is not justified.

## Minimal recovered contract

For Blitz's right-side icon, the only recovered creation input is an ordinary
`0xa2` on the relevant live hero object with
`ModifierGUID=0xEFC98B5C` and a unique runtime instance ID. The remaining
runtime fields must come from authoritative modifier creation rather than from
the profile's `ability_passive` number. Build 103 resolves the icon locally as
`0x190B0DC0`, shows it through `SetBuffData`, refreshes timing/stack state from
`0xa3`, and removes the underlying instance only from `0xa4` or object teardown.
Squad selection independently controls which object's surviving modifier set
is visible.
