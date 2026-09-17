# Build 103 campaign 1-1 boss, exit, and completion

## Result

### Walkthrough evidence boundary

`bin/video/walkthrough/1-1/1-1.mkv` and its indexed `info.md` provide direct
visible evidence that the recorded final encounter contains a large-health-bar
enemy named `Illust the Accelerator`, labelled `Swift Aura, Swift`, with adds.
The support unlock appears around `12:50`, the chamber arms around
`12:58-13:10`, and the fight completes around `14:25` before final drops and
the collect-or-continue screen. This resolves the visible placeholder shape
more strongly than marker topology alone. It still does not identify Illust's
internal noun, exact spawn budget, packet sequence, difficulty transform,
clear predicate, or reward transaction, and therefore does not replace the
missing-authority boundaries below.

Build 103 contains enough client and authored-content evidence to recover the
shape of the end of campaign level 1-1, but not enough to recover a complete
authoritative encounter implementation. The strongest supported path is:

1. the two authored horde sublevels run as separately gated encounters;
2. the boss-security teleporter admits the player only after its local security
   scan is clear and sends the player to the final arena;
3. the boss marker publishes or participates in a `boss triggered` boundary,
   four nearby director-horde markers supply candidate spawn locations, and the
   tutorial support-unlock job runs alongside that boundary;
4. server-authored director state eventually sets `mbBossComplete`;
5. that field, not the `LevelExitPoint` marker and not the objective packet,
   enables the client Beam Out button;
6. clicking Beam Out sends `PlayerStatusUpdate(status=32, progress=1)` once;
7. the server must then commit the level result and move the client into chain
   voting/results/cashout, where the client issues the requests described below.

The decisive missing fact is the server rule that turns the final arena from
active to complete. Neither the linked Lua nor the client native callbacks
implement spawning, wave advancement, clear detection, success, failure, or
reward persistence. Those are server-authority choices. The recovered
incomplete C++ server is useful for packet names and candidate layouts only; it
is not evidence of retail ordering.

Evidence labels used below are:

- **Exact**: encoded bytes, decoded bytecode, reflection layout, or an authored
  content edge directly establishes the claim.
- **Content-correlated**: spatial placement or naming gives one strong reading,
  but an identity edge or runtime publisher is missing.
- **Inferred**: the reading best joins independent facts but is not directly
  observable in the available build.
- **Missing authority**: the retail server had to choose the behavior and no
  client/content evidence determines it.

No retail 1-1 gameplay trace covering this sequence is available. Live Cryos
work is used only where it validates shared client behavior, especially the
director completion field and Beam Out request; it is not treated as proof of
1-1 encounter composition.

## Authored end-of-level topology

Level `56` is `zelems_1`. Its base design set is `1791`, its director-marker set
is `1793`, and its two horde sublevels use sets `1827` (`zelems_1_horde_1`) and
`1828` (`zelems_1_horde_2`). The suffixes and level ordinals support horde 1
before horde 2, but do not prove the exact traversed route or prohibit
backtracking.

### Final arena

| Row / authored ID | Object | Position | Relevant authored edge |
| --- | --- | --- | --- |
| `126383` / `3299474850` | `TriggerZone.Noun` | `(947.2180, 675.6621, 0.1637)` | enter callback `nTutorial_SoloSupportUnlockClient.main`, radius `60` |
| `126384` / `2145860737` | `SpawnPoint_DirectorHorde` | `(940.4082, 642.7053, 0.0880)` | `boss triggered -> HordeSpawner_Register` |
| `126385` / `2145860735` | `SpawnPoint_DirectorHorde` | `(959.2801, 698.7213, 0.0880)` | `boss triggered -> HordeSpawner_Register` |
| `126386` / `2145860738` | `SpawnPoint_DirectorHorde` | `(924.6718, 699.2067, 0.0880)` | `boss triggered -> HordeSpawner_Register` |
| `126387` / `2145860736` | `SpawnPoint_DirectorHorde` | `(972.6259, 652.4561, 0.0880)` | `boss triggered -> HordeSpawner_Register` |
| `126388` / `258231375` | `LevelExitPoint` | `(941.0225, 672.3367, 0.1636)` | no normalized event |
| `126389` / `223774364` | `SpawnPoint_DirectorBoss` | `(948.3736, 674.0089, 0.0880)` | `boss triggered -> nTutorial_SoloSupportUnlock.main` |

