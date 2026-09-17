# Game 5.3.0.103 chat protocol

## Scope

These findings apply to `bin/game/GameBin/Game.exe` version
`5.3.0.103`, SHA-256
`3C7248A4C6614450290A66C22AB7FCDA86052B29A7B555E834D7C23F9B73EE5B`.
They were confirmed from the executable and its IDA database, rather than
inferred solely from the older Resurrection Capsule server.

## Blaze surface

The client registers Blaze Messaging component `0x0F` and recognizes these
request commands:

| Command | ID |
| --- | --- |
| `sendMessage` | `0x01` |
| `fetchMessages` | `0x02` |
| `purgeMessages` | `0x03` |
| `touchMessages` | `0x04` |
| `getMessages` | `0x05` |

`sub_E207B0` at `0x00E207B0` maps exactly those five command IDs. The client
recognizes notification `NotifyMessage` as command `0x01` in `sub_E208B0`.
There is no `sendGlobalMessage` entry in this executable's command map.

The client also recognizes Messaging errors in `sub_E20810`:

| Wire error | Client error |
| --- | --- |
| `1` | `MESSAGING_ERR_UNKNOWN` (`0x1000F`) |
| `2` | `MESSAGING_ERR_MAX_ATTR_EXCEEDED` (`0x2000F`) |
| `3` | `MESSAGING_ERR_DATABASE` (`0x3000F`) |
| `4` | `MESSAGING_ERR_TARGET_NOT_FOUND` (`0x4000F`) |
| `5` | `MESSAGING_ERR_TARGET_TYPE_INVALID` (`0x5000F`) |

The client combines the 16-bit wire error with component `0x0F` when naming
the full error.

## ClientMessage request contract

The Blaze SDK field visitor at `0x00E20900` confirms the complete request:

| TDF field | Wire type | Client object offset |
| --- | --- | --- |
| `ATTR` | map&lt;integer,string&gt; | `+0x2C` |
| `FLAG` | bitfield/integer | `+0x18` |
| `STAT` | integer | `+0x28` |
| `TAG` | integer | `+0x24` |
| `TARG` | object ID | `+0x08` |
| `TYPE` | integer | `+0x20` |

The player-chat send constructors at `0x00C33B10` and `0x00C342A0` both:

- set `TYPE=2`;
- put the entered body in `ATTR[0xFF02]`;
- set `TARG` to the selected Blaze object's object ID; and
- submit Messaging command `0x01` with the callback at `0x00C35120`.

This corrects an ambiguity in the reference server comments: values such as
7, 8, and 9 are UI channel selections, not `ClientMessage.TYPE` values for
ordinary player chat. `sub_C25490` switches over UI selections 7 through 11,
resolves an appropriate party/game/lobby object, and sends all of them through
the `TYPE=2` constructor.

The target object's component selects the audience:

| `TARG` component | Audience |
| --- | --- |
| `0x7802` UserSessions | direct player |
| `0x06` Playgroups | party/playgroup |
| `0x04` GameManager | active game |
| `0x15` Rooms | lobby room |

The current development server intentionally treats client-selected
GameManager "General" chat as server-global while cross-game lobby joining is
not yet exposed by the normal client flow. This is an explicit compatibility
policy, not a claim about retail match-chat scope; party, direct, and Rooms
targets retain their recovered audiences.

The client must retain ownership of ship chat routing. A temporary Fang bridge
prefixed ordinary text with `/game` (and, in an earlier revision, `/lobby`) to
carry Darkspin developer commands through the chat box. Runtime trace proved
that this intercepted every ship message before Messaging: the native converter
selected the forced route, but the ship had no corresponding GameManager target,
so it silently discarded the message. Forcing `/lobby` instead made unrelated
text inherit the native ship-location error.

Fang now leaves ordinary text and every native command unchanged. For a
recognized Darkspin developer command only, it temporarily replaces the
command's leading slash with the established `0x1F` bridge marker in the
caller's buffer, invokes the native converter, and immediately restores the
slash and original page protection. The in-place substitution is required: a
stack copy lets the converter recognize the hook invocation but does not retain
the command guard through its native routing path, producing the `/lobby`
ship-location error. The one-byte guard bypasses native unknown-command
rejection without overriding the client's selected Rooms, party, direct, or
game target. The server normalizes the marker back to `/` before developer-command
dispatch.

