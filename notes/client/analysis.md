# Game 5.3.0.103: first static-analysis pass

## Scope and evidence standard

These findings apply only to `bin/GameBin/Game.exe` with SHA-256:

```text
3C7248A4C6614450290A66C22AB7FCDA86052B29A7B555E834D7C23F9B73EE5B
```

An item is called confirmed here when it is directly observable in the PE
headers, version resources, strings, or x86 instructions. A function name is
only assigned when the behavior is distinguishable from the disassembly.

## Binary identity

| Property | Value |
| --- | --- |
| File size | 18,357,008 bytes |
| Format | PE32, Intel i386, Windows GUI |
| Image base | `0x00400000` |
| Entry point | `0x00CFFB1F` |
| Linker | Microsoft linker 9.0 |
| PE timestamp | 2011-06-09 18:05:31 (raw PE timestamp interpretation) |
| File version | `5.3.0.103` |
| Product version | `5.3.0.103` |
| Original filename | `Game.exe` |
| PDB path | `c:\BuildAgent\cm-spore028-spore\CMBuild\SporeLabs_PROD\out\Win32\Release\Game.pdb` |

The adjacent `version_bin.txt` also contains `5.3.0.103`. This is a decisive
version mismatch with the migrated address notes, which say they target
5.3.0.127. There is no safe global address delta between those notes and this
binary.

## HTTP API surface present in the executable

The following null-terminated method strings are present in `.rdata`:

```text
api.test.null
api.game.exitGame
api.account.logout
api.survey.getSurveyList
api.config.getConfigs
api.account.auth
api.account.getAccount
api.status.getBroadcastList
api.creature.resetCreature
api.game.getGame
api.game.getRandomGame
api.inventory.getPartList
api.game.getReplay
api.status.getStatus
api.account.setNewPlayerStats
api.account.setSettings
api.creature.unlockCreature
api.creature.updateCreature
api.deck.updateDecks
api.inventory.updatePartStatus
api.account.unlock
api.inventory.vendorParts
```

This confirms those method identifiers from the legacy HTTP API note for this
build. It does not by itself prove the complete request or response schema.

### darkspin implementation mapping

The status below describes current server behavior, not merely whether a
method name appears in a switch. **Implemented** means there is dedicated,
meaningful behavior. **Partial** means darkspin returns the expected general
response shape but some validation, mutation, or protocol semantics are still
placeholders. **Stub** means the method only receives a generic success path.

| Confirmed client method | darkspin status | Current behavior |
| --- | --- | --- |
| `api.config.getConfigs` | Partial | `/bootstrap/api` returns host, Blaze, launcher, settings, and optional patch configuration XML, but omits client-recognized service-port fields. |
| `api.account.auth` | Partial | Resolves an existing user from a token/key/cookie and returns account XML; it does not perform EA/Nucleus credential authentication. |
| `api.account.getAccount` | Partial | Returns account state and optionally creatures and decks, but omits several client-recognized account fields. |
| `api.account.logout` | Implemented | Invalidates the current user session through the SporeNet manager. |
| `api.status.getStatus` | Partial | Returns the expected service-health structure with fixed healthy values. |
| `api.status.getBroadcastList` | Partial | Returns the expected broadcast structure with a fixed placeholder broadcast. |
| `api.inventory.getPartList` | Partial | Returns persisted parts but omits several fields recognized by the client part parser. |
| `api.creature.unlockCreature` | Partial | The handler mutates state, but it reads `noun_id` while this client sends `template_id`. |
| `api.survey.getSurveyList` | Partial | `/survey/api` returns a successful but empty survey list. |
| `api.inventory.vendorParts` | Partial | Authenticated requests are acknowledged; requested inventory mutations are not applied. |
| `api.inventory.updatePartStatus` | Partial | Authenticated requests are acknowledged; requested status changes are not applied. |
| `api.creature.resetCreature` | Partial | Acknowledges the request without mutating persisted creature state; the exact retail reset transaction remains unresolved. |
| `api.creature.updateCreature` | Partial | Saves the current user and acknowledges success without applying creature fields. |
| `api.deck.updateDecks` | Partial | Saves the current user and acknowledges success without applying deck fields. |
| `api.account.unlock` | Implemented | Applies and persists the recovered upgrade, then returns updated account fields directly under `response` as required by the native unlock completion handler. |
| `api.account.setSettings` | Partial | Saves the current user and acknowledges success without applying settings fields. |
| `api.game.getGame` | Partial | Returns success with a fixed game ID. |
| `api.game.getRandomGame` | Partial | Returns success with a fixed game ID. |
| `api.game.exitGame` | Partial | Returns success with a fixed game ID; no game-session transition is performed. |
| `api.account.setNewPlayerStats` | Stub | Falls through to the generic success envelope. |
| `api.game.getReplay` | Stub | Falls through to the generic success envelope. |
| `api.test.null` | Stub | Receives endpoint-level compatibility behavior rather than a dedicated method implementation. |

