# Campaign squad-swap ability blink

## Result

The tutorial-specific carryover has a recovered server-visible clear path:
after the first accepted Ride the Lightning cast, `ObjectiveUpdated` hides the
ability lesson objective. The shared gameplay path must preserve that update;
otherwise the action-bar pulse remains active and is reused when Sage replaces
Blitz at the same HUD indexes. This clear is deliberately idempotent and is
sent for every accepted tutorial Ride cast.

The separate generic campaign behavior remains client-owned as described
below. It must not be confused with the tutorial lesson carryover.

Build 103 is not treating the replay as an unbeaten level, and the Sage swap
does not execute the authored campaign unlock callbacks. The remaining blink
comes from the generic client `HowToAbility` popup-tip script. That script uses
`nPlayer.IsAbilityIndexUnlocked`, not `HasBeatenThisLevel`, calls
`PlayAbilityBlink` for the actions it finds available, and gives the resulting
attention presentation a `300`-second lifetime.

The native per-action play/stop bytes are transient requests local to the UI
manager. The action-bar update consumes them and the main UI update clears them
in the same frame. The lasting state is in `HUD_ActionBar`'s separate
`tutorialAbilityMC1..6` movie clips: `PlayAbilityBlink` sends a clip to
`"pulse"`, while only `StopAbilityBlink` sends it to `"outro"`.

`PlayerCharacterDeploy` changes the selected deck index and controlled object.
On the next action-bar update, build 103 repopulates the selected hero's icon,
label, key, visibility, enabled state, and cooldown. It neither writes a native
play/stop byte nor resets the tutorial movie clips. The clips are keyed by HUD
action index rather than creature identity, so an already-pulsing overlay
remains above the same index when Sage's action data replaces Blitz's. This is
the apparent swap-time reassertion. The 16:51 replay's lack of a new
`ability_blink_play_call` during the switch agrees with that path: selection
reveals/reuses existing HUD presentation; it does not make a second Lua/native
play call.

There is **no server packet and no reflected `LabsPlayer` or character field
that clears ability-attention state**. It is strictly client-owned. Do not add
an invented `LabsPlayerUpdate`, `ServerEvent`, cooldown message, Fang write, or
asset change to clear it.

## Runtime evidence and the actual caller

The 16:51 normal campaign replay recorded in `notes/playtest/latest.md` deployed
`zelems_1` with durable chain progression `4`. The current simulator difficulty
/ level threshold was `1`, and no ability-boundary mutation or blink-call trace
occurred during the Sage switch.

The replay predicate is exact:

- `HasBeatenThisLevel` is native `sub_A054C0` (`0x00A054C0`).
- It resolves the simulation player and compares reflected `uint32
  mChainProgression` at player `+0x125c` (`+4700`) with the value returned by
  `sub_9BCC50` (`0x009BCC50`).
- The comparison is `mChainProgression >= currentDifficulty`. Therefore
  `4 >= 1` is true.
- `mChainProgression` is top-level `LabsPlayer` reflection field `17`.

The campaign client unlock scripts (`Tutorial_RandomUnlockClient`,
`Tutorial_SupportUnlockClient`, and `Tutorial_SoloSupportUnlockClient`) return
when the local player has beaten the level. They cannot explain a replay-time
call under these inputs.

The independent caller is runtime `content.db` Lua chunk `1021`:

| Property | Value |
| --- | --- |
| `server_data_resource_id` | `14608` |
| source | `PopupTip/0x467AA8EE.lua` |
| size | `4013` bytes |
| SHA-256 | `df9ae1801b464eb7b6b272873e96772ebec82e020294bbca1b57d60888f349aa` |
| registered popup | `HowToAbility` |
| configured `blinkTime` | `300.0` seconds |

Its `AbilityPopupTip` closure first calls
`nPlayer.IsAbilityIndexUnlocked`. Depending on the highest available group and
`numTimesFired % 3`, it calls `nUIManager.PlayAbilityBlink` for:

- `Ability_Enrage1` and `Ability_Enrage2`;
- `Ability_Support1`, `Ability_Support2`, and `Ability_Support3`; or
- all five actions together.

The earlier entry trace's raw accepted values `0x00010002`,
`0x00010003`, `0x00010006`, `0x00010007`, and `0x00010008` correspond to
indexes `2`, `3`, `6`, `7`, and `8`. That five-call shape is instruction-for-
instruction present in this generic popup closure. The closure contains no
`HasBeatenThisLevel` call and does not read `mChainProgression`.

