# Campaign 2-2 enemy-description audit

## Result

The base-band `verdanth_3` director contains fifteen eligible families. Eleven
already fulfilled their English long descriptions. This pass added the missing
Ragetusk enrage, Necrotic Leech listener heal, Shade Drifter drain aura, and
Blasting Fiend strafing behavior.

| Display name | Server noun family | Authored long-description behavior | Result |
| --- | --- | --- | --- |
| Dread Root | `VerdanthBasicRootmob` | Melee attacks can root their victim, allowing allies to close in. | Matches: accepted attacks make the ranked root roll and publish the timed root lifecycle. |
| Menace Weed | `VerdanthBasicHealer` | Roots to channel healing near allies; otherwise fires rapid thorns. | Matches: the ally-dependent phase selects its repeated six-unit heal aura or its ranked multishot Thorn Dart. |
| Caustic Stinger | `VerdanthBasicSkeet` | Circles its target and periodically darts in for a fast sting. | Matches: navigation-clipped circle movement alternates with the recovered Darting Attack. |
| Botanical Tunneler | `VerdanthSpecialOne` | Tunnels close underground, then emerges with a poison area attack. | Matches; audited in 2-1. |
| Mending Tanglid | `VerdanthSpecialTwo` | Heals damaged allies; roots heroes and calls nearby friends to focus them. | Matches; audited in 2-1. |
| Ragetusk | `NomadSpecialOne` | Charges and knocks heroes up; grows and deals more damage when nearby allies die. | Matches after this pass: the existing charge and knock-up remain, while each nearby allied death now adds a bounded damage and body-size enrage stack. |
| Undermind | `NomadScope` | Ignites heroes, increases their Energy vulnerability, and gives allies a fiery buff. | Matches: its fiery strike applies burning and stacking Energy vulnerability and projects the allied Energy-damage buff. |
| Distracted Mongrel | `NoctBasicMeleeDog` | Fast melee attacker that is easily distracted. | Matches: its recovered fast melee cadence includes the exact one-percent distraction delay. |
| Stealth Slayer | `NocturnaBasicStealth` | Appears from stealth beside heroes, then fights in melee. | Matches: it spawns stealthed, reveals on its first accepted attack or hit, and pursues into close melee. |
| Shade Drifter | `NocturnaSpecialDrift` | Difficult to hit physically; charges through and silences heroes; drains nearby life. | Matches after this pass: noun Dodge Rating governs avoidance, the clipped through-target charge damages and silences every path contact, and each charge cycle pulses a local life-drain aura that heals the Drifter. |
| Necrotic Leech | `NocturnaSpecialLeech` | Fires long-range necro projectiles and heals when nearby allies take damage. | Matches after this pass: its projectile is unchanged and every committed nearby allied hit now heals it for a bounded fraction of that damage. |
| Pterodyne | `nct_lieu_su_stealther` | Stealths to close distance; melee causes future Physical vulnerability. | Matches: Stealth Attack closes from range and its follow-up melee applies the hit-count vulnerability. |
| Hypno Mantis | `CryosSpecialThree` | Lobs a sleeping cloud, then closes for a melee swipe. | Matches; audited in 2-1. |
| Pyro | `CitadelBasicMelee` | Short-range fire breath can hit multiple targets. | Matches; audited in 1-3. |
| Blasting Fiend | `Shooter` | Fires slow ranged projectiles and strafes sideways between attacks. | Matches: ranked Poison Spit alternates with one bounded navigation-clipped lateral strafe or a short idle before the next attack. |

## Authority and remaining limits

The strings come from read-only build-103 class resources and English
localization. The native listeners do not expose Ragetusk's radius, stack
strength/cap, Necrotic Leech's heal fraction, or Shade Drifter's aura numbers.
Darkspin therefore uses conservative bounded values recorded in `notes/help.md`
while retaining exact packaged actions, damage types, and effects wherever
those operands are recoverable.
