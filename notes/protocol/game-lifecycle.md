# GameManager multiplayer lifecycle (build 5.3.0.103)

## Scope and evidence

This note records the client contract recovered for moving a frozen party roster from Blaze game setup into the shared RakNet session. `[E]` marks direct evidence from `bin/game/GameBin/Game.c`, its canonical IDA database, or the captured `2026-08-20T11:00:10Z` two-client exchange. `[I]` marks an inference that the next native run must validate.

## Recovered enums

The generated GameState enum is initialized by `sub_FB2AF0` from the nine entries at `off_10D3B70`: `[E]`

| Value | Client label |
|---:|---|
| `0` | `NEW_STATE` |
| `1` | `INITIALIZING` |
| `0x82` | `PRE_GAME` |
| `0x83` | `IN_GAME` |
| `4` | `POST_GAME` |
| `5` | `MIGRATING` |
| `6` | `DESTRUCTING` |
| `7` | `RESETTABLE` |
| `8` | `REPLAY_SETUP` |

The generated PlayerState enum is initialized by `sub_FB2B50` from the six entries at `off_10D3C10`: `[E]`

| Value | Client label |
|---:|---|
| `0` | `RESERVED` |
| `1` | `QUEUED` |
| `2` | `ACTIVE_CONNECTING` |
| `3` | `ACTIVE_MIGRATING` |
| `4` | `ACTIVE_CONNECTED` |
| `5` | `ACTIVE_KICK_PENDING` |

`ReplicatedGamePlayer` visitor `sub_DEC0B0` proves that `SID` is the one-byte player slot and `SLOT` is the separate SlotType enum. The initial roster must therefore use stable slots `0/1`, `SLOT_PUBLIC=0`, and `ACTIVE_CONNECTING=2`; the mesh transition later publishes `ACTIVE_CONNECTED=4`. `[E]`

## Application callbacks

`sub_C3A960` is `cBlazeGameManager::onPlayerJoining`. `sub_C3A9D0` is `cBlazeGameManager::onPlayerJoinComplete`. The join-complete callback is invoked for replicated players, but only its local-player branch registers the report callback, publishes the selected GameType, reads `ExpectedPlayerCount`, and advances the Game application toward gameplay. `[E]`

This means each client must learn that both replicated players completed joining, while each player's own completion is the event that advances that local application. Sending a completion only to its owner leaves the peer object unfinished; completing one player twice risks repeating local transition work. `[I]` Darkspin therefore reserves and broadcasts each connected/join-complete pair exactly once per player.

The relevant GameManager notification IDs recovered by `sub_DEB660` are: `[E]`

| ID | Notification |
|---:|---|
| `0x14` | `NotifyGameSetup` |
| `0x15` | `NotifyPlayerJoining` |
| `0x1e` | `NotifyPlayerJoinCompleted` |
| `0x64` | `NotifyGameStateChange` |
| `0x74` | `NotifyGamePlayerStateChange` |

`NotifyPlayerJoinCompleted` carries only `GID` and `PID`. `[E]`

`NotifyGameSetup.REAS` is a generated five-member union selected by `sub_DEFFA0`: member `0` is `DatalessSetupContext`, member `1` is `ResetDedicatedServerSetupContext`, member `2` is `IndirectJoinGameSetupContext`, member `3` is `MatchmakingSetupContext`, and member `4` is `IndirectMatchmakingSetupContext`. Dataless `DCTX` distinguishes create from an explicit client join. Indirect join is not `DCTX=2`: its separate member contains `GRID` as the reserving playgroup object ID and `RPVC` as a boolean. `[E]`

`MatchmakingSetupContext` member `3` contains `FIT`, `MAXF`, `MSID`, `RSLT`, and `USID`; its generated defaults are user-session ID `-1` and zero for the remaining fields. The generated vtable at `0x010d549c` binds its visitor to `sub_DEFE00`, whose five field tags establish this schema directly. `[E]` Darkspin fills `MSID` with the receiving player's matchmaking session, `USID` with that player's live Blaze session ID, and selects created-game result `0` for the stable host or joined-new-game result `1` for the other entrants. The matched roster uses the same live session IDs for `PROS[].UID` and the host's `GAME.HSES`; account/persona IDs remain in `PID`, `HPID`, and other persona fields. This context belongs to independently matched rosters; party-reserved peers continue to use member `2`. `[I]`

## Captured failure and corrected exchange

