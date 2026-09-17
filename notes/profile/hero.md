# Build 103 Hero Profile investigation

## Conclusion

### July 20 live implementation follow-up

The instance endpoint now emits the complete decoded field set, including
non-empty fallback text for fields that the callback dereferences before its
owned-creature native override, and a template-thumbnail fallback when editor
images have not yet been saved. Live Wraith responses were captured with every
required node populated, but the shipped page still remained in `reset()`.

The following hypotheses were each ruled out by separate rebuilt live runs:

- legacy `creator_id=0` versus an authenticated-owner projection;
- bare authored locale keys versus `group!key` identities;
- a known-good `GameUI` localization key; and
- a 25 ms loopback response delay for the page's callback-registration order.

The server receives `api.creature.getCreature` with `id=3`,
`include_parts=true`, and `include_abilities=true`. The remaining blank-page
defect is therefore in callback delivery or a native JavaScript bridge failure,
not a missing top-level XML node. Client-side exception/bridge instrumentation
is the next useful discriminator.

The blank Hero Profile is caused primarily by an incomplete XML contract, not by
the tutorial heroes having level 0. The bundled `creatureprofile.html` requests
`api.creature.getCreature` (or `api.creature.getTemplate`) and then reads fields
that the current handlers do not return. After checking `<stat>`, the first
unconditional read is `<name_locale_id>`. Its absence raises a JavaScript error
before the page writes the portrait, title, abilities, parts, or stats. The
screen therefore remains in its `reset()` state, which matches the supplied
screenshot.

This defect affects every instance and template profile served by the current
handlers. Tutorial Blitz/Sage make it easy to reproduce, but neither their
identity nor an account/player level of 0 is the root cause.

There are additional, independent gaps:

1. The template portrait branch requests `/game/service/png`; the compatibility
   route now serves the cached template thumbnail while honoring the path the
   shipped page hard-codes.
2. Newly created tutorial creature instances have no guaranteed instance image
   URLs;
3. equipped parts require content enrichment that the current API does not do;
4. the main bundled web view expects account `<blaze_id>` and `<name>`, which the
   current account response omits;
5. the separate inventory API omits fields and uses the wrong enum base for the
   native part-list parser; and
6. an instance ID is only unique within one account, while the current profile
   lookup has no sound remote-owner identity path.

No Go or C implementation file was modified, and no binary was built or
launched. No new IDA run was necessary: the existing decompilation plus a
read-only decode of the repository's `Data/Web.package` exposed the complete
page script. No material from `C:\src\recap` was inspected.

## Evidence labels

This report uses three labels deliberately:

- **Native-proven**: observed in bundled build-103 JavaScript or in
  `bin/darkspinner/GameBin/Game.c`.
- **Live-log evidence**: observed in existing runtime logs under this repository.
- **Inference**: a conclusion that connects the native contract, current server,
  screenshot, or logs. It is not presented as a captured live HTTP payload.

The current runtime does not log decoded HTTP API method names or XML bodies.
`bin/server/darkspin/logs/traces/game.jsonl` contains socket directions,
lengths, and digests, so it cannot independently prove which Hero Profile request
was sent. The request strings below are native-proven rather than reconstructed
from a live packet capture.

## End-to-end sequence

### 1. Account/bootstrap cache

**Native-proven.** Initial account/auth requests can include:

```text
include_creatures=true
include_decks=true
include_feed=true
include_settings=true
build=<build number>
```

The account creature parser at `Game.c:267488` (`sub_4AC4A0`) expects each
cached creature to contain:

```xml
<creature>
  <id>unsigned decimal</id>
  <name>...</name>
  <png_thumb_url>...</png_thumb_url>
  <xml_url>...</xml_url>
  <noun_id>unsigned decimal</noun_id>
  <version>unsigned decimal</version>
  <gear_score>floating point</gear_score>
  <item_points>floating point</item_points>
</creature>
```

The current `creatureNode` at `server/game/api.go:631` supplies all of these
except `<xml_url>`. That cache object is sufficient to expose/select a hero, but
it is not the rich object required by the profile page.

The native `api.account.getAccount` constructor at `Game.c:266414`
(`sub_4A7D50`) sends `id=<signed 64-bit decimal>` and optionally
`include_decks=true`. The main bundled page also refreshes the current account
with an empty-parameter `api.account.getAccount` call and reads account
`<blaze_id>`, `<name>`, `<id>`, and `<grant_online_access>`. The current account
node at `server/game/api.go:330-359` supplies the last two, but not `blaze_id` or
`name`.

### 2. Opening the bundled screen

**Native-proven.** The native client loads:

```text
game:///UI/XHTML/Labs/Setup/mainwebview.html
```

The resource is bundled in `Data/Web.package`; it is not fetched from a server
`/web/sporelabsgame` route. The native setup around `Game.c:173366`
registers JavaScript bridges including:

- `callSporeNet`;
- localization/content lookup helpers;
- `getCreatureBaseStats`;
- `getCreatureAllBaseAbilityKeyvalues`;
- `getCreatureAbilityKeyvalues`;
- `retrieveAbilityPngKey`; and
- `getLootName`.

The decoded main page contains:

```javascript
function showCreatureProfile(id, template, source, blazeid) {
    // reset/activate frame, then:
    FRAME1.showScreen(id, template, source, blazeid);
}
```

The native call site at `Game.c:174345` (`sub_429900`) invokes
`showCreatureProfile(%lld,'%c')`, passing the selected ID and `Y` for a template
or `N` for an owned instance. The optional JavaScript `source` and `blazeid`
arguments are undefined on this direct local path.

The wrapper marks screen number 1 active (`creatureprofile`), resets/hides the
frame, and begins `LABS_UI_TIME_CREATUREPROFILE` telemetry.

### 3. Profile HTTP/XML request

**Native-proven.** `creatureprofile.html` makes exactly one of these calls:

