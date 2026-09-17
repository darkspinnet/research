# Nonstandard hero basics: build-103 runtime and wire shapes

## Scope and evidence boundary

This note covers the build-103 behavior of four basic-attack families that do
not fit darkspin's current melee-or-linear-projectile decoder split:

- cursor-area: `FireTempestBasic`;
- point-blank AoE: `TCShieldedSentinelBasic`;
- toss: `Trapper_Grenade` and `VoodooTempestBasic`;
- custom lob/cloud: `Sprout`.

The conclusions come from the authoritative runtime `content.db`, the exact
Lua 5.1 payloads extracted from it, and the canonical build-103 decompile at
`bin/game/GameBin/Game.c`. No recap material was used. There is no
retained EA-server capture for these five abilities, so application-message
ordering follows authored call order and the recovered build-103 component
senders; RakNet datagram grouping, dirty-component flush placement, object
allocator values, and the relative placement of deferred `ObjectDelete` or HP
reflection remain capture gaps.

The roots and shared bodies are:

| Role | Chunk / resource | Source | Decoded SHA-256 |
| --- | ---: | --- | --- |
| FireTempest | `459` / `14013` | `Abilities/0xD882C2C9.lua` | `674843475c555f036048d345926254105901a797e39aeb079b01468aa96302dd` |
| Shielded Sentinel | `612` / `14176` | `Abilities/0xB28755E4.lua` | `d63ae1f82f5ccf8f7024a5994c528a299addda0ae4a9107c27da08ba8e4e35a2` |
| Trapper grenade | `909` / `14487` | `Abilities/0x141BA061.lua` | `60f6d4032e9df0fbfc50dfcd703010ec0ab814079e839eb015cb1d058c38978d` |
| Voodoo Tempest | `635` / `14200` | `Abilities/0x493651EB.lua` | `6c2d764632bb94d3f9928d6f924ec084b05f42cf9b395d40a0cff6fa1ae3a3a6` |
| Sprout | `982` / `14563` | `Abilities/0x009B7C57.lua` | `1bc76462d8a63c3c39443fd8ded373623a010754957cec771393da9ed25ab587` |
| point-blank template | `324` / `13871` | `Abilities/0x8C550D00.lua` | `554be7a0fa694b0abe3d96ff8258af147e40e85a843d76c7bcb4b776064152f0` |
| toss template | `630` / `14194` | `Abilities/0xCDD518BA.lua` | `c75b68fab1bd56c399894277954544a697d308b121d551b2e31ee5871b86259f` |
| Voodoo weaken | `644` / `14209` | `Modifiers/0x71564167.lua` | `b0dd46a7208c3b651b111cf1c853d7219a444b591937dcde469ed7a4454d7316` |
| DOT template | `149` / `13679` | `Modifiers/0xD6C886B1.lua` | `01b9ed201b2c5bd0e72820d42f76532a1a229ea1ed4a3a134933fe058a84dcaf` |

The extracted payloads are retained under
`bin/game/logs/ability-basic-shapes`. The dependency identities and their
package ordinals are independently recorded in `notes/abilities/basic-gaps.md`.

## Shared action request and admission

Build 103 sends a character basic as logical action message `29`, wire `0x9c`,
action type `7`. The application packet is 85 bytes including the opcode; its
84-byte body is a 40-byte common header followed by this 44-byte ability tail:

```text
targetObjectID u32
cursorPosition XYZ
targetPosition XYZ
abilityIndex u32
rank i32
unknown u32
userData u32
```

The sole client sender transmits it reliable ordered with literal arguments
`1,3,0`. The sender normalizes the target object to its network/runtime ID.
Targetless held basics use target `0`; a captured build-103 held request used
`unknown=1` and copied the cursor into `targetPosition`. That capture proves the
targetless form, not every targeted UI producer's cursor-copy rule.

