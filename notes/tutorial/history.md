# Playable Build-103 Tutorial - Historical Implementation Checklist

This checklist preserves the live-play findings, implementation history, and
remaining behavior defects from the first hard-coded tutorial reconstruction.
 Items here remain useful evidence and regression coverage, but they
must not dictate architecture when retail bytecode, IDA analysis, assets, or
wire traces establish a better authored sequence.

The current disposition of the route-level disagreements is recorded in
`notes/tutorial/overview.md` under **Route evidence reconciliation**. Unchecked boxes
below preserve the state of the historical replay and are not an independent
active plan; follow the focused root items for work that remains open.

## Scope rules

- [x] Target the bundled `5.3.0.103` binary and its RakNet 3.92 behavior.
- [x] Keep useful build-127 and modern RakNet research notes, clearly labeled as
  non-authoritative for build 103.
- [x] Do not implement modern RakNet (`0x05`/`0x06`, then `0x07`/`0x08`) in Go.
  The runtime implementation must follow build 103's single `0x09`/`0x0a`
  offline opening and its connected transport.
- [x] Put game/IDA diagnostic output in `bin/game/logs`, set automated IDA
  `IDAUSR` to `bin/game/ida_user`, and put protocol captures in
  `bin/server/darkspin/logs/traces`.
- [x] Record newly confirmed packet layouts and tutorial behavior in
  `notes/tutorial/overview.md` or `notes/design/architecture/raknet-gameplay-exchange.md` with
  an evidence label.

## Definition of done

- [x] `mage darkspin:auth`, `mage darkspin:server`, and `mage tutorial:run` start a clean local tutorial.
- [x] The client completes build-103 RakNet opening and connected negotiation.
- [x] The tutorial level loads with Blitz created and deployed and only one
  Blitz portrait appears in the squad. Build 103 still requires valid backing
  records/reflections for all three fixed slots, but top-level field `22` now
  carries locked deck minimum `1`, so only Blitz is exposed to the HUD.
- [ ] The player can move, stop, aim, use abilities, switch heroes, interact,
  pick up drops, and receive authoritative state updates.
- [ ] All authored tutorial encounters spawn in the correct sequence.
- [ ] Enemies can target, move, attack, take damage, die, and drop the expected
  tutorial rewards.
- [ ] Objectives, gates, obelisks, teleporters, horde waves, Sage unlock, and
  the final arena advance in the observed order.
- [ ] Mission rewards and cumulative XP are applied once, tutorial completion
  advances onboarding to `3000`, and `RETURN TO SHIP` succeeds.
- [ ] A fresh-account regression run completes the tutorial without manual
  database repair or packet injection.

### Full-route replay defects (2026-07-17)

- [ ] Restore the authored player beam-in presentation when Blitz first enters
  the tutorial. The fresh full-route replay created the player without a visible
  teleport effect. The event and animation are now generation-fenced and
  published `100ms` after object/deploy state instead of in the same creation
  batch, with a same-response fallback for deterministic adapters. The delay is
  a presentation compatibility candidate pending live verification.
- [x] Show only one Blitz portrait in the initial squad. A live authored-start
  run confirms locked deck minimum `1` hides the two crash-guard records while
  retaining the valid backing data build 103 requires. The client caches fixed
  squad identities from this first snapshot, so hidden slot two is now preloaded
  with Sage rather than a second Blitz. A focused boundary-`2` reveal showed
  exactly Blitz and Sage, and W selected Sage's world object and ability bar.
  The temporary timed reveal used for that proof has been removed; the authored
  route trigger remains the only normal unlock path.
- [ ] Reconcile the authored enemy population before the Ride lesson. The fresh
  replay appeared empty between the route start and Ride unlock. Static marker
  correlation found that darkspin's four groups were sequenced backwards. They
  now begin with the four authored markers at `(197..233,-162..-116)` and remain
  perimeter-triggered instead of being populated globally. Live-verify the
  opening spawn and lesson timing before checking this complete.
- [ ] Keep the Ride-use HUD arrow visible from unlock until the first accepted
  Ride cast, then clear it exactly once. In the full replay it disappeared
  before use. Also recover the correct post-cast animation transition: Blitz
  visibly glitches through frames after Ride currently.
- [ ] Live-verify Voltic Slash's authored 0.4-second cooldown and alternating
  left/right-arm sequence. Targetless `unknown=1` Shift+left-click now repeats
  through the proven type-7 press/type-10 release boundary; an ordinary
  targetless right-click/tap schedules only one miss. Targeted right-click remains
  client-paced and each repeated type-7 request crosses the authoritative
  cooldown/release gate; it must not schedule a recursive server repeat.
  Type-10 release suppresses the next targetless repeat while allowing the
  already-accepted swing to finish.
- [ ] Give pursuing enemies authored facing and attack presentation. They move
  toward and damage Blitz, but remain visually oriented downward and play no
  walk or attack animation. Horde creation now publishes a stopped `0x42`
  locomotion snapshot facing the deployed hero before the spawn modifier runs,
  preventing the default north-facing/backward first frame. Pursuit now uses
  active flags `0x43`; unlike the former `0x41`, this includes bit `0x02`, which
  the recovered locomotion code requires before consuming the packet's explicit
  facing vector. Attack animation and live pursuit-facing verification remain
  open.
- [ ] Recover floating hit text for both player-to-enemy and enemy-to-player
  damage.
- [ ] Award cumulative tutorial XP for each enemy kill and send the resulting
  progression updates. The old one-time XP-`101` stage scaffold is removed.
  All 16 authored pre-obelisk enemies now persist and emit the video-derived
  awards `9,14,12,14,12,12,14,13,9,8,8,10,8,8,8,8`, reaching cumulative XP
  `167`; the strict boundary keeps level 1 at XP `100` and enters level 2 at
  `101`. `TutorialSpecialOne_Intro` is now a distinct Quadra gate at the Sage
  marker with the measured `+54` award, reaching XP `221` and level 3 after
  Sage is revealed. A focused live pass confirmed the `+54` persistent award;
  full-route level-3 presentation still needs verification. Keep this item open
  until the two observed horde awards are mapped onto corrected horde enemies.
- [ ] Keep placed health/mana bars as walk-over pickups. Clicking one must not
  consume it. Recover the missing blue/green pickup effect after movement
  intersects its volume; authoritative resource restoration already works.
- [ ] Fix combat-stage and client-prompt ordering around the first loot obelisk.
  In this replay objects `7`-`10` were created behind the advancing player and
  remained alive, while the client independently presented the obelisk prompt;
  object `15` was therefore absent under the current server predicate and the
  route became unplayable. Trigger or gate encounters at their authored
  perimeters so required clears cannot be silently skipped. The deterministic
  server order is now corrected to follow the traced route and includes the
  previously omitted three-enemy group at `(138..156,20..25)` before object
  `15`; live-confirm the obelisk cannot be prompted or passed before all four
  groups clear.
- [ ] Recover the authored new-enemy introduction: when a new enemy type first
  appears, the tutorial cuts to/focuses that enemy before returning control.
  Do not treat every deterministic follow-up spawn as an immediate generic wave.
- [ ] **Priority: reconstruct the complete Sage/Quadra boss-introduction
  cutscene.** At the authored boundary, reveal Sage and present the swap lesson,
  take camera/control focus, create and center on Quadra, play the introduction,
  then return control for the fight. Determine the exact director, objective,
  camera, and object packet sequence rather than approximating it with a spawn.
