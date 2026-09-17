# Campaign DNA pickup contract

## Scope and result

This note recovers the build-103 client/content half of an ordinary campaign
DNA pickup. It does not claim the absent retail server's enemy-death caller,
currency transaction, or sender packet trace.

The strongest corrections to the current footage fallback are:

- DNA is gated below difficulty `1` and then uses a global float threshold of
  `0.25`; it is not natively shown as an independent 30-in-100 roll.
- The amount is an integer-valued, level-scaled formula written as a `float` to
  `cLootData.mDNAAmount`; it is not a uniform integer draw from `3..20`.
- DNA collection is an authored trigger-to-homing-projectile flow. The native
  `2.5` constant is lob height, not a DNA contact radius, and no 30-second DNA
  lifetime was recovered.
- Spawn-side receiver dependencies require object creation before the separate
  loot-data and lob updates. Exact retail reliability, batching, and flush
  order are still missing.

Confidence is **Exact** for direct constants, branches, reflected fields, and
Lua bytecode; **High** where the decompiler loses a register role but adjacent
calls/components close the interpretation; and **None** for policies assigned
only by the missing retail server.

## Evidence identities

- Canonical executable/decompiler: `bin/game/GameBin/Game.c` and
  `Game.idb`.
- `DNA.Noun` is asset `0xf30084f6`. Startup code at `0x00f53ca0` resolves that
  literal into the handle consumed by `sub_9CABF0`.
- Startup code at `0x00f53c30` binds literal `DNA_Pickup` to `sub_9C98B0` via
  `sub_A173A0`. The adjacent `Orb_Pickup` binding targets `sub_9C98A0`.
- Runtime `ServerData.package` ordinal `275` is resource `13791`, group/type
  `0x3681d755`, instance `0xa98a84b4`. `content.db` indexes it as
  `lua/0xA98A84B4.lua`; its decoded 1,978-byte Lua 5.1 chunk has SHA-256
  `9acb509013188489d4b27eccf18d503d72e5b52dd484fb0a094ad89cfaa5c094`.
  It defines `nDNA_Follow`.
- Build-103 `LabsTuning/0x3B01D7F6.prop` is resource `13516`; its decoded
  4,298-byte payload has SHA-256
  `b57ca3826416ecc6f23db0ad92b9d6bdce2faa376809a5505bb2869a0014bcce`.

## Native drop selection and ordering

`sub_9CE140(source, contextObjectID, selector, mode)` processes categories in
this exact code order:

1. `selector & 0x02`: orbs;
2. apply the source predicate's `+30` amount adjustment when
   `sub_A17630(source)` succeeds;
3. `selector & 0x04`: crystals/catalysts;
4. `selector & 0x10`: DNA;
5. `selector & 0x08`: equipment.

This is also the native random-call ordering when several selector bits are
enabled. An omitted Lua selector becomes `-1`, so orb/crystal draws can consume
the simulator RNG before DNA and equipment consumes it after DNA. The exact
owner/seed/replay policy for the retail server's stream is not recovered.

The source amount passed to DNA is selected before those branches:

- when the source's primary definition pointer at object `+0x80` exists,
  amount is definition `+0x24` and source class is definition `+0x44`;
- otherwise, when object `+0x2f0` exists, amount is that record's `+0x10` and
  source class is forced to `4`;
- otherwise normal mode returns without drops; the alternate mode begins with
  amount `1000`.

The `+30` adjustment occurs before DNA. These offsets and the adjustment are
native facts. Their authored field names and, crucially, the concrete amount
on an ordinary 1-1 enemy are not recovered. The absent server also owns the
ordinary-death call and which object is supplied as `contextObjectID`.

## Difficulty gate, chance, and amount

Let:

- `d` be the simulator difficulty from
  `sub_9BCC50(sub_9BCBE0())`;
- `S` be the selected source amount after the possible `+30` adjustment;
- `M` be context-object attribute `DNADropped` (attribute index `66`), or zero
  when the context/attribute component is absent;
- `U` be the uniform float drawn from the authored interval `[0.5, 1.5]`;
- `q = ceil(d / 4)` and `r = ((d - 1) mod 4) + 1` for positive `d`;
- `e = 10*q + r`.