The four horde markers surrounding the colocated boss marker, support trigger,
and exit marker are the strongest evidence for one final encounter cluster.
They are native director-horde kind `5` markers, which select from the level's
`agent` pool under a budget and a maximum of 15. The boss marker is director
kind `9`, but `zelems_1.Level` has no populated boss-director noun array.
Consequently, the data does **not** establish a distinct `ZelemBoss` actor. The
final fight may be a boss-labelled horde, a server-injected boss plus horde, or
another composition unavailable in client content.

The marker set does not expose a normalized `DirectorTrigger_SpawnBoss`
callback for row `126389`. Its only recovered Lua edge is the support-unlock
job. The listener literal `boss triggered`, the marker type, the four surrounding
listeners, and the trailing `ActivateHordeSpawn` call make this the strongest
activation boundary, but the event publisher and its entry/contact condition
remain missing.

### Horde 1

The set has one trigger, three listener/spawn points, and three gate
teleporters:

- trigger row `128245`, authored ID `2244983668`, at
  `(592.4940, 11.3250, 10.0210)`: `HordeTrigger_OnEnterPlayer`, event
  `horde triggered` on enter and `horde complete` on exit; no positive retail
  activation radius is recovered;
- listener rows `128241`, `128244`, and `128247`:
  `horde triggered -> HordeSpawner_Register`;
- gate rows `128237`, `128239`, and `128240`:
  `HordeGateTeleporter_OnEnter` contact edges;
- blocking-door ordinals `5`, `6`, and `9`.

### Horde 2

The set has one trigger, two listener/spawn points, and three gate teleporters:

- trigger row `128253`, authored ID `2142467792`, at
  `(-533.6099, 561.4774, 0.1342)`: `HordeTrigger_OnEnterPlayer`, event
  `horde triggered` on enter and `horde complete` on exit; no positive retail
  activation radius is recovered;
- listener rows `128251` and `128252`:
  `horde triggered -> HordeSpawner_Register`;
- gate row `128254`: an authored `horde triggered` edge to
  `HordeGateTeleporter_OnEnter`; rows `128256` and `128257` have contact edges
  to the same callback;
- blocking-door ordinals `0` through `2`.

The differing listener counts establish candidate spawn locations, not wave
counts. `SpawnPointDef` has reflected `challengeOverride` and `waveOverride`
fields, but their effective values for these 1-1 instances have not been
recovered. The Cryos value `waveOverride=4` must not be copied into 1-1.

## Linked Lua and native callback boundary

The server-side tutorial job attached to the boss marker is indexed Lua chunk
`62`, resource `13584`, source `0x24F78AA1/0x90CE5ECC.lua`, SHA-256
`b67c0aace5d108e5f6f7b2432f5d8be1aaee3f00d1204fdd56c6b03abcd878c4`.
`nTutorial_SoloSupportUnlock.main` does this exactly:

1. waits 2 and 4 seconds;
2. if all relevant players were initially unbeaten, unlocks the next ability
   for each still-unbeaten player;
3. waits 2, 5, and 2 seconds around that work;
4. resolves the controlled object for the callback player;
5. calls `nGameDirector.ActivateHordeSpawn(sourceObject, controlledObject)` and
   discards the return.

The corresponding presentation job is chunk `175`, resource `13709`, SHA-256
`afb48133fc171b8f9eca258893bceaa63e1721b609eddf81349decdde8249966`.
`nTutorial_SoloSupportUnlockClient.main` waits `2`, `4`, and `0.5` seconds,
blinks the three support-ability controls, waits `6.5` seconds, and returns.

These jobs are tutorial progression and presentation, not encounter authority.
In build 103, `ActivateHordeSpawn` reaches a literal no-op. It ignores the
second argument for gameplay purposes and supplies no spawn, wave, or clear
rule. `HordeSpawner_Register` is absent from all 1,029 indexed Lua chunks and
from the client executable.