- [ ] **Priority: add a two-hero tutorial-start verification checkpoint.** Start
  the dungeon with Blitz and Sage both available, switch W/Q in both directions,
  and verify that deploy, visibility, camera tracking, movement authority,
  resources, target selection, and abilities all transfer to the newly active
  player-controlled world object. Remove or keep the checkpoint explicitly
  debug-only after the behavior is confirmed.
- [ ] **Priority: reconstruct game over and clean tutorial restart.** Drive the
  hero squad to defeat, recover the authored failure screen/audio and server
  status sequence, then accept restart and prove a fresh tutorial begins at the
  normal authored spawn with reset heroes, encounters, objectives, cooldowns,
  interactables, XP transaction state, and no objects from the failed session.
- [ ] Evaluate a bounded Lua 5.1 tutorial-runtime pilot using the recovered
  `FirstAggro_SpecialOne` bytecode/source. Bind only typed, allowlisted engine
  operations (`wait`, `unlockHero`, `startCinematic`, `setVisible`, `animate`,
  `spawn`) to the existing per-session scheduler and RakNet encoders. Keep
  authentication, validation, combat authority, and all persistent mutations in
  Go feature operations. Do not attempt a general Game Lua engine until the
  pilot demonstrates less complexity and better fidelity than typed Go event
  timelines. Keep implementation and reverse engineering inside this
  repository; use `bin/darkspinner/GameBin/Game.c` when executable
  evidence is required. Do not mark the pilot complete until darkspin owns the build-103
  `CinematicMsgs` (`0xC9`) encoder and live client verification.
- [ ] Inventory every native Lua API used by the authored tutorial scripts and
  classify it as client presentation, server simulation, scheduling, gameplay
  mutation, or persistence. Record exact arguments, delays, and dependent
  assets in `notes/tutorial/overview.md`; use this inventory to decide which APIs belong
  in a future sandboxed Lua adapter and which remain explicit Go operations.
- [ ] Verify the later extra group observed after the first post-Ride fights
  against the authored encounter list and cutscene triggers; remove it only if
  route evidence shows it is genuinely spurious.
- [ ] Fix the boss teleporter's duplicate presentation. Object `28` is created
  once, but the noun plus inactive/active `ServerEvent` path currently appears
  as two teleporters; determine which presentation owns the visible model.
- [ ] Make teleporter activation follow the recovered route without deadlocking.
  This replay could not use the absent obelisk, so Sage/teleporter prerequisites
  never completed and the duplicated teleporter remained inactive.

## Milestone 1: Reach controllable gameplay

- [x] Run `mage darkspin:auth` in its own terminal and verify the loopback broker is
  ready.
- [x] Run `mage darkspin:server` in its own terminal and verify HTTP, Blaze, QoS, and
  gameplay UDP readiness.
- [ ] Reset or create local profile 1 with `onboarding_progress = 0` using
  `darkspin db`; never update without an explicit `where` clause.
- [x] Run `mage tutorial:run`, log in as profile 1, and enter
  `Game_Tutorial_cryos_1_v2` in game mode `1`.
- [x] Capture the complete client/server opening exchange and first gameplay
  packets under the prescribed log/trace directories.
- [x] Remove the modern RakNet opening handlers and tests from the Go runtime;
  retain only build-103 transport behavior.
- [x] Complete the build-103 connected handshake: reliable `0x04` connection
  request, `0x0e` acceptance, `0x11` new-incoming notification, ACK/NACK,
  ordering, retransmission, and disconnect behavior needed by the client.
- [x] Pin each remote UDP endpoint to QoS or build-103 RakNet after its first
  validated protocol exchange. Affinity expires after five idle minutes so
  later packets from a shared source port cannot collide with the other route.
- [x] Complete `HelloPlayerRequest`/`HelloPlayer` and initial game-state packet
  sequencing.
- [x] Decode the first client action packet and echo/replicate movement so the
  player can move successfully. Live build-103 verification confirmed client
  `0x9c` for object `1`; the camera and minimap advance to the clicked Cryos
  position.
- [x] Confirm stop movement and orientation updates in a live run. Build 103
  reaches the requested ground point, settles into its stopped pose, and faces
  the destination after locomotion replication.
- [x] Replace debug movement replication with normal locomotion. Routine
  movement no longer sends `ObjectTeleport` (`0x90`). It now sends active-goal
  `ObjectPlayerMove` (`0x91`, flags `0x01`) followed by the fixed build-103
  `LocomotionDataUnreliableUpdate` (`0x95`). A live left-click test visibly
  walked Blitz and tracked the camera without the old teleport snap.

## Milestone 2: Reconstruct the tutorial route

- [x] Restore `tutorialPlayerSpawnPosition` to the preserved authored route
  start `(177.75531,-239.85399,0.08799999)`. Focused test snapshots remain
  explicit and independently testable: opening combat at
  `(102.38726,-91.42871,15.087998)` just outside the enemy alert perimeter, and
  the ability lesson at `(237.008,-103.335,10.150)`, the nearest
  live-confirmed point outside its client-authored one-shot trigger. The earlier
  `(234.049,-99.257,10.150)`, 6.5-unit, 15-unit, and 22.5-unit west-offset
  snapshots all presented the client lesson immediately. A far-south snapshot
  at `(238.987,-138.448,10.150)` is preserved by this history and live-confirmed
  outside. Live movement then isolated the client crossing from
  `(237.008,-103.335)` to `(230.343,-98.431)`. darkspin now tests the full movement
  segment against the oriented box, padded by Blitz's authored 0.8-unit scaled
  bounding radius. A clean rebuilt run confirms the current safe snapshot stays
  dormant with no early blink call; one northward click triggers both the
  client `Press 1` lesson and darkspin's milestone.

- [ ] Work through `notes/tutorial/overview.md` from a clean trace and assign stable
  names to every observed packet ID, subtype, direction, reliability mode, and
  ordering channel.
- [ ] Map level marker sets, fixed enemy placements, smart objects, triggers,
  directors, and gates to stable runtime object IDs.
- [ ] Implement authoritative object creation/destruction and late-join state.
  Hero and first-enemy create/update/combatant snapshots now work live; general
  population, destruction sequencing, and reconnect snapshots remain.
- [x] Implement the minimum opening movement/basic-attack encounter. The two authored
  `TutorialBasicDiseased` objects now render at their exact AI marker positions,
  and live probing confirms basic attack is client `0x9c` type `7` with an
  84-byte body. Live verification confirms 20 -> 10 -> 0 HP, visible health-bar
  updates, and enemy deletion. Their definitions, live health, defeat removal,
  and one-time encounter-complete predicate now live in deterministic encounter
  state rather than a fixed HP map. Proper attack/death animation, AI
  retaliation, loot, and objective progression remain.
- [x] Spawn and fight the next authored simulated encounter after the opening
  infectors. A live build-103 run confirms three `TutorialBasicPoisonNoOrbs`
  Chlorosaurs (object IDs `4`-`6`) appear once at their AI marker positions,
  accept authoritative damage, update health, and delete on defeat.
- [x] Continue into the next four authored `TutorialBasicPoisonNoOrbs`
  Chlorosaurs at `x=170-184` (object IDs `7`-`10`). A clean live run confirms
  the one-time transition, rendering, authoritative health, and deletion.
