# Invincitron orbit report, 2026-09-07

The 0.7.34 report from 3-1 asks for the hover drone to circle Invincitron. The existing follow producer only copied the owner's exact position every 250 ms, so it never generated an orbit. The current spawn adapter also deliberately does not project this drone through the owner-attached create branch; older notes claiming a native client orbit were not supported by that wiring.

The server now spawns and updates the drone on a circular path centered on the owner's latest position. Its NPC position and client position share that path, allowing the existing projectile code to use the moving firing origin. Radius, height, and revolution period are conservative presentation choices documented in notes/help.md, not recovered retail constants. Existing source/session validity checks stop the follow loop when retired or defeated.

Both supplied manual snapshots were inspected. SS-000001 compared eight mapped objects with no drift; SS-000002 compared two with no drift and flagged one SentryDroneLaser projectile without a network-associated client object. Neither reported NACKs, retransmits, or pending output. Missing byte matches and sparse object mapping do not establish packet loss or a projectile root cause, so no speculative transport/projectile change was made.

No builds or tests run. Real-client validation of orbit presentation and the separate missing-projectile observation remains necessary.