```javascript
// Persistent creature instance
parent.Client.callSporeNet(
    "api.creature.getCreature",
    "id=" + creatureid +
      "&include_parts=true&include_abilities=true",
    "spgetcreaturecallback"
);

// Content template
parent.Client.callSporeNet(
    "api.creature.getTemplate",
    "id=" + creatureid + "&include_abilities=true",
    "spgetcreaturecallback"
);
```

For `template == 'N'`, `id` is `sporenet.Creature.ID`. For the template path,
`id` is the template noun/content ID, `sporenet.TemplateCreature.Noun`.

The current dispatch is at `server/game/api.go:193-196`. It discards all include
parameters. `creatureResponse` at lines 428-443 parses a base-10 32-bit ID and
searches only the authenticated user's creatures. `templateResponse` at lines
445-457 parses a base-0 32-bit noun and searches the template store.

There is no Hero-Profile-specific Blaze call after a local hero is selected.
Blaze participates in identity/social discovery for other users, described
below, but the actual profile data is HTTP/XML.

### 4. Callback and screen population

**Native-proven.** `spgetcreaturecallback` checks `<stat>` and then consumes the
following profile schema. These names and nesting are exact.

```xml
<response>
  <creature> <!-- <template> is also accepted because reads are global -->
    <name_locale_id>unsigned decimal/hash</name_locale_id>
    <text_locale_id>unsigned decimal/hash</text_locale_id>
    <name>...</name>
    <type_a>lowercase element token</type_a>
    <creator_id>unsigned decimal</creator_id> <!-- instance -->
    <weapon_min_damage>decimal number</weapon_min_damage>
    <weapon_max_damage>decimal number</weapon_max_damage>
    <gear_score>decimal number</gear_score> <!-- optional in JS -->
    <class>lowercase class token</class>
    <stats_template_ability>...</stats_template_ability>
    <stats>...</stats> <!-- instance -->
    <stats_template>...</stats_template> <!-- template -->
    <stats_template_ability_keyvalues>...</stats_template_ability_keyvalues>
    <stats_ability_keyvalues>...</stats_ability_keyvalues> <!-- instance -->
    <parts>...</parts> <!-- required container for instance, may be empty -->
    <creature_parts>all|no_feet|no_hands</creature_parts>
    <ability_passive>unsigned decimal</ability_passive>
    <ability_basic>unsigned decimal</ability_basic>
    <ability_random>unsigned decimal</ability_random>
    <ability_special_1>unsigned decimal</ability_special_1>
    <ability_special_2>unsigned decimal</ability_special_2>
    <ability>
      <id>unsigned decimal</id>
      <locale_name>unsigned decimal/hash</locale_name>
      <locale_description>unsigned decimal/hash</locale_description>
    </ability>
    <!-- more repeated <ability> nodes -->
    <png_large_url>URL</png_large_url> <!-- instance -->
    <png_thumb_url>URL</png_thumb_url> <!-- instance -->
  </creature>
  <stat>ok</stat>
  <code>200</code>
  <result>1</result>
</response>
```

The present instance response at `server/game/api.go:631-635` contains only:

```xml
<creature>
  <id>...</id><name>...</name><noun_id>...</noun_id><version>...</version>
  <gear_score>...</gear_score><item_points>...</item_points>
  <png_thumb_url>...</png_thumb_url>
</creature>
```

The present template response contains only `id`, `name`, `element_type`, and
`class_type`. `element_type` and `class_type` are not aliases recognized by the
page: it reads `type_a` and `class`.

**Inference, strongly determined.** Because `name_locale_id` is the first
missing unconditional lookup, the callback throws there. The reset frame and
static HTML remain, but every dynamic region stays blank. This is the precise
state visible in the supplied screenshot.

## What populates each visible region

### Portrait, name, and header

**Native-proven.** The instance portrait uses `png_thumb_url`; its expanded
detail image uses `png_large_url`. The template path instead constructs:

```text
http://<SnApiHost>/game/service/png?template_id=<noun>&size=large
```

The compatibility router now exposes `/game/service/png` in addition to
`/template_png`, `/creature_png`, and `/assets`. Its `template_id` form resolves
the cached hero thumbnail at the exact URL hard-coded by the shipped page.

The page localizes `name_locale_id` and overwrites the raw `<name>`. It renders
the text before the first comma as the uppercase title and the text after comma
plus a space as the subtitle. A localized name without a comma is therefore a
malformed display value even when the node exists.

`type_a` and `class` select icons and localized labels. The page's lookup maps
expect lowercase semantic tokens (for example `plasma` and `ravager`), while it
uppercases a copy for icon selection.

The markup has a “Hero Level” label, but the shipped script does not assign a
hero level to it. It reads `gear_score`, but the visible assignment is commented
out. `sporenet.Creature` has no level field; level belongs to the account and to
parts. A blank Hero Level label is therefore not evidence of a level-0 creature.

### Level and stats rows

**Native-proven.** The lower-right area in the screenshot is the eight-row stats
panel, not an inventory list. The stats value is a semicolon-delimited string:

```text
KEY,base,bonus;KEY,base,bonus;
```

`base` and `bonus` are parsed with `parseInt`; display is `base + bonus`. A zero
bonus is gray and a nonzero bonus is green. The first eight entries are remapped
in order `[3, 4, 0, 1, 2, 7, 6, 5]`, and `KEY` is localized through the native
item-string map.

The next eight rows are Critical Damage, Projectile Speed, Cooldown Reduction,
Area Effect Damage, Area Effect Resistance, Movement Speed, Area Effect
Duration, and Health Leech. The seventh fallback key is `AOEDUR` (logical
attribute 95), not the similarly named `BUFDUR`/BuffDuration attribute 50.

For an owned local instance (`template == 'N'` and `mainplayer == creator_id`),
the XML stats are replaced by native `getCreatureBaseStats(creatureid)`, and
ability values are replaced by native helpers. This optimization does not save
the current response: execution has already dereferenced the missing header
nodes before it reaches the local override.

### Ability slots

**Native-proven.** The five left-hand rows are ability slots. Their order is:

1. basic;
2. special 1;
3. special 2;
4. random; and
5. passive.

The five selector nodes associate those slots with repeated `<ability>` records.
Each record needs `id`, `locale_name`, and `locale_description`. The icon comes
from native `retrieveAbilityPngKey(id)`; localization/attribute bridges supply
the final text and dynamic values.

Ability-value formats are delimiter-sensitive:

```text
stats_template_ability_keyvalues / stats_ability_keyvalues:
  abilityId!token,value;abilityId!token,value;

stats_template_ability:
  abilityId!key!token[,token];abilityId!key!token[,token];
```

`TemplateCreature` retains the five ability IDs, but the current response emits
none of them and no locale metadata. `content.db` has
`creature_template_ability` relationships, while script definition/localization
content is the likely source for display metadata and token definitions.

The current `Creature.Update` representation is also not sufficient evidence of
a wire-compatible round trip: `AbilityStat` has `Key`, `Token`, and `Value`, but
the parser at `server/sporenet/creature.go:235` splits an encoded value at `!`,
assigns only token/value, and does not preserve the complete client key
relationship. This needs a contract test before it is used to serialize profile
ability values.

### Item slots, detail slots, and equipped parts

**Native-proven.** The six upper-right “Item Slots” and six “Detail Slots” are
populated from nested `<parts><part>...</part></parts>` in
`api.creature.getCreature`. The page does **not** call
`api.inventory.getPartList` for these slots.

Each nested part expects:

```xml
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
  <png_key>URL/key</png_key>
  <suffix_asset_id>decimal integer</suffix_asset_id>
  <prefix_asset_id>decimal integer</prefix_asset_id>
  <prefix_secondary_asset_id>decimal integer</prefix_secondary_asset_id>
  <rarity>one-based decimal enum</rarity>
  <weapon_damage_modifier>decimal number</weapon_damage_modifier>
</part>
```

For non-flair parts, `type_full` maps to the six item slots. Limited anatomy can
remap a duplicate class slot to foot/grasper: Ravager uses offense, Tempest uses
utility, Sentinel uses defense, in combination with `creature_parts` values
`all`, `no_feet`, or `no_hands`. Flair/detail parts are placed sequentially in
the six detail slots.

`png_key` is used directly as an image source. The tooltip name is resolved by:

```text
getLootName(rigblock, suffix, prefix, prefix_secondary,
            rarity - 1, is_flair)
```

The subtraction proves that this XML consumer expects one-based numeric rarity.

The server must select only parts where
`Part.EquippedToCreatureID == Creature.ID`, then enrich them from content. The
current profile response includes no `<parts>` at all. The generic `partNode`
also lacks type/slot, stats, eligibility, textual rarity, PNG, and damage data,
so simply nesting it would still be malformed for this consumer.

### Inventory list (separate screen)

**Native-proven.** There is no inventory list on `creatureprofile.html`; the
lower-right rows in the screenshot are stats. The separate native `Inventory.swf`
path calls:

```text
method=api.inventory.getPartList
count=10000
filter=creature_id-<signed 64-bit decimal>;
filter=market_status_full-owned;  # when requesting owned inventory
```

The request constructor is `Game.c:268064` (`sub_4A8410`). Its parser at
`Game.c:268492` (`sub_4ACC60`) expects each part to contain:

```text
is_flair, cost, creature_id, id, level, market_status,
prefix_asset_id, prefix_secondary_asset_id, rarity, reference_id,
rigblock_asset_id, status, suffix_asset_id, usage, creation_date
```

IDs are parsed as unsigned decimal. `market_status` and `rarity` are parsed as
decimal integers and then decremented, so their XML representation is one-based.

The current `partList` at `server/game/api.go:407-417` ignores `count` and
`filter`, returns every part, and its `partNode` omits `is_flair`, `creature_id`,
and `market_status`. It emits the stored zero-based rarity directly, which the
client decrements again. These are real inventory defects, but they do not cause
the supplied Hero Profile state because the profile uses its nested rich-part
schema.

### Tab and frame state

**Native-proven.** `reset()` clears the portrait, five ability rows, six item
slots, six detail/flair slots, eight stats rows, and all dynamic labels. It
selects Base Stats over Additional Stats and calls `switchtoability()`, selecting
Ability over Description. The outer page's `setScreenActive("1")` records
`creatureprofile` as the active frame.

Thus the screenshot proves the static page/frame loaded and reset completed; it
does not prove that a populated callback completed. The visible selected-tab
state is the page default.

## Blaze RPCs and remote identity

**Native-proven/current-server-proven.** User discovery uses Blaze UserSessions,
component `0x7802`, command `0x000c` (`lookupUser`). The request has `NAME`; the
current handler returns:

```text
EDAT
FLGS = 1
USER {
  AID, ALOC, EXBB, EXID, ID, NAME
}
```

The current implementation looks up `UserByLoginName(NAME)` and falls back to
the requesting user when it misses. That can turn a display-name mismatch into
a false successful lookup. Native account refresh then calls
`api.account.getAccount?id=<Blaze/account ID>` (optionally with decks).

**Inference.** Remote creature profiles are not safely representable by the
present `getCreature` implementation. `Creature.ID` is a per-user 32-bit key and
different users can both own creature 1. The HTTP request shown above sends only
that creature ID, while the handler searches only the authenticated viewer's
collection. A remote-owner/global identity contract must be established from
native evidence before broadening this lookup; guessing risks returning the
wrong user's creature or leaking data.

This remote limitation is separate from the universal schema failure: even the
correct local creature returns an object too small for the page parser.

## Persistent and content identity relationships

### Creatures

- `Creature.ID`: per-account persistent instance key; sent as `id` for
  `getCreature` and used by `Part.EquippedToCreatureID`.
- `TemplateCreature.Noun`: immutable content/template identity; emitted as
  `noun_id` in account cache objects and sent as `id` for `getTemplate`.
- `Creature.Template`: in-memory link to the template and source of noun, name,
  element, class, anatomy, template stats, and five ability IDs.