The 2026-08-27 14:08 four-client run admitted Four, Three, Rawr, and Foo as matchmaking sessions `1` through `4`, but the fourth StartMatchmaking response failed with `0x4001`. Game formation had completed, then roster projection reused the party-only indirect-join union and rejected the new instance because it had no playgroup ID; rollback removed the formed game and consumed all four sessions. Independently matched rosters now project member `3` to every entrant instead of entering that party-only validation path. `[E]`

The 14:30 follow-up formed game `1`, all four clients opened RakNet, and Four, Rawr, and Three published mesh-ready status before every process exited with access violation `0xc0000005`. The exception address `0x00c3c53c` is inside the dispatcher registration called by `cBlazeGameManager::onMatchmakingSessionFinished`: result codes `0` through `2` require the callback's resolved game pointer, but Darkspin had followed setup with `NotifyMatchmakingFailed` carrying success result `2`, whose schema has no game identity and therefore supplied a null pointer. Notification `0x0a` remains limited to failure and cancellation. `[E]`

The 14:35 follow-up again formed game `1`, admitted all four RakNet players at mask `0xf`, and sent every client the initial campaign chain vote without a crash, but the Map Room remained in matchmaking. Darkspin's member `3` payload incorrectly used `FLGS`, `PID`, and `STAT`, which belong to another generated type; the setup decoder ignored them and retained `MSID=0`, so `cBlazeGameManager::onMatchmakingSessionFinished` could not match any active session. The corrected payload now carries each entrant's exact matchmaking session and terminal success result through `FIT`, `MAXF`, `MSID`, `RSLT`, and `USID`. `[E]`

The 14:51 follow-up formed game `1`, admitted all four RakNet peers at mask `0xf`, and each client socket trace received both the 353-byte campaign vote packet and its 20-byte countdown packet. Only Three left matchmaking and published presence status `5`. The generated `ReplicatedGamePlayer` and `ReplicatedGameData` contracts identify `PROS[].UID` as `PlayerSessionId` and `GAME.HSES` as `TopologyHostSessionId`, but Darkspin had filled both fields and `REAS.USID` with account IDs. Three alone had matching account and live Blaze session IDs (`6/6`), while Foo was `3/2`, Rawr was `2/4`, and Four was `7/8`. Matched setup now resolves the one active Blaze session for every entrant and uses those IDs consistently. `[E]`

The 15:14 follow-up again formed game `1`, admitted Foo, Rawr, Three, and Four at RakNet mask `0xf`, delivered the initial chain vote to all four, and ultimately started all four in the dungeon. Foo, Rawr, and Four published queue presence status `5`, while Three skipped that intermediate presentation despite later publishing preparation status `2` and participating normally. Three was also the first client to publish a mesh update. Darkspin was immediately broadcasting each first mesh reporter's connected and join-complete notifications while that client's matchmaking-finished callback could still be unwinding. Mesh updates now record readiness only; after every frozen member is ready, the elected start publication sends all four connected and join-complete transitions together before closing the barrier. `[E]`

The first failing trace sent each client one `NotifyGameSetup` whose `PROS` already contained Rawr and Foo, then sent two additional `NotifyPlayerJoining` records. Foo immediately requested removal, reported one mesh update, and never opened RakNet. Rawr then finalized game creation before its own mesh update; Darkspin treated finalize as readiness and published `IN_GAME` too early. Only Rawr opened RakNet, leaving the gameplay barrier at mask `0x1` of `0x3`. `[E]`

The 2026-08-27 11:14 trace removed the duplicate joining records and correctly reached Rawr's queue presentation, but Foo still requested removal with reason 6 in the same millisecond as receiving ordinary join-context setup. The 11:21 follow-up repeated that removal after Darkspin placed numeric value `2` inside dataless `DCTX`, proving that indirect admission requires union member `2` itself with its authored `GRID` and `RPVC` fields. Foo remained on the Campaign selector and never opened RakNet in both malformed exchanges. `[E]`

The corrected indirect union allowed both clients to open RakNet and reach connected mask `0x3` in the 11:31 trace. Immediately afterward Foo issued Playgroups `LeavePlaygroup`, then `CreatePlaygroup` carrying `ATTR.PlaygroupKey=1`; this is the client handoff from the ship party into the admitted game, not an authored membership departure. Mutating the party on that intermediate leave split Foo into a one-member replacement while Rawr retained the old roster. Darkspin now preserves active-game membership across that pair and returns the existing party snapshot to the recreate request. `[E]`

