# Content database inventory

This note tracks permanent game-content data normalized into the runtime
`bin/server/darkspin/cache/content.db` and optional
`bin/server/darkspin/cache/meta.db`, the
evidence used to identify it, and the Go schema/import work that remains.
Player-owned rolls and inventory state belong in `user.db`; minimum immutable
gameplay definitions belong in `content.db`; provenance, research indexes, and
client-oriented metadata belong in `meta.db`.

## Level and Lua navigation tree

The runtime content builder now produces this direct navigation tree:

```text
chain_level ──> level

level
  ├─ level_alias
  ├─ level_marker_set
  │    └─ marker
  │         └─ level_event
  ├─ level_director_entry
  └─ level_script
       └─ lua_chunk
            ├─ lua_string_constant
            └─ lua_dependency
```

`level_script` is a many-to-many join. A Lua chunk is package content in its
own right and can serve more than one callback or level, so `lua_chunk` does
not carry a single `level_id`.

| Table | Meaning |
| --- | --- |
| `level` | One row for each of the 61 build-103 `Level` assets. It stores the authored asset name, the matching `Levels.package` group ID, core music/navigation/physics/rendering/planet fields, primary and secondary type, camera overrides, exact AssetData resource identity, and a zlib-compressed copy of the decoded source asset with size and SHA-256. |
| `level_navigation` | The level's decoded type-`0x1999AE0B` BFX navigation image from `Levels.package`, retained with exact DBPF identity, decoded size/hash, and zlib-compressed payload. Campaign setup parses this focused row rather than copying or scanning the full 217 MB package at runtime. Build 103 supplies 59 rows; the non-campaign `test_AI_zoo_ugc` and `test_holodeck` assets are the two authored levels without a BFX resource. |
| `level_alias` | Accepted names for resolving a level without duplicating it. Every level has bare-name and `.Level` aliases. The client-facing `Game_Tutorial_cryos_1_v2` and `.Level` forms resolve to the authored `Game_Tutorial_cryos_1` row. |
| `chain_level` | All 72 zero-based ordered campaign positions from `AssetData_Binary.package` resource type `0xA8A25294`, group `0`, instance `0x304F6F19` (`ChainLevels`). Each row retains the exact `.Level` reference, links to the common source-resource row, and resolves through `level_alias` to `level.id`. The three 24-position passes intentionally reuse each campaign level exactly three times. Labels such as `1-1` are computed from ordinal rather than stored. |
| `crystal_definition` | All 192 authored entries from `AssetData_Binary.package` resource type `0x20426A63`, group `0`, instance `0xF1D031EA` (`CrystalTuning`). Each zero-based row retains the inclusive crystal-level bounds, signed selection weight, serialized noun relocation word, exact `.Noun` text, and source-resource provenance. The native selector passes the separately computed crystal level—not raw encounter difficulty—through these bounds. The decoded build-103 resource is pinned to SHA-256 `1b8869a6a260a01847d3a1b3c619744db48ed48adad391ebb7fb8e5fa3b7431d`; runtime readers return authored ordinal order. |
| `crystal_level_offset` | The ordered weighted offsets applied after the executable converts encounter difficulty to crystal level. Build 103 authors exactly one entry, offset `0` with weight `1.0`; it leaves the level deterministic while still consuming the native RNG draw. |
| `level_marker_set` | Ordered marker-set references authored by a level. The row links to the exact `AssetData_Binary.package` resource when present and retains group, weight, decoded size/hash, and compressed decoded payload. A missing resource remains an ordered reference with nullable source columns instead of disappearing. |
| `marker` | Queryable marker identity and transform: authored marker ID/name, noun, position, rotation, scale, dimensions, visibility, collision flag, and owning ordered marker set. Marker records begin at `0x28` with a `0xC8` stride; the header count must exactly match the string-owned marker boundaries. Immediately adjacent marker-name/noun strings form one authored alias, while noun strings separated by binary metadata begin separate generic markers. Recipe 35 additionally normalizes an interactable ability plus its signed use limit immediately after the noun terminator and challenge immediately before the ability string; build-103 tutorial fixtures prove the loot and health-obelisk layouts. The complete marker-set payload remains authoritative for component fields not yet projected. |
| `level_event` | Conservative event/callback pairs recovered from a marker's shared-component data. It records the owning marker, component, event/callback names, and currently known classification. Dotted Lua trigger locators retain `triggerVolume` / `luaCallbackOnEnter` plus the authored radius, `is_trigger_once_only`, and `is_server_only` policy. Recipe 34 also recognizes the native `HordeTrigger_OnEnterPlayer` record layout, pairing the callback with its following authored enter event and retaining the same radius/flags; raw 1-1 fixtures prove radii `15` and `5`. Unknown component slots remain `unknown` rather than receiving an invented type. Linking remains authorship evidence and does not assert that the inert build-103 callback publishes the event; server execution still requires explicit scenario policy. |
| `level_director_entry` | Difficulty-bounded noun entries recovered from the level's director configuration, including the horde flag, authored global order, and lossless contiguous configuration/entry ordinals. Noun/difficulty and structural pool boundaries are decoded now. Build-103 reflection and instruction evidence labels `zelems_1` configurations `0..3` as `minion`, `special`, `agent`, and `captain`; other levels and every entry-level `spawn_kind` remain `unknown` until separately proven. |
| `lua_chunk` | One row for each of the 1,029 native Lua 5.1 chunks in `ServerData.package`, linked to `server_data` and recording source identity, bytecode size, and SHA-256. The executable bytecode itself remains in `server_data.decoded_payload`. |
| `lua_string_constant` | Every decoded Lua string constant in flattened chunk/prototype order. Exact constants provide a content-owned lookup for registered ability names, assets, events, and module identities without hard-coding package hashes in the server. Ordinals preserve decoder order and are not interpreted as source-line order. |
| `lua_dependency` | Ordered package/module-looking string references found in each chunk, with a nullable exact `target_lua_chunk_id` when the source identity resolves uniquely. An unresolved dependency stays visible instead of being dropped. |
| `level_script` | Exact event-to-chunk links. Standalone callbacks use their packaged string constant. A dotted `<module>.<callback>` locator links only when one Lua chunk contains both exact, case-sensitive constants; zero or multiple same-chunk matches remain unresolved instead of fanning out through common names such as `main`. A callback with no matching chunk remains in `level_event`; this is important for native callbacks such as `DirectorTrigger_SpawnBoss` and `HordeSpawner_Register`. Runtime projection carries the owning marker-set ordinal, asset name, and weight directly, including script-only sets absent from director-placement rows. |

The two packages have distinct jobs. `Levels.package` contains the client's
compiled geometry/physics stream (62,232 resources grouped under the 61 level
IDs); it is recorded as a required input and supplies `level.package_group_id`.
It is not the authored server-event source. The semantic level and marker-set
assets are types `0xB9193960` and `0xA11D3144` in
`AssetData_Binary.package`, and those are what populate the tree. Lua bytecode
continues to come from `ServerData.package`.

This makes `content.db` self-contained for the level/marker source data and Lua
bytecode needed by a future server content loader, but it does not by itself
make every client-requested situation executable. The remaining boundary is
code: server level loading still needs to consume these tables, the remaining
native marker component layouts and director-kind fields need decoding, and
the Go Lua simulator/native bindings must turn linked chunks into validated
typed intents. Keeping the compressed source payloads in the same database
means those decoders can be improved without restoring a loose XML tree or
consulting another repository.

For `Game_Tutorial_cryos_1` (`level.id=9`), the Audio marker set now owns
five exact trigger locators and links them uniquely to chunks `141`, `930`,
`458`, `619`, and `215`. Together with the ten existing obelisk links, the
level has exactly fifteen `level_script` rows. The two client-group chunks are
indexed as authored content but are not thereby authorized for server-side
execution.

## Database split

The comprehensive database has been split into two build-103 projections with
the shared release identity `build-103-content-v1`:

