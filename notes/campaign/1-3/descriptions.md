# Campaign 1-3 enemy-description audit

## Result

The first-visit `nocturna_4` population presents six enemy families. Five
already fulfilled every English long-description claim. Animus resurrected
allies but did nothing between resurrection opportunities; this pass added its
authored flee-facing Ghostly Bolt phase.

| Display name | Server noun family | Authored long descriptions | Result |
| --- | --- | --- | --- |
| Draining Simian | `NocturnaBasicHealthDrain` | `0x47946faf`: “Can drain health over time if it gets close enough to its target.” `0xaa10fa75`: “Becomes exhausted for a short time after stealing a significant amount of health.” | Matches: its close-range eight-tick channel heals from committed damage, then its authored channel lifecycle releases before the next attack. |
| Necrodactyl | `NoctBasicFlyer` | `0x43412897`: “The curse deals damage over time, but has a limit on how much it can stack.” | Matches: projectile impacts apply the six-second curse, repeated hits refresh it and stack its damage only to the authored maximum of three. |
| Necrotic Leech | `NocturnaSpecialLeech` | `0xb60ee158`: “Fires long range necro projectiles.” `0x8c001b31`: “Will regain health when nearby allies are damaged.” | Matches: it uses the ranked long-range supernatural projectile and its nearby-allied-damage listener heals it from committed damage. |
| Animus | `Rezzer` | `0xc0c088b7`: “Resurrects nearby allies who have recently been killed.” `0xbebd3e93`: “In between resurrections, flees and throws ghostly projectiles.” | Matches after this pass: it revives the nearest eligible encounter corpse at 40 percent health; without one, it backs away from a close hero and fires the ranked `GhostlyBolt` projectile before checking for another corpse. The flee pursuit is an explicit secondary phase, so the primary resurrection-family guard no longer releases it as incompatible. |
| Pyro | `CitadelBasicMelee` | `0x08c6d49d`: “Has a short range fire breath attack that can hit multiple targets within its area.” | Matches: Fire Breath evaluates all heroes and targetable companions inside its 120-degree, rank-shaped cone. |
| Lightning Juggernaut | `Boomer` | `0x9bd4527a`: “Attacks with energy damage when up close.” Its packaged AI additionally selects `DeathDetonate`, and localization `0x49b778f7` says it “explodes upon death.” | Matches: footprint-aware charge pursuit hands off to Smash at close range without rejecting that secondary melee phase, and death starts `cast_explode` before the recovered 1.3-second, seven-unit elemental-energy detonation at the defeated world position. |

## Authority and remaining limits

The strings come from the read-only build-103 `NonPlayerClass` resources and
English localization table. `Rezzer.Phase` directly lists `Resurrect` and
`GhostlyBolt`; indexed chunk 25 (`Abilities/0xA8A0C4BD.lua`, SHA-256
`2a4da16acf72f5fd0320cb6c4c8ad7741fe7485e1702faaf331ce176da3bbea7`)
proves the bolt’s ranked cooldown, range and speed plus its damage, projectile,
and effects. The absent retail flee destination policy uses the existing
navigation-clipped campaign flee sampler and is recorded as a compatibility
decision.

The 0.6.0 reports captured repeated idle-action recovery immediately after
these secondary phases. The pursuit runtime had accepted the phase but then
compared its family to the noun's primary family and released it on the first
50 ms step. `Smash` and `GhostlyBoltFlee` now join the already-authorized jump
and stealth secondary pursuits. A concurrent broad report also exposed
Reparatron repeatedly failing `ResurrectionCast`: repair authors only a target
effect, so the shared presentation now treats source and target effects as
individually optional while still requiring at least one.