- `creator_id`: expected profile XML owner identity and stored as
  `Creature.CreatorID`. Tutorial starter creation currently leaves it at zero,
  so creation must assign the owning account ID (or the response must use
  verified owner context) before the page's owned-creature comparison can work.

### Parts

- `Part.ID`: per-account persistent inventory key.
- `Part.ReferenceID`: externally stable composite. When absent, grant code forms
  `(uint64(accountID) << 32) | Part.ID`.
- `Part.EquippedToCreatureID`: relationship to `Creature.ID`, not to noun and not
  to `ReferenceID`; serialized as `creature_id` for native inventory records.
- `rigblock_asset_id`: selects the base loot definition and therefore the
  physical/type slot and base restrictions.
- `prefix_asset_id`, `prefix_secondary_asset_id`, `suffix_asset_id`: affix
  content identities that contribute names/stats.
- Persistent `Part` has no slot field. `type_full`, textual rarity, name, PNG,
  restrictions, and derived stats must come from content lookup/enrichment.

The repository normalizes these numeric identities through generated asset
names such as:

```text
_Generated/LootRigblock%d.LootRigblock
_Generated/LootPrefix%d.LootPrefix
_Generated/LootSuffix%d.LootSuffix
```

Content recipe 24 now promotes the raw loot rigblock resources already retained
by `content.db` into a typed `loot_rigblock` read model. The profile adapter
loads its slot, class, and science relationships without adding wire concerns
to the persistent `sporenet.Part`. Affix-derived names and stats remain separate
content work.

## Live-log evidence

Existing `bin/darkspinner/darkspin/logs/darkspinner.log` records a July 20, 2026
tutorial session:

- login `Test` was accepted at 08:22:43;
- RakNet reported player level 2 at 08:23:24;
- part ID 2 was granted at 08:24:20 with reference ID 4294967298 and rigblock
  206; and
- part ID 3 was granted at 08:24:51 with reference ID 4294967299 and rigblock
  206.

For account ID 1, those reference IDs equal `(1 << 32) | 2` and
`(1 << 32) | 3`, directly confirming the persistent/composite identity
relationship. The player-level event is a game/runtime progression value, not a
`Creature.Level` field.

No existing log captures the Hero Profile XML body or a JavaScript exception.
Accordingly, the exact missing-node failure is native-contract plus
current-response evidence and screenshot-consistent inference, not claimed as a
live captured exception.

## Tutorial Blitz/Sage versus all heroes

**Current-server-proven.** Tutorial completion creates Blitz Alpha and Sage
Alpha with `NewCreature` in `server/sporenet/user_features.go:377-404`. Their
instance IDs ordinarily begin at 1 and 2. `NewCreature` at
`server/sporenet/creature.go:186` enforces at least gear score 1 and initializes
item points to 300; it does not create a creature level. Account level and
in-game player level are separate values.

The two tutorial instances can have additional portrait trouble because new
instances do not inherently receive populated `ThumbImageURL` and
`LargeImageURL`. Static template thumbnails exist under the server's template
image route, so a deterministic template fallback is feasible.

**Inference, high confidence.** Every hero is affected by the fatal schema gap:
both current profile handlers omit `name_locale_id` and most subsequent required
nodes. Tutorial heroes are not a special parser case. Hero-specific differences
only become relevant after the common schema is fixed—for example, whether a
hero has equipped parts, complete ability content, valid localized comma-form
name, or instance images.

## Malformed versus missing data

### Missing and fatal

- Instance: `name_locale_id`, `text_locale_id`, `type_a`, `creator_id`, weapon
  damage fields, `class`, stats/ability strings, required empty-or-populated
  `<parts>`, `creature_parts`, five ability selectors, repeated ability metadata,
  and `png_large_url`.
- Template: the same shared header/stat/ability contract, except instance-only
  fields and nested equipped parts.
- Account refresh: `blaze_id` and `name`.

### Present under the wrong contract or name

- `noun_id`, `version`, and `item_points` belong to the account cache object but
  do not satisfy the rich profile parser.
- Template `element_type` must be `type_a` for this page.
- Template `class_type` must be `class` for this page.
- A localized hero name needs the page's comma-delimited title/subtitle form.
- Numeric part rarity must be one-based on these native XML paths; the stored Go
  enum is zero-based.
- `fmt.Sprint` decimal/floating output for existing numeric nodes is parseable;
  numeric formatting is not the first failure.

### Content lookup omissions

- ability locale IDs, icon/content association, and key/value token definitions;
- rigblock-derived slot/type, name, image, class/science eligibility, rarity
  label, base stats, and weapon modifier;
- prefix/secondary-prefix/suffix contributions; and
- deterministic instance/template portrait resolution.

## Prioritized minimal implementation plan

No implementation is performed by this investigation.

### P0 — make local instance and template profiles parse completely

1. Introduce profile-specific XML builders. Do not expand `creatureNode` blindly:
   its current compact schema is consumed by account/deck cache parsers, while
   Hero Profile has a distinct rich contract.
2. Honor `include_parts` and `include_abilities`. Emit every exact node above,
   including an empty `<parts></parts>` for an instance with no equipped parts.
3. Populate shared template data from `TemplateCreature`: localized IDs/name,
   lowercase `type_a`/`class`, weapon bounds, template stats, anatomy-derived
   `creature_parts`, and all five ability IDs.
4. For an instance, add `creator_id`, correctly serialized stats and ability
   keyvalues, gear score, and image URLs. Define delimiter/empty-value behavior
   explicitly; do not serialize Go structs opportunistically.
5. Add a content-owned read port for ability locale/token metadata and loot
   enrichment. The adapter should query authoritative `content.db`; the feature
   response should receive concrete read models rather than exposing SQL rows.
6. Select equipped parts with
   `EquippedToCreatureID == Creature.ID`, enrich each rigblock/affix combination,
   and emit the rich nested-part contract with one-based rarity.