| Database | Size | Role |
| --- | ---: | --- |
| `content.db` | 67,194,880 bytes | Required runtime projection: normalized gameplay/localization/catalog tables, runtime XML payloads not yet fully typed, all 1,029 original Lua 5.1 bytecode members, and their 1,029 mapped source representations intended for constrained inspection/simulation. |
| `meta.db` | 107,520,000 bytes | Optional self-contained research projection: the complete pre-split database, original packages, generated decompilations, generic property index, relationship graph, package analysis, and catalog-gap evidence. |

Both databases contain `database_manifest`, recording role, source build,
release identity, and runtime requirement. Both pass `PRAGMA integrity_check`
and have no `PRAGMA foreign_key_check` violations. All 11,518 payloads retained
by `content.db` were decompressed and revalidated by length and SHA-256.

`meta.db` deliberately remains a self-contained superset instead of requiring
cross-database joins back to `content.db`. This duplicates the runtime core only
in the optional research distribution, while the normal server ships only the
35.1 MB runtime database.

## Imported database contents

Together the two projections contain the following normalized lookup and
lossless research tables. `content_asset_property`, `content_asset_reference`,
`asset_catalog_tag`, `asset_catalog_gap`, `compiled_animation_resource`,
`package_inventory`, `package_resource_type`, `script_definition_property`,
`script_asset_reference`, and `script_localization_reference` are metadata-only.
The other tables below remain in the runtime projection and are mirrored in the
self-contained metadata projection.

| Table | Rows | Purpose |
| --- | ---: | --- |
| `localization_text` | 50,661 | German, English, French, Polish, and Russian entries extracted from the installed build-103 text packages, including raw localization table and key. |
| `loot_rigblock` | 2,408 | Every extracted `LootRigblock` definition: stable rigblock ID, package asset key/ID, localized name, part type, raw content flags, level range, prop/PNG/category keys, and unique flag; recipe 51 uses the flags for native flair eligibility. |
| `loot_rigblock_class_type` | 6,724 | Allowed Ravager, Sentinel, and Tempest relationships. |
| `loot_rigblock_science_type` | 5,248 | Allowed Bio, Chrono, Cyber, Necro, and Plasma relationships. |
| `loot_rigblock_player_character` | 1,000 | Optional noun-specific restrictions. |
| `loot_affix` | 666 | All 338 prefixes and 328 suffixes with IDs, level ranges, optional names, and unique/basic flags. |
| `loot_affix_requirement` | 5,622 | Affix class, science, and part-type eligibility. |
| `loot_affix_modifier` | 1,796 | Effective nonzero numeric modifiers with their original XML ordinal. Explicit zeroes remain in the lossless/property representations. |
| `loot_affix_effect` | 24 | Ability improvements and granted modifiers. |
| `weapon_tuning` | 381 | Authored item-level, rigblock, suffix, cost, avatar-level, and chain-progression tuples; runtime recipe 50 imports the exact packaged `WeaponTuning.WeaponTuning` rows and uses them as vendor purchase authority. |
| `combat_tuning` / `difficulty_tuning` | 1 / 72 | Recipe 39 retains the build-103 critical-damage base, health/damage party bases, and every difficulty-indexed health multiplier, damage multiplier, expected avatar level, and critical-rating conversion with exact source-resource provenance. Typed simulation reproduces the proven row-and-power projection while leaving party-exponent ownership and captain/elite stages unresolved. |
| `creature_template` | 100 | Creature identity, localization, genetic/class type, base damage, body-slot flags, ability IDs, and build-103 player-facing stat templates decoded from the paired type-`0x474940A5` resource. The imported primary attributes and derived Health, Power, Dodge, Resist, and Critical ratings use the authored class fields and recovered build-103 formulas. |
| `non_player_class` | Build-derived | Runtime combat projection for every fixed 88-byte type-`0x474940A5` class-stat record in `AssetData_Binary.package`. Rows retain the exact `content_source_resource` foreign key and base instance ID; recipe 32 stores authored health, power, Strength, Dexterity, and Mind plus the native-backed derived Dodge, Resist, and Critical ratings. Zero is preserved for authored noncombat fixtures; fields outside the proven stat indices remain uninterpreted in the authoritative source resource. |
| `noun_physics` | 13 | Exact build-103 projection for the five tutorial enemy nouns, Blitz, Sage, `HelperMelee.Noun`, `Ability_Fireball.Noun`, and the four placed/dropped health and mana orb nouns. It retains source provenance, authored lifetime, graphics scale, normalized footprint radius, local bounds, geometry/property references, decoded hash, and compressed source payload. Dropped orbs have a 30-second lifetime; placed capsules have lifetime zero. The five enemy rows also link their distinct type-`0xD117AFCA` `ClassAttributes` source and project its proven `creature_type` field at offset `4`; Poison/Diseased are Life (`2`) and Ranged/Sloth/Special One are Elements (`3`). Recipe 13 additionally links each enemy to its type-`0x17BBCE29` `CharacterAnimation` source and projects the first ordinary death state. Blitz preserves graphics scale `1.6` separately from its base-sphere-derived footprint `0.8`; Sage similarly projects `1.65` and `0.825`. Recipe 15 adds both the four orb contracts and Sage's deployed collision footprint without assigning behavior to unproven fields. Recipe 16 links the hero animation resources too and projects their exact ordinary-death and noun-selected dance states: Blitz `emote_dance_carlton`, Sage `emote_dance_runman_gun_r`. |
| `noun_physics_shape` | 5 | Proven inline Fireball projectile-query box (`1 x 1 x 3`) plus the four orb pickup-trigger boxes, all linked to their source resources. Placed health/mana capsules use `1 x 1 x 1`; dropped variants use `2 x 2 x 4`. Runtime derives half-extents from these rows and keeps the authored shape separate from the entering actor's collision footprint. Unmapped inline floats are not assigned speculative meanings. |
| `creature_part_template` | 2,408 | Lossless structured form of the legacy creature-part JSON rows, keyed to `loot_rigblock`. |
| `content_asset` | 11,518 runtime / 18,052 metadata | Runtime-required XML, original ServerData members, and executable source representations in `content.db`; every imported source/package/decompilation/animation payload in `meta.db`. |
| `content_asset_property` | 429,154 | Searchable leaf properties for every non-marker XML asset and the asset catalog. Paths and ordinals preserve repeated fields. |
| `asset_catalog_entry` | 13,697 | Symbolic asset name/hash/source mappings from `assets_catalog.xml`, linked to stored content where available. |
| `asset_catalog_tag` | 5,002 | Distinct authored tag relationships from the asset catalog. |
| `content_asset_reference` | 10,630 | XML relationship edges to catalog assets, payloads, registered ability/modifier definitions, and exact compiled animation resources. |
| `creature_content_link` | 100 | One-to-one creature-template to noun/catalog/content links established by build-103 asset hash. |
| `server_data` | 1,101 | Original `ServerData.package` inventory with runtime mappings from all 1,029 compiled Lua members to source representations; metadata adds the static-analysis tables. |
| `script_definition` | 980 | 479 registered abilities and 501 registered modifiers recovered from local bytecode. |
| `script_definition_property` | 10,362 | Directly authored static fields retained as typed scalar/raw expressions. |
| `script_asset_reference` | 2,361 | Decompiled-script references to package scripts, catalog assets, animations, and registered definitions. |
| `script_localization_reference` | 614 | Script localization GUIDs linked to their localization table/key; 608 resolve across all five installed locales. |
| `tuning_scalar` | 87 | Typed numeric loot, difficulty, crystal, and core combat parameters. |
| `tuning_series_value` | 538 | Ordered affix, rarity, difficulty, and encounter-director curve values. |
| `tuning_range_value` | 113 | Ordered item-level and gear-score ranges, including the authored gear-score maximum range. |
| `chain_level` | 72 | Ordered campaign level references decoded directly from the authoritative `ChainLevels` package resource and resolved to stored level rows. Unprojected fields remain authoritative in the retained raw resource. |
| `level_objective_entry` | 19 | Named objective and positive/minor/major level-affix pool entries. |
| `level_spawn_entry` | 424 | Difficulty-bounded minion, special, boss, and agent noun pools with horde eligibility and content targets. |
| `phase_definition` | 171 | AI phase type, gambit/start state, raw gambit identity, and optional registered-script target. |
| `unlock_definition` | 48 | Authored unlock identity, prerequisite/cost/level/rank/type/value/function data, and presentation references. |
| `crystal_level_probability` | 1 | Authored crystal-level offset and probability entry. |
| `crystal_definition` | 192 | Ordered weighted crystal noun choices projected directly from the authoritative build-103 `CrystalTuning` binary, with proven inclusive crystal-level bounds. |
| `elite_npc_level_tuning` | 73 | Per-level normal/special elite affix bounds and spawn chances. |
| `ai_definition` | 232 | Stable AI identity, aggro/cooldown/start tuning, and control flags. |
| `ai_node` | 183 | Authored AI graph nodes with coordinates/output and optional phase content targets. |
| `ai_ability_reference` | 364 | Named AI ability roles with 363 registered-script targets and one retained malformed source value. |
| `npc_affix_definition` | 34 | NPC affix identity, modifier script, parent/child relationships, and localized description key. |
| `package_inventory` | 2 | Checksummed package inventory for `CompiledAnimData.package` and `Levels.package`, including resource/compression counts and payload-storage status. |
| `package_resource_type` | 8 | Resource-type count summaries for the two classified packages. |
| `compiled_animation_resource` | 2,407 | All 2,340 compiled animations, 65 gaits, and two auxiliary resources with package group, format, ID where present, and binary magic. |
| `asset_catalog_gap` | 286 | Evidence and confidence classification for every catalog identity without a standalone stored payload. |

