# Build-103 RakNet and gameplay exchange

This note tracks the RakNet transport and Game gameplay-message exchange
for client build `5.3.0.103`. It is the working contract for IDA-02 and the
transport side of the tutorial. Statements about live behavior are dated and
separated from static binary findings.

Evidence labels follow the
[reverse-engineering research roadmap](reverse-engineering-research-roadmap.md).
Unless stated otherwise, virtual addresses below refer to the build-103
executable with SHA-256
`3C7248A4C6614450290A66C22AB7FCDA86052B29A7B555E834D7C23F9B73EE5B`.

## Decisive compatibility finding

Build 103 embeds an older, single-stage RakNet offline opening exchange:

```text
client                         gameplay server
   |--- 0x09 open request ---------->|
   |<-- 0x0A open reply -------------|
   |--- reliable 0x04 connection ---->|
   |<-- connected acceptance/events -|
   |--- Game GMS 0x7f+ --------->|
```

This is not the later two-stage `0x05/0x06` then `0x07/0x08` offline
negotiation found in newer implementations. A responder using those later IDs
cannot complete build 103's opening exchange; the Go runtime intentionally no
longer implements them.

The layers must remain distinct:

1. raw UDP and offline opening (`0x09`, `0x0a`);
2. RakNet connected-peer state;
3. reliability datagrams, ACK/NACK, ordering, and split reassembly;
4. RakNet internal connected messages below `0x7f`;
5. Game GMS application messages `0x7f-0xcc`;
6. authenticated Blaze game/session and account binding.

## Confirmed tutorial entry boundary

A 2026-07-16 trace reached the gameplay endpoint only after Blaze advertised
`127.0.0.1:42127` instead of the client-supplied private endpoint. The first
UDP payload was 1,464 bytes and began:

```text
09 0d 04 90 00 13 d5 92 da 79 00 ff ff 00 fe fe
fe fe fd fd fd fd 12 34 56 78 ...
```

The trace and the static request builder agree. The bytes parse as:

| Offset | Size | Meaning | Evidence |
| ---: | ---: | --- | --- |
| `0x00` | 1 | `ID_OPEN_CONNECTION_REQUEST = 0x09` | Binary-confirmed and trace-confirmed. |
| `0x01` | 1 | protocol version `0x0d` | Binary-confirmed and trace-confirmed. |
| `0x02` | 8 | client RakNet GUID in network byte order | Binary-confirmed; sample value begins `04 90 ...`. |
| `0x0a` | 16 | offline magic `00 ff ff 00 fe fe fe fe fd fd fd fd 12 34 56 78` | Binary-confirmed and trace-confirmed. |
| `0x1a` | 6 | target IPv4 `SystemAddress`: complemented IPv4 plus network-order port | Binary-confirmed. |
| `0x20` | variable | zero padding to the selected UDP payload length | Binary-confirmed. |

The earlier interpretation of `04 90` as an MTU field was wrong: it is the
first two bytes of the eight-byte client GUID. The request has no explicit MTU
field.

### Retry and payload-size policy

The RakPeer connection call made by `cTransportRakNet` supplies 12 attempts and
a 500 ms retry interval. The request builder divides those attempts across the
embedded link-MTU candidates `1492`, `1200`, and `576`, four attempts per tier.
It subtracts the 20-byte IPv4 and 8-byte UDP headers, producing these UDP
payload lengths:

| Attempts | Link MTU candidate | UDP payload length |
| --- | ---: | ---: |
| 1-4 | 1492 | 1464 |
| 5-8 | 1200 | 1172 |
| 9-12 | 576 | 548 |

This explains the observed 1,464-byte request and its smaller retries without
inventing an on-wire MTU scalar.

## Offline open reply

The build-103 server-side branch for a valid `0x09` constructs:

| Offset | Size | Meaning | Evidence |
| ---: | ---: | --- | --- |
| `0x00` | 1 | `ID_OPEN_CONNECTION_REPLY = 0x0a` | Binary-confirmed. |
| `0x01` | 16 | offline magic | Binary-confirmed. |
| `0x11` | 8 | server RakNet GUID in network byte order | Binary-confirmed. |
| `0x19` | 6 | requester's source `SystemAddress` | Binary-confirmed. |
| `0x1f` | variable | zero padding until reply length equals request length | Binary-confirmed. |

There is no explicit MTU field in this reply. The server pads the reply to the
incoming request's byte length, so the successful request/reply size selects
the working tier. The implementation temporarily enables the socket's
do-not-fragment behavior while sending the reply.

If the request's protocol byte is not `0x0d`, the parser emits
`ID_INCOMPATIBLE_PROTOCOL_VERSION = 0x18`, followed by protocol `0x0d`, the
offline magic, and the local GUID. Other branches can report states such as
already connected.

## Transition into connected RakNet

The client-side `0x0a` branch:

1. validates and skips the packet ID and offline magic;
2. reads the server GUID and returned `SystemAddress`;
3. matches the reply to a pending connection attempt;
4. allocates or initializes a remote peer;
5. builds RakNet internal message `0x04`;
6. submits it through the reliability layer with priority `0`, reliability
   value `2`, and ordering channel `0`.

The connected request payload is binary-confirmed as:

```text
[0x04]
[offline magic: 16 bytes]
[client GUID: 8 bytes, network order]
[current millisecond timestamp: 8 bytes, network order]
[configured password: zero or more bytes]
```

The transport passes an empty password in the observed game connection path.
The embedded reliability enum is now typed: value `2` is `RELIABLE`.

### Connection acceptance and new-incoming exchange

The reliable receiver and builders now establish the rest of the internal
connected handshake. On a valid `0x04`, the accepting peer checks that the
remaining bytes exactly match its configured password. An unavailable incoming
slot can produce `0x12`; a password mismatch produces `0x17`. On success it
sends `0x0e` with reliability value `3` and ordering channel `0`:

```text
[0x0e ID_CONNECTION_REQUEST_ACCEPTED]
[client external SystemAddress: 6 bytes]
[client system index: 2 bytes, network order]
[server internal SystemAddress[10]: 60 bytes]
[client request timestamp: 8 bytes, network order]
[server acceptance timestamp: 8 bytes, network order]
```

The client parses that payload, marks the remote peer connected, and sends
`0x11`, also with reliability value `3` and ordering channel `0`:

```text
[0x11 ID_NEW_INCOMING_CONNECTION]
[server SystemAddress: 6 bytes]
[client internal SystemAddress[10]: 60 bytes]
[server acceptance timestamp: 8 bytes, network order]
[client current timestamp: 8 bytes, network order]
```

Receiving a valid `0x11` transitions the accepting peer to connected state and
surfaces the new-incoming event. Values `3` on `0x0e` and `0x11` are
`RELIABLE_ORDERED`.

The embedded event vocabulary and `cTransportRakNet` receive switch establish
these application-visible results:

| ID | Transport interpretation |
| ---: | --- |
| `0x0e` | connection request accepted |
| `0x0f` | connection attempt failed |
| `0x10` | already connected |
| `0x11` | incoming connection |
| `0x13` | disconnection notification |
| `0x14` | connection lost |
| `0x1a` | modified packet |

These IDs are RakNet transport events, not Game gameplay opcodes.

## Reliability datagram envelope

The build-103 serializer/deserializer pair at `0x00ac4130`/`0x00ac3f00`
confirms the outer datagram header. Bits below use the conventional bit number
within the first byte:

| First-byte bit | Meaning |
| ---: | --- |
| `7` | Valid RakNet datagram; always set. |
| `6` | ACK. When set, bit `5` indicates optional congestion feedback after byte alignment. Plain ACK is `0xc0`; ACK with feedback begins `0xe0`. |
| `5` | NACK when bit `6` is clear, yielding `0xa0`. |
| `4` | Packet-pair flag on a normal data datagram. |
| `3` | Continuous-send flag on a normal data datagram. |
| `2` | Needs congestion/B-and-AS feedback on a normal data datagram. |

A normal data header is followed by a three-byte little-endian datagram
sequence. The default `0x84` emitted by the current Go encoder is therefore a
valid normal-data header with the feedback-request flag, not an arbitrary
magic value. ACK/NACK records are byte-aligned and encoded as:

```text
[0xc0 ACK or 0xa0 NACK]
[range count: uint16, network order]
repeat range count:
  [is single: uint8]
  [minimum sequence: 24-bit little endian]
  if not single: [maximum sequence: 24-bit little endian]
```

The current ACK range and triad codecs match this base layout. Its decoder only
accepts exact headers `0xc0` and `0xa0`, however, so it cannot consume the
binary-confirmed `0xe0` ACK form with optional feedback. Retransmission timing,
window behavior, and the feedback scalar's operational effect remain open.

The server tracks the incoming 24-bit datagram sequence per peer. A forward
gap produces compact `0xa0` NACK ranges, late arrivals close the gap, and an
already received datagram is ACKed without dispatching its encapsulated
messages again. Transport ACK publication is independent of gameplay handler
success: once a valid datagram has reached the receive window, an unsupported,
rejected, or panicking application handler cannot induce a reliable retry
storm. Receive gaps, reliable-message history, ordered queues, and split
assemblies are bounded.

At the gameplay layer, do not infer an outbound reflected locomotion packet
from the build-103 client receiver alone. Logical message `21` / wire `0x94`
reflection-decodes `cLocomotionData`, but the native direct-sender mapper table
has no logical-`21` entry even though it contains direct entries for fixed
messages such as logical `37` and `39`. Build-103 object initialization does
maintain per-field reflection baselines, which is consistent with indirect
delta replication, but the selected `0x94` field mask and send timing require a
retail capture or the authoritative sender implementation.

### Encapsulated packet header

`0x00ac1620` serializes and `0x00ac17c0` parses each internal packet:

```text
[reliability: 3 bits][is split: 1 bit][padding: 4 bits]
[payload length in bits: uint16, network order]
[message index: 24-bit little endian, for wire reliability 2/3/4]
[ordering-or-sequencing index: 24-bit little endian, for 1/3/4]
[ordering channel: uint8, for 1/3/4]
if split:
  [split count: uint32, network order]
  [split ID: uint16, network order]
  [split index: uint32, network order]
[payload bytes]
```

The local send API defines values `0-7`: `UNRELIABLE`,
`UNRELIABLE_SEQUENCED`, `RELIABLE`, `RELIABLE_ORDERED`,
`RELIABLE_SEQUENCED`, and the corresponding ACK-receipt forms `5-7`.
Serialization maps receipt forms `5`, `6`, and `7` onto wire values `0`, `2`,
and `3`; they are local receipt-tracking semantics, not extra three-bit wire
formats.

This exposes a concrete current-code mismatch: `server/raknet/datagram.go`
writes both a sequence triad and an order triad for sequenced modes `1` and
`4`. Build 103 writes only the one shared triad plus channel. Reliable-ordered
mode `3`, used by the connected and hello exchange, already follows the
compatible one-triad path.

## RakPeer and transport ownership

The static ownership chain is:

```text
cTransportRakNet
  -> RakPeer factory
  -> RakPeer::Connect
  -> RakPeer update/offline parser
  -> reliability layer
  -> cTransportRakNet receive pump
  -> GMS handler for packet IDs >= 0x7f
```

Useful build-103 anchors:

| Function | Address | Role |
| --- | ---: | --- |
| `cTransportRakNet` initialize | `0x00a897e0` | Creates RakPeer and stores it at transport offset `+0x74`. |
| transport connect | `0x00a89be0` | Calls RakPeer connect with 12 attempts and 500 ms retry. |
| transport receive pump | `0x00a8a1f0` | Handles RakNet events and forwards IDs `>= 0x7f`. |
| GMS dispatch | `0x00a8a400` | Removes one application-ID byte and invokes the registered handler. |
| transport send wrapper | `0x00a8a630` | Builds `[GMS ID][payload]` and forwards priority/reliability/channel. |
| RakPeer connect | `0x00a97420` | Creates the pending connection attempt. |
| RakPeer send | `0x00a97fc0` | Connected send entry point. |
| RakPeer receive | `0x00a98270` | Returns completed RakNet packets to transport. |
| offline parser | `0x00a9fee0` | Handles `0x09`, `0x0a`, incompatible-version, and related opening packets. |
| RakPeer update | `0x00aa1f40` | Builds retries and advances connection state. |

