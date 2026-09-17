# 1-1 low-band enemy action policy (build 103)

## Result and evidence labels

At difficulty `1-24`, base `zelems_1` has one minion family, three agent
families, and three ordinary captain families relevant to Wanderer/Spike and
the walkthrough-backed traversal population. The focused policies below also
include walkthrough-visible Pack Brawler; its exact family and action policy
are recovered, while the missing overlay/selector that admits it into the
observed 1-1 roster remains separate:

| Director role | Noun | AI family |
| --- | --- | --- |
| minion | `ZelemBasicRanged.Noun` | `ZelemBasicRanged.AIDefinition` |
| agent | `ZelemBasicHybrid.Noun` | `ZelemBasicHybrid.AIDefinition` |
| agent | `ZelemBasicRepair.Noun` | `ZelemBasicRepair.AIDefinition` |
| observed/overlay agent | `ZelemBasicPackMelee.Noun` | `ZelemBasicPackMelee.AIDefinition` |
| captain | `ZelemSpecialHaster.Noun` | `ZelemSpecialHaster.AIDefinition` |
| captain | `NomadSnipe.Noun` | `NomadSnipe.AIDefinition` |
| captain | `NomadWithDrone.Noun` | `NomadWithDrone.AIDefinition` |

The tables below use these labels:

- **Exact content** means a value or operation decoded from the build-103 noun,
  phase, Lua chunk, or linked template in the authoritative runtime
  `content.db`.
- **Exact native** means behavior retained in
  `bin/game/GameBin/Game.c` or a build-103 receiver contract already
  instruction-checked in the cited notes.
- **Video** means a visible observation from
  `bin/video/walkthrough/1-1/1-1.mkv`; it is corroboration, not numeric server
  authority.
- **Fallback** means the deterministic server policy to use until missing
  server-side selection or scheduling evidence is recovered.

The low-band/rank-one mapping is a fallback. The content proves that the base
nouns are eligible at difficulty `1-24` and that the abilities have ranked
arrays whose first elements are reported below; no retained server body proves
the campaign difficulty-to-ability-rank conversion.

## Common native action contract

The following is exact native/Lua behavior and applies to all seven focused families:

1. Each object owns its own blackboard, behavior thread, active-ability state,
   and cooldowns. There is no wave-wide attack lock or shared cooldown.
2. All four AI definitions begin in phase zero and author `faceTarget=true`.
3. All four non-player classes author `aggroRange=0` and `alertRange=0`.
   Remote acquisition therefore requires the campaign owner to insert a
   hostile target; an invented perception radius is not content parity.
4. Ability admission checks a live/legal target, blocking state, conflicting
   ability, cooldown, mana, range, and hit predicates. A rejected request
   creates no ability instance and pays no cooldown.
5. An out-of-range result can be classified as pursuit, but the top-level
   campaign NPC scheduler that turns that result into a goal and retries it is
   absent. Locomotion begins without a goal.
6. On acceptance, `faceTarget=true` uses the target-facing `0x91` turn shape.
   A walking goal instead uses `0x91` followed by `0x95`. The target-bearing
   `0x99` must precede either presentation.
7. The instant-cast, projectile, and melee templates play the authored
   animation, wait to `timetohit`, then pay cooldown/mana and apply the hit or
   launch. They retain ownership until `timetorelease`; release and cooldown
   are independent gates.
8. Actors without an authored combat-idle leaf use
   `nBehavior_CombatIdle`: turn when the target is more than 30 degrees off
   facing, then wait `0.1 + random()*0.5` seconds and repeat. This idle delay is
   not an attack cooldown.

### Deterministic scheduler fallback

Use the following server-authority policy for implementation:

1. After the object's create/component/attribute baselines, inject the deployed
   live hero as that object's target using the existing 1-1 targeting fallback.
2. Deactivate the pre-aggro leaf, run the noun's first-aggro presentation to
   completion, and only then admit a phase action. A family with no authored
   first-aggro ability becomes phase-eligible on the next think.
3. Evaluate phase gambits in authored order and choose the first whose named
   condition is satisfied. Treat a conditionless entry as the fallback entry.
4. Think every `100 ms`, and also immediately after target change, navigation
   completion, ability release, or cooldown maturity. The fixed cadence is
   fallback policy, not a recovered retail interval.
5. On out-of-range admission, pursue the live target and keep asking native-
   equivalent admission; stop because admission succeeds, not at a separately
   invented geometric inset. Moving targets continuously replace the goal.
6. While cooldown remains after release, run the authored combat-idle leaf or
   the exact generic fallback. Do not serialize one actor behind another.

## Numeric action summary

Times are seconds from acceptance of the named ability. Cooldowns are authored
base cooldowns and begin at the hit/launch column, not at acceptance.