- [x] Spawn and fight the four authored no-drop Chlorosaurs leading into the
  ability lesson (object IDs `11`-`14`, `x=197-233`). Live verification confirms
  the group is spatially separated, renders after advancing along the route,
  accepts authoritative damage, and clears cleanly.
- [x] Implement ability-one unlock/use.
- [x] Live-verify the level-2 progression beat after the opening encounter.
- [x] Recover and live-exercise the Blitz passive lesson on the authored route.
  It is not an immediate server objective after the level-2 banner:
  build-103 assets define client-owned, one-shot audio
  `vo_ship_tut_abilities_passive` at `(118.88472,-85.76919,14.97249)` with an
  `AudioTrigger_OnEnter` volume. The trigger is a thin 50x2x20 box rotated
  82.28936 degrees around Z. A clean authored-start replay crossed it on the
  movement segment from `(124.21403,-95.78429)` to `(99.72728,-93.68147)` and
  visibly presented the client's blue voiceover callout before the opening
  enemies engaged. Waiting at the defeated wave therefore cannot exercise it.
  A focused run that granted ability boundary `3` before combat also produced
  no immediate prompt, ruling out ability ordering as its trigger.
  Build-103 reflection metadata and code now agree that top-level
  `LabsPlayerUpdate` field `21` writes simulation-player offset `+0x1370`.
  Earlier live tests used count `1 -> 2`; they retained only the basic icon.
  Native HUD tracing explains that result: index `1` is Blitz's passive, while
  the first pressable action-bar ability is index `2`. Build-103 script and
  marker analysis recovered two authored `UnlockNextAbility` calls: the player
  starts at boundary `1`, then the server-only marker schedules `2` followed by
  `3`. A clean build-103 run confirms the resulting transition adds Blitz's
  second red HUD icon, presents `Press 1`, and key `1` emits character-ability
  index `2` while visibly playing Blitz's activation animation. The no-enemy focused snapshot
  reports target object `0`, so a combat-targeted ability result is still next.
  The same trace corrected fixed character offsets for creature type, ability
  points, and ranks from guessed `0x3b8/0x3c8/0x3cc` to registered
  `0x388/0x398/0x39c`. A focused field-3 character reflection plus field-21
  count-2 refresh was accepted without disconnecting but still left only the
  basic icon, so it remains removed as unsupported packet behavior.
  Index `2` is now live-confirmed as Blitz's cursor-directed teleport request.
  Target object `0` is its normal ground-target form; `CursorPosition` and
  `TargetPosition` were identical at `(229.66934,-87.43901,10.627861)`, about
  six units from the previous movement goal. darkspin now rejects non-finite
  destinations, advances authoritative `playerPosition`, and replies with the
  action acknowledgement plus `ObjectTeleport`. A clean rebuilt run confirms
  the client shows the red teleport column and lands Blitz normally at the
  echoed destination. The live build-103 tooltip is authoritative here: Voltic
  Slash deals 4-12 physical damage; Ride the Lightning has 50 m range, deals
  14-22 physical damage, shocks its target for three seconds, and resets its
  cooldown on kill. The observed power cost is 13 and the live tooltip now
  correctly reports a 10-second cooldown. Build-103 static analysis recovered
  the 36-byte `CooldownUpdate` shape. Its first two words are a 64-bit runtime
  ability key rather than an action-bar index and flags; the exact Special One
  HUD key is `0x43c6b5f7`. Removing invented hero attributes 23 and 24 fixed the
  former `0 second cooldown` tooltip: attribute 24 is the client's cooldown
  reduction input, so the old value `1` meant 100% reduction. Live tracing also
  recovered a converter-independent C1 sequence. An all-zero C1 snapshots the
  entry end to the client's current gameplay clock, then a relative C1 with
  duration `10000` extends it by ten seconds. The hook recorded `clock_now=36190`,
  reset `end=36190`, then `end=46190`; live visual confirmation showed the Ride
  icon dim for approximately ten seconds. That fallback was visually incomplete:
  because the entry start stays zero, the clock appears already about 75% full
  and advances to 100%, while retail starts at 0% and sweeps to 100%. A tested
  supposed `ObjectJump` clock sample followed by a type-0
  `ActionCommandResponse` did not cover the Ride slot or produce any cooldown
  UI; the apparently black region was the adjacent empty slot. Later static
  analysis proved the 25-byte ObjectJump body invalid because build 103
  reflection-decodes a 44-byte structure. Its encoder and live uses are removed.
  Trace which response object/runtime key the native handler inserts before
  retrying it. darkspin enforces the same range, power, and cooldown
  authoritatively, rejecting rapid repeats without teleporting. Native handler
  analysis shows that a nonzero source start takes the absolute start/end
  branch. A focused source-start `1`, duration `10000` run produced no cooldown
  indicator at all, proving that the unanchored remote converter cannot treat
  `1` as client-now. Remove that experiment and recover the accepted type-0
  `ActionCommandResponse` local timing fields instead. A later synchronized
  attempt reused the accepted response's darkspin-elapsed timestamp in the C1
  source-start field. The client hook converted source `284345` at local clock
  `190492` into the invalid start `103079215176`, and live testing showed no
  cooldown indicator. That experiment is also removed. The clock value must be
  recovered from the actual protocol epoch/unit rather than copied from darkspin
  process uptime. Static analysis then found the missing clock origin in the
  build-103 `GameState` handler. It reads a fixed 25-byte body and passes its
  first 64-bit field to the remote-clock initializer. This is wire `0x8a`, not
  the empty `Connected` trigger at `0x82`. A Unix-millisecond experiment converted to
  `0x0000000400000000`, not the millisecond gameplay clock, and again produced
  no cooldown; that epoch is removed. The next build uses the same monotonic
  server-uptime milliseconds already carried by RakNet's connection-accepted
  exchange and later ability, modifier, death, and animation timestamps.
  Runtime inspection confirms logical GMS message `11` maps to transport
  ID `0x8a`. Substituting `0x8a` at the `Connected` handshake point with
  zero state/type fields
  reproducibly diverted three clean launches to the squad START/menu path and
  broke tutorial auto-entry. That experiment is reverted. Do not change the
  handshake wire ID again. Send a properly populated GameState only at the
  recovered gameplay setup phase. A clean stable-path retest initially showed
  a cooldown sweep beginning around 50%, no spawn-in effect, and no Ride-use
  arrow. The valid GameState plus cast-time-anchored C1 now converts the cooldown
  start to within roughly 200 ms of client `clock_now` and visibly produces the
  correct sweep. Spawn-in is also confirmed. The Ride-use arrow is now
  live-confirmed too: the safe pre-trigger spawn prevents the client-only
  one-shot Lua job from firing before field-21 count `3` arrives, so its
  availability predicate accepts HUD index `2`; the first accepted Ride cast
  clears the arrow.
  Retail footage also shows an arrow pointing at the newly
  unlocked Ride icon until the player presses it. The retail
  `Tutorial_IntroAbilities.lua` contract waits two seconds and calls
  `nUIManager.PlayAbilityBlink(Ability_Enrage1)`. A later clean live run
  confirmed the server-authored objective displays that arrow; the earlier
  2.188-second observer sample was about 60 ms premature. On the first accepted
  Ride cast, the same objective is sent hidden/completed and clears the arrow.
  Keep launch-DLL hooks observation-only.
  Voltic Slash (ability index `0`, authored key `LightningRogueBasic`) reports
  a 0.4-second cooldown in build 103. It now uses the same absolute-time C1 path
  and an authoritative 400 ms rejection window, with deterministic 8 damage
  (the midpoint of 4-12). A live target cast reduced 20 HP to 12 and traced a
  400 ms entry whose converted start was within 183 ms of `clock_now`; two
  clicks 60 ms apart produced only one request, confirming client enforcement.
  Decompiled retail bytecode establishes that Ride is single-target rather
  than area-of-effect: a hostile target is
  resolved to a good melee position, teleported to, damaged, shocked for three
  seconds, and checked for a kill-triggered cooldown reset. Ground-target use
  teleports without damage. darkspin now separates those paths, preserves the
  target ID, validates range against the encounter target, applies a
  deterministic 18-damage midpoint, tracks the three-second authoritative
  Shock interval, and resets authoritative/client cooldown state on a kill.
  Build-103 disassembly now confirms `ModifierCreated` is a packed 37-byte
  payload, not reflection. A byte-tested encoder sends Shock's definition hash,
  target/source IDs, instance ID, three-second duration/start time, and stack
  count, followed by a scheduled eight-byte `ModifierDeleted`. A live targeted
  cast used object ID `14` and reduced the stationary test mob from 20 to 2 HP.
  Rapid retries were rejected during the authoritative cooldown, and killing
  the target reset Ride immediately. Targeted damage and kill reset are
  confirmed. A fresh timed-message run visibly showed the stun/Shock effect
  on a mob after Ride landed, so presentation is now confirmed too.
  Build-103 runtime observation recovered all 99 cumulative XP boundaries from
  property `0xC0B32F0F`; the opening values are `100, 200, 3000`. Static code
  confirms the comparison is strict, so level 2 begins at XP `101`. darkspin now
  carries account XP into the initial LabsPlayer snapshot and has an exact
  partial fields `15`/`16` progression encoder. The first completed encounter
  stage sends level `2`, XP `101` once per gameplay session. A focused live run
  cleared the opening pair and visibly presented `You reached Crogenitor Level
  2!`, immediately followed by the authored three-enemy wave. The passive
  explanation did not appear during the next five-second observation and is
  tracked separately above.
  The focused snapshot no longer replicates objects `11`-`14` during dungeon
  setup. They remain recorded at their authored marker positions and spawn only
  after entering a focused eight-unit perimeter, which leaves the test spawn
  outside. This test group receives no movement or aggro packet on creation, so
  it stays at its marker until its encounter behavior is implemented. Preserve
  that trigger boundary when restoring the full tutorial route; do not populate
  every map encounter at game start.
  Live traces distinguish the two authored request forms: a ground cast arrives
  with target ID `0` and correctly teleports without damage, while casting on
  the mob arrives with target ID `14` and damages that single target. Do not
  infer a nearby encounter target from a target-zero ground cast.
  To shorten iteration, the ability-unlock event now creates one stationary
  20-HP test enemy six units north of Blitz and registers that exact dynamic
  position in the encounter. Remove this test spawn when the full authored
  encounter sequence replaces the focused snapshot.
  Mob creation originally missed the normal spawn-in particle effect. Recover
  whether build 103 drives it through the authored `character_teleport_in`
  animation, a `ServerEvent`, or both, and emit it before activation without
  coupling it to an immediate movement goal. The extracted build-103 assets
  identify `generic_spawn.ServerEventDef` (`generic_appearance_cover`) as the
  strongest generic candidate. IDA maps application message `28` / packet
  `0x9b` to `sub_539E90`, which reflection-decodes a 0x98-byte ServerEvent
  structure. The build-103 field schema was recovered and a minimal asset,
  object, and position event was accepted, but a live run showed no particle.
  That ineffective event is removed from the live spawn sequence. Test the
  authored species beam-in/teleport events and any required attach/force flags
  instead of treating `generic_spawn` as solved.
  The first unlock-time mob appeared at the intended top-of-screen position but
  immediately walked northeast, consistent with an uninitialized locomotion
  goal falling back to `(0,0,0)`. The focused spawn sequence now follows create
  and combat state with an explicit visible `ObjectUpdate`, `ObjectPlayerMove`
  stop (`0x20`) whose goal equals the authored spawn position, and
  `character_teleport_in`. A live run confirms the mob remains pinned. The
  animation initially remained invisible because its timestamp was evaluated
  without a valid shared source clock.
  The extracted `ability_firstaggro_beamin_tutorial.lua` confirms the authored
  presentation is visibility plus `character_teleport_in`, with no separate
  ServerEvent effect. Build-103 receiver `sub_53A1F0` converts the packet's
  64-bit timestamp through the shared source clock before applying the state.
  A focused sequence also sent the now-removed invalid `ObjectJump` body before
  the animation. It was not a valid clock anchor and provides no evidence about
  animation timing. The minimal `generic_spawn` event remains a valid negative.
  A correctly populated `GameState` clock-origin message now gives animation
  and cooldown timestamps a defined remote epoch. It must not
  be sent as the connection handshake: a zero-valued GameState reproducibly
  routes build 103 to the squad START/menu. Sending Dungeon/Tutorial GameState
  immediately before world replication preserves tutorial auto-entry,
  initializes the clock mapper, and makes the mob spawn-in animation visible.
  Spawn-in presentation is confirmed; no artificial create delay or generic
  spawn ServerEvent is needed.
  Apply the authored-position update and stopped goal to every tutorial mob at
  creation; send a different locomotion goal only when encounter AI
  intentionally begins movement.
