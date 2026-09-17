# Build-103 combat damage RNG

## Result

Build 103 owns its gameplay random stream on `SP_Simulator/cGameSimulator`.
It is not owned by a network session, actor, or ability. A 624-word MT19937
state is embedded in each simulator at byte offset `210060`, seeded once by the
simulator constructor, and shared by all simulator consumers. Consequently,
damage draws are ordered with every other draw made by that simulator. Actor
and ability activation order does not create independent substreams.

The retained client proves the stream, seed site, integer and floating-point
primitives, authored Poison ranges, and the Lua-to-native damage boundary. It
does **not** retain the authoritative damage sampler: the function called by
`nGameObject.TakeDamage` is a literal success stub in this client executable,
and the repository contains no combat trace. Thus the exact retail choice of
integer versus floating-point sampling inside `TakeDamage` is not directly
recoverable from the allowed corpus.

The implementation contract below makes that evidence boundary explicit. For
ranked damage ranges whose endpoints are integral, including Poison melee
`[1,3]` and Poison Cloud `[1,4]`, it uses one multiply-high bounded-integer draw
over `max-min+1`. That is the only recovered primitive that makes both authored
endpoints exactly reachable without bias. It is an implementation-ready parity
decision and a strong inference, not a claim that the missing server routine
was present in `Game.c`.

| Question | Finding | Confidence |
| --- | --- | --- |
| Owner | `cGameSimulator` | Direct native evidence |
| Lifetime | Simulator construction through destruction; no actor/ability reset | Direct native evidence |
| Default seed | Low 32 bits of `RDTSC`, selected by constructor argument `-1` | Direct native evidence |
| Integer primitive | `floor(limit * uint32 / 2^32)`, result `[0,limit)` | Direct native evidence |
| Float primitive | signed-`int32` transform to `[0,1)`, then affine range mapping | Direct native evidence |
| Poison endpoints | melee `[1,3]`; cloud `[1,4]`, repeated for all three authored ranks | Direct content evidence |
| Retail damage sampler | Missing client-side/server-authoritative implementation | Proven absence in retained boundary |
| Go rank-damage policy | inclusive integral selection with `limit=max-min+1` | Required parity inference |

## Evidence corpus

Only repository artifacts were used. No web or build-127 reference was used.

| Artifact | SHA-256 |
| --- | --- |
| `bin/darkspinner/darkspin/cache/content.db` | `a7841ae39546cf0ebbd10d7ee9d6fe3b7288dc6f653a937735353e5c3e0d82e6` |
| `bin/darkspinner/Data/ServerData.package` | `845ef186dcc752071fff8fa43385a0bcd88e14ff8230001bc26b75f44a30892f` |
| `bin/darkspinner/GameBin/Game.c` | `1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aeded662b1eb` |

Both `bin/darkspinner/darkspin/logs/traces` and the conventionally reserved
`bin/server/darkspin/logs/traces` contained zero files during this recovery.
There is therefore no sampled-damage oracle in the allowed traces.

Generated inventories and decoded chunks used during inspection are under
`bin/game/logs/combat-rng`, as required by the workspace convention.

## Authored Poison ranges and call path

`content.db` identifies the two concrete ability chunks:

| Ability | chunk ID | server-data resource | source identity | bytecode SHA-256 |
| --- | ---: | ---: | --- | --- |
| `TutorialPoisonMelee` | 898 | 14476 | `Abilities/0x2B8B0FB2.lua` | `a4d5f23fd349091f1e1cb1deefec4d2dde913cf3b8c3735d36ac156a03a81033` |
| `TutorialPoisonCloud` | 581 | 14143 | `Abilities/0x66D97AC9.lua` | `076202c42dec21692943d05402a1d42ef47a78ec224ba056fd947ad9e126eee3` |

The melee bytecode creates `damage = {{1,3},{1,3},{1,3}}`. The projectile
bytecode creates `damage = {{1,4},{1,4},{1,4}}`. These are two-element range
tables, not preselected values.

The corresponding templates are:

| Template | chunk ID | server-data resource | source identity | bytecode SHA-256 |
| --- | ---: | ---: | --- | --- |
| melee | 832 | 14409 | `Abilities/0x7BF2D7DD.lua` | `2951fef16876a72687c235cc93377ede9dcec31286a8237a9c6a8af978da778a` |
| projectile | 38 | 13559 | `Abilities/0xCE0FC9AA.lua` | `0cf624bee02973c7f6986f493f2120a5f1264987a105cee689dc844c2e5bb6c7` |

Melee `GetDamage` and projectile `StandardDamage` copy the selected rank's
two-element table and apply pet scaling to both endpoints. They do not call
`math.random` for damage. Their hit paths pass the table to
`nGameObject.TakeDamage`. The `math.random` calls near those hit paths are for
modifier chances after damage, so they must not be mistaken for range
selection.

The indexed `lua_string_constant` table contains no `randomseed` row. A scan of
all 1,029 decoded Lua chunks likewise found no `randomseed` occurrence. Retail
content therefore does not reseed the simulator stream per script, actor, or
ability.

## Simulator ownership, seed, and state lifetime

The native evidence in `Game.c` is:

- `sub_9C21D0` allocates `0x39EA8` bytes with the tag
  `SP_Simulator/cGameSimulator` and calls constructor `sub_9C1CA0`.
- `sub_9C1CA0` calls `sub_AE54D0((unsigned int *)simulator + 52515, -1)`.
  `52515 * 4 = 210060`, locating the RNG state inside the simulator object.
- `sub_AE54D0` initializes the cursor and calls `sub_AE53C0`. Seed `-1`
  selects the low 32 bits of `RDTSC`; any other argument is the explicit seed.
- `sub_9BCE70` and `sub_9BCEC0` take a simulator pointer and access its state at
  `+210060`. `sub_9BCE90`, `sub_9BCEF0`, and `sub_9BCF40` obtain the current
  simulator through TLS and access the same offset.
- `sub_9C1E50` is the simulator destructor. There is no reseed in the retained
  constructor/destructor interval.

TLS is only the route to the current simulator. It does not make the RNG
thread-owned: the state bytes live in `cGameSimulator`. Nor is a gameplay
network session the owner merely because a particular deployment may create
one simulator per game. The ownership boundary visible in native layout is the
simulator object.

The default `RDTSC` seed means separate retail simulator constructions are not
reproducible without capturing the seed. The stream is deterministic after
that seed and call order are fixed. A Go server should accept and trace an
explicit 32-bit seed; it should not hide wall-clock or hardware timing inside
the RNG constructor.

## Exact native primitives

### Raw generator

`sub_AE53C0`, `sub_AE52E0`, and `sub_AE5420` implement an MT19937 state/twist
and standard tempering, with a nonstandard seed expansion:

```text
x = seed | 1
repeat 624 times:
    y        = uint32(69069 * x)
    state[i] = (x & 0xffff0000) | (y >> 16)
    x        = uint32(69069 * y + 69070)
twist(state)
```

`sub_AE5420` returns one tempered `uint32` and advances the single simulator
cursor. The state representation also contains a pointer into the 624 words
and a remaining-word count. Go need not reproduce the pointer, but its index
must have identical advancement semantics.

### Bounded integer

`sub_AE5480(state, limit)` is exactly:

```text
floor(uint64(limit) * uint64(nextUint32()) / 2^32)
```

For positive `limit` its result is `[0,limit)`. Multiply-high avoids modulo
bias. It consumes exactly one raw draw, including when `limit == 1`.

The simulator installs `sub_9FEE40` as `math.random`, replacing Lua 5.1's
stock function for gameplay scripts:

```text
math.random()       -> Float01()
math.random(n)      -> Bounded(n) + 1          // [1,n]
math.random(a, b)   -> a + Bounded(b - a)      // [a,b), b excluded
```

That last form is observably upper-exclusive in build 103. It must not be used
as an inclusive damage-range helper. Lua's original CRT-backed `math.random`
and `math.randomseed` still exist in the base library implementation, but the
gameplay environment overwrites `random`, no content chunk calls
`randomseed`, and that CRT stream is not the recovered simulator stream.

### Floating point

