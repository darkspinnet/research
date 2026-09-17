# Campaign 1-1 traversal and combat parity

## Scope and evidence rules

This audit compares the complete `bin/video/walkthrough/1-1/1-1.mkv`
walkthrough with the current campaign runtime, from level entry at about
`02:17` through the Beam Out click at about `14:39`. Timestamps are video
timestamps, not time since deployment. The source is 29.97 fps; citations use
the nearest useful second because the edited overhead recording does not
preserve an authoritative simulation clock.

Evidence labels:

- **Exact**: recovered build-103 `content.db`, Lua, marker, or native-client
  behavior. This can define client/content behavior but still cannot invent a
  missing retail-server selection policy.
- **Video**: visible in this one edited run. Counts, pack boundaries,
  acquisition distances, and per-kill drops are implementation placeholders,
  not universal retail policy.
- **Fallback**: current Darkspin policy chosen where retail server authority is
  absent.
- **Matching**: current behavior agrees with the cited evidence at the stated
  confidence.
- **Contradicted**: current behavior visibly or exactly disagrees. A
  video-only contradiction is explicitly identified as such.
- **Unimplemented**: the current normal campaign path has no owner for the
  observed/recovered behavior.
- **Not observable**: the footage cannot distinguish the current behavior or
  no authoritative retail policy is available.

The footage exposes presentation names such as Cannonator, Reparatron, Space
Barracuda, Decelerator, Invincitron, and Illust. Low-band content maps
Invincitron exactly to `NomadWithDrone`, Decelerator to `NomadSnipe`, and
Illust to `ZelemSpecialHaster_Captain`. Decoded class-attribute resources also
map Cannonator exactly to `ZelemBasicHybrid`, Reparatron to
`ZelemBasicRepair`, and Space Barracuda to `ZelemBasicRanged`; each payload
contains the presentation label, localization GUID, and matching
`.ClassAttributes` identity. The walkthrough fixture's use of those nouns is
therefore exact at the label boundary, while its count, split, and placement
remain presentation-derived fallbacks.

The user-observed 1-1 roster also includes **Chrono Striker, Cannonator, Pack
Brawler, Homing Striker, Sting Raider, and Decelerator**. This establishes
level eligibility, not fixed pack composition or simultaneous spawning.
All six mappings are recovered: Chrono Striker is `VerdanthBasicMelee`,
Cannonator is `ZelemBasicHybrid`, Pack Brawler is `ZelemBasicPackMelee`,
Homing Striker is `ZelemBasicRangedHoming`, Sting Raider is
`ZelemBasicFlyingMelee`, and Decelerator is `NomadSnipe`. Their decoded
class-attribute payloads and English localization rows agree exactly. What
remains unresolved is the retail selection path that admits these families
beyond the base low-band `zelems_1` rows, plus their pack counts and placement.

## Executive result

The current runtime has credible low-level owners for proximity population,
per-family pursuit/attacks, two authored horde marker sets, five ordinary
security hops, obelisks, Illust, drops, and Beam Out/results. It does not yet
reproduce the observed traversal.

The completion-critical mismatch is the missing boss-security handoff. The
normal route ends after five generic security pairs at approximately
`(191,736,0)`. Content places the distinct `BossSecurityTeleporter` at
`(184.2345,558.9578,10.2027)` and its strongest destination candidate at
`(928.1883,668.9045,0.0880)`, beside the final arena. The walkthrough uses that
red transfer at `12:45-12:47`. `campaignSecurityTeleportRoute` has no sixth
entry and no boss-security owner; only the developer event can place the
player in the boss arena. This can block a normal run before Illust.

The next largest parity error is encounter activation. Ordinary actors in the
video are already visible and locally dormant before acquisition. Current
population creates a pack only when a locus is consumed and
`marshalCampaignEnemyTargetedSpawns` immediately installs the player target and
combat state. That is a **video-derived fallback mismatch**, not proof of a
retail perception radius: content authors `aggroRange=0` and `alertRange=0`, so
the missing retail server target-insertion policy remains unknown.

The final arena is also ordered differently. The video shows an add phase from
`13:05`, a second `Horde incoming!` at `13:45-13:47`, and the Illust boss bar
only at about `13:55`. Current code admits Illust and four adds together after
the arming delay, then admits two more adds only after the first four are dead
and Illust is at or below 60 percent health.

## Current-owner and test index

The encounter tables use these compact references:

