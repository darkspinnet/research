# Cryos first-run route ownership

This note records which authored event owns each late phase of the build-103
Cryos first-run route. It deliberately distinguishes content linkage, packaged
Lua behavior, client-native behavior, live observation, and inferred ordering.
An event name proves a dispatch boundary; it does not prove the missing
server-side implementation behind that boundary.

## Result

| Route phase | Best-supported owner | Ownership status |
| --- | --- | --- |
| Sage unlock | Server-only marker `3912233898` -> `nTutorial_IntroSecondCreatureUnlock.main` | Content-linked and bytecode-proven. The job waits `0.5s`, unlocks the second mission creature for each player, notifies `PlayerUnlockedSecondCreature`, and deletes itself. |
| Quadra introduction | AI placement `TutorialSpecialOne_Intro` plus its `FirstAggro_SpecialOne` ability | The placement and ability timeline are proven, but the director edge that activates the ability after Sage unlock is not recovered. |
| Quadra fight / teleporter guards | Five fixed AI placements clustered within chunk `144`'s server-owned `BossSecurityTeleporterPassive` | Content/bytecode-linked spatial set: one Special One, two Poison, and two Ranged placements are `6.69-7.99` units from the platform; every other fixed AI is at least `88.11` units away. Chunk `659` proves the organic filter is creature GUID `0x06c27d00`; chunk `865` owns attribute `112` exclusion. Native deployment copies noun byte `+224` to runtime team `+84`, and all three nouns specify `0`. The behavior-tree deactivation edge and a live five-guard clear remain validation gaps. |
| Enter horde arena | `BossSecurityTeleporter` marker `495984414` targets marker `174193625`; arrival enters boss-director marker `1730050752` | Content-linked. Teleport use and the director entry are distinct boundaries. |
| Horde waves | `SpawnPoint_DirectorBoss` callback `DirectorTrigger_SpawnBoss`, publishing `horde triggered02` to exactly two `HordeSpawner_Register` listeners | Content-link-proven server ownership. The build-103 client callbacks do not implement the waves, and no packaged Lua chunk implements either callback. |
| Horde clear | Terminal state of that server-owned horde director, followed by the `Horde defeated!` client alert | The alert is presentation, not completion authority. Four waves are strongly supported by `waveOverride=4` and live behavior, but the retail budget, clear predicate, and final server callback remain unrecovered. |
| Beam Out offer | Director `mbBossComplete` reflection -> `nGameDirector.IsBossDead` -> `HUD_BeamOut.swf` / `SP_UI/cBeamOut` | Native-proven client gate and live-proven packet shape: `8B 08 01`. This owns the success prompt; `TutorialGameMsgs 0xC8` does not. |
| Completion request | `MaxisBeamOut.OnBeamOutClicked` -> `PlayerStatusUpdate(status=0x20, progress=1)` | Live-observed client request. Retail result/return routing after the request is still incomplete. |
| Completion snapshot | Server-authored `TutorialGameMsgs 0xC8`, subtype `0`, positive cumulative XP | Native-proven receiver semantics. It updates the client snapshot to progress `3000`; it does not perform a server/account save. |
| Completion persistence | A server transaction at an accepted terminal return boundary | Required server ownership, but the retail event and ordering are not proven. darkspin currently persists at accepted Beam Out/remove-player handling; that is reconstruction policy, not recovered retail authorship. |
| Squad death | Failure presentation `HUD_Death.swf`; local `LABS_ALL_PCS_DIED` also feeds survey classification `0` | The UI and survey event are native/content-proven, but neither is proven to be the authoritative game-over director. The squad-death predicate and server message owner are unknown. |
| Restart | No recovered Cryos marker, director entry, or packaged Lua callback | Unresolved. A clean-session rebuild is inferred as required; the request, acknowledgement, teardown, and re-entry events have not been captured. |

The important negative result is that there is no single packaged "Cryos
tutorial script." The route crosses several authority domains: server-only
tutorial jobs, AI first-aggro abilities, level marker event fan-out, client HUD
gates, and server persistence.

## Evidence extracted from `content.db`

Queries used `bin/server/darkrun.exe db ... --config
bin/darkspinner/darkspin.toml`. Binary fields were extracted with `darkrun db
bget`, not copied from the loose historical Lua corpus.

- `level.id=9` is `Game_Tutorial_cryos_1`; aliases include the client name
  `Game_Tutorial_cryos_1_v2`. The level has 16 authored marker-set layers.
- `level_marker_set.id=283` is the design layer and `id=291` is the AI layer.
- `level_director_entry get level_id=9` returns six normalized rows: ordinary
  `TutorialBasicPoison`, followed by the first-visit run of Diseased, Ranged,
  Poison, Sloth, and Special One. All are difficulty `1..100` and horde-legal.
  The structural configuration boundary is preserved by the retained payload;
  its semantic `config_kind` and `spawn_kind` remain unresolved.
