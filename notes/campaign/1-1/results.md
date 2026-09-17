# Campaign 1-1 Beam Out and results transaction

## Result

The build-103 client, runtime `content.db`, retained traces, and recorded 1-1
walkthrough recover the transaction through entry into chain voting, the exact
client requests, the client-consumed semantic layouts of the 337-byte
chain-vote and 712-byte cashout records, and the fixed-size response envelopes
for Continue Fight and Cash Out. They do **not** recover a byte-exact
`ChainLevelResults (0xaa)` body, retail reward values or formulas, the
reward-commit transaction. The final Cash Out transition is recovered in
`notes/campaign/1-1/cashout-transition.md`: after `AB 00`, the Return to
Spaceship SWF callback selects client state 2 locally and sends no second
RakNet, Blaze, or HTTP acknowledgement. The complete record field evidence is
in `notes/campaign/1-1/result-fields.md`.

The narrowest safe implementation is therefore:

1. retain the authoritative final-clear result and publish
   `mbBossComplete=true`;
2. accept one Beam Out request only from that completed match;
3. freeze gameplay, perform the recovered beam-out presentation, and create one
   immutable result snapshot;
4. finalize any objectives selected for this run into that snapshot;
5. enter chain voting and answer the client's exact request from the same
   snapshot;
6. on Continue, carry the at-risk reward ledger into 1-2 without granting it;
7. on Cash Out, atomically grant the snapshot once before displaying a grant
   that the client could mistake for durable inventory, then retain Cash Out
   while the client owns the later Return to Spaceship transition.

Steps 1-5 are exact at the operation/state level. The local server now uses its
pre-existing bounded A9 reconstruction as an explicitly documented
compatibility fallback so the single-player result screen can advance. Cash
Out enters state 12 and uses `AC 04` to atomically commit the documented
deterministic reward and progression before sending `AB 00`. Replays return the
same committed receipt without another write, and storage failure leaves the
client in Cash Out for retry. After the presentation, Return to Spaceship moves
the client from state 12 to state 2 locally without a server close packet.

This note begins after the final-arena transaction has committed
`mbBossComplete=true`. Encounter composition and the final-clear predicate are
covered by `notes/campaign/1-1/completion.md` and are not reopened here.

## Evidence grades

- **Exact client contract**: build-103 receiver/sender behavior, fixed copy
  length, decoded reflection, or a retained live request proves it.
- **Exact content contract**: `content.db` proves the authored identity or
  adjacency, but not a server policy.
- **Walkthrough-visible**: the recording proves presentation and visible order,
  not packet order or persistence.
- **Server authority**: the retail server had to choose the behavior; the
  client/content evidence does not determine it.
- **Fallback**: the conservative local decision recorded in `notes/help.md`.

No retained retail 1-1 packet capture covers this result sequence.
`bin/server/darkspin/logs/traces/client.jsonl` contains no `0xa9`-`0xac` or
`0xb9` result exchange. Shared Cryos live evidence is used only for the Beam
Out gate and request and is not treated as proof of 1-1 rewards or ordering.

## Ordered transaction

