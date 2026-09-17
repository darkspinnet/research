# Campaign 1-2 enemy-description audit

## Result

The first-visit `zelems_3` population presents six enemy families. The English
long descriptions in the build-103 `NonPlayerClass` records now agree with the
production behavior of every family.

| Display name | Server noun family | Authored long descriptions | Result |
| --- | --- | --- | --- |
| Scorpiod | `ZelemBasicMelee` | `0xf5e0af64`: “Fast moving, closes in for quick melee attacks.” | Matches: it pursues at the authored eight-unit combat speed and repeats its recovered physical melee on a 1.5-second cooldown. |
| Homing Striker | `ZelemBasicRangedHoming` | `0x7b9213ab`: “Fires projectiles that periodically home in on the closest hero.” `0x0a73d3b3`: “The projectiles will expire after a short period of time.” | Matches: its retained-target projectile begins homing after one second and is bounded by the authored 18-unit lifetime. |
| Tentacler | `VerdanthBasicPlunge` | `0xb5acd88a`: “Causes spiky thorns to emerge from underneath a hero..” `0x90c8e75b`: “Damage can be avoided by staying mobile.” | Matches: it snapshots the hero’s position, warns there after one second, and damages only actors still inside the strict two-unit radius at 2.5 seconds. |
| Warp Spawner | `ZelemSpecialOne` | `0xe7fdfb5f`: “Being shifted halfway into an alternate dimension makes them highly resistant to area effect attacks.” `0xb77188c4`: “Launches time-space warping projectiles that will randomly teleport heroes who are hit by them.” | Matches: incoming area damage is reduced by 75 percent and a successful projectile hit performs the recovered random reachable-position teleport. |
| Magnetic Master | `ZelemSpecialTwo` | `0x0caa9173`: “Support enemy who tries to stay away from close combat.” `0x563f9600`: “Mastery of gravity allows him to pull heroes towards his friends.” `0xbf74217e`: “Can send out a circular gravitational push that damages and knocks back all nearby heroes.” | Matches: it pulls a distant retained hero toward its formation and switches at five units or nearer to the seven-unit damaging push, displacing every admitted nearby hero away from itself. |
| Acid Shell | `NomadSpecialThree` | `0xe1b3afb7`: “Spits acid at the heroes while strafing between attacks.” `0xb3b8f8c0`: “Highly resistant to energy damage while mobile.” `0xde66dd0a`: “At half health, withdraws into its shell, becoming immune to physical damage and emitting area effect poison.” | Matches: its acid projectile, rank-shaped Energy Defense, bounded post-shot strafe-or-idle movement, half-health Physical-immunity Turtle phase, and ranked poison pulses are active. |

## Authority and remaining limits

The strings come from the read-only build-103 `NonPlayerClass` resources and
English localization table. Ability, passive, and movement evidence is recorded
in [enemy-actions.md](enemy-actions.md). Homing curvature and exact retail
strafe destination scoring remain documented compatibility limits, but they do not
remove any described attack, resistance, control, or shell behavior.