| ID | Current Go owner | Focused tests |
| --- | --- | --- |
| **P** | `server/campaign_population.go`: `(*campaignPopulationSession).PrimeOpening`, `Observe`; `server/campaign_spawn.go`: population planning and `marshalCampaignEnemyTargetedSpawns` | `campaign_population_test.go:TestCampaignPopulationSessionObservesCurrentPositionAndConsumesNearestEntry`, `...PrimesNearestOpeningWandererOnce`, `...LeavesDistantOpeningWandererForTraversal`; `campaign_spawn_test.go:TestCampaignPopulationSessionPlansDifficultyFilteredWandererNouns`, `...PlansSpikeCaptainAndMinions`, `TestMarshalCampaignEnemyTargetedSpawnPublishesTargetAfterBaselines` |
| **A** | `server/campaign_enemy_action.go`: `campaignEnemyFirstAction`; `campaign_pursuit.go`: `(*campaignEnemySession).AdvancePursuit`; family planners in `campaign_enemy_{zelem,haster,snipe}.go` and `campaign_enemy_attack.go` | `campaign_enemy_action_test.go:TestCampaignEnemyFirstActionPreservesRecoveredFamilyPolicies`; `campaign_pursuit_test.go:TestCampaignEnemyAdvancePursuitOwnsLivePositionAndStrictRange`, `...TracksMovingTarget`; `campaign_enemy_attack_test.go:TestCampaignEnemyAttackTimelineRepeatsFromAuthoredCooldown` |
| **H** | `server/campaign_horde.go`: `planInitialCampaignHordeWave`, `planCampaignHordeWave`; `campaign_horde_state.go`: `Defeat`, `AdmitNextWave`, `ConstrainMovement` | `campaign_horde_test.go:TestPlanInitialCampaignHordeWaveUsesScopedListenersAndAgentPool`, `...SecondWaveExpandsTwoAuthoredListenersToThreeActors`; `campaign_horde_state_test.go:TestCampaignHordeSessionAdvancesTwoWaveEncounterExactlyOnce`, `...GateRepelsUntilScopedEncounterCompletes` |
| **S** | `server/campaign_security.go`: `planCampaignSecurityTeleport`, `marshalCampaignSecurityTeleport`; movement integration in `gameplay_udp.go` | `campaign_security_test.go:TestFirstCampaignSecurityTeleportRequiresClearedLocalThreats`, `...ThreatUsesAuthoredRadiusAndEnemyFootprint`, `...PublishesCompleteHandoff`, `...RouteAdvancesToSecondAuthoredPair`; `campaign_contact_test.go:TestCampaignTeleportLandingActivatesClearSecurityTeleporter` |
| **BS** | Recovered generic runner: `server/security_teleporter_run.go:newSecurityTeleporterRun`; **no normal campaign boss-security route owner**. `developer_event.go` is diagnostic-only. | `security_teleporter_run_test.go:TestSecurityTeleporterRunActivatesExactEffectAndTrigger`, `...FiltersEntrantsAndSuppressesActiveModifier`; `sim/lua_content_test.go:TestRetailBossSecurityTeleporterActivationMatchesGoOracle`, `TestRetailTeleporterModifierMatchesGoOracle` |
| **O** | `server/game/campaign_script.go`: `(*CampaignScriptRegistry).PrepareUse`, `CommitUse`; `campaign_interactable.go:newCampaignInteractableRun`; use handler in `gameplay_udp.go` | `campaign_interactable_test.go:TestCampaignInteractableRunExecutesHealthSelectorAtAuthoredDeadline`; `campaign_objective_test.go:TestCampaignObjectiveMessagesSelectObeliskChallengeWithoutPopup`; `game/contentsqlite/director_test.go:TestRetailInitialChainSelectsThreeLootAndTwoHealthObelisks` |
| **D** | Defeat/drop transaction in `server/gameplay_udp.go`; equipment, orb, crystal, and DNA helpers | `campaign_loot_test.go:TestCampaignEnemyEquipmentRollsOnlyOncePerDefeatedActor`; equivalent once-only tests in `campaign_orb_test.go`, `campaign_crystal_test.go`, and `campaign_dna_test.go` |
| **B** | `server/campaign_boss.go`: `planInitialCampaignBossEncounter`, `(*campaignBossSession).Arm`, `Admit`, `ObserveDamage`, `planCampaignBossSecondWave`, `marshalCampaignBoss{Active,Complete}`; scheduling in `gameplay_udp.go` | `campaign_boss_test.go:TestPlanInitialCampaignBossEncounterUsesAuthoredLeaderAndAddAnchors`, `...RequiresThresholdBothAddWavesAndLeaderDefeat`, `...SecondWaveUsesFirstTwoAuthoredAddAnchors`, `...CompleteClearsHordeAndRetainsLeader` |
| **U** | `server/campaign_unlock.go`; unlock scheduling in `gameplay_udp.go` | `campaign_unlock_test.go:TestCampaignInitialAbilityCountUsesFirstClearBoundaryOnlyForOneOne`; `player_unlock_sim_test.go:TestSoloSupportUnlockRunExecutesRecoveredSixSecondMutation` |
| **R** | `(*campaignBossSession).ReserveBeamOut`, `CommitBeamOut`; `gameplay_udp.go` `PlayerStatusUpdate` handler; `campaign_result.go` | `campaign_boss_test.go:TestCampaignBossSessionRequiresThresholdBothAddWavesAndLeaderDefeat`; `campaign_result_test.go:TestCampaignResultSnapshotFreezesMatchIdentity`, `TestHandleCampaignResultRequestRunsProtocolSequenceAndRejectsReplays` |