Database verification does not rely on the Lua row totals alone. Every
`server_data.is_compiled_lua=1` resource must map to exactly one `lua_chunk`, no
chunk may target a resource classified as non-Lua, and `bytecode_size` must
equal the retained decoded payload size. This catches count-preserving mapping
or classification swaps while keeping the package payload authoritative.

The initial item import source was `bin/server/data/lootrigblock`. Localization
was extracted to the required `bin/game/logs` diagnostics area from each
installed `Text.package`, then normalized into both projections. The original
five package files are preserved only in `meta.db`. The opaque locale asset
`0x266C1C25` is identified as `LootUniqueRigblockNames` because all 2,408
rigblock name references resolve exactly after that mapping.

SQLite `PRAGMA integrity_check` returns `ok`, and `PRAGMA foreign_key_check`
returns no violations after the import.

The `meta.db` lossless bundle represents 278,453,570 source/decompiled bytes. Its
compressed payload is 31,531,881 bytes. Every original stored payload was
decompressed after insertion and
its length and SHA-256 were compared with the original file. Marker sets are
preserved as compressed source assets but are not property-expanded because
their 185 MB of XML would produce a disproportionately large secondary index.
All 11,018 non-marker XML files were successfully property-indexed.

The property table was rebuilt after measuring SQLite page usage. The unused
leaf-attribute column was removed (all 383,005 original rows contained `{}`),
the 16.4 MB full-path index was dropped, and the redundant unique index was
replaced by an asset/ordinal lookup index. The database fell from about 90 MB
to 77 MB before the multilingual and catalog additions, without losing payloads
or property rows. Full paths remain stored and can still be filtered; the
optimization only removes the expensive global path index.

The typed affix projection was also reduced from 50,944 numeric fields to 1,796
effective nonzero modifiers. The 49,148 explicit zeroes remain available in
`content_asset_property` and the compressed source XML, so the typed table is
now simpler without sacrificing source fidelity.

## Asset catalog coverage and extraction gaps

`bin/server/assets_catalog.xml` is now content-complete and safe to remove from
an extraction/staging source. The current `LuaRuntime` still reads it directly,
so deleting it from today's runtime requires the DB-backed catalog loader first.

The catalog maps 13,697 symbolic assets and their package hashes. Generated loot
names are linked through normalized rigblock/affix definition IDs rather than
incorrectly treating their symbolic names as extracted filenames. Metadata
coverage is 13,411 resolved entries and 286 classified gaps. The runtime
projection deliberately retains payload links for only 9,368 entries; catalog
identities remain complete, while client-only/research payload links are
cleared rather than pulling those assets back into `content.db`:

| Missing catalog type | Count |
| --- | ---: |
| Noun | 111 |
| Markerset | 84 |
| LootSuffix | 30 |
| NonPlayerClass | 16 |
| ClassAttributes | 10 |
| AIDefinition | 9 |
| ReservedItem | 9 |
| Phase | 8 |
| ServerEventDef | 4 |
| Level | 2 |
| lowercase `noun` variant | 2 |
| `lootpreferences` aggregate | 1 |

All 286 entries now have an explicit `asset_catalog_gap` row:

| Gap classification | Count | Evidence |
| --- | ---: | --- |
| `build_resource_absent` | 145 | No matching member exists in the build-103 `AssetData_Binary.package` extraction, whose per-type counts exactly match `bin/server/data`. |
| `level_logical_identity` | 60 | Marker-set identity belongs to the tutorial-v2, Scaldron arena, or TNX level family but is not a standalone extracted asset member. |
| `level_embedded_identity` | 37 | `!path` noun identity is authored through a level marker set rather than as a standalone noun member. |
| `generated_alias` | 40 | Thirty generated suffixes, nine reserved items, and the loot-preferences aggregate map to shared generated seeds rather than distinct payloads. |
| `case_variant_alias` | 2 | Lowercase noun catalog identities differ from their extracted counterparts only by case. |
| `package_absent` | 2 | Neither missing level group hash occurs among the 61 groups in build-103 `Levels.package`. |

The package-absent levels are `Game_Tutorial_cryos_1_v2.Level`, the
executable-facing tutorial name, and `scaldron_1_BG_arena.Level`. The package
contains exactly the same 61 level groups represented under `bin/server/data`;
the extracted tutorial XML remains the unsuffixed
`Game_Tutorial_cryos_1.Level`. This confirms a real build/catalog identity
gap rather than an extraction omission.

## Relationship graph

`content_asset_reference` turns the property index into a navigable graph. Each
edge retains the source asset/property and relationship name, plus separate
targets for catalog identity, stored payload, and registered script definition.
This distinction prevents a catalog-known resource with a missing payload from
being treated as an unknown reference.

The first graph pass contains 7,886 catalog-resolved edges, of which 7,879 also
have payloads. It also retains 92 typed XML references absent from the catalog.
After strengthening the ServerData registration parser and refreshing NPC-affix
links, 1,224 XML properties
are linked directly to registered abilities/modifiers by exact name. The second
pass recovered 63 definitions hidden behind `abilityName` variables or
unluac-expanded registration calls and added 135 exact XML-to-script edges.
High-volume relationships
include level-to-marker-set, marker-to-noun, noun-to-class-attributes,
noun-to-animation, noun-to-AI-definition, AI-to-phase, and AI/player-class to
ability/modifier.

The compiled-animation pass adds 1,428 exact character-animation property
edges across 111 distinct compiled resource names. Script references add 248
resolved rows across 232 distinct animation names. Matching is case-insensitive
but otherwise exact; no fuzzy animation identities were introduced.

All meaningful named production player/AI ability fields resolve to exactly one
script definition. The former `FireCheese` exception is an excluded developer
fixture, not an unresolved production ability. Its only package occurrences are
the type-`0xE51118C3` legacy player-class resources for
`TestCharacter_Bernd`, `TestCharacter_Holly`, and `TestCharacter_Cimino`
(ordinals `10946`, `11288`, and `11867`; instances `0x3754E922`,
`0x03B091F7`, and `0x0278F3BE`; decoded sizes `371`, `371`, and `384`;
SHA-256 `f9c829eaa5043b6c1bd2dc78c1a04ec355aac0c7791ed672e67cb0de6a719d1b`,
`a7fe6c82a6f82cc704491d9ed83d42d9f4830a6dbc5aa31351abf61363990655`,
and `10dfa4971d912988bc3fc88e07501fc821f7de7bf6b9969c306fd52219e17022`).
Each is paired by instance with a `TestCharacter_*` noun of type
`0x76A8F7D8` that points to the corresponding `.PlayerClass`; none is a
campaign noun. The three class payloads place `FireCheese` beside the test
actions `Fireball` or `PlasmaSentinelBasic`, `FireBreath`, and
`SuperFireball`, plus `RegenModifier`. Exact-byte scans find no `FireCheese`
member in `ServerData.package`, no constant in any of the 1,029 indexed Lua
chunks, and no string in the canonical executable decompilation. Content
validation should therefore exclude these three test-only references rather
than demand a fabricated registration. The remaining apparent misses are
`0x00000000` null sentinels and one malformed replacement character. The 115
numeric player-class `basicAbility` values are stable numeric identifiers, not
script registration names; `randomAbility*AnimState`, `sharedAbilityOffset`
vector children, and `probability` are likewise not missing definitions.