The popup's tick/success closures own the matching stop path. Once the retained
start time plus `blinkTime` is less than game time, each closure calls
`StopAbilityBlink` for all five action types and resets its local start time to
`-1`. The same script can also route through its local `LessonLearned` /
`LessonNotLearned` popup state based on the controlled object's mana/health.
These are client script transitions, not GMS messages.

`nPlayer.IsAbilityIndexUnlocked` is registered at `sub_4EC850`
(`0x004EC850`). It resolves the local `LabsPlayer` and calls
`sub_9C2A60` (`0x009C2A60`). The latter reads top-level player ability boundary
field `21` at player `+0x1370` (`+4976`) and applies the current-deck support
index rule. Field `21` can make a proposed blink call fail availability
validation, but it is not a blink-state field and cannot clear an already
playing HUD clip.

## Native play/stop request bytes

`sub_4EDFE0` (`0x004EDFE0`) registers the Lua-visible
`nUIManager.PlayAbilityBlink` and `StopAbilityBlink` methods to
`sub_4EC0D0` and `sub_4EC150`, respectively.

`sub_425980` (`0x00425980`) returns the UI state block at
`dword_11BEC40 + 0x24`. For action index `i`:

| Meaning | UI-state offset | Manager-relative offset | Writer |
| --- | ---: | ---: | --- |
| play request | `+0x0d+i` | `dword_11BEC40 + 0x31+i` | `sub_4EC0D0` (`0x004EC0D0`) |
| stop request | `+0x16+i` | `dword_11BEC40 + 0x3a+i` | `sub_4EC150` (`0x004EC150`) |

Both wrappers require exactly one Lua argument, resolve the local player, and
call `sub_9C2A60(player, i)`. They set their byte only when that availability
predicate succeeds. Neither wrapper reads chain progression or a server event.

`sub_422820` (`0x00422820`), the action-bar frame update, consumes the bytes:

1. For each `i=0..8`, a set play byte calls the Scaleform method
   `PlayAbilityBlink(i)`.
2. A set stop byte enters the native stop branch, which calls the Scaleform
   method `StopAbilityBlink(j)` for every `j=0..8`.
3. The main UI update `sub_42A730` (`0x0042A730`) later calls
   `sub_41DC80` (`0x0041DC80`). Its terminal cleanup zeros health, mana, and
   overdrive request bytes plus both nine-byte action ranges `+0x0d..+0x15`
   and `+0x16..+0x1e`.

Consequently these native bytes are one-frame command latches, not durable
per-character or per-player state. A stale byte cannot survive until a later
squad swap. The persistent pulse is the state reached by the Scaleform clip
after the latch has been consumed.

## `HUD_ActionBar` movie state

The exact movie resource is `FlashUI.package` ordinal `493`, type
`0x278CF8F2`, group `0`, instance `0x34DE0FAC`. Its stored CFX payload has
SHA-256
`58e88c5575c98bf243f1224944cf4a30331be34d79e2f0067dd39694f7fd0058`;
the decoded payload has SHA-256
`b62667f79b607712e90ee0f664820ca2e3c599d1ede01b74eb398d2dd09812f2`.

Its AVM1 methods are exact:

```text
PlayAbilityBlink(index):
    AbilityTutorialMovieFromIndex(index).gotoAndPlay("pulse")

StopAbilityBlink(index):
    AbilityTutorialMovieFromIndex(index).gotoAndPlay("outro")
```

`AbilityTutorialMovieFromIndex` maps fixed HUD indexes as follows:

| Action index | Tutorial movie |
| ---: | --- |
| `0` (`Ability_Basic`) | `tutorialAbilityMC1` |
| `2` (`Ability_Enrage1`) | `tutorialAbilityMC2` |
| `3` (`Ability_Enrage2`) | `tutorialAbilityMC3` |
| `6` (`Ability_Support1`) | `tutorialAbilityMC4` |
| `7` (`Ability_Support2`) | `tutorialAbilityMC5` |
| `8` (`Ability_Support3`) | `tutorialAbilityMC6` |

