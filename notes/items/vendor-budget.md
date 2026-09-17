# Build-103 vendor transactions and hero editor DNA budget

Date: 2026-07-23  
Client: `bin/game/GameBin/Game.exe`, `version_bin.txt` = `5.3.0.103`  
Canonical decompilation: `bin/game/GameBin/Game.c`

## Result

Two independent systems have been presented as one number:

- `api.inventory.vendorParts` is an ordered form-encoded command batch. Build
  103 emits four operation characters: `b`, `p`, `s`, and `f`. The operand is
  an unsigned 64-bit inventory/offer item ID, not a rigblock, reference, or
  content asset ID.
- The hero editor's displayed DNA budget is the editor's category-0 budget.
  It is not the creature response's `item_points`, the computed gear score,
  the account's spendable DNA, or `unlock_stats`. A separate editor-launch
  value overwrites category 0. A zero launch value therefore produces the
  observed `/0` even when the persisted creature says `item_point=300`.
- The observed `20` is editor model-part cost accumulated by the editor
  (`sub_6A22E0`), not the equipped loot-part item-point contribution used by
  `api.creature.updateCreature`. The retained HTTP logs do not preserve the
  decoded editor model blocks needed to name the particular geometry assets
  that sum to 20, but the calculation path and the reason the denominator is
  zero are proven.
- `unlock_stats` selects the authored hero-stat tier passed to
  `sub_9E3740`. No build-103 reference to `unlock_stats` feeds the editor's
  category-0 budget. Treating genetic upgrades as editor DNA-capacity
  upgrades would therefore join unrelated fields.

The current server implements all four operations. Purchases use the exact
packaged offer, normal sell/buyback resolves the generated content price,
flair conversion applies the recovered rigblock flag guard and full generated
price, and successful responses return authoritative affected parts and DNA.
The all-or-nothing ordered batch rule remains a conservative compatibility
policy because no retained response proves the retired server's partial-failure
boundary.

## Evidence labels

- **Decompiler-proven** means the behavior is directly present in build-103
  code or machine instructions.
- **Capture-proven** means it appears in a retained local request or server
  log.
- **Current-state** means it is a read-only observation of Test's current
  database or current Go implementation.
- **Retired-policy** means behavior inherited from recap/current darkspin or
  selected as a compatibility fallback without a retained retail response.
- **Inference** means it is the narrowest server behavior consistent with
  the client, but the retired server has not been observed performing it.

## Source inventory

### Canonical client paths

| Subject | Address | Fact |
| --- | ---: | --- |
| Vendor request constructor | `0x004AA980`, `sub_4AA980` | Encodes `transactions` and targets `api.inventory.vendorParts`. |
| Vendor queue flush | `0x00430A00`, `sub_430A00` | Maps the four internal operation values to wire characters. |
| Single vendor action | `0x00435820`, `sub_435820` | Performs the client-side checks, optimistic changes, and queueing. |
| Bulk vendor action | `0x004BDB80`, `sub_4BDB80` | Sends selected IDs as `s`, `b`, or `f`. |
| Vendor success parser | `0x004AE360`, `sub_4AE360` | Requires `response/parts` and `response/dna`. |
| Vendor failure parser | `0x004AE520`, `sub_4AE520` | Reads `response/code` and publishes failure. |
| Vendor result consumer | `0x004C1990`, `sub_4C1990` | Upserts returned parts and publishes returned DNA. |
| Generated content price | `0x009CD3D0`, `sub_9CD3D0` | Computes and rounds an authored item price. |
| Content-price resolver | `0x009CD7C0`, `sub_9CD7C0` | Resolves rigblock and affixes before the generated-price calculation. |
| Normal resale price | `0x009CA630`, `sub_9CA630` | Converts authored price to float, multiplies by `0.5`, converts back to integer. |
| Flair resale/buyback price | `0x004B82D0` | Machine code is `mov eax,5; ret`. The IDA name `_RTC_NumErrors` is wrong. |
| Account parser | `0x004AB910`, `sub_4AB910` | Parses `unlock_stats` to account offset `+124`. |
| Update-creature request | `0x004A9230`, `sub_4A9230` | Serializes `cost`, recomputed `gear`, recomputed `points`, stats, and parts. |
| Equipped loot calculation | `0x004CE510`, `sub_4CE510` | Recomputes gear and item points from equipped non-flair loot. |
| Per-loot point lookup | `0x004CE370`, `sub_4CE370` | Returns the equipped loot record's authored point input. |
| Gear/item-point reduction | `0x009CB550`, `sub_9CB550` | Reduces up to six equipped point inputs to item points and gear. |
| Editor default budget load | `0x0071F170`, `sub_71F170` | Loads property `0xB1B76A19` into editor budget category 0. |
| External budget override | `0x00724E70`, `sub_724E70` | Writes launch-message field `+8` to editor budget category 0. |
| Existing model-part debit | `0x00724AC0`, `sub_724AC0` | Debits category 0 by each model part's `sub_6A22E0` cost. |
| Editor/account initialization | `0x004B9FE0`, `sub_4B9FE0` | Initializes budget categories and writes account DNA to category 8. |
| Update event construction | `0x004BA3E0`, `sub_4BA3E0` | Initializes the 80-byte editor-save message, including `+64 = 0`. |
| Update dispatch | `0x004B7C20`, `sub_4B7C20` | Reads global `unlock_stats` and constructs `api.creature.updateCreature`. |

