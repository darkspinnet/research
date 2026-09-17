# 1-1 director budget and callback

## Result

Build 103 does not contain a client call edge from either 1-1 horde callback
to the kind-`5` selector. `HordeTrigger_OnEnterPlayer` is bound to an inert
client handler which always returns false. The generic trigger dispatcher calls
that handler before publishing the authored event, and publishes only when the
handler returns true. Consequently the shipped client does not publish either
1-1 enter-side `horde triggered` event, does not set the enter-success latch,
and does not reach the authored `HordeSpawner_Register` listeners through this
path.

Kind `5` is nevertheless recoverable as a generic list builder. It selects
from the difficulty-eligible `agent` director class under a caller-supplied
floating-point budget. Its result count is `0..15`; fifteen is a hard cap, not
an authored wave size. No build-103 client caller passes kind `5`, and neither
the 1-1 level nor the two horde marker sets supplies the missing runtime budget.

## Kind 5 and budget source

| Finding | Evidence | Confidence |
| --- | --- | --- |
| `SpawnPoint_DirectorHorde.Noun` maps to kind `5`. | `sub_9FA9B0` (`0x009FA9B0`) maps noun-class hash `0x45e6b07b` to `5`. `sub_9FB420` (`0x009FB420`) calls the classifier and stores the result in the registered marker record at record dword `+3`. | High |
| Kind `5` draws from the `agent` class. | The case-`5` branch of `sub_9FE270` (`0x009FE270`) scans the 16-byte records between director dword indices `210` and `211`. Native `LevelConfig` reflection identifies the fourth class as `agent` at `0x00f7185e`/`0x00f718a9`; the director layout places that class at indices `210`/`211`. | High |
| Eligibility is an inclusive difficulty interval. | In case `5`, a record is copied only when `currentDifficulty >= record[+4]` and `currentDifficulty <= record[+8]`. The record's fourth dword is not tested on this path. | High |
| The numeric limit is supplied by the caller, not read from the marker or level entry. | `sub_9FE270` receives the budget through its third parameter; `sub_9FBA70` (`0x009FBA70`) compares against that pointed-to float. On return, `sub_9FE270` overwrites the same slot with the final adjusted cost. No case-`5` instruction reads marker-set weight or another 1-1 marker field as a budget. | High |
| The per-accepted-noun escalation comes from global tuning lookup, not the caller's budget. | Global initializer `sub_9CF190` (`0x009CF190`) loads float storage `dword_14CB084` with `sub_50D9C0(..., 751802685 /* 0x2ccf993d */, 0.1)`; `sub_9FBA70` uses that float. Thus the compiled fallback is `0.1`; the symbolic property name is not recovered. | High for value and source; low for an unrecovered property name |

Recovered semantic signature, preserving the decompiler's unresolved types:

```text
buildSpawnList(director, kind, inoutBudgetOrCost, outNouns[15], difficulty)
    -> acceptedCount
```

For kind `5`, the input is a budget. For accepted zero-based index `i`, with
`rawCost` equal to the sum of previously accepted noun costs and `nounCost`
read from resolved noun data at `+0x24`, `sub_9FBA70` computes:

```text
prospectiveCost = (1 + i * tuning_0x2ccf993d) * (rawCost + nounCost)
```

The candidate is accepted when `prospectiveCost <= inputBudget`. Acceptance
appends its noun pointer, adds `nounCost` to `rawCost`, records
`prospectiveCost`, and increments the count. The noun-data `+0x24` integer is
therefore the value used as this selector's budget cost; no stronger authored
field name is present in the examined code. Confidence: **high** for the
calculation and use, **medium** for calling the field a director cost.

## Selection and count rules

`sub_9FBA70` (`0x009FBA70`) implements one attempted selection:

1. An empty candidate vector returns false.
2. `sub_9BCE90` chooses a bounded random candidate index.
3. The helper resolves the noun and reads the cost at noun-data `+0x24`.
4. A candidate whose prospective cost is strictly greater than the budget is
   removed from the temporary vector and the helper returns false.
5. An accepted candidate remains in the vector, so it may be selected again.

The kind-`5` loop in `sub_9FE270` stops on the first false result, when the
adjusted cost has reached/exceeded the budget check at the loop head, or when
the accepted count reaches `15`. It does not retry a cheaper candidate after a
random over-budget choice. Therefore:

- the result count is from zero through fifteen;
- equality with the budget is accepted, but the next loop-head test terminates;
- eligible-candidate count is not spawn count;
- accepted noun classes can repeat;
- the hard cap is not evidence for a fifteen-enemy wave; and
- without the caller's budget and noun costs, the exact count is unrecovered.

Confidence: **high**. These rules are direct control/data flow in
`sub_9FBA70` and the case-`5` loop of `sub_9FE270`.

### Call graph boundary

The complete IDA xref audit finds one direct code reference to `sub_9FE270` and
no stored raw function pointer:

| Caller | Call site | Arguments relevant here | Meaning | Confidence |
| --- | --- | --- | --- | --- |
| `sub_9FE7D0` (`0x009FE7D0`) | `0x009fea1b` | kind `1`, literal float budget `10`, current difficulty | Conditional generic ordinary-spawn substitution; not a horde-listener call. | High |

