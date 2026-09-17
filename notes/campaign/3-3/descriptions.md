# Campaign 3-3 enemy-description audit

## Result

All sixteen `verdanth_2` director families match their build-103 English long
descriptions. No production change was required on this map.

| Display name | Server noun family | Result |
| --- | --- | --- |
| Charging Brute | `VerdanthBasicPicky` | Matches: normally pursues the closest hero for melee, but periodically selects the farthest live hero for its ranked charge and landing strike. |
| Caustic Stinger | `VerdanthBasicSkeet` | Matches circle-and-dart behavior; audited in 2-2. |
| Tentacler | `VerdanthBasicPlunge` | Matches its avoidable under-target thorn attack; audited in 1-2. |
| Chrono Striker | `VerdanthBasicMelee` | Matches its slowing strike and same-species-death flee; audited in 2-4. |
| Botanical Tunneler | `VerdanthSpecialOne` | Matches its tunneling emergence attack; audited in 2-1. |
| Mending Tanglid | `VerdanthSpecialTwo` | Matches healing, rooting, and ally focus; audited in 2-1. |
| Grappling Pulsar | `VerdanthSpecialThree` | Matches Puller and rapid Fast Swipe; audited in 2-1. |
| Charging Grendel | `CryosBasicCharge` | Matches: uses melee nearby and its ranked charge against distant heroes. |
| Hypno Mantis | `CryosSpecialThree` | Matches its sleeping cloud and close swipe; audited in 2-1. |
| Cannonator | `ZelemBasicHybrid` | Matches its slow ranged projectile and close melee switch; audited in 2-3. |
| Warp Spawner | `ZelemSpecialOne` | Matches area resistance and teleporting projectile; audited in 1-2. |
| Ghostly Tracker | `NoctBasicGhostCharger` | Matches high dodge and its through-target contact charge; audited in 2-4. |
| Shade Drifter | `NocturnaSpecialDrift` | Matches dodge, silence charge, and drain aura; audited in 2-2. |
| Invincitron | `NomadWithDrone` | Matches drone laser and half-health invulnerability; audited in 1-1. |
| Ragetusk | `NomadSpecialOne` | Matches knock-up charge and ally-death enrage; audited in 2-2. |
| Lightning Juggernaut | `Boomer` | Matches its close-range Energy attack; audited in 1-3. |

## Authority

Descriptions are joined directly from read-only build-103 class resources to
English localization. Reused results point to the first full audit.
