# Build-103 ship management reconstruction

## Working profile

The local `Test` profile is the canonical established ship fixture. Its
tutorial-owned rewards were first restored through the server-owned
`CompleteTutorial` operation; after native ship-flow capture proved that the
persisted `6800` lesson boundary could not resume, the stopped runtime was
advanced to the client's established-profile boundary:

- `new_player_progress = 9000`, `new_player_inventory = 1`, cumulative XP
  `335`, level `3`, and DNA `100`;
- no remaining creature reward after the native Arsenal selection;
- Blitz Alpha (`creature.id = 1`), Sage Alpha (`creature.id = 2`), and Wraith
  Alpha (`creature.id = 3`);
- PvE squad slot 1 contains creature IDs `[1, 2, 3]`.

Run the same idempotent operation while Darkspinner is stopped with:

```text
darkrun db user complete Test --config bin/darkspinner/darkspin.toml
```

That command now applies the local no-ship-guides completion policy: it stores
the established boundary `9000`, inventory marker `1`, a usable three-hero PvE
squad, and no pending activation reward. Do not use the intermediate `6800`
state as a skip target.

Do not run offline profile maintenance beside a live server. The server owns an
in-memory aggregate and its later logout/flush can correctly replace the same
SQLite children with that older aggregate.

Any tutorial skip or recovery must restore the tutorial-owned rewards as well
as its progress boundary. At minimum this means cumulative XP/level, DNA, Blitz,
Sage, Wraith, a complete squad layout, and every authored
tutorial inventory/part reward proven by the completed route. Advancing only
`new_player_progress` creates a superficially complete but unusable profile;
The proven level-5 basic Electro Claws reward (rigblock metadata ID `268`) is
therefore granted idempotently by `CompleteTutorial`; it receives stable item
and reference IDs and participates in the same rollback boundary. Completion
also equips an unequipped copy to Blitz, including repair of profiles created
before that association was implemented. It never moves the reward if the
player has already equipped it to another creature. The visible
Onyx Barrier reward still lacks a recovered concrete asset ID, so full
inventory/part parity remains an explicit verification item for the skip path.

## Live boundary captured on 2026-07-20

With `AUTO-LAUNCH WHEN READY` enabled, Darkspinner authenticated and launched
`Test` without a PLAY click. The client then:

1. completed server login;
2. requested `api.inventory.getPartList` with `count=10000` and no owned or
   creature filter;
3. loaded the ship scene, reached `scene_ready=1` and `ui_ready=1`;
4. emitted screen event `0x1002` (ship navigation ready).

The empty inventory result is valid for this fixture and is not the crash
boundary. Both an empty roster and a correctly persisted Blitz/Sage roster
reached the same point.

When Fang's cinematic skip was enabled at progress `3000`, its navigation hook
called the weakly typed Flash callback at VA `0x0052CFD0`. The process faulted
inside that call before the following `ship_start` trace could be written. With
cinematic skip disabled, the same client remained connected and responsive in
the native ship flow. Fang now defers automatic START at exactly progress
`3000`, leaving the first post-tutorial Arsenal lesson to the native UI; later
established-profile progress values retain automatic room entry.

## Client-side map

These anchors are verified against `bin/game/GameBin/Game.c`:

| Address/function | Ship responsibility |
| --- | --- |
| `sub_52CDD0`, VA `0x0052CDD0` | Initializes navigation and gates rooms by `new_player_progress`. Progress `3000` publishes `NewPlayerUnlockedCollectionRoom`. |
| weak `sub_52CFD0`, VA `0x0052CFD0` | `SpaceshipNavigation.OnGoToRoomClicked` Flash callback. Its recovered C prototype is not trustworthy enough for a synthetic direct call at the progress-3000 lesson boundary. |
| `sub_5187D0` | Reads in-memory `new_player_progress`. |
| `sub_4B7100(0)` | Ship/editor entry submission for `api.inventory.getPartList`; see `notes/items/inventory-load.md` for the exact request and response fields. |

Progress `3000` means tutorial combat is complete, but ship-room onboarding is
not complete. The client continues the room/editor/map lesson sequence through
later progress values, so it must not be treated as an established unrestricted
ship profile.

The native ship flow submitted `6000` on entering the Editor and `6500` on
unlocking Navigation. The server accepts authenticated, monotonic submissions
at the recovered Build-103 milestones `3000`, `4000`, `5000`, `6000`, `6500`,
`6800`, `8000`, and `9000`. Progress below `3000`, regressions, and unknown
milestones remain server-owned/rejected. Progress and inventory markers are
persisted as one operation and rolled back together if storage fails.

