# Campaign cash-out final transition

## Result

Build 103 does not use a second RakNet cash-out acknowledgement after
`AB 00`. The two similarly named actions are distinct:

1. Planet Screen **Collect Reward** sends C2S `AC 02`, causing entry into
   `cChainCashOutState`.
2. That state entry sends the one proven C2S `AC 04` request. S2C `AB 00`
   plus its 712-byte record populates and runs the MaxisCashOut presentation.
3. Only after that presentation exposes its Return to Spaceship button does
   the button's click handler play the SWF's `outro` label and call
   `flash.external.ExternalInterface.call(
   "CashOut.OnBackToSpaceshipClicked", false)`.
4. The native binding selects client state `2`, `cSpaceshipState`, locally.

The close action emits no RakNet application message, Blaze RPC, HTTP request,
or second `AC 04`. The server must leave the client in Cash Out after sending
`AB 00`; sending `AF 00` or another state-2 selector at that point would tear
down the cash-out state before the player has finished viewing the result.
Normal ship initialization may perform its ordinary later work, but it is not
a cash-out close acknowledgement and is not an input to this transition.

This resolves the final-transition mechanism. Reward selection, persistence,
and account-refresh authority described in `notes/campaign/1-1/rewards.md`
remain separate questions; they do not justify inventing a close packet.

## Exact ordered path

```text
PlanetScreen Collect Reward
  -> C2S AC 02
  -> client enters state 12 (cChainCashOutState)
  -> state-entry C2S AC 04                         [one request per entry]
  -> S2C AB 00 + 712-byte presentation record
  -> MaxisCashOut remains visible and runs XP/medal/loot presentation
  -> Return to Spaceship button becomes available
  -> SWF gotoAndPlay("outro")
  -> ExternalInterface.call(
       "CashOut.OnBackToSpaceshipClicked", false)
  -> native sub_4062A0
  -> local transition 12 -> 2 (cSpaceshipState)
```

There is no arrow from the close callback back to `AC 04`. `AC 04` is the
cash-out state's data request, not a grant acknowledgement and not a close
request.

## Packaged SWF evidence

The relevant packaged movie is the decoded `FlashUI.package` resource
`000516_278cf8f2_00000000_00000000263201a0.bin`. Its embedded asset names use
the `Chain4` prefix, and the native loader independently requests
`Chain4.swf` / `MaxisCashOut` at `sub_409CA0` (`0x00409CA0`,
`Game.c:147835-147869`). The resource is therefore the movie instantiated
by the native cash-out UI, not a merely similar screen. Its extracted CFX is
58,737 bytes with SHA-256
`509927feaac4af684ac0226ddcbc553aeaadc5df316733ef91014f34a2bf5708`;
its zlib-decoded SWF body is 188,285 bytes.

Its root AVM1 action block proves all of the relevant callback wiring:

| SWF action | Recovered behavior |
| --- | --- |
| `setupButtons` | Adds the `click` listener `OnBackToSpaceshipClicked` to `btnReturn2Spaceship`. It separately binds the stats and editor buttons. |
| `OnBackToSpaceshipClicked` | Traces `outro clicked!`, executes `gotoAndPlay("outro")`, then calls `CashOut.OnBackToSpaceshipClicked` through `flash.external.ExternalInterface` with boolean `false`. |
| `OnEditorButtonClicked` | Also plays `outro`, but calls the same native binding with boolean `true`. This explains the native binding's conditional side effect and proves that the ordinary Return to Spaceship button is the false branch. |
| animation callbacks such as `xpOutroDone` and creature `outroDone` | Advance the movie's presentation timeline locally. None calls the native back-to-ship binding or sends `AC 04`. |

The Return to Spaceship handler is a click callback. It is not a timer, and it
is not automatically invoked by receipt of `AB 00`. The `outro` animation is
started before the ExternalInterface call in the same handler; it is not a
server-controlled delay or acknowledgement.

## Native binding and state transition

The canonical build-103 binding table at `0x0110F23C` contains five
function/name pairs. Its first pair is exact:

```text
0x004062A0  ->  "CashOut.OnBackToSpaceshipClicked"
```

The remaining pairs are the read-only loot helpers
`CashOut.GetLootImageForPlayer`, `CashOut.BuildLootTooltip`,
`CashOut.GetItemName`, and `CashOut.GetItemNameColor`. This table ties the SWF
string to `sub_4062A0`; it is not a name-based guess.

`sub_4062A0` (`0x004062A0`, `Game.c:145797-145815`) does the following:

1. If its boolean argument is true, it publishes local message `192750538`.
   The ordinary Return to Spaceship button supplies false, so this branch is
   skipped. The true branch belongs to the editor button.