7. Keep deterministic instance portrait fallbacks and the implemented native
   `/game/service/png?template_id=...&size=large` compatibility route covered by
   contract tests.

### P1 — repair identity prerequisites and remote behavior

1. Add `blaze_id` and `name` to the empty-parameter current-account response
   expected by `mainwebview.html`.
2. Determine the native remote-profile owner key at the caller boundary, then
   make creature lookup owner-aware. Add a regression with two accounts that
   both own creature ID 1. Do not fall back to the viewer.
3. Align Blaze `lookupUser` with the name domain the client sends (login versus
   display name), and return a real miss instead of the requesting user.

### P2 — fix the separate inventory contract

1. Parse and honor `count` and semicolon filters for `creature_id` and
   `market_status_full`.
2. Emit `is_flair`, `creature_id`, and `market_status` plus the existing IDs.
3. Convert stored zero-based market/rarity enums to the native one-based XML
   representation.

## July 20 implementation and live verification

The blank owned-profile failure was a client callback-registration race, not a
missing HTTP request. Build 103 invokes the generic SporeNet response vtable at
executable RVA `0x00A57D0`; its vtable entry is at RVA `0x00BD8A6C`. Fang now
intercepts only requests whose callback name is exactly
`spgetcreaturecallback`, retains the response body, and invokes that callback
from a one-shot UI timer after `creatureprofile.html` has assigned it. Client
traces distinguish dispatch, queue, and delivery without retaining profile XML.

The profile projection is no longer a Blitz-only identity path. The content
store resolves each of the 100 `creature_template` rows to its English
localization table, reads all five authored ability asset names, converts those
names to the runtime hashes used by build 103, and joins the indexed Lua string
constants following `localizedName` and `localizedDescription`. Sage and Blitz
both live-render their title, subtitle, element/class, five ability icons,
authored names, and descriptions. Wraith uses the same path; a dedicated live
click remains useful as a regression, not as a separate implementation.

Blitz's identity and damage rows are calibrated to the running build-103 client,
with active damage ranges 4-12, 21-32, 21-35, and 14-35. Its stats now come from the
build-103 type-`0x474940A5` sibling resource: Strength 14, Dexterity 23, Mind
13, Health 220, Power 113, Dodge 288, Resist 128, and Critical 192. The
reference image's 265 Dodge and 115 Resist are from a different balance
revision, as the user anticipated. The damage summary key is `DMG`,
not `ATTD`; `ATTD` localizes to an attribute-value template and visibly
overprints the description. The More Stats tab now receives eight explicit
rows even when an editor save contains only the eight primary stats.

The six dark Item Slot backgrounds are the page's built-in capability display:
weapon, hand, feet, defense, offense, and utility. They are not six equipped
gear records and require no server-created `<part>` nodes. A nested `<part>`
adds only the bright image and tooltip for an item that is actually equipped.
Detail circles likewise remain tied only to equipped flair.

The recovered `buildparts` mapping proves how real equipped gear substitutes
for missing anatomy: a `no_feet` or `no_hands` position uses Offense for
Ravager, Utility for Tempest, or Defense for Sentinel. Build 103 requires
runtime-hashed loot asset IDs in those real part records because `buildparts`
calls the native loot-name resolver before making an equipped image visible;
raw database IDs caused the whole record to be discarded.

A Fang post-callback DOM experiment was live-negative and remains removed.
A later server experiment that fabricated six capability `<part>` records was
also rejected: it rendered bright fake gear and “Equipped Item” tooltips.
Profile responses now contain only persistent, actually equipped parts.
The captured paint-only Blitz save sent an empty `parts` field and
the tutorial claws persisted with `creature_id = 0`, so the absence of an item
overlay was the truthful profile result. The earlier claws equip request predated the real
`updateCreature` handler and was only compatibility-acknowledged. Tutorial
completion now associates an unequipped authored claws reward with Blitz in the
same persistence transaction, including legacy repair, while preserving any
later player choice that equipped it elsewhere. Profile slots continue to read
only persisted equipment; no profile-only entries are fabricated.

The bundled callback dereferences `stats.childNodes[0]` and
`png_key.childNodes[0]` before it makes an equipped slot visible. Empty XML
elements therefore caused a real part to be discarded by the callback. The
projection now supplies a whitespace no-stat sentinel. Equipped parts use their
authored transparent item art rather than the page's generic 20-by-20 category
icons. `PrepareContent` scans the build-103 rigblock property resources and
recovers 1,513 ordinary and 775 unique image references spanning rigblock IDs 1
through 1,573 and 10,001 through 10,835, then
resolves their case-insensitive FNV-1 filename stems against `UI.package` and
extracts them to deterministic `assets/images/loot/<rigblock-id>.png` paths.
Content recipe 28 persists all 2,288 definitions in `loot_rigblock`,
including their authored profile slot and comma-separated class/science
allowlists. The ordinary corpus contains 65 weapon, 83 grasper, 79 foot, 940
defense, 175 offense, and 171 utility definitions. The server loads this immutable
catalog at startup, so profile projection no longer contains an Electro
Claws-only type/restriction branch. For example, rigblock 206 is independently
projected as utility, Ravager/Sentinel/Tempest-compatible, and Cyber. Electro
Claws come from build-103 rigblock resource
`0x1BCED3D7:0:0x646569E2` names
`0x100d977c!ce_weapon_lightningRavager_02-symmetric.png`; case-insensitive
FNV-1 resolves the filename stem to instance `0xC062FBA2`, whose PNG resource
is in `UI.package` and becomes `assets/images/loot/268.png`. The profile emits
the server's absolute loopback URL for every nonzero persisted rigblock ID. A
relative key resolves inside the client's packaged Web context
and produces a broken-image placeholder. The absolute URL preserves the native
aspect ratio and alpha inside the octagonal slot without inventing affix stats.