The GUID codec writes eight bytes in network order. The IPv4 `SystemAddress`
codec writes the bitwise-complemented 32-bit address followed by a
network-order 16-bit port. That six-byte form is specific to this embedded
IPv4 path; do not substitute the layout from a newer RakNet release.

## Game GMS boundary

Once RakNet produces a complete packet, `cTransportRakNet` treats any first
byte `>= 0x7f` as a Game application message. Dispatch strips exactly one
ID byte and gives the remaining payload to the registered GMS handler. The
client contains a contiguous application vocabulary from `0x7f` through
`0xcc`.

The current expected startup ordering is deliberately evidence-labeled:

| Stage | Direction | Message | Status |
| ---: | --- | --- | --- |
| 1 | Blaze | game membership and gameplay endpoint | Trace-confirmed entry prerequisite; exact accepted notification contract remains IDA-01. |
| 2 | C2S | offline `0x09` | Binary- and trace-confirmed. |
| 3 | S2C | offline `0x0a` | Binary- and trace-confirmed. |
| 4 | C2S | reliable internal `0x04` | Payload and `RELIABLE` framing binary-confirmed. |
| 5 | S2C | internal `0x0e` connection accepted | Payload, reliability value `3`, channel `0`, and state transition binary-confirmed. |
| 6 | C2S | internal `0x11` new incoming connection | Payload, reliability value `3`, channel `0`, and state transition binary-confirmed. |
| 7 | S2C | `Connected` `0x82` | Handler registration and callback binary-confirmed; this application message triggers the request below. |
| 8 | C2S | `HelloPlayerRequest` `0x7f` | Exactly 8 payload bytes, priority `0`, `RELIABLE_ORDERED`, channel `1`; binary-confirmed. |
| 9 | S2C | `HelloPlayer` `0x80`, then party state | Live-confirmed for a local hello. Injecting `PlayerJoined` here is not supported by the reference behavior. |
| 10 | S2C | GameState `0x8a`, ObjectCreate `0x8c`, party/player/world replication | Opcode vocabulary confirmed; dependency order is IDA-04. |
| 11 | both | movement, combat, objectives, loot, and tutorial runtime traffic | Opcode vocabulary and selected handlers confirmed; authority and framing remain incomplete. |
| 12 | S2C | tutorial completion snapshot `0xc8` subtype `0` | Payload width and client mutation binary-confirmed; reliability/order unresolved. |

### First GMS send: HelloPlayerRequest

`cClientSession` registers logical GMS type `3` (`Connected`, wire `0x82`) to
`sub_A8E8B0`. That handler changes session state, creates logical message type
`0` (`HelloPlayerRequest`, wire `0x7f`) with an exact capacity of eight bytes,
and copies the 64-bit value stored at session offsets `+0x08/+0x0c` into it.
It sends with priority `0`, reliability `3` (`RELIABLE_ORDERED`), and ordering
channel `1`.

The session owner obtains that 64-bit value from a virtual service call at
`sub_5363B0` and writes the returned `EDX:EAX` pair into those offsets before
connecting. Its final semantic identity and equality to a particular Blaze
field are not yet proven. The wire contract is nevertheless exact:

```text
[0x7f]
[session/account identity: uint64, raw little-endian x86 copy]
```

Build 103 does not append a playgroup ID in this serializer. The optional
second `uint64` accepted by the current Go decoder is reference compatibility,
not a build-103 requirement. Server binding still must verify the received
value against the authenticated Blaze game membership rather than trusting it.

## Tutorial-specific exchange

### Recovered tutorial native mapping

The recovered tutorial route uses the following client-visible Lua/native
operations. Each operation either has an exact build-103 application mapping or
is a confirmed client-local consequence; no launch-DLL presentation hook is
required by this mapping.

| Lua/native operation | Typed simulator operation | Build-103 consequence |
| --- | --- | --- |
| `nGameObject.SetIsVisible` | visibility | `ObjectUpdate` `0x8f`, including the authoritative transform and visibility field |
| `nLocomotion.Stop` | locomotion stop | 81-byte `ObjectPlayerMove` `0x91`; turn-in-place uses flags `0x42` |
| `TeleportObject` | teleport | 33-byte `ObjectTeleport` `0x90`: object ID, destination XYZ, and retained quaternion |
| `SetAnimationState` | animation | 25-byte `SetAnimationState` `0xa5`, including simulation timestamp, scale, and zero source/echo gate |
| `ResetAnimationState` | animation reset | the same `0xa5` shape with state ID zero |
| `AddEffect` / `RemoveEffect` | attached or object-bound effect | reflected `ServerEvent` `0x9b`; attached effects use the first-free one-based 16-slot lifecycle |
| at-position event notification | positioned effect | reflected fields `6/10/11` in 33-byte `ServerEvent` `0x9b` |
| `nGameSimulator.StartCinematic` | cinematic | subtype-free 25-byte `CinematicMsgs` `0xc9` described below |
| `nEvent.Notify` for `PlayerUnlockedSecondCreature` | allowlisted client event | exact bytes `9b 0f 36 bc e2 71 ff` |
| objective/ability lesson | objective authority | `ObjectiveUpdated` `0xb8` with `voiceover=0`; authored Cryos voice volumes are client-local |
| combat damage presentation | damage | sparse `CombatEvent` `0xba`; the receiver creates local floating combat text when its controlled-object and option gates permit it |

Trigger-volume creation/destruction, attribute mutation, cooldown/release state,
and job deletion are authoritative simulation operations rather than direct
presentation packets. The generic `DialogueIntent` vocabulary is not emitted by
any recovered Cryos tutorial chunk and deliberately has no build-103 adapter
mapping. It must not be used as a substitute for client-owned tutorial audio.

