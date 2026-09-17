# Teleporter live modifier contract (build 103)

## Scope and conclusion

This audit uses only the checked-out workspace, the authoritative
`bin/darkspinner/darkspin/cache/content.db`, and
`bin/darkspinner/GameBin/Game.c`. No recap source or output was
inspected.

The exact accepted boundary is:

1. chunk 144 requests modifier GUID `0x502f1932` on the entering player, with
   the teleporter as initiator, `kObjIDNone`, rank `1`, and destination
   `(x, y, z)` as float properties 0-2;
2. native build 103 allocates a modifier-instance handle, publishes logical
   message 36 / wire `0xa2` `ModifierCreated`, and only then invokes chunk
   349 `Activate`;
3. chunk 349 stops locomotion, applies `Immobilized += 1`, publishes the
   teleport-out animation, waits `0.5s`, publishes `0x90` teleport, performs
   its authored zero-duration wait, publishes teleport-in, waits another
   `0.5s`, and returns;
4. ordinary completion publishes nothing and does **not** delete either the
   modifier or its immobilization contribution;
5. a distinct modifier-lifecycle owner must remove the modifier. That removal
   runs deactivation/attached-attribute cleanup and then publishes logical
   message 38 / wire `0xa4` `ModifierDeleted(target, instanceID)`.

Consequently, the normal observed packet order is:

```text
0xa2 ModifierCreated
0xa5 character_teleport_out
0x90 ObjectTeleport                         (+0.5s)
0xa5 character_teleport_in                  (after the zero-duration yield)
<no packet when chunk 349 returns>          (+1.0s total)
0xa4 ModifierDeleted                        (only if/when another owner removes it)
```

`ModifierDeleted` is not intrinsically tied to `+1.0s`. If an external owner
tears the modifier down early, deletion can pre-empt the remaining coroutine;
if no owner removes it, there is no eventual `0xa4` merely because chunk 349
finished.

## Content and bytecode proof

`darkrun db lua_chunk get` identifies the authoritative rows:

| Chunk | content.db source | Resource | Size | SHA-256 |
|---:|---|---:|---:|---|
| 144 | `Modifiers/0xA36FCF98.lua` | 13674 | 5,449 | `c2e727c6cf4c13708fa315321fb51f0402a3a4a9763eaf2f8d782c1eca3267f0` |
| 349 | `Modifiers/0x8B5016B1.lua` | 13896 | 1,378 | `9417dfe9163dbb857dc1b423d4e127170a6ba9e128f583d648fe7f258a1c1cd1` |

The extracted bytecode files used for disassembly hash identically to those
rows. Chunk 349 has no `lua_dependency` or `lua_module_alias` rows.

### Chunk 144: request owner

The template constants prove:

- `teleportModifier = nUtil.ToGUID("0x502f1932")`;
- the trigger radius is `2`;
- trigger-enter calls the request closure;
- trigger-exit is empty;
- trigger-stay first calls `GetFirstModifierByGUID(target, 0x502f1932)` and
  requests only when none exists.

Prototype `0.1`, PCs 12-27, obtains
`GetTeleporterDestination(teleportObjectID)` and calls:

```text
nModifier.RequestModifier(
    entrant,
    teleportObjectID,
    0x502f1932,
    kObjIDNone,
    1,
    destinationX,
    destinationY,
    destinationZ)
```

This proves target, initiator, GUID, no ability-instance owner, rank, and the
three float properties at the chunk-144 side of the boundary.

### Chunk 349: activation program

Chunk 349 sets `requiresAgent=true`, `activationType=Unique`, and registers
`TeleporterModifier`. Its timing constants are:

- `beamOutTime = 0.5`;
- `teleportDelay = 0` (the value assigned from the same zero constant);
- `beamInTime = 0.5`.

Prototype `0.0`, PCs 0-20, reads the current agent, current modifier instance
ID, and float properties 0-2. PCs 21-65 then execute, in order:

```text
nLocomotion.Stop(agent)
nAttribute.AddAttributeModifier(agent, Immobilized, 1)
nGameObject.SetAnimationState(agent, character_teleport_out)
nThread.WaitForXSeconds(0.5)
nLocomotion.TeleportObject(agent, property0, property1, property2)
nThread.WaitForXSeconds(0)
nGameObject.SetAnimationState(agent, character_teleport_in)
nThread.WaitForXSeconds(0.5)
return
```

