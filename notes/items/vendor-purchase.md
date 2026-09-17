# Build-103 vendor offer generation and purchase contract

Date: 2026-07-23

Client: `bin/game/GameBin/Game.exe`, build `5.3.0.103`

Canonical decompilation: `bin/game/GameBin/Game.c`

Canonical IDB: `bin/game/GameBin/Game.idb`

## Result

The premise that build 103 obtains vendor offers from
`api.inventory.getPartOfferList` is false. The executable contains no such
method string, request constructor, or offer-response decoder. The method
exists only in current darkspin. Build 103 constructs its 128-byte offer
records locally from packaged authored data:

```text
packaged 32-byte authored row
  -> sub_5451C0
  -> sub_545AF0
  -> 128-byte offer vector at sub_4E4E20()+10760
  -> sub_435820 case 1
  -> p<offer-id>
```

Consequently there are **no build-103 XML element names** for offer identity,
authoritative price, minimum account level, or minimum chain progression.
The similarly named generic part elements `id`, `cost`, and `level` belong to
the 80-byte owned-inventory part decoder and do not populate the offer record.
In particular, `<part><level>` is the generated part's item level; it is not
the offer's minimum account level.

The four offer fields are nevertheless exact and implementation-ready:

| Offer offset | Width | Authored-row source | Meaning |
| ---: | ---: | ---: | --- |
| `+0` | 8 | row dwords `[0..1]` | Authored offer identity; the operand in `p<offer-id>`. |
| `+112` | 4 | row dword `[5]` | Authoritative offer purchase price. |
| `+116` | 4 | row dword `[6]` | Minimum account/Crogenitor level. |
| `+120` | 4 | row dword `[7]` | Minimum account `chain_progression`. |

## Packaged source identification

The registry constructor at `0x00EBA4A0` initializes `unk_143E248` with the exact asset name `WeaponTuning.WeaponTuning`. Its case-insensitive FNV-1 instance hash is `0x8324B40E`, which resolves in `AssetData_Binary.package` to ordinal `12696`, type `0x58E9A177`, group `0`, decoded size `12,200`, and SHA-256 `209805468d39d97df656e64c753afa6a0a4a3a7dd5471d82d515b7bb69549ffb`.

The decoded payload is an 8-byte header followed by exactly 381 contiguous 32-byte rows, matching the native consumer without interpretation gaps. Offer IDs span `1..389` with the eight absent identities `323, 335, 336, 346, 355, 356, 378, 382`; item levels span `16..176`, rigblocks `263..335`, suffixes `5..80`, prices `150..160000`, and minimum account levels `6..60`. All retained build-103 chain-progression gates are authored as zero.

Content recipe 50 pins this source identity and digest, validates unique nonzero IDs and field bounds, stores all rows in `weapon_tuning`, and projects them into the runtime vendor catalog. Purchases therefore resolve the same qword identity and authority fields consumed by the client.

The purchase success response does not reveal whether the retired server
reused the authored offer ID as the owned inventory ID. It returns ordinary
80-byte `<part>` records and the client upserts each by the returned `<id>`
without comparing it to the queued offer ID. A new owned ID and a reused ID
are therefore both accepted by the client. The conservative implementation
fallback is to keep offer identity and owned inventory identity separate,
allocate a fresh nonzero owned inventory ID, and return that new ID in the
successful response.

## Confidence labels

- **Decompiler-proven**: directly visible in build-103 control flow or data
  movement.
- **Binary-proven**: confirmed by the executable's embedded strings.
- **Content-path-proven**: confirmed by the native packaged-content producer
  path, though the original DBPF resource name is not retained.
- **Current-state**: observation of the present Go implementation.
- **Fallback**: conservative policy selected because the retired server's
  choice is not recoverable from the retained client.

## Exact client path

### Local offer producer