## Visible encounter inventory

Pack boundaries after the first security pad are necessarily approximate:
overhead editing, overlapping pursuit, instant movement, and off-screen actors
make an exact per-pack census impossible. “Mixed” means the footage visibly
contains two or more Zelem/Nomad presentation families; it is not a recovered
noun roll.

### Entry through the first security pad

| ID / timestamp | Activation position and visible population | Reveal, aggro, cadence, delay, and drops | Owner/test | Classification |
| --- | --- | --- | --- | --- |
| E1 `02:29-02:53` | Across the first green bridge: 1 Reparatron, 4 Cannonators, 1 Invincitron. Actors are visible by `02:29`; contact begins about `02:35`. | No spawn flash is visible. The pack waits locally, then closes/fires independently. First Invincitron drops two power pills at about `02:51-02:53`. Exact per-actor cadence cannot be isolated. | P, A, D | **Contradicted (video fallback):** current nearest opening may produce only one 1-4 minion Wanderer clump, not this mixed six; current actors target immediately on creation. **Matching:** independent family action clocks and supported orb drops. **Not observable:** roll probability and exact timing. |
| E2 `02:55-03:13` | Next/north island: 3 Cannonators, 1 Reparatron. | Already present as the hero approaches; short local acquisition. No observed drop. No visible inter-wave banner or forced delay. | P, A, D | **Contradicted (video fallback):** no stable four-actor mixed roster or dormant reveal. **Not observable:** no-drop probability and whether E1/E2 are separate director decisions. |
| E3 `03:27-03:51` | Obelisk-side island behind the first Invincitron: 3 Reparatrons, 2 Cannonators. | Visible before engagement; actors pursue/attack concurrently. One Reparatron drops DNA. The nearby obelisk is not used. | P, A, D, O | **Contradicted (video fallback):** current dynamic pool/count cannot guarantee this visible mixed five and immediately targets created actors. **Matching:** DNA and optional untouched obelisk are supported. |
| E4 `03:57-04:13` | Later pre-pad island: at least 1 Invincitron, 2 Reparatrons, 1 Cannonator, plus another visible Reparatron. | Local acquisition; fight overlaps the next group. No discrete reveal or wave pause is visible. | P, A, D | **Contradicted (video fallback):** visible dormant mixed group versus immediate target injection and current 2-6 Spike/1-4 Wanderer selection. **Not observable:** exact boundary/count. |
| E5 `04:09-04:35` | Final pre-pad group: 1 Invincitron and roughly 7 mixed Reparatrons/Cannonators. Three Gravitic Regulators are visible in this region; exact content supplies three mutually exclusive five-object placement variants across the level. | Ordinary hostiles clear before the pad powers; the placed regulators are not required. One visible regulator drops DNA. Electrical/sphere state appears on the pad at `04:37-04:39`. | P, A, D, S plus authored fixture selection | **Contradicted (video fallback):** current ordinary group limits/composition do not reproduce the observed density. **Matching:** one exact five-regulator authored variant is placed as killable, non-aggro fixtures outside security and encounter clear gates; a live ordinary threat can hold the pad closed. |

The five pre-pad counts are the strongest census in the footage, but remain a
video placeholder. `content.db` proves eligible low-band role pools, not these
five fixed retail rosters or their exact marker assignment.

### First pad through the boss-security platform

