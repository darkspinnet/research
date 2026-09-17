# Build-103 campaign 1-1 cash-out rewards

## Implementation progress

The server now has a single-save `CommitCampaignCashOut` operation that
atomically advances chain progression, appends the selected reward item, and
records a private result-and-part identity. Replays return the exact committed
part without another write; a full inventory rejects without mutation, and a
storage failure rolls back progression, inventory, and the identity marker.

The exact 712-byte `AB 00` encoder and immutable receipt-to-presentation
projection are also implemented. Opaque bytes remain zero, account XP is
projected as unchanged until an XP reward policy exists, and the committed
level-11 item supplies the first player reward card. On `AC 04`, the gameplay
transport now selects that deterministic common item using the hero class and
science context frozen when the participant's Beam Out was accepted,
commits it with progression, freezes the returned receipt, and only then sends
`AB 00` and its 712-byte presentation record. Generation, inventory, and
storage failures keep the client in Cash Out for retry. Repeated `AC 04`
requests replay the same committed presentation. The later Return to Spaceship
button transitions the client locally from state 12 to state 2, so the server
must not send the former immediate `AF 00` compatibility response.

## Result

Build 103 contains an exact client-side formula for the **Planet Screen reward
preview level and count**. For the first completed planet it evaluates the
previous difficulty and displays one level-11 item; Continue evaluates the
offered difficulty with a one-planet chain bonus and displays two level-13
items. The formula is in `sub_9CADD0` and its `sub_527DD0` callers; the
walkthrough only corroborates those outputs.

The later `AB 00` cash-out record does not run a reward generator. Its receiver
copies 712 bytes and `sub_407540` translates supplied DNA, XP totals, medal
totals, rarity boundaries, item identities/levels/rarities/rolls, and bonus
flags into UI properties. No inventory, profile, or account write is reachable
on that path. Build 103 therefore does **not** disclose the retail server
formulas for DNA, XP earned, medal awarding, rarity selection, rigblock/affix
selection, or the durable grant transaction. Those values must be selected,
validated, and committed by the server before it sends a grant-looking `AB 00`.

The complete future replacement for Darkspin's progression-only compatibility
result is partly recovered and partly server policy:

- reproduce the exact level/count preview below;
- freeze one immutable per-player result containing all server-selected reward
  values;
- atomically commit completion, progression, DNA, XP, medals, items, bonus
  consumption, and a creature-choice entitlement once;
- answer repeated `AC 04` requests from that committed snapshot without
  granting again;
- grant a selected creature only through the later separately authorized
  `api.creature.unlockCreature` operation.

The atomic and replay rules are a conservative fallback, not recovered retail
server behavior. The executable supplies an address-backed negative result:
the only `AB 00` chain is receive, fixed copy, UI construction, and state
selection.

The current implementation deliberately commits only the evidence-bounded
subset: chain progression, one deterministic level-11 rarity-weighted item, and one
private completion event. It presents unchanged account XP and zero new DNA,
medals, bonus consumption, and creature unlocks. Those fields remain future
work; granting guessed balances or entitlements would make the durable account
state harder to correct when the missing retail policy is recovered.

## Evidence labels and terminology

Offsets are relative to the subtype-0 bodies mapped in
`notes/campaign/1-1/result-fields.md`. Virtual addresses are build-103 image
addresses from canonical `bin/game/GameBin/Game.c` and its IDB.

- **Native exact**: arithmetic or a field read is present in the executable.
- **Content exact**: decoded package content supplies a scalar/table, not a
  server reward policy.
- **Weak visual bound**: the walkthrough shows one run's output only.
- **Server-only**: the client has no derivation or durable mutation.
- **Fallback**: conservative Darkspin policy chosen without retail server
  evidence.

The cash-out code calls these values XP and `playerLevel`; `AB 00` carries no
creature ID or per-creature XP field. This report therefore calls them
**account/player XP totals**. Creature-specific XP is not representable in this
record and remains server-only if Darkspin introduces it.

## Exact equipment preview

### Difficulty decomposition

`sub_9BCC70` at `0x009BCC70` (`Game.c:1365873`) decomposes unsigned
difficulty `d` using the packaged planets-per-chain scalar `P`:

```text
major = floor(d / P) + 1
minor = d mod P
if minor == 0:
    major = major - 1
    minor = P
```

The shipped `LootPreferences` value is `P=4`; the executable default is also 4
at `Game.c:119414`, and the property override loads at
`Game.c:1381745`. For positive `d`:

```text
major = floor((d - 1) / 4) + 1
minor = ((d - 1) mod 4) + 1
```

The exact native zero edge is `(major,minor)=(0,4)`.

### Item-level formula

`sub_9CADD0` at `0x009CADD0` (`Game.c:1378002`) consumes difficulty `d`,
mode `g`, optional bonus flag `f`, and represented chain length `c`. With the
decoded shipped scalars `majorScale=10`, `minorScale=1`, and bonus step 6:

```text
modeOffset(g) = 6 * (g - 4), if g >= 4
                6 * (g - 1), otherwise

chainBonus(c, minor) = min(c - 1, 3), if c > 1 and minor != 4
                       0, otherwise

itemLevel(d,g,f,c) =
    10 * major + minor + modeOffset(g)
  + 6 * [g >= 4 and minor == 4]
  + 6 * [f]
  + chainBonus(c, minor)
```

`sub_9C9A90` at `0x009C9A90` supplies the mode offset
(`Game.c:1377014`). `sub_9CADD0` returns 1 if its tuning object cannot be
resolved. The campaign caller uses `g=4` and `f=false`, so both the mode offset
and optional flag term are zero for this screen.

`sub_527DD0` at `0x00527DD0` calls the formula as follows
(`Game.c:367586-367675`):

```text
collectLevel  = progressionFloor(itemLevel(nextDifficulty - 1, 4, false,
                                             planetsCompleted))
collectCount  = min(planetsCompleted, 4)

continueLevel = progressionFloor(itemLevel(nextDifficulty, 4, false,
                                             planetsCompleted + 1))
continueCount = min(planetsCompleted + 1, 4)
```

For completed 1-1, `nextDifficulty=2` and `planetsCompleted=1`:

```text
collect  = itemLevel(1,4,false,1) = 10*1 + 1 = 11; count 1
continue = itemLevel(2,4,false,2) = 10*1 + 2 + 1 = 13; count 2
```

This corrects the older semantic label for `A9 +0x004`: it is the offered/next
difficulty coordinate, not “1 means 1-1.” The client decomposes it for visible
`1-2` and exports it unchanged as `LABS_local_LEVEL_INT_DIFFICULTY`
(`Game.c:367731-367747`), while Collect intentionally subtracts one.

`progressionFloor` is `sub_9CB050` at `0x009CB050`
(`Game.c:1378151`). When its mode/session predicate enables the floor, it
returns the larger of the candidate and:

```text
itemLevel(max(1, accountChainProgression - 4), mode, false, 1)
```

The enable predicate is `sub_9BF410` at `0x009BF410`. This is local preview
normalization, not proof of a server grant.

### Difficulty recommendation is separate

The decoded 72-pair DifficultyTuning presentation array begins `(0,4)` for
difficulty 1, `(2,5)` for difficulty 2, `(3,6)` for difficulty 3, and `(4,8)`
for difficulty 4. `sub_527DD0` selects a pair and adds `3*starTier` to each
endpoint, capped at 200. The recorded next-threat `LVL 2-5` is therefore the
difficulty-2 recommendation, not a reward rarity, DNA, XP, or medal multiplier.

Runtime `difficulty_tuning` also exposes combat health/damage/rating values.
No `AB 00` consumer reads them and no evidence connects them to cash-out
rewards.

## `AB 00`: supplied values and presentation math

`sub_449410` at `0x00449410` reads exactly 712 bytes
(`Game.c:195673-195683`). `sub_409F10`/`sub_409D60` preserve the record and
call `sub_407540` at `0x00407540`.

| Body offset | Supplied value | Client work | Authority result |
| ---: | --- | --- | --- |
| `0x000` | planets completed | Published; medal-tooltip divisor. | Server supplies; client does not guard zero. |
| `0x004` | DNA amount | Published directly as `dnaAmount`. | Server-only amount/formula. |
| `0x014`, `0x024` | starting/final XP totals, four players | Subtracts for `xpAmount`; derives level/bar fill. | Server-only earned amount and final total. |
| `0x034`, `0x044`, `0x054` | gold/silver/bronze counts | Publishes counts and per-planet averages. | Server-only objective/tier/count. |
| `0x064`, `0x074` | two rarity boundaries/player | Converts to visible chances/ranges. | Server-only boundary generation. |
| `0x084` | four bonus flags | Passed to presentation. | Server-only eligibility/consumption. |
| `0x088 + 0x90*p` | four 36-byte item records/player | Renders hashes, level, rarity, roll. | Server-only count/identity/rarity/roll. |