The authenticated `/dna <positive amount>` developer command adds to the
sender's durable account DNA with uint32 overflow protection. When invoked in
an active dungeon, the server also queues the native sparse DNA player update
so the current gameplay binding and client display adopt the new total.

The authenticated `/warp <location>` developer command queues a one-shot level
override for the sender's next newly launched campaign mission. Fang resolves
the packaged location list using an exact or unique case-insensitive partial
match and forwards the canonical level name; an ambiguous partial is rejected
by returning the numbered area catalog. Bare `/warp` returns the same compact
37-area catalog, and `/warp <1-37>` resolves through Fang to the corresponding
canonical special-purpose level before it reaches the server. The numeric
order follows the disconnected-area catalog and does not replace named access
to the complete packaged level list. Checkpoint resumes do not consume the
request. A solo launch applies its sender's request, while a matchmade launch
applies the warp only when every queued member selected the same level. Warped launches retain
the selected campaign chain as their setup context but do not claim normal
first-clear or chain progression for the disconnected destination.

The authenticated `/spawn <noun>` developer command is available only inside
an active campaign mission entered through `/warp`. Fang resolves its packaged
combat-noun catalog using an exact or unique case-insensitive partial match,
accepts names with or without the `.Noun` suffix, and forwards the exact
canonical noun. Gameplay validates the imported targetable non-player class,
creates it five world units beside the requesting hero, and publishes it
through the ordinary NPC spawn and action paths. Developer-spawned actors are
reward-suppressed and are not registered with campaign objectives, so zoo
experiments cannot grant loot or alter mission completion. Both the chat queue
adapter and gameplay execution binding reject the command outside a warped
zone.

The authenticated `/loc` developer command reads the requesting hero's
authoritative gameplay position and echoes X, Y, and Z to four decimal places.
It is read-only and is unavailable until the hero has entered a live dungeon
deployment or after the zone becomes terminal.

The authenticated `/recap` developer command restores every defeated reserve
squad member to full health while a living campaign hero remains deployed. It
does not reopen a terminal Game Over state or replace the normal hero-selection
lifecycle for a currently defeated deployed hero.

Post-mission result teardown can clear or stale the chat controller's selected
Rooms pointer while the Rooms SDK still owns the joined room. In that state
Fang recognizes a developer command but build 103 rejects channel 9 locally
with `509329563`, so no Messaging request reaches the server. Before converting
a recognized Darkspin command, Fang now copies the SDK's existing room pointer
back into the chat-controller slot. It does not synthesize a room, change
ordinary chat, or bypass server command validation.

The same projection repair now runs on every established ship navigation
frame, not only while converting a Darkspin developer command. This preserves
native `/lobby` after result teardown without manufacturing a Rooms object or
changing the selected room.

The Rooms `joinRoom` ordering is part of the chat contract. Build 103's join
callback immediately searches the Rooms SDK's view/category/room maps for the
returned room ID and stores that pointer as the lobby chat target. The recovered
server emits `NotifyRoomAdded` and `NotifyRoomUpdated` before the `joinRoom`
reply, then emits `NotifyRoomMemberJoined` afterward. Sending every notification
after the reply allows the RPC to succeed while the callback's cache lookup
fails, leaving the lobby pointer null; channel 9 then raises localization ID
`509329563` (`You must be on your ship to use /lobby`). Darkspin preserves this
specific pre-reply ordering while retaining post-reply delivery for the member
notification and ordinary Blaze commands.