Script localization is normalized without duplicating locale text. Of 614
literal localization GUID properties, 600 identify a unique localization table
and eight otherwise-colliding keys are disambiguated by `localizedGroup`. Those
608 references join to 3,040 rows across the five installed locales. Two
references remain table-ambiguous, and four properties reference two keys absent
from every installed locale. Expressions inherited from another script object
remain in `script_definition_property` until expression-level alias resolution
is implemented.

## Campaign and difficulty content

The runtime campaign projection retains all 72 authored `ChainLevels` entries
in order. The recipe locates the authoritative resource by type/group/instance,
decodes it through `dbpf.Reader.Open`, validates the 12-byte header and 72
fixed-size records, and parses each record's nonzero reference handle plus its
required type/presence tag `1` before reading the trailing string pool's
`.Level` references. The build-103 asset contains 24 distinct handles; each
handle is reused by exactly the three records that reference the same level,
and no level is associated with two handles. These are retained process-era
addresses rather than relocatable payload offsets, so they cannot honestly be
used as string-pool offsets. Because direct relocation of those handles is not
reconstructed, recipe 12 additionally compares all 72 references against the
complete independently recovered build-103 sequence; count/repetition checks
alone cannot bless a reordered string pool. Every current reference resolves through
`level_alias` to one of 24 stored level payloads, and every target occurs three
times. The source's additional binary fields and 18 interspersed cinematic or
voice strings are not assigned speculative meanings; the complete raw resource
remains in `content_source_resource` for future projection.

Difficulty tuning uses the compact scalar/series model already used for loot,
plus a range table. It contributes four scalar Star Mode parameters, 288
ordered health/damage/avatar/rating values, and 113 item-level or gear-score
ranges. The normalized level pools add 424 noun choices: 215 minions, 159
specials, 19 bosses, and 31 agents. Each keeps its difficulty interval and
`is_horde_legal` flag, and all 424 nouns resolve to stored content.

The three objective pools contain 10 objectives and nine level-affix choices.
The 171 phase definitions retain phase type and optional start/gambit state; 63
printable gambit names resolve to registered script definitions. Several source
phase files contain binary/non-XML gambit bytes, so normalization uses the
sanitized property index while the original bytes remain authoritative in
`content_asset`.

No fixed campaign reward, item grant, or unlock field appears in `ChainLevels`,
`DifficultyTuning`, `LevelObjectives`, `LevelConfig`, or `Phase`. The only loot
term is the named `LootCrystals` objective. Permanent campaign reward recovery
therefore remains separate from these now-complete projections.

The next gameplay-tuning pass adds 48 unlock rows, 192 weighted crystal noun
rows, one crystal-level probability, 73 per-level elite-NPC rows, 144 encounter-
director curve values, and 19 core combat constants. All 192 crystal nouns
resolve to catalog identities and stored payloads. Unlock fields are retained
with their authored names and numeric values; their hash-like numbers are not
reinterpreted until the client/runtime meaning is confirmed.

`SpaceshipTuning` was reviewed but intentionally remains only in the lossless
asset and property index. Its contents are predominantly front-end cameras,
placards, lighting, colors, and UI timing, so promoting them into authoritative
server gameplay tables would blur the content boundary without improving a
server lookup.

AI and NPC-affix gameplay structures are now typed. The 232 AI definitions
produce 183 graph nodes; 170 nodes resolve to stored phase payloads. Of 364
nonempty AI ability-role values, 363 resolve to registered scripts. The only
remaining value is a malformed byte sequence in `verdanthboss`; empty ability
placeholders are retained in the raw/property representations but excluded from
the typed relationship table.

All 34 NPC affixes resolve to modifier scripts, including three registrations
recovered after the first relationship pass. Ten parent and ten child affix
links resolve to stored NPC-affix payloads. The 31 affixes with descriptions
each resolve their `AssetStrings` key across all five locales, producing 155
localized rows on demand; the three creation/control affixes have no authored
description.

The 1,020 `ServerEventDef` files were also classified. Ninety-five are empty
aggregate markers; the remainder primarily describes client effects, model
hardpoints, screen shake, sound, and voice-over presentation. Server-relevant
event identity is already normalized by `asset_catalog_entry` and retained
losslessly by `content_asset`, so a redundant typed summary table would add no
lookup capability. Individual presentation fields remain searchable through
`content_asset_property`.

All 100 `creature_template` IDs resolve uniquely to noun catalog hashes and
stored noun payloads. For example, Blitz Alpha resolves to `PC_EL_Rogue.Noun`,
which links to `PC_EL_Rogue.PlayerClass` and
`RoguePL_v0.CharacterAnimation`. Complete hero ability chains now depend on the
static script definitions rather than the placeholder zero ability IDs in the
legacy creature JSON.

## ServerData package

The separate client-file assessment is correct. A fresh extraction of
`Data/ServerData.package` contains 1,101 resources. Eighty-seven `lua` member
names correspond to the readable scripts already under `bin/server/data/lua`;
the package members are original Lua 5.1 bytecode rather than byte duplicates.
The remaining 1,014 resources were genuinely absent from the previous content
inventory.

All 1,101 extracted members are stored losslessly in both projections; the
original package itself is metadata-only. The main
groups are 481 abilities, 397 modifiers, 88 shared Lua resources, 42 hints, 31
behaviors, 11 popup-tip scripts, 16 tutorial scripts split across server/client
resource groups, and small tuning/template/animation metadata groups.

All 1,029 Lua bytecode members remain in runtime `content.db` for fidelity, but
GopherLua 1.1.1 parses source and does not load native Lua 5.1 bytecode. Runtime
content therefore also retains all 1,029 mapped source representations: 942
generated decompilations and the 87 readable source files that correspond to
package members. `meta.db` retains the same mappings plus the analysis tables.
This
includes 481 abilities, 397 modifiers, 31 behaviors, 11 popup tips, 16 tutorial
server/client scripts, five test scripts, and one client-global script.
Static registration recovered 479 abilities and 501 modifiers with no duplicate
registration names. This includes registrations expressed through literal API
calls, `abilityName` variables, and unluac-expanded calls. The normalized
metadata projection contains 10,362 direct fields and 2,361 references.
Original bytecode is authoritative for execution; decompiled Lua is the
research representation for dynamic control flow that cannot safely be reduced
to scalar rows.

`ServerData.package` is therefore content-complete and safe to remove from a
future extraction/staging source when the trusted databases are distributed.
The installed client still requires its package copy. darkspin needs a DB-backed
Lua loader that selects the mapped source representation for GopherLua while
retaining the original bytecode identity before removing the readable Lua tree
it currently executes.

The large rendering/presentation package classification also makes sense:
`Arenas_RDX9`, arena/environment textures and models, `PreBaked`, UI/Flash/Web,
32-bit images, audio/movies, rendering configurations, and effects should not
be normalized into the authoritative server schema. They are candidates only
for an optional archival/asset-browser database. Small configuration packages
remain worthwhile targeted probes. Compiled animation/gait metadata and the
resource-type inventory of `Levels.package` are now complete.

`CompiledAnimData.package` contains 2,407 compressed members: 2,340 `ANIM`
resources, 65 `GAIT` resources, and one each of the auxiliary `tlsa` and `pctp`
formats. The original package and every decompressed member are stored
losslessly in `meta.db`. All 64,858,029 imported bytes were decompressed from the database
and revalidated by length and SHA-256.

