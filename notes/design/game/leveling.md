# Leveling experience reconstruction

This note reconstructs what a player experienced while leveling in retail
Game and translates that experience into server behavior. It combines the
release manual, contemporary 2010-2011 coverage, surviving community guides,
the extracted build-103 data, and the current darkspin/legacy ReCap code.

Use the evidence labels defined in
[`progression-and-campaign.md`](progression-and-campaign.md). Public sources can
corroborate behavior, but only shipped data or build-103 executable evidence is
treated as local confirmation.

## The progression loop players experienced

The surviving descriptions consistently show this loop:

1. Enter a roughly 10-15 minute planet mission, alone or with up to three
   other players.
2. Kill enemies, open obelisks/caches, collect DNA, equipment, and temporary
   catalysts, then defeat the final horde or boss.
3. Receive medals and Crogenitor XP on the results screen.
4. Either cash out, keeping the run's reward, or immediately continue to the
   next sequential mission at higher risk for better rewards.
5. Return to the ship to unlock heroes, equip loot, spend DNA on account
   upgrades, edit squads, and choose the next planet.

This is not a conventional RPG where one character earns XP. The player's
**Crogenitor level** advances from combat and gates heroes, abilities, upgrades,
and eventually PvP. A hero's own displayed level is derived from equipped gear.
Contemporary hands-on coverage explicitly says heroes neither collect their own
XP nor level independently.

The distinction is required in the server model:

| Concept | Persistent source | What it controls |
| --- | --- | --- |
| Crogenitor level/XP | Account progression | Hero eligibility, upgrade visibility, ability onboarding, PvP, and suggested difficulty |
| Hero level | Equipped combat parts | Hero power and matchmaking/presentation; it must be recalculated when equipment changes |
| Campaign progression | Highest authoritative stage completion | Which map is selectable |
| Chain-run depth | Active game session | Current risk/reward multiplier and cashout offer |
| DNA | Account wallet | Upgrade and equipment economy |
| Hero reward choices | Account entitlement | How many eligible heroes the player may activate |

## Reconstructed first-session cadence

The following is the best first-session timeline supported by public accounts
and build-103 assets. Community-only UI timings need a clean trace before they
become server rules.

| Moment | Player-facing event | Evidence |
| --- | --- | --- |
| Account level 1 | The tutorial starts with Blitz Alpha. | Contemporary GameSpot coverage plus local `unlockLevel=1`. |
| Early simulator | Blitz's special ability and passive become available. | Community guide; needs trace. |
| Account level 2 | Sage Alpha becomes the only newly eligible authored hero. | Local `unlockLevel=2`; community pages call Sage the second hero. |
| Mid tutorial | Q/W switching becomes available for the first two heroes. | Community controls guide; needs trace. |
| Account level 3 / simulator exit | Goliath Alpha, Wraith Alpha, and Zrin Alpha become eligible. Contemporary sessions exited with different third heroes, strongly suggesting a choice rather than a fixed grant. | Local data plus two differing contemporary play accounts. |
| Tutorial exit | The player can switch among a complete three-hero squad. | Contemporary coverage and community controls guide. |
| 1-1 boss room | Variant ability becomes available. | Community controls guide. |
| 1-2 boss room | Hero squad abilities become available. | Community controls guide. |
| 1-3 boss pit | The first catalyst is introduced. | Community catalyst guide. |
| 2-1 or 2-2 boss room | Shared squad abilities and Overdrive finish the combat onboarding. Surviving guides disagree by one stage. | Community sources conflict; trace required. |
| Start/finish 3-1 | Foot equipment begins appearing. | Community editor guide; needs trace. |
| Start/finish 5-1 | Hand equipment begins appearing. | Community editor guide; needs trace. |
| Account level 10 | PvP becomes purchasable/unlocked in the release game. | Release-era reviews and guides. An official-blog beta account reports level 7, so that lower value is pre-release behavior. |

The server should not automatically grant every hero whose unlock level has
been reached. Build 103 has a separate `creature_rewards` balance and an
`unlockCreature` request. The most likely retail flow is:

```text
award Crogenitor XP
-> cross one or more account levels
-> award a hero-choice entitlement when appropriate
-> expose templates with unlockLevel <= account level
-> player selects one eligible, unowned template
-> consume one entitlement and persist the creature
```

Contemporary wording varies between "a new hero at each level" and "hero
choices at certain milestones." The exact entitlement schedule is not yet
known. Eligibility and entitlement must therefore be separate policies.

## Authored hero eligibility schedule

The 100 `PC_*.playerClass.xml` files define 25 heroes with Alpha, Beta, Gamma,
and Delta variants. Their `unlockLevel` values produce this exact schedule:

| Crogenitor level | Newly eligible variants |
| --- | --- |
| 1 | Blitz Alpha |
| 2 | Sage Alpha |
| 3 | Goliath Alpha; Wraith Alpha; Zrin Alpha |
| 4 | Arakna Alpha; Vex Alpha; Viper Alpha |
| 5 | Jinx Alpha; Lumin Alpha; SRS-42 Alpha |
| 8 | Arborus Alpha; Magnos Alpha; Titan Alpha |
| 11 | Krel Alpha; Maldri Alpha; Revenant Alpha |
| 14 | Andromeda Alpha; Meditron Alpha; Savage Alpha |
| 17 | Blitz Beta; Sage Beta; Wraith Beta |
| 18 | Goliath Beta; Viper Beta; Zrin Beta |
| 20 | Seraph-XS Alpha, Beta, Gamma, and Delta |
| 22 | Arakna Beta; Lumin Beta; Vex Beta |
| 23 | Jinx Beta; Magnos Beta; SRS-42 Beta |
| 25 | Tork Alpha, Beta, Gamma, and Delta |
| 27 | Arborus Beta; Maldri Beta; Titan Beta |
| 28 | Krel Beta; Revenant Beta; Savage Beta |
| 30 | Char Alpha, Beta, Gamma, and Delta |
| 32 | Andromeda Beta; Meditron Beta; Sage Gamma |
| 33 | Blitz Gamma; Goliath Gamma; Wraith Gamma |
| 35 | Skar Alpha, Beta, Gamma, and Delta |
| 37 | Arakna Gamma; Viper Gamma; Zrin Gamma |
| 38 | Lumin Gamma; SRS-42 Gamma; Vex Gamma |
| 40 | Orion Alpha, Beta, Gamma, and Delta |
| 41 | Arborus Gamma; Jinx Gamma; Magnos Gamma |
| 42 | Krel Gamma; Maldri Gamma; Titan Gamma |
| 43 | Andromeda Gamma; Meditron Gamma; Savage Gamma |
| 44 | Blitz Delta; Revenant Gamma; Sage Delta |
| 45 | Goliath Delta; Wraith Delta; Zrin Delta |
| 46 | Arakna Delta; Vex Delta; Viper Delta |
| 47 | Jinx Delta; Lumin Delta; SRS-42 Delta |
| 48 | Arborus Delta; Magnos Delta; Titan Delta |
| 49 | Krel Delta; Maldri Delta; Revenant Delta |
| 50 | Andromeda Delta; Meditron Delta; Savage Delta |

There are intentional gaps. For example, no template first becomes eligible at
levels 6 or 7. A generic "one newly available hero per level" implementation
would therefore be wrong even if the account receives a choice token each
level.

The five special heroes are unusual: all four variants become eligible
together at levels 20, 25, 30, 35, and 40. Availability may have had another
restriction in retail; `unlockLevel` proves level eligibility, not ownership or
entitlement.

## Squads and account upgrades

The release manual says a player starts with **one squad** and buys additional
squad slots with DNA. Each squad contains three heroes. The account protocol can
still carry three squad records from day one, but locked records must not be
treated as usable.

The legacy upgrade table aligns closely with that description:

| Capability | Initial state | Candidate purchases | Confidence |
| --- | --- | --- | --- |
| PvE squads | 1 usable squad | Squad 2: 500 DNA; squad 3: 4,000 DNA | Manual corroborates behavior; costs are legacy-reference values. |
| Catalyst grid | 3 usable slots | Slots 4-9: 2,000; 6,000; 16,000; 30,000; 80,000; 150,000 DNA | Community guide and legacy table agree on 3-9 shape; costs need build-103 verification. |
| Chain capacity | 2 consecutive missions | Candidate capacities 3, 4, and 5: 400; 1,200; 4,000 DNA | Manual confirms a purchase is required for more than two. Mapping from `unlock_fuel_tanks` is strongly inferred. |
| PvP | Locked until level 10 | Legacy upgrade ID 38 has zero DNA cost | Level gate is corroborated; purchase request/cost still needs a trace. |

`NewUser` currently creates three unlocked-looking squad records. The durable
model should retain all three records for protocol compatibility while business
logic derives `is_locked` from the purchased squad capacity.

### PvP slot namespace

Build 103 presents PvE and PvP squads through separate `SetPvESquads` and `SetPvPSquads` arrays. Their slot numbers are therefore destination-local rather than one shared sequence. A 0.7.25 report captured two access violations during ship squad selection after login; both sessions reached account and inventory loading but never created gameplay, while the reporter observed an undefined squad number. Darkspin's profile repair had promoted an unused PvE record into the sole PvP deck without changing its original slot, allowing PvP slot 2 or 3 even though `unlock_pvp_decks=1` exposes only PvP slot 1.

Profile-start repair now migrates the selected or fallback PvP deck to destination-local slot 1, and new PvP provisioning does the same. Deck updates select and reorder records only inside the requested `pve` or `pvp` category, so overlapping destination-local slot numbers cannot redirect a PvP save into the PvE squad with the same slot. IDs and creature memberships are preserved. The report contains no crash instruction pointer or response body, so production client confirmation remains outstanding.

## Chain leveling and rewards

The official manual gives the clearest surviving description of a chain:

- after a mission, leave with earned experience and loot or continue directly
  to the next mission;
- longer chains award more medals, XP, and loot and improve the chance of rare
  and purified items;
- failing later in a chain loses bonus equipment already accumulated for the
  cashout;
- cashing out displays medals, XP, and current account level;
- a new player can chain at most two missions until purchasing an upgrade.

Contemporary coverage adds that continuing prevents the player from equipping
newly found gear before the next stage and increases difficulty. Temporary
catalysts persist throughout the chain.

The legacy gameplay implementation clamps active chain progress to five and
the legacy account table has three `unlock_fuel_tanks` upgrades, naturally
producing capacities 2 -> 3 -> 4 -> 5. One contemporary review instead says
the final capacity became unlimited. Until a build-103 trace resolves that
conflict, five is the safer compatibility cap and should remain data-driven.

The server must keep three reward ledgers during a run:

- **durable account XP**, established before the active mission and increased
  only at an accepted persistence boundary;
- **provisional current-mission XP**, published to the live HUD after each kill
  but not thereby proven durable;
- **at-risk cashout rewards**, whose improved rarity/value is forfeited if the
  chain fails.

An in-mission `LabsPlayerUpdate` proves presentation, not a database commit.
Exactly when retail XP and DNA become durable is still unknown. The manual's
wording says a completed mission earns experience and specifically identifies
bonus equipment as lost on a later chain failure, but it does not explicitly
describe XP after death or abort. Until a trace settles the rule, commit a
completed mission's base kill XP, discard the active mission's provisional XP
on death or abort, retain base XP from earlier completed chain missions, and
forfeit only the uncommitted chain bonus with the at-risk equipment.

## XP values and pacing

No trustworthy public source found in this research gives the numeric XP
thresholds or per-enemy/per-mission award formula. Public accounts do give two
pacing anchors:

- a beta player reached Crogenitor level 7 in about three hours;
- early missions were commonly described as 10-15 minutes each.

Build 103's runtime tuning property `0xC0B32F0F` contains 99 cumulative upper
bounds. An observation-only launch-DLL hook captured the vector directly from
the initialized client. Its opening values agree with the older C++ ReCap
table:

