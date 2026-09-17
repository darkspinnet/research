# PVP end-to-end handoff (build 5.3.0.103)

## Squad selection restored (2026-09-04)

The native scene is `PvPMatchmaking.swf` / `MaxisArenaLobby`: canonical client `sub_404470` registers `ArenaLobby.AcceptMission`, and `sub_404360` selects its two- or four-player layout. `sub_401C70` converts the one-based UI selection into the PVP deck record ID and sends ArenaPlayer subtype 2 with accepted byte 1. `[E]`

Darkspin now keeps this lobby open until every bound member explicitly accepts an owned, unlocked PVP squad containing three distinct valid heroes. The accepted binding supplies the heroes used by dungeon setup. Lobby entry and elapsed branch retries never select a default deck or start loading; loading retries apply only after the full roster accepts. Early loading statuses cannot bypass this barrier. The admission default remains provisional. No shipped asset or Fang compatibility hook is changed.

This supersedes the historical default-deck preparation workarounds described below. Production compilation and `mage build` pass; live multiplayer rendering and Continue acceptance remain unverified. In particular, the earlier intermittent native lobby-entry failures must be checked in the client instead of silently bypassed with squad 1.

## Scope and evidence

This note connects the already recovered application packets to the Blaze
matchmaking route, shipped level content, and darkspin implementation gaps. It
is an implementation handoff, not a claim that PVP currently works.

`[E]` is direct evidence from the canonical
`bin/game/GameBin/Game.c`, canonical-IDB-backed packet research, or
the authoritative imported `content.db`. `[I]` is an inference that must remain
named as such until a retail capture or a successful multi-client probe proves
it.

Arena is the correct first implementation target. It has a complete build-103
lobby/result record, three client requests, five server game-message subtypes,
six shipped PVP level candidates, four-player matchmaking criteria, and a
client UI that explicitly selects two- or four-player presentation. Juggernaut
and Kill Race have narrower packet evidence and no recovered production lobby
payload; keep them out of the first playable slice.

## Recovered client route

The client uses these numeric gameplay modes in `sub_5169C0`: `[E]`

| Value | Client label |
|---:|---|
| `1` | `Tutorial` |
| `2` | `Chain_Matched` or `Chain_Premade` |
| `3` | `Arena` |
| `4` | `KillRace` |
| `5` | `Juggernaut` |
| `6` | `QuickPlay` |
| `7` | `DirectEntry` |

The Map Room SWF calls `MapRoomUI.StartGame`, whose native callback reaches
`cOnlinePlatformBlaze::CreateGame` and then `sub_C387A0`. `[E]` That routine
owns both direct game creation and matchmaking request construction. It always
publishes string attributes `SelectedDifficulty`, `Ranked`, and `GameType`.
`[E]`

For matchmaking (`a4 == 1`) with `GameType == 3`, the client constructs this
Arena contract: `[E]`

- `GameGameTypeRule`, fit threshold `requireExactMatch`, desired value
  `3`;
- `ExpectedPlayerCount="4"`;
- `TeamSize="2"`;
- ranked skill rule `arenaSkillRanked_Rule`, fit threshold `decay`;
- game attributes `GameType="3"`, `Ranked="0"` or `"1"`, and the selected
  difficulty string;
- a 30,000-millisecond request timeout in the local request object.

This is exact evidence for retail Arena being a four-player, two-player-per-team
queue. It does not prove how the retired server selected maps, paired ratings,
or populated every Blaze team field.

The playgroup path exposes a second exact route. When a multi-member playgroup
receives a nonzero `TeamId`, `sub_C2E480` starts a PVP team matchmaking request
with `PvPTeam`, `GameType=3`, `Ranked=0`, `TeamSize=<playgroup size>`, and the
skill name `glickoSkill`. `[E]` Its callback `sub_C2EF80` writes the resulting
`ArenaSkill` game attribute; it uses string `"500"` as the local fallback when
no one-element skill result can be recovered. `[E]` Do not turn `500` into a
global rating policy: it is only the observed client fallback.

