# Game-over owner and squad-death condition

## Result

`AD 01` is a server-to-client transition, not a client-authored death report.
Build 103 consumes it in the active gameplay state and selects client state
`13` (`GameOver`). The client does not contain the authoritative sender.

The recoverable death predicate is nevertheless precise at the player boundary:

```text
living(player) = count(slot in player.selected_squad[0:3] where slot.hp > 0)
player_dead     = living(player) == 0
```

This is **not** “the currently deployed creature died.” Build 103 evaluates all
three selected squad slots. The deployed slot gets its current HP from the live
gameplay object; an undeployed slot gets its cached match HP from the player's
character record. A dead active creature can therefore be replaced by either
living reserve creature, and game over must not be emitted at that first death.
Empty or unavailable slots normally have zero HP and do not create extra lives;
the practical tutorial condition is that every actually available tutorial hero
has reached zero HP.

For a solo tutorial, `player_dead` is also the whole-party wipe, so it is the
condition that should latch failure and publish `AD 01` once. For multiplayer,
the strongest supported reconstruction is:

```text
party_wiped = every participating, non-departed player has living(player) == 0
```

That multiplayer reduction is an inference, not a recovered original-server
call site. The client maintains `player_dead` separately for every player, and
the resurrect-orb native can revive dead characters while another player is
still able to act. Those facts rule out ending the whole match merely because
one player's squad is wiped. No retained retail death trace or original
dedicated-server binary in this workspace proves the exact debounce, departure,
or pending-resurrection policy after the last living party creature dies.

## Native proof for all three squad slots

The relevant build-103 functions are in
`bin/darkspinner/GameBin/Game.c`:

- `sub_9C23F0` (`0x009C23F0`) iterates exactly three entries. It increments the
  alive count only when the entry's HP is strictly greater than zero.
- When the loop index equals the deployed index at player offset `+0x0C`, the
  function resolves the deployed gameplay object through the stored object ID
  at `+0x1238` and reads its current combatant HP with `sub_9D8F50`.
- For either undeployed index, it reads cached HP from the three fixed character
  blocks, `player + 0x3D8 + index * 0x5E8` (the decompiler expresses this as
  `this[378 * index + 246]`).
- `sub_9C2470` (`0x009C2470`), exposed to Lua as
  `nPlayer.GetHealthOfCreatureAtDeckIndex`, uses the same split: live object HP
  for the deployed index and cached per-character HP for either reserve index.
- The player HUD update in `sub_442750` (`0x00442750`) computes
  `dead_count = 3 - sub_9C23F0(player)`, calls `SetNumDeadCreatures`, and calls
  `SetPlayerDead(dead_count == 3)`. This directly proves that the client's
  per-player dead state means all three slots, not only the deployed object.
- Other native consumers reinforce the meaning: squad-ability availability
  branches on a particular reserve slot's HP, while defeat/victory presentation
  treats a player as still alive whenever `sub_9C23F0(player) > 0`.

The result is an alive-count predicate over match character resources. It is
not a count of visible/deployed objects, and it is not derived from persistent
creature ownership.

## Resurrection boundary

`nPlayer.PickUpResurrectOrb` is bound to `sub_A082C0`. It enumerates simulator
players, evaluates their match character state, and attempts resurrection for
eligible dead characters. It reports success if any resurrection occurred.
This explains why a per-player squad wipe need not be terminal in co-op: a
surviving teammate can still consume the resurrection mechanic.

The native does **not** expose a separate “resurrection is pending” flag to the
game-over transition, and the adjacent ReCap implementations do not fill that
gap. Their “all three dead, no resurrect orb consumed” prose is explicitly an
unimplemented future specification, not retail authority. Do not delay or
cancel `AD 01` based on that prose without a retail trace or a recovered server
director branch.

## `AD 01` ownership

The wire and client transition are already exact:

```text
uint8 application_id = 0xAD   // ChainGameMsgs
uint8 subtype        = 0x01   // select GameOver
```

`server/raknet.GameOverMessage` encodes those bytes. On the retail client,
`sub_44DC60` reads subtype `1` and calls `sub_44B0A0`, selecting pending state
`13`. Entry through `sub_44A2C0` opens `HUD_Death.swf` and records
`LABS_ALL_PCS_DIED`. That telemetry event happens **after** the state transition;
it is not the death detector or the producer of `AD 01`.

