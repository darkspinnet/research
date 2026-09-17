# Build-103 persistent inventory load contract

## Conclusion

Existing parts are loaded correctly from `saves/darkspin.db` into
`sporenet.User.Parts`, but the ordinary inventory reload response is not the
build-103 XML contract. The principal defects are in `server/game/api.go`:

1. `partNode` omits `is_flair`, `creature_id`, and `market_status`, although the
   build-103 parser reads all three.
2. The client treats XML `market_status` and `rarity` as **one-based** and
   decrements both. Darkspin emits the stored zero-based rarity directly and
   emits no market status. An owned/basic database row (`0`/`0`) therefore
   becomes `-1`/`-1` in the client instead of owned/basic (`0`/`0`). This is the
   strongest explanation for persisted rows being admitted poorly or excluded
   from the Owned inventory view.
3. The handler ignores the confirmed `count=10000` and `filter` contract. This
   is not what hides the current rows—the unfiltered response is broader—but it
   makes the implementation semantically wrong.
4. The Hero Profile does not use the ordinary inventory list. It calls
   `api.creature.getCreature` and expects equipped parts nested in
   `<creature><parts>`. Darkspin's creature DTO emits no parts and lacks the
   content-enriched fields required by that screen, so its item/detail slots
   are blank independently of the ordinary inventory defect.

Persistence is not the primary gap. A real `/summon` row is present in SQLite,
is restored without filtering, and has valid `Part.ID`, `ReferenceID`, account
ownership, level, capacity, and dual asset identities. Live logs also prove the
same row was presented once through RakNet. That live presentation is an
acquisition notification, not the reload mechanism.

No build or client launch was performed. No Go, C, or test file was modified.
No material under `C:\src\recap` was inspected.

## Evidence labels and limits

- **Native-proven** means directly visible in build-103
  `bin/darkspinner/GameBin/Game.c` or the bundled Web package.
- **Current-source-proven** means directly visible in the current darkspin Go
  implementation.
- **Database evidence** means returned read-only by `darkrun db` from
  `bin/darkspinner/darkspin/saves/darkspin.db`.
- **Content.db evidence** means returned read-only by `darkrun db` from
  `bin/darkspinner/darkspin/cache/content.db`.
- **Live-log evidence** means already present in
  `bin/darkspinner/darkspin/logs/darkspinner.log` or its socket trace.
- **Inference** connects those facts and is called out explicitly.

The existing socket trace records connect/send/receive metadata, byte counts,
and digests, but not decoded HTTP parameters or XML. The runtime log likewise
does not log HTTP methods or bodies. Consequently, action-to-request mapping is
native call-graph evidence, not a live HTTP capture. No new capture was made
because the task forbids launching binaries.

## Client requests by transition

The list below is the complete inventory/account request set attributable to
the named UI transitions in the recovered native paths. Periodic status,
telemetry, authentication handshakes, and unrelated social calls are not caused
by opening or switching inventory UI regions and are not relabeled as such.

### Entering the ship/editor state

**Native-proven.** `cEditorState` initialization at `Game.c:196181-196209`
calls the inventory submit wrapper `sub_4B7100(0)`. It constructs:

```text
GET /game/api?version=<protocol version>
method=api.inventory.getPartList
count=10000
filter=market_status_full-owned;
token=<authenticated token>              # common request serialization
```

The constructor is `sub_4A8410` at `Game.c:268064-268144`; the wrapper is
`sub_4B7100` at `Game.c:279020-279078`. Argument zero causes the owned
filter to be appended. The parser callback is `sub_4AD680` at
`Game.c:271878-271947`.

On first login, the ship is preceded by, rather than caused by, the account
bootstrap request:

```text
method=api.account.auth
key=<token/key form>
cookie=true
build=5.3.0.103
include_creatures=true       # conditional
include_decks=true           # conditional
include_feed=true            # conditional
include_settings=true        # conditional
```

The account XML supplies `unlock_inventory` and
`unlock_inventory_identify`. It does not replace the separate part-list load.
For a new-player state transition, room code can additionally submit
`api.account.setNewPlayerStats` (for example progression `0 -> 1000`). That is
onboarding state, not inventory item serialization.

