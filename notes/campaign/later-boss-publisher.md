# Later campaign boss publisher

## Result

No retail publisher condition can be recovered for any of the 21
`DirectorTrigger_SpawnBoss` listeners used by the 23 post-1-1 campaign source
levels. Build 103 proves that these are named-event consumers and that the
missing authority is server-side, but it does not contain the server branch
that decides when to publish. Imported content proves each level's event name,
listener scope, boss anchor, and same-event `HordeSpawner_Register` anchors.
Packaged Lua proves a negative result: no recovered chunk publishes any of
these names. The available walkthroughs constrain encounter chronology in four
levels but do not expose the causal server condition, packet, or replay policy.

Accordingly, the current "highest-ordinal authored horde completed" behavior is
still a playability fallback, not recovered retail behavior. It must not be
promoted to per-level evidence. Marker-set ordinal is package order, there is
no authored edge from an ordinary horde's `horde complete` state to the boss
event, and the closest ordinary horde trigger to a boss anchor is generally
hundreds or thousands of world units away.

The implementation-ready boundary is therefore:

1. Retain the exact per-level named-event keys and scoped listener sets below.
2. Keep publisher authority separate from listener planning.
3. Treat accepted publication as encounter-scoped and idempotent as a
   conservative compatibility policy, not because the listener rows prove
   once-only publication.
4. Do not claim a retail captain selector, one creature per add anchor, a
   single phase, or any wave count from this evidence.
5. Keep the current final-horde publisher and captain/add construction
   explicitly labeled fallbacks until a retail server trace or server
   implementation is recovered.

## Scope

The build-103 chain catalog has 24 distinct source levels in its first 24
entries. The initial level, `zelems_1`, uses
`nTutorial_SoloSupportUnlock.main` rather than
`DirectorTrigger_SpawnBoss`. Two later levels also use exceptional Lua
listeners instead of that callback:

- chain position 3, `nocturna_4`: event `lvl 4 boss arena`,
  `nTutorial_CatalystUnlock.main`, plus five
  `HordeSpawner_Register` siblings;
- chain position 5, `verdanth_1`: event `boss triggered`,
  `nTutorial_OverdriveUnlock.main`, plus three
  `HordeSpawner_Register` siblings.

Those two exceptional later levels need their own publisher recovery. They are
not evidence for the 21 `DirectorTrigger_SpawnBoss` rows below. The imported
catalog also contains nine `DirectorTrigger_SpawnBoss` rows in tutorial,
Star Mode, Spectra, and TNX test/non-chain levels; they are outside this
campaign task.

## Proven listener inventory

Every row below is content-proven from recipe-43 `content.db`. `Adds` is only
the count of same-marker-set, same-event `HordeSpawner_Register` listener
anchors. It is not a creature count.

| Chain | Level | Marker set | Boss marker ID | Event name | Adds |
| ---: | --- | --- | ---: | --- | ---: |
| 2 | `zelems_3` | `zelems_3_design.Markerset` | `53828967` | `pB` | 0 |
| 4 | `nocturna_1` | `nocturna_1_design.Markerset` | `1863216012` | `pB` | 0 |
| 6 | `verdanth_3` | `verdanth_3_design.Markerset` | `2529828807` | ` B` (leading space) | 0 |
| 7 | `zelems_2` | `zelems_2_design.Markerset` | `3673893878` | `HB` | 0 |
| 8 | `zelems_4` | `zelems_4_design.Markerset` | `487967850` | `zelems nexus lvl 3 boss triggered` | 3 |
| 9 | `cryos_4` | `cryos_4_design.Markerset` | `67129424` | `boss triggered level 3 cryos` | 3 |
| 10 | `cryos_3` | `cryos_3_design.Markerset` | `1761457013` | `boss triggered` | 3 |
| 11 | `verdanth_2` | `verdanth_2_design.Markerset` | `1526579104` | `boss triggered` | 2 |
| 12 | `verdanth_4` | `verdanth_4_design.Markerset` | `3819071901` | `boss triggered` | 3 |
| 13 | `infinity_2` | `infinity_2_Design.Markerset` | `3261815` | `pB` | 0 |
| 14 | `infinity_3` | `infinity_3_Design.Markerset` | `969850532` | `HB` | 0 |
| 15 | `cryos_1` | `cryos_1_design_spawners.Markerset` | `2144473995` | `pB` | 0 |
| 16 | `cryos_2` | `cryos_2_design.Markerset` | `2145860945` | ` B` (leading space) | 0 |
| 17 | `nocturna_3` | `nocturna_3_design.Markerset` | `48742998` | `boss triggered lvl 3` | 7 |
| 18 | `nocturna_2` | `nocturna_2_design.Markerset` | `235370434` | ` B` (leading space) | 0 |
| 19 | `infinity_1` | `infinity_1_design.Markerset` | `3358407950` | `4B` | 0 |
| 20 | `infinity_4` | `infinity_4_Design.Markerset` | `2438446807` | `HB` | 0 |
| 21 | `scaldron_2` | `scaldron_2_Design.Markerset` | `2265014598` | `boss triggered` | 5 |
| 22 | `scaldron_1` | `scaldron_1_design.Markerset` | `1302350299` | `boss triggered` | 6 |
| 23 | `scaldron_3` | `scaldron_3_Design.Markerset` | `2656448186` | `boss triggered` | 8 |
| 24 | `scaldron_4` | `scaldron_4_Design.Markerset` | `3051320805` | `pB` | 0 |

