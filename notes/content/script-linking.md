# Cryos component-link decoder gap

## Result

`Game_Tutorial_cryos_1` is `level.id=9`. Its five tutorial trigger jobs
are authored in `Game_Tutorial_cryos_1_Audio.Markerset`
(`level_marker_set.id=293`, content resource `7744`), not in a separate Cryos
script list. The marker component stores a dotted Lua entry point such as
`nTutorial_IntroSecondCreatureUnlock.main`. The corresponding Lua 5.1 chunk
does **not** contain that dotted string. It contains
`nTutorial_IntroSecondCreatureUnlock` and `main` as separate string constants.

That representation is the `level_script` gap. The builder currently indexes
each Lua constant independently and joins a marker callback only by exact
string equality. Consequently the dotted component value matches no chunk.
Splitting the value and looking up only `main` would be worse: `main` is shared
by many unrelated job chunks and would produce false links.

There is a second prerequisite gap in the same decoder. `markerStringPairs`
accepts a marker only when its authored marker-name stem occurs in its noun
name. The Audio layer contains descriptive names backed by generic nouns (for
example `SecondCreature_trigger.Noun-1` backed by `TriggerZone.Noun`). Those
records are skipped, so their component strings can bleed into a later marker.
The current `content.db` demonstrates this: marker row `14006` is assigned an
event pairing `nTutorial_IntroAbilities.main` with
`SecondCreature_trigger.Noun-1`. That normalized row is a decoder artifact,
not authored linkage. Marker boundaries must be made reliable before adding
the script join.

## Authoritative payload and encoding

The focused payload was extracted from the authoritative runtime database:

```text
bin/server/darkrun.exe db level_marker_set bget source_payload where id=293 \
  --decode zlib \
  --output bin/game/logs/script-linking-cryos-audio.markerset \
  --config bin/darkspinner/darkspin.toml
```

The decoded payload is 7,377 bytes with SHA-256
`e7ce460e02f7c76459563ac13f9a24e525bb296c7f9fdef65fe8315db10a0bc4`.
It is the compiled `0xA11D3144` marker-set resource whose asset name is
`Game_Tutorial_cryos_1_Audio.Markerset`.

Direct boundary checks place consecutive fixed records at `0x20`, `0xe8`,
`0x1b0`, and `0x278`: the record stride is `0xc8`. The current normalized
decoder advances `0xd0`, which explains the eight-byte-per-record drift and
the later cross-marker string ownership. This is payload evidence for the
boundary prerequisite; it does not by itself specify a safe general decoder
for every marker component variant.

The relevant authored component path is the trigger volume's
`luaCallbackOnEnter` field. The adjacent trigger data supplies the sphere
radius and `triggerOnceOnly`; the marker/shared data supplies `markerId`,
world-space `pos`, and `serverOnly`. Numeric scalars are little-endian, floats
are IEEE-754 binary32, booleans are zero/nonzero authored scalar fields, and
text is printable ASCII terminated by `00`.

Each Lua locator is physically one C string. A literal byte `2E` (`.`)
separates its two logical Lua identifiers; there is no NUL between them. The
five strings start at these decoded-payload offsets:

| Offset | Exact `luaCallbackOnEnter` bytes before NUL |
| ---: | --- |
| `0x11A5` | `nTutorial_IntroOverdriveActivate.main` |
| `0x127F` | `nTutorial_IntroAbilitySecond.main` |
| `0x1355` | `nTutorial_IntroHealthAndPower.main` |
| `0x1998` | `nTutorial_IntroAbilities.main` |
| `0x1A75` | `nTutorial_IntroSecondCreatureUnlock.main` |

The complete component fields for the five records are:

| Marker name / noun | `markerId` | Position | Sphere radius | `triggerOnceOnly` | `serverOnly` | `luaCallbackOnEnter` |
| --- | ---: | --- | ---: | --- | --- | --- |
| `TriggerZone.Noun-2` / `TriggerZone.Noun` | `1359119906` | `(-349.459, -223.142, 10.088)` | `30` | false | true | `nTutorial_IntroOverdriveActivate.main` |
| `TriggerZone.Noun-3` / `TriggerZone.Noun` | `1410140146` | `(232.287, -93.752, 10.526)` | `5` | true | true | `nTutorial_IntroAbilitySecond.main` |
| `TriggerZone.Noun-1` / `TriggerZone.Noun` | `3802061458` | `(68.630, -38.059, 19.974)` | `15` | false | false | `nTutorial_IntroHealthAndPower.main` |
| `TriggerZone.Noun` / `TriggerZone.Noun` | `32670756` | `(233.674, -96.281, 10.150)` | `5` | true | false | `nTutorial_IntroAbilities.main` |
| `SecondCreature_trigger.Noun-1` / `TriggerZone.Noun` | `3912233898` | `(259.346, 81.391, 25.088)` | `25` | false | true | `nTutorial_IntroSecondCreatureUnlock.main` |

These are callback-only trigger entries: the authored slot is
`luaCallbackOnEnter`; there is no event-name string to pair with it. They must
not be interpreted as adjacent event/callback pairs by the generic string
heuristic.

## Reconstruction rule

For a marker callback of the exact form `<module>.<callback>`:

1. Split at the single literal dot. Require two nonempty Lua identifiers that
   each satisfy the existing identifier grammar used by
   `isLuaCallbackName`. Do not treat `.Noun`, asset paths, multiple-dot names,
   or arbitrary authored text as script locators.
2. Preserve the complete authored dotted locator on the level event. It is the
   collision-free identity of the component callback. Also retain the exact
   component and slot (`triggerVolume`, `luaCallbackOnEnter`) rather than
   manufacturing an event name.
3. Find Lua chunks that contain **both** the exact module string and the exact
   callback string in the same chunk's inspected Lua constants. Comparison is
   case-sensitive because Lua globals and table keys are case-sensitive.
4. Link only if that same-chunk intersection has exactly one member. The two
   halves may not be satisfied by different chunks.
5. If an authoritative `lua_module_alias` row exists, it may narrow the module
   side, but the target chunk must still contain the callback constant. The
   current alias table covers only an ability template and supplies no
   tutorial aliases, so it cannot presently resolve these five links.

This rule is deliberately separate from the existing standalone native/Lua
callback heuristic. Names such as `InteractWithObelisk`,
`DirectorTrigger_SpawnBoss`, and `HordeSpawner_Register` have no module prefix
and keep their existing behavior.

## Expected Cryos links

The Lua payloads were independently extracted with `darkrun db bget` from
`server_data.decoded_payload` using `--decode zlib`. Every listed chunk contains
its exact module string and the separate constant `main`; none contains the
dotted marker locator.

| Level/event locator | Lua chunk | Server-data resource | Source | SHA-256 |
| --- | ---: | ---: | --- | --- |
| `9 / nTutorial_IntroOverdriveActivate.main` | `141` | `13669` | `0x24F78AA1/0x724391F1.lua` | `909a2ff8a3494f6b9bcfd26461cbe72538d8161778771cc8bc5e11792fcf1ffa` |
| `9 / nTutorial_IntroAbilitySecond.main` | `930` | `14509` | `0x24F78AA1/0xBA0E5C06.lua` | `07c3a923a272fc2313dd00876044398597fd1107e833037cd0e2cf2b7e57ec64` |
| `9 / nTutorial_IntroHealthAndPower.main` | `458` | `14012` | `0xA35BED24/0xBCE26B84.lua` | `f267ff878ef2ef61f913f38fb3471bbc5b6fbc768607d3188bebeb2fd0dd0b2a` |
| `9 / nTutorial_IntroAbilities.main` | `619` | `14183` | `0xA35BED24/0x9E23DE1E.lua` | `8274e9d35a88c2697ca53c0f8583107cd58c7390f533fa4326302825023f7985` |
| `9 / nTutorial_IntroSecondCreatureUnlock.main` | `215` | `13753` | `0x24F78AA1/0xE6FA7F3F.lua` | `e98b1e855e64904689451bd79635c3487a7a20e29f1b943885c9648b61f8e30a` |

