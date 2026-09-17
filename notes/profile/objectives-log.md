# Build-103 Objectives Log contract

## Conclusion

The supplied screen is the **in-planet Objectives Log**, not an account-profile
view populated by a Blaze response. Build 103 opens it from the gameplay UI,
reads a five-record client objective array, resolves each record's objective ID
to a registered Lua objective definition, expands that definition's localized
title/progress templates with three per-player integers, and calls the SWF with
the row index, medal, title, and progress text.

For the captured screen, the population contract is RakNet/GMS
`ObjectivesInit` (`0xb7`), immediately followed by `ObjectiveUpdated` (`0xb8`).
There is no objective-specific Blaze component/command, TDF response, HTTP
account field, SQLite account column, or persistent objective record in this
repository. Consequently there are **no Objectives Log TDF tags or TDF types**
to implement. The native Blaze Stats component is a separate false lead.

This conclusion is proven by an existing runtime trace whose five IDs and
values reproduce every visible row and every anomalous zero in the screenshot.
The content and native evidence below also explains why the first row alone has
a gold medal and `00:01 on planet`.

## Evidence labels

- **Proven native** means directly present in
  `bin/darkspinner/GameBin/Game.c` or an IDA diagnostic made from the
  repository's build-103 executable.
- **Proven content** means directly decoded from the repository's
  `content.db`, DBPF resources, localization, or compiled Lua 5.1 bytecode.
- **Proven runtime** means present in an existing Darkspinner/Fang trace.
- **Inference** is called out explicitly and is not required to explain the
  observed screen unless stated otherwise.

No material under `C:\src\recap` was inspected. The similarly named reference
tree under `bin/game/logs` was also excluded from the searches used here.

## Exact gameplay wire contract and timing

### Message IDs and native receivers

| GMS message | Wire ID | Native dispatcher case | Native receiver | Meaning |
| --- | ---: | ---: | --- | --- |
| `kGMSObjectivesInit` | `0xb7` | 56 | `sub_537AA0` (`0x00537AA0`) | Replace the client objective array |
| `kGMSObjectivesComplete` | `0xb9` | 57 | `sub_536EE0` (`0x00536EE0`) | Completion snapshot/results |
| `kGMSObjectiveUpdated` | `0xb8` | 58 | `sub_536DA0` (`0x00536DA0`) | Update one objective/player and optionally show its notification |
| `kGMSObjectiveAdd` | `0xca` | 75 | `sub_537BF0` (`0x00537BF0`) | Append one objective record |

The non-numeric ordering of dispatcher cases 57/58 is native evidence; the
wire IDs come from the retail message table and the repository's packet table.

The payloads, excluding the one-byte GMS message ID, are:

```text
0xb7 ObjectivesInit
  uint8 count
  repeat count:
    uint32le objective_id
    uint8 medal_or_state[4]
    uint32le token[4][3]

0xb8 ObjectiveUpdated
  uint32le objective_id
  uint8 player_index       // 0xff applies the update to all four players
  uint8 medal_or_state
  uint32le voiceover_guid
  uint8 is_notification_shown
  uint32le token[3]

0xca ObjectiveAdd
  one 56-byte objective record in the same form used by 0xb7
```

Thus the five-objective initializer is 281 payload bytes (282 including
`0xb7`), and an update is 23 payload bytes (24 including `0xb8`). Native
`sub_537AA0` reads the four state bytes separately, reads the following 48 bytes
verbatim, and expands the record to 68 bytes in memory:

```text
uint32 objective_id
uint32 medal_or_state[4]   // each wire byte expanded to a DWORD
uint32 token[4][3]
```

`sub_536DA0` finds a record by ID, replaces the selected player's DWORD state
and all three tokens, and invokes the notification/voiceover path only when the
wire boolean is true. Therefore that boolean is **not** Hidden Bonus category
visibility.

### The exact captured sequence

`bin/darkspinner/darkspin/logs/traces/game.jsonl` records, at
`time_ms=178595593`:

1. `0xaf` QuickGame message;
2. `0xb7`, payload size 281, prefix
   `05ee3397ff0100000000000000000000`;