The short names `pB`, `HB`, `4B`, and the leading-space ` B` are preserved raw
content identities. No recovered schema or code expands them, so assigning
meanings such as "player boss" or "horde boss" would be guesswork.

### `zelems_4` actor identity

The missing generic publisher does not leave the 2-4 actor identity ambiguous.
Build 103 contains the dedicated `ZelemBoss`, `ZelemBoss_2`, and `ZelemBoss_3`
family, directly localized as Polaris, The Gravity Manipulator and backed by a
Polaris-specific phase graph and Lua abilities. The three nouns are the
difficulty tiers with 1500/1750/2000 health; they are not successive encounter
phases. The base-band `ZelemSpecialTwo_Captain` row is the ordinary director
captain Edict/Magnetic Master and is not the final boss. At the base, second,
and third authored `zelems_4` occurrences, publication of the exact event above
therefore constructs `ZelemBoss`, `ZelemBoss_2`, or `ZelemBoss_3` at marker
`487967850` and uses its three same-event add listener anchors. Only the
condition that causes the missing server to publish that event remains a
compatibility fallback.

## Publisher identity

### Proven for every level

- The boss row is an object-lifetime named-event listener, not an authored
  trigger volume. Its normalized radius is zero, and the raw event/callback
  pair occupies the listener layout.
- The listener is registered with the owning
  `SpawnPoint_DirectorBoss.Noun` object as context. Native event dispatch calls
  the resolved native callback and then any Lua callback with
  `(sourceObjectID, listenerObjectID, otherObjectID)`.
- Same-name lookup is scoped to the owning marker set. There is no additional
  textual publisher row beside any of the boss/add listener groups.
- `DirectorTrigger_SpawnBoss` and `HordeTrigger_OnEnterPlayer` are deliberately
  inert client registrations in build 103. `DirectorTrigger_SpawnBoss ->
  sub_9FACF0` rejects the remote/non-authoritative branch, validates a player
  object, obtains the simulator/director address, and returns false without
  event dispatch, state mutation, spawning, or a game message.
- `HordeSpawner_Register` occurs in neither the build-103 executable string
  corpus nor packaged Lua. Its implementation belongs to the missing server.

### Not recovered for any level

The actual publisher object or subsystem, its input condition, the
`sourceObjectID` and `otherObjectID` supplied at dispatch, and its ordering
relative to alerts and object creation are all absent. No level has evidence
that the publisher is:

- completion of its highest-ordinal ordinary horde;
- completion of every earlier horde;
- entry into a fabricated radius around the boss anchor;
- depletion of all ambient enemies;
- a fixed mission clock;
- a packaged Lua callback; or
- a client-authored notification.

The complete build-103 client publisher inventory has six direct calls to the
generic event dispatcher: interaction start/end, Lua `NotifyObjects`, optional
interactable notification, and trigger-volume enter/exit/stay branches. None
can supply these boss names from the recovered content. A missing server can
call the same event map indirectly, which is the remaining authority boundary.

## Timing and prerequisite encounters

No exact publication time or prerequisite encounter is proven for any of the
21 levels. The imported boss listener has no timer, trigger radius, predecessor
ID, objective condition, horde-set reference, or route edge. Ordinary horde
triggers publish `horde triggered`, not the level-specific boss event.