The confirmed paths `/bootstrap/api`, `/game/api`, and `/survey/api` are all
registered by `game.RegisterAPI`. The entire `/web/sporelabsgame/` prefix is
routed to static-file serving, but the bundled darkspin assets currently contain
only `announceen` and `register`; the other confirmed client paths will return
not found unless more assets are installed. The `/web/sporelabs/` prefix used
by `home`, `stats`, `alerts`, and `resetpassword` is not currently registered.

The following legacy-note identifiers were not found as exact ASCII strings in
this executable:

```text
api.creature.getCreature
api.creature.getCreaturePng
api.inventory.purchasePart
api.inventory.flairPart
api.inventory.vendorPart
/game/service/png
/web/sporelabsgame/announceen
```

Absence of a literal is not proof that a capability is absent; a string may be
constructed dynamically or the note may describe another build.

Confirmed endpoint and host literals include:

| File offset | VA | Literal |
| --- | --- | --- |
| `0x00BD6C5C` | `0x00FD865C` | `http://config.game.com/bootstrap/api?version=1` |
| `0x00BD7CA8` | `0x00FD96A8` | `/survey/api?version=%u` |
| `0x00BD7CE0` | `0x00FD96E0` | `/game/api?version=%u` |
| `0x00BD7C4C` | `0x00FD964C` | `content.game.com` |
| `0x00BD7C64` | `0x00FD9664` | `api.game.com` |
| `0x00CD178C` | `0x010D318C` | `gosredirector.online.ea.com` |

There are code references to the bootstrap URL at `0x004A74EE`, to
`api.account.auth` at `0x004A7879`, to `api.status.getStatus` at
`0x004A8861`, and to the `/game/api?version=%u` format at `0x004B595E`.
These references show the literals are part of executable code paths rather
than unreferenced padding.

## Confirmed HTTP request contracts

Game constructs requests from a method name plus a list of key/value
parameters. The common serialization path at approximately `0x004A4D30`
contains the formats `&method=%s`, `&token=%s`, and `&%s=%s`. The table below
comes from method and parameter string references inside the 5.3.0.103 request
constructors, not solely from the older decompilation.

| Constructor VA | Method | Confirmed method-specific parameters |
| --- | --- | --- |
| `0x004A5550` | `api.game.exitGame` | None observed. |
| `0x004A5610` | `api.account.logout` | None observed. |
| `0x004A59E0` | `api.survey.getSurveyList` | None observed. |
| `0x004A7480` | `api.config.getConfigs` | `build=5.3.0.103`, `include_patches=true`, `include_settings=true`; a conditional path also adds `active=false`. |
| `0x004A7810` | `api.account.auth` | `key` formatted as `%s::%s` or `%s::0`, `cookie=true`, optionally `include_creatures=true`, `include_decks=true`, `include_feed=true`, and `include_settings=true`, plus `build=5.3.0.103`. |
| `0x004A7CF0` | `api.account.getAccount` | Decimal 64-bit `id`; optionally `include_decks=true`. |
| `0x004A7F60` | `api.status.getBroadcastList` | `build=5.3.0.103`. |
| `0x004A8080` | `api.creature.resetCreature` | Decimal 64-bit `id`. |
| `0x004A8210` | `api.game.getGame` | Decimal 64-bit `game_id`. |
| `0x004A8210` | `api.game.getRandomGame` | Integer `replay_version`. |
| `0x004A8410` | `api.inventory.getPartList` | `count=10000`; optional `filter` containing `creature_id-%I64d;` and/or `market_status_full-owned;`. |
| `0x004A8670` | `api.game.getReplay` | Decimal 64-bit `round_id`. |
| `0x004A8800` | `api.status.getStatus` | `build=5.3.0.103`, `include_broadcasts=true`. |
| `0x004A8DB0` | `api.account.setNewPlayerStats` | Optional unsigned `new_player_progress` and `new_player_inventory`. |
| `0x004A8F60` | `api.account.setSettings` | `settings`, encoded as repeated `%s,%s;` pairs. |
| `0x004A90B0` | `api.creature.unlockCreature` | Unsigned `template_id`. |
| `0x004A9230` | `api.creature.updateCreature` | `id`, `cost`, `gear`, `points`, `stats`, `stats_ability_keyvalues`, `parts`, `thumb`, `thumb_crc`, `large`, and `large_crc`. `gear` and `points` use `%.3f`. |
| `0x004AA040` | `api.deck.updateDecks` | `pve_creatures`, `pvp_creatures`, `pve_active_slot`, and `pvp_active_slot`. Each creature field is a fixed nine-position sequence containing three consecutive three-hero deck records; IDs are comma-separated decimal 64-bit values and zero preserves an empty position. |
| `0x004AA4B0` | `api.inventory.updatePartStatus` | `status`, `part_id`, and optional `operator`. Part IDs are comma-separated decimal 64-bit values. |
| `0x004AA800` | `api.account.unlock` | Unsigned `unlock_id`. |
| `0x004AA980` | `api.inventory.vendorParts` | `transactions`; entries use `%c%I64d` and are separated by semicolons. |