| Family | First phase choice | First damaging ability | Activation range | Hit / launch | Release | Cooldown | Combat movement |
| --- | --- | --- | ---: | ---: | ---: | ---: | --- |
| `ZelemBasicRanged` | `ZelemBasicRanged_Blink` | chained `ZelemBasicRanged` projectile | blink `50`; shot `12.5` | blink/chain `3.2`; shot `0.666667` | shot `1.066667` | blink `7`; shot `5` | pursue to blink admission; teleport to a reachable target-relative point using `2 / 8 / 10`; fire from the result |
| `ZelemBasicHybrid` | `ZelemBasicHybridProjectile` when `Distance > 8`, else `ZelemBasicHybridMelee` | selected projectile or melee | shot `15/18/21`; melee `1.125` | shot `0.4667`; melee `0.26` | shot `1.6`; melee `1.2` | both `2.5/2/1.5` | pursue for the selected action; crossing the strict eight-unit condition changes the next phase choice |
| `ZelemBasicRepair` | `Repair` when `ShouldRepair`, else `ArcWeldingMelee` | `ArcWeldingMelee` | repair scan/ability `20`; melee `0.75` | repair is alert `1.166667` plus travel and `3.233334`; melee `0.533333` | repair after modifier request; melee `1.6` | repair `0`; melee `1.5` | condition supplies the dead ally and Repair approaches it; otherwise pursue the hostile target for melee |
| `ZelemBasicPackMelee` | `ZelemBasicPackMeleeCower` when `AllyNotNearby`, else `ZelemBasicPackMeleeAttack` | `ZelemBasicPackMeleeAttack` | ally scan `20`; attack `1.125` | cower `3/2/1`; attack hit `0.53` | cower after its ranked wait; attack `1.3` | cower `24`; attack `2/1.5/1` | cower in place only when no other living teammate is within 20; otherwise pursue for melee |
| `ZelemSpecialHaster` | `CastZelemHasteBuff` when `AllyNeedsBuff`, else `ZelemHasterAttack` | `ZelemHasterAttack` projectile | buff `30`; shot `14` | buff `0.26667`; shot `0.33` | buff `1.06667`; shot `1.0` | buff `4`; shot `3` | pre-aggro wander radius `5`; pursue selected attack target to range `14` |
| `NomadSnipe` | `NomadSnipe_Slow` when `EnemyNeedsDebuff`, else `NomadSnipe_Melee` | `NomadSnipe_Melee` | slow `12`; melee `1.5` | slow `0.5`; melee `0.466667` | slow `1.3`; melee `1.5` | slow `4`; melee `1.5` | pursue to `12` for the debuff, then close to `1.5` while it is present |
| `NomadWithDrone` | `NomadWithDronePunch` | `NomadWithDronePunch` | `2.0` | `0.515152` | `1.666667` | `2.4` | pursue to `2.0`; authored combat idle is stationary `nBehavior_Idle` |

The Zelem blink row has two clocks. With the fallback chain below, the earliest
projectile launch is approximately `3.866667s` after blink acceptance
(`3.2 + 0.666667`), excluding first-aggro presentation and scheduler
quantization. The two component waits are exact Lua values; their summed
end-to-end use is fallback because the absent campaign owner controls request
handoff.

## `ZelemBasicRanged`: blink, reposition, shoot

### Exact noun/Lua/native evidence

- Phase resource `9351` has one phase ability string,
  `ZelemBasicRanged_Blink`; AI resource `9355` links that phase.
- Blink chunk `622`, resource `14186`,
  `Abilities/0x968C4A71.lua`, SHA-256
  `71bedb16491ee9935d663872b9e314186ee32683a5b6a45bc68fef1086af89ae`,
  authors rank-one cooldown `7`, range `50`, `timetohit=3.2`, animations
  `zlm_minn_sp_1_blink_cast3/2/1`, and reachable-position distances minimum
  `2`, normal `8`, maximum `10`.
- At the blink continuation, the script anchors navigation at the valid
  target's position (or the actor's position when the target is invalid), calls
  `GetRandomReachablePosition`, and teleports on success. With a still-valid
  hostile target it clears cooldown time for `SPID("ZelemBasicRanged")`,
  requests `StackingCountModifier`, and calls `RequestAbility` for
  `ZelemBasicRanged` with the live target position.
- Shot chunk `700`, resource `14271`,
  `Abilities/0x941F8C76.lua`, SHA-256
  `90dc47bb2f4bb4689db35decf51dd278367a9320462846b36a4b1f6145d45da5`,
  authors range `12.5`, launch `0.666667`, release `1.066667`, cooldown `5`,
  projectile speed `8`, travel budget `30`, and rank-one damage `2..6` plus
  coefficient `0.05`. It uses spacetime/energy damage and the
  `zlm_minn_sp_1_attack1` warp animation (with a separate no-warp animation).