| Level | Cumulative XP upper bound |
| --- | ---: |
| 0 | 0 |
| 1 | 100 |
| 2 | 200 |
| 3 | 3,000 |
| 4 | 6,000 |
| 5 | 9,000 |
| 6 | 12,000 |
| 7 | 15,000 |
| 8 | 18,000 |
| 9 | 21,000 |
| 10 | 24,500 |
| 20 | 73,000 |
| 50 | 412,000 |
| 99 | 2,410,500 |

The build-103 mapper starts at level 1 and advances while cumulative XP is
strictly greater than the current bound. Level 2 therefore starts at XP 101,
level 3 at XP 201, and level 100 at XP 2,410,501 (the captured final bound is
2,410,500). The runtime capture is authoritative for build 103; modern RakNet
or later-client tables must not replace it.

The XP award formula is also unknown. Build-103 assets and public guides imply
that enemy kills, mission completion, optional medals, chain depth, difficulty,
and an equipment modifier such as `+10% Player XP Gained` may all participate.
Those inputs should be represented explicitly instead of hidden in a single
HTTP handler constant.

Build 103 initializes ordinary simulation objects with their `worth XP` flag
enabled. Packaged Lua calls `MarkNotWorthXP` only from Shadow Boss Duplicate,
where the created decoys are also marked unable to drop loot. This proves that
ordinary combatant NPCs are eligible by default and that explicitly created
decoys can opt out; it does not prove that fixtures, self-destructing actors,
summons, or every other non-player object should grant a kill reward. The flag
contains no award amount. The 1-1 result message likewise carries only each
player's starting and final cumulative XP totals, leaving per-rank amounts and
multiplayer ownership to the original server.

As of 2026-08-12, darkspin imports the positive `challengeValue`, derives
eligible spawn-plan XP, and records the first valid live-to-dead transition in
a zone-owned mission ledger. The client receives the resulting provisional
cumulative XP and level immediately. Successful Beam Out persists that mission
ledger once and projects its starting/final totals into campaign results. The
shared account operation uses the complete build-103 threshold vector for both
normal awards and the `/level` developer command.

### Walkthrough and class-data reward reconstruction

The surviving recordings and packaged `NonPlayerClass` data support a useful
first server policy even though no original server formula survives. The class
property decoded as `challengeValue` is the best recovered per-NPC reward
input. Tutorial enemies provide the calibration:

| NPC/class evidence | Authored `challengeValue` | Recorded XP evidence | `2 * challengeValue` |
| --- | ---: | ---: | ---: |
| `TutorialBasicPoison` | 5 | Sixteen pre-obelisk awards total about 167; individual bar-derived estimates range from 8 to 14 | 10 |
| `TutorialSpecialOne` (Quadra) | 30 | About 54 | 60 |
| Space Barracuda class | 6 | No isolated recording | 12 |
| Decelerator class | 27 | No isolated recording | 54 |
| Invincitron class | 30 | No isolated recording | 60 |
| Haster class | 30 | No isolated recording | 60 |
| Illust captain class | 75 | No isolated recording | 150 |
| RED-D-TOR class | 80 | No isolated recording | 160 |

The tutorial values were reconstructed from pixels in the live XP bar, not
from an exact packet trace. Their spread must not be interpreted as proven
randomness: sixteen nominal 10-XP enemies would award 160, only seven below the
estimated 167 total, and the nominal 60-XP Quadra is only six above the
estimated 54. The much larger authored captain and boss values also mean that
the class property already expresses rank. Applying an additional elite,
captain, or boss multiplier would count rank twice.

The tutorial also records cumulative XP updates during the mission through
`0xA1 LabsPlayerUpdate`, and the account reaches level 3 immediately after the
Quadra award. In campaign 6-4 the account reaches level 19 on the final boss's
death before extraction. XP must therefore be awarded and published on the
live death path; the result screen is not the first presentation boundary.
Neither recording observes the account after a death, abort, disconnect, or
re-login, so neither proves that each displayed kill award was already saved.

The campaign result screen carries `starting cumulative XP` and `final
cumulative XP`, and renders their difference as `XP EARNED`. The following
values are visually recoverable from the saved walkthrough frames. They are
calibration totals, not stage constants:

The source ledger is `notes/tutorial/overview.md` for isolated tutorial kills,
`notes/campaign/1-1/rewards.md` for result-message semantics, and each
`notes/campaign/<stage>/info.md` plus its frame directory under
`bin/video/walkthrough` for the campaign totals.

| Stage | Recorded XP earned | Stage | Recorded XP earned |
| --- | ---: | --- | ---: |
| 1-1 | not preserved | 4-1 | 1,618 |
| 1-2 | not visible | 4-2 | 1,180 |
| 1-3 | 1,427 | 4-3 | 1,881 |
| 1-4 | not recorded | 4-4 | 1,538 |
| 2-1 | not visible | 5-1 | result not sampled |
| 2-2 | 1,515 | 5-2 | 1,715 |
| 2-3 | 1,617 | 5-3 | 1,237 |
| 2-4 | 1,910 | 5-4 | 2,006 |
| 3-1 | 1,233 | 6-1 | 1,330 |
| 3-2 | 1,576 | 6-2 | 1,401 |
| 3-3 | 1,698 | 6-3 | no result frames |
| 3-4 | 2,419 | 6-4 | 2,532 |

The observed 1,180-2,532 range does not increase monotonically with stage
number. It is consistent with a sum of encountered enemy rewards affected by
route, skipped packs, encounter composition, and bosses, rather than a fixed
mission-completion award. The recordings do not support hard-coding these
totals into the stage definitions.

### Proposed XP policy

Use the following policy as the prototype implementation. Every numeric guess
is deliberately a data-owned tuning input so later packet or video evidence
can correct it without replacing the progression flow.

1. Read the killed NPC's positive `NonPlayerClass.challengeValue` and award
   `2 * challengeValue` base XP. Preserve the tutorial's measured override
   ledger until its class data follows the same import path.
2. Award exactly once on the first eligible live-to-dead transition. Require
   the simulation object's `worth XP` flag and reject fixtures, scripted
   decoys, player-owned summons, and actors that were never live combatants.
   Revival or repeated death notification must not award again.
3. Credit every eligible party member the full award. Do not divide an integer
   reward among players: no surviving evidence establishes contribution
   splitting, while division would make cooperative progression worse than
   solo progression. Keep this as a replaceable ownership policy.
4. Publish `durable account XP + provisional current-mission XP` and its derived
   level through the normal live player-update path. Keep a mission ledger of
   each accepted NPC award so retries and result generation cannot duplicate
   it. A presentation update must not write durable account state by itself.
5. Start with zero flat mission-completion XP, zero medal XP, and zero daily-
   bonus XP. Completion, medals, rarity, and the daily marker have distinct
   result fields, but the recordings do not prove that any of them adds XP.
6. Apply the equipment `Player XP Gained` percentage to eligible kill XP. For
   the first implementation, calculate `roundHalfUp(baseXP * (1 + bonus))` per
   kill. Both the original rounding boundary and additive-versus-multiplicative
   stacking remain unknown, so retain the unmodified base in the ledger.
7. The manual explicitly promises more XP for longer chains but supplies no
   number. Use a conservative provisional cashout bonus of 10% of the chain's
   modified kill XP for each completed mission after the first, capped at 40%
   at chain depth five. Award only the incremental bonus at mission completion
   or cashout so live kill updates remain attributable to enemies. Commit the
   successful mission's modified kill XP before offering Continue or Cash Out.
   On death or abort, discard only the active mission ledger; on a later chain
   failure, preserve kill XP committed by earlier successful missions and
   discard the failed mission ledger and uncommitted chain bonus.

For a completed chain with mission ledgers `M`, the proposed calculation is:

```text
killBaseXP       = 2 * challengeValue
killModifiedXP   = roundHalfUp(killBaseXP * (1 + playerXPGained))
chainKillXP      = sum(killModifiedXP for every accepted kill in M)
chainBonusRate   = min(0.10 * (completedMissionCount - 1), 0.40)
chainBonusXP     = roundHalfUp(chainKillXP * chainBonusRate)
chainTotalXP     = chainKillXP + chainBonusXP
```