- [x] Implement obelisk activation and the Electro Claws tutorial reward. The
  first authored loot obelisk is marker `2655870385`, object ID `15` in
  darkspin's tutorial sequence, noun `prefab_boss_obelisk.Noun`, position
  `(191.91814,22.70923,29.76409)`, scale `0.75`, one use, challenge `500`.
  Clearing the currently implemented encounter sequence now creates it with
  collision, an explicit visible position, build-103 `cInteractableData`, and
  object interactable state `3`. IDA confirms packet `0x98` reads the object ID
  followed by the three-field bitmap (`mNumTimesUsed`, `mNumUsesAllowed`,
  `mInteractableAbility`); object reflection fields `21` and `22` carry
  `mInteractableState` and the source marker. A focused live run confirms the
  obelisk renders, Blitz walks into range, and the client emits type-11 target
  `15` at `(185.36333,19.537325,28.978449)`. The server now accepts that use
  once, increments `mNumTimesUsed` to `1`, and transitions the object to
  consumed state `1`; focused wire tests cover this response. The follow-up
  build-103 run now verifies the response live: the server logs the one
  accepted use and a second click produces no further client interaction
  request. The used obelisk model remains visible, so the exact `boss_1`
  graphics presentation is not yet distinguished. The earlier login blocker
  was a launch-DLL race: the injected submit also set two controller bytes,
  causing the UI thread to send the same 276-byte request concurrently. The
  hook now submits only through the login manager on the login UI thread; a
  worker-thread retry reproduced the same race even without those bytes. The
  corrected clean trace contains one 276-byte send and one command-40 request
  and reaches gameplay. The build-103 item-acquired path is
  now narrowed precisely: the server sends `loot_acquired.ServerEventDef` as
  the event asset and `LootAwarded` as `clientEventID`, and the retail client
  reads the complete ten-field loot tail. `lootRigblockId` is the
  `AssetCatalog/AssetGlobals` key, not the numeric `rigblockId` metadata or the
  resulting `cLootData` asset hash. For Electro Claws, catalog key
  `_Generated/LootRigblock268.LootRigblock` is `0x096B7C20`; build 103 maps it
  to packaged asset `0x646569E2`. A focused live run confirms retail conversion
  and formatting both return success, the “You've Received — Encrypted Item —
  Electro Claws” card renders, and the grant persists exactly one level-5
  rigblock-268 part for the authenticated user. The authored grant operation is
  idempotent and rolls back its in-memory append when persistence fails. The
  used obelisk's exact `boss_1` graphics presentation remains a visual-fidelity
  follow-up, not a tutorial-playability blocker.
  Interaction and collection are now distinct authorities. Clicking object
  `15` starts the authored timeline without granting inventory or opening the
  Sage phase. At `1.6s`, object `16` is created at the obelisk and moves four
  units toward the player's interaction side. Only movement contact within two
  units persists Electro Claws, emits the `LootAwarded` card, deletes object
  `16`, and advances the route. A failed save leaves the pickup collectible.