Generic admission (`sub_9E0660`, creation at `sub_9E1540`) resolves the
definition and rank, validates target state, blocking agent attributes, an
already-running conflicting ability, cooldown, mana, ranked range, the
ability's hit predicate, and blocking modifiers before creating an ability
instance. Out-of-range `shouldPursue` abilities may return pursuit result `3`;
rejection creates no instance and pays no cost. The original server-side action
consumer is absent, so the exact rejection response and retry cadence are not
recoverable.

The two position fields must not be collapsed in the decoder or command model:

| Family | Runtime use of target/cursor data |
| --- | --- |
| FireTempest | Authored `alwaysUseCursorPos=true`: the cursor-derived stored target point is the blast center even when an object was selected. The object ID is consulted only at hit time as a possible direct hostile target. |
| Shielded Sentinel | Authored `targeted=false`: attack position is the caster's current position. Target ID, cursor, and supplied target position do not anchor the damage query. |
| Toss | The template reads both stored target ID and stored target point. At launch it refreshes the point from a still-valid target object; otherwise it keeps the stored point. The lob is not homing after creation. |
| Sprout | The custom tick reads the target ID but launches both spores toward the stored target point. It does not refresh from the target object and neither lob homes. |

For authority, client header actor/position, cursor, target position, index, and
rank remain requests rather than proof. The authenticated actor, equipped
definition/rank, live target, and navigable/range-valid point must be derived or
validated by the server.

## Runtime requirements

### FireTempestBasic: cursor-area without a projectile

Authored values are range `30`, radius `0.5`, cooldown `0.6s`, mana `0`,
`timetohit=0.1s`, `timetorelease=0.4s`, animation
`cast_firetempestbasic`, and hit event
`fire_ignite.ServerEventDef`. It is Basic + Energy, uses Elements/Energy
damage, has `shouldPursue=true`, `noGlobalCooldown=true`, and
`alwaysUseCursorPos=true`.

The tick performs exactly this work:

1. Snapshot agent ID, team, and attacker attributes; play the animation.
2. Wait until `t=0.1s`, then snapshot the stored target point and emit one
   positioned `fire_ignite` event there.
3. Resolve the stored target ID and validate it as hostile at that point. If it
   validates, call `TakeDamage` only for that object.
4. Otherwise query damageable objects within radius `0.5` of that same point,
   validate each candidate as hostile, and call `TakeDamage` for each accepted
   candidate.
5. Wait until `t=0.4s`; the callback then ends.

Damage is weapon damage evaluated at hit time with the activation-time attacker
snapshot. The event precedes every damage call. The authored callback contains
no object creation, projectile wait, trail, impact/miss branch, modifier, or
`PayCooldownAndMana` call. The last absence is bytecode fact; it must not be
silently replaced with toss-template payment timing when reproducing this
runtime.

Application order is therefore:

```text
t=0.000  0xa5 SetAnimationState(caster)
t=0.100  0x9b positioned fire_ignite
         zero or more 0xba CombatEvent in radius-query order
t=0.400  ability callback ends
```

An HP `0x97` dirty reflection may accompany each accepted damage call, but its
placement relative to `0xba` is not fixed by the client-shaped executable.

### TCShieldedSentinelBasic: caster-centered point-blank AoE

Authored values are activation range `2`, radius `4`, cooldown `0.7s`, mana
`0`, `bonusDamageMultiplier=1/3`, animation
`tc_shieldedsentinel_basic`, `timetohit=0.2s`,
`timetorelease=0.8s`, and per-target impact event
`ctd_minn_tc_4_bullet_hit.ServerEventDef`. It is Basic + AoE + Physical,
Technology/Physical damage, `targeted=false`, `shouldPursue=true`, and
`noGlobalCooldown=true`.

The point-blank template snapshots the caster position, plays the animation,
then immediately calls `ReleaseAgent` followed by `PayCooldownAndMana`. It
waits until `t=0.2s` before calling this root's overridden `ApplyAttack`; the
template remains alive until `t=0.8s`. This immediate agent release is an
authored-template distinction and is not the same as the release timestamp.