The registered client natives are also negative evidence:

- `HordeTrigger_OnEnterPlayer -> sub_9FACB0`;
- `DirectorTrigger_SpawnBoss -> sub_9FACF0`.

They validate the entrant/client-authority branch, obtain the director, and
return false without mutating authoritative state or emitting a game message.
The server therefore owns the accepted trigger, event fan-out, spawn
registration, encounter lifecycle, and retries.

The raw horde trigger records need special care. `TriggerVolumeEvents` places
`horde triggered` in `onEnterEvent` and `horde complete` in `onExitEvent`. On
the client, the false enter callback suppresses enter-event publication and
leaves the generic enter-success latch clear, so a later entry can invoke it
again. There is no native exit callback; the generic no-callback path can
publish the authored exit event. This proves client trigger mechanics, not a
retail clear rule. Treating a client crossing out of the volume as authoritative
`horde complete` would let movement end the encounter and is unsupported.

## Gate and teleporter behavior

### Horde gates

The decoded `HordeGateTeleporter` binary asset, instance `0x1968EB10`,
resource `4894`, type `0x76A8F7D8`, decoded size `960`, SHA-256
`8f613f29e7104d0d4f5ae4fce41071635fc4edaa506d68ec7762afda66829e71`,
maps:

- `horde triggered -> ActivateHordeGateTeleporter`;
- `horde complete -> DeactivateHordeGateTeleporter`.

The packaged Lua does not contain those callbacks or
`HordeGateTeleporter_OnEnter`; they are server-owned commands.

Modifier chunk `199`, resource `13736`, SHA-256
`7674d1af31c91b3f4a5ddbad2c5a87c491a26199fb829846e268f87f18b2b4ec`,
registers `HordeGateTeleporter`. On activation it reads destination float
properties, stops the agent, applies `Immobilized` and `Intangible`, limits the
displacement to seven units, snaps to the closest navigation point, plays the
red beam effect at the old position, teleports, yields, and plays the red beam
effect at the new position. Deactivation has no Lua body; native modifier
teardown removes its scoped attributes.

This supports the interpretation that active horde gates repel or relocate an
entrant and that `horde complete` removes the barrier. It does not identify the
server-side gate instances, destination-property ordering, activation
idempotency, or how the adjacent blocking-door objects are synchronized.

### Boss-security teleporter

The base design has `BossSecurityTeleporter` row `125464`, authored ID
`3977962581`, at `(184.2345, 558.9578, 10.2027)`. Row `125479`, authored ID
`1131366622`, is a `TeleporterSpawnPoint` at
`(928.1883, 668.9045, 0.0880)`, about 13 units from the exit marker and 21 units
from the final cluster center. It is the unique strong destination candidate,
but the normalized data has `target_marker_id=0`; the exact source-to-target
edge is not recovered.

The content link
`BossSecurityTeleporter -> BossSecurityTeleporter.AIDefinition ->
BossSecurityTeleporterPassive` is exact. Passive chunk `144`, resource `13674`,
SHA-256
`c2e727c6cf4c13708fa315321fb51f0402a3a4a9763eaf2f8d782c1eca3267f0`,
creates a radius-2 entry trigger and polls a radius-20 security region. A
qualifying player-controlled entrant receives the teleporter modifier only
while the teleporter is active. Duplicate modifier application is suppressed.
The security test considers live organic objects, team relationship, and the
`InvisibleToSecurityTeleporters` exclusion; when no qualifying threat remains,
the teleporter becomes active and runs its active effect.

The requested modifier is GUID `0x502F1932` (`TeleporterModifier`). Chunk `349`
plays teleport-out, waits `0.5` seconds, performs the authoritative teleport,
waits `0.5`, plays teleport-in, and waits another `0.5` seconds.

This teleporter is distinct from `HordeGateTeleporter`. The strongest route is
that clearing guards near the boss-security platform opens travel to row
`125479`, which lands the player beside the final arena. Missing authority still
has to resolve the exact destination, define the authoritative threat query,
and keep team/death/despawn state consistent with the passive's client view.

## Director state and the clear boundary

