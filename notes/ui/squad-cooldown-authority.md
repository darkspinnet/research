# Squad cooldown clock authority

## Result

Build 103 does not expose the local gameplay-clock origin to the retail
application server. No client-to-server SporeNet application field contains
that clock, and RakNet's connection/ping timestamps use RakNet's platform
monotonic clock instead. A RakNet ping exchange can synchronize that transport
clock with the server, but it cannot recover the unknown additive offset
between the transport clock and the gameplay clock.

Character reflection field `4` is nevertheless conclusively a deadline in the
client's local gameplay-clock domain. There is no second retail epoch hidden in
the field decoder or either consumer. Given the evidence presently available,
an unmodified remote client and a server limited to the recovered build-103
wire vocabulary do not provide enough information to calculate the exact
deadline. Retail therefore used either an unrecovered side channel/integration
outside this application vocabulary, an intentionally aligned clock policy for
which no evidence survives, or did not dynamically populate this field in the
way the recovered reference server suggests.

This specifically invalidates treating RakNet `ID_CONNECTION_REQUEST`'s
`client send time` as the gameplay origin. The current
`server/raknet/server.go::connectionAccepted` records that field as
`clientTimeOrigin`; its arithmetic is valid only for translating into the
client's RakNet clock, not the gameplay clock read by the portrait and switch
validators. The synthetic `129330` in
`TestConnectionAcceptedRecordsClientClockOrigin` came from a gameplay trace;
the test does not prove that a real connection request carries that value.

Confidence is:

- **high** that field `4` is a local gameplay-clock deadline and that raw
  server source time and Windows/RakNet uptime are wrong;
- **high** that no build-103 C2S application payload exposes the gameplay
  origin;
- **high** that RakNet clock synchronization alone cannot determine the
  missing additive gameplay offset;
- **medium-high** that the connection-request timestamp is the ordinary
  RakNet platform clock in this build, based on its wire role, the bundled
  RakNet behavior, and the build's raw platform-clock implementation;
- **low** on the exact mechanism used by the original retail backend, because
  no retail server capture or source remains.

## Three distinct clocks

The relevant clocks must remain separate:

```text
S = server/game source milliseconds placed in GameState and timed S2C packets
G = client's local gameplay milliseconds (qword_14D05D0)
U = client's RakNet/platform monotonic milliseconds
```

The client mapper established from a `GameState` sample is:

```text
G(r) = G0 + trunc((r - R0) * scale)

R0 = the first GameState source timestamp
G0 = local qword_14D05D0 when the pending sample is committed
```

At the observed scale of one:

```text
G(r) = r + (G0 - R0)
field4 deadline D = G(Sdeploy) + 30000
                  = G0 + (Sdeploy - R0) + 30000
```

The server knows `R0` and `Sdeploy`. It does not learn `G0`. Any proposed
server-only derivation must identify where that last term comes from.

RakNet synchronization instead establishes a mapping such as:

```text
U(s) ~= Uhandshake + (s - Shandshake)
```

It supplies no observation of `G - U`. For any constant `k`, replacing every
unobserved gameplay value with `G' = G + k` produces exactly the same C2S wire
traffic and RakNet timing exchange. The desired deadline changes by `k`, so it
is not identifiable from those packets.

## Complete GameState and gameplay-clock path

Canonical client evidence is
`bin/game/GameBin/Game.c`:

- `sub_537CE0` at `0x00537CE0` reads the complete 25-byte `GameState`
  application payload:

  ```text
  +0x00  u64-le source game time R
  +0x08  u64-le elapsed/objective time
  +0x10  u8     game state
  +0x11  u32-le game type
  +0x15  u32-le trailing simulator flag
  ```

  It passes only the first `uint64` to `sub_9D2830`, then applies the other
  fields through `sub_9BD1D0`, `sub_9BCD30`, and `sub_9BCD50`. The packet has no
  client-authored leg or echo field.

- `sub_9D2830` at `0x009D2830` writes the received source sample to mapper
  offsets `+48/+52` and sets the pending byte at `+81`. It does not send
  anything.
- `sub_9D29C0` at `0x009D29C0` commits the pending mapper. It copies the remote
  sample to the remote origin at mapper `+8`, obtains the current local
  gameplay time through `sub_9D2920`, stores that as the local origin at
  mapper `+16`, sets scale `1.0` at `+24`, and marks the mapper initialized.
  This internal local snapshot is the missing value `G0`.
- `sub_9D2A00` at `0x009D2A00` performs
  `G0 + trunc((remote - R0) * scale)` for nonzero remote timestamps.
