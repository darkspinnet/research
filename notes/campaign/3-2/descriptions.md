# Campaign 3-2 enemy-description audit

## Result

The base-band `cryos_3` director contains fifteen eligible families. Fourteen
already fulfilled their English long descriptions. This pass added
Trioculist's missing diminishing lightning arcs.

| Display name | Server noun family | Authored long-description behavior | Result |
| --- | --- | --- | --- |
| Trioculist | `CryosBasicMelee` | Close lightning attack jumps between nearby targets with diminishing damage. | Matches after this pass: an accepted melee hit can arc twice through five-unit contacts, halving damage at each jump and never striking one target twice. |
| Electron Burster | `CryosBasicLightningRanged` | Instant lightning attack fizzles against nearby heroes. | Matches: its phase maintains the recovered minimum range, publishes the fizzle when crowded, and otherwise fires the instant beam. |
| Lightning Stalker | `CryosBasicRanged` | Long-range orbs can briefly shock their victim. | Matches; audited in 3-1. |
| Terrorsaur | `CryosSpecialTwo` | Throws a spread of fireballs that ignite heroes. | Matches: ranked spread angles each resolve independently and accepted hits apply the burning damage-over-time modifier. |
| Ray Killer | `CryosElementalSpecialThree` | Fires one piercing lightning bolt per target and flees when struck. | Matches; audited in 3-1. |
| Molten Crawler | `NomadRuption` | Hurls magma into damaging lava pools and uses melee nearby. | Matches; audited in 3-1. |
| Dimensionist | `NomadDrag` | Creates a lasting sphere that slows heroes and projectiles, then calls debris down while protected. | Matches: Slow Shield uses a removable 16-second globe, every Meteor ends with one of the packaged laugh/taunt animations before the authored 12-unit target-facing strafe, and distant targets are pursued. |
| Acid Shell | `NomadSpecialThree` | Strafes while spitting acid, resists Energy while mobile, then turtles at half health with Physical immunity and area poison. | Matches through the bounded post-shot strafing, projectile, defense, Turtle phase, and poison behavior audited in 1-2. |
| Decelerator | `NomadSnipe` | Cripples movement and attack speed, then pursues for melee. | Matches; audited in 1-1. |
| Pack Brawler | `ZelemBasicPackMelee` | Cowers alone and uses powerful melee near allies. | Matches; audited in 1-1. |
| Sting Raider | `ZelemBasicFlyingMelee` | Fast melee flier whose shifted form resists area attacks. | Matches; audited in 2-3. |
| Magnetic Master | `ZelemSpecialTwo` | Avoids melee and uses gravitational pull and circular push. | Matches; audited in 1-2. |
| Chrono Striker | `VerdanthBasicMelee` | Slows heroes with tail strikes and flees after nearby same-species deaths. | Matches; audited in 2-4. |
| Pathogenic Vegevore | `VerdanthBasicDiseased` | Projectile disease can jump among nearby heroes. | Matches; audited in 1-4. |
| Arachno Striker | `NocturnaSpecialHomer` | Terrifying homing projectile followed by close melee pursuit. | Matches; audited in 2-4. |

## Authority and remaining limits

The strings come from read-only build-103 class resources and English
localization. Trioculist's native jump radius and maximum jump count are not
exposed, so the conservative bounded chain is recorded in `notes/help.md`.