The public [Spore property
registry](https://modthesims.info/wiki.php?title=Spore%3A00B1B104) maps
property hash `0xB1B76A19` to `editorDefaultInitialBudget`. That human-readable
name comes from the registry, not the decompiler. Build-103 control flow is
the authoritative fact needed here: the property is only the fallback when
the editor has no externally supplied budget. `sub_724E70` replaces category
0 with the external value.

### Current server and retained data

- `server/game/api.go:235` dispatches `api.inventory.vendorParts`.
- `server/game/vendor.go` logs the raw transaction string and accepts only
  `s` and `b`.
- `server/sporenet/vendor.go` applies one ordered, atomic in-memory batch and
  persists once.
- `server/game/api.go:427` logs update-creature `gear`, `points`, and `parts`.
- `server/game/api_progression_test.go` contains a synthetic
  `transactions=s17;b17` example. It is not a retail capture.

The retained 2026-07-23 Darkspinner log contains a real build-103 request:

```text
2026-07-23T04:21:03.171729-07:00 Server: inventory_vendor_parts account="Test" transactions="f10;s9;b9;s9;b9;s8;s9;b8;b9"
```

This capture proves all of the following:

- one request may contain multiple semicolon-separated operations;
- different operation characters may be mixed;
- the same item may occur more than once;
- order matters (`s9;b9;s9;b9` is meaningful);
- `f`, `s`, and `b` were emitted by the unmodified client;
- item IDs are the persisted inventory IDs (`8`, `9`, and `10`).

No retained request contains `p`. Its encoding and behavior are
decompiler-proven, not capture-proven. The JSONL protocol traces do not
contain the HTTP form body or retired HTTP response.

The current server rejects the captured batch at `f10`, and the read-only
database snapshot after the request still has parts 8, 9, and 10 owned and
not flair. That confirms current darkspin's all-or-nothing failure behavior;
it does not prove retired-server atomicity.

## `api.inventory.vendorParts` wire contract

### Request encoding

`sub_4AA980` (`0x004AA980`) receives a vector with 16-byte elements:

| Element offset | Width | Meaning |
| ---: | ---: | --- |
| `+0` | 8 | Unsigned item ID, low and high dwords. |
| `+8` | 1 | Operation character. |
| `+9..+15` | 7 | Padding/unused by the serializer. |

Each element is rendered with:

```c
"%c%I64d"
```

Elements are joined with a literal semicolon and sent in the single form
field `transactions`. The method is:

```text
api.inventory.vendorParts
```

Equivalent form examples are:

```text
method=api.inventory.vendorParts&transactions=s17%3Bb17
method=api.inventory.vendorParts&transactions=f10%3Bs9%3Bb9%3Bs9%3Bb9%3Bs8%3Bs9%3Bb8%3Bb9
```

The request wrapper does not send when the vector is empty. The client
serializer never inserts whitespace, empty entries, signs, or an item ID
without an operation.

### Operation map

`sub_430A00` maps its internal queue enum exactly:

| Internal value | Wire | Client meaning | Operand |
| ---: | :---: | --- | --- |
| `0` | `b` | Buy back an inventory item from vendor state. | Inventory item ID. |
| `1` | `p` | Purchase a current vendor offer. | Offer's first qword item ID. |
| `2` | `s` | Sell an owned inventory item. | Inventory item ID. |
| `3` | `f` | Convert an inventory item to flair. | Inventory item ID. |

`sub_435820` queues all four in the same record shape. The `p` path receives
a vendor-offer vector index from the UI, but replaces it with the offer's
first qword before queueing. The wire operand is therefore still an item ID,
not an array index.

### Part wire fields and market status

The full part parser is `sub_4ACC60` (`0x004ACC60`). Relevant fields are:

| XML field | Parsed client field | Notes |
| --- | --- | --- |
| `id` | qword item ID | The vendor command operand. |
| `reference_id` | qword content/reference ID | Not the command operand. |
| `creature_id` | equipped hero ID | Zero means unequipped. |
| `market_status` | wire integer minus one | Wire statuses are one-based. |
| `cost` | response-provided integer | Parsed, but not used for normal `s`/`b` client pricing. |
| `is_flair` | boolean | Selects the constant-5 resale/buyback price. |
| `level`, `rarity`, rigblock/affixes | content identity | Feed generated price and item-point calculations. |

The numeric market graph proven in `sub_435820` is:

```text
sell:       internal 0 (wire 1) -> internal 2 (wire 3)
buyback:    internal 1 or 2 (wire 2 or 3) -> internal 0 (wire 1)
purchase:   returned/new owned part is expected in response/parts
flair:      market status unchanged; is_flair 0 -> 1
```

The successful vendor response reaches `sub_4C1C80` through `sub_4AE360` and the `194453249` event. That handler appends each returned part without replacing an existing item ID. Only newly allocated purchase IDs belong in `response/parts`; sale, buyback and flair already change the existing native entry optimistically. Returning those entries creates duplicates that can remain visible in the editor after sale. The server also rejects sold/offer parts in creature equipment updates; persistence in the buyback inventory does not mean the part is owned.

The names “owned,” “offer,” and “buyback” used by current Go code are useful
labels, but the numeric transitions above are the decompiler-proven
contract. In particular, build 103 accepts both internal statuses 1 and 2
for `b`; the server must not silently narrow this to only status 2 without
new evidence.

### Per-operation checks, price, and local behavior

#### `s` — sell

Decompiler-proven in `sub_435820`, case 2:

1. Resolve the inventory part by its qword item ID.
2. Require internal `market_status == 0`.
3. Require `creature_id == 0`; equipped parts cannot be sold.
4. Resolve authored content through `sub_4C1020` and `sub_9CD7C0`.
5. If `is_flair == 0`, use `sub_9CA630(generated_price)`.
6. If `is_flair == 1`, use constant `5` from `0x004B82D0`.
7. Optimistically set internal market status to 2.
8. Optimistically add the price to the displayed DNA balance.
9. Queue `s<item-id>`.

#### `b` — buyback

Decompiler-proven in `sub_435820`, case 0:

1. Resolve the inventory part by its qword item ID.
2. Require internal market status 1 or 2.
3. Resolve the price exactly as for `s`: half generated price for normal
   loot, constant 5 for flair.
4. Require inventory count below the account-provided inventory capacity at
   global account offset `+112` (`sub_4E4E20()+10888`).
5. Require displayed account DNA to be strictly greater than price. The
   client rejects equality (`currentDNA <= price` is the failure branch),
   even though the later subtraction is saturating.
6. Optimistically set internal market status to 0.
7. Optimistically subtract the price, saturating at zero.
8. Queue `b<item-id>`.

#### `p` — purchase current offer

Decompiler-proven in `sub_435820`, case 1:

1. Validate the UI's offer index against the current 128-byte offer vector.
2. Read price from offer offset `+112`.
3. Require account level (account offset `+40`) to be at least offer offset
   `+116`.
4. Require account `chain_progression` (account offset `+0`) to be at least
   offer offset `+120`.
5. Require displayed account DNA to be at least offer price. Unlike `b`,
   equality is accepted in this branch.
6. Replace the UI index with the offer's qword at offset `+0`.
7. Queue `p<offer-item-id>` and immediately flush the queue.

The `p` path displays the offer price but does not optimistically subtract
DNA or synthesize an inventory part. The response is authoritative for both
the acquired part and final DNA.

#### `f` — convert to flair

Decompiler-proven in `sub_435820`, case 3 and `sub_4354F0`:

1. Resolve the inventory part by qword item ID.
2. Require `is_flair == 0`.
3. Resolve the rigblock content and require `(content_flags & 0x70) == 0`.
4. Optimistically set `is_flair = 1`.
5. Display the full generated content price from `sub_9CD7C0`, not the
   half-price or constant-5 resale price.
6. Queue `f<item-id>`.

The client does not perform a local DNA-funds guard or local DNA subtraction
in this branch. The decompiler proves that the full price is presented as
the conversion price; treating it as the server charge is an implementation
inference reinforced by the `LABS_local_DNA_FLAIR_CONVERT` notification. The
server should validate and charge that price, returning authoritative DNA,
or reject the operation until a retail response proves a different policy.

Content recipe 51 now projects the raw byte at property payload offset 108.
The resulting catalog confirms flag `0x10` on Grasper parts, `0x20` on Foot
parts, and `0x40` on Weapon parts, so the recovered `0x70` mask excludes those
three authored slot families from flair conversion without inferred ID ranges.

### Price source and rounding

There are three distinct sources:

| Operation/item | Price source |
| --- | --- |
| `s`/`b`, normal part | `sub_9CD7C0` generated authored price, then `sub_9CA630` half-price conversion. |
| `s`/`b`, flair part | Constant 5 at `0x004B82D0`. |
| `p` | Current offer field at offset `+112`. |
| `f` | Full `sub_9CD7C0` generated authored price. |

`sub_9CD3D0` derives generated price from the resolved rigblock/affixes and
loot tuning. At `0x009CD3D0` it:

1. evaluates the configured exponential level curve;
2. multiplies by the configured base;
3. rounds to the configured increment;
4. clamps to at least one increment.

This is not the parsed `<part><cost>` value. Test's persisted parts currently
all have `cost=0`, so current `vendorPartPrice(part.Cost)` cannot reproduce
the client price.

The exact `sub_9CA630` machine sequence is:

```asm
cvtsi2ss input, xmm0
mulss     [0x00FD2898], xmm0 ; 0.5f
cvtss2si xmm0, eax
```

`cvtss2si` obeys the MXCSR rounding mode. Under the normal default
round-to-nearest-even mode, odd positive prices are not uniformly floored:
101 halves to 50, while 103 halves to 52. Current Go conversion to `uint32`
truncates and would produce 51 for 103. Tests must pin the client rounding
mode rather than describe the rule merely as “half” or “floor.”

### Success response

`sub_4AE360` unconditionally looks up:

```xml
<response>
  <parts>
    <part>...</part>
  </parts>
  <dna>...</dna>
</response>
```

The generic response parser also understands `stat`, `code`, and `result`;
those compatibility fields may be included. Vendor-specific required data
are the full `parts` container and final `dna`.

On success, event `194453249` is published. `sub_4C1990` then:

1. iterates every returned 80-byte parsed part;
2. upserts each part by item ID into the local inventory;
3. publishes event `184696550` with returned DNA.

The returned part entries therefore need the normal full part field set,
including `id`, `reference_id`, `creature_id`, one-based `market_status`,
`cost`, `is_flair`, level/rarity, rigblock/affixes, status/usage, and
creation date. At minimum, return the final state of every affected existing
part and every newly purchased part. Returning unique final states for a
repeated-ID batch is an implementation inference; no retained retail
response proves whether the retired server returned duplicates.

Current darkspin returns final `<dna>` but no `<parts>`, so it does not meet
this response contract.

### Failure response and client recovery

`sub_4AE520` reads:

```xml
<response><code>N</code></response>
```

and publishes event `194529010`. The numeric code is parsed but is not
branch-selected in this handler.

The client has already made optimistic `s`, `b`, and `f` changes by this
point. Its failure path restores the DNA shadow and publishes a refresh
event, but does not directly undo every mutated part status/flair byte in
the vendor handler. A subsequent inventory reload is therefore required for
complete reconciliation. The retained failed mixed request is followed by
`api.inventory.getPartList`, consistent with that recovery path.

Server failures should use HTTP 200 with a response-level failure code, as
the existing API does. They must not return a success envelope containing
partial unreported mutations.

### Are semicolon operations atomic?

What build 103 proves:

- operations are ordered;
- one request has one success/failure result;
- the response has one final DNA scalar and one aggregate parts list;
- there is no per-operation status or error field;
- repeated operations on one item occur in a real capture;
- the client applies local optimistic operations in order before sending.

What is not retained:

- a retail mixed batch in which a later operation fails;
- the retired server's response and durable state after that failure.

Therefore retired-server atomicity is **not proven**. The implementation-ready
fallback is:

1. validate and simulate all operations sequentially against one working
   account/inventory state;
2. on any failure, persist nothing and return one failure response;
3. on success, persist once and return final affected parts plus final DNA.

This matches the response shape and avoids a partial commit the client
cannot describe. It must remain labeled retired-policy until a retail
failure capture replaces it.

## Hero editor DNA budget

### Four values that must remain separate

| Value | Client source | Purpose |
| --- | --- | --- |
| Editor DNA budget, category 0 | External editor-launch field `+8`, with `editorDefaultInitialBudget` only as fallback | Pays for editor model/body parts and produces the displayed `used/budget` value. |
| Account DNA, category 8 | Account `dna` at global offset `+32` | Persistent currency and vendor/upgrade funds. |
| Creature `points` / persisted `item_point` | Recomputed from equipped non-flair loot by `sub_4CE510`/`sub_9CB550` | Combat item-point rating sent by `api.creature.updateCreature`. |
| Creature `gear` / persisted `gear_score` | Recomputed alongside item points | Combat gear score sent by `api.creature.updateCreature`. |

The names all contain “DNA,” “points,” or “cost” in parts of the UI and wire
protocol, but they are different budget-manager categories and calculations.

### Exact `20/0` dataflow

#### Denominator: 0

`sub_71F170` reads property ID `0xB1B76A19`
(`editorDefaultInitialBudget`) and writes it to budget-manager category 0.
That is only default initialization.

`sub_724E70` later performs:

```text
budgetManager.set(category=0, launchMessage.fieldAt(+8))
```

An externally supplied zero therefore overwrites the fallback and produces
a zero denominator. No read of creature `item_points`, `gear_score`, level,
or account `unlock_stats` occurs on this category-0 path.

The corrective server/client integration requirement is to locate and
populate the editor-launch budget field, not to rewrite `item_point` or
pretend `unlock_stats` is a capacity.

#### Numerator: 20

`sub_724AC0` walks the creature editor's existing model/body parts. For each
eligible part it calls:

```text
sub_6A22E0(modelPart)
```

and applies the negative result to budget-manager category 0. That aggregate
is the editor DNA already spent by the model. It is the proven source class
of the displayed 20.

The current HTTP logger records only the byte length of `large`; it does not
decode the embedded editor model-part records. Consequently the retained
capture proves the aggregate behavior but cannot attribute the 20 to exact
geometry asset names. A future diagnostic can decode `large` read-only and
list each `sub_6A22E0`-equivalent part cost, but that is not required to fix
the zero capacity.

### Why `item_point=300` did not help

The creature list parser accepts stored `item_points`, and darkspin persists
it, but `sub_4A9230` does not reuse that field when saving the editor result.
It calls `sub_4CE510`, which:

1. walks equipped editor loot slots in 480-byte records;
2. resolves each inventory item by qword ID;
3. skips flair items;
4. obtains the part's authored point input through `sub_4CE370`;
5. passes up to six positive inputs to `sub_9CB550`;
6. serializes the resulting independent `gear` and `points` fields.

`sub_9CB550` transforms each equipped input through the authored loot curve,
adds `50 * transformed_input`, and derives gear from the total. This is a
combat-rating calculation, not editor body-part currency.

Recent capture evidence makes the separation concrete:

```text
2026-07-23T04:21:28 ... creature_id=4 gear="1.000" points="302.500" parts="9"
2026-07-23T04:22:29 ... creature_id=1 gear="1.000" points="305.000" parts="4,5"
```

The read-only database snapshot now contains:

| Hero | Equipped inventory IDs | Persisted item point | Gear score |
| --- | --- | ---: | ---: |
| Blitz Alpha, creature 1 | 4, 5 | 305 | 1 |
| Sage Alpha, creature 2 | none | 300 | 1 |
| Wraith Alpha, creature 3 | none | 300 | 1 |
| Goliath Alpha, creature 4 | 9 | 302.5 | 1 |

The observed loot contribution is therefore 2.5 item points per newly
equipped captured part in these examples. It still has no effect on the
editor category-0 denominator.

### Account `unlock_stats` and genetic upgrades

`sub_4AB910` parses `<unlock_stats>` to account offset `+124`, stored globally
at `sub_4E4E20()+10900`.

Every relevant build-103 use found in this path passes it to
`sub_9E3740`, directly or through `sub_4CE780`/`sub_4CEB10`, together with
the hero template's stat-table field at content offset `+164`. Its job is to
select/seed the hero's authored stat tier before stats and ability keyvalues
are serialized.

`sub_4B7C20` also reads `unlock_stats` immediately before constructing
`api.creature.updateCreature`, but `sub_4A9230` uses it for the stats
serialization path, not for `cost`, `gear`, `points`, or editor category 0.

Test currently has:

```text
level=4
dna=176
unlock_stat=0
unlock_inventory_identify=180
```

Current darkspin's legacy/retired policy maps upgrade IDs 8 through 25 to
`UnlockStats = unlockID - 7`, with costs 200 through 250,000 DNA. That mapping
is useful compatibility data, and the level-4 profile event calls a hero
genetic upgrade available. It is **not decompiler-proven build-103 editor
budget policy**. Purchasing such an upgrade should advance the stat tier
only unless new client evidence shows a separate editor-budget grant.

### Hero template, class, level, and gear

- The template's content record supplies the stat table at offset `+164`;
  `unlock_stats` selects within it. This is where hero class/template matters
  to the traced update path.
- The template/class and genetic type also constrain compatible loot and
  authored stats, but no traced reference turns those fields into editor
  category-0 capacity.
- Inventory-part level, rigblock, and affixes influence the generated loot
  price and equipped item-point input.
- Creature level and persisted gear score are not inputs to
  `editorDefaultInitialBudget` or the external category-0 override.
- Persisted `item_point` and `gear_score` are response/cache outputs. The
  client recomputes them when saving.

### Test persisted inventory facts

At the snapshot, Test has ten parts. All have persisted `cost=0`,
`market_status=0`, and `is_flair=0`. Parts 4 and 5 are equipped to Blitz;
part 9 is equipped to Goliath; the others are unequipped.

This proves two current implementation hazards:

1. vendor price cannot be taken from persisted `part.cost` for this account;
2. sell validation must use the current equipped relationship, so parts 4,
   5, and 9 must be rejected by `s` until unequipped.

## Implemented contract

### Vendor operation engine

The feature operation consumes:

```text
actor
ordered []{ operation byte, itemID uint64 }
current vendor offers
content price resolver
inventory-capacity policy
```

It produces:

```text
final account DNA
unique final affected inventory parts
newly purchased parts
```

Sequential simulation rules:

1. Resolve every `s`, `b`, and `f` by owned-account item ID.
2. Resolve `p` by the current offer item ID, never by client-provided index.
3. Apply the exact numeric market-status and equipped/flair guards above.
4. Resolve normal and conversion prices from content identity and loot
   tuning; do not trust the client or persisted XML `cost`.
5. Apply `p` level, chain-progression, DNA, and offer-validity checks.
6. Apply `b`'s strict `DNA > price` guard and inventory-capacity check.
7. Charge `f` its full generated price.
8. Preserve request order, including repeated IDs.
9. Persist once only after the whole simulated batch is valid
   (retired-policy fallback).
10. Return the final full parts plus final DNA.

`p` allocates a new authoritative owned inventory ID while retaining the
offer ID only as the current vendor-offer identity. Whether retail reused the
offer ID is not proven by a retained purchase response, so this remains an
explicit correction-friendly server policy.

### Editor launch

Do not derive the category-0 launch budget from:

- account DNA;
- `unlock_stats`;
- creature `item_points`;
- creature `gear_score`;
- equipped loot point totals.

The editor-launch adapter must provide the intended hero editor budget
explicitly. If no retired tuning source can be recovered, a conservative
playable fallback should be selected as server policy and recorded
separately from this decompiler contract. The client already has
`editorDefaultInitialBudget`; passing zero defeats that fallback.

## Deferred focused Go test plan

Repository prototype policy currently reserves automated test changes and
execution for an explicit user request. The cases below remain the focused
future coverage plan; production builds and content verification are the
current acceptance checks.

### HTTP parsing and response

1. `TestParseVendorTransactionsAcceptsAllBuild103Operations`:
   `b1;p2;s3;f4`, preserving order and uint64 IDs.
2. `TestParseVendorTransactionsPreservesRepeatedItemOrder`:
   the captured `f10;s9;b9;s9;b9;s8;s9;b8;b9`.
3. `TestParseVendorTransactionsRejectsMalformedFields`:
   empty entries, whitespace inside an entry, signs, zero IDs, overflow,
   missing operation, and unknown characters.
4. `TestVendorPartsSuccessReturnsFullAffectedPartsAndDNA`:
   assert `<parts><part>...` and `<dna>`, including one-based market status.
5. `TestVendorPartsFailureReturnsCodeAndNoSuccessMutation`:
   assert response-level failure with no partial success envelope.
6. `TestVendorPartsLogsExactCapturedTransactionString`:
   retain the real mixed request as the regression fixture.

### Pricing

7. `TestVendorNormalSellAndBuybackUseGeneratedContentPrice`:
   persisted `Part.Cost=0`, nonzero price resolved from rigblock/affixes/level.
8. `TestVendorHalfPriceUsesBuild103Rounding`:
   at least 100, 101, 102, and 103; specifically pin 101 -> 50 and
   103 -> 52 under default nearest-even.
9. `TestVendorFlairSellAndBuybackCostFive`:
   both directions use 5 regardless of generated normal price.
10. `TestVendorFlairConversionChargesFullGeneratedPrice`:
    not half-price and not 5.
11. `TestVendorPurchaseUsesOfferPrice`:
    ignore client/persisted part cost and charge offer offset-equivalent
    price.

### Guards and transitions

12. `TestVendorSellOwnedUnequippedTransitionsWireOneToWireThree`.
13. `TestVendorSellRejectsEquippedPartWithoutWrite`.
14. `TestVendorBuybackAcceptsInternalStatusOneAndTwo`.
15. `TestVendorBuybackRejectsCapacityAndFundsWithoutWrite`.
16. `TestVendorPurchaseEnforcesLevelChainFundsAndCurrentOffer`.
17. `TestVendorPurchaseReturnsOwnedInventoryInstance`.
18. `TestVendorFlairConversionRejectsExistingFlairAndContentFlagMask`.
19. `TestVendorBatchAppliesRepeatedItemSequentially`.
20. `TestVendorBatchRollsBackEarlierOperationsOnLaterFailure`.
21. `TestVendorBatchRollsBackOnPersistenceFailure`.

Tests 20 and 21 pin the selected retired-policy fallback, not a
decompiler-proven retail transaction boundary.

### Editor budget separation

22. `TestEditorLaunchBudgetDoesNotUseCreatureItemPoint`:
    `item_point=300` with launch budget 0 remains category-0 budget 0.
23. `TestEditorLaunchBudgetDoesNotUseUnlockStats`:
    changing `unlock_stats` changes stat-tier input but not editor capacity.
24. `TestEditorLaunchBudgetDoesNotUseAccountDNA`:
    account DNA remains the separate currency/category-8 value.
25. `TestEditorLaunchUsesExplicitBudgetOrNonzeroFallback`:
    prevent an accidental zero from overriding the selected fallback.
26. `TestCreatureUpdateRecomputesPointsFromEquippedNonFlairParts`:
    reproduce captured 300, 302.5, and 305 cases.
27. `TestCreatureUpdateExcludesFlairFromItemPoints`.
28. `TestCreatureUpdateRecomputesGearInsteadOfTrustingPersistedGearScore`.
29. `TestUnlockStatsSelectsTemplateStatTierOnly`:
    template/class stat fields change while editor budget, gear, and points
    stay unchanged.

## Unresolved retail evidence

The following cannot be promoted beyond inference without another capture
or content decode:

- retired-server partial-versus-atomic failure behavior;
- exact retired numeric failure-code taxonomy;
- whether successful repeated-ID batches returned duplicate part entries or
  one final entry per item;
- whether a `p` response reused the offer item ID or allocated a new
  inventory instance ID;
- the particular editor geometry assets whose costs sum to the observed 20;
- whether the retired service used a different nonzero category-0 budget than
  Darkspinner's user-authoritative value of 100.

None of these uncertainties justifies using `item_point=300` or
`unlock_stats=0` as the editor budget. The client dataflow disproves both
substitutions.

## Follow-up package scan

An extracted-resource scan on 2026-07-23 found no literal little-endian
`0xB1B76A19` property key in `Editors.package`,
`AssetData_Binary.package`, `Config_Pack.package`, `Config_Ship.package`,
the three RenderingConfig packages, `Other.package`, or
`ServerData.package`. Diagnostics are under
`bin/game/logs/editor-budget/`. The follow-up targeted analysis recovered
resource `0xF86BD1D6` as the actual default Editors property list and proved
that the missing property selects the loader-zeroed fallback. See
`notes/items/editor-budget.md`; the implementation patches only that verified
resource.

## Editor application callback ownership

The canonical IDB narrows that remaining producer boundary:

- `sub_4BCFE0` constructs the 10,464-byte `SP_App` object and
  `sub_4BDB00` registers it as application `"Editors"`;
- derived `SP_App` vtable `0x00FD98D0` slot 55 is `sub_4B9FA0`;
- base editor-interface vtable `0x00FFA184` slot 68 is `sub_724E70`;
- derived slot 69 is `sub_4B9FE0`, while base slot 82 is
  `sub_71F170`.

The two derived/base pairs prove that the launch-budget write and default
initialization are editor-application interface callbacks, not an inventory
HTTP response or RakNet application opcode. IDA finds no ordinary direct call
to either callback: the application/interface dispatcher owns invocation.
The completed callback trace proves the first history snapshot captures
category 0 after property initialization. Darkspinner therefore supplies the
budget through the verified packaged-data source rather than a gameplay
packet, account DNA, or Fang interception. The slot inventory and xref
diagnostics are retained under `bin/game/logs/editor-budget/`.