`sub_AE54A0` interprets the raw word as signed `int32`, converts it to double,
scales by `2^-32`, and adds `0.5`:

```text
u = float64(int32(nextUint32())) * 2^-32 + 0.5
if u >= 1:
    u = 0
```

The result is `[0,1)`. `sub_9BCEC0` and `sub_9BCEF0` map it as
`minimum + (maximum-minimum)*u`, so the floating upper endpoint is not exactly
reachable. This is unsuitable for the requested exact inclusive integral
Poison ranges.

## Missing authoritative damage sampler

Native Lua binding `sub_A0B670` validates that the third `TakeDamage` argument
is a table of exactly two entries, reports them as `(min, max)`, and decodes
both entries as floating-point numbers. It then calls address `0x00E224A0`.
In the retained build-103 client that address is only:

```asm
mov al, 1
ret
```

The binding consequently exposes the client/server authority seam but not the
server calculation behind it. It initializes the returned selected-damage
float to zero and receives no real selection from this stub. Neither
`ServerData.package` nor `content.db` supplies a Lua replacement: the templates
deliberately enter the native boundary.

Accordingly, these artifacts cannot prove whether the missing authoritative
server used `sub_AE5480`, `sub_AE54A0`, rounding around the float primitive, or
another server-only step. Any stronger attribution would be fabricated.

## Implementation-ready Go contract

The following is the contract to implement later; this recovery does not edit
Go.

### Ownership and construction

```go
type SimulatorRandom struct {
	state [624]uint32
	index uint32
}

func NewSimulatorRandom(seed uint32) *SimulatorRandom
func (r *SimulatorRandom) Uint32() uint32
func (r *SimulatorRandom) Index(count uint32) (uint32, error)
func (r *SimulatorRandom) Float64() float64
```

- Store exactly one `SimulatorRandom` on `sim.Simulator`.
- Construct it once from an explicit `uint32` seed when the simulator is
  constructed. Do not attach it to a session, actor, ability, behavior, or Lua
  chunk.
- Use only the simulator event loop to consume it. Concurrent callers must be
  serialized by simulator event order, not by whichever goroutine reaches a
  mutex first.
- Trace the seed and a monotonically increasing draw ordinal. Replay restores
  the seed and replays the same ordered consumers; a snapshot must retain all
  624 words plus the index.
- `Index(0)` returns an error and consumes no draw. `Index(count > 0)` consumes
  exactly one draw and returns the multiply-high result.
- `Float64` consumes exactly one draw and uses the signed transform above, not
  Go's `math/rand` conversion.

### Inclusive integral rank damage

```go
type DamageRange struct {
	Minimum float32
	Maximum float32
}

func SelectRankDamage(random *SimulatorRandom, damageRange DamageRange) (float32, error)
```

`SelectRankDamage` has this exact behavior:

1. Reject a nil RNG, NaN or infinite endpoints, `Minimum > Maximum`, or
   non-integral endpoints. Rejection consumes no draw.
2. Convert both endpoints exactly to signed integers. Compute
   `span = int64(maximum) - int64(minimum) + 1`; reject spans outside
   `1..math.MaxUint32` without consuming a draw.
3. Call `Index(uint32(span))` exactly once, even for a singleton range.
4. Return `float32(minimumInteger + int64(offset))`.

The mathematical selection is:

```text
minimum + high32(uint64(nextUint32()) * uint64(maximum-minimum+1))
```

Therefore:

- Poison melee `[1,3]` passes `count=3` and can return exactly `1`, `2`, or
  `3`.
- Poison Cloud `[1,4]` passes `count=4` and can return exactly `1`, `2`, `3`,
  or `4`.
- Do not implement either as `math.random(minimum, maximum)` because the
  build-103 override excludes `maximum` in its two-argument form.
- Do not use `% span`; it changes the build-103 distribution and test vectors.
- Do not use `Float64` followed by rounding; it changes both the mapping and
  endpoint probabilities.

Selection is a simulator operation. An actor or ability supplies the range but
does not own or reset the stream. The damage resolver must consume the draw at
one documented point in the simulator event order. Rejected target validation
must happen before selection so a rejected hit does not perturb later draws.
Once a hit commits to damage resolution, selection consumes one draw before
damage mitigation or the tutorial safety-floor clamp.