The adjacent affix corpus is measured and projected through the recovered
native transform.
`AssetData_Binary.package` contains 338 prefix resources of type `0x6A1812C6`
and 328 suffix resources of type `0x447DC2E5`: 83 ordinary suffixes plus 245
unique suffixes. Build-103 `Game.c` selection functions prove prefix
minimum/maximum levels at runtime-structure offsets 28/32 with class/science
masks at 548/556; suffix selection uses levels 20/24, category bytes 52/53,
and class/science masks 568/576. Prefix identity is the little-endian ID at
payload offset 0; suffix identity is its ordinary/unique `Loot*SuffixNames`
marker. A complete installed-client corpus check resolves all 338 prefix IDs
and all 328 suffix IDs. Recipe 28 stores them in `loot_affix` with source
provenance, exact level range, class/science allowlists, and the 460-byte
115-float modifier vector beginning at payload offset 36.

The modifier vector is an input vector rather than the final profile stat
array. Native `sub_9CA290` adds most entries directly, but deliberately excludes
indices 0, 1, 2, 4, 5, 7, 9, 10, 102, 103, 104, 105, and 108. Native
`sub_9C9D50` transforms those entries using rarity tuning, item-level scaling,
positive modifier-dimension count, and per-stat divisors before placing them in
different output indices. Recipe 28 stores the exact 1,212-byte
`_generated/lootpreferences1.lootpreferences` projection in `loot_tuning`: a
base point of 50, a 0.2 extra-dimension factor, five exponential level bands,
13 point costs, four rarity block distributions, and authored base-slot
operands. The server applies the native suffix/prefix/prefix2/standard block
order and formats the recovered `unk_115E548` output tokens as
`TOKEN,amount,0;`. For example, build-103 prefix 61 has modifier 2 at input index 52;
suffix 44 has modifier 50 at input index 5 and is eligible at levels 31-1000.
The executable's ordered attribute strings independently recover indices 0-113
(`Strength` through `BodyScale`, including the previously missed
`DistributeDamageAmongSquad`, `ImmuneToTaunted`, and `ImmuneToShocked`); 114 is
the terminal enum slot. Profile `stats` now includes affix results and every
standard contribution that maps to a profile token. Native `sub_9CB150` does
not inspect the viewed hero for weapon anatomy: it resolves the first player
noun embedded in the rigblock. Recipe 28 stores that noun relationship for all
130 ordinary and unique weapon definitions, joins it to the authored hand/foot
flags, and reproduces the `(level - rarity offset) / 5` gate against thresholds
17 and 9. Electro Claws resolve to `PC_EL_Rogue.Noun` and the hand path; their
tutorial level 5 is below the threshold, so the client also contributes no base
weapon operand there. The 138-word block has three state words before its 115
modifiers; subtracting that header aligns the standard writes with the recovered
attribute enum and corrects defense/offense tuning semantics to 75/22.

Native `sub_9C9D50` retains the same header in its destination block: physical
writes `a2[3]`, `a2[4]`, and `a2[5]` are logical Strength, Dexterity, and Mind,
not logical indices 3, 4, and 5. The server's compact 115-float array has no
header, so all 13 rarity-scaled destinations must be reduced by three. Applying
that correction keeps each transformed result on its source attribute: defense
standard points become Health, offense becomes Critical Rating, utility becomes
Mana, and Dexterity no longer appears as Health. Directly accumulated modifiers
already used header-free logical indices and do not move.

Weapon damage uses a separate, fully recovered path. Native `sub_9CB0B0` first
requires the rigblock weapon bit, normalizes level through `sub_9C9B30`, applies
the same five-band `sub_9CAF10` curve, and multiplies it by LootPreferences float
index 221 (`0.588235`). This result is the profile
`weapon_damage_modifier`. The bundled `creatureprofile.html` multiplies both the
hero's authored `weapon_min_damage` and `weapon_max_damage` by the field, falls
back to one only for a nonpositive field, and truncates both results with
`parseInt` before presenting `Base Damage minimum - maximum`. Gameplay now uses
that same equipped-weapon projection for content definitions whose damage range
is inherited from the hero's weapon. The authenticated creature snapshot keeps
this provenance separate from attributes 101 and 102, which are Deploy Bonus
Invincibility Time and Physical Damage Decrease Flat rather than weapon bounds.

All 100 templates now import authored player-class stats rather than receiving
zero-valued fallbacks when no editor snapshot exists. The paired 88-byte stat
resource is joined by the player-class base-name instance ID; recipe 22 derives
Health, Power, Dodge, Resist, and Critical using the recovered build-103
formulas. Recipe 22 also evaluates deterministic instructions in each Lua main
prototype and stores numeric top-level ability properties in
`lua_static_property`. Each row distinguishes a scalar constant, the first
ranked constant, or the first ranked range; calls and runtime-dependent
expressions remain unresolved. The generic profile path converts an authored
`damage` range to `minDamage`/`maxDamage`, emits other proven scalar/range
tokens, and only then fills missing client-required tokens with zero.

Recipe 22 follows the main prototype's `TranslateToken` closure assignment and
indexes branches that reduce to a ranked property, an optional range element,
and an optional constant multiplier. This recovers semantic mappings that raw
property names cannot express. Examples verified in the rebuilt build-103
database include Lightning Ball's `minSecondaryDamage`/`maxSecondaryDamage`
mapping to `lightningDamage`, and Lightning Rogue Support's
`deflectionIncrease` mapping to `energyDefensePercent * 100`, `numOrbs` mapping
to `objectCount`, and `attackSpeedIncrease * 100`. Template-derived tables made
by `Template:new()` receive an empty symbolic shell only after assignment to an
`nAbility_` or `nModifier_` global, allowing subsequent constant writes to be
projected without assuming anything about the factory call itself. Non-range
numeric indexes and other unresolved expressions are rejected conservatively.