| ID / timestamp | Activation position and visible population | Reveal, aggro, cadence, delay, and drops | Owner/test | Classification |
| --- | --- | --- | --- | --- |
| E6 `04:43-05:05` | First destination/landing island: mixed landing guards; an Invincitron label is visible. Exact count is obscured by the arrival effect and overlap. | Pack is present at/just beyond arrival and engages locally. Ordinary loot behavior only; no reliable fixed drop census. | P, A, D, S | **Matching structurally:** teleport landing can feed population contact. **Contradicted (video fallback):** no preserved landing roster/dormancy. |
| E7 `05:11-05:21` | Bridge-edge island: at least 3 small actors visible before the hero closes. | No wave banner or reveal effect; short engagement. | P, A | **Contradicted (video fallback):** current locus roll does not preserve the visible small group and targets on creation. **Not observable:** family/count beyond the visible minimum. |
| E8 `05:23-06:01` | Next broad island: a large mixed pack already visible to the right; about 8 are simultaneously visible near peak overlap. | Actors acquire in proximity and pursue at individual speeds. Equipment is collected during/after this traversal section. | P, A, D | **Contradicted (video fallback):** current 1-4 Wanderer/2-6 Spike fallback cannot explain one visibly dense group unless multiple loci overlap, and it lacks pre-aggro dormancy. **Not observable:** whether several director decisions composed the pack. |
| E9 `06:13-06:27` | Next bridge/island: another small mixed pack, with off-screen entrants preventing a firm count. | No explicit inter-wave delay; ordinary independent attacks. | P, A | **Contradicted (video fallback)** on dormant presentation; **not observable** on exact roster. |
| E10 `06:43-07:17` | Random-unlock bridge/arena: mixed pack; shield bubble visible around `07:03`. The third active-ability slot becomes durable around `06:40-06:50`. | Pack combat overlaps the authored first-clear unlock region; no evidence that the pack itself owns the unlock. | P, A, D, U | **Matching:** proximity population and the recovered paired random-unlock job exist. **Contradicted (video fallback):** immediate spawn targeting. **Not observable:** exact encounter count and server pack/unlock ordering. |
| E11 `07:21-08:03` | Next island: sustained mixed fight; mutation labels and overlapping effects obscure the count. | Independent pursuit/fire; equipment-receipt panel is visible around `07:55`. | P, A, D | **Matching structurally:** current combat/drop owners can produce this. **Contradicted (video fallback):** reveal/dormancy. **Not observable:** exact roster and drop chance. |
| E12 `08:59-09:13` | Elevated platform after the `08:19-08:29` security transfer: at least 5 mixed actors are visible; Reparatron and Space Barracuda labels appear. | Actors are present near the landing/bridge and clear before the next sphere pad is used at `09:15-09:17`. | P, A, D, S | **Matching structurally:** local threats block a security hop and landing contact is integrated. **Contradicted (video fallback):** no exact mixed roster/dormant state. |
| E13 `09:39-10:13` | Destination island: dense mixed group; Space Barracuda is visible around `10:03`. | A shield bubble covers part of the fight. Equipment/DNA/orb outcomes are visible but cannot be assigned as universal noun drops. | P, A, D | **Matching structurally** for independent actions and generic rolls; **contradicted (video fallback)** for reveal/roster; **not observable** for exact cadence. |
| E14 `10:23-10:43` | Next compact island: mixed actors; Decelerator label around `10:37`. | Local pursuit and independent attacks; no wave banner. | P, A | **Matching:** the low-band `NomadSnipe`/Decelerator action family is implemented. **Contradicted (video fallback):** immediate target injection. |
| E15 `11:05-11:23` | Next island/bridge edge: mixed pack; Reparatron label around `11:13`. | The pack is already visible on approach and closes locally. | P, A, D | **Contradicted (video fallback)** for dormant presentation; **not observable** for exact count/composition. |
| E16 `11:41-12:27` | Last long ordinary section before the red boss transfer: dense mixed pack spread over the island; loot remains collectible after the last kill. | Sustained concurrent combat with no visible forced inter-wave pause. The red boss-security pad is reached only after this region clears. | P, A, D, BS | **Contradicted (video fallback):** current generic population does not reproduce the observed staging. **Unimplemented/blocking:** the subsequent boss-security handoff has no normal campaign owner. |

The edited footage shows only three unmistakable sphere transfers:
`04:37-04:43`, `08:19-08:33`, and `09:15-09:23`, followed by the distinct red
boss transfer at `12:45-12:47`. It does not expose enough uninterrupted map
geometry to assign every one of the five current generic source/destination
pairs to a unique frame. Therefore the hardcoded order and pairing of all five
generic hops are **not observable**, even though the visible sphere
presentation and the existence of repeated transfers match structurally.