Application message `0x8B` reflects `cAIDirector`. Its seven fields are exact:

| Mask bit | Field | Client offset | Meaning established by client use |
| --- | --- | --- | --- |
| `0` | `mbBossSpawned` | `+0x0D` | a boss has been spawned |
| `1` | `mbBossHorde` | `+0x0E` | encounter is labelled a boss horde |
| `2` | `mbCaptainSpawned` | `+0x0F` | a captain has been spawned |
| `3` | `mbBossComplete` | `+0x10` | final boss completion gate |
| `4` | `mbHordeSpawned` | `+0x48C` | a horde is active/spawned |
| `5` | `mBossId` | `+0x14` | boss object ID |
| `6` | `mActiveHordeWaves` | `+0x47C` | four-byte reflected handle; element semantics unknown |

An empty update is `8B 00`; the minimal completion update is `8B 08 01`.
`nGameDirector.IsBossDead()` reads `mbBossComplete`, while
`IsHordeActive()` reads `mbHordeSpawned`. The client has no discovered writer
for these fields and no direct constructor for logical message `55` / wire
`0x8B`, so it is a receiver of server state.

The fields suggest, but do not prescribe, a lifecycle such as setting
`mbHordeSpawned`, possibly setting `mbBossHorde`, assigning `mBossId`, tracking
active waves, clearing horde-active state, and finally setting
`mbBossComplete`. The exact order, whether every field is used in 1-1, and
whether the boss ID names a dedicated actor are all missing authority.

The authoritative clear predicate cannot be reconstructed. In particular,
neither the client nor Lua says whether clear means every registered spawn is
dead, every wave is exhausted, a particular boss object is dead, all hostile
organic objects in a region are gone, or some conjunction of those facts.

## Strongest supported runtime ordering

The following is the narrowest ordering that accounts for all available
evidence. Steps marked inferred require server confirmation even when their
relative position is strongly constrained.

1. **Inferred:** normal 1-1 traversal reaches horde set `1827` before `1828`.
2. **Exact authored boundary:** a qualifying player enters horde 1's trigger
   volume; its effective positive activation radius is missing.
3. **Missing authority:** the server accepts the trigger, publishes
   `horde triggered`, registers the three listener markers, closes/activates
   the gates, and creates the chosen waves.
4. **Missing authority:** after its clear predicate is true, the server stops
   wave state, publishes `horde complete`, and deactivates/removes the three
   gate barriers.
5. **Exact authored boundary / inferred traversal:** a qualifying player later
   enters horde 2's trigger volume; the same lifecycle runs with two listener
   markers and three gates. Its effective positive radius is also missing.
6. **Content-correlated:** traversal reaches the boss-security platform. Its
   passive remains inactive while its radius-20 security predicate finds a
   qualifying threat.
7. **Exact client behavior / missing authority:** once that predicate is clear,
   a qualifying entrant receives `TeleporterModifier` and is moved to the
   resolved destination. Row `125479` beside the final arena is the strongest
   destination candidate.
8. **Exact presentation boundary:** entry into the radius-60 support trigger
   schedules the client support-unlock presentation.
9. **Content-correlated:** the boss marker participates in publication of
   `boss triggered`; its server tutorial job starts, and the four surrounding
   horde listeners become final-encounter spawn candidates. The publisher,
   trigger radius, and ordering between steps 8 and 9 are not recovered.
10. **Exact negative evidence:** the tutorial job eventually calls the no-op
    `ActivateHordeSpawn`; this call does not start or advance the fight.
11. **Missing authority:** the server chooses final composition, spawns actors
    and waves, and reflects whichever of `mbBossSpawned`, `mbBossHorde`,
    `mbHordeSpawned`, `mBossId`, and `mActiveHordeWaves` retail used.
12. **Missing authority:** deaths/despawns/wave exhaustion satisfy the final
    clear predicate. The server resolves success versus failure races and
    commits `mbBossComplete=true` only for accepted success.
13. **Exact client gate:** `mbBossComplete` makes `IsBossDead()` true;
    `SetReadyForBeamOut` exposes `HUD_BeamOut` with `RETURN TO SHIP`.
    `LevelExitPoint` is a colocated authored anchor, not a proven contact
    callback and not the source of success.