The older 5.3.0.127 decompilation suggested most of these same contracts, but
the executable establishes two important build-specific differences:

- The 5.3.0.103 part-list constructor formats `count` with the literal value
  `0x2710` (10,000), whereas the old decompilation showed 16.
- The 5.3.0.103 account-auth constructor does not reference
  `include_server_tuning`; that parameter appears in the 5.3.0.127
  decompilation.
- The 5.3.0.103 unlock completion handler at `0x004AE250` passes the
  `<response>` element itself to the account parser, unlike authentication and
  profile handlers that descend through `<response><account>`. Upgrade replies
  must therefore expose the updated account fields directly under `<response>`.

## Confirmed HTTP response vocabulary

The client contains separate XML parser routines whose string references expose
the response fields it recognizes. A field being recognized does not prove it
is mandatory; absent fields may simply retain a default value.

| Parser VA | Response area | Recognized XML elements |
| --- | --- | --- |
| `0x004AB2A0` | Common envelope | `response`, `stat`, `code`, `result`. |
| `0x004AB910` | Account scalar data | `tutorial_completed`, `chain_progression`, `creature_rewards`, `current_game_id`, `current_playgroup_id`, `default_deck_pve_id`, `default_deck_pvp_id`, `level`, `avatar_id`, `id`, `token`, `new_player_inventory`, `new_player_progress`, `cashout_bonus_time`, `star_level`, all progression `unlock_*` fields, `upsell`, `xp`, `grant_all_access`, `cap_level`, and `cap_progression`. |
| `0x004AC4A0` | Creature | `id`, `name`, `png_thumb_url`, `noun_id`, `version`, `gear_score`, `item_points`; adjacent creature processing also references `metadata`, `message_id`, and `account_id`. |
| `0x004ACC60` | Inventory part | `is_flair`, `cost`, `creature_id`, `id`, `level`, `market_status`, `prefix_asset_id`, `prefix_secondary_asset_id`, `rarity`, `reference_id`, `rigblock_asset_id`, `status`, `suffix_asset_id`, `usage`, and `creation_date`. |
| `0x004AE600` | Survey list | `response`, `surveys`, then survey `id`, `trigger1`, and `trigger2`. |
| `0x004AECB0` | Broadcast | `id`, `start`, `type`, `message`, and `tokens`. |
| `0x004AFCF0` | Account aggregate | `response`, `account`, `decks`, `creatures`, `feed`, and `items`. |
| `0x004B0710` | Service status | `response`, `status`, `api`, `blaze`, `gms`, `nucleus`, `game`, `health`, `countdown`, `open`, `throttle`, `vip`, `revision`, `version`, `db_version`, and `broadcasts`. |
| `0x004B2880` | Bootstrap configuration | `response`, `configs`, `config`, `settings`, `open`, `telemetry-setting`, `telemetry-rate`, `blaze_service_name`, `blaze_secure`, `blaze_env`, `sporenet_host`, `sporenet_port`, `sporenet_cdn_host`, `sporenet_cdn_port`, `liferay_host`, `liferay_port`, `http_secure`, `launcher_action`, `launcher_url`, `patches`, `to_image`, and `from_image`. Patch parsing additionally recognizes version, URL, size, hash, locale, and instruction metadata. |

### Actionable darkspin gaps exposed by these contracts

- `api.creature.unlockCreature` is not compatible with this client as written:
  the client sends `template_id`, but `game.API.game` reads `noun_id`.
- `partNode` emits only a subset of fields already available on
  `sporenet.Part`. It omits `is_flair`, `creature_id`, `market_status`,
  `prefix_secondary_asset_id`, `usage`, and `creation_date`, among others that
  the parser recognizes.
- `accountResponse` omits many progression fields already present on
  `sporenet.Account`, including the new-player, cap, star, and `unlock_*`
  values. It also does not emit the recognized `token` element, relying on the
  requested cookie instead.
- The account-auth request can ask for `include_feed` and `include_settings`;
  darkspin currently honors only creature/deck inclusion.