### Final arena

| ID / timestamp | Activation position and visible population | Reveal, aggro, cadence, delay, and drops | Owner/test | Classification |
| --- | --- | --- | --- | --- |
| E17 `12:47-13:43` | Final arena landing near the center. “Squad Abilities are now unlocked!” appears at `12:51`; the arena remains empty through about `13:03`. First `Horde incoming!` and a Cannonator appear at `13:05`; ordinary adds continue entering through about `13:43`. Decelerator is visible around `13:11`. | The first add appears/reveals after the warning rather than all actors existing from landing. Adds acquire the hero promptly. No Illust health bar is visible in this phase. | U, B, A | **Matching:** six-second first-clear support unlock, delayed arming, and an initial four-add plan that reserves Illust without publishing or admitting him. Exact actor count remains a walkthrough-derived fallback. |
| E18 `13:45-14:27` | Second `Horde incoming!` at `13:45-13:47`; Space Barracuda at `13:50`, Invincitron at `13:51`; Illust the Accelerator bar first clear at about `13:55`, with `Swift Aura, Swift`. | About a 2-second warning-to-first-visible-add gap. Illust and adds attack concurrently; exact per-actor cooldowns are obscured. Final hostile dies about `14:27`; final drops appear before completion UI. | B, A, D | **Matching:** clearing the first four adds immediately publishes the second warning and reserves Illust plus two later adds; the shared two-second scheduler then admits them together, publishes Illust's exact identity and affixes, and requires all admitted actors to clear. **Not observable:** exact retail counts and warning delay. |

## Implementation follow-up

- 2026-08-26: first-clear population replaces the captured upper-island
  Haster-plus-eight-Barracuda cluster with one Haster and three Barracudas,
  fixes the reported Section C group near `(255,667,10)` to one Haster, two
  Barracudas, two Reparatrons, and four Cannonators, and publishes the initial
  final-arena horde phase before its delayed actors.
- 2026-08-25: first-clear setup now uses one stable selection seed for ordinary
  population and obelisks, plus the corrected twenty-two ordered Gravitic Regulator anchors
  captured across one complete introductory traversal. It also fixes the
  reported three-Cannonator cluster at the upper Section B anchors around
  `(580..595,-47..-19,27..31)`, two Space Barracuda groups near
  `(621,-60,25)` and `(664,-35,20)`, two Cannonators in the latter mixed group,
  and two Cannonators near `(187,659,5)`. Unique match IDs
  continue to select the authored five-object regulator and obelisk variants
  on replay runs, so the first route remains fixed without removing later
  variation.
- 2026-07-23: ordered gap 1 is implemented. Normal traversal now owns the
  authored red boss-security source and the content-correlated final-arena
  destination as a documented playability fallback. Its local live-threat
  predicate and route index prevent premature or repeated handoff, and the
  destination is verified against the real 1-1 BFX navigation mesh.
- 2026-07-23: ordered gap 5's encounter order is implemented. The first
  admission contains only the four authored add anchors; clearing them admits
  Illust with the later two-add plan after a shared two-second warning delay
  and publishes the boss identity then. Exact counts and delay remain
  walkthrough-derived fallbacks rather than recovered retail authority.
- 2026-07-23: ordered gap 3 is implemented for ordinary population. Traversal
  admission now creates visible, targetable, out-of-combat actors; a separate
  local acquisition transition assigns the hero and starts first actions once.
  The radius-12 policy remains video-derived because retail perception
  authority is absent.

- 2026-07-23: ordered gap 7 is implemented from exact content rather than the
  walkthrough's partial view. One of three equal-weight authored smart-object
  variants places five Gravitic Regulators. They are stationary, targetable,
  killable, and eligible for ordinary defeat drops, but cannot acquire a
  target or block security, locus, encounter, or boss completion.
- 2026-07-23: ordered gap 8 now replicates the exact six-object barrier set for
  each authored horde. Admission creates three gate teleporters and three
  adjacent blocking doors with authored transforms and flags; scoped
  completion deletes those same IDs while server movement repel remains
  authoritative.

## Cross-cut parity findings

### Activation, reveal, and initial aggro

`campaignPopulationSession` correctly treats 395 ordinary director controls as
candidate loci rather than opening-time enemies. Its 32-unit Wanderer and
25-unit Spike activation radii and 1-4/2-6 fallback counts are not recovered
retail constants. `PrimeOpening` consumes only the nearest Wanderer if entry
already intersects its radius.