The chain bonus is the least certain numeric part of this proposal. The 10%
step is a playability guess anchored only by the manual's direction of change;
it is not a recovered retail constant. Store the step and cap in campaign
tuning rather than embedding them in transport or account code.

The completion/failure durability rule is also a conservative reconstruction,
not retail proof. It reconciles the visible per-kill HUD and mid-mission level
dings with the separate server-authored result totals: the client may display
a predicted cumulative total throughout play while the server commits it only
after accepting mission success. Tutorial code may choose stronger immediate
persistence for onboarding safety, but that implementation choice must not be
treated as evidence that campaign aborts paid retail XP.

### Emergency campaign fallback values

The implementation should import the authored class value, not assign one XP
number to every enemy on a planet. If an otherwise valid campaign combatant is
temporarily missing class data, the following one-based `campaign-part`
coordinate supplies a deterministic scaffold: fallback challenge is
`10 * campaign + part`, and fallback base XP is twice that value. This table is
intentionally subordinate to every positive authored `challengeValue`.

| Stage | Fallback challenge | Ordinary base XP | Stage | Fallback challenge | Ordinary base XP |
| --- | ---: | ---: | --- | ---: | ---: |
| 1-1 | 11 | 22 | 4-1 | 41 | 82 |
| 1-2 | 12 | 24 | 4-2 | 42 | 84 |
| 1-3 | 13 | 26 | 4-3 | 43 | 86 |
| 1-4 | 14 | 28 | 4-4 | 44 | 88 |
| 2-1 | 21 | 42 | 5-1 | 51 | 102 |
| 2-2 | 22 | 44 | 5-2 | 52 | 104 |
| 2-3 | 23 | 46 | 5-3 | 53 | 106 |
| 2-4 | 24 | 48 | 5-4 | 54 | 108 |
| 3-1 | 31 | 62 | 6-1 | 61 | 122 |
| 3-2 | 32 | 64 | 6-2 | 62 | 124 |
| 3-3 | 33 | 66 | 6-3 | 63 | 126 |
| 3-4 | 34 | 68 | 6-4 | 64 | 128 |

Do not use the fallback for an authored zero without first classifying the
actor. Zero may intentionally describe a fixture or non-reward actor. A
captain or boss with a positive authored challenge receives exactly twice that
authored value and no additional rank multiplier.

| Reward component | Prototype value | Confidence and reason |
| --- | --- | --- |
| Eligible NPC base XP | `2 * challengeValue` | Medium-high: tutorial bar estimates closely calibrate two authored classes. |
| Flat mission completion XP | 0 | Medium: level changes occur on kills and result totals behave like encounter sums. |
| Medal XP | 0 | Medium: medals and XP are independently represented; no numeric coupling is visible. |
| Chain XP | +10% per extra completed mission, max +40% | Low: the manual proves a positive chain effect, but not its magnitude. |
| Player XP equipment modifier | Apply displayed percentage per kill | Medium-low: the stat exists, but its stacking and rounding order are server-only. |
| Daily XP bonus | 0 | Low: a daily marker is visible, but its XP effect is not isolated. |
| Cooperative ownership | Full award to each eligible player | Low: conservative playability policy pending a multiplayer trace. |

An implementation pass should import `challengeValue` into the runtime NPC or
spawn definition, populate `SpawnPlan.Experience` from the policy, award it on
the authoritative death transition, emit the cumulative `0xA1` update, and
project the same ledger into the result message's starting/final totals. It
should then compare generated mission totals against the eighteen recovered
walkthrough values above. Tune the shared class multiplier or explicit bonus
knobs if the same routes systematically miss; never tune a stage constant to
force one recording to match.

## Replay and difficulty after leveling

The campaign has 24 primary stages and additional difficulty passes. The
release manual says higher difficulties become available only after the first
playthrough. The Maxis AI Director did not regenerate map geometry; it selected
different enemy encounters and placements. Maxis described roughly 16 possible
enemy types per planet with six selected for a mission, while a later EA launch
interview describes six selected from a broader 96-enemy pool. The per-planet
asset data should decide the actual candidate set.