- `level_script get level_id=9` resolves only obelisk callbacks. The tutorial
  jobs are stored as split module and callback strings, so this index gap is
  not evidence that the jobs are unused.
- `lua_chunk.id=215`, resource `13753`, is the second-creature job; chunk
  `619`, resource `14183`, is the client abilities lesson; chunk `502`,
  resource `14061`, is `Abilities/0xC57828E2.lua`, the
  `FirstAggro_SpecialOne` specialization.

Focused retained artifacts are under `bin/game/logs/cryos-route`:

| Artifact | Source | Decoded SHA-256 |
| --- | --- | --- |
| `tutorial-level.bin` | `level.source_payload`, `id=9` | `f97e3491d780e20514f0703e83cc5e6da5ebda0f07831c0eefaad2afab7ba10a` |
| `design.markerset` | `level_marker_set.source_payload`, `id=283` | `651b17394e29be66a3e05c64649ff648e3274d9c9293b9008edd1f1c2ba95376` |
| `ai.markerset` | `level_marker_set.source_payload`, `id=291` | `5067ab36c70cf7aa726d755f39eccf9c7bcdec5f54f6d3eca289e5023cc6698f` |
| `intro-second-creature.luac` | `server_data.decoded_payload`, resource `13753` | `e98b1e855e64904689451bd79635c3487a7a20e29f1b943885c9648b61f8e30a` |
| `intro-abilities.luac` | `server_data.decoded_payload`, resource `14183` | `8274e9d35a88c2697ca53c0f8583107cd58c7390f533fa4326302825023f7985` |
| `first-aggro-special-one.luac` | `server_data.decoded_payload`, resource `14061` | `da18208fc597c67ce3251de30d12ad694ae4d644af0e1cb79b5383cc12ec6d8d` |

The decoded level payload names one ordinary `TutorialBasicPoison` director
choice and five `firstTimeConfig` choices: Diseased, Ranged, Poison, Sloth,
and Special One. All six decoded records allow difficulty `1..100` and are
horde-legal. This proves the eligible first-run noun pool, not the number or
order chosen for a wave.

## Content-link-proven ownership

### Sage and Quadra content

The server-only Sage trigger is marker `3912233898` at approximately
`(259.346, 81.391, 25.088)`, radius `25`, callback
`nTutorial_IntroSecondCreatureUnlock.main`. It is effectively colocated with
the AI marker `TutorialSpecialOne_Intro` and the resistance/Sage audio cluster.
The AI layer contains exactly one Intro Special One, two ordinary Special One
placements, six ranged, three diseased, four orb-dropping poison, and eleven
no-orb poison placements.

This proves the two authored ingredients of the beat, but not a callback edge
from the Sage marker to Quadra's create/focus/first-aggro activation. The Intro
AI marker has no normalized level event of its own.

### Teleporter and horde fan-out

The design payload stores these linked objects:

- `BossSecurityTeleporter` marker `495984414`, at approximately
  `(260.833, 232.784, 20.168)`, targets teleporter destination marker
  `174193625` at approximately `(-347.576, -224.608, 10.088)`.
- `SpawnPoint_DirectorBoss` marker `1730050752` has the authored callback
  `DirectorTrigger_SpawnBoss` and publishes `horde triggered02`.
- Exactly two `SpawnPoint_DirectorHorde` markers register
  `HordeSpawner_Register` for the same event, at approximately
  `(-355.269, -207.711, 10.088)` and `(-345.895, -239.547, 10.166)`.
- The boss-director component carries radius `30` and `waveOverride=4`.

The destination is inside the director volume, so teleporter acceptance leads
to director entry. The callback/listener linkage makes the server-side
`DirectorTrigger_SpawnBoss` event the horde owner. The two listener records
own spawn loci, not independently advancing wave counters.

The level binds no tutorial-specific completion or exit callback. The normal
Cryos `LevelExitPoint.Noun` belongs to a marker layer not referenced by this
tutorial level and therefore cannot own Cryos tutorial completion.

## Bytecode-proven behavior

### `nTutorial_IntroSecondCreatureUnlock.main`

Chunk `215` proves this exact job order:

1. Wait `0.5s` on the job object.
2. Call `nPlayer.GetPlayerIds()` and iterate all returned players.
3. Call `nPlayer.UnlockSecondCreature(playerID)` for each player.
4. Build an event with
   `clientEventID = nUtil.SPID("PlayerUnlockedSecondCreature")` and call
   `nEvent.Notify`.
5. Mark the job object for deletion.

This job owns mission-local Sage availability and its notification. It does
not create Quadra or persist creature ownership.

### `FirstAggro_SpecialOne`

Chunk `502` and its shared first-aggro template prove Quadra's special
presentation after that ability is activated:

| Offset | Authored action |
| ---: | --- |
| `0` | Start a `4.291667s` cinematic centered on the agent with radius `100`; hide the agent. |
| `2.0s` | Show the agent; the shared template repeats the visible state and plays `character_teleport_in`. |
| `3.291667s` | The `1.291667s` animation wait ends. |
| `4.291667s` | The final `1s` delay ends and the ability deactivates. |

The ability owns visibility, cinematic, and beam-in timing. It does not own
the Sage unlock, Quadra object creation, camera target selection before
activation, combat activation, or route progression after Quadra dies.

### `BossSecurityTeleporterPassive` and `TeleporterModifier`

Chunk `144` is content-linked through
`BossSecurityTeleporter.Noun` -> `BossSecurityTeleporter.AIDefinition` ->
`BossSecurityTeleporterPassive`. It owns the radius-20 security scan and the
inactive/power-down/power-up/active effect state machine. An entrant is accepted
only while active and must be player-controlled or owned by a player-controlled
object. It requests GUID `0x502f1932` with the noun-linked destination.

Native registration plus corpus hashing proves that GUID is
`TeleporterModifier`, chunk `349`, not `HordeGateTeleporter`. Chunk `349` owns
the `character_teleport_out` -> `0.5s` -> authoritative teleport -> `0.5s` ->
`character_teleport_in` -> `0.5s` sequence. The modifier's empty deactivate
does not remove its immobilization attribute, so cancellation cleanup remains
an explicit server/session responsibility.

Direct decoding of retained `ai.markerset` records with their actual `0xc8`
stride identifies the only fixed placements inside the radius: Special One
ordinal `0`, Poison ordinals `1` and `14`, and Ranged ordinals `15` and `21`.
The normalized content rows currently use an incorrect `0xd0` stride after the
first record and are not coordinate evidence. All three extracted noun
payloads begin with creature type GUID `0x06c27d00`. Recovered
`GlobalDefinitions` chunk `659` defines the passive's organic table as exactly
that type. All three AI definitions name `nBehavior_Invisible`, and recovered
behavior chunk `865` adds `InvisibleToSecurityTeleporters=1` while active,
stores its modifier handle in thread-data slot `2`, waits forever, and removes
the modifier on deactivation. The type and exclusion mechanisms are therefore
proven. Native deployment copies noun byte `+224` into runtime team byte
`+84`; all three focused nouns specify `0`, resolving the team predicate.
Each AI has one linked phase only (Ranged `TutorialPlasmaLightning`, Poison
`TutorialPoisonMelee`, Special One `BurstShot`), so no alternate authored
phase owns invisibility removal. The exact behavior-tree transition from
invisible/beam-in to countable combat state remains open, as does a live
five-guard clear.

### Packaged horde negative proof

Neither `DirectorTrigger_SpawnBoss` nor `HordeSpawner_Register` occurs in any
of the 1,029 indexed Lua chunks. `ActivateHordeSpawn`, seen in some later
tutorial unlock jobs, is not the Cryos owner: its build-103 native ignores the
second argument and reaches a literal no-op callee. Horde semantics must come
from the server implementation behind the design marker event.

## Native-proven behavior

- `UnlockSecondCreature` is registered to `sub_A050C0`. It resolves a
  simulation player and increments the mission field at `+0x1378`, capped at
  `2`. It has no account submission path, so it proves mission availability,
  not persistent Sage ownership.
- Build-103 registers `DirectorTrigger_SpawnBoss -> sub_9FACF0` and
  `HordeTrigger_OnEnterPlayer -> sub_9FACB0`. Both reject the non-authoritative
  branch, validate the player entrant, obtain the simulator/director, and then
  return false without mutation or GMS. These are intentionally inert client
  registrations. `HordeSpawner_Register` is absent from the executable.
- `IsHordeActive` reads reflected director byte `mbHordeSpawned` at `+0x48c`;
  `mActiveHordeWaves` begins at `+0x47c`. The client executable contains only
  initialization and getter accesses, not a gameplay writer. These fields do
  not reveal the server's wave-clear predicate.
- The success HUD waits for `nGameDirector.IsBossDead` and then calls
  `SetReadyForBeamOut` after its authored delay. Director reflection field 3 is
  `mbBossComplete`, encoded as application message `8B 08 01`.
- `HUD_BeamOut.swf` resolves to `SP_UI/cBeamOut`, button callback
  `MaxisBeamOut.OnBeamOutClicked`, and copy `RETURN TO SHIP`.
  `HUD_Death.swf` instead owns `MISSION FAILED` / `COMMAND RELAY TERMINATED`
  presentation and its own return button.
- `TutorialGameMsgs` is opcode `0xC8`. Subtype `0` reads one signed 32-bit
  cumulative-XP total. Positive XP emits local `LABS_TUTORIAL_COMPLETE`, updates
  client XP/level, and sets in-memory progress `3000`; nonpositive XP clears
  those fields and sets progress `2000`. The handler never submits an account
  update.