- The bootstrap response omits `sporenet_port`, `sporenet_cdn_port`, and
  `liferay_port`. This matters when the replacement service is not on the
  client's default HTTP port.
- `api.inventory.getPartList` ignores `count` and `filter`; returning all parts
  is usable for small inventories but does not implement the confirmed filter
  contract.
- The mutation stubs now have confirmed input keys, so creature, deck, part,
  account-unlock, settings, and vendor handlers can be implemented without
  guessing their top-level parameter names.

## Spore Labs URL initialization

The legacy 5.3.0.127 note names `sub_459BF0` as
`storeSporelabsUrlsForLaterUse`. The homologous routine in this 5.3.0.103
binary begins at **`0x004A10E0`**.

Evidence:

- At `0x004A10F8`, it pushes the `/web/sporelabsgame/register` literal.
- It then supplies host/path hash constants `0x08ABBD29` and `0x0AC68866`.
- At `0x004A1107`, it tests its Boolean argument.
- The true branch calls `0x00717FE0`; the false branch calls `0x007180A0`.
- The routine continues with the same ordered Spore Labs path set recorded in
  the legacy decompilation, including `persona`, `finish`, `resetpassword`,
  `alerts`, `stats`, `home`, `squad`, `inventory`, `store`, `friends`, `lobby`,
  `profile`, `leaderboards`, `wiki`, and `creatureprofile`.

The two destination helpers are also distinguishable:

| VA | Confirmed behavior |
| --- | --- |
| `0x00717FE0` | Combines a host and slash-prefixed path with the format string `https://%hs%hs`, then stores the result in the URL table. |
| `0x007180A0` | Combines a host and slash-prefixed path with the format string `http://%hs%hs`, then stores the result in the URL table. |

This confirms the intent of the legacy HTTPS/HTTP URL-storage notes while also
providing the correct addresses for 5.3.0.103. The Boolean flag selects HTTPS
for the `register`, `resetpassword`, and `store` entries; their false branches
use HTTP. The other listed Spore Labs paths use the HTTP helper unconditionally.
This differs from the migrated 5.3.0.127 decompilation, where `resetpassword`
is shown as unconditionally HTTP.

## Embedded networking and crypto components

The executable contains direct build-path/version evidence for these compiled
components:

| Component | Evidence |
| --- | --- |
| RakNet | Numerous `Core\RakNet\Source` paths, RakNet container headers, `cTransportRakNet`, and `sntransport_transport_raknet.cpp` strings. |
| BlazeSDK | Paths containing `BlazeSDK\3.09.03.1`, plus `snlobby_platform_blaze.cpp` and `snlobby_login_blaze.cpp`. |
| OpenSSL | `OpenSSL 0.9.8g 19 Oct 2007` and many matching `Core\OpenSSL\CurrentBuild` source paths. |
| EAWebKit/DirtySDK integration | The literal `EAWebKit/TransportHandlerDirtySDK`. |

This confirms the broad library classifications in the migrated notes, but not
their 5.3.0.127 function boundaries or individual function names.

### Confirmed gameplay-transport vocabulary

The 5.3.0.103 binary contains a connected-RakNet vocabulary in addition to the
offline open-connection exchange. The `.rdata` file-to-memory delta is
`0x00401A00`; the following are therefore build-specific virtual addresses:

| File offset | Virtual address | Literal |
| --- | --- | --- |
| `0x00C2D1BC` | `0x0102EBBC` | `ID_ALREADY_CONNECTED received?` |
| `0x00C2D224` | `0x0102EC24` | `Unexpected RakNet message: ID_REMOTE_NEW_INCOMING_CONNECTION` |
| `0x00C32700` | `0x01034100` | `ID_CONNECTION_REQUEST_ACCEPTED` |
| `0x00C32720` | `0x01034120` | `ID_CONNECTION_ATTEMPT_FAILED` |
| `0x00C32740` | `0x01034140` | `ID_ALREADY_CONNECTED` |
| `0x00C32758` | `0x01034158` | `ID_NEW_INCOMING_CONNECTION` |
| `0x00C32794` | `0x01034194` | `ID_DISCONNECTION_NOTIFICATION` |
| `0x00C327B4` | `0x010341B4` | `ID_CONNECTION_LOST` |
| `0x00C32810` | `0x01034210` | `ID_INCOMPATIBLE_PROTOCOL_VERSION` |

The connection-message names are installed into a contiguous lookup table by
code at `0x00AADCFB` onward. The transport-specific strings around
`0x0102EBBC`-`0x0102EC24` are associated with `cTransportRakNet`, rather than
being only generic strings compiled into the embedded RakNet library.

Game's gameplay layer also exposes these handler/logging literals:

| File offset | Virtual address | Literal |
| --- | --- | --- |
| `0x00C2E0D0` | `0x0102FAD0` | `OnGmsHelloPlayer [type=%d][gameplayIndex=%u][address=%s][port=%u]` |
| `0x00C2E130` | `0x0102FB30` | `OnGmsPlayerJoined [gameplayIndex=%u]` |
| `0x00C2E2B0` | `0x0102FCB0` | `kGmsHelloPlayer` |
| `0x00C2E2C0` | `0x0102FCC0` | `kGmsReconnectPlayer` |
| `0x00C2E2F0` | `0x0102FCF0` | `kGmsPlayerJoined` |
| `0x00C2E304` | `0x0102FD04` | `kGmsPartyMergeComplete` |
| `0x00C2E390` | `0x0102FD90` | `kGmsObjectCreate` |
| `0x00C2E500` | `0x0102FF00` | `kGmsActionCommandMsgs` |
| `0x00C2E604` | `0x01030004` | `kGmsActionCommandResponse` |
| `0x00C2E6E8` | `0x010300E8` | `kGmsGameStart` |

`OnGmsHelloPlayer` is referenced at `0x00A8EBE7` in the function beginning at
`0x00A8EAB0`; the disassembly supplies the decoded type, gameplay index,
address, and port to that format string. `OnGmsPlayerJoined` is referenced at
`0x00A8ED02` in the following function at `0x00A8ECB0`.

This confirms that a gameplay server must account for connected RakNet state
and a Game hello/join layer. It does not yet confirm the wire layout of the
hello payload, the reliability class of each gameplay message, or the complete
handshake sequence.

### darkspin implementation mapping

| Confirmed client component | darkspin counterpart |
| --- | --- |
| RakNet | Partially implemented in `server/raknet/`: bitstreams, offline negotiation, reliability datagram and ACK/NACK codecs, game state, and action-command codecs. `server.Server` does not start this UDP server, and `raknet.Server` currently dispatches application IDs directly after the offline exchange instead of maintaining connected RakNet peers. |
| BlazeSDK | Implemented in `server/blaze/`: framing, recursive TDF, component registry, redirector, utility, authentication, rooms, messaging, playgroups, and game-manager handlers. This is a compatible server implementation, not the original SDK code. |
| OpenSSL 0.9.8g | Not reimplemented. Optional Blaze encryption uses Go's `crypto/tls`; the obsolete embedded OpenSSL/SSLv3/RC4 stack is intentionally not carried over. |
| EAWebKit/DirtySDK transport | Not implemented as a transport library. darkspin serves launcher and Spore Labs web assets consumed by the existing client. |

darkspin detects the client version from `GameBin/version_bin.txt` when a
game path is supplied. Without that file/path it retains the legacy
`5.3.0.127` fallback. The executable analyzed here and its adjacent version
file both report `5.3.0.103`.

### Startup data-directory preflight

The 5.3.0.103 executable has a data-directory failure path that uses alert code
`1004`. Its message begins at file offset `0x00BDD6B0` (virtual address
`0x00FDF0B0`): `The game cannot find data files necessary to run`, followed by
the reinstall suggestion at `0x00BDD6E2`.

The function beginning at `0x0050F6B0` resolves the data directory and calls
the directory check at `0x0050F813`. Its failure branch pushes severity `2`,
error number `1004`, and message address `0x00FDF0B0` at
`0x0050F81F`-`0x0050F82B`, then returns false. Nearby wide-string literals
confirm that resolution considers the `InstallLoc` registry value, a `Data`
directory, and the current/installation path. A normal client layout places
`Data/version_data.txt` beside `GameBin`.

Code `1004` is not unique to this path. Static caller analysis also finds it in
default-preferences, needed-package, and generic alert construction paths. The
live numeric startup modal observed during tracing did not pass through the
instrumented direct callers of the internal alert function, so its exact
origin remains unconfirmed. It must not be diagnosed as the data-directory
failure from the number alone.

The data check itself runs locally before the client begins the gameplay login
flow. darkspin checks the selected installation for `Data/version_data.txt`
before injection and explains how to select a complete installation.

The PE imports include `WS2_32.dll`, `IPHLPAPI.DLL`, `ADVAPI32.dll`,
`PhysXLoader.dll`, `d3d9.dll`, and `DSOUND.dll`. OpenSSL, RakNet, and Blaze are
represented by compiled-in code/build strings rather than separate imported
DLLs in this file.

## Blaze transport security and local compatibility

The ProtoSSL connection function begins at virtual address `0x00E47340`. At
`0x00E4736D`, `mov ebx,[esp+0x14]` loads the caller's secure-connection flag;
that value is passed into the lower connection path at `0x00E47392`. These
addresses are specific to the 5.3.0.103 executable identified above.