The last ordering sentence is part of the Go parity contract, not direct proof
of the missing server routine. It gives rollback/retry logic a single stable
consumption point and prevents rejected operations from changing simulator
state.

### Seed-1 oracle

For explicit seed `1`, the first raw words from the recovered seed expansion,
twist, and temper path are:

```text
d3f6b9e5 ef284157 0d3639c7 22f28abf
c8061f7e 57974ebc d0d92ee3 e0db49c9
```

Applying the inclusive contract gives:

```text
[1,3]: 3 3 1 1 3 2 3 3
[1,4]: 4 4 1 1 4 2 4 4
```

These sequences are unit-test oracles for the proposed Go implementation.
They are not retail combat captures.

## What would close the remaining proof gap

One allowed build-103 server trace that records the simulator seed, prior draw
ordinal, authored range, and selected damage would distinguish integer and
float sampling. A `[1,3]` observation of exactly `3`, or a `[1,4]` observation
of exactly `4`, would rule out the build-103 two-argument `math.random` wrapper;
fractional selected damage would rule out the integral contract. Until such an
oracle is added under `bin/server/darkspin/logs/traces`, keep the inclusive
rank-damage policy labeled as the evidence-constrained implementation decision
described above.

## Critical roll and multiplier

Build-103 `sub_9E58E0` provides the authoritative critical-roll formula. A
positive `AutoCrit` returns true without advancing the simulator random stream.
Otherwise a player-controlled attacker uses the one-based difficulty entry in
`DifficultyTuning.RatingConversion`:

```text
chance = min(1, CriticalRating / (RatingConversion[difficulty - 1] * 100))
isCritical = chance > simulatorFloatDraw
```

The comparison is strict. A normal attempt consumes exactly one draw even when
the rating is zero or the computed chance is capped at one. Non-player actors
use conversion `1` in this native branch rather than the player difficulty
table.

The exact build-103 `DifficultyTuning` resource is type `0x8e94f44c`, instance
`0x02ca2581`, decoded size `2120`; its 72 float32 rating conversions start at
byte offset `1832`. The exact `MagicNumbers` resource is type `0x36201b51`,
instance `0x966534e4`, decoded size `76`; the float32 at byte offset `48` is the
base `CriticalDamageBonus`, exactly `2`.

Build-103 `sub_9E5B10` applies:

```text
criticalMultiplier = CriticalDamageBonus + CriticalDamageIncrease
criticalDamage = selectedDamage * criticalMultiplier
```

Thus Blitz at difficulty 1 has `192 / (10 * 100) = 19.2%` base critical chance,
and his authored `CriticalDamageIncrease = 0.5` produces a `2.5x` critical
multiplier. Recipe 31 stores both tuning resources in `content.db`, and the Go
simulator has deterministic pure operations for the roll and multiplier. Live
integration began with Voltic Slash and remains open for the other player and
enemy attack families.

The gameplay authorization snapshot now retains the active creature's template
or saved `CRTR`, equipped-item logical attributes 10 (`CriticalRating`), 19
(`AutoCrit`), and 22 (`CriticalDamageIncrease`), plus an authored passive
operand. The content adapter deliberately preserves two representations for a
token binding: Lightning Rogue Passive's UI token is `percent = 50`, while its
raw `lua_static_property` operand is `criticalDamageIncrease = 0.5`. Combat uses
the latter; the Hero Profile continues to use the former.

Voltic Slash and Ride resolve the formula at their scheduled authoritative hit
deadlines after validating the still-live target (and range for Voltic Slash).
They hold the match-session authority lock across the draw and HP mutation, then
use that one result for objective damage, combat presentation, and death
selection. A Voltic Slash miss pays the already-admitted cooldown but performs
no critical roll.