The highest-ordinal fallback has no spatial or authored causal support. Across
the 21 levels, the nearest ordinary `HordeTrigger_OnEnterPlayer` centers are
about `216` to `2031` units from the boss anchors; examples include
`zelems_3` at `1015.73`, `cryos_3` at `1862.81`, `verdanth_2` at `460.70`,
`verdanth_4` at `386.20`, and `infinity_4` at `2030.78` units. This does not
prove that an earlier horde is never a progression prerequisite, but it
excludes treating marker-set ordinal or arena proximity as the missing edge.

Existing footage gives only these level-specific constraints:

| Level | Proven visible chronology | What remains unknown |
| --- | --- | --- |
| `zelems_3` (walkthrough 1-2) | The support unlock is visible near `06:40`; traversal continues for roughly six minutes. `Horde incoming!`, Zunh the Singularity Void, and additional enemies appear in the final arena near `12:39`; the boss bar clears near `13:12`. | The boss event is independent of the support-unlock job. The exact event time, arena-entry condition, preceding encounter predicate, and whether the alert precedes or follows publication are not captured. |
| `cryos_3` (walkthrough 3-2) | A sustained mixed-elite final encounter runs about `13:40-14:55`; the last enemy falling exposes completion. No durable named boss is visible. | Whether this encounter is the `DirectorTrigger_SpawnBoss` result, a prerequisite phase, or both; exact publication and phase boundaries. |
| `verdanth_2` (walkthrough 3-3) | A difficult elite lead-in runs about `12:20-13:30`; Yegg the Hypno Lord is then visible through about `14:45`. | The chronology permits a lead-in/boss phase interpretation but does not prove a server dependency, named-event time, or whether lead-in survivors are boss-event adds. |
| `verdanth_4` (walkthrough 3-4) | Orcus and recurring `Servant of Orcus` enemies are visible from about `11:40` until Orcus dies near `15:30`. | The first boss-event time, any prerequisite clear, and whether servants come from fixed director anchors, the boss noun's abilities, or both. |

There is no retained walkthrough evidence for the other 17 levels that can
resolve publication timing or prerequisites.

## Re-entry and multiplicity

The boss and add listener rows have `is_trigger_once_only=0`, but that field
describes the listener record and cannot prove how often the absent publisher
fires. Object activation registers the listener for its lifetime, and teardown
removes it. Therefore repeated server publication could invoke it repeatedly;
the client gives no encounter lockout policy.

The footage contains one observed completion pass per available level and
cannot distinguish:

- publish once per match;
- publish once after a prerequisite transition;
- suppress while the boss encounter is active;
- retry after a failed/rolled-back publication;
- replay after leaving and re-entering an arena; or
- reconstruct without republishing for a late join.

For compatibility, one accepted boss publication per level-instance encounter
is the conservative policy. Failed atomic admission should remain retryable,
while an accepted active or completed encounter should suppress duplicates.
That is an implementation recommendation, not recovered retail re-entry
evidence.

## Captain selection, adds, and phases

### Captain selection

Imported later-level director data supplies four recurring configuration
ordinals and three difficulty bands, but stores their authored labels as
`unknown`. Darkspin's projection of those ordinals to `minion`, `special`,
`agent`, and `captain` is a documented structural fallback; only 1-1 has
independently named semantics. The content does not connect any configuration
ordinal to `DirectorTrigger_SpawnBoss`.

Build 103 contains a dormant mode-5 selector that filters a cached pool by
difficulty, samples uniformly with replacement, applies an increasing cost,
and caps its noun list at 15. Its only caller is not the boss callback, and no
recovered edge supplies a later-level boss budget. It cannot select the retail
captain here.

Walkthrough names such as Zunh, Yegg, and Orcus prove visible encounter
identities only. They do not map the displayed title to a director noun,
selection seed, mutation/affix roll, or captain-pool entry. Darkspin therefore
uses its public named-boss catalog for the base band and selects each later
band's exact sole captain entry instead of synthesizing the base noun's suffix.
The independently recovered Polaris family remains the explicit `zelems_4`
exception.

### Add count

The table's `Adds` count proves only the number of matching spatial listener
anchors. Nothing proves that each listener creates exactly one actor, that
every listener is used in one phase, or that listeners are invoked in marker
order. The strongest counterexample is `zelems_3`: its `pB` graph has zero
`HordeSpawner_Register` siblings, while the 1-2 footage visibly shows
additional enemies with Zunh. Conversely, Orcus can visibly produce recurring
servants, which may be noun-owned behavior rather than fixed director adds.