3. `0xb8`, payload size 23, prefix
   `ee3397ff000400000000000100000002`;
4. object creation/update messages beginning with `0x8c`.

The `0xb7` and `0xb8` arrive in the same millisecond, before ordinary scene
objects. This is gameplay-session setup, not a request made when the player
presses `O`. Opening the log later only reads the already-populated local array.

The repository encoder supplies these five `0xb7` records, in this order:

| UI slot/category | Objective ID | Registered objective | Initial player-0 state | Initial tokens |
| --- | ---: | --- | ---: | --- |
| 0 / Daily | `0xff9733ee` | `FinishLevelQuickly` | 1, then 4 by `0xb8` | `(0,0,0)`, then `(1,2,3)` by `0xb8` |
| 1 / Standard | `0xac4273f3` | `DoDamageOften` | 1 | `(0,0,0)` |
| 2 / Standard | `0x61c07561` | `TouchAllObelisks` | 1 | `(0,0,0)` |
| 3 / Standard | `0xa28485cc` | `DefeatAllMonsters` | 1 | `(0,0,0)` |
| 4 / Hidden Bonus | `0x0478facb` | `HugeDamage` | 1 | `(0,0,0)` |

The following `0xb8` is exactly:

```text
objective_id           0xff9733ee
player_index           0
medal_or_state         4
voiceover_guid         0
is_notification_shown  false
token                  (1, 2, 3)
```

The screenshot proves that state 4 renders as gold and state 1 as the grey
in-progress medal. The complete conventional mapping `1=failed/in progress`,
`2=bronze`, `3=silver`, `4=gold` is strongly supported by the ordered Lua
statuses, but only the observed 1 and 4 rendering is proven here.

The current Go names the first token `Value` and hard-codes the other two to 2
and 3. Native code proves that all three are peer objective-token DWORDs, not a
single value plus protocol constants. The older `raknet.Objective.Encode` in
`server/raknet/gameplay.go` is also not this contract: its 48-byte description
string is incompatible with the native `0xb7` parser.

## Native UI and client structures

**Proven native:** `MaxisObjectivePopup` is identified at `0x0041D2E0`.
`sub_41D300` (`0x0041D300`) obtains the objective vector at global game-state
offset `+672`, loops exactly five times, and for each row:

1. looks up the objective definition by the record's `objective_id`;
2. selects `medal_or_state[current_player]`;
3. detokenizes definition fields at offsets `+100` and `+104` using the 68-byte
   record;
4. invokes SWF methods `SetMedal(index, state)` and
   `SetObjectiveText(index, title, progress)`.

If a row has no registered record, the client substitutes localization
`0x099532aa` (`Locked Objective`) and `0x099532ab` (`Complete other objectives
to unlock.`). The five slot indices are passed unchanged to the SWF. Combined
with the screenshot and the dedicated UI localization, this proves the slot
presentation: index 0 is Daily, 1-3 are Standard, and 4 is Hidden Bonus. The
category is not carried as a TDF field or account field.

**Inference:** the final index-to-heading layout is implemented inside
`HUD_ObjectivesPopup.swf`; the SWF is not available as a loose authored source.
The native call boundary nevertheless proves that no per-row category value is
passed to it.

There is a second, simulator-side structure. The native object pool allocates
`cObjectiveInstanceData` as 60-byte records aligned to 16 bytes and
`cObjectiveEvent` as 52-byte records aligned to 8 bytes. The Lua binding
`SetObjectiveIntData(player, index, integer, bool, bool)` accepts player `0..3`
or `0xff`, restricts `index` to `0..2`, and stores three 32-bit integers for each
of four players. An all-player write copies the integer into all four slots.
This is the source-side counterpart of the 48 token bytes, not persistent
account state.

The IDA name/xref logs under `bin/game/logs/objectives-log` also identify
`LevelObjectives`, `levelObjectives`, `FillObjectiveData`,
`cObjectiveInstanceData`, `kGMSObjectivesInit`, `kGMSObjectiveUpdated`,
`kGMSObjectivesComplete`, and `kGMSObjectiveAdd`. Broad immediate-value scans
for `0xb7`, `0xb8`, and `0xca` contain many unrelated instruction operands and
must not be treated as message evidence.