**Current handler.** Both account and part-list methods dispatch through
`API.game` at `server/game/api.go:151-186`, registered for GET and POST on
`/game/api` at `server/game/api.go:64-102`. `accountResponse` is
`server/game/api.go:317-404`; `partList` is lines 407-417.

### Opening the inventory screen

**Native-proven.** Opening the screen loads the local bundled
`Inventory.swf` at `Game.c:178808-178870`. There is no call to the only
part-list submit wrapper in this open path. The SWF consumes the native cache
populated on ship/editor entry.

**Result:** no new HTTP, Blaze, or RakNet inventory request is made merely by
opening the screen. If the earlier HTTP response did not populate valid native
cache records, reopening the screen remains incomplete.

### Switching inventory tabs or filters

**Native-proven within the available executable call graph.** There are only
three calls to `sub_4B7100` in the executable: editor/ship initialization,
another state transition using argument one, and game-leave refresh. None is in
the inventory-screen open/close or tab callback range. Tab/filter changes use
the already-populated native inventory cache.

**Result:** switching the inventory's display tabs does not reload the
persistent database. It can expose or hide cached records based on decoded
market status, flair/type, element/class eligibility, and equipped state. This
is why an invalid decoded market status is materially different from an empty
database.

### Opening the Hero Profile inventory region

**Native-proven.** The profile is the bundled XHTML page, not `Inventory.swf`.
For an owned creature it makes exactly this data call:

```text
method=api.creature.getCreature
id=<Creature.ID as decimal>
include_parts=true
include_abilities=true
```

The page consumes `<creature><parts><part>...` from that one response; it does
not call `api.inventory.getPartList`. The current dispatch is
`server/game/api.go:193-196`, and `creatureResponse` at lines 428-443 ignores
the include flags and calls the small `creatureNode` at lines 631-635.

### Returning from the tutorial/game to the ship

**Native-proven.** The game-leave handler at
`Game.c:322991-322995` performs, in order:

1. `api.account.getAccount`, with the current account ID and the constructor's
   `include_decks` argument false;
2. local player/cache teardown work; and
3. `api.inventory.getPartList` with `count=10000` and
   `filter=market_status_full-owned;`.

Thus returning to the ship explicitly refreshes both capacity/account state and
owned inventory. No Blaze inventory component or RakNet inventory snapshot is
called by this path. Tutorial progression may separately submit
`api.account.setNewPlayerStats`; it does not carry part rows.

## Exact ordinary inventory response contract

### Transport and envelope

**Native-proven.** Ordinary inventory is HTTP/XML through
`api.inventory.getPartList` on `/game/api`, not Blaze and not RakNet. The
response is:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<response>
  <parts>
    <part>...</part>
    <!-- repeated -->
  </parts>
  <stat>ok</stat>
  <code>200</code>
  <result>1</result>
</response>
```

`sub_4AD680` selects `response/parts` and passes all direct `<part>` children to
`sub_4ACC60`. Container names are exact and plural: `<response>`, `<parts>`,
then repeated singular `<part>`.

### Part fields and numeric formats

**Native-proven.** `sub_4ACC60` at `Game.c:271380-271679` reads exactly:

```xml
<part>
  <is_flair>0</is_flair>
  <cost>0</cost>
  <creature_id>0</creature_id>
  <id>4</id>
  <level>1</level>
  <market_status>1</market_status>
  <prefix_asset_id>1670541141</prefix_asset_id>
  <prefix_secondary_asset_id>1586359867</prefix_secondary_asset_id>
  <rarity>1</rarity>
  <reference_id>4294967300</reference_id>
  <rigblock_asset_id>2640980717</rigblock_asset_id>
  <status>0</status>
  <suffix_asset_id>3766637798</suffix_asset_id>
  <usage>0</usage>
  <creation_date>1784563502</creation_date>