This means replay progression requires server-owned mission generation state:
difficulty pass, seed, selected encounters, chain depth, party scaling, medal
objectives, and reward rolls. Returning only a static map name will not recreate
the retail loop.

## Minimum server domain flow

The first useful implementation should expose business operations resembling:

```text
StartMission(account, squad, stage, difficulty)
RecordMissionOutcome(session, objectives, deaths, elapsedTime)
CompleteMission(session)
ContinueChain(session)
CashOutChain(session)
FailChain(session)
UnlockHero(account, templateNoun)
PurchaseUpgrade(account, upgradeID)
EquipPart(account, creatureID, partID, slot)
```

`CompleteMission` should calculate a typed result containing XP, DNA, medals,
drop loot, cashout odds, campaign advancement, and crossed account levels.
Level crossing should produce explicit domain events; transports should only
marshal the resulting allowlisted DTOs.

At minimum, the durable transaction for a successful cashout must atomically
persist account XP/level, DNA, campaign progression, reward entitlements,
inventory additions, and upgrade/unlock changes. The active mission and chain
state belongs to the game session until committed.

## Trace targets that remain decisive

For a clean level-1 account, capture account and inventory snapshots before and
after each of these boundaries:

1. Blitz simulator start and completion;
2. the transitions to account levels 2 and 3;
3. the first hero-choice screen and `unlockCreature` request;
4. tutorial exit and the first three-hero squad record;
5. 1-1 completion and first cashout/continue decision;
6. 1-3 first catalyst;
7. a failed two-stage chain versus a successful cashout;
8. purchase of chain capacity 3 and squad 2;
9. account level 10 and the PvP unlock request.

The trace should record the full before/after values for `xp`, `level`, `dna`,
`creature_rewards`, every `unlock_*` field, `chain_progression`, current chain
depth, inventory IDs, and creature/squad membership.

## Public archive sources

Release/official material:

- [EA Game manual, German edition](https://eaassets-a.akamaihd.net/eahelp/manuals/game-manual-german_PC.pdf)
- [EA Game manual, Hungarian edition](https://eaassets-a.akamaihd.net/eahelp/manuals/game-manual-hungarian_PC.pdf)
- [EA interview with Game senior systems designer](https://www.ea.com/news/game-beams-into-stores-today)
- [Archived Maxis blog: The AI Behind the Curtain](https://game-blog.tumblr.com/post/3567316178/maxis-blog-the-ai-behind-the-curtain)
- [Archived official Game blog, page 2](https://game-blog.tumblr.com/page/2)

Contemporary coverage:

- [GameSpot: Updated Hands-On - Band of Heroes](https://www.gamespot.com/articles/game-updated-hands-on-band-of-heroes/1100-6298797/)
- [GameSpot: Game review](https://www.gamespot.com/reviews/game-review/1900-6310899/)
- [GameSpot: Review in Progress](https://www.gamespot.com/articles/review-in-progress-game/1100-6310415/)
- [GameSpot: risk/reward campaign preview](https://www.gamespot.com/articles/game-germinating-at-comic-con/1100-6270903/)
- [Altered Gamer: release PvP level-10 guide](https://www.alteredgamer.com/other-rpg-games/118624-how-to-unlock-pvp-mode-in-game/)

Surviving community documentation:

- [StrategyWiki: controls and ability onboarding](https://strategywiki.org/wiki/Game/Controls)
- [Game Wiki: catalysts](https://gamegame.fandom.com/wiki/Catalysts)
- [Game Wiki: hero editor](https://gamegame.fandom.com/wiki/Hero_Editor)
- [Game Wiki: loot and loot types](https://gamegame.fandom.com/wiki/Loot_%26_Loot_Types)
- [Game Wiki: Goliath strategy](https://gamegame.fandom.com/wiki/Hero_Strategy%3A_Goliath)

Primary local evidence:

- [`playerclass/`](../../bin/server/data/playerclass)
- [`user_features.go`](../../server/sporenet/user_features.go)
- [`progression-and-campaign.md`](progression-and-campaign.md)
- [`game-5.3.0.103-progression.md`](../confirmed/game-5.3.0.103-progression.md)