## Content that supplies the five rows

### Packages and tuning pools

The authoritative packages represented by `content.db` are:

| Package | SHA-256 |
| --- | --- |
| `Data/AssetData_Binary.package` | `faf7b72a27b7f6b6f2c497c9aa5379bd2f20b1e8fe7b59b0d3798a2232774f2b` |
| `Data/Locale/en-us/Text.package` | `45c7e220a73320a5b215cebd89d5c785fe6c227115d6f3520cbfd959f6258739` |

The objective-pool DBPF resource type is `0x68232164`. Two relevant resources
are:

| Instance | Decoded SHA-256 | Proven contents relevant here |
| ---: | --- | --- |
| `0x2ea8fb98` | `c1b3e14a9f43efa6585a535af3cc3683eef16583ff90daccfd39bf797d95c79b` | `DefeatAllMonsters`=`0xa28485cc`, `TouchAllObelisks`=`0x61c07561`, plus four other objectives and nine affix choices |
| `0x71aec1dc` | `0e6841e7f70888dcbe27cc2ed815b48ab60b2def7586839c8fa409507d756965` | `HugeDamage`=`0x0478facb`, `DoDamageOften`=`0xac4273f3`, `DestroyAllDestructables`, `LootCrystals` |

A third resource of this type, `0x99e410f1`, contains `Juggernaut` and is not
part of this screen. `FinishLevelQuickly` is a registered script but is not in
either decoded random-objective pool above; its special selection into Daily
slot 0 is proven by the wire/capture, while the upstream selection rule is not
implemented or recoverable from a named DBPF path. DBPF retains type/group/
instance identities, not original authored filesystem names.

### Localized UI and objective text

UI table `0xe3b800e9` supplies `Objectives Log` (`0x09794faa`), `Daily`
(`0x0bce2c22`), `Standard` (`0x0bce2c23`), and `Hidden Bonus`
(`0x0bce2c24`). Objective strings are in table `0x4352f551`:

| Objective | Title key/text | Progress key/template |
| --- | --- | --- |
| Finish quickly | `0x097562f0` — `Finish Game Threat Quickly` | `0x09d16768` — `~time1~ on planet` |
| Damage often | `0x09e6d276` — `Deal Damage To Game ~int2~ Times` | `0x09e6d277` — `Damage dealt to Game ~int1~ times` |
| Obelisks | `0x097562ee` — `Access All Crogenitor Archive Obelisks` | `0x098cd6b8` — `~int1~ of ~int2~ Crogenitor Archive Obelisks Accessed` |
| Defeat all | `0x097562ef` — `Defeat All Game On The Planet` | `0x098cd6b9` — `Approximately ~int1~% Game defeated` |
| Huge damage | `0x09e6d259` — `Deal Over ~int2~ Damage ~int3~ Times` | `0x09e6d260` — `Dealt more than ~int2~ damage ~int1~ times` |

The screenshot's expanded zero strings do not exist as authored text. They are
the templates above detokenized against zero-filled `0xb7` token arrays.

### Lua definitions, thresholds, and token ownership

The exact compiled Lua resources extracted with `darkrun db` are retained with
their disassemblies under `bin/game/logs/objectives-log`:

