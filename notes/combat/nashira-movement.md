# Nashira stationary report, 2026-09-06

Unresolved: `nashira-is-like-stuck-on-something-and-cant-move` (0.7.32, 2026-09-07T03:31:01Z), campaign 1-4 `nocturna_1`, difficulty four.

The report captures the hero at (-228.588, 424.164, 45.535), near Nashira object 290 at (-231.685, 428.868, approximately 45.65). Their centre separation is about 5.63 units. `campaignNPCActionProfile` selects Shadow Boss Swipe within six units of surface distance, including actor footprints. At this distance it intentionally selects melee instead of pursuit.

The launcher trace continues emitting object 290's stop/face locomotion packets at 23:30:45.637, 23:30:50.637, 23:30:55.637, and 23:31:00.638 (report-local offset -04:00). That matches the base Swipe's five-second cooldown and does not establish a stalled pathfinder. No obstacle-removal, teleport, range reduction, or movement-policy change is justified by this capture alone.

The visible symptom could still be a presentation issue or a failure to pursue outside attack range; neither is confirmed here. Keep the extracted report at `bin/game/logs/bugs/nashira-movement-0732`. A reproduction moving outside ShadowToss range and showing whether Nashira advances would distinguish stationary in-range combat from failed pursuit. No production changes, builds, or tests were made for this report.