Prototype `0.1` (`Deactivate`) is a bare return. Neither prototype calls
`nModifier.MarkForDelete`, subtracts `Immobilized`, or emits any lifecycle
message.

## Native proof

All function names below are decompiler labels from `Game.c`, not claimed
retail symbols.

### Request argument mapping

The Lua binding `sub_A65600` decodes arguments 1-5 as target, initiator, GUID,
optional ability-instance ID, and rank. With more than five arguments,
`sub_A65530` copies arguments 6 onward into modifier float properties. The
binding calls:

```text
sub_9E16C0(GUID, target, initiator, abilityInstance, rank, 0, callback, lua)
```

Thus the teleporter request passes `abilityInstance=0`, `rank=1`, and native
creation parameter 6 as zero.

### Instance-ID allocation

`sub_9E0FB0` calls `sub_9DFAF0`. That function allocates from simulator object
pool component 216, whose object size is 448 bytes and capacity is 2,048
(`sub_9DDF80`). The pool allocator `sub_7B7CC0(pool, 0)` returns a generational
32-bit handle:

```text
instanceID = (generation << 16) | slot
```

The low 16 bits select the pool slot. The high 16 bits come from the pool's
generation counter, which is incremented for allocation and skips a zero low
generation. Freed slots are linked into a free list by `sub_7B7E80` and can be
reused with a later generation. Therefore the exact numeric ID is live-state
dependent: it is neither the target object ID nor a per-teleporter constant,
and assuming a simple monotonically increasing 32-bit counter is incorrect.

`sub_9DFAF0` initializes the allocated 448-byte instance with `sub_9DEC30`,
writes the returned handle at offset `+0`, and returns the selected pool
object. Chunk 349's `GetModifierInstanceID` binding (`sub_A65110`) reads this
same first dword from the current modifier context.

### Exact `ModifierCreated` payload

`sub_9E0FB0` fills the instance and calls `sub_A1FB20` before creating or
activating the Lua thread. `sub_A1FB20` publishes logical message 36 with a
packed, unreflected 37-byte payload. Its layout and teleporter values are:

| Offset | Width | Field | Exact teleporter value / source |
|---:|---:|---|---|
| `0x00` | 4 | target | entering player's object ID, `*(targetAgent+0)` |
| `0x04` | 4 | GUID | `0x502f1932` |
| `0x08` | 4 | instance ID | generational component-216 pool handle above |
| `0x0c` | 4 | duration | `0` ms |
| `0x10` | 4 | overdrive | `1`, copied from request rank / instance `+48` |
| `0x14` | 4 | stack count | `1`, forced by the create call independently of rank |
| `0x18` | 8 | start time | current 64-bit simulator milliseconds from `sub_9D2920`, sampled immediately before publication |
| `0x20` | 4 | source | `0` |
| `0x24` | 1 | bound | `1` |

All multibyte fields are little-endian on the wire.

Duration is zero because `sub_9DE220` calls `nAbility.GetDuration`; a missing or
zero numeric duration returns zero, and chunk 349 authors no duration. The two
`0.5s` waits are coroutine presentation timing, not modifier duration.

The source field is deliberately **not** the teleporter initiator. Native
instance offset `+20` retains the teleporter as initiator for engine behavior,
but the create binding passes native parameter 6 as zero. That becomes instance
offset `+344`; `sub_A1FB20` maps a null `+344` owner to replicated source `0`.

The bound byte is copied from template byte `+90` to instance byte `+44` and
then to the payload. Chunk 349 authors `requiresAgent=true`, so this byte is
`1`. Native agent cleanup (`sub_9DEA80`) tests the same instance byte before
removing agent-bound modifiers.

### Publication and activation order

Within `sub_9E0FB0`, the order is strict:

1. fill instance fields;
2. sample start time;
3. call `sub_A1FB20` (`ModifierCreated`);
4. call `sub_9DDA00` to create the modifier Lua thread;
5. call `sub_9DE4C0` to invoke callback 1 (`Activate`).

