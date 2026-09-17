# `zelems_1` native director and spawn boundary (build 103)

## Scope and provenance

This note records only facts visible in the canonical build-103 client
decompile, `bin/game/GameBin/Game.c`, and its paired executable. It
does not assign enemy-selection, count, budget, marker ownership, or send order
to the client. Those are authoritative-server policy and are not present in
this client-shaped binary.

- `Game.exe` SHA-256:
  `3c7248a4c6614450290a66c22ab7fcda86052b29a7b555e834d7c23f9b73ee5b`.
- `Game.c` SHA-256:
  `1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb`.
- Addresses below are image virtual addresses from the decompile. Decompile
  line references are included as navigation aids, not as a substitute for the
  addresses.

The executable's contiguous message-name data contains `kGmsGameState` at
`0x0102FD6C`, `kGmsDirectorState` at `0x0102FD7C`, and `kGmsObjectCreate` at
`0x0102FD90` (file offsets `0x00C2E36C`, `0x00C2E37C`, and `0x00C2E390`). The
transport attaches 78 entries in `sub_A8FAC0` (`0x00A8FAC0`; C lines
1546652-1546673), assigning generated one-byte transport IDs and retaining a
logical-message mapping. `sub_A8FE50` maps the received transport byte back to
that logical value before the gameplay dispatch switch at `sub_53ADC0`
(`0x0053ADC0`; C lines 381437-381572). Consequently the logical switch values
below are not packet bytes.

## Campaign director state

The client dispatches logical message `55` to `sub_538EE0` at
`0x00538EE0` (switch at `0x0053ADC0`, C lines 381536-381538). The handler obtains
the current simulator/director object through `sub_4E4DE0` and `sub_9BCFC0`,
then calls `sub_A1DF50` (C lines 379979-379989). `sub_A1DF50` selects reflected
type hash `0xD4DF4DA6` and passes it to the generic reflection decoder
`sub_A1D800` (`0x00A1D800`; wrapper at C lines 1446709-1446724). It reads no
object ID or other fixed prefix. The reflected field mask therefore follows the
application opcode immediately.

The application packet ID is **`0x8B`**. This is independently constrained by
the contiguous message names above and by the receiver-valid packets `8B 00`
(no changed fields) and `8B 08 01` (field 3 set true). Logical `55` is dispatch
metadata; it must not be serialized as the opcode.

The seven registered `cAIDirector` fields, in mask-index order, are:

| Index | Field | Director offset | Wire value when selected | Evidence/confidence |
| ---: | --- | ---: | --- | --- |
| 0 | `mbBossSpawned` | `+0x0D` | `bool` | High: it precedes the consecutively registered low flags; both constructors clear `+0x0D`. |
| 1 | `mbBossHorde` | `+0x0E` | `bool` | High: registration stores offset `0x0E` at `0x00F72961`; constructors clear it. |
| 2 | `mbCaptainSpawned` | `+0x0F` | `bool` | High: registration stores offset `0x0F` at `0x00F72A1F`; constructors clear it. |
| 3 | `mbBossComplete` | `+0x10` | `bool` | High: registration stores offset `0x10` at `0x00F72AE0`; `8B 08 01` selects exactly this field. |
| 4 | `mbHordeSpawned` | `+0x48C` | `bool` | High: registration stores offset `0x48C` at `0x00F72BA1`; constructors clear it. |
| 5 | `mBossId` | `+0x14` | `tObjID` / 4 bytes | High: registration stores offset `0x14` at `0x00F72C63`; constructors clear it. |
| 6 | `mActiveHordeWaves` | `+0x47C` | registered four-byte field/container handle | High for name, index, offset, and width; no claim is made about server-side element semantics. Registration stores width `4` and offset `0x47C` at `0x00F72CFF` and `0x00F72D28`. |

The registration closes at `0x00F72DB0-0x00F72DE0` with field count `7`, type
size `0x4D0`, and name `cAIDirector`. The two native initialization paths agree
on the defaults:

- `sub_9FB920` (`0x009FB920`; C lines 1419553-1419597) clears the low flags,
  `mBossId`, the active-wave storage, and byte `+1164` (`+0x48C`).
- `sub_9FD320` (`0x009FD320`; C lines 1421221-1421283) again clears offsets
  `+0x0C..+0x10`, `+0x14`, `+0x18`, the `+0x430..+0x490` region, and
  `+0x48C` before installing the simulator-owned director.

For the campaign setup currently reached after player status `8`, the
receiver-valid idle state is therefore exactly `8B 00`: opcode plus a zero
changed-field mask. This packet does not select a pool, activate a horde, or
spawn an enemy. The constructors establish initial state only.

### Negative native result

The executable contains no direct `cProtocolTransport::CreateMessage` call for
logical message `55`. The generic factory is `sub_A8FC00` at `0x00A8FC00` (C
lines 1546678-1546737); all 35 direct calls in the decompile use logical IDs
`0, 9, 16-19, 28-30, 36-41, 51, 59, 63, 66-69, 76, 77`, never `55`.
This is strong evidence that this binary is a director-state receiver, not the
missing authoritative campaign director sender. It does not rule out an
indirect generic replication path in another build or the retail server.