Runtime evidence from established progress-9000 ship sessions showed no Rooms
RPC at all: login and inventory completed, but the client never invoked
`selectViewUpdates`. The accelerated ship path defers the native room-selection
UI callback and therefore also misses its call through online-platform wrapper
`sub_C28550` to Rooms-manager bootstrap `sub_C32950`. Fang now calls that wrapper
once, after authenticated ship navigation is ready and both the online platform
and its Rooms manager are present. This starts the client's own asynchronous
view -> category -> join sequence; Fang does not construct an SDK room, choose
a category, or replace the resulting lobby pointer. Client trace records
`rooms_bootstrap`, and Darkspinner writes decoded Blaze frames to
`logs/traces/server.jsonl` for direct verification of the sequence.

The decoded trace then confirmed a completely successful sequence on the wire:
view 1 was selected, categories were installed, `joinRoom` created a room, the
room add/update notifications preceded the reply, and member-joined followed.
The client could still leave its lobby pointer null because SDK notification
dispatch and the join callback are asynchronous even when their frames are
ordered. Fang therefore retries the callback's exact native nested lookup
(`RoomsAPI view map -> selected category -> category room map`) on later ship
frames. It assigns the manager's lobby pointer only after that lookup returns a
real SDK-owned room. `rooms_target_id` records the recovered room and
`rooms_target_ready` distinguishes this delayed repair (`1`) from a pointer the
native callback had already installed (`2`).

The packaged social player browser is `Web.package` ordinal 100, resource
`000100_dd6233d6_00000000_000000006a4e9288.bin`. Its `getcallback` parses the
HTTP `searchAccounts` response as XML, requires `<stat>ok</stat>` and `<total>`,
then reads each `<account>` through `account_id`, `name`, and `presence_level`.
It leaves the loading hourglass visible when `<total>` is absent because that
dereference occurs before the spinner is hidden. The Lobby tab is a separate
native path through `getLobbyMembersAsJSON`; Rooms member IDs therefore need
matching UserSessions identity notifications before the member-joined event so
the client can materialize each peer as an `OnlineId`/`Persona` JSON member.

The same packaged page defines search presence level `1` as `Web`, level `2`
as `Online - Lobby`, and level `4` as `Online - Navigation`. Darkspin had
hardcoded every active `searchAccounts` result to level `1`, so the page
reported `Foo is not online` even while Blaze presence correctly resolved Foo
in Navigation. `UserManager.Users` contains only active authenticated users;
search results now conservatively project those users at level `4` until the
HTTP surface consumes the exact live Blaze presence state. `[E]`

The 20:19 run enabled that online branch, but the inviting client exited before
sending `CreatePlaygroup`. Login-time reciprocal `NotifyUserAdded` had created
each peer before either client published its typed `PresenceInfo`; the later
presence broadcast sent only `UserSessionExtendedDataUpdate`. Party admission
previously repaired that owner with another complete `NotifyUserAdded`, but the
native invite now consumes it before a party exists. The first and subsequent
changed presence publications therefore refresh remote peers with the complete
typed owner and full-data update before sending the ordinary extended-data
delta. The publisher remains excluded to avoid the proven republish loop. `[I]`

The join response's `MDAT` describes only the joining membership; room updates
carry population counts but do not populate the SDK member vector. After peer
identity publication, the server must therefore replay `NotifyRoomMemberJoined`
for every pre-existing member to the joining client, while broadcasting the
new member normally to clients already in the room. The joining client's own
member event must precede those replays: sending an existing member first can
reach the SDK before it marks the local user as joined, causing the otherwise
valid peer event to be discarded.
Build 103's generated `JoinRoomResponse` visitor consumes fields in canonical
tag order: `CDAT`, `CRIT`, `MDAT`, `RDAT`, `VDAT`, then `VERS`. The Blaze TDF
reader is forward-only. A response encoded as `CRIT`, `VERS`, `CDAT`, `RDAT`,
`VDAT`, `MDAT` is legible to Darkspin's general trace decoder but causes the
generated client visitor to pass fields before it asks for them. In particular,
the SDK can retain default room/member structures, so the pre-reply room
notification creates a selectable room while its native member vector stays
empty. The server now emits the exact visitor order.
UserSessions and Rooms notifications are independently deferred inside the SDK,
so adjacent socket frames do not provide a cache-order guarantee. Persona login
therefore publishes every active peer in both directions before the client's
automatic room workflow begins; room admission republishes those identities as
an idempotent recovery projection before its member events.
Ordinal 100 also invokes `callSporeNet` before assigning `parent.getcallback`.
On loopback, the native request can finish before that assignment and discard
the only callback, leaving the hourglass visible despite a complete response.
An HTTP response delay does not repair this because native callback dispatch
has already captured the missing page function. Fang therefore allowlists only
this proven `getcallback` name and delivers its response through the existing
web-host timer used for other packaged page-registration races.