`ApplyAttack` computes effective radius as `4 * (1 + AoERadius)`, calls
`GetTargetsAroundAttacker`, and does nothing if the returned list is empty.
Otherwise it snapshots weapon min/max damage and lets `N` be the number of
returned targets. Every target receives the range:

```text
weaponDamage * (1/3) + weaponDamage / N
```

The root divides `damageCoefficient` by `N` as well. For each target, it calls
`TakeDamage` first, then emits the impact event on that target with facing
normalized from target to caster. It does not inspect the damage result before
emitting the event.

```text
t=0.000  0xa5 animation
         ReleaseAgent; PayCooldownAndMana (simulation operations)
t=0.200  for each target: 0xba damage, then 0x9b target impact
t=0.800  template callback ends
```

There is no projectile or cursor-centered effect. Query ordering is the order
returned by `GetTargetsAroundAttacker`; retail spatial ordering is not proven.

### Shared toss runtime

The toss template chooses the near animation when target distance is strictly
less than `closeRange`, otherwise the far animation. It plays that animation,
waits to `timetohit`, pays cooldown/mana exactly once, applies the authored
near/far offsets in caster orientation, and creates one projectile object. It
sets team, activation-time attribute snapshot, and orientation, attaches
`projectileFX`, then starts an object-owned `TickParabolicProjectile` thread.
The casting callback independently remains alive until `timetorelease`; the
projectile thread survives that release.

The projectile thread initializes native `WaitForLobbedProjectile`
(`sub_A091C0 -> sub_A2EF00`). Native movement type is `4`. The initializer
records the simulation start time, start/destination/up direction, height,
duration or speed-derived duration, bounce count/restitution, ground-only and
stop-on-creature flags, plane direction/speed, and the linear/quadratic up
coefficients. The reflected launch is `0x94 LocomotionDataUpdate` with sparse
fields `0`, `1`, and `2`: `lobStartTime`, zero
`lobPrevSpeedModifier`, and the exact 84-byte `cLobParams` memory image. The
complete application packet is 105 bytes including opcode and terminator (104
bytes after the opcode).

For each projectile ID `I`, required application order is:

```text
0xa5 animation(caster)
... at timetohit ...
0x8c ObjectCreate(I, authored projectile noun)
0x9b attached projectileFX(I)
0x94 lob fields 0/1/2(I)
... independent cast release and flight ...
damage/modifier/presentation selected below
ObjectDelete(I) on the later object-manager sweep
```

`ObjectCreate` cannot contain the locomotion component, so `0x94` is required;
`0x95` is not a lob initializer. Whether movement type `4` is folded into the
create object fields, the exact create length, the dirty-component flush
boundary, and the later delete placement need a retail capture.

After the lob wait returns, the template calls `ProjectileLanded`, gathers
damageable objects sorted by distance within the ranked radius, validates them
as hostile, and damages up to `maxTargets` (default `1000`). For each accepted
damage result it requests every authored modifier. It then calls the optional
projectile callback, removes the projectile object, and finally emits the
template `impactFX` when one is configured.

#### Trapper_Grenade

Trapper authors range `16`, cooldown `0.8s`, mana `0`, near/far animations
`trapper_active_support` / `trapper_basic`, hit/release `0.2s` / `0.5s`,
close range `6`, far/near heights `0.9` / `0.5`, far/near speeds `20` / `18`,
radius `3`, and weapon-damage multiplier `0.2`. It creates
`Ability_Epic_Fireball.Noun`, attaches
`cyber_trapper_basicGrenadeProjectile.ServerEventDef`, and uses
`cyber_trapper_basicGrenadeExplosion.ServerEventDef` as `impactFX`.