Every created ordinary or horde actor is then passed through
`marshalCampaignEnemyTargetedSpawns`, which immediately sets the deployed hero
as target and publishes combat state. The footage repeatedly shows already
visible actors waiting until the hero closes (`02:29-02:35`, `05:11`,
`05:23`, and `11:41`). This is a **contradicted video placeholder**, not an
exact retail aggro rule. Exact content instead supplies important negative
evidence: the four low-band AI definitions author zero aggro and alert ranges,
so remote acquisition requires an external campaign owner.

`ZelemBasicRanged` also has an exact invisible/intangible pre-aggro behavior.
The current server starts its recovered first-action loop but does not use a
pack-level reveal owner to reproduce the traversal presentation. A correct
implementation must separate actor creation/reveal from target insertion.

### Pursuit and attack cadence

The current action owners preserve the recovered family mechanics:

| Exact low-band family | Exact first action and timing | Current parity |
| --- | --- | --- |
| `ZelemBasicRanged` | Blink range 50, hit at 3.2 s; chained shot range 12.5, launch 0.666667 s, release 1.066667 s, cooldown 5 s; blink cooldown 7 s | **Matching** in policy/tests; the video cannot isolate these clocks. |
| `ZelemSpecialHaster` | Ally buff range 30, hit 0.26667 s, release 1.06667 s, cooldown 4 s; shot range 14, hit 0.33 s, release 1 s, cooldown 3 s | **Matching** in policy/tests; **not observable** numerically. |
| `NomadSnipe` | Slow range 12, hit 0.5 s, release 1.3 s, cooldown 4 s; melee range 1.5, hit 0.466667 s, release/cooldown 1.5 s | **Matching** in policy/tests; Decelerator presentation is visible but individual clocks are not. |
| `NomadWithDrone` | Punch range 2, hit 0.515152 s, release 1.666667 s, cooldown 2.4 s | **Matching** in policy/tests; Invincitron presentation is visible but individual clocks are not. |

The server owns live pursuit position and retries against moving targets. The
footage supports independent concurrent pursuit/attacks, but camera cuts,
overlapping effects, target switches, and player kills make a numeric
walkthrough cadence comparison **not observable**. No timing should be changed
to fit apparent video intervals.

### Horde waves, inter-wave delay, and gates

Content exactly identifies horde 1's three listeners and three gates around
`(584-602,4-21,10)`, with a radius-15 once-only entry trigger. Horde 2 has two
listeners and three gates around `(-525..-545,561..569,0)`, with a radius-5
once-only trigger. Current H requires horde 1 completion before horde 2,
spawns 3 actors for horde 1, then 2 followed by 3 for horde 2, and uses a
1.5-second delay before horde 2's second wave.

The video does not expose marker coordinates or uninterrupted gate geometry,
so identifying E8/E12/E13 with a particular authored horde is **not
observable**. The final arena does visibly show about two seconds from the
second warning (`13:45-13:47`) to the first actors (`13:50-13:51`), but that is
not evidence that the ordinary horde-2 fallback must use the same delay.

Current gates retain the server-owned two-unit movement constraint which stops
and teleports the player to the previous accepted position until the scoped
horde completes. At admission the runtime now creates that marker set's exact
three `HordeGateTeleporter` and three adjacent blocking-door objects using
their authored transforms, visibility, and collision flags; scoped completion
deletes the same six stable object IDs. Classification: **matching** for the
authored object set and repel-until-clear lifecycle; **fallback** for
create/delete as the unrecovered retail synchronization contract; **not
observable** for exact gate contact order in the edited footage.

### Security teleporters

Exact chunk-144/client behavior is stronger than the footage:

- radius-2 PhysX sphere entry with enter/stay/leave state;
- a radius-20 query expanded by candidate spatial radius;
- only live, team-0 candidates without
  `InvisibleToSecurityTeleporters` keep the pad inactive;
- active electrical/sphere presentation;
- modifier `0x502F1932` plays teleport-out, waits 0.5 s, teleports, plays
  teleport-in without an intervening wait, and retains the modifier for a
  final 0.5 s.

Native consumers are `Game.c:1430928` `sub_A0AC20` (Lua dynamic sphere),
`:1440617` `sub_A16D00` (trigger creation), `:1452108` `sub_A24F90`
(PhysX enter/leave/stay dispatch), and `:1439761` `sub_A15F00` (Lua callback).
The radius query reaches `sub_A10260` at `Game.c:1434912` and
`sub_9F72C0` at `:1415825`.

