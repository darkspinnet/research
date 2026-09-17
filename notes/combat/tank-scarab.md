# Laser Tank and Arcturus scarabs (2026-09-07)

The three 0.7.34 reports captured Hami in 4-1 and 5-4. Both scarab reports repeatedly contain `encounterDamage: instanceDamageObserve: damageBoss: boss damage: unexpected actor`. Arcturus creates `CitadelBossMinon` actors with his marker set, but they are not members of the boss session's wave plans. Boss damage observation now ignores actors outside that roster; registered wave actors retain the phase/live-actor checks. This lets NPC death and detonation processing complete for scripted summons without adding them to the boss completion requirement.

Laser Tank damage already checks the placement segments, but its chained presentation targeted live hero object IDs. Laser Zone now creates stationary, cast-owned source and destination markers and attaches the beam to those markers. Cleanup hard-stops the beam and deletes its markers on completion or the next pulse after interruption/source death/target loss. Range or obstruction cancellation cleans up before repositioning. Retired casts cannot remove another cast's effects because marker IDs are unique.

Validation: source review and formatting only; no builds or tests. Real-client verification remains necessary for beam presentation and scarab explosions in Arcturus's arena.