`Levels.package` is intentionally represented by package/type metadata rather
than duplicating its 217,404,391-byte payload. Its 62,232 resources comprise
33,829 of type `0x2AE9952D`, 28,281 of type `0x76E1259D`, and 61 each of types
`0x2699C284` and `0x1999AE0B`; 31,319 members are compressed and 30,913 are
stored. Its group IDs map to the same 61 extracted level identities. The bulk
is compiled geometry/level state rather than an additional permanent server
definition set, so retaining its checksum and inventory avoids inflating the
runtime database while preserving provenance.

## Source-removal ledger

Every file currently under `bin/server/data` is byte-for-byte recoverable from
`meta.db.content_asset`. Therefore all paths in the table below are
**content-complete and safe to remove from an extraction/staging source** when
the trusted database build is the distribution artifact. Runtime-required
payloads remain in `content.db`; normalized definitions whose raw source is not
required at runtime remain recoverable from `meta.db` only.

This is not yet permission to delete every path from the current darkspin runtime.
The current Go code still opens several of them directly. Those rows are marked
`DB loader required`; removing them from `bin/server/data` today would break
startup or gameplay until the other agent connects the content store. Rows
marked `No direct Go reader found` have no direct filesystem consumer in the
current source inspection and are safe candidates for removal now, though this
pass deliberately did not delete any source file.

| Source path | Files | Database representation | Current removal status |
| --- | ---: | --- | --- |
| `data/lootrigblock/*.xml` | 2,408 | Lossless assets, property index, and normalized rigblock/eligibility tables | No direct Go reader found; safe removal candidate. |
| `data/lootprefix/*.xml` | 338 | Lossless assets, property index, normalized affix/requirement/modifier tables | No direct Go reader found; safe removal candidate. |
| `data/lootsuffix/*.xml` | 328 | Lossless assets, property index, normalized affix/requirement/modifier/effect tables | No direct Go reader found; safe removal candidate. |
| `data/weapontuning/*.xml` | 1 | Lossless asset, property index, and 381 normalized tuning rows | No direct Go reader found; safe removal candidate. |
| `data/creature/creature_templates.json` | 1 | Reconstructed as 100 normalized rows from vanilla `AssetData_Binary.package`; locale text comes from installed `Text.package` resources | Safe to remove: `server.go` now loads `creature_template` from `content.db`, and the builder does not use this JSON. |
| `data/creature/creature_parts_templates.json` | 1 | Lossless asset and 2,408 normalized part rows | Safe to remove from the server runtime: no Go reader remains. The normalized part table is retained for future item/equipment lookups. |
| `data/noun/*.xml` | 4,971 | Lossless assets and property index | DB loader required by `NounDatabase`. |
| `data/aidefinition/*.xml` | 232 | Lossless assets, property index, 232 typed definitions, 183 nodes, and 364 ability-role rows | DB loader required by `NounDatabase`. |
| `data/characteranimation/*.xml` | 183 | Lossless assets and property index | DB loader required by `NounDatabase`. |
| `data/classattributes/*.xml` | 624 | Lossless assets and property index | DB loader required by `NounDatabase`. |
| `data/nonplayerclass/*.xml` | 586 | Lossless assets and property index | DB loader required by `NounDatabase`. |
| `data/npcaffix/*.xml` | 34 | Lossless assets, property index, and 34 typed affix/modifier/relationship/localization rows | DB loader required by `NounDatabase`. |
| `data/phase/*.xml` | 171 | Lossless assets, property index, 171 typed phase rows, and 63 script links | DB loader required by `NounDatabase`. |
| `data/playerclass/*.xml` | 115 | Lossless assets and property index | DB loader required by `NounDatabase`. |
| `data/level/*.xml` | 61 | Lossless assets and property index | DB loader required by level loading. |
| `data/markerset/*.xml` | 2,385 | Lossless compressed assets; no property expansion | DB loader required by level loading. |
| `data/lua/*.lua` | 87 | Lossless compressed assets | DB loader required by `LuaRuntime`. |
| `data/Abilities/PLACEHOLDER` | 1 | Lossless empty asset | Directory/path compatibility currently expected by `LuaRuntime`. |
| `data/servereventdef/*.xml` | 1,020 | Lossless assets and property index; reviewed as client effect/audio/voice-over presentation plus empty aggregates | No direct Go reader found; safe removal candidate. |
| `data/affixtuning/*.xml` | 1 | Lossless asset, property index, and 30 typed ordered chance values | No direct Go reader found; safe removal candidate. |
| `data/chainlevels/*.xml` | 1 | Lossless asset, property index, and 72 ordered/resolved campaign level rows | No direct Go reader found; safe removal candidate. |
| `data/charactertype/*.xml` | 5 | Lossless assets and property index | No direct Go reader found; safe removal candidate. |
| `data/condition/*.xml` | 8 | Lossless assets and property index | No direct Go reader found; safe removal candidate. |
| `data/crystaltuning/*.xml` | 1 | Lossless asset, property index, 1 level probability, 1 bonus scalar, and 192 resolved weighted crystal rows | No direct Go reader found; safe removal candidate. |
| `data/difficultytuning/*.xml` | 1 | Lossless asset, property index, 4 typed scalars, 288 series values, and 113 ranges | No direct Go reader found; safe removal candidate. |
| `data/directortuning/*.xml` | 1 | Lossless asset, property index, and 144 typed encounter-director curve values | No direct Go reader found; safe removal candidate. |
| `data/elitenpcglobals/*.xml` | 1 | Lossless asset, property index, and 73 typed per-level elite-NPC rows | No direct Go reader found; safe removal candidate. |
| `data/levelconfig/*.xml` | 15 | Lossless assets, property index, and 424 difficulty-bounded/resolved noun-pool rows | No direct Go reader found; safe removal candidate. |
| `data/levelobjectives/*.xml` | 3 | Lossless assets, property index, and 19 objective/affix pool rows | No direct Go reader found; safe removal candidate. |
| `data/lootpreferences/*.xml` | 1 | Lossless asset, property index, 63 typed scalars, and 76 typed rarity-series values | No direct Go reader found; safe removal candidate. |
| `data/magicnumbers/*.xml` | 1 | Lossless asset, property index, and 19 typed core combat constants | No direct Go reader found; safe removal candidate. |
| `data/navpowertuning/*.xml` | 1 | Lossless asset and property index | No direct Go reader found; safe removal candidate. |
| `data/objectextents/*.xml` | 1 | Lossless asset and property index | No direct Go reader found; safe removal candidate. |
| `data/pvplevels/*.xml` | 1 | Lossless asset and property index | No direct Go reader found; safe removal candidate. |
| `data/sectionconfig/*.xml` | 1 | Lossless asset and property index | No direct Go reader found; safe removal candidate. |
| `data/spaceshiptuning/*.xml` | 1 | Lossless asset and property index; reviewed as predominantly client presentation/UI tuning | No direct Go reader found; safe removal candidate. |
| `data/unlockstuning/*.xml` | 1 | Lossless asset, property index, and 48 typed unlock rows | No direct Go reader found; safe removal candidate. |
| `data/testasset/*.xml` | 1 | Lossless asset and property index | Test-only; safe removal candidate. |
| `data/version_bin.txt` | 1 | Lossless root metadata asset | Not gameplay content; retain only if installer/version compatibility requires it. |
| `../assets_catalog.xml` | 1 | Lossless asset, 13,697 normalized entries, tags, and content links | DB loader required by `LuaRuntime`. |
| installed `Data/Locale/*/Text.package` | 5 | Lossless packages and 50,661 normalized localized strings | Safe as content sources; client installation still owns its package copies. |
| installed `Data/ServerData.package` | 1 package / 1,101 members | Lossless package/resources, 942 decompilations covering all 1,029 Lua bytecode members, 980 definitions, properties, and references | Safe as a content source; installed client still owns its package copy. |
| installed `Data/CompiledAnimData.package` | 1 package / 2,407 members | Lossless package/resources, normalized resource inventory, and exact script/character-animation relationships | Safe as a content source; installed client still owns its package copy. |
| installed `Data/Levels.package` | 1 package / 62,232 members | Checksummed package/type summary only; bulk payload intentionally not duplicated | Not content-complete in the DB and not a removal candidate; installed client/runtime level data remains authoritative. |

