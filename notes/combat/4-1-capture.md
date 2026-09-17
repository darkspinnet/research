# September 7 4-1 bundle

`4-1 bugs.tar.gz` is actually a RAR5 archive. Native 7-Zip lists two explicit bug reports and eleven automatic snapshots. All eleven analysis reports and server-state summaries were inspected. Early snapshots concern `verdanth_4` (3-4); later snapshots concern `infinity_2` (4-1).

- Power display: the explicit report records Blitz at 10.2/126 power while the player reports a full bar. This remains unresolved; server-side resource authority alone does not establish the rendered HUD value.
- No incoming damage: the 4-1 client trace records 61 `0x0040` Immune events and three `0x0080` Resist events with target object 1. This is no longer silent feedback; the underlying immunity/full-mitigation source still needs identification. Snapshot hero ownership and peer generation match the current session. Equipped defense operands are absent from these snapshots; no arbitrary defense nerf was made.
- Movement: snapshots 1–3 identify VerdanthBasicOoze divergence, 4 VerdanthSpecialTwo (129.78 units), 5 Verdanth_Boss_Spawn, 6 NomadScope, 7/9 CitadelSpecificThree-family actors, 8 CryosBasicCharge (28.10 units), 10 CitadelBasicSuicide, and 11 CitadelSpecialThree (297.90 units). Automated classifications are leads, not proof of their suggested cause.
- Confirmed pursuit failure: at 08:29:12.805 local log time, a scheduled player-pursuit callback fails with `campaignPursuitAdvance: movementAdvance: motionAdvance: movement time regressed`. It sampled the clock before waiting for the registry lock; another producer could advance the shared motion first. The callback now samples time after acquiring that lock. This does not establish that every NPC divergence shares this cause.

No builds or tests were run. Remaining resource and NPC-presentation issues require further investigation; they are not marked fixed by the pursuit timing correction.

## Full logs archive follow-up

Native 7-Zip identifies `logs.tar.gz` as RAR5, with 289 files and 29 snapshot sets: twelve from August 31, five from September 6, and twelve from September 7. All 29 analysis summaries were inspected; the newest manifests identify build 0.7.32. This is not evidence of a regression after the current 0.7.34 pursuit-time fix. Detailed per-packet traces were not exhaustively reviewed in this pass.

- The September 7 snapshots overlap the earlier bundle's eleven captures and add SS-000012 at 06:33:37 UTC in `infinity_3`: three of thirteen compared objects drift, with NomadShielder object 41 triggering the capture at 9.07 units. The issue is not restricted to `infinity_2`.
- In `infinity_2`, SS-000011 has 93 outliers among 110 compared objects. Object 89 remains at server position (-1212.298, 17.803, 12.601), while its client root is (-914.882, 34.727, 10.088) and its client pursuit goal is (-223.862, -17.252, 105.575). The main log shows pursuit retargeting at 08:27:44.734 local time from the old area toward a player position near (-111.534, 19.029, 90.575). Cross-area pursuit and differing navigation progress remain investigation leads, not a proven common cause or authorization to teleport enemies through terrain.
- The sampled sessions have no pending output overflow. Snapshot sequence gaps explicitly lack corroborating NACK/retransmit evidence; unmatched payload counts cannot establish packet loss, especially with the client ring reporting capacity drops. SS-000012 reports 1,705,355 cumulative dropped client lines.
- The main log repeats the already-fixed 08:29:12.805 pursuit-time failure. No additional gameplay change was justified by this summary/source review. Retain the extracted batch while movement and resource presentation remain unresolved; do not count the prior pursuit fix as resolving these NPC-position findings.