These clips are separate from `AbilityMC1..6`, which hold the selected
character's action artwork and data. `SetAbilityData` obtains an
`AbilityMovieFromIndex(index)` and changes its icon, ability name, key label,
and cooldown state. It never calls `StopAbilityBlink`, changes a
`tutorialAbilityMC*` timeline, or stores a creature identity in that tutorial
clip. Thus replacing Blitz's index-`2`/`3` data with Sage's index-`2`/`3` data
does not replace or stop `tutorialAbilityMC2`/`3`.

## Deployment and character-selection trace

`PlayerCharacterDeploy` is wire `0xA7`; build 103 dispatches it to
`sub_53ACA0` (`0x0053ACA0`). The fixed nine-byte payload after the packet ID is:

```text
<player-index:u8> <creature-index:u32-le> <object-id:u32-le>
```

After resolving the player and object, the handler:

- writes the selected creature index to `LabsPlayer +0x0c`;
- writes `-1` to the queued creature index at `LabsPlayer +0x10`;
- writes the resolved controlled-object handle at `LabsPlayer +0x1238`
  (dword index `1166`);
- refreshes the selected creature/request presentation through
  `sub_53AAC0`, the local object manager, and action-input reset
  `sub_4DEC70(0, 0)`.

It never calls `sub_4EC0D0` or `sub_4EC150`, never accesses the
`sub_425980` play/stop ranges, and never calls the Scaleform blink methods.

On the next `sub_422820` frame, the controller compares the selected creature
index with its cached dword at controller `+0x44` and the selected character
identity at character `+0x5c` with its cache at controller `+0x48`. A mismatch
causes `sub_421600` (`0x00421600`) to repopulate action indexes
`0`, `2`, `3`, `6`, `7`, and `8`, followed by ordinary cooldown, visibility,
and enabled-state updates. None of those calls touches the tutorial clips.

This gives the complete swap sequence:

```text
earlier HowToAbility popup
  -> sub_4EC0D0 sets one-frame play byte(s)
  -> sub_422820 calls HUD_ActionBar.PlayAbilityBlink(index)
  -> tutorialAbilityMC(index) enters looping "pulse"
  -> sub_41DC80 clears the one-frame native byte(s)

later PlayerCharacterDeploy(Sage)
  -> sub_53ACA0 changes selected index/object
  -> sub_422820 repopulates Sage's AbilityMC data at the same HUD indexes
  -> tutorialAbilityMC(index) remains in "pulse"
  -> no new PlayAbilityBlink call is required
```

## Packet/reflection audit and implementation boundary

The relevant existing server inputs are all negative:

| Input | What it changes | Why it cannot clear the blink |
| --- | --- | --- |
| `0xA7 PlayerCharacterDeploy` | selected deck index and controlled object | no access to native play/stop bytes or tutorial movie clips |
| `0xA1 LabsPlayerUpdate`, field `17` | `mChainProgression` | used by `HasBeatenThisLevel`; generic `HowToAbility` does not read it |
| `0xA1 LabsPlayerUpdate`, field `21` | ability availability boundary | gates whether a new play/stop wrapper call is accepted; does not stop an existing clip |
| `0xA1` character fields | creature identity, resources, deploy cooldown, ability points/ranks | no field maps to the UI-state request bytes or Scaleform tutorial timelines |
| `0xC1 CooldownUpdate` | ability/global cooldown maps | consumed by cooldown rendering and admission, not attention presentation |
| `0x9B ServerEvent` | typed client events authored elsewhere | there is no reflected blink/stop member in its payload and no generic event decoded into these UI bytes |

No build-103 reflection metadata registers the `sub_425980` play/stop arrays,
and no GMS receiver writes them. The only identified writers are the two local
Lua wrappers; the only identified direct reset is the local end-of-frame
cleanup. Durable stop state exists only in the ActionScript movie timeline and
is reached through the local `StopAbilityBlink` callback.

Therefore the implementation boundary is client presentation:

- keep Fang observation-only; do not write the UI-state bytes or intercept
  `PlayAbilityBlink`;
- do not send an invented packet, field-`21` churn, fake cooldown, or synthetic
  `ServerEvent`;
- do not modify `FlashUI.package`, `UI.package`, or any other shipped asset;
- keep normal server authority limited to truthful progression, ability
  availability, deployment, and accepted gameplay mutations.

If this retail popup behavior is corrected later, the evidence target is the
client-owned `HowToAbility` popup lifecycle and its learned/reset persistence,
not campaign deployment or the RakNet schema. A server-side change is justified
only if separate evidence recovers a real account/profile field already
consumed by that popup system. No such field or packet is presently identified.