Current S matches radius 20 plus enemy footprint and blocks on live local
snapshots, but does not apply exact team/organic/invisibility filtering. It
uses a swept hero segment with a 2.8-unit Blitz compatibility radius. Normal
setup reserves stable collision-free identities for the six level-authored
pads and targets each with the inactive server event; it deliberately does not
create duplicate platform nouns. A clear approach publishes the targeted
power-up and active server events before contact. Normal
movement contact now compiles the recovered modifier for the current route
destination, publishes `ModifierCreated` plus teleport-out immediately,
commits `ObjectTeleport`, teleport-in, authoritative position, and the route
index at 0.5 s, then publishes `ModifierDeleted` at 1 s. A retained transfer
suppresses duplicate contact; `/reset`, disconnect, session replacement, and
scheduler failure cancel it and release its modifier handle without consuming
the route. Approaching a clear pad also publishes the power-up and active
effects before a later contact.

The sixth normal route entry now applies the same lifecycle to the red
boss-security source and its content-correlated final-arena destination.
Classification: **matching** for initial inactive state, targeted activation,
modifier identity, recovered Lua timing, duplicate suppression, and lifecycle
cleanup; **matching structurally** for threat behavior; and **fallback** for
swept contact and route edges. Exact team/organic/invisibility threat
classification remains unimplemented. The source-to-row-125479 destination
edge remains content-correlated rather than exact because normalized
`target_marker_id` is zero.

### Drops

Observed pre-pad outcomes are two power pills from the first Invincitron, DNA
from one Reparatron, DNA from one Gravitic Regulator, and no drops from the other
actors in the first two groups. Later ordinary equipment, health/DNA pickups,
and final-arena drops are visible. Current D independently rolls equipment,
orb, crystal, and DNA once per defeated actor and publishes each before later
completion.

This is **matching structurally**. Fixed per-noun drops and exact probabilities
are **not observable** and are not established by content. The walkthrough
outcomes must not replace the current server-owned random rolls. Gravitic
Regulator defeat now flows through the same server-owned drop transaction.

### Obelisks

The walkthrough does not show a deliberate obelisk use. A red device is
visible near `10:45`, but the player does not stop to activate it. The exact
content provides loot-obelisk range 1.5/hit 0.6 s and health-obelisk range
1/hit 0.4 s, one-use active presentation, loot/crystal or orb outcomes, and the
optional `TouchAllObelisks` objective. Current O selects three loot and two
health obelisks for the initial chain and implements their timelines and
rewards.

Classification: **matching** that obelisks are optional and do not gate 1-1
completion; interaction presentation/reward parity is **not observable** in
this walkthrough.

### Boss transition and completion ordering

The strongest visible order is:

1. red boss-security pad at `12:45`;
2. boss-arena arrival at `12:47`;
3. support-unlock message/icon at `12:51`;
4. empty arming interval through about `13:03`;
5. first horde/add entry at `13:05`;
6. second horde warning at `13:45-13:47`;
7. second-phase adds at `13:50-13:51`;
8. Illust boss bar visible by about `13:55`;
9. final death and world drops at about `14:27`;
10. `Horde defeated!` and `RETURN TO SHIP` available by about `14:31`;
11. player collects drops with the button still available through about
    `14:37`;
12. click/Beam Out around `14:39`, fade about `14:41`, then result/tutorial UI.

The exact client boundary agrees with steps 9-12:
`DirectorState.mbBossComplete` enables `HUD_BeamOut`/`RETURN TO SHIP`;
`MaxisBeamOut.OnBeamOutClicked` sends `PlayerStatusUpdate` status `0x20`,
progress `1.0`, once. Current R requires boss completion, reserves once,
freezes a result snapshot, emits beam presentation, commits the session result,
and enters chain voting. This is **matching** for the observable gating and
one-shot acceptance.

Current B completes only after the leader is dead, both add waves have been
admitted, and all tracked actors are defeated. That conservatively explains
why the button appears after the final hostile and is playability-safe, but
the retail clear predicate is missing authority. It remains **not observable**
whether retail required the same conjunction.

Current results omit or do not fully recover the optional
`ObjectivesComplete` (`0xB9`) and exact `ChainLevelResults` ordering. Those
messages occur after the requested Beam Out boundary and their retail relative
order is **not observable** from the click alone. They do not explain a
pre-Beam-Out soft lock.

## Ordered implementation-ready gap list