- Pre-aggro is `nBehavior_Invisible`: the object is invisible and receives
  scoped `Intangible=1` and `InvisibleToSecurityTeleporters=1`. Replacement
  restores visibility/tangibility before the next behavior.
- First aggro is `FirstAggro_Anim` (chunk `111`, resource `13635`). It calls
  `SetAnimationStateToAggro` and waits the duration returned by that native;
  the content does not author a fixed numeric delay. First alert instead names
  `FirstAggro_FaceTarget`, whose chunk `575` yields once and relies on native
  target-facing behavior.

### Fallback action policy

Use rank one. After `FirstAggro_Anim` completes, request Blink. If it is out of
range, pursue until it admits. At `t=3.2`, choose a reachable position around
the current target in the authored `2..10` envelope, preferring radius `8`;
use stable angle sampling seeded by actor ID. Teleport, clear the shot cooldown,
and request the shot immediately. If no reachable destination exists, retain
the current position and still try the shot only if its `12.5` admission passes;
otherwise resume pursuit/blink selection without paying an invented shot
cooldown. Face the live target for the accepted shot and refresh the target
position at launch.

Do not replace Blink with ordinary walking when it admits, do not collapse the
blink and projectile cooldowns into one timer, and do not make every Zelem
visible at spawn merely because the walkthrough usually enters combat after
the pack is already active.

The current server implements this loop with two explicit authority fallbacks:
the unrecovered first-aggro duration is treated as zero, and the reachable
destination is radius `8` at a stable actor-ID-derived angle because no server
navmesh sampler is available. It retains the teleport as authoritative enemy
state, publishes teleport/update/warp-in presentation, chains the exact
projectile through shared collision and hero damage/death/stat handling, and
re-enters Blink on its seven-second start cadence. It deliberately does not
publish the script's transient `StackingCountModifier` until its wire lifetime
is evidenced. Each launched projectile is session-owned: cleanup cancels its
scheduler and stops its simulator, while stale generations emit no motion or
impact. Source defeat, terminal match state, and committed Beam Out also retire
the run silently; only a target swap inside the same live session resolves as
the authored miss. Normal projectile completion stops only the projectile
simulator and leaves its containing schedule alive for the next Blink cycle.

The client retains its last unreliable locomotion goal independently of the
reliable stop command. Blink admission, teleport completion, and the chained
shot now republish the Barracuda's authoritative stationary position through
both channels, eliminating the small lateral interpolation left over from its
approach.

## `ZelemSpecialHaster`: ally haste, then ranged shot

### Exact noun/Lua/native evidence

- Phase resource `6402` stores two authored entries in order:
  `AllyNeedsBuff -> CastZelemHasteBuff`, then `ZelemHasterAttack`.
  AI resource `6406` links the phase.
- Buff chunk `573`, resource `14135`,
  `Abilities/0x48707BEA.lua`, SHA-256
  `fabe296fc704b48543ecce70c6fd6e0d869580d42076616033c6dbb2b4d95bd2`,
  targets allies at range `30`, hits at `0.26667`, releases at `1.06667`, and
  has cooldown `4`. It applies `ZelemHasteBuff` and preloads animation
  `zlm_lieu_sp_03_attack2`.
- Modifier chunk `475`, resource `14029`, authors duration `20`, max stack `1`,
  `attackSpeedScale=0.25`, `cooldownScale=0.25`, and
  `speedAdjustment=0.5`. Shared AttributeUtils chunk `87`
  (`lua/0x4C983F6A.lua`, SHA-256
  `861450587408da51e6d0a937d7fbe32d7b3a17dffff04f5bfbfded987f228820`)
  proves that `CreateStackingAttributeFunctions` adds the ranked operand to its
  named attribute, and on a stack event replaces the retained handle with
  `stack count * ranked operand`. The modifier binds those functions to
  `AttackSpeedScale`, `MovementSpeedBuff`, and `CooldownScale`; its unique,
  max-stack-one policy therefore applies exact additive `+0.25`, `+0.50`, and
  `+0.25` modifiers and refreshes the 20-second duration without increasing
  their magnitude. Build-103 timing semantics divide basic-attack duration by
  `1 + AttackSpeedScale`, multiply other cooldowns by `1 - CooldownScale`, and
  multiply movement speed by `1 + MovementSpeedBuff`: the authored results are
  20-percent shorter basic-attack intervals, 25-percent shorter cooldowns, and
  50-percent faster movement.