Build 103 gates that route with a second party-wide capability check in
`sub_519A00`. Every playgroup member record starts incapable, and the native
peer packet emitted by `sub_519A70` carries subtype `7` plus the sender's local
PVP-capability byte; subtype `8` carries ready state. `[E]` The 15:57 local
trace recorded UserSessions `ADDR` publication from both accounts, but the
leader's CreatePlaygroup request supplied an unset `HNET` member and no peer
channel carrying those packets appeared in either client socket trace. `[E]`
The Blaze adapter first retained each nonempty `ADDR` on its authenticated
session and substituted it for unset playgroup `HNET` or `PNET`, while retaining
an explicitly supplied playgroup address. `[I]` The subsequent 16:40 two-client
run projected active-member-2 address unions into both roster entries but
opened no cross-client socket; the server was still projecting `NTOP=4`. `[E]`
The corrected 16:58 run then proved that build 103 requests `NTOP=0`, the
Blaze `CLIENT_SERVER_PEER_HOSTED` topology, while Darkspin incorrectly replaced
it with full mesh `130`. `[E]` Darkspin now retains the client-authored
peer-hosted topology. Darkspin no longer projects those client-authored
addresses to other users. It retains them privately as exact transport-binding
hints and projects a dedicated server-owned UDP endpoint into every playgroup
`HNET` and `PNET`. The client still sees the distinct logical party endpoint on
port 42128. Fang prefixes each datagram for that endpoint with the eight-byte
`DSPG 01 01 00 00` channel envelope and transmits it to physical port 42127;
Darkspin removes the envelope and dispatches it into an independent party
RakNet peer/ACK space behind the shared socket. Replies originate on 42127 and
Fang restores 42128 as the source visible to the party socket. The relay
recognizes only native control subtypes `7` and `8`, binds their source to a
committed party member, retains the latest capability/ready byte, and
republishes it to the other bound party transport. No client-authored address
is projected to another player and no additional server port is allocated.
Whether build 103 preserves distinct peer identity for three or four roster
entries with one identical endpoint requires a live probe.

The 17:45 two-client run resolved both roster personas and projected presence
state `4`, but still displayed the native offline and party-wide PVP unlock
errors without opening a post-join party transport. The join adapter was
sending the existing host both `NotifyJoinPlaygroup` (`0x33`) and
`NotifyMemberJoinedPlaygroup` (`0x34`). The former is the local admission
envelope for the joining user and resets the recipient's playgroup object. A
later 19:02 run proved that `0x34` alone leaves the application roster stale,
so the current path sends a recipient-owned full `0x33` snapshot followed by
the identity-matched `0x34` event. `[E]`

The 19:33 Rawr/Foo run used the Fang-enabled binary but emitted no logical
port-42128 datagram and reached no party relay handler. Rawr's party map held
Foo's ID, then the primary platform lookup reported that ID unresolved after
the client had issued a UserSessions `LookupUser` request. That request used
the packed Playgroups external identity, but the response replaced it with the
login-only zero external ID. The 19:43 follow-up emitted the corrected packed
lookup response, and the 19:50 follow-up also published the packed identity in
the pre-roster `NotifyUserAdded`, but Rawr still reported the presence query as
unresolved. `cOnlinePlatformBlaze::GetPresenceInfo` at `0x00C2B480` proves this
result specifically requires the cached user's typed
`Association::PresenceInfo`. Darkspin had changed the immediately following
`NotifyUserUpdated` field mask from the previously successful `2` to `1`; the
19:50 trace therefore isolated that change from the now-correct identity and
presence envelope. Live update notifications again use mask `2`, while lookup
responses retain their distinct mask `1`. The automatic lookup response also
now retains the registry's typed `Association::PresenceInfo` instead of
returning an empty `EDAT.CVAR` that can downgrade the just-published user before
the party gate queries it. `[E]`

The 20:04 follow-up proved Rawr now resolves Foo with presence state `4`,
playgroup size `2`, level `335`, and client data `3`, while neither client opens
logical port 42128. Every `HNET` and `PNET`, including the recipient's own
member, still contained the same relay endpoint. `[E]` The client therefore had
no distinct peer endpoint from which to establish the channel that invokes
`sub_519A70`. Playgroup projection now returns each member's private published
binding only to that same member and substitutes the relay endpoint for every
other member. A client can distinguish itself from its relay peer without
learning any other player's address, and Fang still carries the remote logical
42128 traffic over physical port 42127. `[I]`

