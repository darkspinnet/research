# Build-103 interactive object contracts

## Rule

Byte-valid packets are necessary but insufficient. Build 103 may begin a local
object-owned action before sending its authoritative request. A missing noun
component, local ability dependency, completion callback, or teardown path can
therefore lock input without emitting any network packet.

The 2026-07-22 22:59 playtest proved this failure mode: clicking a dropped item
sent no type-9 or type-11 request, `/reset` found the known action descriptors
already inactive, abort removed the server game, and the same client process
failed to open the next RakNet gameplay session.

## Lifecycle matrix

| Object | Create/components | Selection/request | Authority response | Completion/delete | Scene cleanup | Default status |
| --- | --- | --- | --- | --- | --- | --- |
| Dropped health/mana orb | Generic object and lob are receiver-proven; noun owns its overlap trigger | Server-owned contact; no client action request is required | Resource reflection plus pickup presentation | Resource commit and object delete are implemented | Session maps and producers are discarded | Enabled; human-verified collection |
| DNA | Generic object plus DNA loot data are implemented | Server-owned contact | Durable DNA update and targeted pickup event | Commit precedes delete | Session state is discarded | Enabled; collection is human-verified, presentation needs retest |
| Equipment container | Generic object, one-use `cInteractableData`, full `cLootData`, event, and lob are published in receiver-valid order | Packaged `PickUpLoot` sends type 11 | Immediate bounded `0xa8` acceptance; matching rejection on admission or scheduler failure | Capacity-safe persistence precedes delete; failures retain the object | Pending producer is cancelled and generation-guarded; same-address restart is tested | Enabled; human collection/restart retest remains |
| Mission crystal/catalyst | Generic object, sparse crystal `cLootData`, and lob are published in receiver-valid order | Packaged `PickUpCrystal` sends type 9 and the server normalizes it without losing the sync stamp | Immediate bounded `0xa8` acceptance; matching rejection on admission or scheduler failure | First free mission slot commits before delete/acquired publication; full slots retain the object | Run and producer are cancelled and generation-guarded; same-address restart is tested | Enabled; human collection/restart retest remains |
| Loot obelisk | Generic object plus authored interactable data are implemented | Type 11 interaction and pursuit are implemented | Explicit terminal accept/reject exists | Script timeline, drops, use count, collision, and cleanup exist | Session-owned producer cancellation exists | Enabled |
| Health obelisk | Generic object plus authored interactable data are implemented | Type 11 interaction and pursuit are implemented | Explicit terminal accept/reject exists | Orb output and object completion are implemented | Session-owned producer cancellation exists | Enabled |

## Implemented lifecycle proof

The server test suite now exercises both live handler branches:

1. Equipment uses type 11 and crystals use type 9 with the client sync byte
   preserved through command normalization.
2. A scheduler failure produces a matching type-2 `0xa8` rejection and leaves
   the pickup available for retry.
3. A successful admission produces a type-1 `0xa8` with `PickUpLoot` bounded
   from cast start through 400 ms and `PickUpCrystal` through 1000 ms.
4. Equipment persists before deletion; crystals assign the first free mission
   slot before delete and `0xc3` acquired publication.
5. Session teardown cancels interactable, pursuit, equipment, and crystal
   schedules. A producer retained by the scheduler is generation-guarded and
   emits nothing after disconnect.
6. The same address can send a new hello/status sequence and receive a fresh
   dungeon setup without inheriting the pending pickup.

The remaining evidence gap is authoritative-sender ordering: the absent retail
server prevents proving whether its create/component packets used the same
receiver-valid order. That boundary is documented rather than represented as
an unknown field or a disabled feature. Human verification still owns visible
collection, abort, and second-mission behavior.