- Attack chunk `452`, resource `14006`,
  `Abilities/0xBBAE6A9E.lua`, SHA-256
  `dd1de80ffcf502e3662063136a186151cba15a595b78a387ff761a5bbc13d83e`,
  authors range `14`, launch `0.33`, release `1`, cooldown `3`, projectile
  speed `25`, travel budget `50`, and rank-one damage `5..10` plus coefficient
  `0.05`. It uses spacetime/energy damage and animation
  `zlm_lieu_sp_03_attack1`. The projectile is `Ability_Fireball.Noun`, offset
  `(0.5,2.2,0.7)`, with trail
  `spacetime_lieu_haster_shot_effect.ServerEventDef`, impact
  `spacetime_bite.ServerEventDef`, and miss
  `ineffective_common_small.ServerEventDef`.
- Pre-aggro is `nBehavior_Wander`; exact chunk `973` chooses navigable points
  within radius `5`. The AI definition has no first-aggro ability string, so no
  beam-in/activation delay may be borrowed from another family.

### Fallback action policy

Evaluate `AllyNeedsBuff` first. Select the nearest living in-range ally, using
object ID as the distance tie-breaker, and use self only when no other ally is
present. Cast once; max stack `1` prevents
refresh spam while the 20-second modifier is present. Otherwise target the
deployed hero and use `ZelemHasterAttack`, pursuing to admission at range `14`.
On buff expiry the authored first gambit becomes eligible again.

The current server implements this lifecycle: it creates the modifier on the
selected nearby ally, applies its recovered attack-speed, cooldown, and
movement transforms to authoritative NPC timing, fires the authored ranged
projectile, deletes the modifier at 20 seconds, and returns to the buff-first
gambit. When no ally is present, the same path targets Haster itself. The
modifier handle is session-owned so disconnect,
replacement, and cleanup release it even when delayed callbacks are cancelled.
The launched projectile uses the same session-owned cancellation boundary.
Source defeat, terminal match state, and committed Beam Out retire its pending
windup or impact without leaking a miss into a replacement session. A target
swap within the same live session remains a normal authored miss.
Normal projectile completion likewise leaves the containing schedule alive so
the authored three-second `ZelemHasterAttack` continuation can execute.
The delayed Haste creation uses the same live-source gate, so a source defeated
between cast admission and the `0.26667s` hit cannot publish a late modifier.

## `ZelemBasicHybrid`: ranged beyond eight, melee at or within eight

### Exact phase selection

- Phase package ordinal `11836`, type `0x30728CE7`, instance `0xBDA683F8`,
  decoded size `939`, SHA-256
  `c0aa12bfed338051b26eb99c3e15426007f1b4e2079ef4dfc03cf97a789b038d`,
  contains two entries in authored order. Both use the `Distance` condition,
  threshold float `8.0`, and `_GreaterThan`; the projectile entry requires the
  comparison to be `true`, while the melee entry requires it to be `false`.
  Thus a target beyond eight selects `ZelemBasicHybridProjectile`, and a
  target exactly eight units away or closer selects `ZelemBasicHybridMelee`.
  There is no random selector or missing third action in this phase.

### Exact projectile contract

- Projectile chunk `259`, resource `13800`,
  `Abilities/0xF98BF178.lua`, SHA-256
  `49f5489fdee67b5b554cb0d8d069a464c07cdcf56b102a2c655062f0b480219f`,
  inherits the shared projectile template and registers
  `ZelemBasicHybridProjectile`.
- Ranked cooldown is `2.5/2/1.5`; ranked range is `15/18/21`; mana cost is
  zero. Damage is `7..14` at every rank with coefficient `0.05`,
  Spacetime/Energy typing, and Projectile plus Energy Damage descriptors.
- The projectile is `Ability_Fireball.Noun`, travels at ranked speed `4/6/8`
  with distance budget `50`, launches at `0.466699988s`, and releases at
  `1.600000024s`. The authored offset is `(0.25, 0, 0.2)` and the animation is
  `zlm_minn_sp_04_attack1`.
- Trail, impact, and miss assets are respectively
  `spacetime_lieuSP03_shot_effect.ServerEventDef`,
  `spacetime_hybrid_hit_effect.ServerEventDef`, and
  `ineffective_common_small.ServerEventDef`.

### Exact melee contract

- Melee chunk `813`, resource `14390`, `Abilities/0xA2043C73.lua`, SHA-256
  `9167a6d0553de7aec2e1d6e5fead31ec8a095073d0ad9928d16287fb3eb9c690`,
  inherits the shared melee template and registers `ZelemBasicHybridMelee`.
- It shares ranked cooldown `2.5/2/1.5`, authors range `1.125`,
  `shouldPursue=true`, hit arc `1.75`, hit time `0.25999999s`, and release
  `1.200000048s`. Damage is `2..6` at every rank, Spacetime/Physical, with
  Melee, Basic, and Physical Damage descriptors.
- Presentation is animation `zlm_minn_sp_04_attack2` and hit asset
  `spacetime_bite.ServerEventDef`.

