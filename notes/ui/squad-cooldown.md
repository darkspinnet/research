# Build 103 reserve portrait cooldown contract

## Result

Build 103 does **not** drive the reserve-creature portrait wipes with
`CooldownUpdate` (`0xC1`). Each of the three `LabsPlayer` character records has
its own deploy-cooldown deadline. `HUD_PlayerDeck.swf` polls all three records
every frame and calls `SetCreatureCooldownPercent(slot, fraction)`. A sparse
`LabsPlayerUpdate` (`0xA1`) character reflection, field `4`, is the runtime wire
mechanism that changes that deadline.

Consequently:

- Do not send `0xC1` packets addressed to reserve object IDs for portrait
  cooldowns. The `0xC1` object ID is only an ownership/validity gate before the
  packet updates a player-global ability-cooldown map.
- Ability points/ranks or other replicated ability availability do not drive
  the portrait wipe.
- `PlayerCharacterDeploy` (`0xA7`) selects the deployed character and refreshes
  the player/action HUD, but it does not set the portrait deadline.
- No separate UI event drives the wipe. A blocked-switch toast is synthesized
  locally from the same deadline.
- To put both reserve portraits on cooldown, update field `4` on both reserve
  character records. Their absolute deadlines then continue counting down in
  reserve without further packets or another deploy refresh.

The client proves the presentation contract but not the retail server's choice
of which slots received a deadline after each swap. The conservative policy for
a global 30-second swap lock is to put the same future deadline on both reserve
slots. Setting only the character just left would prevent only the immediate
swap back; the third character would remain locally selectable.

That presentation policy is not directly usable by a standalone server: the
deadline is in a private client gameplay-clock domain whose additive origin is
never sent back over the recovered application or RakNet vocabulary. Darkspin
therefore enforces the global lock in its squad session and writes zero to field
`4`, avoiding a stuck or incorrectly extended portrait lock. See
`notes/ui/squad-cooldown-authority.md` for the exhaustive clock-boundary audit.

## Portrait reader

Canonical build-103 evidence is in
`bin/game/GameBin/Game.c`:

- `sub_420910` (`0x00420910`) is the `HUD_PlayerDeck.swf` frame update. It loops
  slots `0`, `1`, and `2`, updates each portrait's health/mana and enabled state,
  and calls `sub_41E2B0` for every slot.
- `sub_41E2B0` (`0x0041E2B0`) reads the low 32 bits at
  `labsPlayer + 1512 * slot + 936`, subtracts the current mapped gameplay clock,
  clamps the remainder at zero, divides milliseconds by the configured duration
  in seconds, clamps at one, and calls
  `HUD_PlayerDeck.swf.SetCreatureCooldownPercent(slot, fraction)`.
- In formula form,
  `fraction = min(max(deadlineLow32 - nowLow32, 0) / 1000 / durationSeconds, 1)`.
  The value begins at `1.0` and falls to `0.0`.
- `sub_9CF190` (`0x009CF190`) loads `durationSeconds` into `dword_1164AD8`
  from tuning property hash `115251375` (`0x06DE98AF`), with a `30.0` second
  fallback.

The reflection/storage type is a `uint64`, although these two HUD/input readers
use its low 32 bits. It is an **absolute gameplay-clock deadline in
milliseconds**, not a duration, Unix timestamp, or seconds value. Unlike
`0xC1`, the portrait reader performs no `sourceStart` conversion; the deadline
must already be in the gameplay clock domain established by the game-state time
mapping.

More precisely, both readers take the signed low-word difference
`int32(deadlineLow32 - nowLow32)`. Build 103 therefore relies on ordinary
near-future deadlines; the high word is retained by reflection but is not part
of either HUD/input comparison.

`sub_9C2B30` (`0x009C2B30`) validates a requested character switch. After its
range, life, and status checks, it compares the requested slot's same
`+936` deadline against the current gameplay clock and returns status `1` while
the deadline is in the future. `sub_4DF380` (`0x004DF380`) maps status `1` to
local client event `0xA2BC2E34` (`Hero switching is on cooldown.`). That event
is a rejection message, not the overlay source, and the server need not send it
to create the wipe.