## Tutorial item correlations

Names are not unique item identities. Science type, minimum/item level, slot,
class restrictions, and eventually the rolled affixes are required to select a
definition. The current correlations are:

| Visible name | Rigblock ID | Asset key | Permanent definition | Confidence |
| --- | ---: | --- | --- | --- |
| Electro Claws | `268` | `0x646569E2` | Plasma, Ravager-only, Weapon, levels 5-130, `ce_weapon_lightningRavager_02-symmetric.prop` | Asset-confirmed name and definition; video-observed tutorial grant. |
| Onyx Barrier | `887` | `0x62565C1D` | Plasma, all three classes, Defense, levels 5-1000, `ac_plateForehead_blk_01-symmetric.prop` | Asset-confirmed match to the video-observed level-5 Plasma Defense tooltip. |
| Alpha Headgear | `1435` | `0x4354A961` | Plasma, all three classes, Defense, levels 5-1000, `SA_hats_01.prop` | Asset-confirmed level-5 Plasma variant; its tutorial grant/timing is not established by the current tutorial note. |

`Electro Claws` also has unique/developer rigblock `10146` at levels 999-1000,
so display name alone must not select it. `Onyx Barrier` has Bio, Chrono, Cyber,
Necro, Plasma, and unique variants. `Alpha Headgear` has the same five science
variants plus two special variants. The level-5 Plasma definitions are the
only ordinary candidates matching the observed tutorial-era requirements.

The tutorial level assets do not contain concrete reward IDs. `Electro Claws`
and `Onyx Barrier` remain video-observed grants whose server-side roll/grant
callback is unknown. Complete decompilation of the ServerData tutorial scripts
finds calls for `UnlockNextAbility`, `UnlockSecondCreature`, `UnlockOverdrive`,
`UnlockCrystals`, and `DropCrystals`, but no item, inventory, loot, or reward
grant call. The item source is therefore more likely native level/loot runtime
behavior than a hidden tutorial-script reward table. Do not encode either as a
fixed tutorial reward until a
runtime inventory trace, callback recovery, or equivalent authoritative source
confirms whether the retail tutorial selected a fixed definition or constrained
the ordinary loot generator.

## Current schema boundary

`loot_rigblock` represents the permanent visual/eligibility base of an item,
including the raw content byte whose `0x70` mask controls native flair
conversion admission. It does not represent a complete equippable item. A player inventory row will
eventually need to reference permanent content and store only roll-specific or
ownership-specific state, including a stable inventory ID, owner, item level,
rarity, random seed, cost/status/usage, selected prefix/suffix IDs, secondary
prefix ID if applicable, generated modifiers/stats, weapon damage modifier,
equipped creature/slot, and acquisition state.

The extracted `creature_parts_templates.json` mixes those instance fields with
content references. Its rows are now preserved in `creature_part_template`, but
most roll fields are empty/default placeholders. It should not become the final
inventory model. Its misleading `rigblock_asset_id` field is a rigblock
definition ID and is normalized as `creature_part_template.id`, referencing
`loot_rigblock.id`.

## Pending Go/schema work

Another agent owns the Go implementation. No Go files were changed in this
slice. The implementation still needs to:

1. Adopt/version the `build-103-content-v1` runtime schema and manifest without
   making the darkspin repository the long-term data-authoring source.
2. Add a read port for `content_asset` that transparently decompresses and
   verifies payloads, then migrate the current creature, noun, level, marker,
   and Lua filesystem loaders. Lua loading must use the mapped source payload
   because GopherLua does not accept native Lua 5.1 bytecode. Original bytecode
   remains in `content.db` for identity/fidelity; static-analysis tables remain
   optional `meta.db` inputs.
3. Move the one-time population logic into the future dedicated content
   repository and publish reproducible, signed/checksummed `content.db` and
   optional `meta.db` projections.
4. Normalize additional high-value property-indexed tuning when typed queries
   become necessary. Affix chances, loot preferences, and rarity/drop formulas
   are already represented by the generic scalar/ordered-series tables.
5. Define the user-database inventory foreign-key/reference strategy. SQLite
   cannot enforce a normal foreign key across separate database files, so the
   feature operation must validate content references transactionally before
   writing ownership state.
6. Replace JSON-backed creature and creature-part lookups with feature-owned
   content ports only after parity tests prove the database contains every field
   consumed by the server.

## Implementation handoff: first-run content builder

The intended distribution is `darkspin.exe` plus the minimum immutable
`content.db`; `user.db` remains writable player state and `meta.db` remains an
optional research/site/tooling artifact. To eliminate a mandatory content
database download, darkspin may generate `content.db` from an installed compatible
Game client. Implement this as a reproducible content compiler, not as
runtime fallbacks scattered through gameplay loaders.

### Implemented bootstrap status

The first command/startup slice is now present in source:

- `darkspin content build` initializes a temporary SQLite database from a
  build-103 installation and atomically installs it;
- `darkspin content verify` validates the stable manifest plus SQLite integrity
  and foreign keys;
- `darkspin server` verifies an existing `darkspin/cache/content.db` and invokes the
  builder when the file is absent;
- the temporary development default for `--game-path`/
  `--content-game-path` is
  `C:\src\recap\darkspin\bin\game\GameBin`;
- a Go-native DBPF header reader validates version-3 package headers and
  concurrently records SHA-256, size, resource count, index version, index
  size, and index offset for the two runtime packages and five locale packages;
- bootstrap databases record `database_manifest` and
  `content_source_package`, use a temporary sibling file, verify before rename,
  and refuse to overwrite an existing database.

This is intentionally build-state scaffolding, not yet the full 35 MB gameplay
projection. It records `build-103-bootstrap-v1`; package member decompression,
the recipe allowlist, typed importers, Lua source production, and filesystem
loader replacement remain required below. Until those stages land, the current
server still consumes `bin/server/data` even though command startup now requires
a valid runtime-content manifest.

### Required behavior

1. Add an explicit command such as
   `darkspin content build --game-path <installation>` and a read-only
   `darkspin content verify` command.
2. At normal startup, open `content.db` and validate `database_manifest`. If it
   is missing or incompatible, locate or request the game installation and run
   the same builder. Never silently use a partially-built database.
3. Build `content.db.tmp` beside the final database, verify it completely, close
   it, and atomically rename it to `content.db`. A crash or validation failure
   must leave any existing valid database untouched.
4. Identify the input as Game `5.3.0.103` using package/index fingerprints
   before applying build-103 interpretations. Reject an unknown build rather
   than importing it with incorrect type, hash, or field assumptions.
5. Write `database_manifest` with role `runtime-content`, source build `103`, a
   versioned content release, recipe version, and input fingerprints. `user.db`
   compatibility must be validated by content release because SQLite cannot
   enforce foreign keys across database files.
6. Do not generate `meta.db` during ordinary startup. Full package archival,
   generic property indexing, decompilation analysis, relationship discovery,
   catalog-gap evidence, and compiled-animation inventory belong to an explicit
   offline metadata build.

### Package implementation

Replace the external inflate/unpack executable with a Go-native DBPF reader and
decompressor. It must parse package indexes, retain type/group/instance identity,
support stored and compressed resources, validate decoded sizes, and expose a
streaming API. Import directly from package members into SQLite; do not create a
temporary `bin/server/data` XML tree.

Preserve raw package identity beside normalized interpretations. Resource
selection must use type/group/instance IDs and verified names/hashes, not file
offsets. Offsets may be cached only after verifying the package-index
fingerprint, because repacking can move unchanged resources.