The phase record fixes which action is considered for the current distance;
the absent campaign scheduler still controls evaluation cadence, rejected-
admission retry, and when a moving target crossing eight units causes the next
phase evaluation. It must not replace the strict authored threshold with a
random ranged/melee choice.

## `ZelemBasicRepair`: corpse repair, otherwise Arc Welding

### Exact phase and condition policy

- Phase package ordinal `3684`, type `0x30728CE7`, instance `0xA63F868D`,
  decoded size `156`, SHA-256
  `9922a0e258908cb4494677e3e099f33cd5feac65d513c38ab3ba1e6d12c3f00d`,
  stores two entries in authored order: `ShouldRepair -> Repair`, then the
  conditionless `ArcWeldingMelee` fallback.
- Condition chunk `241`, resource `13781`, `lua/0x6C5A08FD.lua`, SHA-256
  `0ee77ef0cfa5c8d2ea6aecf65a98199f0ee6f15ba2f3d52afff480ed581b4403`,
  authors `Radius=20` and `RepairChance=0.5`. It first rejects an actor with
  `DontRepairModifier`; otherwise it accepts only when `0.5 > math.random()`.
- On an accepted roll, `ShouldRepair` iterates the radius query in returned
  order and selects the first valid object other than self that is on the same
  team, dead, not already fading, reports creature type equal to
  `nDamageTypes.Technology`, and does not carry the `Critical` hit descriptor.
  It returns that object's current position and object ID as the phase target.
  This is neither nearest-ally selection nor a hero-target fallback.
- On a failed roll, the condition applies `DontRepairModifier` to the actor at
  its NPC rank. Modifier chunk `795`, resource `14372`,
  `Modifiers/0x6132BA4B.lua`, SHA-256
  `77048fdda3a0db902c8e939647951454776f9bcf27cae0f3f3beba4b00e700bd`,
  is unique, lasts exactly two seconds, requires no agent, and ends on agent
  death. A successful roll with no qualifying corpse returns false without
  adding that lockout.

### Exact Repair and melee execution

- Repair chunk `825`, resource `14402`, `Abilities/0xFD7B669B.lua`, SHA-256
  `6e2aa06297e33482aa7343aa8b10b51d45f805bd0d9a76d9147afb648d1ee614`,
  authors range `20`, cooldown and mana cost `0`, `requiresAgent=true`, alert
  animation `zlm_minn_tc_2_alert` for `1.166666985s`, repair animation
  `zlm_minn_tc_2_rez`, and repair time `3.233334064s`. After the alert it
  revalidates the target, follows the live object until the combined
  footprints are near enough, stops, turns to the corpse, revalidates that it
  is still dead, plays the repair animation, waits the repair time, revalidates
  again, and requests `BeingRepairedModifier` with the repairer's attribute
  snapshot.
- Being Repaired is chunk `763`, resource `14339`,
  `Modifiers/0xE40C78AE.lua`, SHA-256
  `4b92ff9fbd26f2a26df685b7a24d0e0ff85f428a35e009a66577a7fb8e7f9741`.
  Every application resets the corpse death timer to ten seconds. It stacks,
  handles `nAbilityEventFlags.StackModifier`, and completes regular minions
  after ranked stack counts `2/1/1`; other NPC types complete at maximum stack
  counts `6/4/2`. Completion heals `60/70/80%` of maximum HP with coefficient
  zero and `IsHoT`, then requests `RepairResurrected`. Incomplete stacks wait
  indefinitely for another repair application and are removed if the agent is
  destroyed.
- Arc Welding chunk `711`, resource `14283`,
  `Abilities/0xCCC7EEF8.lua`, SHA-256
  `64d399ac639816214b316fbf562fd5d0f08e7dfc320913ed4e885a93d76f4096`,
  is the ordinary fallback attack: cooldown `1.5`, range `0.75`,
  `shouldPursue=true`, hit arc `1.25`, hit at `0.533333361s`, release at
  `1.600000024s`, ranked damage `4..7` at every rank, Technology/Physical
  damage, animation `zlm_minn_tc_2_attack1`, and
  `cyber_common_hit_small.ServerEventDef`.

The missing campaign scheduler still owns think cadence, admission retries,
and which actor evaluates first. It does not own the recovered phase order,
repair chance, corpse eligibility, returned target, lockout, repair timeline,
stack thresholds, or Arc Welding fallback.

The production phase adapter now evaluates `ShouldRepair` before planning
hero pursuit, filters the retained corpse scan to same-faction nouns whose
authored creature type is Technology, and retains the exact 50-percent gate
plus two-second failed-roll lockout. Arc Welding remains the fallback when
the gate or eligible-corpse scan fails.

## `ZelemBasicPackMelee`: cower alone, attack with company

### Exact phase and ally condition