| Address | Symbol | Evidence |
| ---: | --- | --- |
| `0x005467F0` | `sub_5467F0` | Clears and rebuilds the vector at global-state offsets `+10760..+10768`, obtains the packaged authored table through `sub_7A29C0(&unk_143E248)`, emits one 128-byte record per selected 32-byte row, and sorts the result. |
| `0x005451C0` | `sub_5451C0` | Resolves rigblock/suffix content and copies authored row `[0..1]`, `[5]`, `[6]`, and `[7]` to offer dwords `[0..1]`, `[28]`, `[29]`, and `[30]`. |
| `0x00545AF0` | `sub_545AF0` | Calls `sub_5451C0`, finishes the generated part/display fields, and leaves the offer gates at `+112`, `+116`, and `+120` intact. |
| `0x005466F0` | `sub_5466F0` | Appends a complete 128-byte record to the offer vector. |
| `0x005453F0` | `sub_5453F0` | Sort comparator that separately tests offer `+116` against account level and offer `+120` against chain progression. |
| `0x00545170` | `sub_545170` | Computes offer lock color from `+116`, `+120`, and price `+112`. |
| `0x005460E0` | `sub_5460E0` | Produces locked-offer UI text from minimum account level `+116`, minimum chain progression `+120`, and price `+112`. |
| `0x005463A0` | `sub_5463A0` | Publishes the local weapon-buy telemetry, reading display data at `+36` and DNA spent from `+112`; it does not fetch or decode an offer. |

The decisive assignments in `sub_5451C0` are:

```c
*a2     = *v8;    // offer +0  <- authored row +0
a2[1]  = v8[1];  // offer +4  <- authored row +4
a2[28] = v8[5];  // offer +112 <- authored row +20
a2[29] = v8[6];  // offer +116 <- authored row +24
a2[30] = v8[7];  // offer +120 <- authored row +28
```

`sub_5467F0` also uses authored row `[6]` and `[7]` while deciding which
rows are relevant to the current account. This independently corroborates
their level/progression gate roles before a record reaches the purchase UI.

The offer ID is not a rigblock ID, generated part reference ID, or UI index.
It is the authored row's first qword. The record also contains generated
content/display fields, but none replaces that qword as the purchase operand.

### Purchase guard and request

| Address | Symbol | Evidence |
| ---: | --- | --- |
| `0x00435820` | `sub_435820`, case `1` | Bounds-checks the UI offer index, reads the four contract fields, changes the operand from vector index to the qword at `+0`, queues operation `1`, and immediately flushes. |
| `0x00430A00` | `sub_430A00` | Maps internal operation `1` to wire character `p`. |
| `0x004AA980` | `sub_4AA980` | Serializes the queued qword as `%c%I64d` in `transactions` for `api.inventory.vendorParts`. |

The exact purchase conditions in `sub_435820` are:

```text
account.level             >= offer.minimumAccountLevel
account.chainProgression  >= offer.minimumChainProgression
displayedDNA              >= offer.price
```

Account level is read from global account base `sub_4E4E20()+10776`, dword
`[10]` (absolute global-state offset `+10816`). Chain progression is dword
`[0]` at `+10776`. Displayed DNA is vendor UI field `this[9]`. Equality is
accepted for all three comparisons. On success:

```text
transactions=p<decimal unsigned offer qword>
```

Unlike sell, buyback, and flair conversion, purchase does not optimistically
create a part, change a part market status, or subtract DNA. The returned part
and DNA are authoritative.

### Successful purchase response

| Address | Symbol | Evidence |
| ---: | --- | --- |
| `0x004AE360` | `sub_4AE360` | Parses `response/parts` through the generic part parser and parses `response/dna`. |
| `0x004ACC60` | `sub_4ACC60` | Parses each returned 80-byte inventory part, including XML elements `id`, `reference_id`, `cost`, `level`, and `market_status`. |
| `0x004C1990` | `sub_4C1990` | For event `194453249`, sends every returned part to the inventory upsert path using the returned part ID, then publishes returned DNA. |