The 11:37 trace again merged both RakNet peers at mask `0x3`, but only Foo requested chain vote data. Darkspin responded with the queue countdown only to Foo, while Rawr continued polling without receiving the countdown or entering deployment; no dungeon baseline was published before both clients exited. Queue presentation and preparation are shared game decisions: the first accepted request now advances every current member, queues each member-specific vote/setup packet to non-requesting peers, and retains the direct response for the requester. `[E]`

The 11:44 trace merged both RakNet peers but neither requested vote data. Rawr, the host, sent `FinalizeGameCreation`; Foo sent the sole mesh-ready update. Darkspin retained only Foo in the readiness barrier because finalization contributed no host readiness, leaving the game permanently in `PRE_GAME`. Host finalization is the host's readiness signal in this exchange, while each non-host must still report independently; this completes the two-member barrier without restoring the earlier behavior that marked the entire roster ready from one host command. `[E]`

The 11:48 trace showed Rawr performing the preserved leave/recreate handoff. Returning only `INFO` retained authoritative membership but did not repopulate Rawr's cleared client-side member list, so Foo still displayed Rawr while Rawr displayed no Foo. Existing-party recreation now sends the requester the same full `NotifyJoinPlaygroup` snapshot used by an ordinary join, including `MLST`, without notifying or rebuilding members whose client roster remains intact. `[E]`

After one member selected Continue in the same run, both peers received member-specific prepare packets and independently reported status `2`, then status `4` with progress `1`. Neither reported status `8`; the loading screen displayed Rawr at 100% and an unnamed second row at 0%. Darkspin had echoed each full identity/status packet only to its originating peer. Statuses `2`, `4`, and `8` are now also queued to every other member in the game, while the sender retains its direct response, so each client can resolve every row and observe the shared loading barrier. `[E]`

The 12:05 native run reached both dungeon baselines. Object snapshots consistently placed slot 0 at `(-122.597,-154.484,10.088)` and slot 1 at `(-118.372,-155.755,10.088)`, confirming that the authored per-slot entry positions were already distinct and near one another. The later slot-1 baseline included slot 0's objects `1..3`, but the earlier slot-0 client never received slot 1's objects `4..6`; it learned the remote hero only after movement, producing the apparent center-spawn collision and correction. A member completing its baseline now queues its complete roster snapshot directly to every exact-zone peer already in the dungeon. Hero swap fanout also includes combatant, attribute, visibility, warp, and deploy state so peers cannot observe only fragments of a character transition. `[E]`

The corrected server exchange is: `[I]`

1. Send each initially admitted member one `NotifyGameSetup` containing the complete `PROS` roster in `ACTIVE_CONNECTING`; use dataless create context only for the host, indirect-join union member `2` with the playgroup object ID for server-reserved peers, and do not repeat those initial players with `NotifyPlayerJoining`.
2. Treat `FinalizeGameCreation` as the `INITIALIZING -> PRE_GAME` transition and the host's own readiness signal, never as readiness for the remaining roster.
3. On a member's first valid mesh update, record that member as ready without publishing a partial player transition while its matchmaking-finished callback may still be active.
4. After every frozen member has crossed that boundary, broadcast `IN_GAME`, then each member's `ACTIVE_CONNECTED` state and `NotifyPlayerJoinCompleted`; repeat status publication is harmless, but never repeat join completion.
5. Treat chain vote-data and start commands as shared game transitions: publish the countdown and member-specific prepare packet to every admitted RakNet peer even when only one client emits the command.
6. Reject a non-host `DestroyGame` with `GAMEMANAGER_ERR_PERMISSION_DENIED` instead of returning success, because a successful response tells the requesting SDK that destruction succeeded.

For a player admitted after setup, existing clients still receive one `NotifyPlayerJoining`; the newly admitted client receives a complete `NotifyGameSetup` and must not receive a duplicate self-joining record. `[I]`

## Next native-run checks

- Both clients log one local and one remote joining/completion lifecycle, with Rawr slot `0` and Foo slot `1`.
- Foo receives `INDIRECT_JOIN_GAME_FROM_QUEUE_SETUP_CONTEXT`, leaves the Campaign selector, and does not request removal with reason 6.
- Foo opens a RakNet Hello and the server reaches connected mask `0x3` before `PartyMergeComplete`.
- `PRE_GAME` is observed before either `IN_GAME` notification, and `IN_GAME` occurs only after both mesh updates.
- Both clients receive the same game ID, host, level, expected player count, and roster before the first dungeon baseline.