- Phase package ordinal `7032`, type `0x30728CE7`, instance `0xB8A61E77`,
  decoded size `185`, SHA-256
  `28d55038e47d8b69eff64e0aacf47ad2a81600baae8e417f6b07777430d51b1b`,
  stores `AllyNotNearby -> ZelemBasicPackMeleeCower` first and the
  conditionless `ZelemBasicPackMeleeAttack` fallback second.
- Condition chunk `924`, resource `14502`, `lua/0xBE76C5D1.lua`, SHA-256
  `18be908182e826a3134d8b3715980a6a67177232b295170e7b4a8ffb92f2abea`,
  registers both `AllyNearby` and `AllyNotNearby`. The inverse condition used
  here authors radius `20` and returns false as soon as its radius query finds
  any object other than self that is alive and on the same team; otherwise it
  returns true. It does not filter by species. The separate `AllyNearby`
  definition has `IgnoreMySpecies=false`, but that property and species branch
  are not used by `AllyNotNearby`.
- Consequently any other living teammate within 20 suppresses cowering and
  selects the attack fallback. Dead allies, hostile actors, and self do not.
  The returned condition has no target override or position; the attack keeps
  the campaign-selected hostile target.

### Exact cower and attack contracts

- Cower chunk `100`, resource `13624`, `Abilities/0x01008A3C.lua`, SHA-256
  `ff5ecc222c77caff0338920527cd00bf69ed0de9dfdc2fd63ce2c0446d19a9e4`,
  is a cosmetic, agent-required action with mana cost zero, cooldown `24` at
  all ranks, no pursuit, and no face-on-create. It plays
  `zlm_minn_sp_01_cower` and waits ranked cower time `3/2/1` seconds. It
  authors no damage or target mutation.
- Attack chunk `958`, resource `14538`, `Abilities/0x7FF542C4.lua`, SHA-256
  `b65587b9db433a3277564aad71e61a524906364f4f8b5d26f53d3272680edaef`,
  is the ordinary conditionless melee fallback. Ranked cooldown is
  `2/1.5/1`; range is `1.125`; `shouldPursue=true`; hit arc is
  `1.600000024`; hit time is `0.529999971s`; release is `1.299999952s`.
  It deals `6..9` Spacetime/Physical damage at every rank with Melee, Basic,
  and Physical Damage descriptors, animation `zlm_minn_sp_01_attack1`, and
  `spacetime_bite.ServerEventDef`.

The family identity and cower/attack decision need no further reverse
engineering. What remains unavailable is the 1-1 retail overlay or selector
that admits Pack Brawler despite its absence from the base low-band rows, plus
the campaign scheduler's condition-evaluation cadence.

## `NomadSnipe`: reveal, slow, then close for melee

### Exact noun/Lua/native evidence

- Phase resource `793` stores `EnemyNeedsDebuff -> NomadSnipe_Slow` before
  `NomadSnipe_Melee`; AI resource `789` links the phase.
- Slow chunk `160`, resource `13691`,
  `Abilities/0xBC203EB9.lua`, SHA-256
  `a59ad16204922971941d8fbd8e4d22673f98a52610311a3571137059ea5b245f`,
  is a hostile zero-damage instant cast: rank-one range `12`, hit `0.5`,
  release `1.3`, cooldown `4`, animation `nomad_lieu_sp_4_attack2`, and
  modifier `NomadSnipe_SlowDebuff`.
- Slow modifier chunk `13`, resource `13532`, authors duration `6`,
  `speedAdjustment=0.7`, and `attackSpeedAdjustment=-0.3` at rank one.
- Melee chunk `364`, resource `13912`,
  `Abilities/0x28773297.lua`, SHA-256
  `d514b6f468bd5b5da7055eb4861cee8ebc39cb5aa6c7172895d82f94a09ab3e2`,
  authors `shouldPursue=true`, range `1.5`, hit-arc length `2.5`, hit
  `0.466667`, release `1.5`, cooldown `1.5`, and rank-one physical/spacetime
  damage `5..8` with animation `nomad_lieu_sp_4_attack1`.
- The primary pre-aggro leaf is `nBehavior_Invisible`; the AI definition also
  supplies alternate `preAggroIdle2=nBehavior_Wander`. Primary first aggro and
  first alert are `FirstAggro_BeamIn`; an alternate first-aggro slot names
  `FirstAggro_FaceTarget`.
- `FirstAggro_BeamIn` chunk `572`, resource `14134`,
  `Abilities/0x67F43837.lua`, SHA-256
  `d05b6ec1c182b5c878091b3e695b5d4f9932a0e526945ae700daedfd2b8d8886`,
  sets visibility/unstealth, plays `gen_aggro_sp_beam_in`, and waits `1.23`.

### Fallback action policy

