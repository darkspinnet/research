# 1-1 walkthrough observations

## Evidence boundary

These observations come from the walkthrough footage under
`bin/video/walkthrough/1-1`. They are suitable for reconstructing a playable
route and presentation fallback, but edited video does not prove exact spawn
timestamps, director budgets, random-selection weights, or universal loot
probabilities. Counts below are visual counts and should be cross-checked
against frames before treating them as exact fixtures.

## Population before the first security teleporter

The visible route contains substantially more population than the current
single-Barracuda opening fallback:

1. First cluster: one Reparatron, four Cannonators, and one Invincitron.
2. Nearby cluster to the north: three Cannonators and one Reparatron.
3. Cluster near the obelisk behind the first Invincitron: three Reparatrons
   and two Cannonators.
4. Later pre-teleporter cluster: at least one Invincitron, two Reparatrons,
   one Cannonator, and another visible Reparatron.
5. Final pre-teleporter group: one Invincitron and roughly seven mixed
   Reparatrons/Cannonators.

The same pre-teleporter region visibly contains three placed Gravic
Regulators. They appear at stable map locations, are targetable and killable,
and are not part of the ordinary mixed packs. Their gameplay purpose remains
unresolved.

Most ordinary actors use a visibly short acquisition radius. They can already
be on screen without immediately engaging, and the observed acquisition
distance is approximately comparable to Ride the Lightning's usable range.
Treat that as a conservative footage-derived fallback, not an exact native
perception operand.

## Drops

- The first Invincitron dropped two power pills.
- No drop was observed from the other actors in the first two clusters.
- One Reparatron near the obelisk dropped DNA.
- One of the three Gravic Regulators dropped DNA.

These are observed outcomes, not proven fixed per-noun rewards. Retain the
ordinary server-owned drop rolls unless repeated footage establishes scripted
drops.

## First teleporter

The first security teleporter remains inactive until the ordinary combat
population described above is cleared; the Gravic Regulators were not included
in the observed clear requirement. When activated, an electrical presentation
appears on the pad and a sphere appears at its center. Teleporting to the next
area uses the same sphere-like presentation at the destination.

The authored security query remains the recovered radius-20 alive/team/filter
test documented in `security-contact.md`. The footage adds route-population
and visual-state evidence; it does not replace that native predicate.

## Implementation use

- Implemented: the five correlated traversal loci now use the visible
  `6/4/5/5/8` rosters. Cannonator uses low-band agent
  `ZelemBasicHybrid.Noun`, Reparatron uses `ZelemBasicRepair.Noun`, and
  Invincitron uses the proven `NomadWithDrone.Noun`. Cannonator and Reparatron
  are now package-proven label mappings through their decoded class-attribute
  resources, rather than semantic-only associations.
- The first four rosters follow the visible counts exactly. The final
  eight-actor group uses one Invincitron, four Cannonators, and three
  Reparatrons as the deterministic split of the footage's unresolved seven
  mixed ordinary actors.
- Keep each pack dormant until the hero enters a short local acquisition
  radius; do not aggro the route from the mission entrance.
- Place three Gravic Regulators independently of the security-clear roster.
- Require the pre-teleporter ordinary population to be defeated before
  activating the pad.
- Publish the authored active electrical/sphere presentation at the source and
  arrival presentation at the destination.
- Keep pack counts and per-noun drop outcomes explicitly provisional until
  frame coordinates and content markers are correlated.