| Objective / source | Decoded SHA-256 | Authored defaults and token writes |
| --- | --- | --- |
| `FinishLevelQuickly`, `lua/0x389D8EEE.lua` | `cb9020a40265832803e62418ca1141fff829373d57b2f3ef8b644ee77bdc875c` | `groupObjective=true`; gold/silver/bronze at 600/900/1200 seconds. At status evaluation, token 0 is `floor(GetGameObjectiveCompletionTime())`. |
| `DoDamageOften`, `lua/0x2F37FCF3.lua` | `8441e30aa93c5e61cb09cb2817bfc40a6a4fca856aea24645c673dd3d6721db4` | `timeObjectiveRequires=true`; difficulty scale 7; gold 150..500, silver 100..350, bronze 50..200. Init writes the scaled gold count to token 1; qualifying damage events increment token 0. |
| `TouchAllObelisks`, `lua/0x409C9F2B.lua` | `e168a19cf64bddaff0b483b7d1b2546011d14cc4a72b5528331105c4b2133047` | `groupObjective=true`, `timeObjectiveRequires=true`; gold/silver/bronze 3/2/1. Init counts `InteractWithObelisk` objects and writes the total to tokens 1 and 2. Events write touched count to token 0, total to token 1, and remaining count to token 2. |
| `DefeatAllMonsters`, `lua/0x4D899ECC.lua` | `fda2e11e8da8f72c7415c061ee35b98d62a2d8ebbc71d0ce2ebaaf583a96f70a` | `groupObjective=true`, `timeObjectiveRequires=true`; authored gold/silver/bronze values are 95%/75%/50%. Native truncation makes Init write 94 to token 1. Every death writes the current integer kill percentage to token 0; crossing each retained 25-point boundary instead writes that boundary and advances the private threshold. Full clear writes 100. |
| `HugeDamage`, `lua/0x521AF5CB.lua` | `0031aa46c8b68375d6b8976a0655a5c41bc8e1dbef70d1bb4703e30fa1877bfc` | `timeObjectiveRequires=true`; gold/silver/bronze counts 30/15/5; threshold is difficulty health multiplier times 35. Init writes threshold to token 1 and gold count to token 2; qualifying hits increment token 0. |

The current implementation executes all five packaged initializers in isolated
per-session runtimes. It also executes the retained event callbacks for
`DefeatAllMonsters`, `TouchAllObelisks`, `DoDamageOften`, and `HugeDamage`.
Damage events preserve the recovered slots: NPC target GUID at 1,
player-controlled source GUID at 2, float damage at 3, and integer metadata at
4. The two object teams must differ. Normal authoritative enemy damage and the
accepted `InteractWithObelisk` use feed these runtimes and publish silent `0xb8`
token snapshots. All five `ObjectiveStatus` callbacks also execute in their
retained runtimes. `FinishLevelQuickly` floors authoritative completion time
into token 0 and returns Gold through 600 seconds, Silver through 900, Bronze
through 1200, and Failed thereafter. The other callbacks return their authored
result from authoritative kill percentage or their retained touch/damage
counters. The simulator exposes every authored result without yet assigning
live medal or `0xb9` publication ownership.

This gives exact numerator/denominator ownership:

- token 0 is the live numerator, percentage, count, or elapsed seconds;
- token 1 is the target/threshold/total used by `~int2~` (95 for defeat-all is
  retained although that progress template only displays token 0);
- token 2 is the second title parameter or remaining count used by `~int3~` or
  the obelisk error text.

The Lua flag name `timeObjectiveRequires` is proven; interpreting its broader
selection semantics would be inference. It is not an expiry timestamp.

## Why the screenshot says zero and what “Daily” means here

The current initializer sends all 48 token bytes as zero for every objective.
It does not run or mirror the authored Lua `Init` functions before constructing
`0xb7`. Therefore:

- Damage Often receives `(0,0,0)` and formats both count and target as 0.
- Obelisks receives `(0,0,0)` and formats `0 of 0`.
- Defeat All receives token 0 = 0 and formats approximately 0%.
- Huge Damage receives `(0,0,0)` and formats threshold, progress, and target as
  0.
- Finish Quickly is then updated to `(1,2,3)`; only `~time1~` is used, yielding
  `00:01 on planet`. State 4 yields the gold medal.

There is no expiry/rollover value in the record, and `00:01 on planet` is
elapsed gameplay completion/progress time, not “expires in one second.” The
separate front-end localizations `Daily Bonus Active` and `Next Daily Bonus...`
do not participate in this popup. That cash-out bonus uses the account's
`cashout_bonus_time` and an exact rolling 79,200-second profile countdown, as
documented in `notes/campaign/daily-bonus.md`. “Daily” is the heading assigned
to slot 0 in this five-slot gameplay layout.

## Blaze and persistence: explicit negative result

