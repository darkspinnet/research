# Build 103 squad portrait cooldown clock

## Result

Character reflection field `4` is an unsigned, little-endian `uint64` absolute
deadline in the **client's local gameplay-clock milliseconds**. It is not a
duration and it is not in the server/RakNet source-clock domain. Build 103
stores all 64 bits, but both known consumers compare only the low 32 bits by
signed subtraction:

```text
remaining = int32(deadlineLow32 - localGameplayNowLow32)
isCoolingDown = remaining > 0
fraction = clamp(max(remaining, 0) / 1000 / 30, 0, 1)
```

There is no remote-to-local conversion in the field-`4` reflection decoder,
portrait reader, or switch validator. The sender must convert the deployment
instant into the client's local gameplay clock before adding the 30,000 ms
duration.

For a successful deployment to slot `d`, set the same future deadline on both
other character slots and do not set it on `d`:

```text
localNow = remoteToLocal(packet.SourceTime)
deadline = localNow + 30_000
updateMask = (0b111 & ~(1 << d))
field4[each selected slot] = uint64LE(deadline)
```

At the scale `1` used by the trace, the directly implementable form is
`deadline = packet.SourceTime - (remoteOrigin - localOrigin) + 30_000`.

Both reserve slots are necessary for a global swap lock. Updating only the
departed hero leaves the third hero selectable. The newly deployed slot does
not need a new deadline; on an accepted transition its prior reserve deadline
has already expired. Expiration is passive and requires no clearing packet.

Confidence is **high** for the field type, units, comparison clock, lack of an
implicit conversion, wire encoding, and two-reserve-slot rule. Confidence is
**high for the trace's scale-1 conversion and exact 7,652 ms error**. The
general scale formula is directly recovered from the converter. It is not
server-computable from build-103 wire traffic, however: GameState establishes
the remote source origin but the client never returns the paired local
gameplay-clock origin. `notes/ui/squad-cooldown-authority.md` records that
closed protocol boundary and the zero-deadline compatibility policy.

## Client evidence and addresses

- `sub_53ADC0` at `0x0053ADC0` dispatches gameplay messages. Its
  `LabsPlayerUpdate` route reaches `sub_539E00` at `0x00539E00`.
- `sub_539E00` reads the player slot and calls the nested Labs-player reflection
  decoder `sub_A1E070` at `0x00A1E070`.
- `sub_A1E070` selects character bits `0`, `1`, and `2`, walks three character
  records at a stride of `1512` bytes, and calls generic reflected-type decoder
  `sub_A1D800` at `0x00A1D800` for each selected record. No clock conversion is
  performed on this route.
- The recovered character descriptor/serializer names reflected field `4`
  `DeployCooldown`, declares it `uint64_t`, writes it at fixed-record offset
  `0x3C0`, and sparse-serializes it as field tag `04` followed by eight
  little-endian bytes. The canonical files are
  `bin/game/logs/recap_server-reference/game_server/source/game/character.h`
  and `character.cpp`.
- `sub_420910` at `0x00420910` polls all three squad portraits every frame and
  calls `sub_41E2B0` at `0x0041E2B0` for each slot.
- `sub_41E2B0` reads the deadline low word at player-relative
  `+936 + 1512 * slot`, obtains the local gameplay clock through
  `sub_4E4DE0` (`0x004E4DE0`) and `sub_9D2920` (`0x009D2920`), performs the
  signed low-word subtraction above, divides milliseconds by `1000` and the
  configured duration, clamps at `1`, and calls
  `SetCreatureCooldownPercent`.
- `sub_9CF190` at `0x009CF190` loads the duration from tuning hash
  `0x06DE98AF` (`115251375`) with a `30.0` second fallback.
- `sub_9C2B30` at `0x009C2B30` validates a requested slot against the same
  `+936 + 1512 * slot` low word and the same local gameplay clock. It returns
  status `1` when `int32(deadlineLow32 - nowLow32) > 0`; this is why the bad
  deadline delays both the wipe and actual selection eligibility.

The reflected storage and wire type are unsigned 64-bit. The effective
comparison is nevertheless a signed 32-bit modular difference. Deadlines must
therefore remain ordinary near-future values within `INT32_MAX` ms of the
comparison clock; the 30-second contract is safely inside that window. A high
word is accepted and retained but cannot extend either known build-103
consumer past low-word wrap semantics.

## Remote-to-local clock conversion