Because the HUD polls every slot against the clock, portrait cooldowns continue
while a hero is in reserve. Expiration is passive: no zero/reset packet is
needed unless the server wants to cancel the deadline early.

## `LabsPlayerUpdate` wire contract

The recovered reference serializer under
`bin/game/logs/recap_server-reference/game_server/source` supplies names
and the reflection order. Its full-record offsets are stale for build 103:

- `game/character.h` names character reflection field `4` `DeployCooldown` and
  stores it as `uint64_t mDeployCooldown`.
- `game/character.cpp` writes it at byte offset `0x3C0` in its reference full
  record and as field `4` in a sparse character reflection. Clean build-103
  experiments proved the fixed type/ability region is `0x30` earlier than that
  reference layout.
- The adjacent per-creature ability state is field `5` `AbilityPoints`
  (`uint32`, full-record offset `0x3C8`) and field `6` `AbilityRanks` (nine
  `uint32` entries beginning at `0x3CC`). Neither is read by
  `SetCreatureCooldownPercent` or the switch-cooldown predicate.
- `game/player.h` assigns update-bit `0x0001` to slot 0, `0x0002` to slot 1,
  and `0x0004` to slot 2. `PlayerBits` is `0x1000`.
- `raknet/server.cpp::SendLabsPlayerUpdate` writes packet ID, player slot,
  little-endian update bits, an optional player reflection first when bit
  `0x1000` is set, then selected character reflections in ascending slot order,
  then selected catalyst reflections.

For only reserve slots 1 and 2, the application packet is:

```text
a1
<player-slot:u8>
06 00
04 <slot-1-deadline:u64-le> ff
04 <slot-2-deadline:u64-le> ff
```

The update mask identifies which character record each following reflection
belongs to; a character object ID is neither present nor required. If a
different pair is in reserve, use the corresponding two bits and still emit
the selected reflections in slot order. Each `ff` terminates one character
reflection. The exact build-103 fixed offset for deploy cooldown is not
recovered from the stale reference image; sparse field `4` is the focused
runtime update.

## Deploy and controlled-object messages

`PlayerCharacterDeploy` (`0xA7`) is fixed-size and has this payload order:

```text
<player-slot:u8> <creature-index:u32-le> <object-id:u32-le>
```

`sub_53ACA0` (`0x0053ACA0`) decodes exactly nine bytes. After resolving the
object, it writes the player's current deck index, clears the queued index,
sets the controlled creature handle, and refreshes selection/action-HUD state.
It never writes the per-character `+936` deploy deadline.

The top-level `LabsPlayer` controlled-object reflection is independently field
`9`, currently encoded as:

```text
a1 <player-slot:u8> 00 10 09 <object-id:u32-le> ff
```

That field establishes the controlled-object identity used by combat and local
presentation. It does not alter any character's deploy cooldown. Top-level
player field `1` is the current deck index; `0xA7` itself also writes the deck
index and controlled-object handle in the client, so neither top-level field is
the portrait-cooldown source.

There is no client dependency ordering between `0xA7` and character field `4`:
the next HUD tick reads whichever deadline is present. For a newly introduced
target object, the safe complete transition order is:

1. create/initialize the target creature object;
2. publish `LabsPlayer` controlled-object field `9`;
3. publish `PlayerCharacterDeploy` (`0xA7`);
4. publish the selected character field-`4` deadlines in `LabsPlayerUpdate`.

The recovered reference `Instance::SwapCharacter` likewise broadcasts
`PlayerCharacterDeploy` and then sends its aggregate `LabsPlayerUpdate`.
Its `Player::SwapCharacter` explicitly leaves cooldown checks as a TODO, so it
is useful for serialization/order evidence but is not proof of the retail
slot-assignment policy. The portrait update is state-based and can technically
arrive on either side of `0xA7`; sending it after `0xA7` matches that recovered
sequence. Steps 1-3 above are object/control safety, not prerequisites for the
overlay; only the per-character field-`4` reflection creates or changes it.

## Why `CooldownUpdate` (`0xC1`) is different

`sub_53A520` (`0x0053A520`) decodes an exact 36-byte payload:

```text
<object-id:u32-le>
<ability-key:u64-le>
<duration-ms:i64-le>
<source-start-ms:i64-le>
<global-cooldown-ms:i64-le>
```