Build-103 client control flow explains the otherwise stuck `6800` profile.
`sub_527810` advances `6500 -> 6800` whenever the map initializer first runs,
but advances `6800 -> 8000` only on a later invocation whose prior internal
state byte at offset `+64` is already nonzero. Restoring the process directly
at `6800` loses that transient prerequisite, leaving the Navigation pad lit but
unable to finish the lesson. The normal established-profile startup path near
`sub_448D80` writes `9000` directly. Separately, `sub_518860` sets the account
inventory marker to `1` after its one-time inventory-flow condition succeeds.
These are client-state findings, not evidence that retail tutorial completion
advanced beyond the recovered `3000` combat boundary. Darkspin nevertheless
uses `9000` as an explicit server policy after a successful tutorial because
the resumable ship lessons have repeatedly produced soft locks. The completion
transaction also sets inventory marker `1`, ensures Blitz, Sage, and Wraith are
owned, fills the first PvE squad, grants one activation choice on the first
successful completion, and grants the known tutorial part before the save. The
choice reconciles the server's terminal `9000` policy with the live client's
positive-completion transition into its native Arsenal lesson; after that
selection, the established stored balance is again zero. This deliberate
compatibility policy is separate from the recovered retail contract.

Client navigation code associates selector `4` with the Map Room/Navigation
path (`sub_518810(4)` and the Navigation timing branch in `sub_52CAA0`), but that
does not make it safe to pass `4` through Fang's weakly typed synthetic room
callback. A rebuilt `9000` run reached `ship_navigation_ready` and read the
correct progress, then disconnected before Fang could record `ship_start`.
Repeating the experiment with the old selector `1` at progress `9000` failed at
the same boundary, proving that no synthetic post-tutorial room selection is
currently live-safe. Fang now defers its room callback for every progress value
at or above `3000`; it still accelerates camera presentation and automatically
starts the actual tutorial at `0..2000`. Established profiles use the native
bottom-right Navigation control. Direct room acceleration remains unimplemented
until the callback argument and room-transition ownership are recovered.

An interrupted `3000` profile was live-confirmed with three occupied PvE squad
positions, an already-activated Wraith, and no activation choice. For the
genuine three-owned-hero state, profile start atomically clears slot 3 and
restores one activation choice so the native Arsenal lesson can activate its
third hero. A later Test profile already owned Blitz, Sage, Wraith, and Goliath;
repeating that repair produced a phantom reward which could not select any
hero, soft-locking the mandatory popup. Profiles at `3000` which already own at
least four heroes now bypass the impossible activation step by advancing to
`4000`, consuming the phantom reward, and filling an empty third PvE position
from an owned reserve. This was live-verified from the stuck Test database:
login changed progress `3000 -> 4000`, reward `1 -> 0`, and restored Goliath to
slot 3 without altering level 3, `335` XP, or `100` DNA. Both recovery paths
persist atomically; persistence failure restores the account and squad and
rejects login rather than exposing a half-repaired profile.

## Matchmaking cancel crash

The Navigation `Invite` action emitted Fang command `6`, while starting and
cancelling Matchmaker used Blaze GameManager commands `0x0d` and `0x0e`. No game
or matchmaking object was created; the server returned synthetic session ID
`1`. The Build-103 client exited immediately after cancellation.

Comparison with the reference GameManager contract found two defects in the
`NotifyMatchmakingFailed` (`0x0a`) payload: `USID` was absent and `RSLT` was `1`
(`JoinedNewGame`) rather than `4` (`Cancelled`). The corrected notification is
`MAXF=0, MSID=<request>, RSLT=4, USID=<authenticated account id>`. Start and
cancel now also emit semantic `matchmaking_start` and `matchmaking_cancel` logs
so the next live verification can distinguish the request, reply, and
post-reply notification.

## Protocol surface

Ship management uses multiple transports, and captures must keep them distinct:

- HTTP/XML `/game/api` owns account, deck, creature, and persistent part-list
  operations.
- Blaze owns the authenticated session and disconnect lifecycle, but no ordinary
  ship inventory list has been observed there.
- RakNet owns active gameplay. Its part Drop command is relevant after returning
  from a game, not for initial Arsenal population.
- Flash callbacks and native cache filtering can change ship UI without any
  packet. Opening Inventory and switching its tabs are known examples.