**Blaze:** build 103 contains standard `StatsComponent` with component ID
`0x0007`; command `0x0002` is `getStats`, and its response class is
`Blaze::Stats::GetStatsResponse` containing
`mKeyScopeStatsValueMap`. The only native caller found is
`cBlazePlaygroupsManager`, which constructs an RPC job with a 10,000 ms timeout.
It is unrelated to `MaxisObjectivePopup`. No call from the popup or gameplay
objective handlers reaches Stats, and the existing server does not register
component `0x0007` at all. It registers `0x01`, `0x04`, `0x05`, `0x06`, `0x09`,
`0x0f`, `0x15`, `0x19`, and `0x7802`.

Accordingly, assigning Stats TDF tags to this screen would be fabrication. The
exact Objectives Log RPC/component/TDF contract is: **none / none / no TDF
fields**. The relevant request/response timing is likewise none; the only
timed population event is the same-millisecond `0xb7`/`0xb8` gameplay sequence
above.

**Persistence:** `sporenet.Account`, the SQLite `user` schema/row adapter, and
the account HTTP representation have no objective IDs, objective token values,
medals, selected categories, timestamps, or expiry fields. `cashout_bonus_time`
belongs to the independent chain Cash Out Daily Bonus and must not be
repurposed. Objective selection and progress are owned by the live
game/simulator session and the client array.
The existing trace does not show a persistence write for either message.

## Minimal implementation and test plan

No Blaze component and no account migration are needed for the log shown here.
The smallest correct gameplay implementation is:

1. Introduce a typed 56-byte objective wire record with `ID`, four states, and
   `[4][3]uint32` tokens. Replace the misleading `Value` plus hard-coded 2/3
   model and retire the incompatible description-string encoder from objective
   traffic.
2. Select the five records in stable UI order: Daily, three Standard, Hidden
   Bonus. Resolve selections through content identities rather than embedding
   screenshot strings.
3. Run the selected objective Lua `Init` behavior (or an equivalent typed
   simulator operation) before `0xb7`, so tokens 1/2 contain authored targets.
   Keep token 0 live in the match session and emit `0xb8` after accepted events.
4. Send `0xb7` after the gameplay clock/session setup and before ordinary scene
   traffic, preserving the observed `0xaf -> 0xb7 -> 0xb8 -> 0x8c` ordering.
   Use `is_notification_shown` only for the transient banner/voice path.
5. At level completion, calculate Gold/Silver/Bronze/Failed from the Lua rules,
   emit the final `0xb8` state(s), then `0xb9`. Persistence should receive only
   the separately defined mission reward/result if product requirements demand
   it—not the live popup array by default.

## Implemented boundary

The gameplay initializer now uses a typed 56-byte record with the exact four
state bytes and twelve token DWORDs. It assigns state `1` to the authenticated
player slot, initializes Damage Often's target to `150`, Obelisks total and
remaining to `3`, Defeat All's target to `94`, and Huge Damage's threshold and
gold count to `35` and `30`. The immediate Finish Quickly update retains its
captured `(1,2,3)` token triplet. `ObjectiveUpdatedMessage` now exposes all
three tokens directly rather than naming token zero `Value` and fabricating
tokens one/two inside the encoder. The incompatible description-string
`Objective.Encode` model has been removed.

Difficulty-dependent scaling beyond the tutorial baseline, final medal
evaluation, `0xb9`, and `0xca` remain unimplemented and must stay match-session
state rather than account persistence. The normal tutorial's Defeat All
numerator is now live session state and is replicated silently as described
below.

The constrained Lua VM now exposes the exact five-argument
`nObjective.SetObjectiveIntData` boundary. A compiled objective job must supply
its nonzero objective identity; accepted calls retain the player selector,
token index, signed 32-bit integer, and both booleans in an authoritative
`ObjectiveDataIntent`. Player indices outside `0..3`/`0xff`, token indices
outside `0..2`, non-finite data, missing identity, and incorrect Lua types fail
closed without emitting a step. Fractional finite data follows the recovered
native cast and truncates toward zero. The two booleans deliberately remain an
opaque two-element flag array because their separate native meanings have not
yet been proven. The production tutorial now applies those intents to a
session-owned, non-persistent objective state. It expands `0xff` writes across
all four player rows, retains source-order mutations, snapshots stable
objective order into `0xb7`, and rejects unknown objectives or invalid
selectors without changing state. The RakNet package now only serializes the
provided snapshot; it no longer owns the tutorial target constants.

