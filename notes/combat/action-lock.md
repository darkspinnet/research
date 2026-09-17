# Campaign whole-input soft lock (build 103)

## Result

`sub_4D5EF0` at VA `0x004D5EF0` (RVA `0x000D5EF0`) is a real
whole-*combat-action-singleton* reset, but it is not a whole client-input reset.
It clears the state returned by `sub_4E4E00` at VA `0x004E4E00`:

- active action definition/sync/deadline at `+4/+8/+16..+23`;
- accepted response definition/sync/times at `+24/+28/+32..+47`;
- current local action handle at `+48`;
- queued pursuit definition/sync at `+52/+56`;
- the per-ability cooldown collection beginning at `+84`;
- global cooldown fields at `+576/+580/+584`.

It does **not** clear the upstream action descriptors, mouse/keyboard intent,
selected action-bar state, object-owned running-action list, player character
selection/deploy cooldown, or UI/death-selection state. Any one of those can
prevent a new command from reaching `sub_5370F0` even while Fang's post-`0xa8`
snapshot shows every singleton definition as zero.

The 2026-07-22 traces prove a client-side pre-send boundary, not one unique
latched field. The strongest reusable hazards are:

1. a targeted/interaction descriptor outside the singleton remains active;
2. held-basic or logical action-16 state keeps owning the click resolver;
3. an object-owned action/release entry remains live and makes all ordinary
   admission return deferred/rejected;
4. death/modal/deploy selection owns input before the gameplay sender; or
5. a delayed type-`2` `0xa8` cancellation, after one-byte token reuse, clears a
   newly staged descriptor.

Per-ability and squad deploy cooldowns are real local gates, but neither can
explain the combined loss of movement, abilities, and squad switching.
Movement does not consult an ability cooldown, and abilities do not consult a
reserve portrait's deploy deadline. A whole-input lock therefore requires an
earlier shared gate or more than one stale state.

Confidence: **exact** for the reset's writes and the enumerated global
descriptors; **high** for the pre-send localization and token-reuse hazard;
**medium** for object-owned action/release state as the repeated-combat cause;
**low** for choosing one causal field without the missing pre-admission
capture.

## Trace correlation

### Repeated pursuit/basic lock at 22:21

The last successful gameplay command in
`bin/darkspinner/darkspin/logs/darkspinner.log` is:

- `22:21:20.963`: Blitz basic accepted against object `10`;
- its `0xa8` response is sync `0x3d`, response type `1`, definition
  `0x33cb6b29`, followed by the normal combat, animation, and cooldown
  publications;
- `22:21:21.221`: the client acknowledges the resulting reliable datagrams.

Immediately before that success, the server had:

- rejected two busy basics;
- admitted pursuit with sync `0x2b`;
- suppressed one duplicate retry;
- accepted a 79-byte cancel which replaced that pursuit; and
- admitted the final basic with a new sync.

No movement, ability, interaction, or deploy command follows the
`22:21:21.221` acknowledgement, although the human attempted those inputs.
Thus “first missing input” has no packet body or sync token to compare: it was
lost before the application sender. The final success does not itself contain
an invalid response type; it uses the native type-`1` branch.

This is the most relevant sequence for token/action churn. It exercises
several one-byte action identities in less than a second and crosses pursuit
cancel -> new basic -> no later input.

### Ranged-action locks at 17:40 and 18:14

- At `17:40:34.377`, the last gameplay exchange is the acknowledgement of
  accepted Zetawatt Beam. Later Arc Weld, another active, movement, and squad
  selection produce no request.
- At `18:14:07.695`, targeted Shockwave is accepted; its final acknowledgement
  arrives at `18:14:07.970`. Movement, basic, active, and squad selection then
  produce no request.

These two boundaries rule out a server-side rejection loop after the lock:
there is no request to reject. They also rule out only the old duplicate
animation override, because the 18:14 reproduction occurred after that
override was removed. They do not distinguish an unreleased object-owned
action from an upstream input/modal latch.