Build-103 native code also explains the difference between authored damage
ranges and the numbers written by `api.creature.updateCreature`. The profile
detokenizer at `sub_43A170` classifies `minDamage`, `maxDamage`,
`minSecondaryDamage`, and `maxSecondaryDamage` by their lowercase FNV hashes,
routes them through `sub_9E5B10`, floors minimum tokens, and ceilings maximum
tokens. `sub_9E5B10` calls `sub_9E4E60`, which applies the ability's authored
damage coefficient to the class-selected primary attribute relative to the
build's `-1` base. The resulting profile projection is
`raw * (1 + (primary + 1) * coefficient)`. Ravager, Sentinel, and Tempest select
Strength, Dexterity, and Mind respectively. Blitz's Strength 14 and coefficient
0.05 therefore produce the observed 1.75 multiplier: 12-18 becomes 21-32,
8-20 becomes 14-35, and Lightning Ball's 4-10 secondary range becomes 7-18.
Token bindings retain their source property name so secondary damage selects
`lightningDamageCoefficient` rather than the primary `damageCoefficient`.
The source table is retained as well: a chunk can define several abilities or
modifiers with identically named properties, so matching coefficients by the
Lua chunk alone is ambiguous. The build-103 hash switch additionally proves
the same primary-attribute projection for `petMinDamage`, `petMaxDamage`, and
single-value `damage`, while `sub_9E5550` applies `healingCoefficient` to
`minHealing`, `maxHealing`, and `healing`. The profile projection now follows
the client's minimum-damage floor, maximum-damage ceiling, and healing
truncation rules for those families. Recipe 22 currently contains 314 explicit
`TranslateToken` bindings spanning 116 distinct token names.

A full recipe-28 join audit, without the database command's 1,000-row display
limit, compares all 314 bindings with all 4,502 static properties. Exactly 311
resolve. The three unresolved references are not unsupported constant
arithmetic: their defining chunks never assign a numeric value to the named
field. They are `nModifier_LightningRogueActive.duration`,
`nAbility_SoulRavager_Basic.damagePerSoul`, and
`nAbility_PlasmaRandom_LightningBall.duration`. The first is referenced through
native modifier registration; the latter two depend on state absent from their
defining chunks. They must not be materialized as zero or guessed constants.

Build-103 `sub_9E5B10` confirms that damage detokenization begins with the
current creature modifier aggregate. Its call to `sub_9E4E60` reads the
class-primary modifier index, subtracts the build tuning baseline, and applies
the authored ability coefficient before the later descriptor, weapon, flat,
and active-modifier stages. Exact editor-saved ability tokens remain
authoritative. When those tokens are absent, the profile fallback now applies
the coefficient to the creature instance's saved current primary stat rather
than always using the immutable template stat. The remaining native stages are
not collapsed into that partial projection until their descriptor flags and
modifier-index semantics are typed.

The retained original attribute enum independently names the hidden indices in
the next native stage: 13 is `DamageBuff`, 16 is
`DefenseBoostBasicDamage`, 19 is `AutoCrit`, 22 is
`CriticalDamageIncrease`, 108 is `DirectAttackDamage`, and 109 is
`DirectAttackDamagePercent`. Build-103 `sub_9E4F40` initializes its multiplier
to `1`, adds attribute 13 unconditionally, and only then enters descriptor-
gated branches for damage type, AoE, DoT, physical/energy source, and direct
attacks. The profile fallback now aggregates the transformed logical attribute
13 from persistent equipped parts and applies `1 + DamageBuff` after the
primary/coefficient term but before minimum-floor and maximum-ceiling. Exact
editor-saved ability tokens still bypass this derivation. Descriptor-gated
attributes remain disabled whenever their authored mask is unavailable.

Recipe 30 completes the typed operands for that stage. `GlobalDefinitions.lua`
and the retained build-103 header agree that `nDamageSources` is Physical `0`,
Energy `1`, while `nDamageTypes` is Technology `0`, Spacetime `1`, Life `2`,
Elements `3`, Supernatural `4`, and Generic `5`. The conservative evaluator now
seeds those two tables just as it seeds `nDescriptors`; ordinary `GETGLOBAL` /
`GETTABLE` bytecode therefore materializes numeric `damageType` and
`damageSource` properties without executing a chunk. The verified runtime
database contains 514 `descriptors`, 342 `damageType`, and 342 `damageSource`
rows. Lightning Rogue Active is a complete example: physical source `0`,
Elements type `3`, descriptors `65` (melee plus physical), and authored damage
`12-18` with coefficient `0.05` all share one source table.

The generic profile fallback now follows native `sub_9E4F40` in its exact
additive order. A basic hit begins at
`1 + DefenseBoostBasicDamage * (EnergyDefense + PhysicalDefense)`; all paths
then add DamageBuff, the selected science-type damage attribute, projectile
damage for descriptor `0x2000`, EnergyDamageBuff for source `1`, AoEDamage for
descriptor `0x8`, and either DoTDamageDoneIncrease (`0x4`) or
DirectAttackDamagePercent. Physical and energy descriptors add their general
done-increase attribute and, for non-basic abilities, their ability-only
increase. Native `sub_9E4EF0` then adds DirectAttackDamage unless the ability is
DoT (`0x4`) or HoT (`0x1000`). Missing authored descriptor/type/source metadata
leaves the corresponding gated branch disabled. The evaluator now lives in the
game feature as a typed pure operation shared by profile rendering and future
combat authority rather than remaining page-only arithmetic. Gameplay
authorization snapshots the selected hero's current class-primary stat plus
every equipped-item operand consumed by that evaluator. Voltic Slash, Sage's
basic projectile, and Sphere of Transfusion now consume that authenticated
damage snapshot before range and critical resolution. Tree of Life consumes a
parallel healing snapshot containing the primary stat and native modifier
indices 96/98 after selecting its raw authored pulse range. Incoming tutorial
enemy attacks consume a target-owned defense snapshot containing native
modifier index 30 after attacker range and critical resolution. A timing
snapshot additionally carries AttackSpeed index 23 and CooldownReduction index
24; the latter now drives Sphere and Tree's live cooldown gates and packets.
The constrained runtime compiler now uses the same
numeric build-103 descriptor, damage-type, and damage-source enums as the static
content importer, evaluates numeric `nBit.Or`, and retains those optional fields
plus `damageCoefficient` on every typed ability definition. Captured
compatibility results such as Ride's
observed 18 damage remain unchanged until range sampling and the later
weapon/target stages are connected in order.

