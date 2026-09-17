# Campaign action-bar dim artwork

## Result

The campaign icons are valid artwork being deliberately disabled by the
action-bar movie. The defect is not missing resource loading, resource-name
formatting, descriptor selection, or stale movie data. Campaign setup sends
all three permanent passive modifiers with the final `ModifierCreated` byte
set to `1`. Build 103 copies that byte to modifier-entry `+44`, and
`sub_4DEFE0` treats any unexpired entry with `+44 != 0` as a global action
blocker. Because these passives have duration `0`, the blocker never expires.
`sub_4DF7A0` consequently remains false, `sub_422820` passes false to
`sub_41E0A0`, and Scaleform receives `SetAbilityEnabled(index, false)` for
every populated action.

The smallest fix is server-side and one byte wide: in campaign setup's passive
`ModifierCreatedMessage`, send the final flag as `0` for
`LightningRoguePassive`, `SupportHealerPassiveModifier`, and
`EnergySentinelPassive`. In current code this means removing/changing
`IsBound: true` in the passive creation at `server/gameplay_udp.go:13883`.
The Go field name is misleading for this use: the native sender supplies
definition byte `+90`, not a generic statement that a passive is attached to
an agent. No Fang resource remap or `SetAbilityEnabled` override is needed.

## Descriptor-to-Scaleform path

`sub_422820` calls `sub_421600` for indexes `0`, `2`, `3`, `6`, `7`, and `8`
when the selected deck index or selected character identity changes.
`sub_421600` performs the following path:

1. `sub_9C25C0(player, index, -1)` selects the ability GUID.
2. `sub_9DAC00(guid)` resolves the registered descriptor. A null descriptor
   returns without calling Scaleform.
3. Descriptor dword `+120` becomes the resource instance. The function fixes
   type `796721156` (`0x2F7D0004`) and group `161348501`
   (`0x099DFB95`). The older action-bar note transcribed both hex constants
   incorrectly as `0x2F7DDC04` and `0x099DE095`.
4. `sub_7AE040` formats `group!instance.type`: first `0x%08x` for the group,
   then `!0x%08x` for the instance, then either the registered type name or
   `.0x%08x` as the fallback. For the installed raster type, Blitz basic's
   input is the identity `0x099dfb95!0xab6f92f1` with type `0x2f7d0004`.
5. Descriptor dword `+100` supplies the localized ability label, the numeric
   action index supplies argument three, and `sub_421500` supplies the
   localized key label.
6. `sub_54D240` invokes
   `SetAbilityData(iconResource, abilityLabel, index, keyLabel)` with four
   Scaleform values. Enabled state is not an argument to this call.

The 14:35 Fang records show a resolved descriptor and nonzero label/icon for
all six calls on each selected hero. The three character-specific sets are:

| Hero | Index `0` GUID/icon | Index `2` GUID/icon | Index `3` GUID/icon |
| --- | --- | --- | --- |
| Blitz | `0x33CB6B29` / `0xAB6F92F1` | `0x43C6B5F7` / `0xCC9DA84B` | `0xD02C30E2` / `0xDF9E45AF` |
| Sage | `0x34CA1979` / `0xEA24A69A` | `0x1947A88C` / `0x66C93285` | `0x5E0C8BF6` / `0xE5D02F53` |
| Goliath | `0x3877506D` / `0x632948A3` | `0x1C46A6FE` / `0x5F395C7D` | `0xCC1E09D0` / `0xD3A626A1` |

The shared deck-support calls resolve GUIDs/icons
`0xA5AAE182/0x225CA176`, `0x6D05F872/0xBB32B362`, and
`0xD8A4785B/0x0547F0B9` for indexes `6`, `7`, and `8`.

`darkrun inspect bin/game/Data/UI.package` finds every one of those twelve
instances with exact type `0x2F7D0004` and group `0x099DFB95`. The inventory
also contains size/style siblings under `0x099DFB88` and, for most instances,
`0x0B4FF337`; those are not what `sub_421600` requests. The requested
`0x099DFB95` members are present and have nonzero stored/decoded sizes. Thus
the formatted identities exactly match installed package members; there is no
missing package member and no type/group correction to make.

## Working tutorial comparison

The working tutorial Blitz uses the same `LightningRogueBasic` descriptor as
campaign Blitz. Its GUID is `0x33CB6B29`, and the campaign trace resolves that
descriptor to icon `0xAB6F92F1`. Therefore the working and failing paths share
the same descriptor, formatted resource identity, installed UI package, and
`HUD_ActionBar.swf` receiver. The tutorial's live-confirmed red Voltic Slash
and Ride the Lightning icons prove that the movie can render this resource
family.