A live unmodified connection to darkspin's redirector sent a 52-byte legacy
secure handshake rather than a Blaze frame. This is consistent with the old
server implementation's SSLv3 and RC4 cipher configuration, neither of which
is suitable for the Go server. For local compatibility, `fang.dll`
signature-matches the 5.3.0.103 instruction sequence and replaces the secure
flag load with `xor ebx,ebx` followed by two no-ops. The redirector response
also advertises `SECU=0`.

With that patch applied, the same client produced these decoded plaintext
interactions in sequence:

| Listener | Component/command | Observation |
| --- | --- | --- |
| Redirector | `5/1` | 165-byte request and 47-byte response |
| Main Blaze | `9/7` | Utility pre-auth request and 208-byte response |
| Main Blaze | `9/2` | Utility ping |

This confirms that the secure flag controls the legacy transport layer for
this path and that the client accepts plaintext Blaze plus `SECU=0` for a
locally injected session. It does not establish that plaintext is appropriate
for a public deployment or for other executable builds.

The redirector and main service were then advertised on the same TCP port,
42127. A live run connected for `5/1`, disconnected, reconnected to that same
port for `9/7`, and continued through authentication. This dynamically
confirms that one listener can serve both roles for this client build; darkspin
now creates only one listener for coincident Blaze ports.

## Live authentication and session bootstrap

A redacted paired trace against a locally registered account confirmed this
post-redirector sequence. Field values were excluded; only labels, TDF types,
and empty markers were recorded.

| Component/command | Direction | Confirmed observation |
| --- | --- | --- |
| `1/0x28` | Client request | Authentication login sends `DVID`, `MAIL`, `PASS`, `TOKN`, and `TYPE`. Empty UI inputs produce empty `MAIL`/`PASS` and error `0x000b`; populated inputs receive a successful login response. |
| `1/0x6e` | Client request | LoginPersona sends `PNAM`. The response is followed by UserSessions `UserAdded` (`0x7802/2`) and `UserUpdated` (`0x7802/5`) notifications. |
| `9/8` | Client request | Utility PostAuth requests the `PSS`, `TELE`, `TICK`, and `UROP` service structures. |
| `0x7802/0x14` | Client request | UpdateNetworkInfo sends `ADDR`, `NLMP`, and `NQOS`. |
| `1/0x24` | Client request | GetAuthToken follows session establishment. |
| `0x7802/0x19` | Client request | UpdateUserSessionClientData contains two `CVAR` fields: an integer list and a struct. The wire payload ends with short zero terminal padding. |

The original recap server also sends UserAdded and UserUpdated from its
LoginPersona handler. Adding the same notifications to darkspin caused the
client to advance immediately into network, token, association, room/view,
category, and presence initialization. Accepting only all-zero terminal TDF
padding resolved the next compatibility failure while retaining rejection of
nonzero truncated data.

After those changes, the traced client received no Blaze error replies and
advanced from the login shell into a rendered 3D game scene. This establishes
a working authentication/session bootstrap for 5.3.0.103. It does not yet
establish multiplayer matchmaking or connected RakNet gameplay.

The injected client callback at return RVA `0x83ff40` is reached on a failed
Blaze login and corresponds to the static `Blaze login failed` path. Recording
this callback alongside the paired frame trace distinguishes a client-side
login-state rejection from an HTTP or transport failure.

## Opening-movie control

The pre-game `SINGLEPLAYER / MULTIPLAYER / EXIT` screen is the EAWebKit
bootstrap page. Its enabled Multiplayer button calls the same
`Client.playCurrentApp()` bridge method as the legacy Singleplayer button. The
native dispatcher at VA `0x0050FD60` handles that method by setting the launcher
bridge result at `+0x1C` to `1` and posting `WM_QUIT` to the bridge window at
`+0x10`. The bridge initialization routine is VA `0x0050E7D0`, called from VA
`0x00510BE6` before the bootstrap window is shown. fang.dll hooks that call,
retains its initialization, and applies the native play result immediately.
This default behavior is independent of cinematic skipping.

The 5.3.0.103 executable's `SP_SporeLabs/cBootState` update routine begins at
VA `0x00448D80`. At VA `0x00448DAD` (RVA `0x00048DAD`) it resolves the
`playOpeningMovie` boolean and tests the result at VA `0x00448DC6`. The
conditional branch at RVA `0x00048DC8` skips directly to the next boot state
when that value is false.

`launch --skip-cinematic` now opts into signature-checked, process-memory-only
change of that conditional jump to an unconditional jump. The executable on
disk is not modified, normal launches keep the original flow, and hook
initialization fails rather than patching if the expected instruction sequence
is absent.

