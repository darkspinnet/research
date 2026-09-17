# Campaign 3-1 enemy-description audit

## Result

The base-band `cryos_4` director contains fifteen eligible families. Fourteen
already fulfilled their English long descriptions. Ray Killer's piercing bolt
existed, but its per-target volley and struck-flee behavior did not; both are
now implemented.

| Display name | Server noun family | Authored long-description behavior | Result |
| --- | --- | --- | --- |
| Pyrachnid | `CryosBasicFiery` | A fire-shrouded melee enemy that ignites heroes on contact. | Matches: every surviving contact applies or refreshes its ranked burning damage-over-time stack. |
| Lightning Stalker | `CryosBasicRanged` | Slow-moving long-range orb shooter whose hits can briefly shock. | Matches: its ranked projectile carries the authored shock chance and timed stun modifier. |
| Ray Killer | `CryosElementalSpecialThree` | Fires one fast horizontal piercing bolt per target in range and flees when struck. | Matches after this pass: a cast now targets every live hero or companion in range with its own piercing bolt, and an accepted incoming hit queues a navigation-clipped flee before the next volley. |
| Quadrakiller | `CryosSpecialOne` | Fires lightning-orb volleys and becomes exhausted briefly afterward. | Matches: its recovered four-shot burst retains the long release/exhaustion interval before the next action. |
| Molten Crawler | `NomadRuption` | Hurls magma at nearby heroes to create damaging lava pools and uses melee close up. | Matches: its phase selects per-target retained lava pools or the recovered melee fallback. |
| Invincitron | `NomadWithDrone` | Orbiting drone fires lasers; owner becomes temporarily invulnerable at half health. | Matches; audited in 1-1. |
| Shielded Grenadier | `NomadShielder` | Frontal shield blocks damage; lobs cluster grenades and knocks back close frontal heroes. | Matches: directional shield admission, cluster submunitions, and the close bash all use their recovered phases. |
| Vampiric Leaper | `NoctBasicHopper` | Uses a hopping area attack. | Matches; audited in 2-3. |
| Arachno Striker | `NocturnaSpecialHomer` | Terrifying slow homing projectile followed by melee pursuit. | Matches; audited in 2-4. |
| Pyro | `CitadelBasicMelee` | Short fire breath can hit multiple nearby targets. | Matches; audited in 1-3. |
| Laser Unit | `CitadelBasicRanged` | Charges up, then fires a piercing laser bolt. | Matches: its ranked wind-up leads into a range-extended piercing projectile. |
| Fragbot Mech | `CitadelBasicGunner` | Highly resists Energy damage and fires rapid machine-gun bursts. | Matches; audited in 2-1. |
| Reconstructionist | `CitadelSpecialFour` | Ground slam knocks nearby heroes upward; repairs itself when damaged. | Matches; audited in 2-1. |
| Reparatron | `ZelemBasicRepair` | Repairs fallen robots and uses its tools for melee. | Matches; audited in 1-1. |
| Grappling Pulsar | `VerdanthSpecialThree` | Slows pulled heroes and follows a successful pull with rapid melee attacks. | Matches: Pull Modifier retains the three-second slowed presentation and forced movement, then the phase selects the recovered nine-hit Fast Swipe. |

## Authority and remaining limits

The strings come from read-only build-103 class resources and English
localization. Ray Killer's native flee destination scorer is unavailable, so
it reuses the established bounded navigation-clipped flee sampler recorded in
`notes/help.md`.
