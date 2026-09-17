# Remaining hero-basic compiler boundaries

## Result

The recipe-40 database does not leave one common compiler problem for the
remaining four definitions. It proves three different boundaries:

| Definition | Exact next boundary | Safe now | Genuinely absent |
|---|---|---|---|
| `ShadowRavagerBasic` | implemented campaign multi-hit execution | The compiler preserves all five two-hit rows; runtime selects one row and schedules a complete damage commit at each authored absolute timestamp. | Nothing missing for timing or damage count. |
| `TimeRavagerBasic` | implemented campaign multi-hit execution | The fifth row executes both hits; the first four rows remain ordinary single hits. | Nothing missing for timing. The modifier alias and AttributeUtils/bootstrap work are complete. |
| `PlasmaSentinelBasic` | implemented paired melee callback policy | Parallel `hitEffects`, `hitModifiers`, and `modifierChances` remain zipped through animation-index selection, effect notification, inclusive roll, and modifier request. | Nothing in this policy is missing. Shock and Burn bodies are packaged and linked. |
| `LightspeedTempestBasic` | implemented scoped compatibility link | Orion alone substitutes exact counter chunk `339`, template chunk `169`, haste chunk `201`, and AttributeUtils chunk `87`. | The generic aggregator body remains absent, so Citadel receives no substitution. |

All four former boundaries are now implemented. The evidence rules below
remain the parity contract: Ravager hits are separate commits, Plasma arrays
stay zipped, and the Lightspeed fallback remains scoped to Orion rather than
becoming a fabricated global alias.

The 0.6.10 Zrin capture showed authoritative damage and death projection but
no `ObjectEffectMessage` for either exact `PlasmaSentinelBasic` hit asset. Hero
melee now suppresses speculative timed-run hit presentation and emits the
selected content-authored `HitEffectName` from the committed, non-immune hit,
including attacker, target, facing, and critical state. The accepted hit also
emits the client's common player-melee contact event unless that is already the
authored event, preserving both the hero-specific elemental/status presentation
and the shared impact sound and visual. The 0.6.15 follow-up proved that merely appending this event from
the outer hit continuation placed it before the authoritative damage
publication. It now uses the shared accepted-result publisher's after-damage
callback, preserving the recovered `CombatEvent` then melee `ServerEvent`
ordering for every melee hero.

The shared melee-pursuit handoff now retains its progress continuation when a
target moves out of range during the arrival-to-attack retry. This closes the
Wraith Pummel race where the client received another pursuit response after
the server had already ended its progress loop and could follow indefinitely
without reaching an admitted hit. Arrival failures now cancel the pursuit and
publish an explicit stop and rejection instead of dropping all recovery
packets with a scheduled-producer error.

## Evidence set

The authoritative database is
`bin/game/logs/content-recipe40/darkspin/cache/content.db`. The relevant rows
are:

| Body | Chunk | Resource | SHA-256 |
|---|---:|---:|---|
| `TimeRavagerBasic`, `Abilities/0x5FB76978.lua` | `385` | `13933` | `e810d003b1d9920a425b649fece4f4e3c3837f8afddc35d20bdd678931b055d4` |
| `PlasmaSentinelBasic`, `Abilities/0x5CF2E729.lua` | `409` | `13960` | `ae92ae103e71a51028993223cfdf7a588635db7d35731a1de651c1546282d987` |
| `ShadowRavagerBasic`, `Abilities/0x5D05F491.lua` | `674` | `14245` | `f5d5d0217cd9a68f48bf67b3090d126b6e2664cc0c5be15192eab4c391d09700` |
| `template_ability_melee`, `Abilities/0x7BF2D7DD.lua` | `832` | `14409` | `2951fef16876a72687c235cc93377ede9dcec31286a8237a9c6a8af978da778a` |
| `LightspeedTempestBasic`, `Abilities/0x55608FEA.lua` | `938` | `14517` | `82083fe2d37f13ca2a8035564c3e561fbe01dafed18847648dd284a000a54309` |

Read-only extracts used for this pass are under
`bin/game/logs/ability-basic-remaining`. Floating-point values below are
reported at authored millisecond precision; the bytecode constants are
ordinary Lua numbers with the expected float32 round-off.

## Ravager animation rows and damage commits

### Shadow Ravager

Every Shadow row has two hit timestamps:

| Sequence index | Animation | Hit 1 | Hit 2 | Release |
|---:|---|---:|---:|---:|
| `0` | `su_shadowRavager_basic` | 260 ms | 310 ms | 430 ms |
| `1` | `su_shadowRavager_basic_b` | 240 ms | 290 ms | 440 ms |
| `2` | `su_shadowRavager_basic_c` | 280 ms | 350 ms | 520 ms |
| `3` | `su_shadowRavager_basic_d` | 250 ms | 350 ms | 525 ms |
| `4` | `su_shadowRavager_basic_e` | 200 ms | 280 ms | 675 ms |