## Send reply

The callback at `0x00C35120` reads the returned message ID from the response
object at offset `+0x08` and logs
`SendMessageCb [msgId=%u][error=%u][jobId=%u]`.

The compatible reply contains:

- `MGID`: the first attribute key, normally `0xFF02`;
- `MIDS`: every submitted attribute key as an integer list.

## NotifyMessage contract

The `ServerMessage` constructor at `0x00E210A0` embeds a complete
`ClientMessage` at offset `+0x38` and stores a pointer to it at `+0x88`. Its
field visitor at `0x00E209B0` confirms:

| TDF field | Wire type |
| --- | --- |
| `FLAG` | bitfield/integer |
| `MGID` | integer |
| `NAME` | string |
| `PYLD` | `ClientMessage` structure |
| `SRCE` | object ID |
| `TIME` | integer |

`PYLD` is mandatory nesting. A flat notification containing `ATTR`, `TARG`,
and `TYPE` beside the outer fields does not match the client's class and will
not reach the chat UI correctly.

The notification dispatcher at `0x00C35170` reads `PYLD.TYPE` and handles
types 0, 1, and 2. Type 2 enters the chat handler at `0x00C35A10`. That handler:

1. checks the blocked-user list before dispatch;
2. reads the attribute map from `PYLD`;
3. prefers filtered body `ATTR[0xFF05]` when configured;
4. falls back to original body `ATTR[0xFF02]`;
5. logs `received chat message without a body` and drops the notification when
   neither attribute exists; and
6. forwards the body, `NAME`, and `SRCE` identity to the game UI event system.

The executable contains and registers `ShowLobbyChat`, `ShowPartyChat`,
`ShowFriendsChat`, `StartChatMessage`, `DisplayChatMessage`, and
`LABS_CHAT_MESSAGE`, confirming that the Blaze notification feeds the visible
Flash chat interface rather than an unused SDK facility.

## darkspin implementation

darkspin now implements player chat as a guarded `server/chat` feature. The
Blaze adapter decodes `ClientMessage`, maps the target component to a domain
audience, and asks the chat service to resolve recipients. The service rejects
unknown targets and group messages from non-members before any notification is
queued.

Accepted messages are delivered after the request reply to every active Blaze
session for:

- the direct recipient and sender;
- all current playgroup members;
- all players in the active game; or
- all users in the selected lobby room.

Party delivery is deliberately tied to the feature-owned transient party
membership; it never trusts the persisted account default as proof of active
membership. Create, invite, invited join, leave, leader promotion, destroy,
login reconciliation, and Playgroups projection now keep that membership live,
so direct, party, active-game, and joined-lobby audience resolution are wired.

The native `inviteToGroup` path creates a playgroup before it sends the first
Messaging invitation. `sub_C2D650` adds `PlaygroupKey` to the local playgroup
attribute map and passes the selected persona separately to `sub_E04E00`.
`sub_E04E00` constructs that second string directly at create-request offset
`+0x20`, which is `PGRP.UKEY`; `sub_E04660` then copies the local attribute map
into the same request. A build-103 capture proved that the broken path serialized
an empty `UKEY` and omitted the empty map. The later `143833686` create-success
event drains the selected-persona queue and invokes the native Messaging sender;
`144247916` only enumerates an already populated roster. Generated type evidence
identifies the structure consumed by `sub_E043F0` as `NotifyJoinPlaygroup`, not
the create response: the handler copies `INFO`, installs every `MLST` member, and
selects the local member by comparing that roster to `USER`. Returning those
fields in the create response cannot initialize the runtime group because the
generated response accepts only `INFO`. A new create therefore sends the full
`NotifyJoinPlaygroup` before its reply, while existing handoff recreation retains
its post-reply roster refresh. Both paths preserve canonicalized attributes,
`UPRS`, and recipient-specific network projection. The 21:02 follow-up proved
that build 103 received that complete notification before the create reply but
still emitted neither its queued UserSessions lookup nor its Messaging invite.
When this broken create shape has no `UKEY` or attributes and exactly one other
account is online, the target is unambiguous, so Darkspin now records and sends
the native type-1 invitation after the reply. It does not guess among multiple
online accounts, and the candidate Fang request-repair hook remains removed
rather than making a behavior-changing client patch part of the runtime. `[E]`