</part>
```

All integral fields are base-10 text. `id`, `reference_id`, `creature_id`,
`status`, and `creation_date` are parsed through the client's unsigned 64-bit
conversion path. The remaining numeric fields are parsed as decimal integers;
`is_flair` is false only when zero.

`market_status` and `rarity` are wire enums with a one-based XML
representation. The parser executes `--market_status` and `--rarity` at
`Game.c:271666-271667`. Darkspin's stored/domain representation is
zero-based, so the response DTO must encode `stored + 1` for both fields.

The ordinary list contract has no textual rarity, stats string, PNG key,
owner/account ID, avatar ID, explicit deleted flag, or explicit equipped
boolean. `creature_id != 0` is the equipped relationship.
The authenticated request determines the owning account.

### Mission inventory drop acknowledgement

Mission Drop now prepares a ground equipment pickup before `DropPartTo` removes the inventory row. Placement failure leaves inventory unchanged; save failure restores the row and rolls back the provisional pickup before any packets are sent. The gameplay registry lock excludes collection during this transfer. The dropped item's affixes, rarity, level, and creation date are preserved; collection allocates inventory IDs again and returns the item to its original owner without another co-op loot roll. Requests without an active ground destination are rejected without deletion. Ground items follow the existing mission pickup lifetime.

The 0.7.24 report captured at `2026-09-04T02:39:49Z` records item 13 being removed once on the server, followed by four retries for the same item while it remained visible in the client. The UI's `sub_5372C0` sends the eight-byte persistent ID but does not erase the local cache. Native `sub_4E17E0` handles client event `0xA717F30B`: it reads the item ID at event offset `+112` (ServerEvent reflection field 18), calls `sub_4C1880` to remove that inventory entry, then calls `sub_42F290` to mark the inventory panel for refresh. Send this sparse event only to the requesting peer after successful persistence, including an already-absent retry; do not send it after an equipped-item rejection or save failure. No full inventory snapshot or client hook is required.

### Identity roles

- `id` is the persistent per-account inventory instance ID (`Part.ID`). It is
  the identity used by ship Drop wire `0xcb` and by darkspin's `DropPart`.
- `reference_id` is a separate 64-bit stable/deduplication identity. Current
  grants form `(accountID << 32) | Part.ID`; for row 4 this is
  `0x0000000100000004` / `4294967300`.
- `creature_id` is `Part.EquippedToCreatureID`, referring to the owning
  account's `Creature.ID`; it is neither `ReferenceID` nor a template noun.
- Despite their `_id` names, `rigblock_asset_id`, `prefix_asset_id`,
  `prefix_secondary_asset_id`, and `suffix_asset_id` are the generated asset
  hashes on this HTTP/XML boundary. Persisted catalog IDs remain server-side.
  The same hash identities are used in RakNet acquisition events.

**Native-proven.** After parsing, the response handler resolves
`rigblock_asset_id` through `sub_9C5FF0`. It calls the cache insertion routine
only when that lookup succeeds (`Game.c:288017-288073`). The reference ID
is used by the native duplicate lookup before insertion
(`sub_4C1C80`, `Game.c:288174-288222`). Therefore:

- an invalid rigblock hash is a hard cache-admission failure;
- missing/zero reference IDs are unsafe deduplication identities, especially
  when more than one legacy row is zero;
- ID zero is not rejected by the XML parser itself, but is not a usable
  persistent command identity;
- level zero or one and `creature_id=0` are not rejected in this load path.

### Capacity, ordering, and pagination

Capacity is not part of `getPartList`. It comes from account XML. Build 103
uses `unlock_inventory` as the active slot ceiling, while the server's stored
`UnlockInventory` field is an upgrade tier. The HTTP adapter therefore projects
the computed `UnlockInventoryIdentify` capacity through both wire elements:

```xml
<unlock_inventory>180</unlock_inventory>
<unlock_inventory_identify>180</unlock_inventory_identify>
```

The build-103 parser reads the elements into separate account fields at
`Game.c:270856-270928`. Live differential testing proved that sending
`unlock_inventory=0` and `unlock_inventory_identify=180` admitted the item but
raised `Inventory Full!`; sending `180` through both rendered
`Capacity: 1/180`.

The request has `count=10000` but no offset, cursor, or page number. The native
parser appends records in XML order. Current persistence restores by
`ORDER BY position` (`server/sporenet/sqlite/records.go:238-251`), and
`partList` retains slice order. There is no native evidence of another page for
normal account inventory. A correct handler should apply confirmed filters,
cap the result at the requested count, and keep deterministic position/ID
ordering.

### Notification and RakNet requirements

No extra network notification is required for initial/reload inventory. The
HTTP callback emits an internal client message (`166024845`), and the native
inventory manager resolves and inserts the parsed records. This is in-process
message dispatch, not Blaze or RakNet.

RakNet acquisition reflection is required only to present a newly acquired item
live during a game. Darkspin does that at `server/gameplay_udp.go:1996-2033`.
It sends both `ReferenceID` and `Part.ID`, plus hashed asset identities. That
path explains why `/summon` can appear live while the later HTTP reload fails.

## Hero Profile equipped-part contract

The Hero Profile's nested part schema is richer and is not interchangeable with
the ordinary list DTO. Each equipped part requires at least:

```xml
<parts>
  <part>
    <type_full>weapon|grasper|foot|defense|offense|utility|detail</type_full>
    <is_flair>0|1</is_flair>
    <stats>KEY,base,bonus;...</stats>
    <cost>decimal integer</cost>
    <level>decimal integer</level>
    <class_types_full>...</class_types_full>
    <science_types_full>...</science_types_full>
    <rarity_full>basic|uncommon|rare|epic|unique|rareunique|epicunique</rarity_full>
    <rigblock_asset_id>decimal integer</rigblock_asset_id>
    <png_key>URL-or-client key</png_key>
    <suffix_asset_id>decimal integer</suffix_asset_id>
    <prefix_asset_id>decimal integer</prefix_asset_id>
    <prefix_secondary_asset_id>decimal integer</prefix_secondary_asset_id>
    <rarity>one-based decimal enum</rarity>
    <weapon_damage_modifier>decimal number</weapon_damage_modifier>
  </part>