### Build-103 cinematic message

Static receiver recovery confirms that logical GMS type `0x4a` maps to
`CinematicMsgs` wire opcode `0xc9`. The callback at `0x00450f20`, registered by
the handler constructor at `0x004517e0`, consumes this exact application
message:

| Offset | Size | Field |
| ---: | ---: | --- |
| `0x00` | 1 | GMS opcode `0xc9` |
| `0x01` | 8 | signed duration in milliseconds, little-endian `int64` |
| `0x09` | 4 | focus X, little-endian `float32` |
| `0x0d` | 4 | focus Y, little-endian `float32` |
| `0x11` | 4 | focus Z, little-endian `float32` |
| `0x15` | 4 | camera radius, little-endian `float32` |

There is no subtype byte. The raw reader at `0x00a87bf0` uses `memcpy`, which
establishes native x86 scalar order. The receiver ignores nonpositive duration.
For a positive duration it sets the active flag, adds the duration to the
current 64-bit gameplay clock, and stores that deadline with the focus point
and radius. `sub_450dd0` permits immediate entry when no controlled agent exists
or when the agent's distance from the focus point is strictly less than the
radius. When permitted, the handler enters game state `8` through
`sub_455a10(8)`.

The regular update at `0x00451960` clears the active flag when the deadline is
reached. Before then, it rechecks the same radius predicate and requests state
`8` when more than `3500` milliseconds remain. The state initializer at
`0x0044e100` copies the cinematic deadline and focus point into its instance
when the active flag is set; otherwise it uses the ordinary camera focus and a
zero deadline.

State-8 update `sub_44e8f0` owns the cinematic camera. It derives a clamped,
smoothed progress scalar from the gameplay clock, interpolates from the camera
position captured on entry to the packet's focus point, and submits the result
to the camera controller through `sub_52d750`. When the copied deadline is
reached it requests state `6`, returning control to normal gameplay. State exit
`sub_44e680` performs the corresponding controller/listener cleanup. These
findings establish proximity gating, camera ownership, deadline cleanup, and
normal completion. The wire format has no explicit cancellation operation: a
nonpositive packet is ignored. Darkspin therefore cancels only unpublished
continuations when a simulation scope ends; a cinematic packet already
accepted by the client runs until its embedded deadline and returns to state
`6`. Sending a replacement positive packet overwrites the handler's active
deadline, focus point, and radius before the next update.

The Go codec preserves the duration as a signed `int64` millisecond field and
therefore also preserves nonpositive and full-width two's-complement wire
representations. Its `time.Duration` constructor rounds to the nearest
millisecond, with exact half milliseconds away from zero; the authored
`4.291667` second cinematic consequently encodes as `4292` milliseconds. This
is an explicit adapter conversion rule, not a claim about the unwired retail
Lua native.

The adjacent C++ server's unwired sender wrote two `uint32` test fields in the
same eight-byte region. Build-103 proves that split was a semantic
misinterpretation of one `int64`, not two timestamps. Darkspin publishes both
the immediate cinematic packet and its scheduled visibility/animation
continuations as `ReliableOrdered`; the shared outbound serializer preserves
authored order within each deadline. The generic simulator adapter has an
exact 25-byte golden covering the rounded duration, focus coordinates, and
radius, while the Quadra integration tests cover the surrounding stop,
cinematic, visibility, and animation sequence. A paired retail/live capture
remains desirable validation, but no implementation field or lifecycle edge
is left unspecified by the recovered receiver and current adapter contract.

### Build-103 objective messages

Static handler recovery and live testing establish these objective payloads.
Each layout starts after the one-byte GMS application ID:

```text
ObjectivesInit 0xb7:
[objective count: uint8]
repeat count times:
  [objective ID: uint32 little-endian]
  [player state: 4 * uint8]
  [player data: 4 players * 3 * uint32 little-endian]

ObjectiveUpdated 0xb8:
[objective ID: uint32 little-endian]
[player selector: uint8; 0xff means every player]
[state/medal: uint8]
[voiceover/sound ID: uint32 little-endian]
[show notification: uint8 bool]
[data: 3 * uint32 little-endian]

ObjectivesComplete 0xb9:
[objective count: uint8]
[objective records: count * 56 bytes]
[player results: 4 players * 4 uint8 fields]

ObjectiveAdd 0xca:
[objective record: 56 bytes]
```

The current five-objective initializer is therefore 281 payload bytes (282
including `0xb7`), and each update is 23 payload bytes (24 including `0xb8`).
`0xb9` is a full level-completion snapshot, not an encounter-clear signal. The
client's update handler presents the objective notification only when the show
flag is true and plays the supplied sound when its ID is nonzero. A clean live
run accepts the corrected initializer and shows the authored movement banner.
Later live testing confirms tutorial voice audio is playing normally.

`TutorialGameMsgs` uses GMS opcode `0xc8`. The confirmed subtype-zero payload
at the GMS boundary is:

```text
[0xc8 application ID]
[0x00 subtype]
[signed cumulative XP: 4 bytes]
```

The handler stores a positive cumulative XP total, derives the Crogenitor
level, and moves in-memory onboarding progress to `3000`. A nonpositive value
clears XP/level and returns progress to `2000`, so a server must not use zero as
an empty completion notification. The four-byte field's byte order and the
RakNet priority/reliability/ordering values remain unresolved.

Per-kill progression uses a different message. GMS logical `35` / wire `0xa1`
dispatches to `sub_539E00`, which resolves the one-byte player slot and invokes
`sub_A1E070`. Top-level selection mask `0x1000` chooses base Labs player type
`0x15ff16e2`; reflected base fields `15` and `16` carry the account level and
cumulative XP float. The exact combined application shape is:

```text
[a1][slot u8][00 10]
[0f][level u32 little endian]
[10][cumulative XP float32 little endian]
[ff]
```