For a newly created ordinary 1-1 captain, use the primary invisible leaf and
`FirstAggro_BeamIn`; use the alternate wander/face-target pair only when a
future recovered spawn-state flag explicitly selects it. After reveal, pursue
until Slow admits at `12`, apply the debuff at `t=0.5`, then pursue until melee
admits at `1.5`. While the target retains `NomadSnipe_SlowDebuff`, skip the
first gambit and use melee. When the six-second debuff expires, Slow becomes
eligible again. Both abilities face the selected target on acceptance.

Ranged captain/lieutenant attack cycles now enter the shared bounded lateral
movement phase after their projectile sequence, even when the ability is not
one of the previously hard-coded special profiles. Navigation clipping keeps
the short left/right walk on the current island, while melee-only captains
retain their authored pursue-and-attack cycle.

`NomadSnipe_Slow` is the first hostile/offensive action but deals zero damage;
`NomadSnipe_Melee` is the first damaging action. Keeping that distinction is
important for damage-driven route and death logic.

The current server implements the primary reveal, Slow modifier creation and
deletion, target-refreshed pursuit, repeated melee damage, squad death/handoff,
and the phase return to `NomadSnipe_Slow` after the six-second modifier expires.
Its modifier handle is likewise session-owned, and stale deletion callbacks
cannot target a replacement gameplay generation. An admitted
`NomadSnipe_Melee` retains its selected hero through the hit clock; a squad
swap during windup cancels that hit rather than redirecting it to the newly
deployed hero, while the next cycle may acquire the replacement normally. Slow
creation also revalidates that retained target and the live source at its
`0.5s` hit, including the terminal and Beam Out boundaries.

## `NomadWithDrone`: activate robot, pursue, punch

### Exact noun/Lua/native evidence

- Phase resource `1473` contains only `NomadWithDronePunch`; AI resource
  `1477` links the phase.
- Punch chunk `315`, resource `13862`,
  `Abilities/0xF9F039B2.lua`, SHA-256
  `77659dbb6b0809a2825ad9b1f2011cf3c9ee73bb3ae6e403ad3265246c209424`,
  authors `shouldPursue=true`, range `2`, hit-arc length `2.75`, hit
  `0.515152`, release `1.666667`, cooldown `2.4`, and rank-one
  physical/technology damage `5..8` with animation
  `nomad_lieu_tc_3_attack`.
- Pre-aggro and combat idle both name `nBehavior_Idle`. Exact chunk `672`
  performs no activation movement and waits forever in Tick; the higher-level
  selector must replace it when an action becomes eligible.
- First aggro is `FirstAggro_ActivateRobot`: chunk `71`, resource `13594`,
  animation `zlm_minn_tc_2_aggro`, duration `1.3`, and
  `faceTargetOnCreate=false`. Subsequent aggro names
  `SubsequentAggro_ActivateRobot`: chunk `731`, resource `14303`, animation
  `zlm_minn_tc_2_reactivate`, duration `0.7`, also with target-facing disabled
  for the presentation.
- The AI definition additionally names `NomadWithDronePassive`; modifier chunk
  `301` requires `Abilities!ability_sentrydronelaser.lua`, recovered by
  registration/package identity as exact chunk `954`. That module registers
  the 0.7-second-cooldown, range-15 `SentryDroneLaser` before the passive
  creates its sole `NomadDrone.Noun`. The laser's projectile and damage
  contract are exact; the drone noun's requesting AI selector and native
  attack-speed transform remain unproven.

### Fallback action policy

Remain stationary before aggro. Play the `1.3s` activation without rotating
to the target, then pursue until Punch admits at range `2`. The accepted Punch
uses the AI definition's ordinary `faceTarget=true`, hits at `0.515152`, and
releases at `1.666667`. During the `0.733333s` release-to-cooldown remainder,
use the authored stationary idle; replace it as soon as Punch admission is
ready. Use the `0.7s` reactivation only after a genuine loss-and-regain of
aggro, not on every cooldown cycle.

Use only the recovered `SentryDroneLaser` profile for the passive-owned drone:
hit at `0.17s`, speed `20`, distance `50`, and `1-3` Technology/Energy damage
with coefficient `0.05`. Retain the passive/drone identity separately from the
captain's Punch. The compatibility selector uses the owner's retained hero
target whenever it is inside the exact range `15` and keeps the exact `0.7s`
baseline cooldown; do not guess how `NomadDroneRage` changes that cadence.

The current server creates one untargetable owner-bound `NomadDrone.Noun` when
Invincitron first reaches combat range, schedules the exact laser, projects the
owner relation for the client's orbit presentation, and deletes the drone when
its captain dies so it cannot retain the encounter. It retains the hero selected
when `NomadWithDronePunch` is admitted. A squad swap during its `0.515152s`
windup cancels the hit rather than
redirecting the committed Punch to the replacement hero; its next authored
cycle may select that replacement normally.