1. **P0 — Instantiate the boss-security route in normal campaign play.**
   **Done.**
   Normal traversal owns row `125464` after the final ordinary/horde clear at
   `(184.2345,558.9578,10.2027)`, use the recovered BS active/threat/entry
   lifecycle, and hand off to the final arena. Until the exact edge is
   recovered, explicitly record row `125479`
   `(928.1883,668.9045,0.0880)` as the conservative playability fallback.
   Integration coverage proves normal traversal reaches B without a developer
   event and that a qualifying threat keeps the pad inactive.
2. **P0 — Make every completion prerequisite retry-safe and scoped.**
   **Done.** The last pre-boss threat set cannot retain a defeated,
   uncommitted, invisible, non-organic, or wrong-team actor in the
   boss-security query. Live snapshots prune defeated actors, spawn rollback removes uncommitted
   actors, the typed threat predicate excludes fixtures and scoped invisible
   actors, and a retained transfer suppresses repeated contact until completion
   or cancellation.
3. **P1 — Separate ordinary actor creation/reveal from target insertion.**
   **Done.** P owns a run-local pre-aggro/dormant state and local acquisition
   transition. It preserves exact zero noun aggro/alert ranges and labels the
   chosen acquisition radius as a video-derived fallback. Coverage proves no
   entrance-wide pull and exactly-once first aggro.
4. **P1 — Add a 1-1 traversal population fixture rather than fixed retail
   claims.** **Done for the five pre-pad clusters.** Exact authored Wanderer A
   anchors at `(-147.9,-85.2)`, `(-136.9,-44.5)`, `(-143.8,-11.3)`,
   `(-141.1,38.9)`, and `(-145.0,71.5)` now own provisional count bands
   `6/4/5/5/8`. The overlapping pre-pad Wanderer/Spike candidate cloud is
   replaced by those five one-shot traversal clusters, while later positive-X
   section-A loci remain director-driven. Their deterministic rosters now use
   `ZelemBasicHybrid` Cannonators, `ZelemBasicRepair` Reparatrons, and
   `NomadWithDrone` Invincitrons rather than randomly filling ordinary slots
   with `ZelemBasicRanged` Space Barracudas.
   Coverage proves the final live cluster holds the first pad closed. Later
   route-region fixture correlation remains optional visual parity rather than
   a 1-1 completion blocker.
5. **P1 — Stage the final arena in the visible order.** **Done.** For
   first-clear 1-1, support unlock precedes an add-only phase; a second
   warning/phase introduces Illust and more adds. Exact wave counts and the
   current two-second warning delay remain fallbacks until server evidence is found.
   Test timestamps/order, not wall-clock precision: unlock -> empty arming ->
   adds -> second warning -> Illust -> final clear.
6. **P1 — Publish exact security active state and modifier pacing.** **Done.**
   Generic campaign pads now receive targeted inactive, power-up, and active
   server events and contacts use the recovered modifier runner, its exact
   0.5-second teleport and 1-second release deadlines, duplicate suppression,
   and cancellable lifecycle. The typed enemy-session boundary contains only
   team-0 organic damageable actors and excludes its non-organic fixtures.
   Pre-aggro Zelem Basic Ranged and Nomad Snipe actors retain their presentation
   state but remain authoritative security threats, and pad state evaluation is
   independent of hero proximity. Swept-contact tolerance remains a
   compatibility layer rather than an exact-content claim.
7. **P2 — Implement placed Gravitic Regulators independently.** **Done.**
   First-clear 1-1 uses the twenty-two positions captured across the complete
   introductory route, including the upper-island regulator at
   `(545.755,1.496,33.088)` and the lower Section C placement at
   `(203.116,628.193,5.088)`. Repeat clears select one of the three exact
   equal-weight five-object content variants by match ID. The devices remain
   targetable/killable but excluded from aggro and every security/encounter
   clear roster, and their defeat routes through D. Their DNA outcome remains
   random policy; the single observed drop is not a fixed fixture outcome.
8. **P2 — Synchronize horde gate presentation with H.** **Done.** Publish the
   exact three authored gate teleporters and three blocking doors when a scoped
   horde starts, remove those stable object IDs when that horde completes, and
   retain movement repel as the authoritative compatibility boundary.
9. **P3 — Preserve current exact family cadence and generic drop rolls.** Do
   not tune attack clocks or fixed drops from this video. Add only integration
   coverage showing independently scheduled actors continue pursuing and
   attacking across the new dormant/reveal and wave transitions.
10. **P3 — Keep obelisks optional.** No walkthrough-parity change is warranted.
    Retain the current O tests; add visual parity work only when footage or a
    trace actually shows an activation.
