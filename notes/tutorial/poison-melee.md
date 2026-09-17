# Tutorial Poison melee wire

## Result

`TutorialPoisonMelee` does **not** attach
`life_common_melee_hit.ServerEventDef`, and it does **not** send the
fields-`6/10/11` positioned-effect recipe. The shared melee template sends a
one-shot, object-bound `nEvent.Notify` after accepted damage.

For the ordinary primary hit, the authored Lua table is equivalent to:

```lua
local facingX, facingY, facingZ =
    nGameObject.GetObjectDirection(attackerId, targetId)
nEvent.Notify({
    asset = life_common_melee_hit,
    objectId = targetId,
    facing = { facingX, facingY, facingZ },
    attackerId = attackerId,
    bCritical = isCritical,
})
```

The table contains no effect-slot index, `forceAttach`, removal flag, or
position. The target object is the hostile object that survived the hit-time
validation/arc selection and for which `TakeDamage` returned success. The
attacker is `nAbility.GetAgentID()` for this ability execution.

## Evidence labels

- **Build-103 content-proven:** exact Lua constants, calls, arguments, branches,
  and ordering in the packaged Lua 5.1 bytecode.
- **Build-103 client-native-proven:** reflected `ServerEvent` field numbers and
  the client packet family/decoder.
- **Native comparison:** the available later native body establishes the
  arithmetic behind a Lua native whose build-103 server implementation is not
  present in the build-103 client executable.
- **Inference/open:** a consequence not fixed by those artifacts. It must not be
  promoted to a retail packet golden without a server sender or capture.

## Content identity

The build-103 runtime `content.db` provides both relevant chunks:

| Role | Indexed source | Resource / package ordinal | Decoded SHA-256 |
| --- | --- | ---: | --- |
| Ability definition | `Abilities/0x2B8B0FB2.lua` | `14476` / `960` | `a4d5f23fd349091f1e1cb1deefec4d2dde913cf3b8c3735d36ac156a03a81033` |
| Shared melee template | `Abilities/0x7BF2D7DD.lua` | `14409` / `893` | `2951fef16876a72687c235cc93377ede9dcec31286a8237a9c6a8af978da778a` |

The ability chunk creates an `nAbility_Melee_Template`, sets cooldown `2`,
range `0.75`, hit time `0.43`, release time `2`, rank-one damage `1..3`, and
`hitEffect = PreloadAsset("life_common_melee_hit.ServerEventDef", ...)`. Its
`tick` and `deactivate` closures only delegate to the shared template. Thus the
effect replication recipe comes from the template, not the ability-definition
chunk.

## Exact event fields and identities

In the primary-target branch of the template, `TakeDamage(...)` occurs first.
Its three results are retained as accepted-damage state, applied damage, and
critical state. Only a successful result with applied damage greater than zero
reaches the hit-effect branch. The template then executes:

1. `GetObjectDirection(agentId, hitTargetId)`;
2. construct `asset`, `objectId`, `facing`, `attackerId`, and `bCritical`;
3. `nEvent.Notify(event)`.

Build-103 `ServerEvent` reflection maps those members as follows:

| Field | Lua member | Exact value/source |
| ---: | --- | --- |
| `5` | `bCritical` | Third result returned by the accepted `TakeDamage` call. False is a default and is omitted by the sparse encoder. |
| `6` | `asset` | `SPID("life_common_melee_hit.ServerEventDef") = 0xe571cf70`. |
| `7` | `objectId` | The actual hit target, not the attacker and not an effect helper object. |
| `9` | `attackerId` | `nAbility.GetAgentID()`, the Poison creature performing the melee attack. |
| `11` | `facing` | The three results of `GetObjectDirection(attackerId, targetId)`. |

There is no field `10` position. There are also no attached-effect fields `1`,
`2`, `3`, or `4`.

**Native comparison, consistent with the bytecode call contract:**
`GetObjectDirection(a, b)` subtracts the live position of `a` from the live
position of `b` and normalizes the result. Therefore this facing is the
attacker-to-target direction at effect time. It is not the target object's own
orientation/facing, and it is not a previously snapshotted target position.
The exact arguments and three-result use are build-103 content-proven; the
subtraction/normalization body is version-comparison evidence because the
build-103 client does not contain the retail server-native implementation.

For an ordinary noncritical, non-coincident hit, the complete application
message is 30 bytes:

```text
9b
06 70 cf 71 e5
07 TT TT TT TT
09 AA AA AA AA
0b FX FX FX FX FY FY FY FY FZ FZ FZ FZ
ff
```

`TT` is the target ID, `AA` is the attacking Poison object's ID, and the facing
components are little-endian `float32`. A critical hit inserts `05 01` before
field `6`, making 32 bytes. A zero/default facing would be subject to sparse
default omission; ordinary separated melee actors produce a normalized,
nonzero vector.

This differs from both nearby recipes:

- attached effect: fields `1/4/6/7`, a persistent per-object slot requiring
  later removal or object teardown;
- positioned effect: fields `6/10/11`, with no object identity;
- Poison melee: fields `6/7/9/11`, plus conditional field `5`.

## Damage, HP, and packet ordering

The exact build-103 content order at the `0.43s` hit continuation is:

1. revalidate/select the hostile target and pay cooldown;
2. call `OnHitTarget(target)`;
3. call `nGameObject.TakeDamage(...)`;
4. if damage was accepted and the applied amount is positive, send the
   object-bound hit `ServerEvent`;
5. call `OnDamageTarget(appliedDamage, isCritical, target)`;
6. continue optional additional-target/self-modifier/post-hit work, wait until
   the independent release deadline, and return.

The authoritative HP mutation is therefore complete before the melee-hit event
is constructed. The general damage native emits the ordinary sparse
`CombatEvent` (`0xba`, mask `0x9b`) during `TakeDamage`, so the two direct
notifications are ordered:

```text
0xba CombatEvent (damage)
0x9b ServerEvent (life_common_melee_hit on target)
```

The effect-first ordering currently modeled by the Go melee simulator is not
the authored order.

**Open/inference for the HP component packet:** build 103 proves the receiving
`CombatantDataUpdate` family (`0x97`) and that an HP-only mutation is represented
by field mask `0x01`, but the retail-server dirty-component flush relative to
the two direct notifications is not present in the client executable and no
retail capture closes it. Semantically the HP value is already changed before
the effect. A likely same-tick flush is `0xba -> 0x9b -> 0x97`, but that final
`0x97` placement is inference and must not be asserted as a byte-exact golden.
What is proven is that `0x9b` cannot precede the `TakeDamage` call or its
authoritative HP change.

## Cleanup and cancellation

There is no hit-effect cleanup operation. This notification allocates no one
of the sixteen attached-effect slots, returns no effect handle, and has no
matching `RemoveEffect`/`RemoveEffectIndex` packet. The template's
`deactivate` function is empty, and the ability's `deactivate` merely delegates
to it.

Consequences:

- cancellation before the hit continuation produces no melee-hit event;
- once sent, cancellation/deactivation does not retract or stop it;
- target death or later `ObjectDelete` needs no effect-slot cleanup for this
  event;
- the client-owned lifetime of the one-shot `ServerEventDef` presentation is
  the only presentation cleanup.

The shared template has a separate additional-target branch whose event omits
`attackerId` and `bCritical`, but `TutorialPoisonMelee` does not override the
template's zero additional-target default. That branch is not part of this
ability's ordinary wire contract.
