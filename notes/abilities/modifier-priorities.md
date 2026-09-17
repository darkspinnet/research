# Modifier priorities and `StackModifier`

## Result

Build 5.3.0.103 does **not** define `nModifierPriorities.StackModifier` in the
packaged Lua corpus. There is therefore no numeric modifier-priority value to
recover for that qualified name. The similarly named, proven constant is:

```lua
nAbilityEventFlags.StackModifier = 32
```

`32` is an ability/modifier event-mask bit, not a priority. The previous
diagnostic attribution to `nModifierPriorities.StackModifier` conflated two
global tables used only a few instructions apart. The bounded result for the
requested member is thus **absent / Lua `nil` in the shipped build**, not an
unknown integer and not permission to invent one.

## Packaged global-definition evidence

The authoritative runtime `content.db` identifies GlobalDefinitions as
`lua_chunk.id=659`, server-data resource `14227`, source
`lua/0x2E64AA9E.lua`, 14,305 decoded bytes, SHA-256
`25927418c849f13c1f45592387f0d9d0fd1e8b67b6537daf0ac8b57d663d57b3`.
Direct extraction and Lua 5.1 disassembly show:

- the `nAbilityEventFlags` table at instructions `264..283`, including
  `StackModifier = 32` at instruction `269`;
- the `nModifierPriorities` table at instructions `665..755`;
- no `StackModifier` key in that priority table; and
- no `lua_static_property` row defining `StackModifier` anywhere in the
  packaged corpus.

The complete shipped priority table is:

| Priority names | Value |
|---|---:|
| `Recall` | 2000 |
| `Banished` | 1500 |
| `Caged` | 1250 |
| `RaisedIntoTheAir` | 975 |
| `Knockedback`, `Pulled`, `Teleported` | 950 |
| `HitReact` | 900 |
| `Stunned`, `Shocked` | 850 |
| `Slept` | 800 |
| `Terrified` | 750 |
| `Taunted` | 700 |
| `Silenced`, `HealingReduction` | 650 |
| `Enraged` | 625 |
| `Cursed`, `Weakened`, `Vulnerable` | 600 |
| `Poisoned`, `Diseased`, `Burning` | 550 |
| `Rooted` | 500 |
| `Slowed` | 450 |
| `Snared` | 400 |
| `Haste`, `ProjectilesSlowed`, `Resurrection` | 350 |
| `FollowingOwner` | 300 |
| `Aura` | 250 |

The retained decompile under
`bin/game/logs/recap_server-reference/game_server/res/data/serverdata/lua/GlobalDefinitions.lua`
independently reconstructs the same assignments. Its apparent
`StackModifier = 32` at line 274 is inside `nAbilityEventFlags`; the
`nModifierPriorities` assignment is a different table ending at line 760.

## Lightning Tempest cross-check

Packaged modifier chunk `897`, `Modifiers/0x768636B6.lua`, SHA-256
`3f43665f94d756759e07efdc1017585f386d2556d440cde1b6a2029c0af971b4`,
proves these separate assignments:

```lua
handledEvents = nAbilityEventFlags.StackModifier -- 32
modifierPriority = nModifierPriorities.Vulnerable -- 600
activationType = nActivationType.Stacks -- 4
maxStackCount = { 5, 5 }
```

The modifier lasts three seconds. Its event handler tests the current event
against `nAbilityEventFlags.StackModifier`, increments the stack count,
replaces its prior attribute modifier, applies
`0.1 * currentStackCount` to `EnergyDamageIncrease`, and resets duration.
This gives the `32` bit concrete semantics: it dispatches the notification
that an existing stacking modifier instance received another application.
It does not rank the modifier.

The earlier alias-probe failure at root instruction `18` is exactly the
`GETTABLE nAbilityEventFlags["StackModifier"]` operation. The constrained
simulator seeded `nModifierPriorities.Recall` but did not seed
`nAbilityEventFlags`; the resulting message reported only the missing key and
was misread as a priority lookup.

## Time Ravager cross-check

Packaged modifier chunk `893`, `Modifiers/0xFBFDCB83.lua`, SHA-256
`1e31483a329c374b339d96b8b3dc2b7432ba7813e6466a370b6e72b35bc2d62d`,
proves:

```lua
handledEvents = nAbilityEventFlags.StackModifier -- 32
modifierPriority = nModifierPriorities.Slowed -- 450
activationType = nActivationType.Stacks -- 4
maxStackCount = { 5, 5 }
```

Its duration is `{ 5, 5 }`. On the same event bit, its handler increments the
stack count, refreshes the attack-speed and movement-speed adjustments, and
resets duration. This independently confirms that `StackModifier` describes
stack-event handling while the actual priority remains the named status
priority (`Slowed`). The alias-probe failure at instruction `43` is likewise
the event-table lookup; the priority lookup occurs later at instructions
`55..58` and asks for `Slowed`.

## Native and reflection semantics

The build-103 reflection table identifies `handledEvents` as a four-byte field
at ability/modifier definition offset `0x1A4`. Native modifier-request routine
`sub_9E16C0` reads a definition integer at offset `0x178` and compares it with
the same offset in existing modifier definitions before admitting, rejecting,
or removing a conflicting instance. The retained reference layout labels
`0x178` as `modifierPriority`, matching the packaged Lua property and its
native use. The comparisons establish that larger priority values have
precedence over smaller values; equal priority is also sufficient for the
incoming definition in the observed comparison paths. This is a separate
mechanism from the `handledEvents` bitmask at `0x1A4`.

Neither the executable strings/native binder nor the reflection search shows
a native definition or late mutation of a Lua member named
`nModifierPriorities.StackModifier`. Runtime logs contain no contrary value.
They contain only the two constrained-decoder missing-key failures described
above. Consequently the evidence closes the original unknown as a namespace
error: seed/use event flag `32` only when implementing that separately
authorized work, and preserve Lightning Tempest priority `600` and Time
Ravager priority `450`; do not create a new priority constant.

## Reproduction artifacts

Read-only extraction and disassembly artifacts are retained under
`bin/game/logs/modifier-priorities/`, with the GlobalDefinitions payload at
`bin/game/logs/modifier-priorities-globaldefinitions.luac`. Existing native
evidence is in `bin/game/GameBin/Game.c` around `sub_9E16C0` and
`bin/game/logs/ida-ai-field-xrefs-all.log` around the `handledEvents`
reflection registration.