The far offsets are `(0.7,1.7,1.0)` and the close path overrides Z with `0.3`.
It always targets the far-range point, strikes ground only, stops bouncing on
creatures, and authors bounce range `8`, four bounces, restitution `0.4`
(`0.25` close). With `flightTime=0`, native lob duration is derived from
horizontal distance and the selected speed.

Trapper overrides `ProjectileLanded`. Only after the native lob/bounce wait
finishes does it start a `1.5s` grenade timer. Every `0.5s` it queries for any
target within radius `3`; it returns early when one appears, otherwise returns
on the first poll at or after the timer. Damage is therefore not fixed at
cast `t=0.2s` or at first ground contact: it occurs after lob completion plus
`0..about 1.5s`, quantized by the half-second poll (scheduler ticks can add
jitter).

The explosion path is all `TakeDamage` calls first, then object removal, then
the positioned explosion event. Because removal is deferred, the event versus
wire `ObjectDelete` order remains a flush/sweep gap.

#### VoodooTempestBasic

Voodoo authors range `30`, cooldown `0.8s`, mana `0`, near/far animations
`voodootempest_basic_close` / `voodootempest_basic_far`, hit/release
`0.18s` / `0.5s`, close range `8`, far/near height `3` / `0.1`, and far/near
speed `30` / `20`. It creates `Ability_Epic_Fireball.Noun`, attaches
`skullbomb.ServerEventDef`, uses radius `1`, and has no bounce configuration.

The authoritative toss admits an explicitly selected living hostile even before
that NPC has acquired the caster, preventing an ordinary first aimed shot from
being rejected while the client retains its targeting camera. At landing, the
radius-one skull bomb expands contact by each hostile's authored footprint so a
visible impact on a large body is not treated as a center-point miss.
It authors `maxTargets=1`. Offsets are `(0.5,4.5,1.4)` with close Z `-1`.

The 2026-09-04 synchronization review removes the server-only moving landing
point: damage and effects now retain the ground-origin destination sent to
native lob locomotion. Cast release survives an earlier projectile landing.
Muzzle offsets start from the actor-center estimate, and the projectile's
close/far flight operands interpolate by distance as authored by template 630.
The target ground position is refreshed after windup, and the fixed launch
plan supplies both the published lob and its independently scheduled landing.
See [the synchronization review](../combat/movement-sync.md) for evidence and
remaining destination-constraint, collision, and geometry gaps.

Damage is weapon-derived Supernatural/Energy damage. Although the root requires
and thereby registers the Voodoo weaken modifier module, it never assigns the
toss template's `modifiers` field and no callback requests that modifier. This
basic therefore emits no weaken `0xa2 ModifierCreated`; the required module is
a decode/link dependency, not runtime evidence of application. The optional
projectile callback chooses one positioned event: the large
`shadow_skull_lob_explosion_large.ServerEventDef` when the direct landed target
has Curse or Fear, otherwise `skullbomb_explosion.ServerEventDef`. This callback
runs before the template removes the projectile. A curse/fear multiplier of
`1.2` participates in the overridden damage calculation; it is not a second
hit.

```text
landing -> at most one damage (0xba)
        -> one normal/strong positioned 0x9b -> remove projectile
```

### Sprout: two lobs replaced by two cloud objects

Sprout is custom code, not an instance of the toss template despite requiring
that module. It authors range `30`, cooldown `0.7s`, mana `0`,
`shouldPursue=true`, `noGlobalCooldown=true`, and Basic + AoE + DoT + Energy
descriptors. Both animation sequence rows have `hit=0.1s` and `release=0.7s`,
using `lf_shroomtempest_basic_a` then `_b` in sequence.

At `t=0.1s`, the selected animation row creates two separate
`Ability_TerrainOnlyProjectile.Noun` objects, one from each transformed
left/right offset. The left offset is `(1,2,1.5)` and the right mirrors X to
`-1`. Each object gets the attached
`life_shroomtempest_sprout_projectile.ServerEventDef`, team, attribute snapshot,
orientation, and its own `TickProjectile` thread. The cast callback ends at
`t=0.7s`. It contains no `PayCooldownAndMana` call; that absence must be
preserved as recovered behavior rather than borrowing toss timing.