## Enemy spawn / object-create boundary

Build 103 does not expose an enemy-specific spawn packet. An enemy first enters
the client through ordinary **`kGmsObjectCreate`, application opcode `0x8C`**.
The dispatch switch routes its logical message `12` to `sub_53A760` at
`0x0053A760` (C lines 381130-381187; dispatch at lines 381458-381460).

The receive/construct path is:

1. `sub_53A760` reads a four-byte object ID and rejects an already-resolved ID.
2. `sub_A1DB90` (`0x00A1DB90`) initializes a 47-byte
   `cGameObjectCreateData` image (C lines 1446574-1446595).
3. `sub_A1DB60` (`0x00A1DB60`) reflection-decodes type hash `0x37748BD7`
   into that image (C lines 1446565-1446572).
4. `sub_9D6910` (`0x009D6910`) translates it into the simulator creation
   arguments and calls `sub_9D1DF0` (C lines 1387307-1387394).
5. If the created object has the network component at object offset `+680`,
   `sub_A1E710` reflection-decodes type hash `0x258CF09D`
   (`sporelabsObject`), installs the supplied object ID in the component, marks
   the object network-ready, and invokes `sub_9E6930` (C lines
   381172-381181 and 1447101-1447169).

The fixed create structure has ten fields. Offsets are native structure
offsets; selected wire widths exclude native alignment:

| Fixed index | Field | Native offset | Wire width | Default from `sub_A1DB90` |
| ---: | --- | ---: | ---: | --- |
| 0 | noun | `+0x00` | 4 | `0` |
| 1 | position | `+0x04` | 12 | global zero vector |
| 2 | rotation X (degrees) | `+0x10` | 4 | `0` |
| 3 | rotation Y (degrees) | `+0x14` | 4 | `0` |
| 4 | rotation Z (degrees) | `+0x18` | 4 | `0` |
| 5 | asset ID | `+0x20` | 8 | `0` |
| 6 | scale | `+0x28` | 4 | `1.0f` |
| 7 | team | `+0x2C` | 1 | `0` |
| 8 | `hasCollision` | `+0x2D` | 1 | `true` |
| 9 | `playerControlled` | `+0x2E` | 1 | `false` |

All ten fields use mask `0x03FF`; their serialized values occupy 43 bytes. The
wire prefix is thus:

```text
8C
<objectId:u32-le>
FF 03
<noun:u32-le>
<position:3xf32-le>
<rotationDegrees:3xf32-le>
<assetId:u64-le>
<scale:f32-le>
<team:u8> <hasCollision:u8> <playerControlled:u8>
```

The following `sporelabsObject` reflection has 23 registered indices:

| Indices | Fields |
| --- | --- |
| 0-3 | team, player-controlled, input stamp, player index |
| 4-7 | linear velocity, angular velocity, position, orientation |
| 8-12 | scale, marker scale, last animation state/time, movement-animation override |
| 13-17 | graphics state, two graphics-state times, visibility, collision |
| 18-22 | owner ID, movement type, disable repulsion, interactable state, source marker ID |

For the existing receiver-valid ordinary enemy snapshot, the tail selects field
`6` position and field `7` orientation, with quaternion identity
`(0,0,0,1)`, then terminator `0xFF`. Combined with zero rotation/asset/team,
scale `1`, collision chosen from the noun/placement, and not-player-controlled,
this is an 81-byte application packet including opcode. Confidence is **high**
for decoding and client construction because `sub_53A760`, `sub_A1DB60`,
`sub_9D6910`, and `sub_A1E710` execute that exact receive path.

Confidence is only **receiver-valid, not sender-proven** for selecting precisely
tail fields `6` and `7` for every campaign enemy. The binary has no direct
`sub_A8FC00` call for logical `12`, and the ordinary native creation routines
(`sub_9D6910`, `sub_9D6A30`, and Lua-facing `sub_A07F40`) construct simulator
objects without constructing a GMS message. The authoritative server may dirty
additional `sporelabsObject` fields and must separately replicate combatant,
attribute, locomotion, modifier, or other components. `ObjectCreate` itself
decodes only `cGameObjectCreateData` plus `sporelabsObject`; it does not carry
those component reflections.

## Conclusions constrained to native evidence

- Status `8` is a session boundary outside these two packet decoders. Nothing
  in the recovered client constructors binds it to a `zelems_1` pool, marker,
  count, budget, difficulty band, or weight.
- `8B 00` is a valid idle seven-field director snapshot. It communicates no
  active encounter state.
- `0x8C` is the generic object-create family used to materialize an enemy, but
  the client binary does not contain the missing campaign enemy-selection or
  outbound spawn constructor.
- Pool choice, marker choice, object IDs, spawn count, component-update order,
  RakNet reliability/channel, late-join replay, and activation timing remain
  unproven by `Game.c` and must be sourced from retail server code or a
  capture. No policy is inferred here.