- [ ] Implement squad ability, hero switching, orb/capsule pickup, and shared
  health/power recovery. The authored Sage sphere at
  `(259.346,81.391,25.088)`, radius `25`, now fires once after the loot
  obelisk. A live build-103 run confirmed a full `LabsPlayerUpdate` with
  `PC_LF_Mage.Noun`, asset `0x55D1408F`, and creature boundary `2` produces
  the unlock presentation without crashing. Pressing W/Q emits type-5 action
  commands for creature indices `1`/`0`; accepted response plus
  `PlayerCharacterDeploy` changes the selected portrait and action bar between
  Sage's Bio abilities and Blitz's Plasma abilities. Historical server source
  and build-103 packet structure establish that deploy targets a distinct
  object for each creature; `SetObjectGFXState` only carries object ID, state,
  and timestamp and cannot replace a noun. darkspin now creates Sage as object
  `17`, keeps Blitz as object `1`, positions the selected object at the shared
  authoritative location, and deploys the matching object on W/Q. Live testing
  now shows only Sage after W, accepts movement from object `17`, and returns to
  only Blitz after Q. A sparse tagged update of fixed character field `3` is
  invalid: it treated the primary noun hash as a pointer and crashed at
  `0x009C8EAB`; do not restore that candidate. The capsule half of this item is
  now implemented and live-confirmed in build 103. All ten
  authored fixed smart objects use stable runtime IDs `18`-`27`, exact marker
  positions, their `HealthOrbPlaced.Noun`/`ManaOrbPlaced.Noun` assets, and a
  stationary goal so they cannot drift toward the origin. Swept proximity
  collection is one-shot, restores 15 up to 200, updates Blitz and unlocked
  Sage together, emits `health_orb_pickup`/`health_orb_full` or the mana
  equivalents, and deletes the consumed object. Static registration corrects
  ServerEvent reflection field `14` to `textValue`; it was previously
  mislabeled as a loot player index. Live-verify capsule visibility, pickup
  presentation. A focused run from the preserved recovery checkpoint showed
  stationary green/blue capsules, one-shot deletion, `Health Full`, visible
  power-arc restoration, and `Power + 15`; the server recorded the preceding
  health capsule as restoring 15 before the next full capsule. Shared Blitz/Sage
  recovery still needs a post-unlock live pass before checking this combined
  item complete. Static reflection now establishes sparse recovery candidates:
  HP-only `0x97` mask `0x01`, mana-only mask `0x02`, both mask `0x03`, and no
  combatant packet for already-full feedback. The present encoder always sends
  both fields and the pickup path also does so for full events; this is
  decodable scaffolding, not native parity. Capture post-Sage object IDs and
  masks before deciding whether recovery is active-only, squad-mirrored, or
  copied during deploy. Darkspin now uses active-only as the narrowest
  compatibility policy and no longer overwrites the reserve hero with the
  entrant's resource value. The build-103 client callback is now closed:
  `Orb_Pickup -> sub_9C98A0` is a false-returning stub and sends no request, so
  accepted movement remains the authoritative input. Also keep the noun
  variants distinct: placed capsules are lifetime-zero with authored `1x1x1`
  boxes; dropped orbs are lifetime-30 with `2x2x4` boxes. Recipe 15 now stores
  those four noun lifetimes and pickup boxes in `content.db`, and swept capsule
  collection consumes the projected box expanded by the deployed hero's
  content-owned footprint (`0.8` Blitz, `0.825` Sage) instead of the old
  universal `1.3` sphere. Sage's packaged
  SupportHealerBasic and TreeOfLife paths are now active; the unidentified
  numeric index-0 basic and return persistence remain separate work.
- [ ] Complete authoritative creature-swap presentation and visible cooldown.
  An accepted
  type-5 command now publishes the old hero's normal teleport beam-out and
  animation, transfers the deployed authority at the shared position, then
  publishes the new hero's beam-in and animation. Same-hero requests and
  attempts before the deadline are rejected. Build 103 loads the creature-card
  cooldown from game tuning with a retail default of `30s`; darkspin uses that
  same server-side duration instead of allowing immediate W/Q swap-back spam.
  The beam is now emitted as the authored positioned swarm effect rather than
  a small object attachment. Live smoke testing still shows no portrait fill,
  so the tutorial's effective duration and client-visible card timestamp path
  remain open and the 30-second default is not considered parity-complete.
- [x] Implement the enemy-clear security gate and teleporter activation. The
  authored `BossSecurityTeleporter.Noun` marker `495984414` is runtime object
  `28` at `(260.83334,232.78407,20.16802)`. It starts with the retail inactive
  boss-teleporter event, remains server-locked while the tracked encounter has
  living enemies. Final-enemy defeat now emits the authored
  `zelem_boss_teleporter_powerup` event, then publishes the persistent active
  event one second later. Production uses delayed publication; deterministic
  adapters without a scheduler preserve power-up-before-active ordering in one
  response. Trigger/handoff authority is installed at that same deadline, so
  contact during the power-up interval cannot teleport early. This replaces the earlier same-batch pair that rendered two
  overlapping teleporters and the later single-active shortcut that omitted
  the startup animation. A preserved focused checkpoint at
  `(250,225,20.16802)` creates one stationary blocker for repeatable testing.
  A live build-103 run showed the inactive gate, final-enemy deletion, and the
  visible power-up dome. A separate clean crossing of the 6.67-unit swept
  trigger teleported Blitz once to destination marker `174193625` at
  `(-347.57553,-224.60829,10.08803)` and visibly moved the camera into the red
  arena. The next slice is the radius-30 arena director and its horde waves.

## Milestone 3: Enemies and combat

- [ ] Spawn the first-visit noun set: Basic Diseased, Ranged, Poison, Sloth,
  Special One, and Special One Intro where the authored runtime requires them.
  The first two Basic Diseased now spawn and render from the AI markers.
