# Nashira shriek and illusion deaths

The 2026-09-07 0.7.34 reports capture Hami fighting Nashira in 1-4. Shadow Panic's existing implementation was only checked by `producePlunge`, while Nashira's current Shadow Toss and Shadow Boss Swipe routes use lob and melee producers. Those routes now check Nashira's cooldown-gated panic after movement deferral and respect silence. The new entry points are restricted to Nashira, not other bosses sharing panic code.

The existing profile supplies `cast_shadow_panic`, a 400 ms hit boundary, 2533 ms release, fifteen-second cooldown, twelve-unit radius, and three-second fear. Fear now occurs at the scheduled hit boundary, checks the current source generation, and uses the shared Terrified lifecycle for nearby live targets. The next normal action follows release rather than idling for the entire cooldown.

Owned Nashira nouns identify illusions. Their shared death publication now suppresses the death-animation packet, emits the existing `shadow_boss_duplicate_effect.ServerEventDef` at the death position, and deletes the illusion after 100 ms. This reuses the known duplication visual for the requested disappearance; it is not a recovered dedicated illusion-death effect. The real boss retains `shadowboss_dead` and the normal boss completion timing.

Validation is source review, gofmt, and diff checks only. No builds or tests were run. Real-client validation is needed for the smoke presentation, shriek animation, and nearby fear.