`GameState` receiver `sub_537CE0` at `0x00537CE0` passes its first `uint64`
source timestamp to `sub_9D2830` at `0x009D2830`. That marks a pending remote
clock sample. `sub_9D29C0` at `0x009D29C0` captures the remote and local origins,
and `sub_9D2A00` at `0x009D2A00` converts a nonzero remote timestamp `r` as:

```text
remoteToLocal(r) = localOrigin + trunc((r - remoteOrigin) * clockScale)
remoteToLocal(0) = 0
```

This converter is used by the combat ability-cooldown path
(`sub_4D6320`, `0x004D6320`) before it stores a `CooldownUpdate` source start.
It is conspicuously absent from character field `4`. The correct deploy value
is `remoteToLocal(sourceNow) + 30_000`, matching the ability path's rule of
converting the start and then adding an unscaled millisecond duration. Do not
use raw `sourceNow + 30_000`; and, if a measured scale is not exactly one, do
not convert the already-future `sourceNow + 30_000`, because that scales the
authored 30-second duration as well as the clock origin.

## 10:23 Fang trace correlation

The trace used for this analysis was
`bin/darkspinner/darkspin/logs/traces/game.jsonl` during the 2026-07-22
10:23 PDT playtest.

After clock initialization, every traced `cooldown_apply` sample reports the
stable mapper:

```text
remoteOrigin = 136982 ms
localOrigin  = 129330 ms
clockScale   = 1
remote - local offset = 7652 ms
```

For example, at trace line `73115`, remote `source_start=334178` becomes local
`start=326526`, exactly `334178 - 7652`. The simultaneously sampled local
`clock_now` is also `326526`. This independently validates the converter and
the offset immediately before the final deployment.

The four successful swaps in this run contain these exact field-`4` updates:

| Trace line/time | Deployed slot | Mask / updated reserve slots | Sent deadline | Correct scale-1 deadline |
| --- | ---: | --- | ---: | ---: |
| `42081` / `86749203` | 1 | `0x0005` / 0, 2 | `216811` (`0x034EEB`) | `209159` (`0x033107`) |
| `51622` / `86798640` | 2 | `0x0003` / 0, 1 | `266238` (`0x040FFE`) | `258586` (`0x03F21A`) |
| `57788` / `86829984` | 0 | `0x0006` / 1, 2 | `297587` (`0x048A73`) | `289935` (`0x046C8F`) |
| `73439` / `86897625` | 1 | `0x0005` / 0, 2 | `365231` (`0x0592AF`) | `357579` (`0x0574CB`) |

Each sent deadline is `server source now + 30,000`. Subtracting the measured
`7,652` ms clock offset yields the local-domain value the client actually
expects. For the final swap, server source now was `335231`; its mapped local
instant was `327579`, so the exact correct deployment value was:

```text
327579 + 30000 = 357579 = 0x00000000000574CB
uint64 little endian = cb 74 05 00 00 00 00 00
```

Instead, the wire carried `365231`:

```text
a1 00 05 00
   04 af 92 05 00 00 00 00 00 ff
   04 af 92 05 00 00 00 00 00 ff
```

The low-word HUD calculation therefore began at approximately
`(365231 - 327579) / 30000 = 1.255`, which clamps to `1.0`. The wipe cannot
start visibly changing until the surplus `7,652` ms elapses, matching the
10:28 observation that it appeared noticeably after the accepted swap without
a rejected attempt. The switch validator uses the same subtraction, so it also
keeps both reserve heroes blocked for about `37.652` seconds rather than
`30.000` seconds. This predicts an extra `7.652` seconds of rejection around
normal expiry. It does not by itself explain an attempt made later than
`37.652` seconds after deployment; the trace contains no later deploy event,
and the playtest note correctly leaves dead-reserve or other admission state as
a separate possibility.

## Exact focused wire rule

For player slot `p`, deployed character `d`, and local deadline `D`, send:

```text
a1
<p:u8>
<(0x0007 & ~(1 << d)):u16-le>
for slot 0..2 selected by the mask, in ascending order:
    04 <D:u64-le> ff
```

For the final traced transition (`p=0`, `d=1`) with the corrected value, the
packet would be:

```text
a1 00 05 00
   04 cb 74 05 00 00 00 00 00 ff
   04 cb 74 05 00 00 00 00 00 ff
```

The deployment packet `0xA7` changes the active character but does not write
this field. Publishing field `4` after `0xA7`, as in the recovered sender and
the live trace, is valid; the portrait state is picked up on the next HUD tick.