Sage's direct projectile attack now defers damage selection, the critical draw,
authoritative HP mutation, objective transitions, XP, and death selection until
its scheduled projectile collision. Cast admission publishes only animation,
movement stop, cooldown, and projectile ownership. The impact continuation
revalidates that the target is still live under the session authority lock; a
stale target resolves as a visual miss without consuming either damage or
critical RNG. Its existing 5-8 weapon compatibility range, and Voltic Slash's
existing fixed compatibility damage, now pass through the authenticated
attacker-side equipped-item projection before range sampling and critical
resolution. Sphere of Transfusion now projects each authored 3-damage pulse
through the same authenticated profile using its build-103 coefficient `0.05`,
descriptor mask `140` (energy, area, DoT), damage type `2`, and damage source
`0` before range selection and critical resolution. An empty modifier profile
leaves the existing fixed/ranged compatibility attacks unchanged; unverified
weapon and target-side stages remain disabled. Schedule failure rolls back the cooldown, release gate, projectile
ID, and active run. Sphere of Transfusion resolves independently for each live
target on each pulse, in stable object-ID order. All target rolls finish before
any HP mutation, so a resolver failure leaves the entire pulse unapplied rather
than partially damaging the area.

## Healing projection

Build-103 `sub_9E5550` projects one authored healing amount as:

```text
primaryMultiplier = 1 + (primaryAttribute + 1) * healingCoefficient
modifierMultiplier = 1 + attribute[96]
if descriptors & 0x1000: modifierMultiplier += attribute[98]
healing = authoredHealing * primaryMultiplier * modifierMultiplier
```

The `+1` primary baseline is the same `sub_9E4E60` class-primary path used by
damage. `sub_9E5500` proves the general attribute-96 contribution and gates
attribute 98 on the HoT descriptor. The native function returns a float; the
separate Hero Profile detokenizer truncates it only when producing page text.
Authoritative combat therefore retains fractional healing.

Tree of Life chunk 460 supplies coefficient `0.05`, descriptors `4104`
(`AoE | HoT`), type `2`, source `1`, six 3-point pulses, and an additional
12-18 final range. Its live continuation now selects the raw final range from
the match simulator stream, applies Sage's authenticated primary/healing
profile, and heals every living squad member by that one projected pulse
amount. Singleton ordinary pulses retain their raw authored selection without
an unnecessary random draw. Missing metadata disables only the descriptor-
gated attribute-98 branch.

A killing critical emits flags `0x000d` and enters the recovered `0.1s + 3s`
non-boss critical corpse branch; a non-killing critical emits flags `0x0009`.
Recipe 32 decodes the shared 88-byte class-stat record rather than treating its
first float as the only usable non-player field. The same native-backed stat
projection used for heroes yields:

```text
dodgeRating = baseDodge + Dexterity * 6
resistRating = baseResist + Mind * 6
criticalRating = baseCritical + Dexterity * 4
```

`HelperMelee` therefore has critical rating `1 + 1*4 = 5`; each of the five
tutorial enemy families has `5 + 10*4 = 45`. Companion melee resolves its own
rating with the native non-player conversion `1` after live hit validation, and
carries the result through damage, critical presentation, and fast critical
death. Tutorial enemy timed-melee attacks now resolve once per accepted actor
contact; Poison Cloud and Plasma Lightning resolve at accepted projectile
collision; BurstShot resolves independently for each accepted projectile.
Misses, expired projectiles, stale controlled targets, and invalid contacts do
not call the critical resolver. These enemy results drive actual squad HP and
the build-103 critical flag. Neither actor family inherits the deployed player's
snapshot.

## Target healing reduction

Build-103 `sub_9E5D10` is the recipient-healing path. It applies this
target-owned stage to an already projected healing amount immediately before
adding the result to current HP:

```text
receivedHealing *= 1 - targetModifier[30]
```

Gameplay authorization now snapshots equipped logical attribute 30 for each
selected hero. Tree of Life applies each living recipient's own reduction after
Sage's healer-side projection, so reserve heroes do not inherit the deployed
hero's equipped modifier. A negative modifier increases received healing. A
modifier above one is clamped to zero final healing instead of turning a heal
into damage. Attribute 30 is not a damage-defense stage; tutorial enemy damage
therefore remains unchanged by it.

## Ability timing

Build-103 `sub_438F00` selects one of two modifier branches from descriptor bit
`0x2`:

```text
basicDuration = authoredDuration / (1 + max(attribute[23], -0.9))
nonBasicCooldown = authoredCooldown * (1 - attribute[24])
```

The first branch's `-0.9` floor prevents a zero or negative divisor. The second
branch is the authored cooldown-reduction percentage. Missing descriptor
metadata remains an identity operation, and reductions beyond 100% clamp at a
zero duration.

Gameplay authorization snapshots both timing operands. Sphere of Transfusion,
Tree of Life, and Ride the Lightning have complete non-basic descriptors, so
their client cooldown packets and server admission gates now share the same
projected cooldown. Ride's build-103 chunk 129 is compiled through the exact
`Modifiers!modifier_shocked.lua` dependency: Game's case-insensitive FNV-1
resource identity maps it to `Modifiers/0xEABB0526.lua` (chunk 901, SHA-256
`f2a8a51e3cb4c52d255784c43f85edc89e0d949f423cbbd90f6177fa6876003a`), whose
static definition proves descriptor 32 and the three-second Shock duration.
Ride also consumes its authored 370ms hit and 770ms release deadlines from the
compiled definition. Its 13-power/fixed-18-damage live compatibility remains
explicitly separate from the raw 12-power and 12-18 content operands. Voltic
Slash now projects a detached copy of its complete five-entry basic sequence:
cooldown, every hit and release deadline, the chosen animation continuation,
and the held-repeat admission clock all use the same AttackSpeed divisor. The
content catalog remains immutable.

Projectile speed has a distinct native chain. `sub_A0FBE0` reads attacker
attribute 26 (`ProjectileSpeedIncrease`) and, only when positive, requests the
same amount as attribute 48 (`MovementSpeedBuff`) on the projectile before
motion begins. `sub_A2FC20` reads attribute 48, adds one, clamps that multiplier
at zero, and applies it to the velocity-times-step displacement used by both
movement and swept collision. The effective attacker-owned formula is therefore
`authoredSpeed * (1 + max(ProjectileSpeedIncrease, 0))`; negative attacker
values are not copied. Sage's authenticated shot now publishes that speed and
uses it for the authoritative travel/collision deadline. Its cooldown, 170ms
hit delay, and 450ms release are projected through the same basic AttackSpeed
branch at admission, so animation and projectile timing cannot diverge.
Searching the build-103 executable found no
dodge/resist `0..1` random roll; the only such combat draw is critical. Dodge
and resist therefore remain server-authority gaps rather than inferred client
rolls.

## Mana cost

Build-103 `sub_9DD720` calls `nAbility.GetManaCost`, then performs the exact
nonzero-cost projection. Its `ability + 448 + 4*rank` read is the authored
ranked `manaCoefficient`; the other multiplier operand is the actor's
class-primary aggregate returned by `sub_9E3680`, including that function's
optional ability-owned primary override:

```text
if actor is overdrive charged: cost = 0
else if authoredCost == 0: cost = 0
else: cost = authoredCost + selectedProperty * rankCoefficient
```

The function retains a float throughout admission and payment; the later
payment path subtracts it from current mana and clamps the remaining mana at
zero, without rounding the cost first. The ability definition constructor at
`sub_9DF590` initializes rank coefficient zero to `0` and ranks 1 through 8 to
the exact float `0.1`; authored Lua can replace those defaults. This is not the
commonly assumed `base * (1 - ManaCostReduction)` formula. The ReCap reference
implementation's `0.05 * value` branch is explicitly marked TODO and is not
authoritative.

The constrained Lua compiler retains the ranked coefficients for the three
live tutorial abilities: Ride the Lightning `0.06`, Sphere of Transfusion
`0.10`, and Tree of Life `0.15`. Their Build-103 tutorial hero snapshots publish
class-primary `12` for Blitz and `15` for Sage, producing exact admitted and
paid costs `12.72`, `21.5`, and `32.25`. `ResolveAbilityManaCost` preserves the
float without rounding, and each ability uses the same projected value for
power admission, simulator intent, subtraction, and the client mana update.
Zero-authored costs remain zero. The server does not yet own the runtime
overdrive-charged flag, so the native free-cast branch remains disabled rather
than being inferred from the separate overdrive-unlock tutorial event.