| Order | Boundary | Exact client/content requirement | Server-owned decision or fallback |
| ---: | --- | --- | --- |
| 0 | Final clear already committed | Director reflection field 3 is `mbBossComplete`; minimal update is `8b 08 01`. The client exposes `HUD_BeamOut.swf` / `RETURN TO SHIP` only after this field makes `nGameDirector.IsBossDead()` true. | The final-clear predicate, success/failure race, and durable match latch are authority. Retain one terminal success epoch so a replayed Beam Out cannot complete another match. |
| 1 | Player clicks Beam Out | C2S application `0x88` has eight payload bytes: `status:u32le=0x20`, `progress:f32le=1.0`, hence `88 20 00 00 00 00 00 80 3f` at the application boundary. `MaxisBeamOut.OnBeamOutClicked` has a local one-shot latch; live testing observed this request. | Authenticate the player and match; require exact progress `1`, committed boss completion, a living/eligible participant, and no prior reservation or commit. Multiplayer quorum/host policy is missing. |
| 2 | Reserve and freeze | The client supplies no reward, objective, level, or vote fields in Beam Out. | Reserve by `(game ID, success epoch, player/party)`, reject duplicates, close action/damage/spawn input, and stop encounter timers. Roll back only if beam presentation construction fails before the snapshot is committed. |
| 3 | Beam presentation | The recovered character beam-out path is presentation; it is not a result or persistence acknowledgement. | Emit the presentation from retained authoritative object/position state. Do not infer that presentation delivery commits rewards. |
| 4 | Freeze result snapshot | `ObjectivesComplete (0xb9)` is a result snapshot, not the signal that enables Beam Out. World drops visible before the result screen are distinct from the later chain reward offer. | Snapshot selected objectives and medals, per-player level statistics, ordinary loot already accepted, XP/DNA deltas, campaign completion, chain position, and the still-at-risk cashout offer. Use one immutable snapshot for every retry and later choice. |
| 5 | Finalize objectives | `0xb9` is `[count:u8] + count * 56-byte objective records + four one-byte result fields for each of four players`. It must describe the final selected-objective state. | The server emits the selected objective snapshot after accepted Beam Out and before chain-voting presentation. The sixteen unrecovered result bytes use zero; latent/unselected objectives are excluded. |
| 6 | Chain-level result handoff | Application ID `0xaa` is assigned to `ChainLevelResultsMsgs`, but this pass recovered no build-103 body reader, fixed copy, retained trace, or safe field map. | Keep an internal chain-level snapshot. Do not emit a fabricated `0xaa`. Whether `0xaa` is required, optional, empty/subtyped, or ordered before voting remains a protocol blocker. |
| 7 | Enter chain voting | Client state `11` is `cChainVotingState`. `GameState (0x8a)` carries two `u64le` times, state byte, `GameType:u32le`, and trailing mode word, but live testing proved that metadata alone does not select the UI. Exact `ChainGame (0xad)` subtype `0` selects state 11; on state entry the client sends C2S `ac 00`. | After gameplay teardown, send the state metadata followed by `AD 00` once. In single-player there is no quorum to await. Treat `AC 00` as a retryable request for the already-frozen snapshot, not as a second completion. |
| 8 | Supply voting/result screen | S2C `a9 00` is followed by exactly 337 body bytes. Its completed level, level index, tier, time, chain count, six enemy previews, two presentation triplets, ordinal guard, and offered next level now have exact offsets and widths. S2C `a9 01 <seconds:f32le>` updates the countdown. S2C `a9 02 <bool:u8>` selects party/cashout handling. | The local encoder now uses the recovered layout and separately identifies `zelems_1.Level` as completed and `zelems_3.Level` as offered. Actual remaining time, reward tier/counts, presentation meanings, countdown duration, party flag meaning, and retail emission order remain authority or unresolved semantics. |
| 9a | Continue Fight choice | The Planet Screen calls C2S `ac 01 01 <selected-record-id:u32le>`. The leading `1` and total four-byte identifier are exact. Runtime chain resource `693` places `zelems_1.Level` at ordinal 0 and `zelems_3.Level` at ordinal 1; the walkthrough names the latter Gnarled Plateau 1-2. | Validate the identifier against the offered record rather than trusting it. The identifier's semantic type is not proved; the incomplete server calls it a squad ID. Carry the at-risk ledger into a new 1-2 run. The precise state/departure/setup order is missing; fallback is normal `PreDungeon` setup for `zelems_3` after the vote is accepted. |
| 9b | Cash Out choice | `PlanetScreen.OnCashOut` sends exactly C2S `ac 02`. | Resolve a single-player vote immediately. Do not grant solely because this choice packet arrived unless the grant can be completed atomically and replayed idempotently. |
| 10 | Enter cashout | Client state `12` is `cChainCashOutState`; its entry sends exactly C2S `ac 04`. | Move to state 12 only after the Cash Out decision is accepted. Treat `AC 04` as a retryable read of the same committed/grantable snapshot. |
| 11 | Supply cashout presentation | S2C `ab 00` is followed by exactly 712 body bytes. Exact fields cover planets completed, DNA, four-player starting/final XP, medal counts, rarity boundaries, bonus flags, and a 4x4 loot matrix with exact record strides and item-card fields. | The record is presentation, not a grant command. The server atomically commits its documented deterministic fallback first, then sends AB and replays the same receipt for repeated `AC 04`. |
| 12 | Commit rewards | The UI's later `api.creature.unlockCreature` request submits `template_id`; this proves a separate account-authorized selection step, not that `AB 00` granted a creature. | Current fallback: atomically and idempotently commit progression, one deterministic level-11 rarity-weighted item, and its private completion event keyed by the frozen result ID. Keep XP unchanged and DNA, medal, bonus, and creature-entitlement grants at zero until their server policies are recovered; the broader all-or-nothing reward model remains future work. |
| 13 | Final transition | After AB populates MaxisCashOut, Return to Spaceship calls `CashOut.OnBackToSpaceshipClicked(false)` and native code locally selects state 2. It emits no RakNet, Blaze, HTTP, second `AC 04`, or server close acknowledgement. | Leave the session in Cash Out after AB. Ordinary ship initialization may later refresh account state, but it is not part of the cash-out close contract. |

The strict dependencies are:

```text
mbBossComplete
  -> accepted Beam Out
  -> gameplay freeze + immutable result snapshot
  -> final selected-objective snapshot
  -> 8A state metadata / AD 00 ChainVoting selector / AC 00 / A9 result data
  -> exactly one accepted Continue or Cash Out branch
     -> Continue: carry pending ledger -> 1-2 PreDungeon
     -> Cash Out: atomic grant -> ChainCashOut / AC 04 / AB data -> ship
```

Objective evaluation can occur internally at final clear, but the snapshot sent
to the result UI must be immutable no later than accepted Beam Out. The client
evidence does not prove whether retail placed `0xb9` or `0xaa` before or after
the Beam Out request, so the implemented `0xb9` placement is a documented
compatibility fallback rather than claimed retail order.

## Exact message contracts

All byte strings below start at the one-byte application ID; RakNet framing,
reliability, ordering channel, and timestamps are outside the contract.

### Director completion and Beam Out

```text
S2C DirectorState minimal boss-complete update
8b 08 01

C2S PlayerStatusUpdate Beam Out
88
20 00 00 00             status = 0x20
00 00 80 3f             progress = 1.0f
```

The Beam Out request contains no exit-object ID. The colocated authored
`LevelExitPoint` is therefore not an input contract and is not evidence for a
contact-to-complete rule.

### Objective completion

```text
S2C ObjectivesComplete
b9
<count:u8>
repeat count times:
  <objective record:56>
repeat 4 players:
  <result field 0:u8>
  <result field 1:u8>
  <result field 2:u8>
  <result field 3:u8>
```

Each objective record uses the same 56-byte wire representation consumed by
`ObjectivesInit` and `ObjectiveAdd`. `TouchAllObelisks` is only eligible content
with ID `0x61c07561`; it must appear here only if server selection placed it in
the run. Its authored status is Gold/Silver/Bronze/Failed at 3/2/1/0 unique
touches, but it does not complete the mission.

There is no evidence-backed 1-1 value for `count`, the selected IDs, or the four
trailing per-player byte fields. The walkthrough's visible medal totals (`0`, `1`, `4`) describe
that recorded run's presentation, not a reusable packet body or retail
selection rule.

### Chain voting and choices

```text
C2S request voting data       ac 00
S2C voting data               a9 00 <337-byte body>
S2C voting countdown          a9 01 <seconds:f32le>
S2C party/cashout decision    a9 02 <flag:u8>
C2S Continue Fight            ac 01 01 <selected-record-id:u32le>
C2S Cash Out                  ac 02
```

The recovered subtype-0 consumer copies 337 bytes into the voting state. The
current repository encoder names plausible offsets for level, six enemy nouns,
two level nouns, movies, and voice, but those names are reconstruction, not a
complete build-103 semantic proof. They are not promoted to an exact contract
here.

### Cashout

```text
C2S request cashout data      ac 04
S2C cashout data              ab 00 <712-byte body>
HTTP optional creature pick   api.creature.unlockCreature(template_id=<noun>)
```

The client cashout handler accepts only subtype 0 and copies exactly `0x2c8`
bytes. The exact encoder and the evidence-bounded deterministic fallback are
implemented; opaque fields remain zero and are not promoted to retail meaning.

### `ChainLevelResults (0xaa)` negative result

The application enum assigns `0xaa`, but unlike `A9 00` and `AB 00`, the
available build-103 pass found no exact reader/copy contract and no retained
trace. The result screen demonstrably consumes the `A9 00` voting record after
entry into `cChainVotingState`; that does not prove `0xaa` is absent from retail.
It proves only that no safe `0xaa` bytes can be authored from current evidence.

Implementation rule: model chain-level results internally, reserve opcode
`0xaa`, and emit nothing on it until a receiver or retail trace supplies a body
and ordering contract.

## Reward and durability boundary

Three reward classes must remain separate:

| Class | Evidence | Durability rule |
| --- | --- | --- |
| In-level drops | The walkthrough shows multiple encrypted items collected before Beam Out. | Preserve the existing pickup transaction. Do not recreate or double-grant these from the result snapshot. Whether retail risked these on later chain defeat is not established; local fallback keeps already committed pickups. |
| Chain cashout offer | The result screen offers one level-11 reward for collecting after 1-1, or two level-13 rewards for continuing to 1-2. | Keep an at-risk, deterministic reservation in the result snapshot. Continue carries it without inventory grant; Cash Out commits it once. Recorded level/count/roll values are a visible example, not a formula. |
| Creature unlock choice | Build 103 sends a later account request containing a template noun. | Store eligible choices/entitlement separately and validate the selected template. Do not infer an unlock from the cashout packet alone. |

The fallback cashout commit should be one feature operation with a stable result
ID. It must either commit all authoritative effects or none:

- completed 1-1/campaign position;
- chain termination and cashout state;
- selected-objective medal results;
- XP, resulting account level, and DNA;
- awarded part/inventory records and any capacity decision;
- cashout bonus consumption;
- creature-choice entitlement, but not an unselected creature.

