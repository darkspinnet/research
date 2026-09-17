# Projectile client/server synchronization audit

## Result

The projectile paths had several independent sources of visible disagreement:

- travel time was measured from the actor center although `NewProjectileRun`
  launches the visual projectile one actor footprint toward its aim;
- hero basics, targeted bursts, status projectiles, and the Field Medic Sentry
  Drone committed a hit against the target's live server position even when the
  non-homing visual projectile followed its original straight ray elsewhere;
- several paths aimed at an NPC's ground origin rather than the center of its
  authored noun bounds;
- enemy projectile launch and impact sometimes used a hero pose that had not
  been advanced to the current server deadline;
- gravity-deflected enemy projectiles applied the source actor footprint a
  second time to a projectile position that was already in flight;
- enemy damage presentation discarded the collision contact and replaced it
  with the target origin;
- consume-corpse movement committed its final server position without first
  publishing that exact transform to the client.

The corrected paths derive travel time from the same footprint-offset launch
position sent to the client, retain each non-homing ray, query the target's
current authored collision box at impact, resolve misses at the visual endpoint,
and resolve hits at the first contact. Enemy homing and gravity branches retain
their distinct trajectories but now use consistent endpoints and live poses.

Structured `RakNet projectile trajectory launched` and `resolved` log records
include projectile/source/target IDs, ability, origin, launch, aim, live target,
contact or endpoint, travel distance, delay, and drift where available. Hero
basic launch records also preserve the client-reported target and its difference
from the server aim. These records are captured by the existing recent-log bug
report window.

## Remaining accuracy work

The 2026-09-04 follow-up in [movement-sync.md](movement-sync.md) corrects
hero target-observation aiming, externally resolved miss endpoints, navigation
integration, NPC pursuit clock drift, and Skull Bomb's independent lob/cast
lifecycle. It also records the remaining synchronization work in priority order.

Ordinary non-homing hero basics now sweep each flight update through eligible
zone objects with `sim.ProjectileFlight`; the earliest contact determines both
damage and presentation, and misses continue to the published range. The shared
toss path also refreshes its fixed launch destination after windup and schedules
the matching landing time. See the second-pass details in `movement-sync.md`.
Ordinary enemy shots without homing or piercing, including supported stun, silence, and
damage-over-time modifiers, also sweep
living heroes and companions throughout their full published range. Collision
and aim use each target's noun bounds when available. Launch authorization
survives subsequent NPC actions and target changes, while source death and zone
cleanup retain their existing retirement policy. Sampled hero and enemy shots
consume the motion owner's retained travel segments, including freeze, slow,
and gravity changes, and continue polling when controls extend their lifetime.

Accepted status-shot contact retains the actual struck object for modifier
application and preserves the existing chance and damage-eligibility rules.
Ranged silence retains state on the struck hero's session and expires through
the shared timer, keeping multiplayer ability blocking, cleansing, and modifier
deletion consistent when the NPC owner is a different player.
Enemy stun and sleep also stop the active server movement route immediately,
so later collision queries observe the same stopped pose as the control effect.

Bursts, other status shots, companions, Electron Sphere, homing basics, and special NPC
shots retain their separate terminal queries. Migrating those paths must retain
their control, terrain, and piercing semantics. Current per-update bounds do
not reconstruct motion entirely between updates or through a long server stall.

The 0.7.26 Jinx report captured `campaignTossSchedule.produceLaunch` holding the registry write lock while `scheduledProducers` attempted to read-lock it through `identity`; the peer watchdog disconnected the player after five seconds. Toss launch now builds the producer identity from the already-locked session and uses `scheduledProducersForIdentity`, preserving stale-session guards without recursive locking.

NPC pursuit currently publishes the initial locomotion command and subsequent
goal changes through the server's reliable-ordered application path. Under
packet loss, head-of-line blocking can delay those goals relative to server
simulation. The original outer reliability for build-103 locomotion is
not yet recovered, so this audit does not speculatively change delivery policy.
A future transport change should carry per-packet reliability metadata and
coalesce superseded locomotion corrections only after a retail capture or live
compatibility proof establishes the correct ordering behavior.
