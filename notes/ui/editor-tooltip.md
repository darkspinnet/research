# Campaign editor tooltip

## Result

Build 103's one-shot owner is the packaged `PickUpLoot` ability's
`nGameSimulator.SetFirstLootPickedUp()` predicate. It is **not**
`new_player_progress`, `new_player_inventory`, an `unlock_*` account field, or a
predicate in the web/Flash renderer.

The one-shot is stored only in the live game simulator: a byte at simulator
offset `+0x39E89` (`237193`). The native reads the old byte, writes `1`
immediately, and returns the old state. Packaged pickup Lua emits
`loot_acquiredFirstTime.ServerEventDef` only when that returned old state is
false. Consequently the flag suppresses repeats within one simulator, but a
fresh campaign simulator can begin false and present the guidance again even
for an account whose onboarding is complete.

There is no recovered packet that sets or acknowledges this seen state after
the popup plays. The `0x9b` packet is the presentation event; it does not carry
a persistent `hasSeenFirstLoot` field. No recovered code copies the simulator
byte to the account, and no account response field has been tied to it.

Confidence in the predicate, storage location, and session lifetime is
**high**. Confidence that the reported 2026-07-22 campaign popup was reached
through this exact event is **medium-high**, because the trace records socket
digests and truncated response prefixes rather than the complete event stream.

## Exact authored path

The authoritative runtime `content.db` resolves
`Abilities/0x238E6F16.lua` to Lua chunk `610`, server-data resource `14174`,
1,680 bytes, SHA-256
`4ac582bd4a70c229f248c21bfde74a5d252b9d40cc72fcb4fad0d99932a38fd4`.
Its indexed constants include, in execution order:

- `nAbility_PickUpLoot`;
- `loot_acquiredFirstTime.ServerEventDef`;
- `nGameSimulator`, `SetFirstLootPickedUp`;
- `asset`, `objectId`, `nEvent`, `Notify`;
- `PlayersRollForLoot`, then `MarkForDelete`.

The recovered release path is therefore:

1. Validate that the targeted loot object still exists and is not marked for
   deletion.
2. Call `SetFirstLootPickedUp()`.
3. If the returned prior state is false, call
   `nEvent.Notify({asset = loot_acquiredFirstTime.ServerEventDef, objectId =
   targetID})`.
4. Run `PlayersRollForLoot`, delete the pickup, and finish its animation wait.

`Game.c` independently closes the predicate:

- registration at line `1432179` binds `SetFirstLootPickedUp` to
  `sub_A05DE0`;
- `sub_A05DE0` at line `1427284` defaults the result to true, and only in a
  one-player simulator calls `sub_9BD140` and returns whether its result was
  nonzero;
- `sub_9BD140` at line `1366197` reads the byte at `simulator + 237193`, writes
  `1` to that same byte, and returns the old byte.

The multiplayer default-to-true branch is another guard against presenting
the first-loot lesson there. The relevant state is thus a simulator-local
boolean-equivalent latch, not tutorial completion stored on the user.

## Event and packet

The asset catalog maps
`loot_acquiredFirstTime.ServerEventDef` to `0x466263AF`. `nEvent.Notify` reflects
the authored table into the build-103 `ServerEvent` message (`0x9b`). Its
minimal application-message form is:

```text
9b 06 af 63 62 46 07 <loot object ID, little-endian uint32> ff
```

Field `6` is the event asset; field `7` is the object ID; `ff` terminates the
reflected fields. This packet **presents** the first-time event. It neither
sets the `+0x39E89` byte nor reports that the UI finished displaying it. The
byte has already been set by `SetFirstLootPickedUp()` before `Notify` is
called.

This is distinct from normal item acquisition:

- `loot_acquired.ServerEventDef` is `0x7DC8A374`;
- `LootAwarded` is client event `0x615F3861` and carries fields `17` through
  `25` containing the loot tail;
- normal world presentation uses `loot_spawn.ServerEventDef`, object creation,
  `cLootData`, and movement.

Those ordinary messages/cards must remain available. Suppressing them would
hide or break real loot rather than only the stale lesson.

## Display ownership