A live injected run confirmed that this branch skips the EA and Maxis opening
movies and reaches login in seconds.

The post-login sequence belongs to `SP_SporeLabs/cSpaceshipState`. Its entry
routine begins at VA `0x004552C0` and retains the normal scripted-scene
initialization. The transition manager selects the cinematic resource and, at
VA `0x0052CACD`, calls its playback routine before recording the new transition
state. The spaceship update routine begins at VA `0x00455520`. The checks
ending at VA `0x00455581` and
`0x0045558A` hold that state until the sequence is ready to complete, after
which the normal path emits screen event `0x1002`.

Live tests establish that the surrounding 3D cinematic cannot be skipped by
suppressing the transition playback call: the call has required initialization
side effects, and removing it leaves the client on a black transition. Bypassing
the two readiness branches also leaves the state incomplete. Overwriting the
transition timer scale drives the evaluator to its 99.7 percent clamp but does
not complete cleanup.

The normal spline evaluator is VA `0x00A1AFE0`; the spaceship frame update calls
it at VA `0x004F30D8` (RVA `0x000F30D8`). Live tracing distinguishes the initial
three-node, 4-second transition from the actual ship tour, which is a 14-node,
23.75-second spline. Game's manual Escape/X completion routine is VA
`0x00A1B840`. It advances the active spline, dispatches skip-safe messages, and
returns the completed spline object. Its caller at VA `0x0052C20C` then performs
required work beginning at VA `0x0052C211`: it applies the spline's final
transform through VA `0x0052F0E0` and finishes the movie manager through vtable
slot `+0x20`. Calling `0x00A1B840` without that continuation leaves the renderer
on the sequence's white transition frame.

`launch --skip-cinematic` hooks the frame-update call and recognizes the ship tour
by its build-specific node-count and duration signature. It invokes the native
completion routine, reproduces the final-camera and movie-manager continuation,
and leaves shorter camera transitions untouched. A live traced login recorded
`spline_skip`, immediately rendered the bridge's `START` prompt, and remained
responsive. Selecting `START` then completed a separate six-node, 2.75-second
transition and reached the squad/hero screen normally.

The `START` action is the Flash callback
`SpaceshipNavigation.OnGoToRoomClicked` at VA `0x0052CFD0`. Its numeric room
argument selects room `1`, the Arsenal/squad screen, through the normal
transition manager. The spaceship state emits screen event `0x1002` only after
its scene and UI readiness checks pass and its navigation controller has been
initialized. `launch --skip-cinematic` records that event and invokes the START
callback from the following navigation update. This preserves the native
room-selection and progression path without waiting at the prompt. The spline
hook then completes the resulting six-node, 2.75-second transition.

A current established-profile trace also records a distinct five-node,
4-second spline before the START presentation. It does not share the 14-node
tour's manual completion continuation and must not be passed through that skip
path. Waiting for screen event `0x1002` avoids completing this spline directly.
A 45-second live test recorded `ship_navigation_ready`, `ship_start`, and
`ship_start_transition_skip` in order and left the game process running.

The first-run tutorial is a separate native path from room `1`. The account
parser at VA `0x004AB910` stores `tutorial_completed`, but the actual game-mode
selector is `new_player_progress`. `MapRoomUI.StartGame` is registered at VA
`0x00518340` and implemented at VA `0x00516100`; progress values through `2000`
construct game mode `1` (`Tutorial`) and submit it through the normal game
service. The first tutorial level remains
`Game_Tutorial_cryos_1_v2`, matching the server level table.

The completion-side gameplay callback is now bounded. Build 103 registers
`TutorialGameMsgs` (wire opcode `0xC8`, name VA `0x0103035C`) at
`0x00451C16-0x00451C36` to callback `sub_4510F0`. The callback reads a one-byte
subtype; subtype `0` calls `sub_4507C0`, which consumes exactly one signed
32-bit value. A positive value is stored as cumulative XP, mapped through the
threshold-vector function `sub_9CEA00` to a one-based account level, and moves
the in-memory `new_player_progress` to `3000`. A zero or negative value clears
the XP/level-like fields and returns progress to `2000`.

This leaf emits `LABS_TUTORIAL_COMPLETE` but does not call the account update
submission path. It confirms a server-authored in-memory XP/completion
snapshot, not persistence timing. RakNet framing/reliability, the relationship
to the `RETURN TO SHIP` action, and the HTTP/account refresh that makes the
state durable remain open. In particular, a compatible server must not send a
zero placeholder for successful completion.

The separate tutorial scripting registration table maps
`UnlockSecondCreature` to `sub_A050C0` at `0x00A0BF23-0x00A0BF2E`. That native
resolves a simulation/player object and increments field `+0x1378`, capped at
`2`; it does not invoke the generic account creature-unlock API. This confirms
a mission-local second-creature unlock mechanism but does not by itself prove
that Sage is persisted to the account.