The build-103 direct-user dispatcher routes `ClientMessage.TYPE=1` to
`sub_C355A0`, which requires integer-string attribute `720897` as the sender's
playgroup ID. `TYPE=2` is the ordinary/general message path and does not present
an invitation even though the outer component-15 notification decodes. Native
and recovered party invitations must therefore use type 1; ordinary chat keeps
type 2.

The direct-message constructor `sub_E1F850` writes the target as UserSessions
object type `1`: `(0x7802, 1, personaID)`. Type `0` is not the native identity
contract even when the component and persona ID are correct. Its party-invite
constructor also sets `ClientMessage.FLAG=4`; recovered invitations preserve
both values so the receiver follows the same identity and invitation path as a
client-authored request.

Remote party portraits are not sourced directly from the browser image endpoint.
`sub_442010` passes the numeric avatar retained in the party player record to
`Party.swf` as `SetAvatarId`, while the remote player path requests that player's
public account profile. The profile-complete handler at `0x0051B520` first keys
the response back to the party member with the legacy account `id` at account
offset `+56`, then copies `avatar_id` from offset `+44` into the party record.
Darkspin's public projection carried `blaze_id` and `avatar_id` but omitted that
separate `id`, so the client downloaded the right thumbnail without associating
it with the member. Public account responses now carry both identity fields.
The packaged search callback also creates and caches account records, and a
later party insertion can reuse that cache without issuing another `getAccount`
request. Search results therefore include `avatar_id` and `level` so earlier
player discovery cannot leave the reused party record at avatar zero.
Playgroups member projections also retain
the recovered `(avatarID << 32) | 1` external identity, while login-time
UserSessions identity keeps its proven zero external ID because applying the
party encoding there stalls persona initialization before the account-profile
request.

`PlaygroupMemberInfo.SID` is the member's one-byte playgroup slot ID, not a user
session or persona identity. The exact visitor at `0x00E3BAF0` reads `SID` from
object offset `+144` as `uint8`; `PlaygroupInfo.HSID` is likewise a one-byte host
slot. A zero `SID` for every member can survive the joining client's full
`NotifyJoinPlaygroup` snapshot, but the inviter's incremental
`NotifyMemberJoinedPlaygroup` update collides with its local member and does not
insert the invitee. Slots are therefore the zero-based projection of stable
party join order, while nested `USER.ID` retains the member Blaze/persona ID.
The joining client's `JoinPlaygroupRequest.PNET` union is also part of the
member projection contract. Replacing it with an unset union in the inviter's
incremental notification leaves the new remote member without the network
identity it just published. Darkspin relays that exact client-authored union in
`NotifyMemberJoinedPlaygroup`. The request's `USER` structure must be retained
for the same reason. Darkspin preserves the joiner's canonical user envelope
while overriding its member ID, name, and party-specific portrait `EXID` with
server-authoritative values, and retains the exact client-authored network
union, join time, and slot.
Build 103 can still leave the creator's application roster unchanged after the
SDK accepts that incremental notification. Darkspin therefore sends the creator
the same recipient-specific full `NotifyJoinPlaygroup` snapshot that reliably
populates the accepting client, followed by `NotifyMemberJoinedPlaygroup` after
the member exists in the SDK roster. The full snapshot and following
incremental event must use the same member identity. Earlier builds mixed a
portrait-packed snapshot with the joiner's zero-EXID request identity, making
the creator issue an asynchronous UserSessions lookup after the application
callback had already been missed. Darkspin now uses the portrait-packed
Playgroups identity in both events. This ordering makes the authoritative
two-member roster converge before the application receives its incremental
member event and gives both peers the same avatar projection.