- `sub_9D2A70` at `0x009D2A70` can refine scale from later remote samples, but
  it still stores the paired local observation only inside the client. It has
  no C2S publication.
- `sub_9D2710` at `0x009D2710` initializes the independent gameplay clock
  `qword_14D05D0` from the engine high-resolution timer.
  `sub_9D27A0` at `0x009D27A0` advances it by frame deltas and clamps an
  invalid or greater-than-333-ms delta to `333`. `sub_9D2810` returns that
  clock, while `sub_9D2920` returns the simulator's mapped/current gameplay
  time. This clamping alone prevents a general identity with platform uptime.

The `GameState` receiver therefore proves a one-way sample: the server provides
`R0`; the client privately pairs it with `G0`.

## Field 4 has no alternate epoch

The complete field-`4` path rules out an implicit remote/server epoch:

- `sub_539E00` at `0x00539E00` receives `LabsPlayerUpdate`.
- `sub_A1E070` at `0x00A1E070` selects character records and invokes generic
  reflection decoder `sub_A1D800` at `0x00A1D800`. It stores field `4` without
  calling `sub_9D2A00` or any other clock converter.
- The recovered character descriptor names field `4` `DeployCooldown`, types
  it as `uint64_t`, and sparse-serializes eight little-endian bytes.
- `sub_41E2B0` at `0x0041E2B0` computes the portrait wipe from the stored low
  word and current local gameplay time.
- `sub_9C2B30` at `0x009C2B30` uses the same stored low word and local clock to
  reject a switch.

Both consumers effectively use:

```text
remaining = int32(deadlineLow32 - gameplayNowLow32)
isCoolingDown = remaining > 0
```

The high word is retained but does not select an epoch. A Unix time, Windows
uptime, RakNet time, or raw server source value cannot be distinguished and
converted by these readers; it is simply subtracted from `G`.

## Every client-authored timestamp candidate

### SporeNet application messages

Build 103's active C2S application endpoints are
`HelloPlayerRequest`, `PlayerStatusUpdate`, `ActionCommandMsgs`,
`ChainPlayerMsgs`, `ArenaPlayerMsgs`, `JuggernautPlayerMsgs`,
`CrystalDragMessage`, `KillRacePlayerMsgs`, `LootDropMessage`, and
`DebugPing`. Their recovered senders leave no gameplay timestamp:

| C2S family | Sender/address | Relevant payload | Clock result |
| --- | --- | --- | --- |
| Hello | `sub_A8E8B0`, `0x00A8E8B0` | 8-byte player/user identity | none |
| Player status | `sub_538C30`, `0x00538C30` | two `uint32` values | none |
| Action command | `sub_5370F0`, `0x005370F0` | 40-byte common header plus typed tail | `+4` is an input-sync sequence copied from object state, not time |
| Chain player | `sub_448350`, `0x00448350`; `sub_448520`, `0x00448520`; `sub_449620`, `0x00449620`; `sub_449850`, `0x00449850`; `sub_519A70`, `0x00519A70`; `sub_519B90`, `0x00519B90`; `sub_527970`, `0x00527970` | subtype bytes, slot/flag/result data | none |
| Arena player | `sub_401C70`, `0x00401C70` and the Arena UI senders | subtype and fixed game-mode data | none |
| Juggernaut player | `sub_437CB0`, `0x00437CB0`; `sub_452710`, `0x00452710` | subtype/boolean data | none |
| Kill-race player | `sub_438380`, `0x00438380`; `sub_452D90`, `0x00452D90` | subtype/result/UI data | none |
| Crystal drag | `sub_537210`, `0x00537210` | fixed 24-byte drag command | none |
| Loot drop | `sub_5372C0`, `0x005372C0` | 8-byte item identity | none |
| Debug ping | `sub_538AC0`, `0x00538AC0` | `_time64(0)` response in 8 bytes | Unix seconds, not gameplay milliseconds |

`sub_A8FC00` at `0x00A8FC00` is the common application-message allocator used
by these senders. The packet vocabulary also establishes that `GameState` is
S2C-only. There is no hidden GameState acknowledgement carrying `G0`.

### RakNet control timestamps

RakNet adds timestamps below the SporeNet application layer:

- connected ping/pong carries the original ping time and the peer's pong time;
- `ID_CONNECTION_REQUEST` (`0x04` in this build's connected handshake) carries
  the client GUID and an 8-byte client send time;
- `ID_CONNECTION_REQUEST_ACCEPTED` (`0x0e`) echoes the client send time and
  adds the server's accept time;
- an `ID_TIMESTAMP` wrapper carries a RakNet time before the wrapped message.

These fields synchronize RakNet peers. They are not populated through
`sub_9D2920`, and the GameState receiver never consumes them.

Game's transport receives control ID `0x04` in `sub_A8A1F0` at
`0x00A8A1F0`; `sub_A89BE0` at `0x00A89BE0` calls the RakPeer connection path.
The bundled platform-time primitive `sub_553C50` at `0x00553C50` is exactly
`1000 * timeGetTime()`, i.e. microseconds derived from Windows monotonic
uptime. The corresponding RakNet millisecond value is obtained by dividing by
`1000`. RakNet's own source contract likewise constructs connected
ping/connection times with `RakNet::GetTime`; its Windows implementation is a
platform monotonic clock, not the game's `qword_14D05D0`.

The current server parses the request as:

```text
00       u8      0x04 ID_CONNECTION_REQUEST
01..16   bytes   RakNet offline magic
17..24   u64-be  client GUID
25..32   u64-be  client RakNet send time U0
```

Recording bytes `25..32` is useful for RakNet synchronization. Naming that
value `clientTimeOrigin` does not make it `G0`.

## Trace correlation and rejected clocks

`bin/server/darkspin/logs/traces/client.jsonl` contains only a failed-login
session and no successful gameplay `GameState` sample. Its `time_ms` values
around `93,400,000` are diagnostic/platform time, not evidence for the
gameplay clock.

The successful live captures retained at
`bin/darkspinner/darkspin/logs/traces/game.jsonl` directly separate the
domains:

- In the 2026-07-23 capture, trace line `6521` receives
  `R0 = 27,783`. Once committed, line `11439` reports
  `R0 = 27,783`, `G0 = 19,549`, and `scale = 1`. Thus
  `G(r) = r - 8,234`.
- At that same line the diagnostic/platform clock is `150,805,562`, while
  gameplay `clock_now` is `38,047`. Windows/RakNet uptime is therefore not the
  gameplay clock.
- The prior 2026-07-22 capture recorded `R0 = 136,982`,
  `G0 = 129,330`, and offset `7,652`. The differing offsets reject a fixed
  server constant.
- Ability cooldown samples validate the mapper. For example, 2026-07-23 line
  `11439` converts source start `46,282` to local start `38,048`, exactly the
  recovered `-8,234` mapping up to the sampling millisecond.

These observations reject:

```text
D = WindowsUptimeNow + 30000
D = RakNetClientTimeNow + 30000
D = rawServerSourceNow + 30000
D = rawServerSourceNow - fixedConstant + 30000
```

They support only:

```text
D = G0 + trunc((Sdeploy - R0) * scale) + 30000
```

with a per-client `G0` that the recovered wire protocol does not return.

## Server-only fallbacks

There is no evidence-backed autonomous fallback that preserves both the exact
30-second native portrait wipe and a completely unmodified arbitrary remote
client. Two bounded server-only policies are implementable:

1. **Safe general fallback:** enforce the 30-second swap rule on the server's
   own monotonic clock and do not publish a future field-`4` value (leave it
   zero/expired). This preserves authority and avoids a falsely extended local
   lock, but the native portraits do not show the wipe. A rejected early C2S
   swap may receive the existing rejection response.
2. **Calibrated local-development fallback:** accept an explicit per-session
   `(R0, G0, scale)` calibration from the already-existing clock diagnostic or
   an operator-provided trace, then calculate the exact field with the formula
   above. This needs only server-side implementation and no new Fang rewrite,
   but it is out-of-band calibration, not a recovered retail protocol and
   must not be enabled as a general remote-client inference.

Do not use the connection-request timestamp as option 2's `G0`. If a future
raw capture proves equality for a particular run, it would still need to prove
the clock-producing call path, not merely numerical proximity.

The focused field-`4` wire shape remains:

```text
a1
<player-slot:u8>
<(0x0007 & ~(1 << deployed-slot)):u16-le>
for each selected reserve slot in ascending order:
    04 <D:u64-le> ff
```

Only the authority for `D` remains unrecovered. The two-reserve-slot shape,
field type, duration, and client consumption are unchanged.

## Evidence needed to replace this conclusion

Any of the following would close the remaining retail gap:

- a retail C2S capture containing a value equal to the simultaneously observed
  `G0` or `gameplayNow`;
- a client xref from `sub_9D2920`/`qword_14D05D0` into a C2S serializer not
  present in the recovered application vocabulary;
- retail server source showing a per-client gameplay-origin side channel;
- a retail S2C field-`4` packet paired with the client's mapped origins,
  proving how the backend authored its deadline.

Absent one of those, deriving `G0` from RakNet synchronization is an
underdetermined clock-domain substitution, not a recovered solution.