English Text-package table `Labs Popup Localization` maps key `0x3E794A2A` to:

> Items may be used in the Editor to upgrade your heroes.

The playtest wording “update items” is a paraphrase; the packaged build-103
string says “upgrade your heroes.” The adjacent loot key `0x3E794A2B` says
that Game sometimes drop items.

`HUD_Toaster.swf` is a generic renderer. Native `sub_41F290` loads it, and
`sub_41F2E0` drains queued 332-byte alert records into `AddToastAlert` with
seven arguments. Decompressed Flash UI contains `HUD_Toaster`, `ToastAlert`,
and `AddToastAlert`, but no first-loot asset name, localization key, onboarding
field, or gating predicate. Packaged web account pages read progression and
inventory fields for profile state but do not own this gameplay alert.

The content/native split is therefore:

```text
PickUpLoot Lua -> simulator first-loot latch -> first-time ServerEvent ->
native alert queue/localization -> generic HUD toaster
```

## Account and trace checks

The affected `Test` account at the reported run had:

- `onboarding_progress = 9000`;
- `new_player_inventory = 1`;
- level `4`, XP `3001`;
- `unlock_inventory_identify = 180`.

These values disprove incomplete account onboarding as a necessary condition.
The database's separate `unlock_inventory = 0` is also not reliable evidence
of the wire value: the current account XML serializes both `unlock_inventory`
and `unlock_inventory_identify` from `UnlockInventoryIdentify` in
`server/game/api.go` lines `646` and `651`.

The 2026-07-22 playtest places the popup in ordinary campaign 1-1 after the
failed first obelisk interaction and around the first loot/switch sequence.
The current JSONL trace exposes socket hashes and selected client-state hooks,
including `loot_resource_*`, but not the complete reflected `0x9b` payload or
the native `AddToastAlert` arguments. The server log records only a response
prefix, which may end before a batched event. Absence of the literal asset hash
from those text logs is therefore not an end-to-end negative packet capture.

Current darkspin source does not explicitly emit
`loot_acquiredFirstTime.ServerEventDef`. That leaves two viable locations for
the observed path: client-side simulator execution of the packaged pickup
ability, or an event hidden inside an incompletely captured batch. It does not
change the recovered one-shot owner or its storage semantics, but it matters
to where a suppression hook can actually work.

## Smallest safe server-side suppression

The narrow policy is:

> In ordinary campaign mode, suppress only
> `loot_acquiredFirstTime.ServerEventDef` (`0x466263AF`); retain it in tutorial
> mode, and retain all normal loot spawn, acquisition, inventory, and
> `LootAwarded` traffic in every mode.

If the event crosses the server's presentation-intent/event bridge, this is a
single asset-and-mode filter. It avoids global `showConfigAlerts` suppression,
does not falsify account onboarding, and preserves legitimate tutorial
guidance.

If a full capture proves that build 103 generates the event wholly inside its
local simulator, dropping server egress cannot suppress it. In that case the
smallest server-controlled equivalent is to initialize the client's
first-loot latch as already true for campaign and false for tutorial. No
build-103 packet that performs that initialization has been recovered, so one
must not invent a progression field or reuse `new_player_inventory` for it.
Until such an initialization route is found, changing the shape/timing of
ordinary loot solely to avoid the client predicate would be broader and less
safe than the event filter.

## Explicit unknowns

- The constructor/reset site that initializes simulator byte `+0x39E89` was
  not identified. Its fresh-simulator false state is inferred from the atomic
  first-use behavior and repeated-session symptom, not from that constructor.
- A complete wire capture of the reported popup is still needed to distinguish
  a server-transmitted `0x466263AF` event from local client simulator
  execution.
- The exact native mapping step from event `0x466263AF` to locale key
  `0x3E794A2A` is packaged/data-driven and was not recovered as a hard-coded
  comparison in `Game.c`.
- `new_player_inventory` is known to become `1` after a collection/creature
  condition, but its full retail semantics remain unknown. There is no
  evidence that it backs the first-loot one-shot.
- No playback-complete acknowledgement, durable seen bit, or account update
  for this alert was found in current traces, content, web/Lua, or native code.