The server now logs every unknown `/game/api` compatibility fallback as
`game_api_compat`, including the method, authenticated account, and sorted
non-credential field names. Values and authentication fields are deliberately
excluded. This is the starting capture boundary for unknown Arsenal, Collection,
Editor, and Map Room actions: exercise one UI action at a time, then promote a
method from the compatibility fallback only after its command fields and client
response parser are verified.

## Live Editor and Objectives findings

The repaired Arsenal route activated Goliath, added it to the third PvE squad
position, and entered Editor. Editor entry requested the owned part list and
submitted onboarding milestone `6000`; both succeeded. The lower-left `0/0`
counter is DNA, not part capacity, and proves the skipped profile is missing
the tutorial's `100` DNA grant. `CompleteTutorial` now restores a monotonic DNA
floor of `100` in the same transaction as XP, heroes, squad, and Electro Claws.

The stored level-5 Electro Claws row was returned by `getPartList`, but the
client initially rejected it because the HTTP projection sent numeric catalog
ID `268` where build 103 resolves a hashed asset identity. The ordinary list
now sends `RigblockAssetHash` and the three affix hashes. Account XML also sends
the computed slot capacity through both inventory wire fields; the persisted
`unlock_inventory` scalar is an upgrade tier, not the capacity expected by the
client's `unlock_inventory` element. Live verification after rebuilding showed
the Electro Claws icon, `Capacity: 1/180`, `Equipped: 0`, and DNA `100` on the
ship Inventory screen.

Changing Blitz's appearance and pressing Apply caused another part-list reload
and advanced onboarding from `6000` to `6500`, but creature row 1 remained at
version 1 with unchanged stored fields. `api.creature.updateCreature` is still
only a compatibility acknowledgement and discards the submitted editor state.
It now emits the same redacted field-name diagnostic as other compatibility
methods so a rebuilt run can classify the request without logging thumbnails,
render blobs, or field contents.

The Navigation Objectives modal is client-local static onboarding copy: build
a Genetic Hero collection, upgrade heroes with items, chain Threat Levels for
stronger items, and destroy the Game. Opening and closing it produced no
HTTP or semantic server event. The underlying destination screen enables
Campaign/Onslaught, disables PvP and two other mode icons, and maps consecutive
threat nodes across squad slots. Nodes beyond available progression disable
Start and Matchmaker with a prerequisite message; selection itself is local.
Invite opens the Social Friends/Lobby/Ignore overlay, whose empty lists also
produce no semantic server log in the current build.

## Live Campaign 1-1 boundary

Selecting Campaign, node 1-1, Start, and Continue produced a Blaze dedicated
server reset with `GameType=2`, `SelectedDifficulty=1`, mode `0`, and no level
asset string. The authenticated RakNet peer joined slot 0, received a chain
vote, then sent the six-byte chain prepare command followed by player status
updates `2` and `4`, both with progress `1`. The client remained at the ship
loading spinner.

The first concrete defect was server-side level resolution: mode `0` with an
empty level caused `GamePrepareForStart` to hash `".Level"` and
`"_ai.Markerset"`. For a completed account at chain progression `0`, the
server now recognizes the captured `GameType=2` request as campaign mode and
selects the authored first level `zelems_1`. Later chain-progression-to-level
selection and the non-tutorial dungeon bootstrap remain separate unimplemented
boundaries; they must not silently reuse tutorial objectives, spawns, or game
state after the level handoff.

The rebuilt live run confirmed the resolved binding in both Blaze and RakNet:
`game_create` reported `level="zelems_1"`, the deployment screen named Floating
Isles, and the prepare payload changed from the invalid empty-level hashes to
the authored level and markerset hashes. The client then repeated the same
status `2` -> status `4` sequence and waited because the existing player-status
handler discarded every non-tutorial update. Campaign status `4` now repeats
the resolved prepare packet as an evidence-bounded handshake response. It does
not enter the dungeon stage or publish any tutorial-only player, objective, or
spawn state; the next client message after that response is the next capture
boundary.

That prepare-only response was live-tested: the client ACKed it but emitted no
new application command. Pairing status `4` with the non-initial
`LabsPlayerUpdate` status/progress delta, followed by the resolved prepare
packet, crossed that boundary. The rebuilt client immediately advanced to
status `8` and entered its black level transition.

An isolated status `8` experiment echoed the player delta, sent `GameStart`,
then answered the timestamp ping with only Chain `GameState`, `DirectorState`,
and a QuickGame reset. The client terminated immediately after acknowledging
that bundle. Fang recorded `creature_asset_lookup_input asset_id=0` immediately
before the access violation, proving that the state trio exposed a missing
player roster rather than being independently invalid.