Prepare and validate the complete write before mutating the in-memory session.
On success, retain the committed snapshot long enough to replay `AB 00` and
profile refreshes without granting twice. On failure, remain in the results
state with the reservation intact; do not show a newly granted item that is not
durable.

For Continue, commit only durable mission facts that must survive the next
session (at minimum the accepted 1-1 success and any already-committed world
pickups), then attach the at-risk ledger to the new chain epoch. Do not insert
the cashout item into inventory. A later defeat may discard only that at-risk
ledger, not rewrite the already accepted 1-1 success.

## What the walkthrough and `content.db` establish

The recorded ending visibly orders:

1. Illust's defeat and final field drops;
2. a `Collect Reward or Continue` explanation;
3. a Planet Screen marking Floating Isles 1-1 complete;
4. a choice between one level-11 reward and continuing to Gnarled Plateau 1-2
   for two level-13 rewards;
5. collection followed by roll 39 and the displayed item Leto's Pale Guard.

The recording does not visibly prove a `RETURN TO SHIP` click, packet order,
the moment the roll was chosen, or persistence. Edited time between the arena
and ship result screen cannot supply those facts.

Runtime `content.db` proves that content resource `693` begins with:

| Ordinal | Level reference | Resolved level |
| ---: | --- | --- |
| `0` | `zelems_1.Level` | level `56`, Floating Isles 1-1 |
| `1` | `zelems_3.Level` | level `60`, Gnarled Plateau 1-2 |

The resource contains later repetitions and alternative ordering, so only the
ordinal-0 to ordinal-1 adjacency is used for the recorded first chain. It does
not determine the selected-record ID in `AC 01`, reward levels, vote timeout,
or grant formula.

## Exact contract versus server authority

| Concern | Exact client/content contract | Server authority |
| --- | --- | --- |
| Beam eligibility | Client waits for `mbBossComplete`; click sends status `0x20`, progress `1`. | Who may click, party/quorum rules, retry, timeout, and success/failure conflict. |
| Objectives | `0xb9` record/count/trailing-field widths; selected objective records are final snapshots. | Selected set, status evaluation, trailing values, and emission order. |
| Chain result | Opcode `0xaa` is assigned only. | Necessity, subtype, body, timing, and persistence effect. |
| Result/vote UI | State 11 requests `AC 00`; `A9` subtypes and fixed subtype-0 size are exact. | Full body semantics, offered next record, timer, party vote, disconnect/tie policy. |
| Continue | Exact `AC 01 01 <u32le>` request; `zelems_3` follows the initial `zelems_1` content entry. | Identifier validation, carried ledger, session creation, and state/departure ordering. |
| Cash Out | Exact `AC 02`, then state-12 `AC 04`; `AB 00` body size is exact; Return to Spaceship selects state 2 locally without another request. | Retail reward formulas and opaque AB fields. |
| Rewards | UI consumes medal/XP/DNA/roll/loot/unlock presentation; creature selection is a separate HTTP request. | Every formula, eligibility rule, durable transaction, idempotency key, and account refresh. |

## Remaining implementation gates

The client-complete compatibility path now has the bounded `A9 00` voting
record, the exact `AB 00` encoder, atomic replay-safe Cash Out persistence, and
the client-local return transition. The remaining evidence boundaries are:

1. recover or capture the `0xaa` consumer and decide whether the message is
   required in this path;
2. recover the exact state/departure ordering after Continue, beyond the
   working `zelems_3` pre-dungeon transition;
3. recover the sixteen trailing `0xb9` player-result bytes and its retail
   ordering to replace the zero-valued compatibility fallback;
4. replace the documented A9/AB reward, timer, medal, and opaque-field
   fallbacks if a retail authority capture recovers their original formulas.

The server therefore omits only the unsupported `0xaa` body. It sends the
selected-objective `0xb9` snapshot before chain voting, sends A9 from the
immutable result snapshot, and sends AB only after the documented reward
transaction has committed.

## Evidence cross-references

- `notes/campaign/1-1/completion.md`: boss-complete reflection, Beam Out request, chain
  request inventory, and missing-authority boundary.
- `notes/campaign/1-1/objectives.md`: exact objective record/update behavior and why
  registration is not selection.
- `notes/campaign/1-1/native.md`: seven-field `cAIDirector` reflection.
- `notes/design/architecture/raknet-gameplay-exchange.md`: application framing and
  build-103 objective layouts.
- `notes/campaign/progression.md`: account-side creature
  unlock request.
- `bin/video/walkthrough/1-1/info.md`: timestamped visible ending and reward
  choices.
- `bin/game/GameBin/Game.c`: canonical build-103 sender, receiver,
  state, and fixed-copy evidence.