Therefore `ModifierCreated` precedes every chunk-349 action, including
teleport-out. Chunk 349 itself proves teleport-out precedes the first wait,
teleport precedes the zero-duration wait, and teleport-in follows that wait.

### Deletion and immobilization ownership

`sub_A1FD10` publishes logical message 38 with exactly eight bytes:
`target` followed by `instanceID`. It is reached from `sub_9DFBA0` only while a
modifier is being removed from the target's modifier list. The full removal
path is:

```text
sub_9E12D0(instanceID)
  -> sub_9E11F0(instance)
     -> sub_9E09D0(instance)       // Deactivate callback and owned resources
     -> sub_9DDD40(instance)       // remove attached attribute contributions
     -> sub_9DFBA0(target, ID)
        -> sub_A1FD10(target, ID)  // ModifierDeleted
```

`sub_9DDD40` walks the modifier's 32 attached attribute slots, removes each
contribution through the target attribute container, and clears the slot.
That is the owner which reverses chunk 349's `Immobilized += 1`; the empty Lua
`Deactivate` body does not do it.

The Lua function `nModifier.MarkForDelete` maps to `sub_A65800`, which calls
`sub_9E12D0`. Other native owners also call the same removal entry point,
including Unique/replacement policy and requires-agent teardown when the bound
agent is destroyed. Chunk 349 calls none of them.

Coroutine completion is separate. `sub_8F0EC0` starts the Lua callback thread;
`sub_8F0FB0` resumes it; and scheduler sweep `sub_8F10C0` removes a finished Lua
thread from scheduler containers. That sweep does not call `sub_9E12D0`,
`sub_9E11F0`, `sub_9DFBA0`, or `sub_A1FD10`. This is native proof that a normal
return from chunk 349 is not modifier teardown.

## Live local proof

There is no permitted retail packet capture in the inspected corpus, so this
section intentionally does not promote local replay into a retail observation.
The following focused executions passed against the exact content.db hashes:

- `TestRetailTeleporterModifierMatchesGoOracle` executed chunk 349 and produced
  the eight semantic steps above;
- `TestRetailBossSecurityTeleporterActivationMatchesGoOracle` executed chunk
  144 and confirmed the GUID, target/initiator roles, rank, entry/stay/exit
  policy, and destination handoff;
- `TestTeleporterRunProducesExactAuthorityAndPacketDeadlines` confirmed
  teleport-out is immediate, then at `+500ms` the packet list is exactly
  `0x90` teleport followed by `0xa5` teleport-in, and at `+1000ms` ordinary
  completion produces zero packets while immobilization remains owned by the
  run;
- `TestModifierLifecycleWireShapes` confirmed build-103 wire `0xa2` uses the
  packed 37-byte field order above and wire `0xa4` uses packed
  `(target, instanceID)`.

These runs are live proof of the repository's content interpreter and packet
encoder. The retail-only claims above remain grounded in the bytecode and
native `Game.c` paths, not inferred from the local Go implementation.

## Proof versus inference ledger

### Directly proved

- chunk identities, hashes, constants, callbacks, and instruction order;
- chunk-144 request arguments and chunk-349 timing/action order;
- generational pool allocation algorithm and component-216 capacity;
- complete 37-byte created payload layout and the field-producing native
  values/formulas;
- created-before-activate ordering;
- eight-byte deleted payload and its removal-only call path;
- no delete or immobilization reversal in chunk 349;
- scheduler completion is not lifecycle deletion;
- attached attributes are removed by native modifier teardown;
- the focused local executions and their packet ordering.

### Inference or live-variable qualification

- The concrete target ID, instance ID, and start timestamp cannot be constants:
  they depend on the entering agent, current pool/free-list generation, and
  current simulator clock. Their exact construction is proved, but no single
  numeric value is valid for all live entries.
- An `0xa4` observed after a successful teleport identifies a later lifecycle
  owner, but packet order alone cannot identify which owner. Owner identity
  requires the surrounding destruction/replacement/explicit-delete event.
- No retail packet capture was available under the permitted evidence scope;
  same-deadline grouping of `0x90` and teleport-in is locally executed proof.
  Retail bytecode/native proof guarantees their order, while transport batching
  across scheduler passes is not asserted beyond that order.