### Current Fang capture cross-check

The current Fang file is
`bin/darkspinner/darkspin/logs/traces/game.jsonl`. Its latest process
starts at approximately 22:25 and therefore does not retain a decoded Fang
snapshot for the 22:21 boundary. It does show why the existing snapshot is
insufficient:

- at `time_ms=129992812`, Ride the Lightning receives sync `3`, type `1`;
  active and queued definitions are zero, the bounded response definition is
  `22153248`, and movement continues afterward;
- at `time_ms=129996031`, an out-of-range pickup receives sync `4`, type `2`;
  at `129996046`, active, response, and queued definitions are all zero, while
  a retained target handle remains nonzero.

The hook records singleton fields only after `0xa8`. It does not record
`byte_1438A4C`, `byte_1438AC0`, the movement descriptor, click/hold globals,
the selected action index, the controlled object's running-action list, UI
modal ownership, or deploy cooldown. The current file also contains no
`chat_reset_*` event, so the successful local `/reset` call reported in the
20:02 playtest cannot be re-proved from this reset capture.

## Gates outside `sub_4D5EF0`

Addresses are build-103 image VAs; parenthesized values are Fang/module RVAs.
Data RVAs subtract the image base `0x00400000`.

### Pending descriptors and delayed cancellation

| State | Exact address | Admission/retirement behavior | Reset coverage |
| --- | --- | --- | --- |
| ordinary movement intent | `dword_1438A0C..1438A1C` (`RVA 0x1038A0C`) | `sub_4DFA50` (`0x004DFA50`) installs it; `sub_4DF800` (`0x004DF800`) waits for `sub_4DF140 == 1`, sends/applies it, then clears the active low byte | not cleared |
| ability/generic targeted descriptor | `byte_1438A4C..1438A4F`, payload through `0x1438A74` (`RVA 0x1038A4C`) | `sub_4DFBE0` (`0x004DFBE0`) installs it; token is `byte_1438A4E`; `sub_4DF800` sends once and retires it after admission/cancel | not cleared |
| auxiliary/interaction/pickup/deploy descriptor | `byte_1438AC0..1438AC2`, type at `0x1438AC4`, payload from `0x1438AC8` (`RVA 0x1038AC0`) | `sub_4DFB40` (`0x004DFB40`) installs it; token is `byte_1438AC2`; `sub_4DF800` sends it before waiting for local completion | not cleared |
| shared next-token counter | `byte_1119510` (`RVA 0x00D19510`) | incremented by both descriptor installers; wraps from `0xff` to `1`, so the effective identity space is 255 values | not cleared |
| last action user data | `dword_1438B1C` (`RVA 0x1038B1C`) | copied into later action bodies; response types `1`/`2` can replace it | not cleared |

`sub_4DEC70` at `0x004DEC70` is called by the type-`2` branch of
`sub_4D64D0` (`0x004D64D0`). It compares only the response's one-byte sync
against `byte_1438A4E` and `byte_1438AC2`; a match clears the corresponding
current active byte. It has no request generation, object ID, definition, or
time comparison.

Therefore a delayed cancellation remains semantically live after
`sub_4D5EF0`. If 255 later staged descriptors reuse its sync byte,
`sub_4DEC70` can clear an unrelated new descriptor before its first send. The
same matching type-`2` branch also clears singleton active/queued slots when
their sync byte matches. This is an exact ABA/token-reuse mechanism; whether
the July 22 lock actually crossed a full 255-token cycle remains uncaptured.

The existing worktree's uncommitted Fang experiment clears only the three
descriptor active bytes. That is evidence that these globals were already
suspected, not a complete recovery: it leaves click/hold intent, running
object actions, UI/deploy state, and delayed response reuse untouched.

### Held basic, combo/release, and selection intent