The Go implementation now completes the package-identity foundation directly
against the vanilla-style `bin/darkspinner` install. It supports the `10 FB`
and `50 FB` RefPack headers present in build 103, validates every decoded size,
and stores every member of the seven required packages in
`content_source_resource`. Each row retains package/ordinal and
type/group/instance identity, compression and entry flags, the exact stored
payload, and SHA-256 for both stored and decoded bytes. A clean validation build
produced an 18,157,568-byte `build-103-package-v2` database and passed SQLite,
foreign-key, package-count, and resource-count verification.

The current reproducible projection is `build-103-serverdata-v4`. A clean build
from `bin/darkspinner/GameBin` produced a 26,697,728-byte database with
seven tables. It adds a typed `server_data` lookup containing all 1,101
decoded `ServerData.package` members, compressed independently with zlib for
fast SQLite lookup. Exactly 1,029 rows are marked as compiled Lua, and all
1,029 decoded payloads begin with the standard Lua 5.1 `\x1bLuaQ` header. The
builder verifier now enforces the package resource count and the 1,029 Lua
count. The prior DarkSpinner v3 database is retained locally as
`darkspin/cache/content-v3.db` for rollback; the active
`darkspin/cache/content.db` is the verified v4 rebuild.

This does not yet reproduce the complete 35,074,048-byte reference projection.
That database contains 33 runtime tables, including 11,518 `content_asset` rows
and the item, tuning, AI, phase, level, and unlock projections itemized above.
The v4 builder currently produces package/resource identity, localization,
creature templates, creature ability links, and ServerData payloads. Its
resource names use stable group/type/instance identities where the vanilla
packages do not expose authored filenames. Reproducing the human-readable
1,101-name index requires a versioned build-103 name-hint map or an equivalent
catalog decoder.

The reference database also maps every compiled Lua row to an executable source
representation: 942 unluac decompilations plus 87 authored readable scripts.
The vanilla-only v4 builder preserves every authoritative bytecode chunk but
does not yet generate those 942 decompilations, so the current GopherLua runtime
cannot switch to the database solely from v4. Add a deterministic decompiler
stage (or embed a trusted versioned source recipe), validate every generated
source with GopherLua, and compare all 980 ability/modifier registrations before
removing `data/lua`. `Levels.package` is also not part of the seven-package v4
recipe yet; classify and import its level/marker payloads separately. It is not
the source of the 1,029 ServerData Lua chunks.

### Reproducible metadata build

`darkspin build meta` now provides the explicit offline metadata build requested
by the runtime/metadata split. With no flags, a distributed executable reads
`GameBin` and `Data` beside itself and writes `darkspin/cache/meta.db`.
Development builds can select both paths with `--game-path` and `--output`.
The command never runs during server startup and refuses to overwrite an
existing database.

The initial `build-103-meta-v1` recipe is a self-contained superset of the v4
runtime projection. A clean vanilla build produced a 43,896,832-byte database
with 11 tables, 15,246 package resources, 1,101 ServerData rows, 1,029 compiled
Lua chunks, 100 creature templates, and 50,661 localization rows. It also adds:

- exact archived bytes for `CompiledAnimData.package`;
- all 2,407 decoded animation/gait/auxiliary resources, individually compressed
  and checksummed;
- package/type inventory for `CompiledAnimData.package` and `Levels.package`;
- the complete 62,232-resource Levels type/compression inventory without
  duplicating the 217,404,391-byte package payload.

Verification checks SQLite integrity and foreign keys, package and resource
counts, the archived-package SHA-256, and every decoded animation payload's
size and SHA-256. The generated database passed all checks against
`bin/darkspinner/GameBin`.

This v1 metadata recipe does not yet reproduce the older 107,520,000-byte
research database. The remaining work is the generic XML/property graph,
authored ServerData filename hints, 942 deterministic Lua decompilations,
script-definition/reference analysis, catalog-gap evidence, and the remaining
typed item/tuning/AI/level projections. Those importers should extend
`BuildMeta`; they must not be copied into ordinary content startup merely to
match the historical database size.

This version is an intentionally complete package-backed staging projection,
not the final pruned runtime recipe. Once typed importers and loader-parity tests
cover a resource family, the recipe should omit its raw member when normalized
rows are sufficient and retain raw bytes only for assets still consumed in
their source representation.

Do not use `GameBin/Server/data/serverdata` as a build input or semantic
reference. That tree is not present in a vanilla client and is an externally
converted XML projection whose presence in a development install can conceal
missing package decoders. The authoritative validation corpus is
`bin/darkspinner/GameBin` with its sibling `Data` package directory.

`AssetData_Binary.package` members decode to Game's type-specific binary
asset structures, not XML. The Go recipe now decodes the playable noun and
player-class structures directly, joins all 100 authored pairs, and writes
`creature_template` plus symbolic `creature_template_ability` relationships.
The gameplay noun ID is the case-insensitive FNV-1 hash of the noun asset key;
it is not the DBPF member instance ID. The zero-based binary enums map Cyber,
Chrono, Bio, Plasma, and Necro to 0 through 4, and Sentinel, Ravager, and
Tempest to 0 through 2. Three noun families carry an extra locomotion string,
so the player-class link is located by its `.PlayerClass` suffix instead of a
fixed dynamic-string ordinal.

Recipe 10 corrects the player-class ability-slot projection. The dynamic fields
are `basic`, `support`, `random`, `active`, `passive`, then the class-attributes
reference; the earlier projection shifted the public slots and discarded the
real passive. `creature_template_ability` now retains all five authored names
with `special_1=active` and `special_2=support`. The legacy integer columns are
populated from the authored basic ID and case-insensitive asset hashes instead
of literal zeroes. Sage Alpha therefore resolves `SupportHealerBasic`, active
`SupportHealerSupport`, support `TreeOfLife`, random `CastEnrage`, and passive
`SupportHealerPassiveModifier`.

The Lua index now includes the instruction-checked targeted-AOE, DrainSnare,
and speed-modifier aliases needed by `SupportHealerSupport`. `LuaModules`
returns the complete cycle-fenced require closure rather than only direct
imports, allowing the constrained VM to compile Sphere of Transfusion without
an external Lua tree.

Sage's packaged passive companion is also recoverable under shared instance
`0x75bbfd0f`: the package contains `HelperMelee.Noun`,
`HelperMelee.NonPlayerClass`, `HelperMelee.ClassAttributes`,
`HelperMelee.AIDefinition`, and its follow-owner behavior. The AI definition
names `PetTargeting` and `SupportHealerPetBasic`. The latter resolves to Lua
chunk `889`, source `Abilities/0xD16D3E2A.lua`, SHA-256
`e43e778d80c5691668e9c2de7d0fca7e598461f9b7904b54386cc4c5fbe440b2`.
Decoded focused evidence is retained under `bin/game/logs/helper-melee`.
Recipe 12 projects the pinned `HelperMelee.Noun` payload into `noun_physics`.
Its authored bounds are `(-0.5,-0.5,0)` through `(0.5,0.5,1)` and its runtime
companion melee envelope therefore consumes the content-owned `0.5` horizontal
footprint rather than a transport constant.
The non-player-class projection exposes authored scalar `0.1`; it must not be
treated as proven final runtime HP until the mirrored-owner attribute formula
called by chunk 270 is reconstructed.

The five locale `Text.package` files decode to BOM-prefixed UTF-8 authored text
tables. The recipe preserves their resource instance as `table_id`, accepts
authored multiline continuations and known non-hex key typos, and produces the
expected 50,661 `localization_text` rows. A vanilla validation build produced
exactly 100 creature rows; Blitz Alpha resolves to noun ID `1667741389`, locale
key `0x0ababafd`, Plasma/Ravager, and authored 4-12 weapon damage. Remaining
legacy non-ability combat fields keep their existing runtime defaults while the
symbolic ability rows and integer IDs preserve the authored loadout.

### Build-103 recipe

Embed or ship a small versioned, compressed build recipe. It should be generated
from the comprehensive `meta.db` inventory and the retained runtime projection,
then reviewed as source data. The recipe should contain:

- required package names and package/index fingerprints;
- the allowlist of required resource type/group/instance identities;
- resource-type-to-importer routing;
- stable asset hashes, generated aliases, and known build-103 corrections;
- typed XML field extraction plans and repeated-field ordering rules;
- localization resource/table mappings;
- Lua package-resource to executable-source mappings;
- import-stage dependencies;
- expected row counts, uniqueness constraints, and validation invariants.

