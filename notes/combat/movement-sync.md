# Movement and attack synchronization review

## 2026-09-04 findings and implementation

This review covers shared hero movement, NPC pursuit, straight projectiles,
and the toss path used by Jinx's Skull Bomb. It establishes code-level causes
and client/content contracts; it does not establish that all visible desyncs
have been eliminated in live play.

The supplied 0.7.24 report identifies Jinx Alpha in campaign 1-2 and describes
Skull Bomb passing over enemies and a temporary camera lock. Its retained
minute, 2026-09-03 03:36:59-03:37:59 UTC, contains background traffic rather
than the triggering attack. That capture cannot prove the exact timing or
positions of the reported miss. The findings below are independently visible
in the production code and canonical content.

| Cause | Consequence | Implemented correction |
| --- | --- | --- |
| Hero admission checked navmesh reachability, but `LinearMovement` advanced directly to the final goal. | Server combat origins and target positions could cut across an obstacle while the client walked around it. | `Motion` retains a route through the selected navigation layer, advances its segments on the existing monotonic clock, and rebuilds the remaining route after an accepted pose observation. Failed route construction preserves the prior motion state. |
| Navigation routes visited every portal midpoint. | Server routes could zigzag and take longer than a straight traversable corridor. | Remove unnecessary turns only when the proposed segment crosses every intervening portal in order; retain crossing elevations to follow the mesh surface. This is corridor simplification, not proof of the client's exact route solver. |
| NPC pursuit recursively scheduled 50 ms callbacks and always integrated exactly 50 ms. | Callback execution and lock delays accumulated as permanent server movement lag. | Carry the previous integration time through the pursuit generation and integrate actual elapsed monotonic time. Suspended steps reset that boundary; large steps stop at attack range. Existing goal-update publication remains intact. |
| Ordinary hero projectiles used the client's target point plus an added height while collision used the server NPC's geometry. | Launch aim and hit authority described different positions. | Derive selected-target aim from the same server position and geometry used by collision; retain the client observation in trajectory diagnostics. |
| Externally driven straight-projectile misses discarded the supplied collision endpoint. | A miss effect could jump to full range even though the scheduled collision resolved earlier. | Preserve the externally resolved endpoint; internally scheduled range-expiry behavior is unchanged. |
| Skull Bomb's landing moved the damage/effect center to the target's latest position, but its published lob had a fixed destination. | An invisible impact followed an enemy independently of the visible skull. | Resolve damage and presentation at the same retained destination sent in the lob packet. Use the target ground origin specified by the toss template. |
| Toss landing deleted its run before the independent casting release. | A sufficiently short flight skipped the release response and animation reset. | Retain the run until both landing and cast release finish; either ordering is valid, and duplicate landing callbacks cannot damage again. |
| Toss offsets were added to ground position, and far flight height/speed switched abruptly at close range. | Muzzle elevation and arc timing differed from the script. | Apply muzzle offsets from the established actor-center estimate and interpolate flight operands using the recovered distance formula, independently of the casting animation's range selection. |

Movement acceptance diagnostics now distinguish authoritative position/goal
from the reported position/requested goal. Toss trajectory records include
launch, retained destination, flight duration, release duration, target position
at impact, hit count, and whether cast release had already occurred.

## Canonical evidence

- `Game.c`, `sub_4DED90` and `sub_4D53C0`: action messages copy the
  actor's position/orientation snapshot. Ability target/cursor coordinates are
  separate fields; a target observation is not a server transform update.
- `Game.c`, `sub_A00E60 -> sub_9D6D40`: `GetCenterPoint` returns object
  position plus half the scaled noun height along world up. The existing
  footprint/bounds estimate is still used where exact noun-height metadata is
  unavailable; it must not be described as an exact recovered muzzle hardpoint.
- Runtime `content.db`, Lua chunk 630, `Abilities/0xCDD518BA.lua`, SHA-256
  `c75b68fab1bd56c399894277954544a697d308b121d551b2e31ee5871b86259f`:
  casting prototype 0.6 reads the actor center, waits for `timetohit`, applies
  transformed muzzle offsets, reads `GetPosition` for a valid target, and
  passes fixed XYZ to `TickParabolicProjectile`. The projectile continuation
  receives those coordinates through `WaitForLobbedProjectile`.