The required success envelope is:

```xml
<response>
  <parts>
    <part>
      <id>NEW_OR_REUSED_OWNED_INVENTORY_ID</id>
      <reference_id>...</reference_id>
      <is_flair>0</is_flair>
      <cost>...</cost>
      <creature_id>0</creature_id>
      <level>...</level>
      <market_status>1</market_status>
      <rigblock_asset_id>...</rigblock_asset_id>
      <prefix_asset_id>...</prefix_asset_id>
      <prefix_secondary_asset_id>...</prefix_secondary_asset_id>
      <rarity>...</rarity>
      <suffix_asset_id>...</suffix_asset_id>
      <status>...</status>
      <usage>...</usage>
      <creation_date>...</creation_date>
    </part>
  </parts>
  <dna>FINAL_AUTHORITATIVE_DNA</dna>
</response>
```

`market_status` and `rarity` are one-based on the wire. The XML example shows
the existing generic part contract, not offer-list XML. `sub_4C1990` receives
no queued offer ID and performs no equality check between the offer ID and
returned `<id>`. The response path therefore cannot prove ID reuse.

## Why `api.inventory.getPartOfferList` is not build-103 evidence

Binary strings and constructors provide a useful negative cross-check:

- `api.inventory.getPartList` is embedded at VA `0x00FD8930`; request
  constructor `sub_4A8410` (`0x004A8410`) uses it.
- `api.inventory.updatePartStatus` is embedded at VA `0x00FD8D3C`.
- `api.inventory.vendorParts` is embedded at VA `0x00FD8E08`; constructor
  `sub_4AA980` uses it.
- `api.inventory.getPartOfferList` does not occur in the build-103 executable.
- The extracted `bin/game/Data/Web.package` resources contain no
  `getPartOfferList`, `vendorParts`, or part-offer producer.
- Current darkspin retains `api.inventory.getPartOfferList` only as an empty
  compatibility endpoint because build 103 consumes the packaged catalog.

The build-103 packaged producer is native content loading, not packaged HTML
or JavaScript. `sub_5467F0` loads the authored table represented by
`unk_143E248`, generates the offer records, and sorts them. Although DBPF
indexes do not preserve original filesystem paths, the registry constructor
recovers the trustworthy asset identity `WeaponTuning.WeaponTuning`, and the
decoded resource exactly matches the native 32-byte row layout.

The only relevant inventory web decoder is:

```text
sub_4A8410 api.inventory.getPartList
  -> sub_4AD680 response/parts
  -> sub_4ACC60 generic 80-byte inventory part
```

It never writes the 128-byte vector at global-state offset `+10760`. Reusing
its XML names for offers would invent a server contract the retail client did
not consume.

## Current Go cross-check

### `sporenet.Part`

Current `server/sporenet/part.go` has:

```text
ID, ReferenceID, IsFlair, Cost, EquippedToCreatureID, Level,
MarketStatus, Rarity, Status, Usage, CreationDate, and content asset IDs/hashes
```

This is an owned/generated part model. It has no distinct:

```text
OfferID
Price
MinimumAccountLevel
MinimumChainProgression
```

`Part.ID`, `Part.Cost`, and `Part.Level` happen to have names resembling three
offer concepts, but their generic `<part>` meanings are inventory instance
ID, part cost, and item level. They are not evidence for the authored offer
row or its gates.

### Removed placeholder offer output

The former `Vendor.Refresh` created 50 recap-era placeholders with:

```text
ID             = 0
ReferenceID    = 0
Cost           = 0x1234
Level          = 30
Status         = 0x62
Usage          = 0x55
MarketStatus   = 0xAB
```

`partNode` emitted these as generic parts. In particular:

- all offers serialize as `<id>0</id>`, so no valid `p<offer-id>` can be
  formed and `parseVendorTransactions` correctly rejects `p0`;