14. **Exact request:** the player clicks Beam Out. The client sends
    `PlayerStatusUpdate(status=0x20, progress=1.0)` once behind a local latch.
15. **Missing authority:** the server authenticates and deduplicates that
    request, finalizes objectives and level rewards, emits result/transition
    messages, and enters the chain continuation flow.
16. **Exact client request on chain-voting entry:** the client sends C2S
    `ChainPlayerMsgs` subtype `0`, application bytes `AC 00`.
17. **Exact choice requests:** Continue Fight sends C2S subtype `1`; Cash Out
    sends C2S subtype `2`. Vote responses and the chosen transition are
    server-owned.
18. **Exact cashout request:** entering `cChainCashOutState` sends C2S subtype
    `4`, `AC 04`; S2C `ChainCashOut` subtype `0` supplies a 712-byte body for
    the cashout UI.
19. **Missing authority:** the server durably grants rewards and any eligible
    creature unlock, updates campaign/chain state, and returns the player to the
    appropriate ship or next level.

Steps 13 and 14 are strongly ordered: the button is gated by the reflected
completion state. The ordering of `ObjectivesComplete (0xB9)`,
`ChainLevelResults (0xAA)`, player departure, and the state transition around
steps 14-16 is not recovered and must not be invented from the incomplete
reference server.

## Mission success and failure

### Success

`ObjectivesComplete` (`0xB9`) is a full result snapshot, not an encounter-clear
signal. It carries an objective count, `count * 56` bytes of objective records,
and four `u32` player-result fields. The optional `TouchAllObelisks` objective
is a Standard-medal objective and is not evidence for mission completion.
Which 1-1 objectives are selected, when they are evaluated, and how their
medals feed results are not recovered.

The earliest client-visible success fact is `mbBossComplete`. A last enemy
death is therefore insufficient on its own, and sending a tutorial message in
place of director completion is incorrect. A safe implementation boundary
would require one idempotent server transaction to accept final clear, freeze
encounter state, reflect completion, and remember that only authorized players
may later Beam Out; the exact retail transaction remains missing.

### Failure

The exact active-gameplay failure transition is S2C application `AD 01`:
`ChainGameMsgs` subtype `1` selects client state `13` (`GameOver`). That state
opens `HUD_Death` with `MISSION FAILED` / `COMMAND RELAY TERMINATED`.
`AF 00` can subsequently server-drive the spaceship state. `AE 00` and `AE 01`
are consumed while in GameOver but have no recovered state mutation. No
specific client game-over acknowledgement has been found.

C2S `AC 01 ...` remains the distinct Continue Fight request described below.

The failure predicate is entirely server-owned. Available evidence does not
decide whether failure occurs when the controlled creature dies, the deployed
squad is exhausted, all multiplayer participants are unable to revive, a timer
expires, or another rule fires. Nor does it define revive grace, disconnect and
reconnect handling, success/failure race priority, checkpoint restart, or what
pending loot survives. `ReloadLevel (0xBF)` exists but is not proven to be the
normal 1-1 failure route.

## Chain continuation, rewards, and cashout requests

All byte sequences below are application payloads. RakNet framing is outside
this table.

| Direction and moment | Payload | Exact client behavior | Unknown server decision |
| --- | --- | --- | --- |
| C2S, click Beam Out | `PlayerStatusUpdate`, status `0x20`, progress `1.0` | sent once by `MaxisBeamOut.OnBeamOutClicked` | acceptance, quorum, commit, and next state |
| C2S, enter Chain Voting | `AC 00` | one-byte ChainPlayer subtype `0` request | response timing and available chain choices |
| S2C, voting data | `A9 00` plus subtype-0 body | current recovered layout consumes a 337-byte body | exact retail field semantics and 1-1 next-level entry |
| S2C, voting countdown | `A9 01` plus float | supplies countdown | duration, timeout choice, and multiplayer resolution |
| S2C, party/cashout decision | `A9 02` plus boolean | selects stay-in-party/cashout handling | authoritative vote rule |
| C2S, Continue Fight | `AC 01 01 <u32-le>` | Planet Screen calls the subtype-1 sender with flag `1` and a selected-record ID | exact meaning of the `u32` (the incomplete reference calls it squad ID), validation, and next level |
| C2S, Cash Out | `AC 02` | `MaxisPlanetScreen.OnCashOut` sends subtype `2` | commit timing and transition |
| C2S, enter Cashout state | `AC 04` | requests cashout presentation data | correlation, replay, and retry behavior |
| S2C, cashout data | `AB 00` plus 712-byte body | `cChainCashOutState` consumes exactly `0x2C8` body bytes | authoritative layout, rolls, and grant status |
| HTTP, choose offered creature | `api.creature.unlockCreature` | submits the selected template noun | eligibility, token consumption, idempotency, and persistence |