- The same chunk's prototype 0.5 computes flight distance independently and
  interpolates close/far speed, height, and restitution with
  `(distance - closeRange) / (farRange - closeRange)`, using positive
  `bounceRange` in place of range. Prototype 0.0 is the unclamped linear
  interpolation helper. Casting release remains independent of that thread.
- Runtime chunk 635, `Abilities/0x493651EB.lua`, SHA-256
  `6c2d764632bb94d3f9928d6f924ec084b05f42cf9b395d40a0cff6fa1ae3a3a6`:
  Skull Bomb has 0.18-second windup, 0.5-second release, near/far heights
  0.1/3, speeds 20/30, close range 8, and at most one affected target.
- `Game.c`, `sub_A0FBE0 -> sub_A08A50`: native straight-projectile waits
  retain remaining range, subtract traveled displacement, and resume on
  collision or range expiry. They are not an instruction to damage the selected
  target at a precomputed arrival time regardless of the intervening path.

No shipped client binary or asset was modified, and no gameplay Fang hook was
introduced.

## Second pass: flight sampling and reachable corrections

Ordinary non-homing hero projectile basics now use `sim.ProjectileFlight` on
16 ms scheduled updates. Each update sweeps only the displacement since its
preceding sample against published living hostile/fixture bounds, selects the
first contact deterministically, and retains that result. The projectile can
strike an intervening eligible object; damage and presentation bind to the
object actually struck. Misses continue to the same authored range sent in
projectile locomotion. Native range exhaustion still takes precedence on the
update that consumes the remaining range.

The flight and the adapter's motion snapshot share the accepted cast epoch.
Delayed callbacks catch up to elapsed time once rather than replaying historical
segments against a later target pose. Cast release continues after early impact,
and a resolved shot is removed only after its release boundary. A failed
multi-shot schedule also cleans up surviving shots after a sibling has finished.
Impact uses accepted damage and source position without a second range check
from the caster's new position. Damage-per-speed bonuses use actual contact
distance and acceleration rather than the original selected-target deadline.

Skull Bomb and the shared toss path refresh a living target's ground position
after windup. The resulting fixed destination, arc, launch timestamp, and
landing deadline are scheduled together. Landing has its own cancellable
schedule; session cleanup cancels both cast and landing work. The accepted
animation, muzzle offset, damage snapshot, and release remain independent.

Hero reconciliation now measures the navigation route to the reported pose,
including projection distance, against the existing correction allowance.
Accepted corrections use the reachable projected surface. Unreachable or
over-budget observations retain the advanced server pose and its route.
This prevents the old straight-distance allowance from accepting disconnected
positions or wall shortcuts whose reachable route exceeds that allowance.

## Collision and enemy projectile follow-up

Ordinary non-homing, non-piercing enemy shots, including supported stun, silence, and
damage-over-time modifiers, now use
the same per-update collision owner as ordinary hero shots. They sweep current
living hero and companion bounds, choose the earliest intersected object, and
continue to their published range when they miss the selected target. Candidate
hero poses advance to one shared sample time. Imported noun bounds determine
candidate geometry and initial aim where available; missing noun geometry keeps
the existing fallback and is reported in diagnostics.

Enemy launch authorization is retained by a zone-issued, single-hit operation.
Windup cancellation still prevents launch, but a subsequent NPC action, target
change, or stun does not invalidate an already launched shot. Existing source
death, connection, and zone cleanup still retire it. Projectile deletion no
longer cancels the independent next-action schedule on these sampled paths.
Enemy collision polling keeps one recurring callback per live shot instead of
prequeuing every physics update for its full authored range.

Both sampled hero and enemy flights now consume travel segments retained by
the projectile motion owner. Freeze, speed changes, and gravity can advance or
change that owner between collision callbacks without losing a segment or
replacing a bent path with a straight chord. Frozen and slowed flights keep
polling beyond their original deadline. Gravity replacement motion uses the
constant speed represented by its native packet; remaining-flight estimates
include acceleration. This is still current-pose sampling, not rewind or target
motion reconstruction through a delayed update.