- [ ] Implement the minimum AI loop needed by the tutorial: aggro, navigation,
  target selection, attacks, cooldowns, damage, status effects, and death.
  Enemies must remain in their authored idle/pre-aggro state after spawning,
  enter aggro only when the hero crosses their perception perimeter, pause for
  the authored first-aggro beat, pursue by smooth locomotion, and attack in
  range. The opening pair is now withheld from the dungeon setup snapshot and
  created once when player movement first crosses the authored 18-unit alert
  perimeter, preventing the old immediate level-load spawn. A clean live run
  confirmed the one-time boundary at `(97.636,-89.381)`, 17.6 units from the
  first marker. The recovered build-103 `FirstAggro_BeamIn_Tutorial` script
  makes the noun visible and targetable, assigns the hero target, plays
  `character_teleport_in`, then waits `1.291667` seconds. darkspin now sends that
  first-aggro state and an authoritative stop at the AI marker; a live run
  confirms the enemy remains pinned instead of running north. The animation
  packet is sent but the beam-in animation is not visibly playing yet. The
  delayed RakNet application-send path now works through the shared UDP mux:
  a clean live run acknowledged every scheduled datagram after the exact
  pause. Build 103 ignored `ObjectPlayerMove` (`0x91`) or repeated
  `ObjectUpdate` (`0x8c`) positions in isolation. The confirmed transition is
  `ObjectPlayerMove` with active-goal flags `0x01`, immediately followed by the
  fixed 16-byte `LocomotionDataUnreliableUpdate` (`0x95`) body containing the
  object ID and goal vector. A clean live run visibly walked both infectors
  from their AI markers to the hero after the authored pause and left them
  settled around the target. The old immediate 4-unit/1.5-second retaliation
  timer is removed. Diseased enemies now use the compiled
  `TutorialPoisonCloud` definition: eight-unit admission, target-facing stop,
  `ver_minn_lf_diseased_attack1`, a `0.17s` shot, generic projectile creation,
  attached trail, authoritative swept collision over the 12-unit travel budget,
  rank-one `1-4` damage, impact presentation, independent `1.86s` release, and
  cooldown maturity at `3.17s`. Each actor owns its own cancellable run and
  refreshes player position and HP during flight. Out-of-range actors pursue
  and retry on the compatibility polling cadence; defeated actors and replaced
  sessions cannot publish stale continuations. Attack animation, projectile
  presentation, and floating combat text still require live confirmation.
- [ ] Implement player damage, ability resolution, cooldown/resource costs,
  death prevention or recovery behavior required by the tutorial.
- [x] Establish a live minimum player-to-enemy damage slice: type-7 ability
  acknowledgement, `CombatEvent`, combatant HP update, and `ObjectDelete` all
  work against the opening infectors.
- [ ] Live-confirm floating combat numbers for damage in both directions.
  Build-103 static analysis shows `CombatEvent.deltaHealth` is a positive
  display magnitude while `integerHpChange` remains signed; darkspin incorrectly
  sent `-10` for both and now sends `deltaHealth=10`, `integerHpChange=-10` for
  hero-to-enemy hits. A live retest still showed no rising number. The first
  enemy-to-player `CombatEvent` plus health update now visibly lowers the hero's
  health bar too, but likewise produces no rising number. Static receiver
  tracing closes the packet boundary: no separate floating-text packet is
  expected. `sub_4E2CA0` synthesizes a local combat-text ServerEvent only when
  `sub_4E57B0` resolves the active controlled simulation object and the relevant
  `ShowDamageDone*` setting is enabled. Build-103 reflection metadata proves
  top-level `LabsPlayer` field `9` is the controlled object at `+0x1238`.
  Darkspin previously omitted it; initial snapshots now bind Blitz object `1`,
  and squad swaps send the exact sparse field-9 update for object `1` or `17`
  before `PlayerCharacterDeploy`. Fang records the controlled-object handle,
  resolved object ID, player index, decoded sparse target/source, signed HP change, and a relation
  bitset (`1` means controlled source; `2` means controlled target) for every
  received damage event. The 2026-07-20 trace proved offset `+0x1238` stores the
  resolved client object handle rather than the wire ID itself; the old direct
  comparison therefore produced false relation zero for every valid hit.
  The initial full player snapshot also preceded Blitz `ObjectCreate`, so its
  resolved `tObjID` could not establish the client handle. Dungeon setup now
  refreshes sparse field `9` immediately after object creation and before
  deployment, with a sequence regression for that ordering.
  Live-confirm a nonzero relation and visible text with
  the corrected reflection; retain the target object through the presentation
  interval. Native `sub_A207F0` sends reflected type `0x5f1cc727`;
  ordinary damage selects mask `0x9b`, flags `0x0001` (`0x0005` on a killing
  blow), positive float damage, target/source IDs, and negative integer HP
  change for a 20-byte application packet. Darkspin now uses that minimal
  native form and locks it with a byte-exact golden; the remaining live defect
  is downstream of packet construction.
- [x] Replace the scaffolded 1.5-second death deletion with recovered
  `Behavior_Death` semantics. At zero HP, clear the target, emit the selected
  build-103 death `SetAnimationState` (`0xa5`), immobilize, stop/turn, and
  disable collision. An ordinary NPC permits revival for ten seconds, then
  attaches its creature-type fade for five seconds before marking for delete;
  `TutorialBasicDiseased` is Life and therefore uses
  `fadeaway_bio.ServerEventDef`. A critical non-player/non-boss uses the
  separate 0.1-second plus 3-second fast-delete branch and attaches no explicit
  elemental fade. Revival/interruption must reset animation, remove the effect
  and modifier, restore collision, and cancel stale deletion. Animation
  duration is not the authored deletion clock. The deterministic death behavior
  now covers the ordinary, critical non-boss, critical boss, player-revival,
  interruption, and stale-callback branches. The build-103 enemy adapter emits
  the killing combat/HP update, selected animation, physics collision change,
  an authoritative `ObjectPlayerMove` stop at the corpse position, elemental
  fade attachment/removal, and final object deletion on the recovered clocks.
- [ ] Implement loot/orb ownership, pickup validation, inventory mutation, and
  replicated attribute changes. Build-103 analysis now separates this from XP
  and corpse deletion: object byte `+153` gates loot, byte `+154` gates XP,
  `DropStuffForObject` queues combatant drop state at `+92..+104`, and the later
  processor creates pickups without changing Labs XP. Native selector bits are
  `2` orbs, `4` catalysts, `8` loot, and `16` DNA; `1` is no drop. Preserve
  authored `dropType=2` for orb-dropping `TutorialBasicPoison` and `dropType=1`
  for `TutorialBasicPoisonNoOrbs`; do not infer drops from their shared visible
  Chlorosaur identity. The exact drop presentation is ServerEvent fields `6`
  asset and `10` position, not field `13` target point. Generic pickup create,
  launch replication, ownership, and restore policy remain open.
- [ ] Verify rejected actions do not mutate or persist game state and storage
  failures do not leave accepted in-memory mutations behind.

## Milestone 4: Horde, Sage, and completion