The cashout UI reads fields for planets completed; bronze, silver, and gold
medals; rarity/rare/unique/epic chances; DNA and XP; per-player rolls and epic
loot; valid players; starting and final levels and XP fill; cashout bonus; and
creature-unlock choices. That proves presentation requirements, not when or how
the server grants them. The incomplete reference structure agrees on only some
offsets and even uses an incorrect opcode in one path, so it cannot close the
layout or transaction semantics.

`api.creature.unlockCreature` is a separate account-authorized request after a
choice is presented. It must not be treated as evidence that receiving the
cashout packet already persisted the unlock. Likewise, world loot, medals, XP,
DNA, chain bonus, and campaign progression may be previewed before commit; the
retail commit boundary is missing.

## Missing server-authority choices

Every unresolved policy needed for a faithful implementation is listed here.
This investigation deliberately selects no fallback.

### Encounter activation and spawning

- exact traversable order of horde 1, horde 2, boss security, and final arena;
- eligibility for an enter trigger: owner, controlled creature, team, alive
  state, multiplayer participant, and spectator handling;
- whether one entrant starts an encounter for all players or a quorum is
  required;
- activation radius/shape, once-only consumption, retry after partial failure, duplicate
  contacts, late join, reconnect, and reset behavior;
- event fan-out order between gates, listener registration, tutorial jobs, and
  spawn creation;
- effective `challengeOverride` and `waveOverride` for every 1-1 marker;
- horde budget, difficulty scaling, eligible noun selection, random seed,
  count, captain/affix assignment, listener selection, spawn delay, aggro
  target, and maximum concurrent actors;
- whether listeners are simultaneous positions, alternatives, sequential
  waves, or reusable points;
- whether the final fight has a dedicated boss actor, which noun it uses, or is
  solely a boss-labelled horde;
- the final boss-trigger publisher, activation radius/shape, and relationship
  to the radius-60 tutorial trigger;
- whether tutorial support unlock completion is allowed to affect encounter
  timing; the no-op native provides no such authorization.

### Active, wave, and clear state

- which 1-1 events set and clear `mbHordeSpawned`, `mbBossSpawned`,
  `mbBossHorde`, and `mbCaptainSpawned`;
- when `mBossId` is assigned/cleared and what it identifies if there is no
  dedicated boss noun;
- the representation and update rules for `mActiveHordeWaves`;
- authoritative membership of an encounter after summons, conversions,
  despawns, leashes, out-of-bounds cleanup, disconnects, or failed spawns;
- whether clear requires all registered actors dead, all scheduled waves
  exhausted, one boss dead, a region empty, or a compound predicate;
- handling of simultaneously cleared waves, delayed death, corpse lifetime,
  and enemies that escape the arena;
- publication timing and idempotency of `horde complete`;
- whether and how the server suppresses or ignores the raw client-side
  `onExitEvent=horde complete` publication until the authoritative clear rule
  succeeds;
- ordering between clearing active fields, opening gates, removing blockers,
  and beginning the next encounter;
- state replay for late join/reconnect and recovery after a server restart.

### Gates and teleporters

- authoritative object targeted by each horde-gate callback and the exact
  destination floats supplied to modifier chunk `199`;
- synchronization of horde gate modifiers with adjacent blocking-door objects;
- whether entrants are repelled, held, or rerouted on every contact and how
  repeated contacts are throttled;