It is 15 bytes including `0xa1`. This packet is a cumulative presentation/state
snapshot after an authoritative XP mutation, not a kill notification or XP
delta. XP `100` remains level 1 and XP `101` becomes level 2 because the native
threshold comparison is strict. Corpse fade/delete and loot pickup messages do
not replace this update.

Native loot scheduling is server-side state before it becomes packets.
`DropStuffForObject` queues recipient ID, selector, deadline, and a processing
flag at combatant offsets `+92..+104`, only when object byte `+153` permits
loot. Object byte `+154` independently controls XP eligibility.
`Behavior_Death` calls neither path. The eventual drop processor creates and
launches pickup objects and can emit `ServerEvent`, but does not mutate Labs
player XP. Build-103 selector bits are now native-confirmed: `0x02` orbs,
`0x04` catalysts, `0x08` loot, `0x10` DNA, and `1` no drop. The orb budget
guarantees one selection per complete hundred points and treats the remainder
as a percentage chance. Orb choice weights health and mana from roster-average
resource fractions using `2 + 5 * missingFraction`, with a separately gated
resurrection override on the health side. Each accepted orb is created at the
drop position, receives an asset-plus-position `ServerEvent`, then enters
authoritative launch motion. The event is exactly fields `6` and `10`, a
20-byte application recipe; native offset `0x24` is `position`, not the old
C++ scaffold's `targetPoint`. Exact pickup `ObjectCreate`, launch replication,
and server recovery policy remain open.

Health and mana pickups are networked locomotion-type-6, non-combatant objects
with client-instantiated `Orb_Pickup` overlap triggers. Placed tutorial variants
have lifetime zero and authored `1x1x1` boxes; dropped variants have a 30-second
lifetime and authored `2x2x4` boxes. All request object-dimension substitution,
so fixtures must not combine their unused sphere/capsule values with the box.
They are collected by authoritative movement intersection, not an interaction
click.

The client-side trigger is not a hidden request path. Construction installs
the noun trigger through `sub_9D1930 -> sub_A16F40`; entry dispatches through
`sub_A16810 -> sub_A15FB0`. Static initializer `0x00F53C10` binds
`Orb_Pickup` to `sub_9C98A0`, whose entire client implementation queries the
client-role flag and returns false. It sends no GMS message, changes no
resource, emits no event, and deletes no object. Therefore the Go simulation
must consume accepted movement as the overlap command and must never wait for
an orb-specific client packet. Resource replication and pickup/full
`ServerEvent` feedback precede authoritative deletion; popup scripts merely
observe the result. Dropped-orb expiry is also authoritative: its noun lifetime
becomes an absolute object deadline, while placed capsules have none.

Resource replication must preserve the native change boundary. Build-103
`CombatantDataUpdate` wire `0x97` has bit `0` for HP and bit `1` for mana, so a
green pickup is `[97][object u32][01][HP f32]`, a blue pickup is
`[97][object u32][02][mana f32]`, and only a simultaneous change uses mask
`0x03`. Full-resource feedback changes no component field and therefore needs
no `0x97`. Go now emits these sparse mutations while retaining the all-fields
expansion for initial object state. The presentation packet is independently a
`ServerEvent`: fields `6/7/10/14` for a non-full pickup and `6/7/10` for full,
followed by `ObjectDelete` `0x8e` only after the overlap has been accepted.
Whether retail resources belong to the active hero or are mirrored across
Blitz and Sage remains a gameplay-policy question requiring a post-unlock
capture; the component codec itself does not answer it. Darkspin currently
mutates and republishes only the deployed entrant so independent squad state is
not overwritten by an unevidenced mirror.

Pickup application encoding and authority mutation form one locked transaction.
The pickup result carries the entrant object, squad slot, previous resource,
resulting resources, and collection identity. A same-command encoding failure
discards all staged application packets and rolls the pickup batch back in
reverse order. Transport publication occurs afterward and retains RakNet/UDP's
ordinary non-transactional delivery boundary.

The chunk-215 notification is a distinct minimal reflection shape. Lua creates
only `clientEventID = SPID("PlayerUnlockedSecondCreature")`, whose build-103 ID
is `0x71e2bc36`. The exact application packet is
`9b 0f 36 bc e2 71 ff`: `ServerEvent`, reflected field `15`, the little-endian
ID, and the reflection terminator. Fields `6`, `7`, and `10` must be absent;
encoding their zero defaults changes the authored shape. This notification
follows the mission-local second-creature mutation and does not itself grant
persistent ownership or create Quadra.

`TutorialGameMsgs 0xc8` is a server-to-client tutorial/account snapshot; it is not the
same thing as the local Lua/native `UnlockSecondCreature` mission primitive.
It also does not itself persist the account. The subsequent result, teardown,
HTTP mutation, and return-to-ship sequence remains IDA-06.

A whole-code immediate search for logical GMS type `0x49` (`0xc8 - 0x7f`)
found 45 numeric uses. The protocol-relevant hit at `0x00451c1d` is the known
client handler registration; none of the direct build-103
`cProtocolTransport::CreateMessage` callers constructs logical type `0x49`.
This is useful negative evidence, not proof that no indirect constructor
exists: the retail client appears to contain the receiver but not an obvious
server-side sender. Further sender research should prioritize a server/dev
binary, dynamic packet evidence when execution resumes, or the generic
gameplay broadcast path rather than repeatedly searching this leaf handler.

### Build-103 combat-number boundary

GMS logical message `63` maps to application byte `0xba`. Dispatcher
`sub_53ADC0` routes it to `sub_539F60`, which reflection-decodes the eight
fields of a 40-byte in-memory `CombatEvent` and invokes `sub_4E2CA0`. Selecting
all fields uses mask `0xff` followed by a 39-byte application body: 16-bit
flags, `deltaHealth`, `absorbedAmount`, target/source/ability IDs, a three-float
damage direction, and signed `integerHpChange`.