One projected agent per add anchor is therefore a diagnostic construction, not
a recovered count. The base-cycle `zelems_3` plan now has one explicit
walkthrough-backed exception: because the footage proves plural enemies beside
Zunh while the `pB` graph has zero add listeners, Darkspin chooses two agent
entries and places them at the nearest two authored horde loci in the owning
marker set. If fewer than two usable loci survive content preparation, bounded
three-unit offsets from the boss anchor preserve playability. Two is the
minimum count supported by the plural observation; exact nouns, positions,
budget, and wave timing are absent from build 103 and require a retail server
trace or implementation rather than being attributed to `pB`. The selected
fallback is recorded as `campaign-zunh-adds` in `notes/help.md`.

### Wave phases

No boss event row stores a wave count, delay, clear predicate, or phase
transition. No packaged Lua implements the callback. Build-103 director fields
and selection primitives prove that generic horde/wave machinery exists, but
not that a particular later boss uses it or with what values.

The footage proves only presentation-level diversity:

- Zunh appears with adds despite a zero-add listener graph; Darkspin now
  preserves that minimum visible composition as the explicit fallback above;
- `cryos_3` ends in a mixed-elite clear without a durable named boss;
- Yegg follows a visible elite lead-in;
- Orcus receives recurring servants during a long boss fight.

Those observations rule out one universal, evidence-backed "leader plus fixed
adds in one phase" retail model. They do not recover exact per-level phases.

## Packaged Lua negative result

The recipe-43 database contains 1,029 indexed Lua chunks. Exact
`lua_string_constant` lookups return no rows for:

- `NotifyObjects`;
- `DirectorTrigger_SpawnBoss`;
- `HordeSpawner_Register`;
- any of the 21 boss event names, including `pB`, `HB`, `4B`, and ` B`.

The only `BossDead` string is chunk `618`, bytecode SHA-256
`c1d29a663a81885e40631b0600dd46bc28b76ce3fc11cce54c3c1c95dd895a89`.
It registers the generic condition and returns false; it does not publish a
boss-start event. No chunk contains `IsBossDead`. The known tutorial tails that
call `ActivateHordeSpawn` reach a build-103 native no-op and contain none of
the later `DirectorTrigger_SpawnBoss` event names.

Thus Lua can be excluded as the recovered publisher for all 21 rows. This does
not exclude missing server-native logic that is not shipped in the client
content.

## Evidence and confidence

- `bin/darkspinner/darkspin/cache/content.db`, recipe `43`, source build `103`,
  input fingerprint
  `b088d6767e4485d90ff108c4eb091dd729b5cc8604b99b459ccc94cd1223b3ec`,
  file SHA-256
  `e9a34d31a8a969ff8bc857cb368face13f198f6fd3a1f757c32e8948911f1cc7`:
  chain scope, marker/event rows, listener fan-out, Lua constants, and director
  configurations.
- `bin/game/GameBin/Game.c`, SHA-256
  `1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb`:
  `sub_9FACF0`, event-listener registration/dispatch, and client-native
  boundaries.
- `bin/game/logs/ida-event-publisher-xrefs.log`, SHA-256
  `06e726fe71899b5bda0765db089f8bd0a1905fa2baeee7d844437996f4ba22b3`:
  complete direct client event-dispatch caller inventory.
- `bin/game/logs/ida-director-xrefs.log`, SHA-256
  `a4b25b156952d33690a8b6b212f82ee64f72d4bd94053009d89c31c963c1707b`:
  callback registration, missing listener string, reflected director fields,
  and unconnected selector boundary.
- `bin/video/walkthrough/1-2/info.md`,
  `bin/video/walkthrough/3-2/info.md`,
  `bin/video/walkthrough/3-3/info.md`, and
  `bin/video/walkthrough/3-4/info.md`: visible encounter chronology and Lua
  linkage negatives.

Confidence is high for the listener inventory, client/Lua negative result, and
the conclusion that the current final-horde condition is not recovered retail
policy. Confidence is intentionally zero for a concrete retail publisher,
exact timing, prerequisites, re-entry, captain selection, actor count, and
phase plan because the required server evidence is absent.

## Remaining decisive capture

A retail server trace or recovered server implementation must provide, for
each event:

1. the state transition that chooses the event name;
2. source/listener/other object IDs and phase generation;
3. ordering relative to `Horde incoming!`, object creation, and boss-active
   director reflection;
4. prerequisite encounter and optional-route state;
5. duplicate/re-entry and late-join behavior;
6. selected noun, mutations/title, budget, listener allocation, and phase
   counters.

Without one of those sources, further client-content analysis can refine
anchors and visible chronology but cannot turn the fallback publisher into a
proven per-level contract.