Each projectile thread fixes its destination to the request-time stored target
point. It first derives a nominal duration by clamped linear interpolation from
`0.6s` at distance zero to `1.08s` at distance 30. Across distances `5..8`,
the actual duration interpolates from `0.3s` to that nominal duration and the
height independently interpolates from `0` to the authored height `4`; both
clamp outside that interval. It starts a ground-only, no-bounce native lob. On
completion it:

1. marks the projectile for deletion;
2. emits positioned
   `life_shroomtempest_sprout_detonate.ServerEventDef` at the returned landing
   position;
3. creates a distinct `Ability_Sprout_Shroom.Noun` cloud job object there;
4. copies team and the projectile's attribute snapshot; and
5. starts an object-owned `TickCloud` thread.

Per spore, the semantic order is therefore:

```text
projectile 0x8c -> attached projectile 0x9b -> lob 0x94
... flight ...
mark projectile -> positioned detonate 0x9b -> cloud ObjectCreate 0x8c
-> attached continuous-cloud 0x9b -> trigger-volume lifetime
```

The main callback creates left then right, but object-owned thread scheduling
may interleave the first lob initialization with the second create. Assert the
per-object partial order, not an uncaptured global sequence such as
`create-left, create-right, locomotion-left, locomotion-right`.

`TickCloud` attaches
`life_shroomtempest_sprout_cloud_continuous.ServerEventDef`, creates a trigger
volume with effective radius `2 * (1 + AoERadius)`, waits `4s`, clears its
tracked entrants, destroys the trigger, removes the continuous effect, waits
`0.3s`, and marks the cloud object for deletion. The preloaded `cloud_pulse`
asset is not called by the recovered execution path.

On first hostile entry, the callback requests the `SproutPoison` modifier and
then emits `life_shroomtempest_sprout_cloud_hit.ServerEventDef` on that object.
Overlapping clouds increment a per-target cloud count instead of creating an
independent poison tail for each overlap. Modifier creation therefore precedes
the entry effect (`0xa2` then `0x9b`).

The poison tick deals Life/Energy weapon-derived damage immediately, waits
`0.5s`, and tests whether the target remains inside at least one Sprout cloud.
While inside, its completed-tick counter resets to zero, so damage continues
every `0.5s` without consuming the six-tick tail. Once outside all clouds, it
performs six half-second ticks before ending. Cloud expiry drives the same exit
state. Damage is thus neither a projectile impact hit nor six ticks measured
from cast time.

## Decoder requirements

These are compile/decode obligations only. They do not authorize reusing the
generic projectile simulator or inventing runtime defaults.

### Shape dispatch and common data

- Stop selecting melee only from string `hitEffect` and treating every other
  table as a generic projectile. Preserve an explicit cursor-area,
  point-blank, toss, and Sprout/custom-lob shape.
- Retain the 44-byte action tail's target ID, cursor position, target position,
  index, rank, unknown, and user data as separate fields. Shape-specific
  runtime decides which positions matter.
- Preserve ranked scalar/table values and authored absence. A missing
  projectile `distance`, generic `speed`, `trail`, or `animationstate` is not a
  decoder error for these shapes.
- Keep callback identity/overrides (`ApplyAttack`, `ProjectileLanded`,
  `CalculateDamage`, `OptionalProjectileCallback`, `TickProjectile`, and
  `TickCloud`) when the runtime projection needs behavior that data fields
  alone cannot express.

### FireTempest projection

Decode `alwaysUseCursorPos`, `range`, `radius`, `animationstate`, `timetohit`,
`timetorelease`, `hitEvent`, descriptors, damage type/source/coefficient,
cooldown/mana, pursuit/global-cooldown flags, and weapon-damage access. Do not
require or synthesize projectile noun, distance, speed, trail, impact, or miss
fields.