Native sender `sub_A207F0` uses reflected type `0x5f1cc727` and does not force
that all-fields form. Generic HP-change constructor `sub_9E4D90` sets flags bit
`0` for damage, adds bit `2` for a killing blow and bit `3` for critical, then
leaves absorbed amount, ability ID, and direction at their defaults. Ordinary
tutorial damage therefore selects fields `0/1/3/4/7`, mask `0x9b`, and occupies
20 application bytes including `0xba`. Its values are flags `0x0001` (or
`0x0005` at zero HP), positive float damage, target ID, source ID, and negative
integer HP change. This minimal native shape is the parity target; `0xff` is a
decodable but non-native expansion.

The health-state companion is logical `24` / wire `0x97`. Receiver
`sub_539AA0` resolves the combatant component at object offset `716`, and
`sub_A1DD10` reflection-decodes Build-103 type `0xfab3107f`. Its two fields are
HP and mana at component offsets `64` and `68`; initialization seeds their
comparison cache at component offset `76`. A damage operation changes only HP,
so mask `0x01` and the ten-byte application shape
`[0x97][object ID][0x01][HP float]` are the native-compatible delta shape now
used by Go damage replication. The client direct-sender table contains no
logical-`24` mapper call, however, so this remains a focused capture/server-
sender target rather than a capture-confirmed golden. Mask `0x03` is retained
for initial two-field baselines rather than unchanged-field mutations.

The generic healing path reuses `CombatEvent` rather than defining another
message. `HealDamage -> sub_9E5D10 -> sub_9E5800 -> sub_9E4D90` emits the same
minimal mask `0x9b` fields `0/1/3/4/7`, with flags `0x0002`, negative
`deltaHealth`, and negative `integerHpChange`. When healing caps at maximum HP,
the float delta is the negated requested amount but the integer change is
derived from the actual old/new HP; a zero applied heal suppresses the event.
This is a confirmed generic-health rule, not proof that `Orb_Pickup` invokes
it. Orb recovery already has a numeric pickup `ServerEvent`, so a second
healing event must not be inferred without its native handler or a capture.

Floating combat text is not another server packet. For damage,
`deltaHealth > 0` selects the damage path and `integerHpChange` supplies the
signed displayed integer. The client chooses a local
`combattext_damage*.ServerEventDef` and queues it through `sub_506D90`. Flags
zero is valid ordinary damage; flag bit `3` selects the critical style. The
path requires the active controlled simulation object to resolve through
`sub_4E57B0`, then honors the applicable `ShowDamageDoneByMe`,
`ShowDamageDoneByAllies`, `ShowDamageDoneToMe`, or
`ShowDamageDoneToAllies` preference. Thus a visible HP mutation with no rising
number can prove the health-state packet while still exposing a missing local
control binding. Do not compensate with an invented floating-text
`ServerEvent`; diagnose the player-slot/control association and preserve the
target object long enough for its client-local presentation.

That association is top-level `LabsPlayer` reflection field `9`. Reflection
resolves the wire `uint32` object ID into a client game-object handle stored at
offset `+0x1238`; the object's wire ID is at handle offset `+16`. Initial tutorial snapshots now include that
field, and a character switch emits the sparse application shape
`a1 <slot> 00 10 09 <object:uint32> ff` before `PlayerCharacterDeploy`.
Previously darkspin omitted field `9`, leaving the combat-text resolver without
an authoritative controlled-object binding even though the portrait and action
bar switched successfully.

Fang now records that exact relationship for every sparse `0x9b` event. Along
with the controlled object and player index, it logs the decoded target, source,
signed integer HP change, and a `combat_control_relation` bitset (`1` means the
controlled object is the source, `2` means it is the target). A value of zero
proves the event cannot select either local-player preference branch; a value
of one or two moves the remaining diagnosis beyond wire attribution into the
client preference/resource/presentation queue.

## Current Go implementation and loading boundary