These are absolute offsets from the start of the selected animation, not a
delay followed by an interval. For example, row 0 commits at `start+260 ms`
and `start+310 ms`; the second commit is not `start+570 ms`.

### Time Ravager

Time has four single-hit rows and one multi-hit finisher:

| Sequence index | Animation | Authored hit timestamp(s) | Release |
|---:|---|---:|---:|
| `0` | `sp_timeRavager_basic_a` | 200 ms | 330 ms |
| `1` | `sp_timeRavager_basic_b` | 233.333 ms | 360 ms |
| `2` | `sp_timeRavager_basic_c` | 266.667 ms | 400 ms |
| `3` | `sp_timeRavager_basic_d` | 233.333 ms | 370 ms |
| `4` | `sp_timeRavager_basic_e` | 300 ms, 333.333 ms | 660 ms |

The fifth animation therefore commits at `start+300 ms` and
`start+333.333 ms`, only about 33.333 ms apart. Treating the pair as a range,
choosing one endpoint, summing the values, or applying two hits to all five
Time animations would contradict the root table.

### What one timestamp means

Chunk 832's `GetTimeToHit` returns the selected animation row's `hit` member.
For sequence selection it converts the native zero-based animation index to
Lua's one-based table index. Its activation path converts a scalar hit to a
one-element table, iterates the table with `ipairs`, and calls
`nThread.WaitUntilTime` for each entry. After every wait it repeats the full
target-validation and hit path: construct the arc, resolve the target, call
`OnHitTarget`, call `nGameObject.TakeDamage` with `GetDamage`, damage type,
damage source, coefficient, descriptors, and multiplier, publish the hit
effect, and call the ability's `OnDamageTarget` callback. Thus two timestamps
mean two independent full-damage attempts and two possible effect/callback
publications. They are not one damage result with two cosmetic effects.

Cooldown and mana payment occur once in the activation path, not once per
timestamp. A later timestamp still revalidates the target, so a target that
was defeated or became invalid after the first hit must not receive a second
commit.

The typed compiler and runtime boundaries are crossed:
`AnimationHitDelays` retains the nested rows, `IsMultiHitAnimation` identifies
them, and campaign admission schedules the selected row as follows:

1. Sequence selection returns the selected row's entire hit schedule,
   while retaining its one animation and one release timestamp.
2. Projection transforms every member of `AnimationHitDelays` through the same authored
   attack-speed/timing transform already applied to scalar `HitDelay` and
   `HitDelays`.
3. Admission validates the whole operation before publishing it, then schedules
   one commit producer for each absolute timestamp so schedule failure can
   roll back the accepted run consistently.
4. Each producer revalidates and performs a separate damage/critical draw,
   hit effect, ability-specific `OnDamageTarget` behavior, defeat transition,
   and semantic trace. Stop later damage after defeat or invalidation.
5. Cooldown and release state commit once; the row's multiple hit entries do
   not duplicate admission, animation, or cooldown.

The database proves the commit count, ordering, timestamps, and callback
placement. It does not prescribe a new storage transaction API or packet
batching design; those are server implementation concerns, not absent
content.

## Plasma Sentinel paired policy

Chunk 409 authors three parallel two-element arrays:

| Pair index | Sequence animations | Hit effect | Hit modifier | Rank-1 chance |
|---:|---|---|---|---:|
| `1` | A (`0`), C (`2`) | `plasma_common_electric_hit_medium_player_effect.ServerEventDef` | `PlasmaSentinelShock` | 10% |
| `2` | B (`1`), D (`3`) | `plasma_common_fire_hit_medium_player_effect.ServerEventDef` | `PlasmaSentinelBurn` | 100% |

The root callback obtains the native zero-based
`GetAnimationSequenceIndex()`, computes
`math.mod(animationIndex, #hitModifiers) + 1`, and uses that same selected
index for all three arrays. This is parity selection over the four-animation
sequence: A/C are electric+shock+10, and B/D are fire+burn+100. The arrays
must remain zipped; independent random selection, rotating each array
separately, or flattening to one generic `hitEffect` would be wrong.

For every successful melee damage callback, the root first publishes the
selected effect event with the damaged object, attacker, and critical flag.
It then evaluates an inclusive `math.random(1, 100)` draw against the selected
ranked chance using `draw <= chance`. On success it calls
`nModifier.RequestModifier` for the selected modifier on that same damaged
target, carrying the attacker ID, attacker attribute snapshot, ability rank,
and callback context. Consequently the 100% burn branch still performs the
authored draw but always succeeds for valid draws; the 10% shock branch
succeeds for draws 1 through 10.