The 19:33 run delivered reciprocal UserSessions ownership before both roster
events, but Rawr still queried Foo through `LookupUser` immediately after
finalizing the playgroup and the later native owner query remained unresolved.
The lookup request carries the packed Playgroups `EXID`, while Darkspin had
answered it with the login-only zero `EXID`. The 19:43 follow-up confirmed the
packed lookup response was emitted. Party visibility and lookup now use one
server-authoritative packed avatar identity; login-time and room visibility
retain their proven zero external ID. `[E]`

The 19:02 Rawr/Foo run sent Rawr only `NotifyMemberJoinedPlaygroup`; the SDK
accepted Foo's slot-1 member structure and fetched Foo's account profile, but
Rawr's application roster still omitted Foo. This confirms that the full
recipient-specific snapshot is required in the live join path rather than only
being a compatibility fallback. `[E]`

Foo left at 19:03 and Darkspin authoritatively removed slot 1, returning the
leave response and notifying both sessions. Rawr's subsequent invite attempt
produced no Messaging or Playgroups frame, proving the client suppressed it
before the server boundary. An initial comparison with lookup responses
mistook `FLGS` for an online-status value and changed live `NotifyUserUpdated`
from `2` to `1`. The 19:50 run then carried the correct packed identity and
typed presence data but made `cOnlinePlatformBlaze::GetPresenceInfo` fail for
Foo. Client code proves that query returns false unless the cached user retains
its typed `Association::PresenceInfo`; `FLGS` on the update notification is a
field mask, and the earlier mask `2` is required to retain that full user data.
Live notifications now use `2` again while lookup responses remain `1`. The
automatic lookup had also returned an empty `EDAT.CVAR`, which could overwrite
the typed presence published immediately before the roster; lookup now carries
the same authoritative presence record as `NotifyUserAdded`. `[E]`

The Lobby tab is read from the current native Room object's member vector at
offsets `+0x12c/+0x130`; each vector entry is omitted from JSON when its SDK user
pointer is null. Focused Fang diagnostics report both the vector size and the
number of entries with resolved user pointers. This distinguishes a rejected
`RoomMemberJoined` event from a UserSessions identity-resolution failure without
changing client-owned lobby state.

Each resolved SDK user obtains the displayed lobby columns from the
`UserSessionExtendedData.CVAR` variable TDF. The concrete build-103 type is
`Blaze::Association::PresenceInfo` (`0xc0513268`), whose visitor at `0x00C40CB0`
confirms the exact fields `GRP`, `LVL`, `STAT`, and `XTRA`. The enclosing
notification's `USID` follows the same Blaze/persona identity established by
`NotifyUserAdded`, not Darkspin's internal TCP connection counter. The packaged
browser renders these respectively as party size, cumulative Crogenitor XP,
presence state, and threat progression. TDF wire type 7 is `Variable`, not an
integer-list type. A populated variable is encoded as the one-byte present
marker, its polymorphic type ID, one nested labelled field, and a separate outer
zero terminator after that field. The
client-authored command therefore contains one logical `CVAR:Variable`; its
nested field is the identically labelled `CVAR:Struct`.