The owner must be the authoritative match simulation, at the same serialized
boundary that applies lethal damage and updates the three character HP records:

```text
lethal damage / death resolution
  -> update match-local character HP/death state
  -> recompute player_dead for the affected player
  -> recompute party_wiped
  -> atomically latch match failure
  -> cancel encounter/object jobs for the match epoch
  -> publish AD 01 once to the match peers
```

It must not be owned by any of the following:

- `sporenet.User`, `sporenet.Creature`, or the persisted squad/deck. Death is
  transient match state and must not mutate the account aggregate.
- the RakNet receive goroutine or packet encoder. Transport publishes an
  already accepted state change; it must not decide gameplay invariants.
- the client HUD, `LABS_ALL_PCS_DIED`, or a client-reported status. Those are
  presentation/telemetry and are forgeable or downstream.
- a per-peer tutorial cache when multiple peers share a game. A wipe is a
  match-wide decision and must see every participating player's character state
  in one serialized snapshot.

## Current darkspin state and the missing integration

Darkspin has pieces of the correct model but no authoritative game-over owner
yet:

| State | What it owns now | Game-over suitability |
| --- | --- | --- |
| `server/game.Instance` | Blaze-visible game membership, slots, and lifecycle | Correct match identity, but it owns `sporenet.User` memberships rather than live simulation players or objects. |
| `server/game.Player` | Fixed `[3]*Character`, `DeployedIndex`, and deploy validation | Correct shape for the per-player squad predicate. |
| `server/game.Character` | Match link to a creature/object plus `IsDeployed` and `IsDead` | Intended transient home, but `IsDead` is currently never mutated and can drift from `Object.Combatant.HitPoints`. |
| `server/game.Object` | Authoritative replicated object and `Combatant.HitPoints` | Correct live HP source for the deployed character, but object death is not integrated with the `Player.Characters` cache or a match terminal latch. |
| `gameplayPeerSession` in `server/gameplay_udp.go` | Tutorial peer flags, one `heroHitPoint`, and `deployedObjectID` | Insufficient. One peer-local shared HP scalar cannot represent three independent squad slots or a multiplayer wipe. |

The implementation target should therefore be one game-loop-owned match
aggregate that contains simulation `Player` records, their three match
characters, object bindings, and a terminal/failure latch. `Instance` may own or
reference that aggregate, but its Blaze membership map is not itself the combat
state. The deployed object's HP and the reserve-character HP cache must be
updated in the same command/tick transaction so `living(player)` cannot observe
a mixed state.

For the current one-player tutorial, replacing `gameplayPeerSession.heroHitPoint`
with that per-character match state is the important prerequisite. Mirroring one
shared HP value into Blitz and Sage presentation packets is useful scaffolding,
but it cannot answer which reserve is dead and must not be used as the final
`AD 01` predicate.

## Confidence and remaining capture gate

| Claim | Confidence | Basis |
| --- | --- | --- |
| `AD 01` is S2C and selects `GameOver` | Certain | Exact client handler and byte encoder. |
| One player's dead condition is zero living entries across three selected squad slots | Certain | `sub_9C23F0`, `sub_9C2470`, `SetNumDeadCreatures`, and `SetPlayerDead`. |
| Undeployed creature HP participates | Certain | Fixed cached character blocks are read for indices other than `DeployedIndex`. |
| A solo tutorial emits failure only after every available tutorial hero is dead | Very high | Exact per-player predicate plus one-player party. |
| Multiplayer match failure waits until every participating player's squad is dead | High, inferred | Per-player dead flags and cross-player resurrection behavior; original server sender absent. |
| Existing/pending resurrect pickups delay the last-party-member failure | Unknown | No authoritative predicate or clean death trace. |
| Exact delay/reliability/order around lethal replication and `AD 01` | Unknown | Requires a retained retail squad-wipe capture. |

The decisive remaining oracle is a clean retail multiplayer capture with one
player squad-wiped while a teammate survives, followed by the last living squad
death. Until then, implement the solo predicate exactly, keep the multiplayer
reduction isolated as match policy, and do not encode speculative resurrect-orb
exceptions into transport code.
