# Campaign ground-click regression (build 103)

## Conclusion

The 2026-07-22 14:35 campaign failure is not a stale combat-action slot and is not caused by the initial `LabsPlayerUpdate`. The latest Fang trace shows the active (`+4/+8`) and queued (`+52/+56`) definitions at zero immediately before the bad target-zero requests. A plain terrain click can reach the targetless-basic constructor from `sub_44CFA0` only when the input manager reports logical action `16` as down. That branch runs before object, terrain, tutorial, or movement predicates.

The repeated Blitz requests are especially diagnostic. A selected ground ability is consumed once, but `sub_44C7F0` arms the held-basic bytes and `sub_44D740` repeats the basic while action `16` remains logically down. The observed target-zero burst therefore matches a false/stale or misbound logical Shift state, not a server response that left definition `22333792` queued.

The smallest safe correction is a Fang-side guard at the `sub_44CFA0` logical-action-16 branch: honor it only when an actual Shift key is down. When the guard is false, resume the original click resolver so a valid enemy still gets pursuit/basic, an interactable still gets interaction, and empty ground reaches `sub_44A870` and native type-3 movement. Do not translate target-zero ability packets into movement.

## Run evidence

The campaign starts at 14:35:03, receives its initial player update at 14:35:08, and starts gameplay at 14:35:11. Normal targeted Blitz pursuit/attacks occur at 14:35:29-31. Beginning at 14:35:35.036, ordinary ground clicking produces a rapid target-zero Blitz-basic stream until 14:35:37. The same pattern recurs after switching to Sage at 14:37:02-07. No campaign type-3 movement request appears in that interval.

The corresponding Fang trace session begins at `time_ms=101692671`. Relevant snapshots include:

- `101713515`: native type-2 response; active, response, and queued definitions are all zero.
- `101714234`, `101715953`, and `101719390`: native type-1 acknowledgements name Blitz basic `22333792` only in the bounded response slot; active and queued remain zero.
- `101719984`: immediately inside the bad-click interval, Fang records active, response, and queued definitions all zero.
- `101720187`: the next native type-1 acknowledgement again has only response definition `22333792`; active and queued remain zero.

This supersedes the stale-action theory in `notes/campaign/ground-movement.md` for build 103. The combat slots Fang records are downstream simulation/response state. They are not predicates read by `sub_44CFA0` when it decides what a click means.

## Exact `sub_44CFA0` decision order

`sub_4502C0` is the input-frame dispatcher. It invokes `sub_44CFA0` on the press edge reported by the input manager's virtual `+60` query for action `1000` (left mouse). The exact order inside `sub_44CFA0` is:

1. Clear `dword_11BEF14`, the right-click fallback-position flag.
2. If `dword_11BEF24 != -1`, fire the selected action through `sub_44CDD0` and immediately reset the selected index to `-1`. This is the one-shot ground-fired/targeted-ability path.
3. Otherwise query virtual `+56` for logical action `16`. If true, call `sub_44C7F0(..., 1)` with target zero and set mouse mode `dword_11BEF0C=2`. No target or terrain predicate has run yet.
4. Otherwise resolve the clicked object with `sub_9D94F0`. Treat the click as non-target if the object is missing, `sub_44A750` rejects it, `sub_4E4C70` reports a viewport exclusion, or `sub_44A940` reports the UI/tutorial gate.
5. In the non-target branch, `sub_44A7C0` first recognizes an interactable/dead-clickable object and calls `sub_44CAF0`.
6. Otherwise cancel an armed held basic if necessary, clear the retained target, and call `sub_44A870`. This is the plain empty-ground path; it constructs native type-3 movement through `sub_4DFA50`.
7. A valid combat target instead calls `sub_44C860`, stores it in `dword_11BEF10`, and starts target pursuit/basic behavior.

Consequently, neither the tutorial gate nor a movement status lock can turn a plain click into `sub_44C7F0`. The action-16 branch is the only such routing predicate in `sub_44CFA0`.

## Predicate and writer map