| Area | Current repository state | Build-103 requirement |
| --- | --- | --- |
| Shared UDP routing | An unknown remote IP:port is initially classified per packet. A successful build-103 `0x09` opening or valid QoS exchange pins that endpoint exclusively to its route until five minutes of inactivity. A one-minute cleanup sweep removes abandoned bindings even when their endpoints never send again. Tests cover misleading packets in both directions after binding and active expiry. | Keep the binding identifier limited to the build-103 opening; do not recognize modern RakNet opening IDs. |
| Offline opening | The server answers the build-103 `0x09` request with a padded `0x0a`; modern `0x05-0x08` handlers were removed. | Preserve the single-stage build-103 exchange. |
| Connected peer | Internal `0x04`, `0x0e`, and `0x11` complete in a live run and application `Connected` `0x82` reaches the client. Application and poll payloads require that connected state. Incoming `0x13` disconnection and defensive `0x14` connection-lost notifications are transport-ACKed, remove the peer, and release its address-scoped gameplay session only after publication leaves the outbound lock. A peer with no received data, ACK, or NACK for 30 seconds is expired through the same lifecycle callback. | Confirm the 30-second receive-idle threshold and defensive `0x14` handling against a prolonged retail-client trace. |
| Reliability | Reliable outbound datagrams are retained per peer. ACK retires them; NACK and timeout resend their exact original bytes without allocating new datagram, message, order, or split identities. Incoming datagram gaps emit bounded NACK ranges; duplicates are ACKed without redispatch; reliable indices are deduplicated; reliable-ordered packets wait for their per-channel order index; sequenced packets discard stale updates; and split packets are bounded, expired, and reassembled before ordering and application dispatch. Transport ACKs are published even when gameplay handling fails. Timeout retries back off from 500 ms to 2 seconds, the outbound window is bounded at 4,096 datagrams, eight failed sends retire the peer, and all 24-bit transport indices wrap explicitly. | Recover any remaining congestion-feedback, adaptive-window behavior, and `0xe0` ACK feedback when a trace exercises them. |
| GMS dispatch | Application payloads are accepted only after the peer reaches connected state. | Maintain the connected-peer boundary. |
| Hello binding | The exact eight-byte identity is checked against accepted Blaze game membership, then the server sends local hello and party state. | Keep the local response free of an invented `PlayerJoined`. |
| Tutorial loading | Live build-103 evidence on 2026-07-16 confirms status `4`, exact 16-byte `GamePrepare`, scene resolution/change, status `8`, four-byte `GameStart`, and the dungeon setup response. Cryos, the HUD, and deployed Blitz render and remain alive. The two locked entries are currently Blitz-backed crash guards and produce duplicate portraits. A negative test kept the fixed records valid while zeroing only their trailing reflected noun/assets; build 103 reached dungeon setup and then closed, proving both sections require resolvable data. | Recover a valid non-player/locked noun and type for both representations that leaves only the selected Blitz portrait visible. |
| Tutorial movement | The client emits `ActionCommandMsgs` `0x9c`; a left-click movement body is 64 bytes after the application ID. Routine movement now responds with active-goal `ObjectPlayerMove` (`0x91`, flags `0x01`) immediately followed by the fixed object-ID/goal `LocomotionDataUnreliableUpdate` (`0x95`). A clean build-103 left-click test visibly walked Blitz and tracked the camera without the old `ObjectTeleport` snap. | Connect the confirmed packet path to authoritative navigation and collision as the simulation grows. |
| Opening combat | Live build-103 evidence confirms the two `TutorialBasicDiseased` objects render from AI markers `(86.052,-76.175,14.976)` and `(88.801,-67.449,14.976)` when replicated as 81-byte enemy `ObjectCreate` plus combatant/attribute state. Clicking one emits `ActionCommandMsgs` type `7`, 84-byte body, targeting the enemy object. A type-2 action acknowledgement, `CombatEvent`, combatant HP update, and `ObjectDelete` visibly drive 20 -> 10 -> 0 HP and remove the defeated enemy. Their clear spawns three `TutorialBasicPoisonNoOrbs` objects, followed by groups at `x=170-184` and `x=197-233`. The opening pair is withheld from dungeon setup and a clean live run confirmed their one-time create at `(97.636,-89.381)`, 17.6 units from the first marker. First aggro now pins them at their markers for the authored `1.291667`-second beat, then sends `ObjectPlayerMove` active-goal flags `0x01` followed by the exact 16-byte `LocomotionDataUnreliableUpdate` (`0x95`) object-ID/goal body. A clean live run visibly walked both infectors from their AI markers to the hero. The current immediate 4-unit/1.5-second retaliation is obsolete scaffolding: recovered `TutorialPoisonCloud` has activation range `8`, independent travel distance `12`, a 3-second cooldown, a `0.17`-second wind-up, `1.86`-second release, speed `6`, a dedicated attack animation, projectile noun/trail, impact cloud, and rank-one damage `1-4`. Native `WaitForProjectile` is an authoritative scheduler/collision primitive, not a client packet. The projectile noun is a replicated locomotion object without a combatant component; its generic create/movement snapshot remains to be recovered. Native `AddEffect` sends the trail as a one-based force-attached `ServerEvent` slot. Poison Cloud's at-position cloud uses only reflected fields `6`, `10`, and `11`; its successful direct-hit path can emit that cloud once for collision and again for accepted damage, while range expiry falls back to one cloud event. With inherited `timeAfterImpact=0`, it deletes the projectile without an explicit trail-stop message. The exact 16-byte trail-attachment and 33-byte at-position impact application packets are now derived from the build-103 `nEvent.Notify` bridge. Native ordinary `CombatEvent` is the 20-byte mask-`0x9b` shape; rising text is generated client-locally and still remains invisible because the controlled-object/settings gate has not been live-proven. The exact 25-byte build-103 `SetAnimationState` encoder is tested, but attack/beam-in animation remains invisible. A minimal `generic_spawn` event produced no visible effect. The attempted ObjectJump clock anchor was invalid and removed after confirming its receiver reflection-decodes a 44-byte structure. Recovered death authority now publishes the selected animation, collision disable, stop-at-corpse goal, ordinary ten-second revival plus five-second fade, critical fast-delete branch, cleanup, and final delete. | Replace the retaliation timer with a semantic-parity-tested animation/projectile/impact timeline. Recover generic projectile `ObjectCreate` and motion, live-verify both derived `ServerEvent` recipes and local-control combat-text gate, and confirm the HP-only `0x97` delta. |

| Ability lesson | Count `3` unlocks Ride at HUD index `2`. The initial boundary is `1`; the server-only five-unit marker waits its authored one second and sends `2` then `3`, matching its two `UnlockNextAbility` calls. A clean focused replay crossed both markers in one movement, exposed only Ride, and traced accepted blink availability `0x10002` with the white arrow visible. A valid Dungeon/Tutorial GameState `0x8a` before world replication initializes the shared source clock without diverting to START/menu; cast-time-anchored C1 then produces the correct ten-second sweep and makes authored `character_teleport_in` visible. The accepted Ride cast clears the arrow, teleports, applies 18 damage and Shock, enforces ten seconds, and resets on kill. Voltic Slash uses `LightningRogueBasic` (`0x33cb6b29`), applies 8 damage, and has a live-confirmed anchored 400 ms cooldown. The default spawn is restored to the authored route start; focused snapshots remain explicit test inputs. | Continue the level-2 beat. |
| Tutorial completion | `0xc8` subtype-zero semantics are known. | Recover framing/order and the surrounding result/persistence exchange before implementation. |