Thus no shipped client caller passes kind `5`. `sub_9FBA70` itself is shared by
the selection routines beginning at `sub_9FD660` (`0x009FD660`),
`sub_9FDD50` (`0x009FDD50`), and `sub_9FE270` (`0x009FE270`). Those additional
helper call sites do not create an indirect caller of `sub_9FE270` and do not
supply a 1-1 horde budget. Confidence: **high**.

## 1-1 event topology

The content relationship is exact, but it proves consumers rather than a
successful client publication:

| Contact trigger | Authored trigger fields | `horde triggered` listeners |
| --- | --- | --- |
| marker row `128245`, authored ID `2244983668`, `SpawnPoint_HordeTrigger.Noun-1` | level-event row `32474` preserves native callback `HordeTrigger_OnEnterPlayer`; the same raw marker record contains enter event `horde triggered` and exit event `horde complete` | rows `128241`, `128244`, and `128247`, whose callback is `HordeSpawner_Register` |
| marker row `128253`, authored ID `2142467792`, `SpawnPoint_HordeTrigger.Noun-2` | level-event row `32478` preserves native callback `HordeTrigger_OnEnterPlayer`; the same raw marker record contains enter event `horde triggered` and exit event `horde complete` | rows `128251` and `128252`, whose callback is `HordeSpawner_Register` |

The `TriggerVolumeEvents` reflection layout distinguishes `onEnterEvent` at
definition offset `+0x0c` from `onExitEvent` at `+0x1c`; the raw records place
the two strings in those slots. The first marker set contains three horde
listener loci and the second contains two. That is a possible fanout
cardinality after a successful enter publication, not an enemy count or proof
of runtime listener order. Confidence: **high** for the authored links;
**none** for deriving a wave size from them.

## Dispatch order

The client trigger path has a strict, recoverable order:

1. Trigger entry reaches `sub_A15FB0` (`0x00A15FB0`).
2. When both a native enter callback and an event are present, the dispatcher
   calls the native callback at `0x00a1604c`.
3. It tests the callback result at `0x00a16051`-`0x00a16053`.
4. Only a true result reaches event publication through `sub_9BF5A0` at
   `0x00a1605b`; success then sets runtime latch byte `+0x3e`.
5. `sub_9BF5A0` (`0x009BF5A0`) walks the registered listener vector in its
   stored order and calls `sub_A1D0F0` (`0x00A1D0F0`) for each entry.
6. Within one listener, `sub_A1D0F0` calls its resolved native callback first
   and its Lua callback second. Lua receives
   `(sourceObjectID, listenerObjectID, otherObjectID)`.

The executable proves sequential iteration of the runtime listener vector, but
does not prove how the five 1-1 listener objects are ordered in that vector.
Marker ordinal, database row order, and marker-set weight must not be presented
as dispatch order. Confidence: **high** for native-before-event and
native-before-Lua; **unresolved** for ordering among same-event listener
objects.

## Callback and activation lifecycle

Static startup binds the literal `HordeTrigger_OnEnterPlayer` to
`sub_9FACB0` at `0x00f72ea0`-`0x00f72eaf` through `sub_A173A0`.
`sub_9FACB0` (`0x009FACB0`) has no successful branch: depending on role and
entrant validation it may query the simulator/director, but every path returns
false. It does not mutate director state, call `sub_9FE270`, publish an event,
or send a gameplay message. Because `sub_A15FB0` sees false, the authored event
is not published and the trigger's enter-success latch remains clear; a later
entry may invoke the callback again. Confidence: **high**.

The exit side is separate. Neither raw trigger record contains a native exit
callback, but both author `horde complete` as `onExitEvent`. In the no-callback
branch, `sub_A15FB0` publishes a nonzero event directly at `0x00a1603a` and
sets the success latch. Thus a client-side exit can publish `horde complete`
even though the inert enter callback suppressed `horde triggered`. This proves
generic trigger behavior, not that a retail horde was completed; no
`HordeSpawner_Register` listener consumes `horde complete` in these two sets.
Confidence: **high** for dispatch behavior and authored fields, **low** for any
gameplay meaning beyond an exit notification.

The horde-spawner objects have a separate object-lifetime listener lifecycle.
On object activation, `sub_A1D190` (`0x00A1D190`) allocates the listener, fixes
its context to that object, and calls `sub_9C2030` to register its 40-byte
`EventListenerData` entries. On teardown, `sub_A1D290` (`0x00A1D290`) calls
`sub_9BF490` to remove those registrations. This proves when a listener is
available for events; it does not prove that `HordeSpawner_Register` activates
a wave. Confidence: **high**.

`HordeSpawner_Register` is absent from the complete build-103 executable string
inventory and from the indexed packaged Lua chunks, although it is present in
the authored 1-1 marker records. Accordingly there is no shipped client body to
recover for that callback. The strongest evidence-backed conclusion is that
the client lacks the authoritative registration/activation implementation;
assigning budget selection, spawning, wave bookkeeping, timing, or clear-state
transitions to this name would be inference. Confidence: **high** for absence,
**medium-high** that the missing implementation was server-owned.

Finally, `sub_9FB420` registering a kind-`5` marker and an event-listener object
being live are availability/setup states, not proof of encounter activation.
In this client the contact callback never advances beyond that setup. The
retail server's budget source, same-event listener order, spawn-object creation
order, active-wave bookkeeping, completion predicate, and teardown after a
cleared horde remain unrecovered.