The input-frame dispatcher is `sub_4502C0` (`0x004502C0`). The click resolver
`sub_44CFA0` (`0x0044CFA0`) chooses a selected action, forced/Shift basic,
combat target, interaction, or ordinary movement before it reaches any
singleton action slot.

| State | Address | Effect |
| --- | --- | --- |
| prior input edges | `byte_11BEF08..0A` | previous left, right, and logical action-16 states |
| current mouse mode/target | `dword_11BEF0C`, `dword_11BEF10` | owns left/right action mode and retained combat target |
| fallback point | `dword_11BEF14..20` | retains a right-click fallback point |
| selected action-bar index | `dword_11BEF24` | any value other than `-1` diverts the next click through `sub_44CDD0` (`0x0044CDD0`) |
| movement press/hold | `byte_11BEF28`, timer/point `0x11BEF2C..38`, `byte_11BEF3C`, `dword_11BEF40` | owns click-to-move and drag/held locomotion intent |
| basic press/repeat | `byte_11BEF44`, timer `dword_11BEF48`, `byte_11BEF4C` | `sub_44D740` (`0x0044D740`) promotes the press into repeated basics and release/cancel behavior |
| logical action `16` | input manager returned by `sub_425630`, virtual query `+56` | precedes target/terrain resolution and routes a plain click directly to targetless basic |

`sub_451A70` (`0x00451A70`) zeroes the full `0x4c`-byte block beginning at
`byte_11BEF08` and restores `dword_11BEF24=-1` during a larger scene/state
initialization. It is not a safe standalone recovery call: it also changes
scene and UI state, installs handlers, and performs other initialization.
Copying its blanket `memset` into Fang would clear fields whose meanings have
not all been proved.

The combat combo/release gate also has an object-owned half. The controlled
agent's action collection pointer is at object `+692`. `sub_4DEFE0`
(`0x004DEFE0`) scans live entries; `sub_4DF0A0` (`0x004DF0A0`) reports their
remaining time. `sub_4DF140` turns a live entry into movement defer/rejection,
and `sub_4DF1F0` does the same for targeted actions. This collection is not
inside the reset singleton. The exact owner-specific cancellation/destructor
for a stuck entry has not been recovered, so Fang must not erase object
`+692` or its nodes.

### Modal, viewport, death, and deploy selection

- `sub_44A940` (`0x0044A940`) is the UI/tutorial input predicate used by the
  click resolver. A true result disqualifies combat-target routing.
- `sub_4E4C70` (`0x004E4C70`) applies the optional safe-viewport exclusion
  controlled by `byte_14CB160`. It changes target classification, not the
  singleton.
- `dword_11BEF24` is the persistent action-bar selection described above.
- A death/survivor dialog and other Flash/modal states can own keyboard or
  portrait input before `sub_4502C0`/`sub_5370F0`. The 22:27 run proves this
  boundary operationally: after HP zero, selecting Goliath emitted no deploy
  request. The current Fang trace has no modal-stack or callback-entry hook,
  so its exact state field is not yet identified.
- Reserve-character selection is independently admitted by `sub_9C2B30`
  (`0x009C2B30`). It checks slot validity/life/status and the selected
  character's deploy deadline at player
  `+936 + 1512*slot`. `sub_4DF380` (`0x004DF380`) maps an active deadline to
  the local “Hero switching is on cooldown” event.

The deploy deadline is outside `sub_4D5EF0`, but it can suppress only that
portrait/slot. It cannot suppress movement and active abilities, so it is a
parallel gate to record rather than the common whole-lock cause.

### Ability cooldown and admission

`sub_4DF1F0` (`0x004DF1F0`) is the shared targeted-action admission function.
It checks:

- controlled player/object existence and simulation time;
- out-of-range pursuit;
- selected squad-slot validity;
- live object-owned action/release time through
  `sub_4DEFE0`/`sub_4DF0A0`;
- blocking attributes/modifiers and power/range predicates;
- per-definition cooldown through `sub_4D67E0` (`0x004D67E0`);
- active/accepted singleton action timing through
  `sub_4DE910` (`0x004DE910`) and `sub_4DE940` (`0x004DE940`);