The 20:24 follow-up still displayed `Foo is not online` and rejected PVP after
Foo had joined Rawr's playgroup. The wire trace showed Rawr's automatic
UserSessions `LookupUser` request immediately after the join. The native lookup
callback in `sub_C34CD0` sends an invitation only when `sub_DFF280` does not
return `1`. `sub_DFF280` decodes the cached user's flags as unknown when bit 0
is clear, offline when bit 0 alone is set, and online only when bits 0 and 1 are
both set. Darkspin returned `FLGS=1` in every successful lookup and therefore
explicitly converted the resolved Foo user to offline. Successful lookup
responses now return `FLGS=3`; notification `FLGS` fields retain their separate
update-mask meanings. The temporary single-peer recovery formerly sent an
invitation from inside `CreatePlaygroup`, before this native lookup could
finish. That produced the contradictory offline line followed by a usable
invite, and could race the client's genuine message once lookup became valid.
The recovery is removed so the native lookup and Messaging request are again
the only invitation path. The PVP popup's immediate gate is separate:
`sub_519A00` requires a valid local three-creature squad and a true byte on
every native party-member record. Restoring the normal invite lifecycle is
required before treating a remaining capability-byte failure as a relay bug.
`[E]`

## Matchmaking lifecycle

The 17:53 four-client run formed Arena game 1 with Foo, Rawr, Three, and Four,
and the gameplay relay merged all four slots (`connected_mask=0xf`). Before the
match formed, Foo's client sent the normal leave transition while Rawr still
owned matchmaking session 4; preserving a party only after `CurrentGameID`
was assigned allowed that transition to split the roster. The playgroup adapter
now shares the active matchmaking registry and preserves such a leave while a
different party member remains queued. After the relay merge, every client sent
`DebugPing`, which the shared setup path incorrectly routed through
`CampaignSetup` and rejected with `campaign setup: invalid mode`; Arena pings
now bypass the campaign vote/dungeon setup path. `[E]`

The corrected 18:23 run retained all four gameplay transports and merged the
complete `0xf` roster. Only Foo, the final matchmaking caller, then entered
Arena Lobby and requested `ArenaPlayer` subtype `0`; Three, Four, and Rawr
continued sending pre-lobby pings. The Arena adapter now sends subtype `5` with
branch zero to every other connected match member when the first member enters
the lobby, selecting build 103 state 15 before each client requests its own
lobby snapshot. `[E]` A two-member party also no longer counts as one player:
its leader-owned reservation contributes both authenticated members as one
contiguous team, while the teammate receives an indirect playgroup setup rather
than a matchmaking reason for a client-local session it never created.

The 18:53 party-plus-two-solos run formed Arena game 1, displayed the native
match-found state on the clients, joined all four gameplay sockets, and merged
the complete `0xf` roster, but no client emitted `ArenaPlayer` subtype `0`.
This proves lobby entry cannot depend on one participant advancing first. The
roster merge now sends subtype `5` branch zero to all four Arena sessions
immediately after `PartyMergeComplete`; the first-player lobby handler remains
an idempotent fallback. `[E]`

The 22:10 direct-1v1 run merged Foo and Rawr and emitted subtype `5` in the
same response as `PartyMergeComplete`, but neither client emitted
`ArenaPlayer` subtype `0` during the following three minutes. Earlier runs
advanced roughly half a second after the same branch packet, showing that a
transport-acknowledged branch can arrive before the current client state has
registered its ArenaGame callback. Gameplay now retries the branch per client
at short bounded intervals and stops immediately when that client enters the
lobby. `[E]`

`sub_C3A700` is the start-matchmaking callback. On success it retains the
returned matchmaking session handle; on error it raises the client failure
message and clears the local session. `[E]` `sub_C3B0E0` consumes the terminal
matchmaking result: result values `0`, `1`, and `2` take the success path and
attach the returned Blaze game; values `3` through `6` take the failure path.
`[E]`

The client mirrors lifecycle state into the playgroup attribute
`MatchmakingStatus`: `[E]`

| String | Client behavior |
|---|---|
| `"1"` | queue entered; Map Room receives `OnPlaygroupMatchmaking` |
| `"2"` | matchmaking session finished successfully |
| `"3"` | terminal failure; Map Room receives `OnMatchmakingFailed` |

The joined game's `GameType` attribute is later read in
`sub_C3A9D0` and emitted through the client's internal game-mode message. `[E]`
Returning only an `MSID` from Blaze command `0x0d` cannot reach Arena: the
server must eventually finish the matchmaking session with a real joined
Blaze game whose attributes still contain `GameType="3"`.

Cancellation uses GameManager command `0x0e`. The existing crash fix is still
required: `NotifyMatchmakingFailed` notification `0x0a` needs
`MAXF=0`, the request `MSID`, `RSLT=4` (cancelled), and the authenticated
`USID`. The next implementation must additionally make session IDs unique,
bind them to their owner, remove queued membership atomically, and make repeat
cancellation harmless.