The bundled page places Abilities before Hero Backstory and calls
`switchtoability()` from `reset()`. That is explicit build-103 behavior, not a
server ordering defect. Fang therefore preserves Abilities as the first/default
tab and Backstory as the second tab; its deferred callback only solves the
page-registration race.

An actual July 20 paint save established the instance wire values that the
client computes. In addition to the four visible ranges, the payload contains a
3-second stun, 7-18 secondary damage, radius 4, six orbs, 100 deflection, and a
50-percent passive. `api.creature.updateCreature` already decoded these fields,
but the SQLite adapter discarded them on the next repository save. Ordered
`creature_stat` and `creature_ability_stat` child tables now preserve the exact
editor payload. A full Game/Darkspinner process restart retained all eight
stat rows and all 15 ability-token rows.

## Minimal test plan

1. Add exact XML/DOM fixture tests for `getCreature` and `getTemplate`. Assert
   every required node by name and nesting, not only status 200.
2. Add a contract regression that evaluates the bundled callback (or a small
   maintained required-node manifest derived from it) against those fixtures;
   deleting `name_locale_id`, `<parts>`, or an ability node must fail.
3. Table-test Blitz Alpha, Sage Alpha, and a non-tutorial hero. The same shared
   header/ability/stat assertions must pass for all three.
4. Cover no equipped parts, one normal part, one flair part, and duplicate-slot
   anatomy remapping for `all`, `no_feet`, and `no_hands`.
5. Assert number formats: base-10 IDs, parseable decimal weapon/stat values,
   semicolon/comma/exclamation ability delimiters, and one-based wire enums.
6. Test image behavior for empty instance image fields and the exact template
   compatibility URL.
7. Test current-account `blaze_id`/`name`, Blaze real-miss behavior, and two-user
   creature-ID collision isolation.
8. Keep separate inventory tests separate from Hero Profile tests so a part-list
   fix cannot mask a missing nested profile part contract.

These tests are handler/content-adapter tests and do not require launching the
game client.

## Account menu View Profile

This is a separate page and API contract from the creature Hero Profile above.
The build-103 `Web.package` page calls `api.account.getAccount` with
`include_feed=true`, `include_decks=true`, `include_creatures=true`, and
`include_stats=true`, then passes the response to
`spgetplayerprofilecallback`.

The exact recovered page is `Web.package` ordinal 118, resource
`000118_dd6233d6_00000000_00000000a072f231.bin` (type `0xDD6233D6`, group
`0x00000000`, instance `0xA072F231`). Its callback does not use the returned
`avatar_id` as an image URL. It sets the portrait source to
`/game/service/png?account_id=<blaze_id>`. The server therefore resolves the
active account's persisted avatar ID and serves the corresponding early-cache
portrait. IDs 1 through 15 are registration choices; ID 0 is retained only as
the missing-account or invalid-persisted-ID fallback. The same compatibility
handler also accepts the page's `template_id` form and serves the cached hero
thumbnail.

The page directly consumes the response `blaze_id`; account identity,
progression, XP, unlock, and star fields; localized creature names and
thumbnail art; each deck's name and creature identity/class/science/gear/art;
the feed collection; and the lifetime statistics object, including mandatory
`wins`. The server now projects all of these nodes for the authenticated local
account. A typed, one-to-one `user_stat` row durably stores lifetime PVE/PVP
counters; ratios remain derived at the profile projection boundary. PVE session
time, accepted damage dealt and taken, minion/lieutenant kills, hero deaths,
and actual capped healing are connected to authoritative tutorial outcomes.
Boss and PVP counters remain zero until those authoritative gameplay paths
exist. `pve_xp` and `pve_progression` come from the persisted account. The
account contract regression names every PVE and PVP field accepted by the
recovered page and requires a nonempty text child for each one. The feed is
currently an empty collection.

The local request does not name another account. Remote Account Profile now
honors the page's recovered `id=<Blaze ID>` and `name=<display name>` forms for
authenticated active accounts. Resolution is exact and fails closed for a
missing or ambiguous name. Its explicit public projection contains only the
fields consumed by this page plus the already-public creature/deck/stat
builders; it omits authentication tokens and cookies, settings, currency,
entitlements, and live game/playgroup identifiers. Offline lookup remains
disabled until storage has a purpose-built public read operation. Remote Hero
Profile also remains disabled because its request still lacks a proven owner
key for account-local creature instance IDs.

The first live test still remained on the page's loading state despite the
complete HTTP response. The response arrived before
`spgetplayerprofilecallback` was registered, the same embedded-page race
already proven for `spgetcreaturecallback`. Fang now recognizes both callback
names, percent-encodes the returned XML, and invokes the matching callback from
the request's owning web-script host on a one-shot UI timer. After rebuilding,
the live page rendered Test, Crogenitor level 3, XP 335, four activated heroes,
progression 1-1, zero PvP wins, and Blitz's squad card. The spinner in the Helix
Log pane is the separately empty feed collection, not a blocked profile load.

The single-card result was a second page-level abort rather than a deck-storage
failure. Test's persisted active squad contained three owned creature IDs, but
only the edited Blitz instance had nonempty custom `png_large_url` and
`png_thumb_url` fields. `builddeck` writes each card in order and unconditionally
dereferences both image elements; the empty Sage image element therefore threw
after Blitz was drawn and prevented the remaining cards from being processed.
Account-profile projection now preserves an instance's custom image URL and
otherwise supplies the cached template-image service URL for both image fields.
A three-creature contract fixture covers the active deck with two unedited
instances, proving that all cards retain nonempty art and reach the response.
The page always renders `deck[0]`, so the response now orders the persisted
`default_deck_pve_id` first while retaining every other deck afterward. This
keeps View Profile aligned with the squad selected in ship management even when
its persistent ID is not the first storage row.
