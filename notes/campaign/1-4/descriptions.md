# Campaign 1-4 enemy-description audit

## Result

The authoritative low-band `nocturna_1` director presents the six families
below. Four already matched every English long-description claim. Muting
Leucopod did not strafe between shots, and Pouncing Stalker had no described
self-resurrection; both behaviors are now present in production.

| Display name | Server noun family | Authored long descriptions | Result |
| --- | --- | --- | --- |
| Muting Leucopod | `NocturnaBasicRangedSilence` | `0x8969cea7`: “Will sometimes strafe between attacks.” | Matches: its slow projectile and two-second Silence are followed by one bounded navigation-clipped lateral strafe or a short idle before the next attack. |
| Vampiric Leaper | `NoctBasicHopper` | `0x297804b4`: “Hopping area effect attack” | Matches: its authored leap lands with a four-unit area attack against every admitted hero and targetable companion. |
| Pterodyne | `nct_lieu_su_stealther` | `0xb0fb65ea`: “Will stealth in an attempt to move in close.” `0x748a8a6a`: “Melee attacks cause the hero to become vulnerable to future physical damage.” | Matches: Stealth Attack relocates into its target-relative band and chains Fear Nova; its ordinary melee applies the ranked charge-limited Physical Vulnerability modifier. |
| Carrion Shambler | `NocturnaSpecialMunch` | `0x98d5e072`: “Can consume the corpses of its allies to become larger and stronger.” `0x7d048ff6`: “Uses its large fists to smash heroes in melee range.” | Matches: it claims and consumes eligible allied corpses, gaining stacked damage and body scale, and otherwise uses its multi-target frontal melee. |
| Pathogenic Vegevore | `VerdanthBasicDiseased` | `0x31fdf928`: “Fires projectiles that inflicts a disease that can jump between nearby heroes.” | Matches: its projectile can apply the ranked disease, whose retained run spreads among nearby heroes subject to the authored immunity interval. |
| Pouncing Stalker | `NomadBioSpecialTwo` | `0xb03f65a6`: “Can close distance quickly with his leap attack.” `0x6883f0ba`: “Will resurrect if killed at the wrong time.” | Matches after this pass: it chooses the authored leap outside Swipe range and its persistent self-resurrection sign now consumes the first lethal hit, restores it at 40 percent health, and resumes combat once. |

## Authority and remaining limits

The strings come from the read-only build-103 `NonPlayerClass` resources and
English localization table. The Pouncing Stalker AI links an indefinite
`IsSelfResurrect` modifier with `self_rez_aura_effect.ServerEventDef`, but the
retail server’s fluctuating window and restored-health fraction are absent.
Darkspin therefore exposes the aura continuously and gives the family one
40-percent resurrection per spawn. Muting Leucopod uses the shared bounded
campaign strafe sampler because the packaged AI names strafe behavior without
retaining the retail destination scorer.