### Point-blank projection

Resolve `Abilities!template_ability_pointblankaoe.lua` to exact chunk `324`.
Decode `targeted=false`, `radius`, activation `range`, animation, hit/release
timings, impact event, bonus multiplier, descriptors/damage fields, and the
root's `ApplyAttack` override. Do not project activation range `2` as a
projectile distance or damage radius `4` as a cursor radius.

### Toss projection

Resolve `Abilities!template_ability_toss.lua` to exact chunk `630`. The toss
schema must support near/far animations, `closeRange`, hit/release timing,
height/close-height, flight-time and speed/close-speed alternatives, projectile
noun and FX, impact FX, radius, normal/close offsets, max targets, far-target,
ground-only and stop-on-creature flags, bounce range/count/restitution,
modifiers, and the three toss callbacks.

For Trapper, preserve the root's post-landing timer/poll callback and damage
multiplier. For Voodoo, also resolve weaken chunk `644` plus its global
bootstrap dependencies because the root executes that `require`, and preserve
`maxTargets=1`, the curse/fear multiplier, normal/strong impact fields, and the
optional callback. Do not synthesize a modifier list merely because the
required module registers one. Neither root has generic `animationstate`;
requiring it loses the near/far selection.

### Sprout projection

Resolve the typed global bootstrap (chunk `659`), toss identity `630`, DOT
template `149`, and the Sprout root. Decode the two-row animation sequence with
independent `hit`, `release`, and `animationstate`; left/right offsets; range,
height, flight-time curve inputs; projectile and cloud-job nouns; projectile,
detonate, continuous-cloud, entry-hit, poison-status, and preloaded pulse
assets; cloud radius/duration; DOT period/count; poison modifier metadata; and
the custom tick/trigger callbacks.

Do not flatten Sprout into one toss projectile, one impact AoE, or a generic
projectile with fabricated `distance`. Its minimum runtime projection needs
two projectile identities, two replacement cloud identities, per-target
overlap counts, and the reset-while-inside poison-tail rule.

## Wire checklist and remaining capture gaps

- Input is reliable-ordered `0x9c` type `7`, 85 bytes including opcode, with
  cursor and target position retained separately in its 84-byte body.
- Every animation uses `0xa5 SetAnimationState`: 26 bytes including opcode,
  with a 25-byte body containing object, state, timestamp, overlay flag, scale,
  and the zero source/echo-gate field.
- `nGameObject.AddEffect` uses the 16-byte attached `0x9b` shape (fields `1`,
  `4`, `6`, and `7`); hard removal uses its 11-byte fields-`1/2/7` shape.
- FireTempest, Trapper impact, Voodoo impact, and Sprout detonation use the
  20-byte positioned `0x9b` shape (fields `6` and `10`). Shielded Sentinel's
  per-target impact uses fields `6`, `7`, and `11` (25 bytes). Sprout cloud
  entry uses the 12-byte target effect shape (fields `6` and `7`).
- Toss/Sprout projectile and Sprout cloud identities require independent
  `0x8c ObjectCreate`; object IDs come from the global allocator and must not
  be derived from one another.
- Every lob requires reliable `0x94` fields `0/1/2`, 105 bytes; never replace
  it with `0x95`.
- `TakeDamage` emits the 20-byte native sparse `0xba CombatEvent`; accepted
  requested modifiers emit the 38-byte `0xa2 ModifierCreated` application
  packet before later Lua presentation calls. Of these basics, that ordering is
  exercised by Sprout cloud entry, not Voodoo's unused weaken dependency.
- Object removal is semantic at the stated point, but actual `ObjectDelete`
  publication occurs on the object-manager sweep.
- Retail packet batching, RakNet reliability for application messages other
  than the proven input and reliable locomotion families, exact dirty
  reflection placement, allocator values, cross-thread interleaving, and the
  delete/HP-delta flush positions remain uncaptured and must not be fixture
  guessed.