The spaceship navigation setup at VA `0x0052CDD0` disables ordinary room
selection for progress `0`, `1000`, and `2000`. Consequently, synthesizing an
Arsenal room click for an incomplete account bypasses the intended tutorial
route and leaves the player in an invalid onboarding state. Auto-play must call
the native `MapRoomUI.StartGame` path for those progress values and reserve the
room-`1` callback for accounts whose effective progress is at least `3000`.
Spaceship entry emits `0x1003` before the readiness event. Fang uses
that event to complete the automatic-login watchdog and reset its per-visit
navigation and spline flags while preserving the process-wide START selection.
The watchdog allows 30 seconds for older machines to reach this boundary; an
attached 0.7.11 laptop report showed that the client did not issue
`api.account.auth` until roughly ten seconds after JWT submission, after the
former eight-second deadline had already expired. An Arsenal return from the creature editor
marks a return-only START request. After the next `0x1002` readiness event and
a short main-thread settling interval, the DLL invokes the room callback again
and skips the new six-node transition. The delay prevents the callback from
overlapping the uninitialized first frame of the five-node presentation.

The long film shown inside that 3D sequence is a distinct VP6 resource and can
be stopped safely. The spline message handler at VA `0x00531E70` recognizes its
`Movie` command, copies the VP6 filename, and calls the resource-key constructor
at VA `0x007ADD40` from VA `0x00531F64` (RVA `0x00131F64`). The global movie
manager getter is VA `0x007B6FD0`. Game's own boot-movie update routine
uses movie-manager vtable slot `+0x14` to test whether playback is active and
slot `+0x24` for its native skip operation.

`launch --skip-cinematic` hooks the resource-key call without suppressing it. For a
`.vp6` command it lets the original loader initialize normally, waits until
slot `+0x14` reports active playback, then invokes slot `+0x24`. A live traced
login recorded `movie_stop`, kept the client responsive, and allowed the ship
camera script to continue. At most the first frame can briefly appear on the
in-world monitor; the long VP6 playback does not continue.

Package inspection also confirms resource type `0x376840D7`, group `0`, for all
installed VP6 files. The global `intro` resource contains 313 timing units. The
shortest localized ship resource, `fmv_01_prologue_end` (instance
`0x807968E0`), contains 636; the remaining localized films contain 1,338 to
3,909. Substituting another packaged film therefore cannot provide an instant
skip. Substitution with the global `intro` key also proved incompatible with
the localized ship lookup and terminated the client, so that experiment is not
installed.

## Live QoS probe behavior

After Blaze utility pre-auth, the 5.3.0.103 client requests `/qos/qos` with a
local probe port (`prpt`) of 3659. When darkspin advertises a distinct local
server port of 3660, the client sends a 20-byte UDP version-1 packet from
`127.0.0.1:3659` to `127.0.0.1:3660`. The decoded header contains request ID 2
and version 1. darkspin's compatible response is 30 bytes and returns to the
client's source endpoint.

This distinct-port run confirms that `qosport` in the HTTP response is a
destination service port, not merely a value that must echo `prpt`. Binding
both endpoints to 3659 on one Windows host caused the server to receive its own
response repeatedly; that is a local topology collision, not a client protocol
requirement. The server now defaults to 3660 locally and rejects packets whose
source endpoint exactly matches its listener as a defensive invariant.

The client reaches the login UI after this exchange, but an alert `[1004]`
remains. No HTTP or Blaze response in the captured sequence carries that code,
so its trigger is still under investigation. The injected trace now includes
late-loaded EAWebKit imports and alert call sites to associate a future capture
with executable/module code before changing server behavior.

## Reproduction

The initial pass used GNU binutils available in the workspace:

```powershell
Get-FileHash bin\GameBin\Game.exe -Algorithm SHA256
(Get-Item bin\GameBin\Game.exe).VersionInfo
objdump -f -p bin\GameBin\Game.exe
objdump -h bin\GameBin\Game.exe
strings -a -t x bin\GameBin\Game.exe
objdump -d --start-address=0x4A10E0 --stop-address=0x4A1200 bin\GameBin\Game.exe
objdump -d --start-address=0x4A7480 --stop-address=0x4AAB00 bin\GameBin\Game.exe
objdump -d --start-address=0x4AB2A0 --stop-address=0x4B4450 bin\GameBin\Game.exe
objdump -d --start-address=0x717FE0 --stop-address=0x718160 bin\GameBin\Game.exe
objdump -d --start-address=0xA8E9A0 --stop-address=0xA8EE80 bin\GameBin\Game.exe
```