Each marker must own one callback event and that event must own exactly one of
the links above. The current Cryos level has ten `level_script` rows, all from
the obelisk callbacks. With only these five missing tutorial links added, the
expected total is fifteen. The two client jobs (`HealthAndPower` and
`IntroAbilities`, group `0xA35BED24`) remain valid content links even though
their execution authority is client-side; `level_script` records authorship,
not permission for the server to execute UI code.

## Ambiguity and rejection rules

- Zero same-chunk matches: retain the level event, create no `level_script`
  row, and expose the unresolved locator in build verification/diagnostics.
- More than one same-chunk match: create no links. Do not fan out merely
  because multiple chunks contain `main` or even the same module constant.
  Resolution requires stronger authored evidence such as an exact module
  alias or a unique package/resource binding.
- Module in one chunk and callback in another: no match.
- Empty half, multiple dots, invalid identifier characters, wrong case, or a
  noun/asset suffix: not a dotted Lua locator; leave it to the ordinary event
  decoder or reject it as unresolved component text.
- Repeated occurrences of a constant within one chunk do not create
  ambiguity; candidates are deduplicated by chunk ID.
- The same unique chunk referenced by two distinct marker events produces two
  event links, not one coalesced event.
- Native callbacks with no packaged Lua match remain level events without
  script links. Absence of a Lua link is not proof that an authored callback is
  unused.

## Tests required before changing the builder

1. **Full marker fixture.** Decode the retained 7,377-byte Audio marker-set
   payload by SHA-256 and assert all five marker names, noun names, IDs,
   positions, radii, once-only flags, server-only flags, and exact
   `luaCallbackOnEnter` strings. This must fail if component strings cross a
   marker boundary.
2. **Nonmatching marker names.** Add a focused boundary test for
   `SecondCreature_trigger.Noun-1` / `TriggerZone.Noun` and for an audio marker
   whose descriptive name is backed by `AudioTrigger_3D.Noun`. Marker identity
   cannot depend on the name stem occurring in the noun.
3. **Locator parsing.** Cover the five real locators plus empty halves,
   leading/trailing dots, multiple dots, `.Noun`, paths, spaces, punctuation,
   and case changes.
4. **Same-chunk intersection.** Given inspected constants, prove that
   `{module, main}` in one chunk yields one candidate, the two constants split
   across chunks yield none, repeated constants in one chunk still yield one,
   and two chunks each containing both constants are ambiguous.
5. **No `main` fan-out.** Seed several unrelated job chunks containing `main`
   and assert that a dotted marker links only through its module half. This is
   the regression test that prevents the dangerous callback-only join.
6. **Alias narrowing.** Test an exact, case-sensitive authoritative module
   alias, a stale alias whose chunk lacks the callback, and conflicting aliases.
   A stale or conflicting alias must not silently link.
7. **Level projection integration.** Build the content fixture and assert five
   callback-only events on the five marker IDs, with empty event names and the
   exact trigger component/slot, followed by the five chunk links in the table
   above. Assert that no other Cryos marker receives one of these callbacks.
8. **Existing behavior.** Keep the horde event/callback pairing and obelisk
   tests. Assert that level 9 retains its ten obelisk script rows, gains exactly
   five tutorial rows, and does not create Lua links for the native horde
   callbacks.
9. **Authority preservation.** Assert that all five links are indexed while
   the three `serverOnly=true` and two `serverOnly=false` component flags remain
   distinguishable. Linking must not imply execution authority.
10. **Determinism and diagnostics.** Rebuild twice and compare event/link
    ordering and IDs. Zero-match and ambiguous locators must have stable,
    testable diagnostics and must never depend on map iteration order.

Until those tests pass, the builder should not be changed: correcting only the
dotted-name lookup would attach valid Lua chunks to component events whose
marker ownership is currently unreliable.
