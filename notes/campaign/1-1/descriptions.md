# Campaign 1-1 enemy-description audit

## Result

The build-103 `NonPlayerClass` records and English locale table identify seven
enemy families encountered or visibly called out in campaign 1-1. After the
2026-08-13 parity pass, each long-description claim has a production behavior.

| Display name | Server noun family | Authored long descriptions | Result |
| --- | --- | --- | --- |
| Space Barracuda | `ZelemBasicRanged` | `0x2a0e539c`: “Will periodically bend space and teleport to a new location.” `0xe29c6795`: “Fires ranged projectiles of quantum energy.” | Matches: the seven-second cycle performs an authoritative radius-eight blink and launches the recovered spacetime/energy projectile. |
| Cannonator | `ZelemBasicHybrid` | `0x7a2e5e7e`: “Fires a large slow moving projectile when at a distance from his target.” `0xa30a2d9b`: “When engaging in close combat, will switch to close range melee attacks.” | Matches: authored phase order selects the slow projectile beyond eight units and melee at eight or nearer. |
| Reparatron | `ZelemBasicRepair` | `0x1843d636`: “Repairs nearby fallen robots, bringing them back to life if given enough time.” `0x526c9061`: “Can use its repair tools as melee weapons if given the opportunity.” | Matches: it resurrects an eligible nearby robot through the repair channel and otherwise attacks with Arc Welding melee. |
| Pack Brawler | `ZelemBasicPackMelee` | `0x4ac4f670`: “Believes in the safety of numbers and will cower if alone.” `0xc1a2d160`: “Once an ally is nearby, will work up the courage to deliver a powerful melee attack.” | Matches: it cowers without a living ally inside the authored 20-unit check and admits its ranked melee when supported. |
| Haster | `ZelemSpecialHaster` | `0xd48fccc8`: “Can bend time to haste a nearby ally's movement and attack speed.” `0x6a140cc9`: “Has a moderately damaging projectile attack.” | Matches after this pass: it deterministically selects the nearest living ally inside range 30, falls back to itself only when alone, applies the exact 20-second movement/attack/cooldown haste, then fires its recovered projectile. |
| Decelerator | `NomadSnipe` | `0xfcc7e567`: “Can instantly inflict a crippling debuff on nearby heroes, slowing their movement and attack speed.” `0x039257e3`: “Chases after slowed heroes and uses a physical melee attack.” | Matches after this pass: its six-second slow now enters the modifier inventory, reaches the client with the exact movement and attack operands, and contributes its movement operand to authoritative hero motion before the melee pursuit. |
| Invincitron | `NomadWithDrone` | `0xb5cd7f6b`: “Has an orbiting drone that will shoot lasers at heroes.” `0xc018f3e0`: “At half health, will become invulnerable for a limited time.” | Matches after this pass: one untargetable owner-bound `NomadDrone` uses the exact baseline `SentryDroneLaser` against the captain's current target and is deleted with its owner; the captain retains its strict-below-half ranked shield. |

## Authority and remaining limits

The description strings came from read-only extraction of
`bin/game/Data/AssetData_Binary.package`; the decoded diagnostic copy is under
`bin/game/logs/assetdata-unzip`. Behavioral values come from the indexed Lua
and noun/AI evidence documented in `enemy-actions.md` and `invincitron.md`.

Two selection details remain compatibility policy because the retail campaign
scheduler is absent: Haster chooses the nearest eligible ally with object-ID
tie-breaking, and the owner-bound Invincitron drone attacks its owner's current
hero target whenever that target is inside the exact 15-unit laser range. The
drone retains its baseline 0.7-second cooldown; its unrecovered rage timing
transform is not fabricated.