The meaningful setup difference is modifier state. Tutorial setup creates the
controlled Blitz, publishes health/attribute state, and does not inject the
campaign passive `0xA2`. Campaign setup creates Blitz, Sage, and Goliath and
then sends one permanent passive `0xA2` per object with the final byte `1`.
The 14:35 trace shows those three 37-byte messages immediately after each
hero's positive combatant and attribute baselines, before the controlled-object
refresh and deploy. This difference reaches the exact enabled predicate below;
the descriptor/resource path does not.

## Exact enabled/dim predicate

`sub_422820` computes one global gate with `sub_4DF7A0`, then computes a
per-index gate with `sub_4DEE60`. It passes their conjunction as the final
argument to `sub_41E0A0`. `sub_41E0A0` first requires the index's descriptor to
resolve, suppresses duplicate values with its cached byte at controller
`+85+index`, and calls `SetAbilityEnabled(index, enabled)`.

In domain terms, the exact predicate is:

```text
enabled(index) =
    controlled agent exists
    and current HP > 0
    and MindControlled (attribute 57) <= 0
    and no active modifier entry has entry[+44] != 0
    and not insufficient mana for index
    and not unavailable reserve-support index
    and (index == 0 or Silence (attribute 14) <= 0)
```

The native details are:

- `sub_4DF7A0` requires `sub_4E5800()` to resolve the controlled agent,
  `sub_9D8F50(agent) > 0`, attribute `57 <= 0`, and `!sub_4DEFE0()`.
- `sub_4DEE60(index)` returns false when `sub_4DEA70(index)` reports an
  unaffordable mana cost or `sub_4DEBC0(index)` reports an unavailable
  reserve support. Otherwise it returns true when the agent/attribute
  component is absent, Silence is nonpositive, or `index == 0`.
- `sub_4DEFE0` walks the controlled agent's modifier map at agent `+692`.
  It returns true for an entry when node byte `+44` is nonzero and either its
  duration dword at `+32` is zero or `start(+24) + duration(+32)` is later
  than the gameplay clock.

The campaign state rules out every other persistent false input. Wire `0x97`
gives all three heroes positive HP/mana, wire `0x96` gives positive maxima and
does not set attributes `14` or `57`, and wire `0xA1` refreshes controlled
object `1` before deploy. A mana failure could dim individual paid abilities
but not index `0`; Silence could dim non-basic actions but explicitly cannot
dim index `0`; reserve availability affects only `6`-`8`. The permanent
entry-`+44` blocker is the only observed input that forces every action false
across all three heroes.

## Why the passive flag is wrong

`sub_537780`, the `ModifierCreated` receiver, copies the packet's final byte
into modifier payload `+36`, which is tree-node `+44` when traversed by
`sub_4DEFE0`. For locally requested modifiers, `sub_9E0FB0` fills the runtime
entry's byte `+44` from descriptor byte `+90` and passes that byte as the final
argument to the native `ModifierCreated` sender. Registration finalization in
`sub_9DA000` fills descriptor `+90` from the presence of the Lua
`IsAbleToHit` member.

Runtime `content.db` independently resolves the three campaign passives to
`LightningRoguePassive` (chunk `845`), `SupportHealerPassiveModifier` (chunk
`270`), and `EnergySentinelPassive` (chunk `263`). Their indexed string
constants contain their registration and callback bodies but no
`IsAbleToHit`. Blitz's chunk is pinned to
`d797193ce6fdb9c7833ed3356913fe30e3c3ba1f871d3e81fbc8003e82c276d0`;
it registers a unique, on-agent-death passive and waits forever. The native
sender would therefore publish final byte `0` for these definitions. Darkspin
instead hard-codes `IsBound: true` for each campaign passive and gives it
duration `0`, creating precisely the permanent blocker tested by
`sub_4DEFE0`.

The final byte is not used to select the passive HUD icon, modifier GUID,
stack count, or lifetime. Correcting it to `0` preserves the passive entry and
right-side passive artwork while allowing the action bar's normal HP, mana,
Silence, reserve, and active-blocker rules to operate.

## Fix and verification

No client or Fang patch is justified. The minimal server change is:

```text
campaign passive ModifierCreated final flag: 1 -> 0
```

After that change, a focused campaign entry should verify:

1. `0xA2` for Blitz, Sage, and Goliath ends in `00`.
2. All populated action icons render at normal brightness at rest.
3. Paid abilities still dim when mana is insufficient, non-basic abilities
   dim under Silence, and support indexes retain their deck availability.
4. The passive modifier icons remain present.

No unresolved authority remains for this correction, so `notes/help.md` does
not need an entry.
