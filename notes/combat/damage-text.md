# Offscreen damage numbers: unresolved capture

Report `damage-numbers-are-offscreen-they-should-be-where-the-damage-darkspin-bug-0.7.30-20260906T162003.044468500Z` contains logs and traces but no screenshot or floating-text transform. The retained sparse `0xba` events include target 205/source 2 for 24 damage at 16:19:04.672 UTC and target 206/source 2 for 40 critical damage at 16:19:10.448 UTC. They have the expected positive display magnitude and negative HP mutation.

Build-103 `sub_4E2CA0` copies target/source IDs into its combat-text event and dispatches through `sub_506D90`; the combat-event packet has no standalone screen-position field. The capture does not establish whether the wrong location is the target model, a text anchor, or camera projection. No speculative positioning change was made. Capture a screenshot/video while the misplaced numbers are visible, ideally together with `/bug`, before selecting a correction.