## Shipped level candidates

The authoritative imported `content.db` contains six levels whose authored
names end in `_PVP`. `[E]` These are content-proven candidates, not a recovered
retail rotation or weighting policy.

| Level row | Name | Resource | Package group | Primary / secondary type |
|---:|---|---:|---:|---:|
| `6` | `cryos_2_PVP` | `7683` | `941053233` | `3 / 1` |
| `12` | `infinity_1_PVP` | `12778` | `1668680222` | `0 / 3` |
| `20` | `nocturna_2_PVP` | `3333` | `2367644331` | `4 / 3` |
| `25` | `scaldron_1_PVP` | `9043` | `2921965874` | `5 / 5` |
| `53` | `verdanth_3_PVP` | `5660` | `2472613768` | `2 / 4` |
| `59` | `zelems_2_PVP` | `829` | `4124993463` | `1 / 0` |

Their authored design sets contain `SpawnPoint_Team1.Noun` through
`SpawnPoint_Team4.Noun` groups and per-group variants. `[E]` Infinity,
Nocturna, and Zelem also contain two `prefab_PVParena_lobby_*` markers in their
PVP-common sets. `[E]` Each lobby prefab center coincides with the corresponding
Team1 or Team2 marker cluster center, while the Team3 and Team4 clusters occupy
the playable arena near one another. `[E]` Team1 and Team2 are therefore the
opposing preparation placements and Team3 and Team4 are their respective
combat starts. Most levels provide four variants per group; Nocturna has one
Team3 and one Team4 combat marker, so a two-player team needs a small
server-owned separation around that authored anchor. The server now preserves
all four groups, stages by gameplay team in groups one and two, and moves those
teams to groups three and four after the opening countdown.

`Juggernaut_Mode_Testing` is a separate level row (`17`, resource `4044`). Its
design set contains three team spawn points and four
`SpawnPoint_JuggernautOrb.Noun` markers. `[E]` Its name and missing production
siblings make it test-content evidence only, not proof of a released playlist.

Use the supported database command for further inspection rather than a
one-off SQLite script:

```text
bin/server/darkrun.exe db level get name!=__none__ --config bin/darkspinner/darkspin.toml --limit 1000
bin/server/darkrun.exe db level_marker_set get level_id=12 --config bin/darkspinner/darkspin.toml --limit 100
bin/server/darkrun.exe db marker get level_marker_set_id=405 --config bin/darkspinner/darkspin.toml --limit 1000
```

## Arena lobby and deck selection correction

The packet layouts remain in [`arena.md`](arena.md). Two semantic connections
are now stronger than the earlier opcode-only research.

The `0xb4` subtype-`3` snapshot's six 205-byte player records are also the
Arena lobby roster. `sub_403230` iterates all six records, resolves the 64-bit
player ID against connected players, reads avatar ID at record offset `+8`,
and reads team at `+12`; valid entries are passed to SWF `AddPlayer`. `[E]`
Unused roster slots therefore need zero player IDs. The same records feed the
round-results UI later.