- [ ] Recover and implement the horde director semantics, including the four
  waves and two authored spawn loci. The playable build-103 state machine now
  waits the authored two seconds after arena entry, runs exactly four waves,
  and places one stationary enemy at each registered listener per wave. It
  deterministically exercises all four first-time minion nouns and the special
  noun from the level configuration, uses the packaged `horde_beam_in`
  presentation, accepts combat damage, and advances 1.5 seconds after a clear.
  A focused live pass confirmed gate entry, both wave-one loci, authoritative
  kills, and wave two's Sloth. Recover the retail director's random budget and
  exact counts/timing before checking this complete and add per-wave
  aggro/attacks. `Horde incoming!` and `Horde defeated!` are now emitted at
  arena entry and the terminal fourth-wave clear through standalone build-103
  `ServerEvent.clientEventID` field 15. IDA confirms the ServerEvent handler
  passes this field directly to the alert dispatcher; the exact retail IDs are
  `0x1d42121d` and `0x8047eaf4`. A focused live pass trace-confirmed the
  incoming field-15 packet in the same response batch as the successful gate
  teleport, followed by the authored two-second wave-one spawn. The short-lived
  banner is now live-confirmed. A complete four-wave pass proved the defeated
  event is sent, but the immediately following `0xC8` tore the client down to
  a black screen before its banner could render. `0xC8` is therefore withheld
  from the kill boundary until the retail return interaction is recovered.
  Both authored arena health
  obelisks are now created only on arena entry at markers `785296806` and
  `2012042454`. Each accepts `InteractHealthObelisk` once, enters consumed
  state, emits its activation event, and creates one nearby green health
  capsule without directly changing hero HP. The capsule reuses authoritative
  movement pickup and restores the ordinary `15` health. Full-health contact
  shows the full-resource event and leaves the capsule present. Marker,
  one-use, spawn, movement-pickup, and non-consumption tests pass; live-confirm
  the model, activation effect, ejection direction, and click range before
  treating this slice as visually complete. The English localization keys are
  pinned as `0x09ed690c` and `0x09ed6f0c` respectively.
- [ ] Identify and implement the final clear predicate and arena exit. The
  server treats the second kill in wave four as the terminal horde clear and
  leaves the completed session in the arena awaiting a return interaction.
  A focused build-103 pass proved that emitting `TutorialGameMsgs` subtype `0`
  directly after the defeated alert immediately fades the scene to a permanent
  black screen; it does not create `RETURN TO SHIP`.
  Asset inspection rules out the normal
  invisible `LevelExitPoint.Noun`: the tutorial level does not include that
  Cryos markerset layer. Build-103 event dispatch confirms that the local
  `LABS_TUTORIAL_COMPLETE` emitted by `0xC8` writes survey classification `1`
  into `SP_SporeLabs/cSurveySystem` (failure uses `0`, ordinary level end uses
  `2`). This is not the mission-result controller. Release IDA and package
  extraction identify the retail success presentation as `HUD_BeamOut.swf`
  (`SP_UI/cBeamOut`), gated by `nGameDirector.IsBossDead`. Build 103's director
  reflection is carried directly after opcode `0x8B`; field 3
  (`mbBossComplete`) is therefore exactly `8B 08 01`. A live final-wave run
  confirmed that packet reveals the clean in-game `RETURN TO SHIP` button with
  no `MISSION FAILED / COMMAND RELAY TERMINATED` overlay. Clicking it sends
  `PlayerStatusUpdate` status `0x20`, progress `1`. The recovered response is
  `ReconnectPlayer(ChainVoting)`, `DebugPing`, then `PlayerDeparted`; a live run
  confirmed that sequence leaves the arena for a departure/chain-voting screen,
  not the ship. That screen renders `UNDEFINED` result copy, so recover the
  missing tutorial result/route data that advances it to the ship before
  checking this complete. The next focused build moves the already-implemented
  positive `TutorialGameMsgs` subtype `0` snapshot to the confirmed Beam Out
  status-`0x20` acceptance boundary and removes the generic chain reconnect;
  compile tests pass, but this candidate has not yet been live-verified.
- [x] Finish the Sage unlock and immediate hero-switch behavior. The authored
  trigger, unlock presentation, second HUD slot, Sage ability bar, W/Q request
  decode, validation, acknowledgement, and deploy response are live-confirmed.
  Separate Blitz/Sage objects (`1`/`17`) now live-confirm clean world-model
  switching in both directions, Sage locomotion, and the retail `Hero Unlocked!
  Press W to switch to Sage, a Bio Genesis Hero.` prompt. Video review corrected
  the encounter order: crossing the authored marker reveals Sage first, then
  introduces the colocated `TutorialSpecialOne_Intro`/Quadra. Quadra's defeat
  awards XP and advances the encounter; it does not grant Sage. The earlier
  focused pass proved the packet mechanics but tested the wrong ordering. The
  normal authored start is restored afterward.
- [ ] Implement the tutorial-required Sage abilities and determine whether Sage
  is persisted only at mission return.
- [ ] Send `TutorialGameMsgs` (`0xC8`) subtype `0` with positive cumulative XP
  at the correct point and reliability/order. The six-byte application shape
  (`C8 00` plus a little-endian signed cumulative-XP value) is implemented and
  unit-tested. Live testing rejects the terminal fourth-wave kill as its send
  boundary because it immediately produces a permanent black screen. The
  retail return interaction is now recovered as Blaze GameManager
  remove-player reason `6`, but the packet's placement and enclosing RakNet
  priority/reliability/order remain open.
- [x] Persist tutorial completion and the server-owned terminal transition
  idempotently. Gameplay records the final horde boundary on
  the joined tutorial instance without mutating the account. Only the same
  player accepting `RETURN TO SHIP` (Blaze remove-player reason `6`) invokes
  the persistent completion operation before removal. A focused live run
  returned to the ship, opened the Arsenal onboarding lesson, and verified
  profile 1 at onboarding `3000`, cumulative XP `101`, and level `2`. That live
  result remains the retail evidence boundary; the current local policy now
  persists `9000`, inventory marker `1`, and a complete Blitz/Sage/Wraith squad
  to bypass the repeatedly soft-locking ship guides.
  Persistence failure retains the player and emits no removal notification.
  The accepted return now first emits the deployed hero's positioned
  `character_teleport_beam_out` event and `character_teleport_out` animation.
  Production delays the terminal `0xC8` snapshot by `650ms` so the beam can
  render; transports without delayed publication retain beam-before-terminal
  packet order. Duplicate return requests remain fenced by the terminal latch.
- [x] Initialize tutorial gameplay with exactly `100` match-owned DNA. The
  initial build-103 `LabsPlayerUpdate` field `12` now receives `100` regardless
  of persistent account currency, and a restart rebuilds the same clean value.
  Non-tutorial game modes continue to publish the account-owned DNA scalar.
- [ ] Persist intermediate `1000`/`2000` onboarding boundaries and remaining
  tutorial rewards. Terminal local `9000` persistence and starter ownership are
  complete.
- [ ] Finish the retail tutorial-success return and verify the collection room
  receives creature ownership. The mission-failed overlay is replaced: live
  build-103 testing confirms `DirectorState` field 3 (`8B 08 01`) opens
  `HUD_BeamOut.swf`, and status `0x20` plus the recovered reconnect/ping/depart
  response reaches a departure/chain-voting screen rather than the ship.
  Remaining: replace its `UNDEFINED` result data, route onward to the ship, and
  verify tutorial rewards/creature ownership.

## Verification and regression coverage

- [ ] Add packet codec vectors from build-103 captures for every implemented
  transport and gameplay message.
- [ ] Add deterministic simulation tests for each encounter, gate, wave, drop,
  Sage unlock, and completion transition.
- [ ] Add operation tests for success, rejected invariants with no storage
  write, and persistence rollback.
- [ ] Run `go test ./...` and `mage build` after server, launcher, or launch-DLL
  changes.