The simulator now also owns the packaged objective registration and initializer
boundary. `CompileLuaObjective` executes `nObjective.RegisterObjective`, resolves
the authored `nObjectiveFns.Init` closure, and retains its bytecode provenance.
All five tutorial selections now have content-backed initializer fixtures and
are loaded into the production simulation program in authored objective order.
They produce no initial mutation for `FinishLevelQuickly`, then targets `150`,
`3/3`, `94`, and `35/30` for the other four objectives. Native build-103
evidence at `sub_A05E70` proves that `SetObjectiveIntData` casts its Lua number
to signed 64-bit and then passes the low 32 bits, truncating toward zero. The
authored `0.949999988079071 * 100` therefore initializes token 1 to `94`, not
the earlier handwritten `95` approximation. Catalog-free compatibility tests
retain a typed fallback with the same results; normal build-103 startup uses
the pinned packaged chunks and their bytecode provenance.

Objective execution is now stateful beyond Init. `LuaObjectiveRuntime` retains
the registered callback closures and authored private Lua table for one match
session, serializes callback entry, and gives each invocation a fresh bounded
instruction allowance while retaining the bounded allocation arena. The first
event fixture executes `DefeatAllMonsters.HandleEvent`: 10%, 25%, 49%, 50%, and
full-clear inputs prove the exact intermediate writes, boundary flags, private
25% threshold progression, and callback bytecode provenance. Production
dungeon setup now constructs five independent runtimes from the pinned inputs
for every peer session and derives initialization from those instances, so no
private Lua state is shared across matches.

Normal authored tutorial play now dispatches authoritative defeat events into
that retained runtime. A per-session ledger is constructed from the 30 object
identities in the currently implemented opening, Quadra, security-guard, and
four two-actor horde definitions. Unknown identities and repeated death results
do not advance it. Every first defeat invokes `Death` with
`defeated_count / 30`; the final identity invokes `Death` followed by
`FullClear`, preserving the Lua callback's private 25% threshold. This is the
exact denominator of darkspin's current compatibility route, not a claim that
the unrecovered retail director used the same actor budget. Debug checkpoints
leave the ledger disabled until their skipped prior-defeat seed is modeled, so
they cannot publish misleading low percentages. Each emitted token mutation is
now followed by a silent per-active-player `0xb8`: it preserves that player's
existing medal/state, supplies no voiceover, does not show a notification, and
serializes the complete current token triplet. All-player Lua writes are
expanded only to active player rows so one shared medal byte cannot overwrite
different per-player states. The two Lua booleans remain recorded but do not
control wire presentation until their meanings are proven. Other objectives'
event-native inputs and final medal publication remain the next wiring steps.

Tests should include:

- byte-exact golden tests for `0xb7` (count, all five IDs, four state bytes, and
  all twelve tokens), `0xb8` (including `0xff` all-player behavior), `0xb9`, and
  `0xca`;
- a captured-sequence regression asserting the five screenshot templates
  produce `00:01`, `0 times`, `0 of 0`, `0%`, and `0 damage` from the observed
  zero/default packets;
- content tests for all title/progress GUIDs, the two pool resources and their
  SHA-256 values, and each Lua threshold/token write;
- operation tests for qualifying/non-qualifying damage, obelisk deduplication,
  kill-percent reporting, difficulty scaling, elapsed time, medal thresholds,
  and four-player/all-player addressing;
- ordering tests ensuring initialization precedes scene objects and popup
  updates do not write the account repository;
- UI-boundary tests asserting slots 0/1-3/4 map to Daily/Standard/Hidden Bonus
  and that the notification boolean does not hide the log's Hidden Bonus row.

## Diagnostics retained

All generated output is under `bin/game/logs/objectives-log`, including the five
Lua payloads and disassemblies, IDA name/xref/contract logs, immediate-value
scans, and the read-only content-pool query script. No Go or C implementation
file was edited, and no binary was built or launched for this investigation.