- `LABS_ALL_PCS_DIED`, `LABS_TUTORIAL_COMPLETE`, and `LABS_LEVEL_END` map to
  survey classifications `0`, `1`, and `2`. That consumer is
  `SP_SporeLabs/cSurveySystem`; it is not the mission-result controller.

## Live-observed behavior

These observations are from the documented build-103 focused runs against
darkspin. They validate client consumption and ordering candidates, not retail
server internals.

- The security teleporter remained locked with a live tracked guard. Removing
  the final guard and sending only the continuous active teleporter effect
  produced one usable platform; also sending the power-up effect produced two
  overlapping presentations. A swept entry moved Blitz and the camera to the
  authored arena destination.
- Arena entry displayed `Horde incoming!`; wave one appeared at the two authored
  listener loci after `2s`. Clearing each deterministic pair advanced through
  four waves, with a later wave visibly resolving `Simulated Game Sloth`.
- Sending the defeated field-15 alert followed immediately by positive `0xC8`
  on the final wave kill caused a permanent black screen before the victory UI.
  This disproves final-kill ownership for the completion snapshot.
- Sending director state `8B 08 01` after horde defeat opened the clean
  `RETURN TO SHIP` Beam Out UI without the mission-failed overlay.
- Clicking Beam Out sent `PlayerStatusUpdate` status `0x20`, progress `1`.
  The generic reconnect/ping/depart response reached a chain-voting screen with
  `UNDEFINED` result text, not the ship. The tutorial-specific result payload
  and collection-room transition remain unrecovered.
- No clean squad-death/restart capture is present in the retained trace set.
  Death and restart therefore have no live-observed ownership claim here.

## Inferred ordering, kept separate

The following is the smallest ordering consistent with all proven evidence:

1. Enter Sage's server-only marker.
2. After `0.5s`, unlock Sage mission-locally and notify
   `PlayerUnlockedSecondCreature`.
3. A missing director edge creates/focuses `TutorialSpecialOne_Intro` and
   activates `FirstAggro_SpecialOne`.
4. After the `4.291667s` presentation, enable Quadra combat.
5. Quadra and later fixed guards die; the security passive's radius-20 query
   observes no live, visible-to-security, team-0 creature and changes the
   platform to active. Creature type, default team `0`, and behavior-owned
   exclusion are proven; the deactivation transition still requires
   validation.
6. Accept teleporter entry, move the active hero to destination `174193625`,
   and enter the radius-30 boss director.
7. Publish `horde triggered02`; the two registered listeners receive each wave.
   `waveOverride=4` supplies the best content explanation for four waves, but
   exact selection, counts, budget, inter-wave delay, and aggro remain server
   implementation details.
8. On terminal clear, emit `Horde defeated!`, then set director
   `mbBossComplete`. Do not send `0xC8` on the final kill.
9. The HUD offers Beam Out. The click sends status `0x20`; only an accepted,
   authenticated terminal request may cross the persistence boundary.
10. The server persists completion, removes the player from the tutorial
    instance, retires the completed Blaze game shell, and sends one positive
    `C8 00` tutorial-completion snapshot after the hero beam presentation. Its
    native handler updates the in-memory onboarding snapshot and selects the
    ship state atomically. It must not be stacked with `AF 00` because the pair
    performs the state transition twice and is crash-proven.

Death branches away from any active phase before steps 8-10. The failure HUD
does not prove who detects squad death. Restart must invalidate object-scoped
jobs such as the still-running Quadra first-aggro sequence and construct a new
tutorial session, but cancellation, teardown, and re-entry are reconstruction
requirements until their authored messages are captured.

## Remaining ownership gaps

1. The director/marker edge ordering Sage notification, Quadra create/focus,
   `FirstAggro_SpecialOne`, objective state, control return, and combat start.
2. Recovery of the behavior-tree transition that deactivates
   `nBehavior_Invisible`, and a full five-guard clear, plus
   retained-trigger and teleport-modifier cleanup.
3. The retail horde budget, selected nouns/counts, active-wave bookkeeping,
   clear event, and exact relation between the defeated alert and
   `mbBossComplete`.
4. The retail server transaction that persists progress `3000`, persistent
   Sage/rewards, and the precise ordering of status `0x20`, persistence,
   departure, account refresh, and collection-room arrival. Darkspin's direct
   ship route uses only `AF 00`; the original server's safe use of `0xC8`, if
   any, remains unproven.
5. Squad-death detection, failure director/message sequence, restart-button
   request, server acknowledgement, and clean-session reconstruction.

Until those edges are recovered, they should remain explicit typed director
gaps rather than being assigned to a nearby Lua job, UI callback, alert, or
client-native no-op.