Treating those bytes as two independent top-level fields happened to recover
the four scalar values but never constructed the receiver's native variable
owner. The browser then dereferenced unrelated object memory, so Crogenitor
level and party size changed every time the Lobby tab rendered and status varied
between Offline and undefined. Darkspin's TDF codec now decodes and encodes type
7 as that owning variable, retains each client's exact `PresenceInfo` type ID
and nested structure. Darkspin originally consumed the outer terminator as
harmless top-level padding and did not emit it. That appeared to decode a
client-authored command correctly, but inside a larger `DATA` structure the
receiver consumed the following extended-data field at the wrong boundary and
left `PresenceInfo` storage corrupt. Darkspin now owns and emits the terminator
and republishes the single `CVAR` variable through the generated
`UserSessionExtendedDataUpdate` DTO. Bootstrap retains a null
type-7 variable until the native publisher at `0x00C2B2D0` constructs the real
`PresenceInfo` and sends command `0x19`. The command response stays void; the
presence envelope is a notification, not a fabricated RPC result. A revision
registry replays the latest envelope to later sessions and suppresses unchanged
updates. The server must never send that notification back to its publisher:
doing so makes build 103 republish command `0x19` at roughly 50 Hz even when the
decoded fields are unchanged. Live evidence shows the publisher's `LVL` and
`XTRA` are structurally valid, but `XTRA` describes transient client context
rather than durable campaign authority. Darkspin retains the proven type
discriminator, replaces `LVL` with durable cumulative account XP, `XTRA` with
the highest selectable campaign index (`chain_progression + 1`), and `GRP`
with authoritative playgroup size. It preserves the client-authored `STAT`:
captured state `3` is squad selection and state `4` is the campaign Navigation
room.

Room admission also replays `NotifyUserAdded` so the native room member owns a
resolved SDK user pointer. Rooms and UserSessions share the presence registry
and can place the retained client-authored variable directly inside that
initial owner, while later changes use `UserSessionExtendedDataUpdate`. If a
member has not published presence yet, room admission keeps
the null variable and lets the native publisher populate it later rather than
fabricating a PresenceInfo from only the known type ID.

Navigation readiness is the custom Association `PresenceInfo.STAT` enum, not a
Playgroups member attribute or peer-mesh state. Both live clients send
`finalizePlaygroupCreation` (`0x09`) and no `setMemberAttributes` request. The
application gate in `sub_519960` calls the primary online-platform vtable entry
at `+192`, which resolves to `sub_C28060` and `sub_C2B480`; that method reads the
remote UserSessions `PresenceInfo` and returns `STAT` as its first output. The
gate requires that value to equal Navigation state `4` for every party member.
A live failing capture named remote party key `3` correctly but could not
resolve that ID through the primary online platform, so no status was available
to test. Party join now refreshes reciprocal full `NotifyUserAdded` and
`NotifyUserUpdated` ownership before either side consumes its Playgroups roster
updates. This retains the remote SDK owner needed by the later native Navigation
gate while still requiring every client to publish actual state `4` from the
campaign room.

The campaign-click call at `0x00515a3f` is temporarily observed by Fang without
altering its return value. The diagnostic records every remote key in the
application party map, whether the primary online platform resolves that key to
a UserSessions owner, and the four fields returned by the same vtable method as
the native gate. This separates a stale `STAT` from a mismatched playgroup user
ID or missing `PresenceInfo` owner on the next two-client capture.

The browser removes the local profile by comparing each member `OnlineId` with
the `blaze_id` loaded by the root page's `accountinfocallback`. The packaged page
registers that callback immediately after issuing the request, so loopback replies
can win the registration race. The existing deferred SporeNet callback bridge now
delivers this root callback on the web-host timer as it already does for player
profiles and social search, ensuring the local Blaze ID exists before the Lobby
tab applies its filter.

Notifications use the confirmed nested `ServerMessage` shape, preserve the
client's allowlisted `ClientMessage` payload, identify the sender with a
UserSessions object ID in `SRCE`, and supply a server timestamp and monotonic
message ID. `fetchMessages` and `purgeMessages` return `MCNT=0`; durable/offline
message history is not implemented.

Chat text is transient and intentionally does not enter profile persistence.
Profanity filtering and offline delivery remain future features; when a filter
is added its safe output belongs in `ATTR[0xFF05]` while `ATTR[0xFF02]` remains
the original client body.

## Reproduction artifacts

Generated IDA output is under `bin/game/logs/ida-chat-*.log`. The reusable
helpers are:

```text
scripts/reverse/string_xrefs.py
scripts/reverse/ida_dump_context.py
scripts/reverse/ida_dump_xrefs.py
scripts/reverse/ida_find_names.py
```