StalkerShock and admitted damage-over-time projectile profiles select their
modifier once after accepted contact, using the struck object rather than the
originally selected target. Existing chance, surviving-target, shield, and
modifier admission rules remain in place. Debuff immunity is checked against
the struck hero's session, including another player in the same zone. Modifier
creation runs after releasing the collision lock; recurring samples and cast
release cannot apply the same status twice.

Enemy stun and sleep now stop the hero's current server route and publish the
stopped pose when the control effect is applied. Previously these effects set
input-blocking state but allowed the already accepted movement route to keep
advancing during subsequent collision queries.

Ranged silence now uses continuous collision too. Its retained state belongs to
the struck hero's peer session, so immunity, ability blocking, cleanse, and
cleanup use the affected player. Expiry runs on the shared timer and publishes
deletion to active members of the same zone independently of the firing NPC
owner's transport. Refresh preserves the longer retained expiry and uses that
duration in native presentation; rollback cannot overwrite a later independent
silence extension. Creation state is recorded under the registry lock so a
concurrent refresh cannot skip native modifier cleanup.

Homing remains separate. The canonical `sub_A2EBA0`
initializer at `Game.c:1460032` computes startup velocity from source and
target centers and initializes additional packed travel state; its steering
update must be recovered before replacing the server's terminal range-based hit
shortcut with an equivalent trajectory.

## Remaining work, in priority order

1. **Extend continuous projectile contact across the zone.** Bursts, other status
   shots, companions, Electron Sphere, homing basics, and special NPC shots still query a selected
   target at one impact deadline. A moving target can cross the ray earlier,
   leave before the query, or enter a ray segment the projectile already passed.
   Extend the flight state to these adapters while preserving terrain, piercing,
   homing, gravity, freezing, and acceleration behavior. The new straight-shot
   sweep samples current bounds; motion entirely between samples and target
   movement during a long server stall still need shared pose history.
2. **Exact toss destination constraints and lob contact.** Launch-time target
   refresh is implemented. Recover the template's full close/far destination
   constraints, transformed muzzle orientation, and projectile-center geometry.
   Recover early creature/terrain lob contact and
   bounce completion instead of treating nominal duration as complete physics.
   Retain the native distinction between fixed lobs and homing projectiles.
3. **Common pose time for all combat queries.** NPC positions remain stepped
   snapshots; the elapsed-time fix removes accumulating lag, not all sub-tick
   error. Area, melee, projectile, pursuit, companion, and PvP queries should
   observe a zone-owned pose at the same simulation instant, including stops,
   status-speed transitions, knockback, teleport, switch, and death boundaries.
4. **Exact navigation and collision metadata.** Compare the simplified server
   corridor with the client's corner selection, local avoidance, turn speed,
   acceleration, noun height/scale, and transformed collision bounds. Current
   noun-bound and muzzle fallbacks remain approximations. Shared mesh reachability
   alone does not prove identical routes or surfaces.
5. **Timestamped reconciliation and transport policy.** The current bounded
   pose correction is a distance envelope, not latency compensation. Recover
   command ordering/time semantics before adding history-based admission. Measure
   queue delay and reliable-ordered blocking before changing delivery modes or
   coalescing packets; stop, teleport, and cast boundaries must remain ordered.
6. **Capture the divergence when it happens.** Prefer Sync Snapshot with both
   server state and client locomotion probes over a report submitted after the
   recent-log window expires. Record generation, pose sample time, next path
   corner, speed, target, and active flight. The automatic NPC drift monitor's
   object-ID-only matching across sessions also needs zone/client identity
   scoping before it is trusted with concurrent unrelated sessions.

## Validation boundary

An earlier snapshot passed Go compilation for `./server/...` and `mage build`.
The subsequent collision, toss, and reconciliation edits, including this enemy
projectile follow-up, have not been built: the user explicitly stopped builds.
Only source review, Go formatting, and whitespace checks were performed for
this follow-up. Automated tests were not created, modified, or run, as required
by repository policy. The earlier build result
does not substitute for a real-client confirmation of Skull Bomb at close and
far range, movement around obstacles, moving targets, and delayed delivery.
The original report remains available for that follow-up because its camera
and hit symptoms have not yet been reproduced with the new binaries.