`sub_9CABF0` first rejects `d < 1`. The property that could override this
minimum (`0x7c5e2bbd`) is absent from the shipped LabsTuning payload, so the
executable fallback `1` is effective.

It next rejects when `randomFloat() > 0.25`, equivalently accepting the native
comparison when the draw is `<= 0.25`. LabsTuning property `0x95189906` is
present as float `0.25`, matching the executable fallback. This establishes a
threshold operand and comparison, not the retail server's PRNG endpoint or an
independent integer percentage abstraction.

The level growth factor is:

```text
G(e) = 1.03^e                                      when e <= 70
     = 1.03^70 * 1.02^(e - 70)                    when 70 < e <= 130
     = 1.03^70 * 1.02^60 * 1.01^(e - 130)         when e > 130
```

LabsTuning property `0x21903fa4` supplies `1.03`; the `1.02` and `1.01` bases
are executable constants. Property `0x29259e01` supplies `0.1`. Property
`0xfd06c1af` supplies the `[0.5, 1.5]` random interval. All match the executable
fallbacks.

The amount written into the pickup is exactly the scalar result of:

```text
amount = max(1, trunc((S / 0.25) * 0.1 * G(e) * U * (M + 1)))
       = max(1, trunc(0.4 * S * G(e) * U * (M + 1)))
```

The truncation is a signed integer conversion before the value is converted
back to `float`. The cached player count is passed as `sub_9CABF0`'s third
argument but that function never reads it: one DNA selector evaluation makes
at most one DNA object, unlike player-count loops in adjacent categories.

## Noun, component, placement, and lob

On an accepted chance, `sub_9CABF0` resolves `DNA.Noun`, creates one object at
the source position through `sub_9D6A30`, requires its `cLootData` component at
object `+744`, and writes the computed amount to component `+0x40`.

Reflection proves `cLootData.mDNAAmount` is field `9`, type `float`, at component
offset `+0x40`. Client wire `0x9a` (`kGmsLootDataUpdate`) reflection-decodes that
field only after resolving the target object and its component. `ObjectCreate`
does not carry this component image.

After `sub_9CABF0` returns, `sub_9CE140` consumes the next source-centered,
navmesh-sampled destination and calls `sub_A2EF00` with height `2.5` and
duration `0.5s`. The DNA branch emits no explicit drop `ServerEvent` between
creation and lob initialization. This gives the in-process order:

```text
create DNA.Noun -> write mDNAAmount -> sample destination -> initialize lob
```

It does not prove network flush order.

## Pickup/contact trigger

`DNA_Pickup -> sub_9C98B0` is a role gate: it returns true only when
`sub_9BCF80() != 1`. The client-role body contains no DNA grant, player-field
mutation, event send, persistence, or deletion. As with the adjacent orb stub,
the authoritative implementation belonged to the absent server role.

The shipped `nDNA_Follow` bytecode supplies the concrete trigger behavior:

1. `main(trigger, entrant)` gets the trigger owner (the DNA object), then
   requires the entrant to be a valid, alive, player-controlled object.
2. It starts an object-owned `triggerThread(DNA, entrant)`.
3. After one coroutine yield, the thread constructs a homing projectile state
   for the DNA object: initial speed `1`, range `1000`, acceleration `20`, turn
   acceleration `20`, zero spin/eccentricity/homing delay, homing flag set, and
   ground collision ignored.
4. `WaitForProjectile` targets the entrant. Its validator retains only a valid,
   alive, non-stealthed player-controlled target. Otherwise it searches objects
   sorted by distance within radius `10` of the DNA object and returns the first
   non-stealthed player-controlled object, or no target.
5. After projectile completion/contact, the thread waits `0.03s` and calls
   `nGameObject.DNAPickup(DNA, target)`.

The radius `10` is a retarget-search radius, not the initial noun-trigger
footprint and not a direct collection sphere. Neither this chunk nor the
traced native branch supplies a DNA lifetime. Therefore the current 2.5-unit
walk-over and 30-second lifetime are compatibility policies, not build-103 DNA
facts.

## Player DNA and presentation