- `<cost>4660</cost>` is a placeholder, not authored offer `+112`;
- `<level>30</level>` is decoded by build 103 as item level, not offer
  minimum account level `+116`;
- market status becomes one-based `<market_status>172</market_status>`;
- status `98` and usage `85` are generic part fields and have no role in the
  offer purchase guard;
- there is no output at all for minimum chain progression `+120`.

These records have been removed. Build 103 does not request the compatibility
offer-list endpoint, which now returns an empty list instead of claiming
authority from fabricated records. These constants must not be copied into a
purchase implementation or treated as defaults.

The HTTP adapter parses `p` as a purchase operation and resolves its operand
against the imported vendor catalog before any owned-part lookup.
`ApplyVendorTransactions` enforces both authored gates and exact price,
allocates a distinct owned part ID, debits DNA, and rolls back the complete
batch if validation or persistence fails.

## Implementation-ready server model

Use a dedicated feature-owned offer type rather than persisting offers as
owned `sporenet.Part` rows:

```go
type PartOffer struct {
	ID                      uint64
	Price                   uint32
	MinimumAccountLevel     uint32
	MinimumChainProgression uint32
	Part                    Part
}
```

The names above are server-domain names, **not recovered XML names**. `Part`
is the content template to clone into owned inventory; its `ID` must not be
used as the offer ID. The offer catalog owns rotation/lifetime and lookup by
`PartOffer.ID`.

For `p<offer-id>`:

1. Resolve the exact nonzero offer ID in the current catalog. Reject unknown,
   expired, zero, or ambiguous IDs.
2. Read price and both gates only from the resolved offer. Do not accept them
   from the request, `sporenet.Part.Cost`, or `sporenet.Part.Level`.
3. Require `Account.Level >= MinimumAccountLevel`.
4. Require
   `Account.ChainProgression >= MinimumChainProgression`.
5. Require `Account.DNA >= Price`; equality succeeds.
6. Validate inventory capacity server-side even though case 1 has no local
   client guard. This is a conservative authority check, not a recovered
   client condition.
7. Clone the offer's part template, allocate a fresh nonzero owned inventory
   ID, set `EquippedToCreatureID=0` and internal market status to owned
   (`0`, wire `1`), and debit the exact offer price.
8. Persist account and part atomically. On failure restore both.
9. Return the new full `<part>` and final DNA in the normal vendor success
   envelope.

Do not serialize the four offer-authority fields back as generic part fields.
The retail client does not need them over HTTP because it already has the
packaged offer vector. If current darkspin keeps `getPartOfferList` as an
operator/compatibility extension, its response should be explicitly treated
as a darkspin extension and should not be cited as build-103 XML.

## Conservative fallbacks and unresolved evidence

| Question | Evidence | Fallback |
| --- | --- | --- |
| XML names for the four offer fields | No offer-list request/decoder exists in build 103. | Define typed server-domain fields; do not invent retail XML names. |
| Price authority | Client uses packaged offer `+112`, not generic part `cost`. | Catalog price is authoritative; reject a catalog entry whose price is unavailable rather than use `0x1234` or part cost. |
| Level/progression gates | Exact package offsets and all 381 authored rows are imported; every retained chain-progression field is zero. | Preserve explicit gates on each server offer so later content can remain authoritative without changing the transaction contract. |
| Successful owned ID | Response accepts either reuse or allocation; retired policy is unrecoverable. | Allocate a new owned inventory ID and keep authored offer ID as catalog identity. |
| Offer consumption/stock | Client proves neither global stock nor one-purchase removal. | Treat offers as reusable catalog entries unless future retail evidence proves stock semantics. |
| Batch atomicity | One aggregate success/failure response; retired failure boundary not retained. | Preserve the existing ordered atomic-batch policy. |

Fresh owned-ID allocation is safer than reuse because authored offer IDs and
per-account inventory IDs have different lifetimes and may collide. This is
an implementation fallback, not a claim about the retired server.