There is no cash-out DNA derivation. In-level pickup arithmetic is a different
transaction and cannot be reused as the `AB +0x004` formula. Likewise, the only
exact XP amount is `finalXP[p]-startingXP[p]`
(`Game.c:146971-146981,147140-147142`). No difficulty or chain multiplier
is applied to DNA, XP, or medals on this path.

### XP curve

`sub_9CEA00` at `0x009CEA00` maps supplied total XP `x` to the first one-based
level whose inclusive endpoint is at least `x`. `sub_9CEA40` at `0x009CEA40`
defines the level start:

```text
level(x) = first i such that x <= endpoint[i]
start(1) = 0
start(i) = endpoint[i-1] + 1, for i > 1
fill(x,i) = 100 * (x - start(i)) / (endpoint[i] - start(i))
```

`sub_407540` applies this independently to starting and final totals
(`Game.c:147210-147230`). The packaged LabsTuning array contains 99
endpoints. Its first ten are `100, 200, 3000, 6000, 9000, 12000, 15000,
18000, 21000, 24500`; levels 95-99 end at `2,080,500, 2,149,500, 2,225,500,
2,310,500, 2,410,500`. The property loads at
`Game.c:1381657-1381661`. This is exact presentation after the server
supplies totals; it does not determine 1-1 XP earned.

### Medals and rarity boundaries

The medal arrays are copied directly to `numGoldMedals`, `numSilverMedals`, and
`numBronzeMedals` (`Game.c:147008-147016`). Tooltip code divides each
supplied total by planets completed; it does not evaluate objectives or choose
a tier. Authored objective thresholds prove only an individual objective's
rule, not its selection or the aggregate cash-out counts.

For player `p`, let `A=AB[0x064+4*p]` and `B=AB[0x074+4*p]`. The client
publishes:

```text
uniqueChance = max(100 - A, 0)
rareChance   = max(100 - B, 0)
epicChance   = B

visible ranges:
  1 .. 100-A
  max(1,101-A) .. 100-B
  max(1,101-B) .. 100
```

These are formatting transforms at `Game.c:147032-147124` and
`147355-147430`, not recovered generator probabilities. The client does not
verify that item rarity or roll agrees with them. Item rarity enum 6 merely
sets the corresponding `playerNEpicLoot` UI boolean
(`Game.c:147164-147207`).

### Equipment record

The A9 preview count is exactly `min(chainLength,4)`. AB instead always has
capacity for four records per player and no separate count scalar. The first
nonzero rigblock marks that player's block populated; the UI receives four
slots. Each 36-byte record supplies:

```text
+0x08 rigblock hash       +0x0c suffix hash
+0x10 prefix A hash       +0x14 prefix B hash
+0x18 item level          +0x1c rarity enum
+0x20 displayed roll
```

The first eight bytes remain opaque. No client path derives rigblock, affixes,
rarity, level, or roll from the boundaries or difficulty. The exact preview
formula gives an expected level, but the server must create authoritative item
records and choose populated slots. Package/runtime content supplies loot stat
and eligibility tuning, not the missing cash-out selector.

## Creature unlock eligibility

For the current player, `sub_407540` scans its local creature catalog and emits
at most three choices (`Game.c:147232-147341`). It reads an unlock
threshold at runtime object offset `+28` and includes an entry exactly when:

```text
startingLevel < unlockLevel <= finalLevel
```

It publishes type, class, image, and `numCreatureUnlocks`; it does not grant a
template. On selection, `sub_40E8B0` at `0x0040E8B0` resolves the entry and
calls `sub_4B6F70`. `sub_4A90B0` at `0x004A90B0` creates a separate
`api.creature.unlockCreature` request with `template_id`
(`Game.c:151679-151694,268675-268711,278947-278969`).

Runtime `creature_template` has no unlock-level field. A decoded 48-record
`UnlocksTuning` resource was inspected, but its mixed categories do not provide
a safe creature-template/level mapping for this consumer. Exact 1-1 identities
and account authorization therefore remain server-only. Commit an entitlement
with an eligible allowlist, then validate and consume it in the later account
operation.

## Atomicity, replay, and address-backed negatives

The exact client chain is:

```text
AB 00
  -> sub_449410 (0x00449410): copy 712 input bytes
  -> sub_409F10 (0x00409F10)
  -> sub_409D60 (0x00409D60): copy 0x2c8 bytes to cash-out UI state
  -> sub_407540 (0x00407540): build UI properties
  -> sub_455870(12): select client state
```