The top-level reflected `LabsPlayer` field is `mDNA`, field index `12`, scalar
offset `+0x1248`. Wire `0xa1` (`kGmsLabsPlayerUpdate`) reflection-decodes it; a
minimal sparse body is:

```text
<slot:u8> 00 10 0c <mDNA:u32le> ff
```

The executable also resolves `dna_pickup.ServerEventDef` (`0xa2023d17`), so the
client has a pickup presentation asset. No recovered authoritative body proves
whether retail emitted that event, its target/fields, or its position relative
to the player update and object deletion. The client-role `DNAPickup` stub does
not answer those questions.

## Publication and transaction boundary

### Native/receiver facts

- A receiver must have the DNA object and its `cLootData` component before a
  field-9 `0x9a` update can decode.
- Lob state refers to the already-created object.
- The native simulator constructs the object, writes its amount, and only then
  initializes lob state.
- Collection has an exact player-total replication field (`LabsPlayer.mDNA`,
  field `12`) and an available pickup event asset, but no recovered sender.

### Safe publication ordering, not a retail trace

Until an original server capture is recovered, publish a spawned DNA pickup as
one reliable-ordered dependency chain:

```text
ObjectCreate -> cLootData field 9 (mDNAAmount) -> lob locomotion
```

On collection, first complete one idempotent authoritative currency mutation.
Only after commit should a reliable-ordered batch publish the new player field
`12`; if the pickup event is used, publish it after the state update and before
the terminal `ObjectDelete`, with deletion last. This is a conservative
state-before-presentation dependency policy, not evidence of retail packet
order. A persistence or encoding failure must retain the pickup and permit a
retry without a second grant.

## 1-1 walkthrough cross-check

The referenced 9:32 [Game 1-1 walkthrough](https://www.youtube.com/watch?v=wZK5bU45Tmw)
visibly shows ordinary DNA pickups. One unambiguous collection frame at roughly
`3:15` displays `DNA +6`. The player-authored footage sheet records an observed
`3..20` envelope and 30% annotation across its covered campaign rows.

`+6` lies inside that observed envelope and is compatible with the native
formula for multiple possible source amounts and random factors. It does not
identify `S`, `U`, `M`, difficulty, the global threshold, or an inclusive
minimum/maximum table. The video's cuts also make the displayed timestamp
unsuitable for contact, lob, deletion, or packet-timing claims.

## Missing retail server authority

The client/content evidence does not recover:

- the ordinary enemy death caller, its selector/context object, or the 1-1
  source amount `S` stored in the missing/unnamed authoritative definition;
- whether ordinary enemies use another server-side eligibility/chance layer in
  addition to `sub_9CABF0`'s native 0.25 gate;
- the retail PRNG seed, stream partitioning, endpoint behavior, replay rules,
  or multiplayer synchronization;
- the initial `DNA.Noun` trigger footprint, natural lifetime/expiry policy, and
  late-join reconstruction;
- which player receives a retargeted/shared pickup under multiplayer races;
- whether collected DNA is committed immediately to account currency, kept in
  a match/chain ledger until cashout, or reconciled across both;
- the atomic grant/idempotency/persistence boundary and failure behavior;
- whether/how `dna_pickup.ServerEventDef` was sent, followed by exact
  player-update/event/delete order, reliability, channel, batching, and
  acknowledgement behavior.

These gaps must remain visibly separate from the native facts above. In
particular, the footage-only 30% / uniform `3..20`, 2.5-unit contact, and
30-second lifetime must not be promoted to recovered build-103 policy.

## Reproduction

```text
darkrun db content_source_resource get content_source_package_id=3 ordinal=275 --config bin/darkspinner/darkspin.toml
darkrun db lua_chunk get server_data_resource_id=13791 --config bin/darkspinner/darkspin.toml
darkrun db server_data bget decoded_payload where content_source_resource_id=13791 --decode zlib --output bin/game/logs/campaign-dna-follow-build103.luac --config bin/darkspinner/darkspin.toml
darkrun lua bin/game/logs/campaign-dna-follow-build103.luac
darkrun db server_data bget decoded_payload where content_source_resource_id=13516 --decode zlib --output bin/game/logs/campaign-dna-labs-tuning.prop --config bin/darkspinner/darkspin.toml
```