## Focused Go tests

### Offer projection and identity

1. `TestPartOffersAssignNonzeroUniqueOfferIDs`: every active offer has a
   stable nonzero catalog ID; no `<id>0</id>` is emitted by any compatibility
   projection.
2. `TestPartOfferSeparatesItemLevelFromMinimumAccountLevel`: changing the
   part template's `Level` does not change `MinimumAccountLevel`, and changing
   the gate does not change returned owned `<level>`.
3. `TestPartOfferSeparatesPartCostFromPurchasePrice`: purchase debits
   `PartOffer.Price` even when template `Part.Cost` differs.
4. `TestPartOfferSeparatesOfferIDFromOwnedPartID`: the request uses offer ID,
   while the success `<part><id>` contains a newly allocated owned ID.
5. `TestPartOfferCompatibilityXMLDoesNotUsePlaceholderConstants`: forbid
   zero IDs and the recap tuple `0x1234/30/0x62/0x55/0xAB`.

### Purchase validation

6. `TestVendorPurchaseRejectsUnknownOrExpiredOfferWithoutWrite`.
7. `TestVendorPurchaseRejectsAccountBelowMinimumLevelWithoutWrite`.
8. `TestVendorPurchaseAcceptsAccountAtMinimumLevel`.
9. `TestVendorPurchaseRejectsChainBelowMinimumProgressionWithoutWrite`.
10. `TestVendorPurchaseAcceptsChainAtMinimumProgression`.
11. `TestVendorPurchaseRejectsDNAOneBelowPriceWithoutWrite`.
12. `TestVendorPurchaseAcceptsDNAEqualToPrice`: pin the case-1 equality rule
    and final DNA zero.
13. `TestVendorPurchaseRejectsFullInventoryWithoutWrite`: pins the selected
    server-authority fallback rather than a client-side case-1 check.
14. `TestVendorPurchaseDoesNotTrustTemplateCostOrLevelAsOfferAuthority`.

### Mutation, response, and rollback

15. `TestVendorPurchaseAllocatesOwnedPartAndReturnsFinalDNA`: assert fresh
    ID, owned status, unequipped state, copied content identity, exact debit,
    and one persistence write.
16. `TestVendorPurchaseResponseContainsFullPartAndOneBasedOwnedStatus`: assert
    `response/parts/part`, all generic part fields needed by `sub_4ACC60`, and
    `<market_status>1</market_status>`.
17. `TestVendorPurchaseDoesNotPersistOfferAsOwnedRow`: catalog state and user
    inventory remain separate.
18. `TestVendorPurchaseRollsBackPartAndDNAOnPersistenceFailure`.
19. `TestVendorPurchaseBatchRollsBackEarlierOperationOnLaterFailure`.
20. `TestVendorPurchaseRepeatedOfferUsesDistinctOwnedIDs`: under the reusable
    catalog fallback, two successful purchases allocate two inventory IDs.

Tests 13, 19, and 20 pin conservative darkspin policies, not recovered
retired-server behavior. Test 4 pins the selected ID-allocation fallback and
should be revised if a future retail success capture proves ID reuse.

## Evidence boundary

What is proven:

- the four exact 128-byte offsets and their 32-byte authored-row sources;
- their meanings and comparison direction;
- purchase equality behavior;
- `p` carries the authored offer qword;
- successful purchase returns generic parts plus final DNA;
- returned parts are accepted by their returned IDs;
- build 103 does not request or decode
  `api.inventory.getPartOfferList`.

What remains unavailable:

- the retired server's owned-ID reuse/allocation choice;
- original authored filesystem/schema names for the package table;
- retired stock/offer-consumption policy;
- a retained retail `p` request and success response.

None of these unknowns permits treating zero item IDs, cost `0x1234`, level
30, status `0x62`, usage `0x55`, or market status `0xAB` as build-103 offer
authority.