No Plasma content recovery is needed for this policy. `PlasmaSentinelShock`
also occurs in chunk `732`, `Modifiers/0x2C20F980.lua`, resource `14304`,
SHA-256 `fd8c76bceebcf4b077735e0427963b259d8a4afd6a99aceed0975cd480f3be89`;
that body defines `nModifier_PlasmaSentinel_Shock`, duration 1, descriptors
32, and registers the matching runtime name. The burn body is chunk `306`,
and its module alias is already implemented as stated in the task.

The next safe compiler/runtime slice is therefore a distinct paired-hit melee
projection (or equivalently explicit parallel typed fields with strict equal-
length validation), followed by a campaign callback that uses the selected
animation index and the server's deterministic random source. Reject malformed
or unequal arrays; do not silently fall back to projectile decoding or a
single generic effect.

The current server owns that strict paired projection and deterministic
inclusive chance draw. Successful nonimmune, nonlethal hits now install either
a one-second immobilizing Shock or a five-second Burn with one-second ticks and
rank-one damage `1 + primary-snapshot scaling at coefficient 0.05`. Both use
the build-103 modifier lifecycle, replace the previous caster-unique entry,
release pooled instances on replacement/expiry/session teardown, and route
Burn damage through normal enemy death, loot, stats, horde, boss, and shield
transitions.

## Lightspeed: exact missing boundary

Recipe 40 records three dependencies for chunk 938. The projectile template
resolves to chunk `38`; the haste module is now handled by the implemented
alias/bootstrap work; but dependency ordinal 1 remains:

`Modifiers!modifier_chain_ability_counter.lua` -> `target_lua_chunk_id = NULL`.

The same unresolved require appears in only one other root, chunk `4`
(`CitadelSpecificFour_Bolt`). The case-insensitive FNV-1 identity for
`modifier_chain_ability_counter` is `0xA53AC836`, but recipe 40 contains no
matching content resource, Lua chunk, or module alias. This is genuinely
absent data, not another constrained-decoder shape.

Two nearby bodies are real and independently identifiable:

- chunk `169`, `Modifiers/0x184258A8.lua`, is exactly
  `template_chain_counter`; it supplies generic stack-event behavior;
- chunk `339`, `Modifiers/0x82C8A3E8.lua`, is exactly
  `modifier_lightspeedtempest_basic_counter`; it derives from chunk 169,
  registers `LightspeedBasicStackCounter`, and authors max stack 2 and
  duration 1 second.

Those are safe aliases only under their own authored names. Neither is an
instruction-proven replacement for the differently named missing module.
Because an unrelated Citadel root also requires the missing name, directly
aliasing it to Orion's concrete chunk 339 is especially unsound. Aliasing it
to the generic template 169 would also omit any unknown aggregator side
effects. Requiring both would manufacture load order and registration policy
that the database does not contain.

The server now applies a deliberately narrow fallback under the documented
educated-guess policy. While compiling `LightspeedTempestBasic` only, it maps
the absent require to the shipped Orion-specific counter chunk 339, loads that
counter's shipped generic template chunk 169, and loads the independently
verified haste chunk 201. It does not write a global content alias, so the
unrelated Citadel require remains unresolved. This unlocks all four Orion
variants without claiming the absent aggregator was recovered; a live
stack/haste verification or another build can still replace the fallback.

## Implemented work versus missing evidence

Implemented from the recovered authored data:

- execute the already-decoded Shadow and Time nested hit schedules as ordered
  absolute offsets with one full damage commit per entry;
- project the nested hit schedules through the same attack-speed timing rule
  used for the existing scalar and per-animation delays;
- make multi-hit admission/scheduling transactional, with later-hit target
  revalidation and one cooldown/release lifecycle;
- decode and validate Plasma's three zipped arrays;
- select Plasma's pair by `animationIndex % 2`, always emit its selected hit
  effect after successful damage, then perform the exact inclusive chance roll
  and request the paired modifier on success;
- retain exact aliases for chain template chunk 169 and Orion counter chunk
  339 only when their own authored module names are requested.

Work that is not justified by recipe 40:

- inventing a second-hit damage scale, shared critical result, combined damage
  packet, or cosmetic-only second Ravager hit;
- choosing Plasma's effect, modifier, or chance independently, skipping the
  100% branch's draw, or applying a modifier before the damage callback;
- mapping `modifier_chain_ability_counter.lua` directly to chunk 169 or 339,
  or fabricating an aggregator that loads both.

Only the final Lightspeed module/aggregator semantics remain absent. The
scoped fallback allows all 100 hero basic definitions to compile, while its
assumed Orion load side effect remains explicitly tracked in `notes/help.md`.