2. It obtains the client transition manager through `sub_4E4DA0`.
3. It calls `sub_455A10(manager, 2)` and stops the results UI timing marker.

`sub_455A10` (`0x00455A10`, `Game.c:205243-205257`) invokes the transition
manager's state-change operation and records the accepted target state.
`sub_455A70` explicitly permits current state `12` to target state `2`, while
`sub_455640` constructs state `2` as `SP_SporeLabs/cSpaceshipState`
(`Game.c:205044-205063`). This is an address-backed local-state
transition.

The callback contains no construction or send through the game-message
transport, no Blaze component lookup/RPC, and no web request. Cash-out state
exit `sub_449350` unregisters its logical-message receiver, destroys the
MaxisCashOut UI, restores its audio mix, and continues normal scene teardown.
The logical-44 operation there is receiver removal for `ChainCashOut (AB)`,
not a packet send.

## `AC 04`, `AB 00`, and state lifetime

The native state path keeps the data request and close transition separate:

- `sub_449850` is cash-out state entry. It registers the logical-44 / wire-AB
  receiver and sends logical ChainPlayer subtype `4`, producing C2S `AC 04`.
- `sub_4499D0` accepts only AB subtype `0`.
- `sub_449410` copies exactly 712 bytes, passes the record to the cash-out UI,
  and selects/retains state `12` (`Game.c:195673-195683`). It does not
  select state `2`.
- The later SWF click reaches `sub_4062A0` and selects state `2` locally.

Thus the exact one-entry sequence has one `AC 04`. A reconnect or a genuine
new state entry could create another entry request, but no evidence turns the
same entry's button click into an `AC 04` replay. Retry and reconnect policy
remain server authority; the close contract does not.

`AF 00` is a server-authored `QuickGame` selector that can send an active
client directly to state `2`. Its existence elsewhere does not place it after
cash-out data. On this path it would race the SWF-owned presentation and is
neither called by the close callback nor required to return to the ship.

## Walkthrough cross-check

The 1-1 recording visibly separates Collect from the later cash-out
presentation:

- `frame-900.png` at about `15:00` shows the Planet Screen's **Collect Reward**
  choice.
- `frame-915.png` at about `15:15` still shows the MaxisCashOut reward card,
  roll 39, and Leto's Pale Guard over the ship scene.

The 1-2 recording provides the same cross-check: its late frames retain the
reward screen and roll 69 rather than disappearing when the cash-out data is
displayed. Neither edited recording retains a visible final Return to
Spaceship click or a packet trace, so the videos do not prove the callback by
themselves. They do prove the important presentation constraint: receipt of
the cash-out result must not immediately replace the result UI with ordinary
ship control. The SWF/native binding supplies the missing exact close action.

## Server contract

For a completed and durably committed cash-out snapshot, the exact client
transition contract is:

1. Accept `AC 02` only in the voting phase and enter Cash Out once.
2. Treat the resulting state-entry `AC 04` only as the request for the fixed
   snapshot.
3. Reply `AB 00` plus the same 712-byte committed presentation record on an
   allowed retry.
4. Do not follow that reply with `AF 00`, a fabricated acknowledgement, a
   timer-based state change, or a second interpretation of `AC 04`.
5. Let `CashOut.OnBackToSpaceshipClicked(false)` perform the client-local
   state-2 transition when the player closes the screen.

No additional retail server-authority decision remains for the close
mechanism, so this pass does not add or alter an entry in `notes/help.md`.
Persistence atomicity and retry policy remain governed by the existing
campaign result/reward decisions, independently of this recovered UI
transition.

## Reproducibility

The Flash package was inventoried and decoded under
`bin/game/logs/cashout-transition/flashui`. The small AVM1 diagnostic
`bin/game/logs/cashout-transition/disasm_avm1.py` walks nested SWF action tags
and exposes the exact root callback actions used above. These are generated
reverse-engineering artifacts under the workspace's required diagnostics
directory, not implementation changes.

Primary evidence:

- `bin/game/GameBin/Game.c`: `sub_4062A0`, `sub_409CA0`,
  `sub_449350`, `sub_449410`, `sub_449850`, `sub_4499D0`, `sub_455A10`,
  `sub_455A70`, and `sub_455640`.
- `bin/game/GameBin/Game.idb`: canonical address and binding-table
  source corroborated by the executable's function/name pairs.
- `notes/campaign/1-1/results.md` and `notes/campaign/1-1/rewards.md`: prior
  packet, 712-byte record, reward, and durability boundaries.
- `bin/video/walkthrough/1-1/frame-900.png`, `frame-915.png`, and the 1-1
  video; late 1-2 frames/video provide the repeated visual cross-check.
