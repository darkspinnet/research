# Campaign 2-3 enemy-description audit

## Result

The base-band `zelems_2` director contains fourteen eligible families. Twelve
already fulfilled their English long descriptions. This pass corrected Sting
Raider's shifted area defense and added Laser Tank's missing Energy resistance.

| Display name | Server noun family | Authored long-description behavior | Result |
| --- | --- | --- | --- |
| Pincering Carapace | `ZelemBasicChargeup` | Uses physical melee and sometimes charges up so its next attack deals more damage and knockback. | Matches: its standard melee alternates with ranked Build Charge and a stored Discharge strike, and death removes any unused charge attachment. |
| Sting Raider | `ZelemBasicFlyingMelee` | Fast flying swarm attacker whose shifted form strongly resists area attacks. | Matches after this pass: its fast close melee remains and an area hit now activates the shifted state on the correct noun family, reducing subsequent area damage while active. |
| Cannonator | `ZelemBasicHybrid` | Fires a large slow projectile at range and switches to melee nearby. | Matches: its distance phase selects the recovered projectile or close melee profile. |
| Warp Spawner | `ZelemSpecialOne` | Resists area attacks; projectile hits randomly teleport heroes. | Matches; audited in 1-2. |
| Magnetic Master | `ZelemSpecialTwo` | Avoids close combat, pulls heroes toward allies, and emits a damaging circular push nearby. | Matches; audited in 1-2. |
| Raytheoid | `ZelemSpecialThree` | Piercing lasers pass through multiple heroes and it buffs nearby allies' Energy damage. | Matches; audited in 2-1. |
| Decelerator | `NomadSnipe` | Instantly slows nearby heroes, then chases and attacks them in melee. | Matches; audited in 1-1. |
| Undermind | `NomadScope` | Ignites heroes, increases their Energy vulnerability, and gives allies a fiery buff. | Matches; audited in 2-2. |
| Robo-bomber | `CitadelSpecificThree` | Rolls grenades toward a target point where they explode in an area. | Matches: the rolling retained grenade detonates at its destination and damages every admitted area target. |
| Fragbot Mech | `CitadelBasicGunner` | Highly resists Energy damage and uses rapid-fire machine guns. | Matches; audited in 2-1. |
| Dynosphere | `CitadelBasicShield` | Uses melee behind a periodically recharging absorption shield. | Matches; audited in 2-1. |
| Laser Tank | `CitadelSpecialThree` | Highly resists Energy damage and fires up to three sustained laser beams. | Matches after this pass: it now publishes rank-shaped Energy Defense and its recovered cone zone sustains up to three one-second target pulses. |
| Vampiric Leaper | `NoctBasicHopper` | Uses a hopping area attack. | Matches: its leap lands with the recovered radial supernatural damage. |
| Shade Drifter | `NocturnaSpecialDrift` | Avoids physical attacks, charges through and silences heroes, and drains nearby life. | Matches; audited in 2-2. |

## Authority and remaining limits

The strings come from read-only build-103 class resources and English
localization. Sting Raider's native shifted-form duration is unavailable, so
the existing four-second bounded lifecycle remains visible in `notes/help.md`.
Laser Tank uses the same conservative rank-shaped Energy Defense recovered for
the explicitly resistant Fragbot family.
