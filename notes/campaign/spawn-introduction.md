# Campaign spawn introduction

## Evidence

Campaign actors do not share one pre-combat presentation. The retained
build-103 AI/content evidence distinguishes these cases:

- `NomadWithDrone.AIDefinition` uses `nBehavior_Idle` before aggro and names
  `FirstAggro_ActivateRobot`. Chunk `71` plays `zlm_minn_tc_2_aggro` for
  `1.3s`; the shared robot shutdown state is
  `zlm_minn_tc_2_shutdown`. The same recovered activation pair is used by the
  Citadel suicide and laser-zone robot profiles.
- Other known first-aggro families use beam-in, burrow-out, drop-in, roar,
  face-target, or no authored first-aggro animation. Those are not evidence
  for a grounded robot startup.
- Authored A/B/C suffixes classify director marker sets, not traversable
  floors. Runtime reports proved that `WandererA` occurs on both the starting
  island near `(-173,-33)` and the later teleporter island near `(550,28)`.
  The prepared 1-1 BFX instead places those locations in connected components
  `2` and `1`, respectively.
- The BFX loader assigns a stable connected-component ID to every navigation
  polygon. Candidate anchors can therefore be projected once during campaign
  preparation and compared by component without running a path search.
- Elite clusters produced by `planCampaignFloor` are centered on positions
  selected from authored Spike loci. That source geometry is the stable
  ambush indicator; ordinary fill points come from Wanderer loci.

## Runtime policy

At run creation, levels with ordinary minion and captain/special pools choose
a stable two-minion population theme and retain the eligible lieutenant
roster. Existing level-specific themes take precedence. Every pre-baked
candidate anchor is projected onto the hero navigation layer once and retains
its BFX connected-component ID. Until all populated components have been
entered, movement performs one bounded player projection; the first player in
a component introduces every ordinary pre-baked group connected to it. It
does not calculate a path per actor or use A/B/C marker suffixes as floors.

An actor with the explicit recovered robot shutdown profile is created visible
in `zlm_minn_tc_2_shutdown`. It remains untargeted and stationary until the
ordinary campaign aggro boundary admits a live player, then plays its authored
`1.3s` activation before pursuit or attack.

Every other ordinary component actor is created visible and receives the
packaged `SpawnModifier` presentation immediately on component entry: stop locomotion,
immobilize, add `generic_spawn.ServerEventDef`, play `horde_beam_in`, wait
`0.5s`, remove the effect, and reset animation. This consumes the actor's
introduction presentation, so later nearby target acquisition begins combat
without replaying another beam-in.

Spike-centered clusters remain proximity-triggered. When the player reaches a
cluster's authored boundary, the complete group is published atomically and
receives its noun-specific first-aggro or generic spawn presentation. Members
that acquire a target later do not replay the generic introduction. This
preserves the ambush shape without making individual members repeatedly poof
into view as the player crosses their separate aggro radii.

Raw director loci, candidates that cannot be projected, and levels without a
usable navigation mesh retain the existing proximity-driven Wanderer/Spike
policy. No shipped game asset is modified.
