# Campaign 2-1 enemy-description audit

## Result

The base-band `verdanth_1` director contains fourteen eligible families.
Eleven already fulfilled their English long descriptions. This pass added the
missing Dynosphere shield, Fragbot Mech Energy Defense, and Mending Tanglid
ally-focus behavior.

| Display name | Server noun family | Authored long-description behavior | Result |
| --- | --- | --- | --- |
| Swarming Herbipod | `VerdanthBasicRanged` | Stationary until threatened; launches slow, relentlessly homing attacks. | Matches: zero locomotion keeps the turret stationary, its aggro presentation activates it, and its bounded projectile uses delayed retained-target homing with terminal cleanup even when the target becomes invalid or takes no damage. |
| Pathogenic Vegevore | `VerdanthBasicDiseased` | Disease projectile can jump between nearby heroes. | Matches; audited in 1-4. |
| Botanical Tunneler | `VerdanthSpecialOne` | Tunnels close underground, then emerges with a poison area attack. | The server now separates a stationary 1.5-second hidden/intangible phase from `burrow_attack1` emergence and its hit 1.3 seconds later. Eight-unit radial hits apply poison. Burrow travel selection and exact poison tuning remain provisional; see `notes/help.md`. |
| Mending Tanglid | `VerdanthSpecialTwo` | Heals badly hurt allies; roots heroes, calls nearby friends to attack them, and keeps away from heroes. | Matches after this pass: its strict-below-half support branch heals first, its repeated close-range retreat keeps separation, and Entangle applies the ranked root and redirects all living allies inside 20 units to that victim. |
| Grappling Pulsar | `VerdanthSpecialThree` | Pull victims are slowed, then struck with rapid melee attacks. | Matches: the three-second Pulled modifier owns the forced-movement interval, then the phase uses the recovered nine-hit Fast Swipe while Puller cools. |
| Toxiraptor | `CryosBasicPoison` | Connected melee attacks can inject poison. | Matches: every accepted surviving hit rolls the ranked chance and applies or refreshes the authored damage-over-time modifier. |
| Hypno Mantis | `CryosSpecialThree` | Lobs a small sleeping cloud, then closes on sleeping heroes for a melee swipe. | Matches: Sleep Mushroom creates the three-tick radius-two cloud and close targets switch to Rez Melee. |
| Dynosphere | `CitadelBasicShield` | Straightforward melee protected by a periodically recharging absorption shield. | Matches after this pass: its frontal melee is unchanged and its visible rank-shaped absorption shield recharges after 12/10/8 seconds. |
| Fragbot Mech | `CitadelBasicGunner` | Highly resistant to energy damage; attacks with rapid-fire machine guns. | Matches after this pass: it publishes rank-shaped 250/500/750 Energy Defense and fires its recovered five-shot burst. |
| Reconstructionist | `CitadelSpecialFour` | Ground slam knocks nearby heroes upward; when damaged, repairs itself. | Matches: its phase chooses ranked self-repair while wounded, otherwise radial Ground Slam or close Piston Punch. |
| Reparatron | `ZelemBasicRepair` | Repairs fallen robots and can use its tools as melee weapons. | Matches; audited in 1-1. |
| Pack Brawler | `ZelemBasicPackMelee` | Cowers alone and uses powerful melee when an ally is nearby. | Matches; audited in 1-1. |
| Raytheoid | `ZelemSpecialThree` | Piercing lasers cross multiple heroes; buffs nearby allies’ energy damage. | Matches: the piercing path damages each contact once and resolves at the selected target, while the support branch applies one ranked Energy Damage buff per 30-second modifier cycle. |
| Lightning Juggernaut | `Boomer` | Uses energy damage at close range. | Matches; audited in 1-3. |

## Authority and remaining limits

The strings come from read-only build-103 class resources and English
localization. Swarming Herbipod’s numeric projectile body is absent from the
packaged Lua, so its existing bounded homing values remain the documented
native-behavior fallback. Dynosphere’s native shield amount is likewise absent;
Darkspin uses 10/20/30 absorption with the established 12/10/8-second ranked
recharge cadence while preserving the authored shield effects.
