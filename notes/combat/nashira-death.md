# Nashira death state, 2026-09-06

The 0.7.32 report `nashira-doesnt-do-her-death-animation-when-she-dies` (2026-09-07T03:32:50Z) describes missing death animation in campaign 1-4, `nocturna_1`, difficulty four. The trace identifies Nashira as `ShadowBoss.Noun`, object 290.

The death override passed `nct_boss_su_shadowboss_death`, a raw clip name, to `SetAnimationState`. Read-only inspection of `AssetData_Binary.package` resource 9903 proves the noun selects `ShadowBoss1.CharacterAnimation`; resource 9861 contains the state `shadowboss_dead`. The base `ShadowBoss.CharacterAnimation` resource 9900 contains the same state. Neither inspected character-animation resource exposes the raw clip name as a state.

The Nashira family override now selects `shadowboss_dead`. The 227-frame clip duration and rounded eight-second corpse/final-boss presentation window are unchanged. Damage, critical-hit behavior, rewards, and encounter completion rules are unchanged. This corrects the raw-clip naming recorded in older campaign notes.

Formatting and diff checks passed. No builds, compilation, or tests were run; real-client playback remains unverified. Shipped content was read only.