Opening-enemy death is no longer an animation-duration guess. Recovered
`Behavior_Death` sends descriptor-selected `SetAnimationState` (`0xa5`) at
entry, while authoritative scheduler state owns corpse lifetime. An ordinary
NPC waits up to ten seconds for HP to rise above zero, attaches its
creature-type fade for five seconds if it remains dead, and only then marks for
delete; the opening Life-type `TutorialBasicDiseased` uses
`fadeaway_bio.ServerEventDef` (asset `0xea273c08`) through the one-based,
force-attached 16-byte `ServerEvent` recipe. A critical non-player/non-boss
instead marks for delete after `0.1+3` seconds and does not explicitly attach
the elemental fade. `SetCorpseFadingAway`, `WaitForHitpointsAbove`,
`WaitForFadeOutInXSeconds`, and `MarkForDelete` do not directly emit packets;
normal replication later produces `ObjectDelete`. Go parity therefore needs
separate killing-combat, HP-zero, death-animation, revival/fade deadline,
collision cleanup, and deletion intents with phase/session cancellation.
Revival's `ResetAnimationState` is the exact 25-byte `0xa5` message with state
ID zero and a fresh simulation timestamp. Target clearing and physics/nav
collision changes have no direct sender call in their Lua wrappers; treat their
wire consequences as ordinary replication evidence targets. The build-103
adapter now sends a reliable flags-`0x20` `ObjectPlayerMove` at the target's
authoritative position when the death behavior emits its locomotion stop, so a
corpse cannot continue a previously published pursuit goal.

### Generic object-create snapshot

Build 103 registers `nEvent.Notify` at `sub_A11670` with callback
`sub_A11560`. The callback reflection-decodes type `0x8619ff24` and passes it to
`sub_A20910`, which emits logical message `28` / wire `0x9b` and a `0xff` field
terminator. Poison Cloud's minimal attached-trail application packet is 16 bytes;
its asset/position/facing impact packet is 33 bytes. This is build-103-local
provenance; the analogous Build-127 callback/sender addresses are reference only.

Build 103 registers `cGameObjectCreateData` as ten fixed fields: noun, position,
three rotation-degree floats, asset ID, scale, team, collision, and
player-controlled. The packet begins with object ID, a `0x03ff` mask selecting
all ten fields, and 43 serialized field bytes (the registered in-memory offsets
include alignment through offset `46`). Its reflected `sporelabsObject` tail
uses indices `0` team, `1` player-controlled, `2` input stamp, `3` player index,
`4` linear velocity, `5` angular velocity, `6` position, `7` orientation, `8`
scale, `9` marker scale, `10` last animation state, `11` last animation time,
`12` movement-animation override, `13` graphics state, `14` graphics-state
start, `15` new graphics-state start, `16` visible, `17` collision, `18` owner,
`19` movement type, `20` disable repulsion, `21` interactable state, and `22`
source marker ID. This registration confirms field meaning, not presence in a
particular snapshot. For ballistic projectiles, initial velocity/movement fields
and subsequent motion remain trace/native-sender targets. Fixed unreliable
`LocomotionDataUnreliableUpdate` (`0x95`) is not a ballistic snapshot: the
Build-103 receiver copies its sole vector into `mGoalPosition` and
`mPartialGoalPosition`. Reliable reflected `LocomotionDataUpdate` (`0x94`) can
carry nested projectile parameters, target object/position, initial direction,
external motion, and expected collision state, making it the leading ballistic
candidate; its actual retail field mask and presence after `ObjectCreate` are
not yet proven.

Build-103 sender/receiver analysis refined the opening-combat animation row.
`SetAnimationState` (`0xa5`) does consume the full 25-byte body. Native sender
`sub_A1FDD0` writes object ID, state ID, timestamp, overlay, scale, and a final
32-bit source/echo-gate field. The Lua animation wrappers pass zero for the
final field; the older C++ repeated-state guess could bypass the receiver's
duplicate/current-state suppression and darkspin no longer reproduces it. The
older `sub_A25410` attribution came from the Build-127 decompile and is not
Build-103 provenance.

## Static research queue

1. Recover the level-2 XP threshold and the partial player-update fields that
   drive the account-level HUD beat without guessing from modern data.
2. Decode and implement the first tutorial enemy/objective trigger after the
   opening movement lesson.
3. Determine what causes the client to send the six-byte `ChainPlayer` request
   after vote/countdown, and whether prepare belongs solely on that response.
4. Compare retry timing and congestion feedback against a lossy live build-103
   trace, then adapt the fixed backoff only where client evidence requires it.
5. Trace the `0xc8` sender-side constructor and transport-send arguments, then
   place it within result, teardown, persistence, and return-to-ship ordering.

## Reproducibility and generated evidence

The primary IDA database is `bin/game/GameBin/Game.idb`. Automated
queries can use the IDA 7.0 CLI at `C:\Program Files\IDA 7.0\idat.exe`; the
licensed GUI is available at `C:\Program Files\IDA 7.0\ida.exe`. Set `IDAUSR`
to `bin/game/ida_user` and keep generated logs under `bin/game/logs`.

Relevant generated logs currently include:

- `ida-raknet-transport-core.log`
- `ida-raknet-transport-events.log`
- `ida-raknet-peer-connect-send.log`
- `ida-raknet-peer-update.log`
- `ida-raknet-offline-parser.log`
- `ida-raknet-offline-builders.log`
- `ida-raknet-wire-helpers.log`
- `ida-raknet-wire-primitives.log`
- `ida-raknet-connected-handshake.log`
- `ida-raknet-datagram-header.log`
- `ida-raknet-wire-codecs.log`
- `ida-raknet-encapsulated-hello.log`
- `ida-client-session-gms-handlers.log`
- `ida-client-session-gms-registration.log`
- `ida-application-registration.log`
- `ida-server-event-decoder.log`
- `ida-server-event-struct.log`
- `ida-gms-hello-codecs.log`
- `ida-client-session-owner.log`
- `ida-gms-c8-logical-type.log`
- `ida-tutorial-unlock-primitives.log`
- `ida-tutorial-ability-count.log`
- `ida-tutorial-player-copy.log`
- `ida-gms-gameplay-dispatch-full.log`
- `ida-labs-player-update-handler.log`
- `ida-labs-player-update-fields.log`
- `ida-labs-player-reflection-dispatch.log`
- `ida-labs-player-reflection-fields.log`
- `ida-labs-player-metadata-hash.log`
- `ida-reflection-registry-xrefs.log`
- `ida-reflection-registry-insert.log`
- `ida-reflection-registry-init.log`

Generated logs are working evidence and may be ignored by version control;
stable conclusions belong in this note and, once independently verified, in a
build-specific confirmed note.