The campaign binding now carries the selected default PvE squad's three
persistent IDs, nouns, and versions. Status `2` publishes one complete three-character
`LabsPlayerUpdate`; RakNet correctly fragments its application
message into four transport frames. The first live run accepted all four,
advanced through status `4` and `8`, and resolved every subsequent creature
asset lookup. Tutorial-only characters still use their recovered opaque
appearance identities; persistent campaign characters use their stored creature
IDs as roster asset identities so two selected heroes cannot alias the same
client character record.

With the full roster sent first, the status `8` player delta, `GameStart`, and
timestamp-triggered Chain `GameState` / `DirectorState` / QuickGame sequence is
live-safe and loads Floating Isles. The first successful state-only run showed
the world and three squad portraits but no controlled hero. The next bounded
bootstrap now creates selected slot zero as object `1`, refreshes the
LabsPlayer controlled-object handle only after creation, publishes its initial
position and hero state, and deploys it. The start position
`(-123.8707,-151.64705,10.037109)` is authored `zelems_1_design.Markerset`
marker `SpawnPoint_Affix.Noun`, adjacent to the four authored camera spawn
points. A rebuilt live run stayed connected, displayed the deployed portrait,
abilities, minimap player dot, and followed a click toward the start platform.
Campaign movement commands now receive the ordinary authoritative object-move
and unreliable-locomotion echo without entering tutorial route, objective,
orb, encounter, or progression logic.

The first reserve portrait click exposed another mode leak: campaign
`ActionSwitchCharacter` was entering tutorial unlock and Sage-passive checks,
so the server rejected the request. Campaign setup now creates one distinct
player object for each of the three selected squad slots at the authored start,
hides slots one and two, and deploys slot zero. A campaign-only switch accepts
an available reserve, hides the old object, teleports and reveals the target,
refreshes the controlled-object handle, and publishes the selected creature
index. It has no tutorial objectives, passive summons, or unlock rules. The
focused handler test covers the three-object bootstrap, movement, and the
slot-zero to slot-one handoff. A live established-profile run then loaded
Floating Isles with all three portraits, switched slot zero to slot one, and,
after the authored 30-second swap cooldown, switched slot one to slot two. The
third portrait's first click during cooldown changed its local highlight but
was correctly rejected by the server; retrying after cooldown deployed the
third persistent creature and updated its portrait, model, and ability bar.

The loaded Floating Isles currently has no encounter population. That is not a
reserve-character defect: campaign director/object state has not yet been
recovered or implemented, so authored terrain loads but enemies, interactables,
objectives, and completion flow do not. Populate those from campaign captures
rather than importing the tutorial encounter route.

The first campaign-director prerequisite is now recovered in the content
importer. Level binaries interleave a run of fixed 16-byte eligibility records
with its matching contiguous `.Noun` strings; the former prefix-only decoder
discarded every level whose later configuration introduced more nouns. The
corrected decoder recovers all 24 `zelems_1` entries across four nine/three-row
configuration groups, including exact difficulty bands and the horde-legal
flag, and all six tutorial entries across three structural pools. The runtime
database now retains both the configuration ordinal and the entry ordinal
inside that configuration rather than flattening those boundaries away.
This establishes eligible assets but still does not prove the selected group,
budget, count, placement policy, or attack ownership for the first 1-1 spawn.

Build-103 reflection now supplies semantic names for those four `zelems_1`
configuration ordinals: `minion`, `special`, `agent`, and `captain`. The
content importer stores those names directly and leaves other levels unknown.
It also leaves entry-level `spawn_kind` unknown: native marker registration
maps `SpawnPoint_DirectorHorde.Noun` to kind `5`, which selects the `agent`
pool, but that marker fact does not make every agent entry itself spawn kind
`5`.

That marker relationship is now retained explicitly in campaign setup. Every
`SpawnPoint_DirectorHorde.Noun` placement carries known spawn kind `5` and
references the recovered `agent` pool; other director marker nouns remain
unknown. Setup rejects a known marker whose referenced pool is absent. It now
consumes the match's validated `SelectedDifficulty` and filters every pool by
the authored inclusive interval, but still performs no budget selection,
object creation, or event activation.

The feature model exposes the native-proven inclusive difficulty filter as a
pure candidate query. It resolves exactly one named pool, retains authored
entry order, and includes an entry when `minimum <= difficulty <= maximum`.
The query deliberately returns the stored `is_horde_legal` bit without using
it as a filter because that bit's role in the kind-5 native branch has not yet
been instruction-proven. Campaign startup now applies the same predicate to
all loaded pools using the typed gameplay difficulty. `Game.c` shows the
client copying its selected string directly into the `SelectedDifficulty`
Blaze attribute; malformed, zero, and greater-than-100 chain values are
rejected before gameplay binding. The numeric spawn budget remains open.