It resolves `object-id` and requires the object's player-controlled flag at
object offset `+92`. Once that gate passes, the object is discarded and
`sub_4D6320` (`0x004D6320`) updates the singleton action/cooldown state:

- the map is at singleton offset `+84` and is keyed only by the 64-bit ability
  key;
- each entry holds converted start and end times plus an active/dirty byte at
  entry offset `+16`;
- `source-start-ms` is converted through the game clock mapper and duration is
  added to form the end time;
- `global-cooldown-ms` updates the singleton global deadline at `+576` and its
  active byte at `+584`.

`sub_4222F0` (`0x004222F0`) is the ability-bar consumer. It resolves the ability
key for the current player's HUD index, reads this singleton map through
`sub_421F50`, computes remaining/duration using `sub_4D67E0` and `sub_4D68A0`,
and calls `SetAbilityCooldownPercent`. There is no creature identity in the map
key and no lookup of a reserve character record. Sending the same ability key
with another controlled reserve object ID therefore targets the same shared
entry; it cannot create a reserve portrait wipe or distinct per-hero ability
cooldown. Conversely, a reserve object's ID can pass the packet's gate if that
hidden object still has its player-controlled flag, but this changes only
whether the shared update is accepted, not where it is stored.

The remaining-time helper has three non-visual call paths as well:

- `sub_4DEB10` (`0x004DEB10`) exposes the boolean "HUD ability index is on
  cooldown" predicate.
- `sub_4DF1F0` (`0x004DF1F0`) uses that predicate during command preflight and
  returns the cooldown rejection when more than its small input-buffer grace
  remains (`1000` ms for nonzero ability indices, `250` ms for index zero).
- `sub_4E54D0` (`0x004E54D0`) is a lower action validator and rejects a
  non-exempt ability whenever `sub_4D67E0(abilityKey) > 0`.

These checks merge the keyed ability deadline with the singleton global
deadline where the ability is subject to global cooldown. They still resolve
the ability through the current `LabsPlayer`/action context; none addresses or
stores a reserve creature.

`sub_4D6150` resets action/cooldown state during an object/action-context
refresh. Before clearing the map it saves only the entries whose current HUD
ability keys resolve at indices `6`, `7`, and `8`, then restores those entries;
normal hero ability entries are cleared. Character reflection fields `5`
(`AbilityPoints`) and `6` (`AbilityRanks`) describe availability/rank, not
cooldown state, and no reader copies them into the portrait deadline.

Thus the two kinds of cooldown have different reserve behavior:

- **Deploy/portrait cooldown:** per-character absolute deadline; all three
  records are polled, so it visibly continues in reserve.
- **Combat ability cooldown:** singleton current-action HUD state keyed by
  ability, not a per-character reserve state. Build 103 provides no client-side
  evidence that ordinary hero ability cooldowns continue independently and
  visibly while that hero is in reserve. A server that maintains such authority
  should republish the relevant ability cooldown when that hero is redeployed;
  reserve object-targeted `0xC1` packets do not establish separate state.

## Walkthrough cross-check

`bin/video/walkthrough/1-1/1-1.mkv` and its 5-second contact sheets cover the
campaign run. At every sampled gameplay frame in `contact-5s-01.png` through
`contact-5s-05.png`, the upper-left active portrait/current creature remains the
same and the Q/W/E portrait selection does not visibly change. The sampled key
frames, including the point where all three Q/W/E slots are already available,
also do not show a completed swap during an active cooldown. Five-second
sampling cannot exclude an unusually brief transition between samples, but
there is no visible before/after hero change to motivate such a window. The
walkthrough therefore neither confirms nor contradicts the recovered deadline
behavior; it does not visibly exercise the requested swap-during-cooldown case.

## Confidence and remaining gap

Confidence is high for the HUD reader, reflection field, payload layout,
countdown behavior, and separation from `0xC1`: each is directly evidenced by
the build-103 client and serializer. Confidence is intentionally lower for the
retail policy that chooses which character deadlines to set after a swap. No
retail packet capture or visible walkthrough swap was found, and the recovered
server leaves that policy unimplemented. A retail capture containing `0xA7`
plus the following `0xA1` character reflections would settle whether retail set
both reserves, only the departed hero, or another combination.