- [ ] Perform and document one clean, full live run using exactly
  `mage darkspin:auth`, `mage darkspin:server`, and `mage tutorial:run`.

## Current investigation queue

- [x] Establish the first live failure after the `0x09`/`0x0a` exchange from
  `bin/server/darkspin/logs/traces/server.jsonl` and game logs.
- [x] Confirm the build-103 opening, connected handshake, ACK framing, message
  indexes, and shared sequenced/ordered triad layout in static analysis and a
  live client run.
- [x] Confirm that the local hello response is `HelloPlayer` followed by party
  state, without an invented local `PlayerJoined` message.
- [x] Reach the live loading boundary: the client acknowledges the hello/party
  exchange, chain vote/countdown, a split initial `LabsPlayerUpdate` containing
  tutorial Blitz, player status `4`, the follow-up player update, and
  `GamePrepare`.
- [x] Prove that sending `GameStart` at status `4` is invalid: build 103 closes
  the gameplay connection. Keep `GameStart` gated on client status `8`.
- [x] Recover the build-103 `GamePrepareForStart` (`0xB0`) receiver and exact
  16-byte body: level asset, marker-set asset, player mask, and level index.
- [x] Match the packaged tutorial assets by trimming the server game name's
  `_v2` suffix: `Game_Tutorial_cryos_1.Level` and
  `Game_Tutorial_cryos_1_ai.Markerset`.
- [x] Add build-103 launch-DLL scene diagnostics for asset lookup, load request,
  and applied scene change. Enable the client trace in `mage tutorial:run` and
  `mage tutorial:run2` at `bin/server/darkspin/logs/traces/client.jsonl`.
- [x] Confirm live scene resolution, load request, applied scene change, and
  the client's transition from status `4` to status `8`.
- [x] Gate four-byte `GameStart` (`0xB1`) on status `8`, then perform the
  build-103 dungeon sequence: `DebugPing` (`0xCC`), idle `DirectorState`, and
  reset `QuickGame` (`0xAF 01`).
- [x] Resolve the post-`GameStart` creature asset crash. The initial
  `LabsPlayerUpdate` now reflects all three fixed character slots; build 103
  successfully resolves each slot and remains alive in the rendered level.
- [ ] Replace the remaining inferred initial `LabsPlayerUpdate` fields with a byte-accurate
  build-103 layout and confirm all tutorial creature asset prerequisites. The
  locked placeholders currently retain valid Blitz asset/noun metadata while
  remaining type Unknown (`6`), which avoids the fixed-array null lookup but
  visibly duplicates the Blitz portrait. Recover a valid locked-slot payload
  that keeps the fixed-array lookup non-null without adding squad portraits.
  A live negative test retained valid fixed records but zeroed the two trailing
  reflected noun/assets; build 103 reached dungeon setup and then closed, so
  both representations require resolvable locked-slot data. A second controlled
  test zeroed both the fixed and reflected locked records consistently, matching
  the old server-source empty-slot shape; build 103 terminated gameplay and
  returned to login with `80040000`. That experiment formerly mislabeled
  top-level field `23` as a creature counter; the complete reflection map now
  proves it is deck score. The complete local evidence set contains no retail
  aggregation formula, so it remains zero unless external server evidence
  replaces that policy. A third test used the live-resolved
  non-player `TutorialBasicPoisonNoOrbs` noun/asset in both locked records; the
  client reached deployment and then exited, proving a generally resolvable
  gameplay asset is insufficient. The portrait issue itself is now resolved:
  fields `21`, `22`, and `23` are distinct tutorial boundaries. Field `22` is
  the locked deck minimum; setting it to `1` initially keeps the valid backing
  records but visibly presents only Blitz. A live normal-start run remained
  stable and showed one portrait. The Sage refresh sets the boundary to `2`.
- [ ] Determine why the client does not emit the expected six-byte
  `ChainPlayer` request after the chain vote/countdown exchange.
- [x] Keep the client alive after `GameStart` and create/deploy the selected
  tutorial creature. Blitz, the HUD, health/power, abilities, and minimap render
  in Cryos using object ID `1`.
- [x] Identify the initial authoritative movement response and object mapping:
  client `0x9c` movement for hero object ID `1` is answered with build-103
  `0x90` teleport plus `0x91` locomotion flags `0x21`.
- [x] Live-verify movement replication.
- [x] Live-verify stop replication and orientation.
- [x] Send the next authored encounter after both opening infectors are
  defeated. The clear now spawns the three nearby simulated Chlorosaurs once.
- [x] Recover the immediate path transition after the three simulated
  Chlorosaurs. Their clear now spawns the next four authored no-drop
  Chlorosaurs at `x=170-184` once.
- [x] Continue from that four-enemy clear into the authored `x=197-233` group
  around `(233,-96)`.
- [x] Trace the continuous playable route from the correct lower-platform
  start `(177.755,-239.854,0.088)` through the paired ability marker and to the
  last visible map milestone `(267.13,233.87,20.09)`. A clean rebuilt
  `mage darkspin:auth` / `mage darkspin:server` / `mage tutorial:run` session visually confirms the server
  now deploys Blitz on the intended lower Cryos walkway. Initial facing still
  needs an authored quaternion instead of the client default.
- [x] Complete the first active-ability lesson at `(233,-96)`. The initial
  build-103 snapshot now sets the simulation-player ability boundary to `1`;
  the server-only marker applies its authored delayed `1 -> 2 -> 3` progression
  and exposes Ride the Lightning
  at HUD index `2`, sends the objective/`vo_ship_tut_abilities` lesson, and
  displays the client-authored `Press 1` arrow. The first accepted cast clears
  the arrow, teleports Blitz, deals 18 damage, shocks a surviving target for
  three seconds, spends 13 power, and runs a source-clock-anchored ten-second
  cooldown. Voltic Slash at index `0` also deals 8 damage and observes its
  authored 0.4-second cooldown.
- [x] Live-verify the separate server-only second-ability marker at
  `(232.287,-93.752,10.526)`. It is implemented as an idempotent, player-radius-
  padded segment trigger that waits the authored one second and emits boundary
  updates `2` then `3`, reproducing the callback's two `UnlockNextAbility`
  calls. A clean focused replay crossed both authored markers in one movement:
  the server scheduled the boundary updates, the HUD exposed only Ride, and
  the blink observer recorded accepted availability `0x10002` while the white
  arrow visibly pointed at the Ride icon.
- [x] Restore fallback tutorial Blitz's authored 4-12 weapon range when an
  incomplete profile omits combat stats, preventing the opening enemy from
  rejecting every basic attack as zero projected damage.
- [x] Make developer `/victory` emit the normal horde-defeated presentation,
  then wait the authored two-second result boundary before boss completion so
  the client observes the order used by the ordinary final-horde path.
- 2026-08-26: tutorial Blitz now starts at ability boundary `2`, which retains Basic and his passive without exposing the first pressable secondary action; the authored two-call lesson consumes its redundant passive-boundary call and publishes boundary `3` only once.
- 2026-08-26: fallback tutorial Sage now receives the same 4-12 weapon range as fallback Blitz, preventing SupportHealerBasic from failing zero-damage projection. NPC-only pursuit projection now also reaches authored boss-arena enemies outside the ordinary three-unit navigation envelope, restoring their AI and the player's pursuit targetability without relaxing hero movement validation.