- cursor viewport exclusion for the applicable path.

`sub_4D6320` (`0x004D6320`) applies `0xc1` cooldown state to the singleton's
map at `+84` and global deadline at `+576/+580/+584`; `sub_4D5EF0` clears
both. A bad per-ability cooldown can block that ability until reset, but it
cannot account for missing movement or portrait selection. A live
object-owned action entry can affect movement and abilities and survives the
reset, making it the more important release-state diagnostic.

## Smallest safe Fang diagnostic

Do not add another write first. Add one game-thread snapshot at four boundaries:

1. click/keyboard/portrait callback entry;
2. return from `sub_4DF140` or `sub_4DF1F0`;
3. entry and exit of `sub_5370F0` (`0x005370F0`);
4. immediately before and after `sub_4D5EF0`.

One record should contain:

```text
reason, clock, controlledObject, physicalLeftShift, physicalRightShift,
logicalAction16,
input08_0a, mouseMode(11BEF0C), retainedTarget(11BEF10),
fallbackActive(11BEF14), selectedIndex(11BEF24),
movePress(11BEF28), moveHold(11BEF3C),
basicPress(11BEF44), basicRepeat(11BEF4C),
movementActive(1438A0C.low), movementSent(1438A0C.byte1),
genericActive/sent/token/type/definition/target
  (1438A4C/A4D/A4E/A4F/A50/A6C),
auxActive/sent/token/type (1438AC0/AC1/AC2/AC4),
nextToken(1119510), lastUserData(1438B1C),
singleton active/response/queued fields already traced,
runningActionCount, longestRunningActionRemaining,
admissionResult,
deploySlot/deployDeadline/localNow,
uiGate(44A940), viewportGate(4E4C70), modalOwner/callbackID
```

For every incoming `0xa8`, also record receive order, sync, type, definition,
user data, and which of the four token-bearing slots matched **before** calling
`sub_4D64D0`. Keep a reset epoch in Fang-owned memory and tag records with it;
do not write the epoch into client memory.

This is sufficient to distinguish:

- input never reached the resolver (modal/death/UI ownership);
- resolver chose the wrong intent (logical Shift/selection/held state);
- intent staged but was cleared before send (descriptor/token race);
- sender ran (problem below input admission);
- movement/ability deferred on object-owned release state;
- only deploy selection failed on its own deadline/status.

## Smallest safe recovery

Recovery must be diagnostic-driven and run on the game window thread.

1. Snapshot the state above and increment the Fang-owned reset epoch.
2. Call native `sub_4D5EF0` on the validated `sub_4E4E00()` pointer.
3. Retire only proved upstream intent:
   - clear the active low byte of `dword_1438A0C`;
   - clear `byte_1438A4C` and `byte_1438AC0`;
   - set `dword_11BEF24=-1`;
   - clear only `dword_11BEF0C`, `dword_11BEF10`,
     `dword_11BEF14`, `byte_11BEF28`, `byte_11BEF3C`,
     `byte_11BEF44`, and `byte_11BEF4C`.
4. Keep a bounded Fang-owned tombstone set for the pre-reset generic,
   auxiliary, active, and queued sync bytes. Before dispatching a later
   type-`2` `0xa8`, ignore it for descriptor clearing only when the record is
   demonstrably from the prior reset epoch; log the suppression. Expire the
   tombstone on connection/game replacement or after the transport's bounded
   reliable-delivery window. Do not advance or rewrite `byte_1119510`.
5. Re-snapshot. If the controlled object still has a live running-action entry,
   a modal/death selector still owns input, or a deploy deadline is the only
   failure, stop. Use the owning native teardown/selection path once recovered;
   do not clear object lists, UI stacks, character records, cooldown deadlines,
   or unknown memory.