The `0xb3` subtype-`2` dword is not a mission index. The
`ArenaLobby.AcceptMission` callback treats the SWF argument as a one-based
index into the local 64-byte PVP squad records, stores the selected record's
first dword as the active PVP deck, and sends that dword after the literal
accepted byte. `[E]` The server field should be treated as `deck_id` (or
`squad_id` if that is the feature's established domain noun), validated
against the authenticated player's unlocked PVP squads and three owned hero
slots.

The lobby UI chooses `SetTwoPlayerMode` when no team has more than one roster
member and `SetFourPlayerMode` when a team has exactly two. `[E]` This can make
a two-client diagnostic lobby render, but it does not override the exact
four-player retail matchmaking criteria above.

## Current darkspin gap inventory

The repository now has the first connected Arena foundations, but no playable
match yet:

- `server/pvp.Matchmaking` owns unique authenticated Arena queue sessions,
  enforces the recovered four-player/two-team criteria, makes repeated entry
  idempotent, only permits the owning user to remove a session, and atomically
  consumes the first four compatible sessions as one ordered admission.
- `cancelMatchmakingHandler` removes owned queue state and still emits the
  crash-safe cancelled notification; missing/repeated cancellation is harmless.
- The fourth compatible request creates one admission-reserved `game.Instance`,
  adds the users at slots 0--3, assigns slot pairs to teams 0/1, publishes the
  common GameManager setup/player roster, and sends each owner terminal result
  `2` with that owner's `MSID` and `USID`. This ordering and the team fields are
  an instrumented implementation hypothesis pending the first four-client run.
- `gameplayPacketHandler.dispatch` routes `0xb3` through
  `DecodeArenaPlayerCommand`. Lobby entry now returns a subtype-3 `0xb4`
  snapshot populated from all connected members of the same game; mission
  acceptance is currently admitted only for the authenticated binding's
  default PVP squad.
- Arena mode 3 randomly selects one of the six shipped `_PVP` levels, binds the
  account's default PVP squad, and uses RakNet gameplay type 3. The
  client-proven lobby prefix uses flag `1` to allow squad swapping and word `1`
  to enable squad statistics. Once every player commits dungeon setup, a
  server-owned five-second timer moves Team1/Team2 staging occupants to the
  corresponding Team3/Team4 combat starts before admitting damage.
- Arena game and result encoders still have no normal round-state producer.
- PVP now admits player-versus-player basic attacks with opposing-team and
  opening-state checks and cross-projects remote hero ownership. Special and
  squad ability authority, death/respawn or round elimination,
  spectator/disconnect policy, and server-owned statistics remain incomplete.
- Profile fields and leaderboard projections already expose the decoded PVP
  lifetime contract. `PlayerStatDelta` can now atomically persist PVP play time,
  wins/losses, kills/deaths, damage dealt/taken/max, and healing
  dealt/received/max, but no Arena authority produces those deltas yet.

Build 103's destination controller computes `SetPvPUnlocked` from the parsed
account `unlock_pvp_decks` scalar being greater than zero. Dark Spin now grants
the first PVP deck in new-account defaults and while normalizing older local
profiles, so the native PVP destination becomes selectable without the retired
level-10 purchase service. The exact retail purchase request remains a capture
target; this local compatibility policy exposes the restored Arena path rather
than changing packaged UI state.

For party members, the same controller consumes each remote player's public
`api.account.getAccount` response before evaluating the party-wide gate. The
public allowlist must therefore project `unlock_pvp_decks` as a gameplay
capability; omitting it leaves the remote member's `SetPvPUnlocked` state false
even though the stored account is normalized to an unlocked PVP deck. `[E]`

The native direct 1v1 path uses `resetDedicatedServer` rather than matchmaking.
Its request identifies Arena with `GameType=3`, sends `SelectedDifficulty=0`,
and records the selected persona IDs in `CustomTeam11`, `CustomTeam12`,
`CustomTeam21`, and `CustomTeam22`. A two-player capture sent Rawr in
`CustomTeam11` and Foo in `CustomTeam21`; these attributes are authoritative
for the direct game's two team assignments. `[E]`

The shipped Map Room movie contains separate presentation restrictions in
`MapRoomUI.OnUnrankedClicked` and `MapRoomUI.UpdatePvPButtonsForLeader`. The
selection handler assigns `PVPMode2V1.disabled = true` directly, and the
update handler's shared unranked/ranked branch repeats that unconditional
assignment while gating 2v2 on party size.
The custom branch instead gates 1v1 on an exact two-member party, so redirecting
unranked execution into that branch still rejects a solo player. Later
play-button logic explicitly accepts 1v1 when the party size is one and the
rank type is unranked, making the unconditional disable the isolated blocker
rather than the native start-game contract. The first Fang implementation
polled committed private memory, but the live client proved that Scaleform
streams this action data without retaining the complete signed sequence in
such an allocation. Fang now hooks the build-103 decompressed SWF stream read
vtable entry at RVA `0xCE4ED0`, scans each buffer before Scaleform parses it,
and changes both exact Map Room `PVPMode2V1.disabled` literals from `true` to
`false`. The update signature includes the actual constant-16 encoding of
`kPvPRankType_Ranked`; the selection and update signatures are each unique
across all 109 shipped CFX movies. This exposes solo unranked 1v1 without
changing `mPvPRankType`, the 2v2/custom gates, the shipped movie, or any other
client asset. `[E]`

The native `MapRoomUI.StartGame` callback receives the selected game size but
discards it before constructing every Arena matchmaking request as four players
with teams of two. Fang now captures the exact Arena, 1v1, unranked callback
tuple and rewrites only its outgoing Blaze start-matchmaking string criteria to
`ExpectedPlayerCount=2` and `TeamSize=1`. The server accepts that roster shape
alongside the original four-player shape and forms two one-player teams; 2v2 and
custom Arena flows retain their prior criteria. `[E]`

The 21:43 direct-1v1 follow-up joined both gameplay sockets and merged mask
`0x3`, then both clients entered Arena Lobby and sent subtype `0`. The server's
snapshot still recomputed team as `slot / 2`, incorrectly placing slots zero
and one together despite the opposing `CustomTeam` selection. Lobby snapshots
now retain each game member's assigned team. `[E]`

The 22:21 rerun proved the bounded Arena-branch retry reached both clients:
Rawr sent lobby-entry subtype `0` at 22:21:35.918, Foo sent it at
22:21:35.948, and both received the subtype-`3` `0xb4` roster snapshot. Neither
then sent `AcceptMission` subtype `2`. Decoding the shipped `PvPMatchmaking`
ActionScript corrects the earlier interpretation of `sub_4039A0`: its final
three `SetSquadData` arguments are the active squad index, `mAllowedToSwap`, and
`mbEnableStats`. Lobby flag `1` therefore allows swapping and lobby word `1`
enables statistics; `DisableSquadSelect` is a separate native call used only
when no local player exists or its flags contain `0x100`. `[E]`

The 22:29 rerun delivered a complete subtype-`3` lobby snapshot to both clients
with the correct player IDs, avatar IDs, and opposing teams, but both remained
in the void. Build 103's lobby update at `sub_447FE0` retains that snapshot and
refuses to construct the UI until `sub_9BF0E0` sees every allocated gameplay
player's data-setup byte at offset `+8` set. `OnGmsHelloPlayer` and
`OnGmsPlayerJoined` allocate/reset those player records through `sub_9C2340` but
do not set data setup. The initial `LabsPlayerUpdate` reflection does so through
player field `0`. Arena roster merge now publishes every member's initial
player reflection, including its assigned one-based team, to every client before
`PartyMergeComplete` and the Arena branch, satisfying the native readiness gate
before the lobby snapshot arrives.
`[E]`

The 23:04 rerun exposed an ordering hole in that baseline delivery. Rawr, the
last gameplay joiner, received both fragmented initial-player streams directly
in its join response. Foo received the Arena branch through the retry path but
none of the asynchronously queued initial-player stream; both clients then sent
lobby-entry subtype `0` and received their snapshots, but neither sent deck
acceptance subtype `2`. The lobby-entry response now repeats the complete,
slot-ordered initial-player baseline before its snapshot. This makes the
client's own acknowledged lobby transition the synchronization boundary and
does not depend on the earlier peer queue arriving first. `[E]`

The 23:25 rerun proves that synchronization boundary works: both clients
received both ordered, fragmented initial-player records followed by their
lobby snapshot. Neither emitted deck acceptance subtype `2`, so the remaining
stall is no longer packet loss or snapshot ordering. Until a Fang observation
identifies why build 103 does not construct the native picker, the server uses
the authenticated default PVP squad already bound during gameplay admission as
the conservative selection when every required member acknowledges ArenaLobby.
It then publishes `GamePrepareForStart` to the complete roster, allowing the
normal status `2/4/8` loading handshake to continue. This fallback is recorded
in `notes/help.md` because a retired-server no-response policy is unavailable.
`[E]`

## Implementation order

### 1. Matchmaking owner

Implemented: unique sessions, authenticated ownership, strict Arena criteria,
idempotent repeat entry, owner-checked cancellation, and atomic four-entry
admission. Disconnect cleanup remains.

Create a PVP feature-owned queue/session operation rather than putting mutable
queue state in the Blaze handler. Parse and retain `GameType`, `Ranked`,
`ExpectedPlayerCount`, `TeamSize`, and the exact criteria names above. Reject
unsupported modes before writing state. Use unique session IDs, authenticated
ownership, idempotent cancellation, and disconnect cleanup.

The first supported queue should require `GameType=3`, expected count `4`, and
team size `2`. Keep direct/premade game creation distinct from queued
matchmaking because `sub_C387A0` uses different Blaze calls and attributes for
the two paths.

### 2. Blaze game completion

Implemented with an explicitly provisional field policy: four compatible users
create one `game.Instance`, occupy stable slots, receive two teams, retain the
Arena attributes, use the first fallback map, and finish every matching session
with the same joined game through the existing GameManager notification
builders.

The exact retired-server map rotation, rating window, and Blaze team-field
encoding remain unknown. Instrument the parsed request and resulting game
field shapes during the first multi-client run. Do not claim `TIDX`, `PvPTeam`,
or a spawn-group mapping as authoritative until the client visibly assigns
teams correctly.

### 3. Shared gameplay allocation

Partially implemented: each admitted peer resolves through the same game ID,
loads the selected `_PVP` level through the existing content/navigation path,
uses its default PVP squad, and receives gameplay type `3`. The existing shared
zone roster publishes remote players. Competitive action and target authority
must still reject remote-object ownership and friendly targets before combat is
enabled.

### 4. Arena lobby handshake

Route `0xb3` through a dedicated Arena operation:

- subtype `0`: mark that peer in the Arena lobby and send a subtype-`3`
  `0xb4` snapshot containing the current roster;
- subtype `2`: validate the sent PVP deck ID and mark that player ready;
- subtype `1`: acknowledge that the client entered round-results state and
  send the authoritative `0xb6` subtype-`4` snapshot for the completed round.

Do not let packet arrival alone advance the match. Required-player admission,
deck validation, and current-phase checks belong to the Arena aggregate. The
meanings of the `0xb4` prefix flag/word are still unresolved, so log the client
effect of candidate values during a controlled probe before treating either as
business state.

### 5. Round authority

Start with one Arena rules owner that controls countdown, active round,
elimination, round timeout, inter-round reset, and match completion. Use the
existing movement/ability/stat systems, but add explicit competitive target
authorization before enabling damage. Track kills, deaths, damage dealt,
damage taken, and healing from accepted authoritative events, never from a
client result packet.

Arena results reserve three rounds per player and treat round outcome `3` as a
win in the client UI. `[E]` This strongly supports a best-of-three presentation
but does not by itself recover the retired server's draw, timeout, respawn, or
disconnect rules. Keep those unresolved in the rules owner so later evidence
can replace policy without changing wire code.

### 6. Results and persistence

Populate all six fixed roster slots deterministically, zero unused slots, and
send the same authoritative results body through `0xb4` lobby refreshes and
`0xb6` round-results snapshots. The per-player lifetime counters and per-hero
health/mana percentages are decoded in [`arena.md`](arena.md). Resolve the
still-opaque selectors/counts and per-round `field_04` through a two-client UI
probe before giving them durable domain names.

Commit profile wins/losses/kills/deaths/damage/healing, rewards, and rating in
one feature-owned persistence operation after terminal match acceptance.
Rejected or duplicate completion must not write. Only after persistence
succeeds should public profile and leaderboard projections expose the result.

### 7. Alternate modes

Defer Juggernaut and Kill Race until Arena can complete through matchmaking,
gameplay, results, and return-to-ship. Their recovered packet notes remain
useful for later state transitions, but neither family currently supplies the
missing authoritative rules or production lobby content.

## Required observations

The next live research should preserve Blaze and RakNet traffic from at least
two clients and answer these questions in order:

1. Which Navigation state or unlock makes the PVP icon selectable at account
   level 10, and what request purchases or confirms unlock ID `38`?
2. What complete TDF field shape does command `0x0d` send for ranked and
   unranked Arena, including team capacity and criteria containers?
3. Which GameManager notifications and game/player team fields occur between
   matchmaking success and the RakNet connection?
4. Which of the four authored spawn groups correspond to the two teams and to
   later rounds on each map?
5. What are the first `0xb5` subtype sequence and values for a real Arena
   lobby-to-round transition?
6. Which opaque Arena result fields change for a kill, death, round win,
   timeout, DNA reward, and match win?
7. Does a disconnected player forfeit a round, permit reconnection, or end the
   match?

Keep protocol captures under `bin/server/darkspin/logs/traces` and any new IDA
or decoded diagnostics under `bin/game/logs`.

The Arena Lobby branch is acknowledgement-driven. Once the complete gameplay
roster is merged, the server repeats the branch at a one-second cadence until
that session sends `ArenaPlayerEnterLobby`; a fixed short retry window can
expire while build 103 is still consuming its initial player stream. Retry
logging stays limited to the first three attempts and each tenth attempt.

After default-deck preparation, build 103 reports player statuses `2`, `4`,
and `8` on the ordinary gameplay status opcode. Status `8` is the loaded Arena
boundary: it needs the member-status projection and `GameStart`, but must not
enter campaign director initialization. The 2026-08-27 two-client trace showed
both peers being discarded when that campaign-only operation rejected mode 3.

Subtype `5` alone is not a deterministic lobby barrier. Its five client
receivers are owned by specific active-game states, and the 23:49 run discarded
more than thirty reliable branch deliveries without registering any receiver
or sending ArenaPlayer subtype `0`. A later two-client run split at this exact
boundary: Foo entered ArenaLobby while Rawr discarded the branch, so the old
all-lobby barrier prepared neither client and the per-client fallback prepared
only Rawr. Starting at any member's fifth branch attempt, the server now sends
the ordinary `GamePrepareForStart` to every unacknowledged member using that
member's authenticated default PVP squad. Each member then receives an
individual retry until status `2`, `4`, or `8` acknowledges loading.

The 00:03 two-client run reached status `8` on both peers, but the Arena path
returned only member status and `GameStart`. Unlike campaign, it had no later
dungeon publisher, so neither client received `GameDungeon`, hero object
creation, controlled-object ownership, or the initial deploy. The client then
submitted a hero switch with object ID zero, whose unavailable rejection
retired that RakNet peer. Arena now owns a minimal dungeon publisher: it creates
the bound three-hero squad, publishes team-correct hero objects and resources,
assigns and deploys hero zero, and fans each roster out to the other clients.
Its temporary origin-adjacent spawn coordinates remain a recorded fallback
until the authored four-group mapping is recovered.

The first rerun after adding that publisher proved it was still unreachable:
both clients continued sending `DebugPing`, but `gameplaySetupRuntime.handle`
returned early for every Arena session before calling `publishDungeon`. Removing
that guard wholesale exposed a second phase distinction in the 00:21 rerun:
pre-lobby pings entered the campaign chain-vote preview, failed with `campaign
setup: invalid mode`, and retired both peers before Arena Lobby. The correct
split is stage-sensitive. Arena pings remain inert until status `8` enters the
dungeon stage. The 00:24 rerun then showed that build 103 does not emit the
campaign `DebugPing` setup poll after the Arena start handshake: both peers
reached status `8` and stayed connected, but `publishArena` was never called.
Arena status `8` now enters the dungeon stage and immediately proceeds through
the setup reservation into `publishArena`, preserving packet order without
depending on a campaign-only follow-up poll.

The 00:30 rerun confirmed both hero rosters and controlled objects spawned, but
every movement command entered campaign navigation and failed because Arena has
no campaign zone route. Character-action types `7` and `8` were excluded by the
campaign ability mode gate, returned no terminal response, and were recovered
as rejected actions before damage admission. Arena now owns direct movement and
targeted combat admission. It accepts only the caller's deployed object, selects
only a live opposing-team deployed object in the same game, applies authored
range and projected damage with the recorded midpoint roll fallback, mutates
the target squad's authoritative health, and publishes the combat/health state
to every Arena peer without requiring shared campaign-zone identity. Special-two
and random hero actions additionally spend projected mana, reserve their
authored cooldown and release windows, and publish their native animation.
Their first playable PvP adapter applies the authored damage projection to the
selected enemy as a conservative single-target fallback; exact per-shape PvP
area, projectile, teleport, and status policies remain pending. Squad/support
abilities remain explicitly rejected until their PvP policies are implemented.

The same 00:30 run showed voluntary squad swaps reaching the shared switch
runtime, then aborting at `switchCampaignNPCSession: unavailable`. The roster
and cooldown admission were valid; only the implementation incorrectly required
Arena to own campaign NPC state. Arena now revalidates the requested living
reserve hero under the session lock, commits its authoritative squad cooldown
and controlled-object identity, and publishes the source hide plus target
deploy, arrival, and resource state to every Arena peer without an NPC session.

The 00:48 rerun then submitted opposing-team basic attacks correctly, but both
were rejected solely by range admission while the clients presented the heroes
as close enough to attack. Arena had reconciled only the attacking client's
pose and compared it with the opponent's last stored movement pose. It now
advances both motion timelines to the attack instant and gives melee the
client-recovered three-unit held-target radius that campaign pursuit uses for
close-target acquisition. Rejections retain measured distance and maximum
range in one compact log line so the fallback can be corrected from later
captures.

Developer `/victory` now provides a bounded result-screen probe without
pretending normal Arena completion is implemented. The consuming gameplay
session marks the invoker's team as winner, terminates action admission for all
members, and sends ArenaGame subtype `5` with a true branch to every peer. Each
client consequently enters the named ArenaRoundResults state and sends its
proven ArenaPlayer subtype `1`; the Arena adapter answers that request with the
fixed ArenaResults subtype-`4` body containing the complete slot-ordered roster,
two winning outcome-`3` rounds for the selected team, and current per-hero
health/mana percentages. The still-opaque header selector/count policy remains
a recorded developer-only fallback and no profile stats are committed.