The immutable content store now exposes that projection through one
`LevelDirector` read: structural candidate pools and every
`SpawnPoint_Director*.Noun` marker are returned together in authored
marker-set/entry order. Marker-set boundaries, names, ordinals, and weights are
retained rather than flattened. This is intentionally a read-only input and
contains no selection or AI policy. For `zelems_1`, 405 director placements are
owned by 18 of the level's 41 marker sets. The three Wanderer sets own
`106/141/94`; the Spike sets own smaller route-local clusters; Horde 1 and 2
own `3/2`. `zelems_1_design_spawners.Markerset` owns four director-horde
placements plus one director-boss placement near `(940, 670)`, proving it is a
far-end cluster rather than the opening encounter. Those boundaries must not
be collapsed into one global spawn population.

Director placements now also retain their ordered authored event bindings
through the content and campaign feature projections. This is metadata only;
campaign setup does not execute a callback or convert a listener into a spawn.
Runtime `content.db` confirms that the four far-end horde placements listen for
`boss triggered`, while Horde 1's three and Horde 2's two placements listen for
`horde triggered`; all nine call `HordeSpawner_Register`. The Wanderer and
Spike placements do not inherit those bindings. Consequently none of those
nine horde placements is an unconditional opening population, and status `8`
must continue to leave them dormant until the corresponding authored trigger
flow is implemented.

The projection also retains the two distinct player-entry placements from
`zelems_1_Ai_Horde_1.Markerset` and `zelems_1_Ai_Horde_2.Markerset`. They are
kept as triggers rather than counted as director spawn markers: each carries
its exact position and `HordeTrigger_OnEnterPlayer` event metadata alongside
the `3/2` listener placements owned by the same marker set. Campaign setup
only snapshots these records and logs their count. It does not assume that the
callback publishes `horde triggered`, because the missing server-side callback
implementation, budget, selection, and replication contract remain unrecovered.

Campaign status `8` now crosses a feature-owned `CampaignSetup` boundary before
entering the dungeon stage. The operation rejects non-campaign bindings, loads
the exact resolved level through a content adapter, requires both pools and
placement markers, and snapshots them on the gameplay session. It still emits
the existing player-only bootstrap: no noun is selected and no encounter is
spawned. Runtime logging includes only the pool and marker counts so a live run
can prove the correct content reached the session without claiming recovered
director policy.

### Creature editor update capture

A July 20 live editor capture confirmed that changing only Blitz's skin coat
marks the editor dirty and calls `api.creature.updateCreature` when Apply is
pressed. The request used creature `1`, proposed version `2`, gear score
`0.000`, item points `300.000`, an empty `parts` field, 87 bytes of stats, 376
bytes of ability stats, 47,322 bytes of base64 thumbnail text, and 114,093
bytes of base64 large-image text. This proves that paint-only saves must not
depend on an equipped-part delta.

The implemented update path accepted that request, incremented Blitz from
version 1 to 2, preserved the authoritative gear score and item points, left
the Electro Claws unequipped, and persisted both generated image URLs. The
decoded files are `creature_png/1_2_thumb.png` (35,031 bytes) and
`creature_png/1_2_large.png` (84,459 bytes); the latter visibly contains the
new purple/blue skin-coat accents. The subsequent inventory refresh completed
without an editor error.

## Next captures

1. Re-run Matchmaker then Cancel and verify the corrected Blaze notification no
   longer terminates the client.
2. Capture squad slot selection and deck save separately; compare with the
   implemented `api.deck.updateDecks` command.
3. Enter Collection, Editor, and Map Room one at a time and classify each new
   `game_api_compat` record before adding persistence.
4. Capture equip, unequip, identify, sell, and vendor actions. The current
   `api.inventory.updatePartStatus` and `api.inventory.vendorParts` responses are
   compatibility acknowledgements and must not mutate storage until their exact
   invariants are decoded.
5. Verify each response parser in `Game.c`, then add success, rejection
   without write, and persistence-failure rollback tests for every mutation.
6. Extend the campaign runtime from the live-proven three-character bootstrap
   into authored `zelems_1` director/object state without importing tutorial
   objectives or encounters.
7. Instrument the embedded creature-profile callback/bridge boundary; the
   endpoint now returns populated creature XML, but the live profile page still
   renders blank after the documented schema, locale, owner, and timing probes.