No branch calls part-list/inventory APIs, mutates the account profile, writes
persistence, or acknowledges a durable grant. The creature pick separately
leaves this UI via HTTP. This proves `AB 00` is a presentation snapshot and is
an address-backed negative result for client-owned durability.

No retained retail trace covers the 1-1 `A9`-through-`AC`/`AB` exchange.
Neither executable nor content identifies a retail result ID, transaction
boundary, retry key, commit acknowledgement, capacity policy, or disconnect
recovery rule. Replaying identical AB bytes is client-side presentation-safe,
because the receiver only recopies/rebuilds UI; it does not prove retail server
idempotency.

The required Darkspin fallback is one feature operation keyed by immutable
`(game ID, terminal-result epoch, player ID)`:

1. Validate the frozen result, item capacity, XP/DNA bounds, objectives, and
   creature allowlist before mutation.
2. In one durable transaction commit campaign/chain completion, progression,
   XP, DNA, medal records, all items, bonus consumption, and creature-choice
   entitlement. All effects commit or none do.
3. Build `AB 00` from that committed snapshot. Repeated `AC 04` returns the
   same bytes and never reruns selection or increments balances.
4. On storage failure, retain the retryable reservation and do not present a
   newly granted reward.
5. Have `api.creature.unlockCreature` atomically validate and consume the
   entitlement rather than trust the selected template.

This prevents presentation getting ahead of storage and retries duplicating a
grant. It is not a claim about missing retail server behavior.

## Walkthrough: weak bounds

The edited recording shows one run with completed 1-1, offered Threat 1-2,
next recommendation `LVL 2-5`, medal totals `0/1/4`, one level-11 Collect item,
two level-13 Continue items, rarity text splitting rolls at 65/66, roll 39, and
level-11 Leto's Pale Guard. Only level/count and recommendation are reproduced
independently by native formulas/content. Medal totals, boundaries, roll,
rarity, identity, and apparent cash-out count remain weak visual bounds. Video
provides no packet, RNG, commit, replay, or persistence evidence.

## Reproducibility inventory

| Artifact under `bin/game/logs` | Bytes | SHA-256 | Use |
| --- | ---: | --- | --- |
| `labs-tuning.bin` | 4,298 | `b57ca3826416ecc6f23db0ad92b9d6bdce2faa376809a5505bb2869a0014bcce` | XP endpoints |
| `difficulty-tuning-103.bin` | 2,120 | `78717ee97b03f0062b804e3434715c7c18a55feaa41436318858274c3c4184b2` | 72 recommendation pairs |
| `loot-preferences.prop` | 1,212 | `b44be85aebf22433ff7618040ff5dcefc2ceba10abce76754a4a6e9b72e53940` | preview scalars |
| `1-1-rewards/unlocks-tuning.bin` | 7,517 | `1aac92a86bf68d3ba9c7489f3d572d675bbd8bf1358b18ff2afbd7ad038d5826` | negative mapping check |

Runtime `content.db` corroborates source resource 7455 for `loot_tuning` and
7813 for `difficulty_tuning`. No fixed 1-1 DNA, XP, medal, rarity, rigblock,
affix, or creature reward row was found.

## Recovered versus unresolved

| Concern | Exact recovered | Server-only remainder |
| --- | --- | --- |
| DNA | AB amount is displayed directly. | Amount, scaling, bonus, cap, grant. |
| Account/player XP | Delta and 99-level presentation curve. | Earned amount, multipliers, final total, grant. |
| Creature XP | No field or creature ID in AB. | Entire policy/formula, if supported. |
| Medals | Three count arrays and tooltip average. | Objective selection, tiers, aggregation, persistence. |
| Equipment count | A9 `min(chainLength,4)`; AB capacity four/player. | Populated AB slots and capacity behavior. |
| Equipment level | Exact A9 formula and chain/mode/boundary terms. | Retail enforcement in granted item. |
| Equipment rarity | Formatting of supplied boundaries; enum-6 UI test. | Boundary/RNG generation and rarity selection. |
| Equipment identity | Exact hash/affix record positions. | Eligibility, weighting, persistent item ID. |
| Unlocks | Level crossing, max three, separate HTTP choice. | Template-level map, entitlement policy, grant. |
| Multipliers | Exact 22-hour profile cooldown, cash-out grant flag, Beta 7's 1.5x Daily Bonus headline, and equipment preview/recommendation math. | Daily rounding/Purified treatment plus DNA, XP, and remaining rarity multipliers. |
| Atomicity/replay | AB is presentation-only; identical replay has no client grant. | Retail transaction/key/ack; fallback required. |
