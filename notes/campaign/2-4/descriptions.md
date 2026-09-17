# Campaign 2-4 enemy-description audit

## Result

The base-band `zelems_4` director contains fifteen eligible families. Fourteen
already fulfilled their English long descriptions. This pass added Strafing
Drakon's missing same-species haste condition.

| Display name | Server noun family | Authored long-description behavior | Result |
| --- | --- | --- | --- |
| Strafing Drakon | `ZelemBasicPackfly` | Moves and attacks faster near members of its own species. | Matches after this pass: another living Drakon inside ten units grants a 25-percent movement increase and 25-percent shorter attack cadence. |
| Homing Striker | `ZelemBasicRangedHoming` | Fires erratically tracking projectiles that periodically home toward the closest hero and expire shortly. | Matches; audited in 1-2. |
| Warp Spawner | `ZelemSpecialOne` | Resists area attacks and teleports heroes hit by its projectiles. | Matches; audited in 1-2. |
| Magnetic Master | `ZelemSpecialTwo` | Avoids melee, pulls heroes toward allies, and emits a damaging circular push. | Matches; audited in 1-2. |
| Raytheoid | `ZelemSpecialThree` | Fires piercing lasers and greatly buffs nearby allies' Energy damage. | Matches; audited in 2-1. |
| Haster | `ZelemSpecialHaster` | Hastes a nearby ally's movement and attack speed and fires a moderate projectile. | Matches; audited in 1-1. |
| Chrono Striker | `VerdanthBasicMelee` | Tail strikes slow movement and attacks; it flees when nearby members of its species die. | Matches: its melee applies the timed movement/attack slow, while the nearby-death counter drives its navigation-clipped flee phase. |
| Ghostly Tracker | `NoctBasicGhostCharger` | Has high dodge and damages every hero touched during its charge. | Matches: noun Dodge Rating participates in avoidance and its clipped through-target charge resolves every path contact. |
| Necrodactyl | `NoctBasicFlyer` | Lobs a small-area stacking curse that deals damage over time. | Matches: its lob applies the bounded ranked poison stack and timed ticks. |
| Arachno Striker | `NocturnaSpecialHomer` | Slow homing projectiles terrify heroes, then it chases terrified victims into melee. | Matches: the projectile applies Fear and the phase selects close melee against the affected target. |
| Shade Drifter | `NocturnaSpecialDrift` | Avoids physical attacks, silences heroes with charges, and drains nearby life. | Matches; audited in 2-2. |
| Pterodyne | `nct_lieu_su_stealther` | Stealths to close distance and makes heroes vulnerable to future Physical damage. | Matches; audited in 2-2. |
| Hover-Bot | `ScaldronBasicCopter` | Connects to other Hover-Bots with damaging laser beams. | Matches: its persistent paired-beam run damages every hero or companion crossing each link. |
| Pouncing Stalker | `NomadBioSpecialTwo` | Leaps to close distance and can resurrect when killed at the wrong time. | Matches; audited in 1-4. |
| Blasting Fiend | `Shooter` | Fires slow shadowbolts and strafes between attacks. | Matches through the bounded post-shot strafe-or-idle behavior audited in 2-2. |

## Authority and remaining limits

The strings come from read-only build-103 class resources and English
localization. The native Drakon proximity radius and haste magnitude are not
recoverable, so the conservative bounded values are recorded in
`notes/help.md`.