For build 103, `AssetData_Binary.package`, `ServerData.package`, the required
locale text packages, and catalog identity inputs are relevant. The ordinary
runtime build must skip `Levels.package`, `CompiledAnimData.package`, rendering,
UI, audio, movie, effect, and environment packages. Their useful inventories
are already metadata-only. Server-event presentation data, compiled animations,
original package blobs, and the generic property graph must not be pulled back
into runtime `content.db` merely because they exist in the client.

The recipe should distinguish resources whose raw payload is still required
from resources whose normalized rows are sufficient. Current runtime payload
requirements include nouns, marker sets, levels, AI/class/phase/property assets,
catalog input, and Lua source mappings. Loot definitions, item tuning, creature
templates, and other completely projected definitions should be read from typed
tables after loader-parity tests, without retaining their raw staging source in
the runtime database.

### Lua constraint

Do not implement the runtime loader as native-bytecode-only. The client package
contains standard Lua 5.1 bytecode, but the repository uses GopherLua 1.1.1,
whose `LState.Load` path parses source and does not load native Lua bytecode.
Runtime `content.db` therefore currently retains both:

- all 1,029 original compiled Lua members for exact identity/fidelity; and
- all 1,029 mapped source representations used for execution, consisting of
  942 generated decompilations and 87 corresponding readable source assets.

A DB-backed Lua loader must select the mapped source, load modules in a stable
dependency order, and preserve the original package resource identity for
diagnostics. Before removing `bin/server/data/lua`, compile/load every retained
source with GopherLua and run ability/modifier registration parity tests. A
future compatible bytecode VM or verified bytecode translator could remove the
runtime source requirement, but the present GopherLua runtime cannot.

### Concurrent build pipeline

Use concurrency for package reading, decompression, hashing, XML parsing,
payload compression, localization decoding, and typed projection. Do not use
multiple contending SQLite writers. The intended bounded pipeline is:

```text
sequential reader per package -> decode/parse worker pool -> single bulk writer
```

Use a bounded worker count such as `min(GOMAXPROCS, 8)` and bounded channels so
expanded marker XML cannot exhaust memory. Multiple package readers are useful
on SSDs; on hard drives, keep package reads sequential and parallelize CPU work
after each compressed member is read. Worker completion order must never assign
database identity or ordinal. Sort deterministic inputs and preserve authored
package/XML ordinals.

The single SQLite writer should use prepared statements, large stage-level
transactions, and deferred index creation. Because the target is a disposable
temporary database, its build connection may use `journal_mode=OFF` and
`synchronous=OFF`; durability comes from complete verification followed by the
atomic rename. Create the database fresh so a final `VACUUM` is unnecessary.

Where practical, allow the content store to retain an original compressed
member with an explicit compression codec instead of always decompressing and
recompressing it. This is most valuable for large marker sets. Any such codec
must be supported by the content read port, must retain decoded-size and SHA-256
verification, and must not expose DBPF details to feature code.

### First-run build performance

The vanilla v4 content build was profiled on an i7-8700K with a warm filesystem
cache using a separately compiled Darkrun executable. Three clean baseline
builds averaged 4,020.76 ms (3,979.27-4,052.67 ms). The optimized build averaged
2,244.05 ms (2,217.80-2,276.12 ms), a 44.2% reduction in first-run generation
time.

The profile showed that adding decode workers was not the useful first move.
The original DBPF index parser issued a Windows `ReadAt` syscall for nearly
every integer field. It now reads each bounded index once and parses it from
memory. Content import also decodes RefPack from the already-retained stored
payload instead of reading compressed bytes twice, batches localization and
package-resource inserts, and builds the large uniqueness/lookup indexes after
bulk insertion. SQLite remains a single deterministic writer.

Two independent optimized builds were compared across every column of all
seven tables and had zero differences. The optimized output also had zero
semantic differences from the active v4 database. A preexisting nondeterminism
in `creature_template_ability.id` was removed by inserting ability slots in a
fixed order. CPU profiling after these changes showed SQLite binding, stepping,
and index creation as the dominant remaining cost; DBPF decompression was no
longer large enough for bounded worker concurrency to justify its ordering and
memory complexity.

### Determinism and validation

The builder must produce logically identical rows regardless of worker count.
Validate at least:

- `PRAGMA integrity_check = ok` and an empty `PRAGMA foreign_key_check`;
- the expected manifest, source build, and recipe version;
- required table and row counts;
- uniqueness of stable content IDs and catalog hashes;
- decompressed length and SHA-256 for every retained payload;
- all 1,029 compiled Lua members and all 1,029 source mappings;
- successful GopherLua parsing/loading of every runtime Lua source;
- item, affix, creature, noun, level, marker, AI, phase, tuning, localization,
  and catalog lookup parity against the current filesystem loaders;
- rejected/missing inputs leave the previous valid `content.db` unchanged.

The current build-103 reference result is a 35,074,048-byte `content.db` with
11,518 retained payloads and 15,137,282 compressed payload bytes. File-byte
identity is not required if SQLite layout changes, but logical rows, stable IDs,
payload hashes, and lookup behavior must match. A concurrent recipe-driven
builder on a modern SSD should target roughly 3-10 seconds; correctness and
atomic recovery take priority over that target.

## Next content slices

Priority order:

1. Recovery of permanent campaign rewards and the tutorial's actual item-roll/
   grant callback and inventory payload, joining the normalized rigblock,
   affix, and tuning records.
2. Resolve the six literal script-localization exceptions and expression-level
   localization aliases. The former `FireCheese` ability exception is closed as
   a three-resource developer-test fixture and is not part of production
   definition coverage.
3. Probe small configuration packages for permanent server-relevant constants,
   keeping rendering, UI, audio, movie, and environment payloads metadata-only.
4. Reduce the runtime projection further only after loader parity tests. The
   largest remaining payloads are marker sets (9.0 MB compressed) and nouns
   (3.2 MB compressed); both are currently required because their complete
   runtime structures have not yet been normalized into typed tables.

Raw package asset IDs/keys and raw property names should remain beside every
normalized interpretation. This permits later corrections without losing the
original extracted identity.

## Generic DBPF archive tooling

The package reader now belongs in `content/dbpf`. DBPF navigation, lossless
archive extraction, and package writing are reusable content capabilities. The
SQLite content compiler consumes that sibling package but does not own it.

`dbpf.NewReader` accepts an `io.ReaderAt` and size, following the same basic
shape as Go's ZIP reader. It reads only the 96-byte header and compact resource
index. `OpenRaw` returns a bounded section reader for one stored member, and
fingerprinting and extraction use bounded streaming buffers. A caller can
therefore navigate large packages without copying the whole package into
memory. The build-103 `AssetData_Binary.package` and `ServerData.package`
indexes were checked directly: both use index flag `0x4` and 28-byte entries,
matching the shared high-instance parser.

The root CLI has two lossless archive commands:

```text
darkspin unzip foo.package                 -> _foo.package/
darkspin unzip foo.package destination/    -> destination/
darkspin zip _foo.package/                 -> foo.package
darkspin zip source/ destination.package   -> destination.package
```

Extraction writes `dbpf.json` plus a `resource/` payload directory. Payloads
remain exactly as stored, including RefPack-compressed members and unknown
resource types. The manifest preserves TGI identity, decoded and stored sizes,
compression/entry flags, the original header, and the shared-index layout.
`zip` streams those members back into a package and rebuilds offsets and the
index without recompression. It refuses to overwrite an existing destination.
This raw representation is the safe universal interchange layer; future
decoded XML/Lua editing should be an additional codec layer, not a replacement
for the lossless payloads.

The content bootstrap now opens each source package once, passes its filesystem
reader directly to `dbpf.NewReader`, and fingerprints through the same handle.
The full content recipe should likewise consume indexed members directly into
its bounded decode/parse worker pipeline. It should not invoke `darkspin unzip`
or create an intermediate extraction tree during first-run `content build`.