Runtime timing follows the authored absolute cooldown boundary. Carrion
Shambler returns to phase selection when its consume release completes while
the separate 15-second corpse-consume readiness prevents an immediate second
consume. Pouncing Stalker waits for the greater of landing recovery and its
cooldown rather than adding both intervals. Ray Killer retains the authored
50-unit piercing capacity, but its selected-target shot terminates and resolves
at that target so damage is not delayed until the maximum-range endpoint.

## Walkthrough cross-check

The recording is consistent with, but cannot numerically calibrate, the
recovered content:

- `00:02:20-00:03:00` shows small opening packs already transitioning into
  combat; the approach is not clean enough to time the invisible-to-first-
  aggro boundary.
- `frame-250.png` / about `00:04:10` explicitly labels an `Invincitron`, the
  locale-backed `NomadWithDrone` archetype. It closes into the player's melee
  space, consistent with the range-2 Punch envelope.
- Later normal arenas show repeated bright blue teleport/beam flares, ranged
  bolts, and close melee contacts. Multiple actors and player effects overlap,
  so no individual retail cooldown should be measured from the edited video.
- The walkthrough note identifies `Space Barracuda` and `Reparatron` along the
  route, but the content and locale evidence reviewed here does not map either
  display name to one of the four focused families. Do not use those names as
  family evidence.
- `frame-840.png` / about `00:14:00` labels `Illust the Accelerator` with
  `Swift Aura, Swift`. Illust is the locale-backed Haster captain identity and
  visibly fights with adds, consistent with an ally-buffing ranged family.
  The mutation aura is run-specific and must not be folded into the base
  `ZelemSpecialHaster` action definition.

The video establishes presentation plausibility and family presence. Exact
ranges, hit clocks, releases, and cooldowns come from build-103 Lua, not pixel
measurement.

## Implementation handoff

An implementation can represent each actor with the following independent
state:

```text
preAggroLeaf
firstAggroConsumed
selectedTargetID
phase = 0
activeAbility
abilityReleaseAt
cooldownReadyAt[ability]
activeModifier[asset, expiresAt]
navigationGoal
nextThinkAt
```

The transition order is:

```text
CREATED
  -> PRE_AGGRO (invisible, wander, or idle per family)
  -> TARGET_INSERTED
  -> PRE_AGGRO_DEACTIVATE
  -> FIRST_AGGRO_PRESENTATION (if authored)
  -> PHASE_SELECT
       -> OUT_OF_RANGE: PURSUE -> retry admission
       -> ACCEPTED: FACE when enabled -> WINDUP
       -> HIT_OR_LAUNCH: pay cooldown once -> consequence
       -> RELEASE
       -> cooldown ready: PHASE_SELECT
       -> cooldown running: authored/generic COMBAT_IDLE
  -> TARGET_LOST: cancel/preserve effects according to the ability template,
                  reacquire through campaign authority
  -> DEATH: cancel object-owned behavior, locomotion, and ability work
```

The following values remain server-authority choices, not recovered facts:

- difficulty-to-ability-rank mapping;
- phase-condition evaluator and first-valid ordering in the absent campaign
  scheduler, despite the authored record order;
- exact think/retry cadence and navigation goal construction;
- Zelem random-reachable angular distribution and failure retry;
- which NomadSnipe primary/alternate pre-aggro slots ordinary campaign spawns
  select;
- the retail Haster ally selector beyond the deterministic nearest-ally policy;
- the native `NomadDrone` selector, orbit trajectory, and
  `AttackSpeedScale` rage timing transform beyond the baseline compatibility
  policy;
- loss-of-target cancellation points, retarget policy, and re-aggro timing;
- packet reliability, batching, and ordering beyond the receiver dependencies
  already stated in `notes/campaign/1-1/enemy-runtime.md`.

## Evidence index

- Roster, nouns, phase IDs, and phase ability strings:
  `notes/campaign/1-1/nouns.md`.
- Spawn baselines, AI slots, blackboard, target-facing, and receiver ordering:
  `notes/campaign/1-1/enemy-runtime.md` and `notes/campaign/hostility.md`.
- Native admission, pursuit boundary, combat idle, ability concurrency, and
  template timing: `notes/tutorial/horde-ai.md`, especially build-103 functions
  `sub_9E0660`, `sub_9E1540`, `sub_9E4310`, and `sub_9E4640`.
- Canonical native source: `bin/game/GameBin/Game.c` and
  `bin/game/GameBin/Game.idb`.
- Authoritative content source:
  `bin/darkspinner/darkspin/cache/content.db`, manifest build `103`.
- Focused extracted bytecode used for this pass:
  `bin/game/logs/1-1-enemy-actions`.
- Video timeline and recording hash:
  `bin/video/walkthrough/1-1/info.md`.