Step 3 is the smallest memory mutation justified by exact writer/reader
recovery. Clearing only the three descriptor bytes, as in the current
experiment, is incomplete. Calling `sub_451A70`, zeroing its whole `0x4c`
block, erasing object `+692`, clearing all player/UI state, or fabricating
movement/ability packets is not safe.

The tombstone rule needs one implementation refinement before use: Fang must
associate a response with the pre-reset epoch from observed send/receive
ordering, not merely by its one-byte sync. A sync-only suppression would
recreate the same ABA bug it is meant to prevent. Until that association is
captured, log delayed matches but do not suppress them.

## Implementation decision

The 2026-07-23 follow-up identified a protocol-level cause in the server.
Build 103's `0xa8` response type `8` moves the active action descriptor into
the client's queued-pursuit slot and invokes native object/point locomotion.
The server had instead sent type `2`, which clears/rejects that descriptor,
then fabricated a separate `0x91` move, repeated `0x95` positions, and a
timer-driven retry. That produced the observed cancel, fresh-basic,
busy-rejection, and presentation churn.

Campaign melee pursuit now sends one matching type-`8` response. The server
does not synthesize player transforms or replay the captured command. Build
103 owns locomotion and submits the follow-up action; its reported position is
then used for range admission and the normal type-`1` acceptance. This removes
the known protocol cause while retaining the narrow Fang reset only as a
diagnostic recovery path.

Fang now implements the narrow step-3 recovery on the game window thread.
After calling the native whole-action reset, it clears the three proven
descriptor active bytes, restores the selected action-bar index to `-1`, and
clears only the proven mouse, target, fallback-point, movement, and basic-attack
intent fields listed above. The before/after trace now records these fields and
reports whether the upstream intent reset ran.

The hook does not change the action token, timers, cooldowns, object-owned
running-action list, modal state, deploy state, or any other client memory.
Those boundaries remain diagnostic-only until a locked capture proves their
ownership. In particular:

- if only the known descriptor/input flags remain active and no object/modal
  owner remains, the implemented reset should recover input;
- if a late type-`2` response clears a new descriptor, implement an
  epoch-associated response tombstone;
- if `sub_4DF140`/`sub_4DF1F0` reports a live object action after reset,
  recover and call its native cancellation owner;
- if the input callback is never entered, recover the modal/death-selection
  owner instead of touching combat state.

The 2026-07-23 playtest added a narrower automatic recovery case. Its final
client snapshot retained `basicRepeat=1` after the physical left button had
been released, while movement and later chat commands stopped reaching the
server. Fang now watches only that proven repeat byte. If it remains set for
750 ms with the physical button released, the game-window thread clears the
basic press/repeat edge and their retained mouse mode/target. It does not react
to the transient basic-press byte alone, so an ordinary click still receives
the client's normal frame to resolve and send.

## Remaining capture gaps

1. Reproduce the pursuit/ranged lock with all four diagnostic boundaries and
   record the first human input edge that fails, not only the last packet.
2. Capture `/reset` in the same locked process. The current trace has no
   `chat_reset_*` record and therefore cannot compare every pre/post field.
3. Count descriptor allocations from the last accepted action through the
   lock and determine whether a 255-token wrap actually occurs.
4. Retain send-side action metadata long enough to associate delayed `0xa8`
   responses with a Fang reset epoch without relying on sync alone.
5. Identify the controlled object's `+692` entry type, its native
   cancellation/destructor owner, and the exact stuck entry after Shockwave,
   Zetawatt Beam, and repeated basic pursuit.
6. Hook the death/survivor portrait callback and modal-stack owner to locate
   why the 22:27 Goliath selection never reaches deploy admission.
7. Record `sub_9C2B30` status and deploy deadlines during a whole lock to
   separate genuine portrait cooldown from upstream input ownership.
8. Run successful comparison cases for plain movement, held basic release,
   pursuit cancel/retry, ranged completion, active-to-squad swap, death
   selection, and same-process campaign restart.