</parts>
```

The server must select parts where `EquippedToCreatureID == Creature.ID`, then
enrich them from build-103 loot content. The current `creatureNode` has no
`<parts>` container, and the generic `partNode` lacks the rich content-derived
fields. This is an exact, independent implementation gap.

## Real-row end-to-end trace

This trace uses user 1, position 4, the item live-presented at 09:05.

### 1. Database row

**Database evidence:**

```text
user_id=1 position=4 item_id=4 reference_id=4294967300
is_flair=0 cost=0 creature_id=0 level=1 market_status=0 rarity=0
status=0 usage=0 creation_date=1784563502
rigblock_asset_id=560 prefix_asset_id=29
prefix_secondary_asset_id=23 suffix_asset_id=40
```

The owner row is account ID 1/login `Test`, account level 2, avatar ID 0, and
`unlock_inventory_identify=180`. The `ReferenceID` high word equals owner ID 1.

### 2. SQLite adapter to `sporenet.Part`

`loadRecordTx` selects all user parts ordered by position and maps every stored
column directly at `server/sporenet/sqlite/records.go:238-251`. Then
`NewUserFromRecord` copies all parts and calls `Part.Normalize` at
`server/sporenet/user_record.go:48-69`.

Normalization retains numeric asset IDs and rebuilds the hashed forms. For this
row:

```text
RigblockAssetHash          = 0x9d6a2aed
PrefixAssetHash            = 0x63926f55
PrefixSecondaryAssetHash   = 0x5e8dee3b
SuffixAssetHash            = 0xe08254e6
```

No load filter excludes unequipped, level-1, status-zero, or basic parts.

### 3. Current response DTO/projection

`partList` copies `user.View().Parts` and calls `partNode` for every row
after applying the confirmed count, owned, and creature filters. `partNode`
emits every native-parsed scalar, one-based wire enums, and asset hashes. For
the traced row the asset portion is:

```xml
<part><id>4</id><reference_id>4294967300</reference_id>
<rigblock_asset_id>2640980717</rigblock_asset_id>
<prefix_asset_id>1670541141</prefix_asset_id>
<prefix_secondary_asset_id>1586359867</prefix_secondary_asset_id>
<suffix_asset_id>3766637798</suffix_asset_id><cost>0</cost><level>1</level>
<market_status>1</market_status><rarity>1</rarity>
<status>0</status><usage>0</usage>
<creation_date>1784563502</creation_date></part>
```

The complete document wraps this in `<response><parts>...` and appends
`<stat>ok</stat><code>200</code><result>1</result>`.

### 4. Client decode and lookup

The build-103 parser decrements `market_status` and `rarity`, restoring the
server's zero-based Owned and Basic values. The response manager resolves hash
`0x9d6a2aed` through `sub_9C5FF0` and inserts the record under reference
identity `4294967300`. Sending catalog ID `560` instead fails that lookup and
drops the row before the SWF sees it.

### 5. Live presentation comparison

**Live-log evidence:** `darkspinner.log:37398` records:

```text
RakNet summoned item presented id=4 reference=4294967300 rigblock=560
```

That event was built at `server/gameplay_udp.go:2016-2026` with
`ReferenceID=4294967300`, `InstanceID=4`, hashed rigblock/affix IDs, level 1,
rarity 0, and the same creation time. It bypasses the broken HTTP enum/schema
projection, which exactly fits “appears live, disappears/incomplete on reload.”

## Exclusion audit

| Candidate | Finding |
| --- | --- |
| Unequipped | Not excluded by persistence or ordinary part-list handler. Native `creature_id=0` means unequipped and is accepted. |
| Level 0/1 | Native parser stores the decimal level without a load-time rejection. Real summoned rows are level 1. Not the cause. |
| Missing `ReferenceID` | One legacy row per users 1 and 2 has `item_id=0/reference_id=0`; those rows need migration/repair. Current `/summon` rows have valid unique composite references. Not the cause for rows 1-4. |
| Wrong owner/avatar | Row 4 has `user_id=1`, matches authenticated account 1, and its reference high word is 1. Avatar ID is not in the inventory part contract. Not the cause. |
| Wrong asset identity | Was a primary defect: HTTP sent catalog IDs while native cache admission requires hashes. The repaired projection is live-verified with Electro Claws. |
| Deleted/invalid status | There is no explicit deleted field. Stored `market_status=0` is the internal Owned enum and its XML form is 1. Stored `status=0` is transmitted and not decremented. |
| Zero capacity | Was a second primary defect: the adapter sent the stored upgrade tier through `unlock_inventory`. It now projects the computed capacity through both wire fields. |
| Hero Profile XML | Definitely incomplete and independently causes blank equipped item/detail slots. It does not populate the separate `Inventory.swf` list. |

## Content.db evidence

The authoritative runtime database identifies itself as:

```text
database_role=runtime-content
source_build=103
content_release=build-103-content
recipe_version=19
```

The current recipe includes source resources, ServerData, creatures, levels,
Lua, localization, physics, and related runtime projections. It does **not**
contain `loot_rigblock`, `loot_prefix`, `loot_suffix`, or
`creature_part_template` tables, and queries for those tables fail as missing.
Queries of `content_source_resource.instance_id` for the generated loot-name
hashes also return no rows; those generated gameplay catalog IDs are not DBPF
resource instance IDs in this projection.

Therefore content.db currently cannot validate or enrich persistent inventory
assets. This is positive evidence of a server-side content-port gap, not
evidence that client asset 560 is absent. The actual build-103 client registry
in `Data/AssetData_Binary.package` remains the native authority used by
`sub_9C5FF0`.

## Current handler assessment

| Request | Current function | Assessment |
| --- | --- | --- |
| `api.inventory.getPartList` | `API.game` -> `API.partList` -> `partNode` | Applies confirmed filters/count, emits all required scalars, one-based enums, and hashed asset identities. Live-verified with one persisted Electro Claws row. |
| `api.account.auth/getAccount` | `API.game` -> `accountResponse` | Projects computed capacity through both inventory wire elements. Parts are loaded separately. |
| `api.creature.getCreature` | `API.game` -> `creatureResponse` -> `creatureNode` (`server/game/api.go:193-194`, `428-443`, `631-635`) | Ignores `include_parts`; emits no equipped-parts container and no rich part DTO. |
| `api.inventory.updatePartStatus` | Exact captured Create Detail status-4 batch is persisted atomically; other operators and status transitions reject. | Equip/unequip and unknown status mutations still require recovered semantics. Not the initial-list cause. |
| RakNet Drop `0xcb` | `server/gameplay/inventory.go` | Transfers the persistent item through `DropPartTo` to a ground pickup, then sends the native inventory-removal event after persistence succeeds; already-absent items receive the same idempotent acknowledgement without another pickup. |
| `/summon` live presentation | `itemSummonChatAdapter.SummonItem` and gameplay pending presentation (`server/developer_chat.go:18-47`, `server/gameplay_udp.go:1996-2033`) | Persists first, then optionally emits a one-shot RakNet acquisition event. This path is correct evidence for the transport split. |

No Blaze component currently answers inventory-list or Hero Profile item data,
and build-103 native evidence does not require one for these paths.

## Exact implementation-change locations

1. `server/game/api.go:185-186` — pass request values into the part-list
   operation instead of discarding `count` and `filter`.
2. `server/game/api.go:407-417` — decode and enforce the confirmed owned and
   creature filters, bound the count, retain deterministic order, and project a
   dedicated ordinary-inventory DTO.
3. `server/game/api.go:615-628` — add `is_flair`, `creature_id`, and
   `market_status`; encode `market_status` and `rarity` as stored value plus
   one. Keep `id=Part.ID` and `reference_id=Part.ReferenceID`.
4. `server/game/api.go:193-194` and `428-443` — honor `include_parts=true`,
   select only parts equipped to the requested creature, and use a distinct
   Hero Profile projection.
5. `server/game/api.go:631-635` — expand the creature profile contract and add
   the required `<parts>` container. Do not reuse the ordinary list part DTO.
6. `content/sqlite` schema/store and the consuming game feature port — add a
   build-103 loot-definition lookup capable of validating rigblock/affix IDs
   and supplying type, stats, eligibility, rarity text, PNG key, and damage
   fields. This is required for Hero Profile enrichment and desirable for grant
   validation.
7. A focused persistence migration/normalization path near
   `server/sporenet/sqlite/repository.go:258-281` — repair legacy zero
   `item_id/reference_id` rows deterministically without changing valid IDs.
   Do not conflate this cleanup with the response-contract fix.

## Prioritized minimal implementation plan

### P0: restore ordinary persisted inventory

1. Introduce an ordinary inventory response DTO with all 15 native-parsed
   fields.
2. Encode `MarketStatus+1` and `Rarity+1`; emit booleans as `0`/`1`; emit all
   IDs and timestamps as unsigned decimal.
3. Parse the semicolon filter grammar needed for
   `market_status_full-owned;` and `creature_id-<id>;`, apply `count`, and keep
   stable order.
4. Repair legacy zero IDs/references on load or migration, preserving all valid
   item identities.

This is the smallest change expected to make `/summon` rows 1-4 survive a ship
reload and appear in Owned inventory.

### P1: restore Hero Profile equipped parts

1. Add `include_parts` handling to `getCreature`.
2. Select by `EquippedToCreatureID == Creature.ID`.
3. Add the loot-content lookup/projection and emit the rich nested schema.
4. Implement `updatePartStatus` through a feature operation so equip state can
   be persisted transactionally.

### P2: harden content identity and acquisition refresh

1. Validate grants against actual build-103 loot content rather than numeric
   ranges alone.
2. Keep RakNet acquisition reflection for active-game presentation, but always
   treat HTTP `getPartList` as reload authority.
3. Add decoded, redacted HTTP method/response tracing under
   `bin/server/darkspin/logs/traces` for future live contract captures.

## Test specification

### Unit tests

- Ordinary part DTO: every field, decimal formatting, false/true flair,
  equipped/unequipped creature ID, and `MarketStatus+1`/`Rarity+1` across all
  enum values.
- Filter parser: owned, creature ID, combined filters, empty filter, malformed
  filter, count zero, count above inventory size, and stable ordering.
- Identity migration: zero legacy IDs/references become unique composites;
  existing valid IDs/references are unchanged.
- Hero Profile selection: only parts equipped to the requested creature are
  included; unequipped and other-creature items are excluded from that nested
  region only.

### HTTP integration tests

- Authenticate a user with mixed equipped/unequipped parts and call the exact
  build-103 query. Assert Owned/basic rows encode as `1`/`1` and all 15 fields
  exist.
- Exercise `creature_id-<id>;`, owned, and combined filters.
- Assert unauthenticated calls fail without leaking any rows.
- Assert account response returns capacity 180 independently of part count.

### Response-golden tests

- Golden XML for empty inventory, one basic unequipped item, one equipped flair
  item, maximum 64-bit IDs/date, and multiple items in persistence order.
- Decode the golden with a small test model that mirrors build-103 behavior and
  decrements market/rarity; assert the resulting internal enums are valid.
- Separate Hero Profile golden containing the exact rich nested part schema.

### Persistence-reload tests

- Grant two identical definitions through `GrantPart`, close/reopen the SQLite
  repository, authenticate again, and assert distinct IDs/references plus
  byte-equivalent inventory XML before and after reload.
- Seed a legacy zero-ID/reference row, run normalization, reload twice, and
  prove identity stability/idempotence.
- Persist equipped and unequipped rows and prove ordinary Owned inventory
  retains both while Hero Profile contains only the equipped row.

### Live verification (after implementation, outside this research task)

1. Start with the exact user-1 row 4 or summon a known valid rigblock.
2. Capture redacted HTTP method/query and XML response under
   `bin/server/darkspin/logs/traces`.
3. Verify it appears live, open Inventory, switch every tab/filter, close and
   reopen Inventory, return to ship, restart the server/client, and verify it
   remains visible.
4. Equip it, verify Hero Profile's correct item/detail slot and tooltip, return
   to Inventory, then unequip it and repeat.
5. Drop it via wire `0xcb`, verify `Part.ID` is sent, and confirm it remains
   absent after persistence reload.
6. Record reverse-engineering-only diagnostics under `bin/game/logs`, never
   directly under `bin`.

## Implemented ordinary-list boundary

The ordinary `api.inventory.getPartList` response now emits all 15 fields read
by build 103, including `is_flair`, `creature_id`, and `market_status`.
`market_status` and `rarity` are encoded as the stored zero-based enum plus one,
so an owned/basic row decodes back to `0`/`0` instead of `-1`/`-1`. The handler
also honors the proven `count`, `market_status_full-owned;`, and
`creature_id-<id>;` query forms while retaining persistent slice order.

Focused tests cover every emitted field, enum offset behavior, combined
filters, and stable permissive defaults for absent or unknown query fields.
Legacy zero-identity repair, loot-catalog validation, persisted equip/status
mutation, and the distinct content-enriched Hero Profile part projection remain
open; the generic inventory DTO is intentionally not reused for that richer
screen.

The current Test account was subsequently checked through `darkrun db`: user 1
has the tutorial Electro Claws with distinct nonzero item/reference identities.
The ordinary response path therefore now logs a safe projection summary
(`account`, numeric user ID, stored/returned counts, count limit, and parsed
filters) without logging credentials or XML. The next live run can distinguish
a missing client request from client-side rejection of rows that the handler
did return.

The empty Hero Profile slots observed on July 20 had a separate, persisted
cause: the claws row carried `creature_id = 0`. The original editor equip call
arrived while `api.creature.updateCreature` still used the compatibility
acknowledgement and therefore could not save the association; later paint-only
saves authoritatively sent an empty `parts` field. Tutorial completion now
equips an unequipped authored claws reward to Blitz transactionally and repairs
that legacy state when the idempotent completion command is rerun. Normal
editor saves remain authoritative afterward, and completion does not steal a
reward already equipped to another creature.
