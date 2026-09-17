# Destructor boss-event timing, 2026-09-06

The 0.7.32 report `destructor-fights-should-trigger-the-boss-event-after-their` (2026-09-07T03:31:33Z) requests the boss event after the introduction cinematic. The capture is campaign 1-4, `nocturna_1`, difficulty four.

Campaign admission, boss follow-up, developer boss spawn, and co-op spawn projection previously emitted the active Director state alongside creation, before first-aggro presentation. Final bosses with an authored first-aggro delay now defer that publication to the shared first-action schedule's intro-completion boundary. For Nashira this is 9.6 seconds, including the two-second entrance delay. The event precedes the first combat continuation and is projected to other connected players.

The delayed event requires the original session and zone, a completed intro deadline, and a living boss. It does not require the original combat target to remain selected. Owned duplicates, Captain waves, floor-warp spawns, and bosses without an authored intro delay retain their existing behavior. Boss admission and authoritative encounter state are not postponed; only client boss-event presentation moves. An immediate scheduler-failure fallback cannot emit the event before its intro deadline.

Formatting and diff checks passed. No builds, compilation, or tests were run. Real-client and co-op verification remain outstanding; no shipped assets were modified.