- exact source-to-destination edge for the boss-security teleporter; row
  `125479` is a strong candidate, not a recovered identity link;
- server-side security membership, team relationship, alive/dead threshold,
  organic classification, invisibility exclusion, scan cadence, and cleanup;
- whether dynamically spawned enemies, summons, or player-owned creatures can
  keep boss security closed;
- multiplayer simultaneous use, duplicate modifiers, failure in mid-teleport,
  reconnect position, and authoritative correction.

### Success, objectives, exit, and failure

- exact final clear predicate and the operation that sets `mbBossComplete`;
- ordering and atomicity of final clear, director updates, gate cleanup,
  objective evaluation, loot drops, and Beam Out eligibility;
- selected 1-1 objectives, per-objective predicates, medal thresholds, player
  result fields, and the complete `0xB9` emission order;
- any server meaning for `LevelExitPoint`, including whether it is only an
  anchor or participates in placement/validation;
- Beam Out authorization, per-player versus party acceptance, multiplayer
  quorum, host authority, timeout/automatic exit, and idempotent replay;
- validation of status `0x20` against director and player state;
- exact response sequence after Beam Out, including `0xB9`, `0xAA`, departure,
  chain-state selection, and any account save;
- mission-failure predicate, revive/grace policy, disconnect policy,
  simultaneous success/failure priority, and failure transition sender;
- restart versus ship-return choice, checkpoint restoration, and what pending
  encounter rewards survive failure.

### Chain and reward transaction

- exact `ChainLevelResults (0xAA)` payload and whether it precedes chain voting;
- chain capacity, current run position, permitted next planet/level after 1-1,
  squad restrictions, and whether Continue Fight is always offered;
- full `ChainVote (0xA9)` subtype-0 layout, voting timer, host/majority/tie rules,
  disconnect handling, party retention, and timeout default;
- semantic identity of the Continue Fight `u32`, validation of its flag, and
  how it selects the next squad/record;
- whether Cash Out subtype `2` commits immediately or only chooses a state;
- full 712-byte cashout layout, integer widths, probability/roll semantics,
  per-player sections, and unused/reserved fields;
- authoritative loot roll, rarity promotion, medal, DNA, XP, cashout bonus,
  inventory-capacity, duplicate, and per-player allocation rules;
- whether rewards are granted at final clear, Beam Out acceptance, result
  delivery, Cash Out choice, cashout-data request, or a later account request;
- transactionality across campaign progress, chain progress, inventory,
  currency, XP/level, medals, and creature-unlock eligibility;
- deduplication keys and replay behavior for Beam Out, `AC 00`, Continue Fight,
  `AC 02`, `AC 04`, cashout response, and creature unlock;
- creature-unlock offer generation, number of choices, eligibility, token or
  entitlement consumption, rejection behavior, and fallback if no choice is
  made;
- final transition to the ship or next gameplay session and the account/profile
  refresh messages required to make committed rewards visible.

## Evidence cross-references

- `notes/campaign/1-1/overview.md`: linked Lua inventory and final marker callbacks.
- `notes/campaign/1-1/route.md`: horde sets, gates, route markers, and teleporter
  coordinates.
- `notes/campaign/1-1/director.md`: director marker pools and final-arena correlation.
- `notes/campaign/1-1/budget.md`: raw horde-trigger fields, listener fan-out, native
  enter suppression, and generic exit-event behavior.
- `notes/campaign/1-1/native.md`: exact `cAIDirector` reflection layout.
- `notes/campaign/1-1/objectives.md`: objective-selection and `ObjectivesComplete`
  evidence.
- `notes/tutorial/overview.md`: decoded modifier Lua, native callback boundaries,
  director/Beam Out analysis, and broader build-103 disassembly.
- `notes/tutorial/route.md`: shared boss-security and Beam Out client behavior,
  clearly separated from 1-1 composition.
- `notes/tutorial/restart.md`: exact GameOver transition and restart/ship
  behavior.
- `notes/campaign/progression.md`: account-side creature
  unlock request.
- `notes/design/architecture/raknet-gameplay-exchange.md`: application message framing
  and result-packet layouts.