| State or predicate | Meaning in this path | Writer / source |
| --- | --- | --- |
| Input action `1000`, virtual `+60` | Left-button press edge that enters `sub_44CFA0` | Trigger-set input event processing; `sub_50BEB0`/`sub_50BF70` feed press events through `sub_50BD60` |
| Input action `1000`, virtual `+64` | Current left-button down state used for hold/release | Trigger-set down-state arrays, updated by the same input event path |
| Input action `16`, virtual `+56` | Logical Shift/force-basic state; true diverts a click directly to target-zero basic | Keyboard/control binding input manager, not any Blaze packet; press/release events are propagated through `sub_50BD60` (`sub_50BEB0` press and `sub_50BF10` release). The current trace does not expose which physical binding is falsely driving action `16` |
| `byte_11BEF08` | Previous-frame left-button state for release-edge handling | `sub_4502C0` |
| `byte_11BEF09` | Previous-frame right-button state | `sub_4502C0` |
| `byte_11BEF0A` | Previous-frame action-16 state | `sub_4502C0` |
| `dword_11BEF0C` | Mouse action mode (`2` left, `1` right) | `sub_44CFA0` / `sub_44D100`; release handling clears it |
| `dword_11BEF10` | Retained combat target for pursuit/repeat | Target branches in `sub_44CFA0` / `sub_44D100`; non-target, modifier, cancel, and validation paths clear/update it |
| `dword_11BEF14` plus `+18/+1C/+20` | Valid stored right-click fallback position and coordinates | `sub_44D100`; `sub_44CFA0` clears the flag, so it does not select a left-click basic |
| `dword_11BEF24` | Selected action-bar index | `sub_44D920`; initialized and cancelled to `-1`, and consumed/reset by `sub_44CFA0` / `sub_44D100` |
| `byte_11BEF28` | Movement press is armed | `sub_44A870`; `sub_44D600` clears it when the press becomes a hold |
| `dword_11BEF2C` | Movement hold timer | `sub_44A870` / `sub_44D600` |
| `dword_11BEF30/+34/+38` | Initial movement-click point | `sub_44A870` |
| `byte_11BEF3C` | Movement is in held/drag form; also passed as the action transform flag | `sub_44D600`; click/release/cancel paths clear it |
| `byte_11BEF44` | Initial basic press is armed | `sub_44C7F0` and targeted `sub_44C860`; `sub_44D740` converts or clears it |
| `dword_11BEF48` | Basic hold/repeat timer | Basic press and `sub_44D740` repeat logic |
| `byte_11BEF4C` | Repeating basic is active | `sub_44C970` / `sub_44CA80` and `sub_44D740`; movement/cancel/release paths clear it |
| `sub_44A750` | Valid live combat target | Reads object/component state: existence, positive health, opposing/allowed team, blocker state, targetable byte, and hit relation. It does not write click intent |
| `sub_44A7C0` | Interactable/dead-clickable object | Reads the interaction component and health/clickability predicates |
| `sub_4E4C70` | Optional cursor safe-viewport exclusion | Enabled by `byte_14CB160`, loaded from property hash `410109169`; bounds come from hashes `462207583`, `-871438339`, and `-1769798373` |
| `sub_44A940` | UI/tutorial input gate | Queries the UI/tutorial service with property/event hash `988711392` (`0x3AEE89E0`) and is true only for result `1`. It can disqualify a combat target, but then resolution continues toward interaction/movement, not basic |
| Combat definitions at action `+4`, response `+24`, queue `+52` | Applied and acknowledged simulation state | Combat descriptor/response processing downstream of the click resolver; no read of these fields occurs in `sub_44CFA0` |

`sub_4DF140` is the movement admission/status check reached only after the click has already selected movement. It checks for a controlled agent and positive simulation time, then active-action timing, component query `sub_9E3490(..., 15, 1)`, `sub_4DE8E0`, and timed response state from `sub_4DE910`/`sub_4DEC20`. Its outcomes can accept, defer, or reject movement (`1`, `2`, `-9998`, or `-9992`), but none calls `sub_44C7F0` or creates a type-7 ability descriptor. Status locks therefore cannot explain this regression.

## Why the repeating requests identify the branch

The selected-action branch cannot produce the observed stream: `dword_11BEF24` is reset to `-1` on the first click, and `sub_44CDD0` does not arm the held-basic bytes. By contrast, `sub_44C7F0` sets `byte_11BEF44`; `sub_44D740` promotes it after the hold threshold and repeats through `byte_11BEF4C` at roughly 50 ms internal intervals. If there is no valid retained target and action `16` is released, that loop cancels. The server log's repeated target-zero basics are the externally visible signature of this exact state machine.

The physical Shift key was not held during the reported clicks. Static control flow plus that observation therefore isolates the fault to the logical action-16 input value (stale state, bad release propagation, or a binding that is active without physical Shift). A definitive choice among those three input-subsystem causes needs one narrow Fang trace at the resolver boundary; it does not require another packet experiment.

## Campaign snapshot versus working state

The initial campaign `LabsPlayerUpdate` sends active slot `0`, controlled object `1`, camera-lock field `18=0`, ability count `5`, locked-deck count `3`, and three real creature records. The working tutorial update also sends active slot `0`, controlled object `1`, and camera-lock field `18=0`; its smaller ability/deck/creature counts reflect tutorial availability. Field `18` is written by the native `LockCamera`/`UnlockCamera` handlers and affects camera behavior, not `sub_44CFA0`.

The campaign-only values can make more action-bar abilities available and determine whose index-zero basic is constructed (Blitz before the switch, Sage afterward), but none writes input action `16`, `dword_11BEF24`, or the held-basic bytes. The working tutorial and campaign therefore agree on the packet fields relevant to initial control. Retail input follows the same native resolver; the divergence is client-local logical input state after entering campaign, not a missing `LabsPlayerUpdate` control field.

## Smallest correction and verification

No packet change is justified. Native type-1 acknowledgements are now correct, and the initial player packet has no field that writes the trigger-set action state.

The narrow Fang correction is:

1. At the action-16 test in `sub_44CFA0`, compute `effectiveShift = logicalAction16 && physicalShiftDown` using the actual left/right Shift high-bit state.
2. Take `sub_44C7F0` only for `effectiveShift`. If false, continue at the original object/terrain resolver.
3. Preserve the earlier `dword_11BEF24` branch unchanged so selected ground-fired abilities still work.
4. Apply the same physical-release safeguard to `sub_44D740` only if tracing shows logical action `16` remains stuck there; otherwise the click-entry guard alone prevents the erroneous repeat state from being armed.

Before patching, add a temporary Fang record at `sub_44CFA0` and `sub_44D740` containing selected index, logical action `16`, raw left/right Shift state, clicked object ID, results of `sub_44A750` / `sub_44A7C0` / `sub_4E4C70` / `sub_44A940`, and bytes `11BEF28`, `11BEF3C`, `11BEF44`, and `11BEF4C`. Expected verification cases are:

- plain empty-ground click: action `16=false`, `sub_44A870`, native type `3`;
- physical Shift + empty-ground click: `sub_44C7F0`, target-zero basic, native type `7`;
- selected ground ability: `sub_44CDD0`, never reinterpreted as movement;
- valid enemy: `sub_44C860` pursuit/basic;
- tutorial/UI gate or movement status lock: original rejection/defer behavior without synthesizing a basic.

No `notes/help.md` entry is needed: this is a client input-state defect with a conservative local guard, not an unresolved server-authority policy.
