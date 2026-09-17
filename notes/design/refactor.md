# Gameplay execution refactor proposal

## Purpose

Make gameplay state easier to reason about now and allow multiple game instances
to simulate concurrently later without making one shared level nondeterministic.
The proposal intentionally starts with small behavior-preserving extractions and
does not require an immediate rewrite of `gameplay_udp.go` or the existing
feature-sized `sim.Session` runtimes.

The authoritative concurrency boundary should be one game/level instance, not
one ability and not necessarily one player. Players sharing a level must commit
movement, combat, death, loot, objectives, random draws, and timers through the
same ordered owner. Unrelated game instances should be able to run concurrently.

## Current design summary

- `gameplayPeerSession` contains most authoritative movement, action admission,
  encounter, pickup, enemy, objective, timer, and cancellation state.
- The gameplay handler stores session structs by value in an address-keyed map
  protected by one `sync.RWMutex`. Callbacks repeatedly copy a session out,
  mutate it, and assign it back.
- RakNet holds one server-wide outbound mutex while decoding connected traffic,
  running the gameplay handler, encoding responses, and writing them. Scheduled
  packet producers acquire the same mutex before running their callbacks.
- This global serialization currently prevents many races, but also prevents
  unrelated games from executing concurrently and makes transport ordering an
  accidental gameplay synchronization mechanism.
- Scheduled producers can mutate gameplay state directly and some held actions
  recursively invoke the main handler. Cancellation is spread across run
  pointers, cancel functions, epochs, and feature-specific generation fields.
- The `sim` package already provides the desired deterministic core: a
  single-threaded monotonic clock, ordered task heap, scoped cancellation,
  semantic events, and an ordered dispatcher. It is currently used by many
  short feature-sized runtimes rather than as the owner of an entire game.
- Packaged Lua does not run in one shared conventional VM. Most invocations use
  an isolated custom Lua bytecode compiler/interpreter to produce typed
  `sim.Program` data. Retained objective runtimes protect their own private
  compiler state. Lua VM contention is therefore not the primary scaling issue.

## Design decision

Use many concurrent producers and one ordered consumer per authoritative game
instance:

```text
RakNet player input ----+
scheduled deadlines ----+--> bounded game mailbox --> game executor --> effects
developer commands -----+                              |              packets
disconnect/lifecycle ---+                              |              schedules
                                                       +--------------persistence
```

Different game executors may run concurrently. Commands within one game commit
sequentially. Reliable packet sequencing remains ordered per peer, but must not
require holding one server-wide lock across gameplay execution.

For the current solo case, a game instance and a peer session are nearly the
same thing. The API and registry should nevertheless use game identity so that
multiple players can later share one executor when they share one level.

## Non-goals

- Do not treat concurrency as a fix for the current client action soft lock.
  The latest playtest shows that affected movement, active-ability, and squad
  switch requests stop reaching the server.
- Do not execute state-mutating abilities concurrently inside the same game.
  Damage, death, cooldowns, movement interruption, objectives, loot, packet
  order, and the simulator random stream require a deterministic commit order.
- Do not introduce one goroutine or Lua VM per ability.
- Do not migrate every existing short `sim.Session` in one change.
- Do not move persistence out of the ordered game transaction until an explicit
  reserve/perform/commit protocol exists for the affected operation.
- Do not add a distributed event bus merely to run unrelated games in separate
  processes. Stable game-to-instance routing is sufficient initially.

## Phase 0: characterization and guardrails

This is the lowest-risk starting point and should land before structural work.

1. Add focused sequence tests for the action combinations repeatedly exercised
   by playtests:
   - held basic followed by movement;
   - pursuit followed by a duplicate client retry;
   - pursuit target death or despawn;
   - basic followed by an active ability;
   - active ability followed by squad switch;
   - any active action followed by developer reset;
   - scheduled continuation arriving after switch, death, reset, or disconnect.
2. Add a game/session revision and include it in diagnostic logs for accepted,
   rejected, cancelled, and stale gameplay work. Keep protocol timestamps
   separate from this internal revision.
3. Record lightweight queue-independent timing around gameplay command handling,
   scheduled producer execution, persistence, encoding, and UDP writing. This
   establishes whether later changes improve isolation without guessing.
4. Add or retain race-enabled focused tests for session replacement, schedule
   cancellation, and immediate-versus-scheduled packet ordering.

Exit criteria:

- The important action and cancellation sequences have deterministic tests.
- Logs can correlate an input, its game/session generation, its state commit,
  and its produced response.
- No gameplay behavior or wire representation changes.

## Phase 1: consolidate action admission

This is the first implementation refactor because it directly reduces the
fragility currently visible during movement and ability playtests.

Extract a `campaignActionState` from `gameplayPeerSession`. It should own at
least:

- the basic combo and held-input sequence;
- the current basic run identity;
- the global ability release deadline;
- player pursuit source, target, ability, and generation;
- the cancellation token or generation for action-owned continuations.

Expose domain operations rather than fields. Candidate operations are:

```text
AdmitMovement
AdmitBasic
AdmitActive
BeginPursuit
AcceptPursuitRetry
CompletePursuit
CancelForMovement
ResetForDeployment
ResetAll
```

Each operation should return a typed decision containing any previous run or
token that the caller must stop after the state transition. It should be
impossible for movement or switching to clear only part of the action gate.
Keep packet marshalling and RakNet response types outside this state object.

Use the Phase 0 sequence tests as the contract. Initially preserve all existing
admission timing and response behavior, including the build-103 held-input and
duplicate-pursuit rules.

Exit criteria:

- Movement, ability, pursuit, switch, and reset paths no longer manipulate the
  extracted fields independently.
- One reset operation releases every server-owned action gate.
- Focused and full server tests pass with no wire changes.

## Phase 2: introduce stable game runtimes

Separate registry synchronization from game-state synchronization before adding
concurrent execution.

1. Introduce a stable `gameplayRuntime` object containing the authoritative
   state, game identity, generation/revision, lifecycle context, and current
   execution primitive.
2. Change the registry from copied session values to stable runtime references.
   The registry lock should protect only lookup, insertion, replacement, and
   removal; it should not be held while simulating gameplay or performing
   persistence.
3. Route addresses and authenticated players to a runtime keyed by the
   authoritative game instance ID. During the solo transition, an address may
   still be the only member of its game.
4. Encapsulate state access behind one method such as `Execute` or `Do`. At this
   phase it may use a per-runtime mutex and run synchronously. The important
   change is that callers stop copying and assigning the aggregate directly.
5. Keep generation checks at the runtime boundary so work from a replaced peer
   or completed game cannot enter the current state.

This phase deliberately does not make gameplay concurrent. It removes the
value-copy and global-map-lock assumptions that would otherwise become races
when RakNet is made concurrent.

Exit criteria:

- Unrelated runtime state is not protected by one gameplay map lock.
- A gameplay state mutation has exactly one stable owner.
- Session replacement and cleanup cannot overwrite a newer runtime.
- Existing immediate and scheduled ordering tests still pass.

## Phase 3: command and result boundary

Make network input, timers, control commands, and lifecycle changes enter the
same game-owned execution API.

Define typed command envelopes with at least game identity, runtime generation,
arrival sequence, source time, actor identity, and payload. Avoid passing a full
`raknet.Packet` into game-domain code where only a decoded command is needed.

Define an execution result that describes effects instead of publishing them
while state is being mutated. A possible shape is:

```go
type gameplayResult struct {
	Packets   [][]byte
	Schedules []gameplaySchedule
	Cancels   []gameplayScheduleToken
}
```

The concrete types should distinguish protocol packets from semantic scheduled
commands if that distinction prevents invalid combinations.

Migrate incrementally:

1. Developer reset and one small scheduled feature.
2. Movement interruption and contact evaluation.
3. Basic action admission and its hit/release continuations.
4. Active abilities, enemy actions, death, loot, and encounter transitions.

Scheduled deadlines must enqueue a typed command back to the owning runtime.
They must not mutate the session map directly or recursively call the RakNet
handler. Stale work is discarded using runtime and action generations at the
game boundary.

Exit criteria:

- At least one complete input-to-timer feature uses typed commands and results.
- Its scheduled callbacks contain no direct session-map mutation.
- Command replay in a fixed order produces the same authoritative result and
  packet order.

## Phase 4: per-game executor

Replace the temporary per-runtime mutex execution with a bounded mailbox and a
single ordered consumer.

- RakNet, timers, developer commands, and lifecycle events become producers.
- The executor owns the game state, simulator random stream, and command
  sequence. Code running inside it should not need gameplay-state mutexes.
- The mailbox must be bounded. On saturation, reject or disconnect explicitly;
  never allow unbounded memory growth.
- Preserve FIFO ordering initially. Movement coalescing may be considered only
  after measurement and only for unprocessed redundant movement updates. Never
  coalesce abilities, damage, death, switch, reset, or lifecycle commands.
- Avoid executor reentrancy. A command handler must not synchronously enqueue
  work to its own mailbox and wait for that work.
- Persistence may initially block only the owning game executor. This is safer
  than allowing later commands to observe an uncommitted durable mutation and
  is already an improvement over blocking every game through RakNet's global
  lock.

One goroutine per active game is acceptable for the first multi-game version.
If profiling later shows too many mostly idle executors, map game IDs onto a
fixed number of executor shards while preserving order within each game.

Exit criteria:

- Commands for one game always commit in a deterministic order.
- Two unrelated games can execute their command handlers concurrently.
- A slow persistence operation in one game does not block another game's state
  execution.
- Queue saturation and shutdown have explicit tested behavior.

## Phase 5: remove transport-wide gameplay serialization

RakNet should own transport ordering, not gameplay serialization.

1. Stop holding the server-wide outbound mutex while invoking game execution.
2. Add a per-peer outbound sequencer that exclusively owns reliable message,
   order, split, and datagram counter allocation for that peer.
3. Submit immediate and scheduled gameplay results to the same per-peer output
   path so their final wire order remains defined.
4. Keep connection replacement and connection-generation validation at enqueue
   and write boundaries.
5. Ensure one peer's slow handler, encoder, or socket failure does not block
   unrelated peers.

This should happen only after stable game runtimes exist. Removing the current
global lock earlier would expose copied-session races and stale overwrites.

Exit criteria:

- Immediate and scheduled packets retain their required per-peer order.
- Separate games can decode, simulate, encode, and publish concurrently.
- Connection replacement discards stale queued output without affecting the
  replacement peer.
- Transport and server race tests pass.

## Phase 6: converge scheduling on the game simulator

Gradually make one game-level deterministic clock the source of gameplay
deadlines. RakNet scheduling should ultimately publish already-committed output
or wake a game deadline; it should not be the owner of combat semantics.

For each migrated feature:

1. Represent hit, release, pulse, expiry, pursuit, and respawn deadlines as
   game/simulator tasks with scoped cancellation.
2. Emit semantic intents through the existing `sim` dispatcher and adapters.
3. Remove the equivalent RakNet producer recursion, run pointer, and duplicated
   generation field only after parity tests pass.
4. Preserve the feature-sized behavior implementations where useful. They can
   become programs or components driven by the game-level session rather than
   independent authorities.

Do not require every feature to migrate before multi-game concurrency ships.
The command boundary from Phases 3-4 can safely contain legacy scheduled runs
while migration proceeds.

Exit criteria:

- Migrated gameplay deadlines are replayable from ordered commands and a clock.
- Cancellation follows session, phase, and role/action scopes rather than
  ad-hoc callback ownership.
- Semantic trace fixtures cover the migrated behavior.

## Phase 7: optional intra-game parallel calculation

Only pursue this after multi-game profiling demonstrates that one busy game is
CPU-bound. Ability execution itself is expected to be too small and too coupled
to benefit.

Potentially expensive pure work such as pathfinding, large spatial queries, or
AI planning may use workers through a snapshot/proposal protocol:

1. Read an immutable world snapshot and revision.
2. Calculate a proposal concurrently without mutating game state or consuming
   the authoritative random stream.
3. Return the proposal tagged with its input revision.
4. Validate and commit it in the ordered game executor, or discard it if stale.

Never allow workers to commit damage, death, resources, cooldowns, objectives,
loot, random draws, or outbound packets directly.

## Deployment model

The same ownership model supports all expected deployments:

- A single process can run many game executors concurrently.
- A fixed worker pool can shard games by `hash(gameID) % workerCount` while
  maintaining per-game ordering.
- Separate server processes or machines can own disjoint game IDs without a
  shared gameplay event bus.
- Immutable compiled content can be cached process-wide. Mutable objective,
  simulator, random, and world state remains game-owned.
- Matchmaking or the composition root records which process owns a game and
  routes authenticated peers to it for the lifetime of that instance.

## Recommended work order

1. Phase 0 sequence tests and diagnostics.
2. Phase 1 `campaignActionState` extraction.
3. Phase 2 stable runtime registry and state ownership.
4. Phase 3 typed command/result boundary, beginning with reset and one timer.
5. Phase 4 bounded per-game executor.
6. Phase 5 per-peer RakNet outbound sequencing and removal of global gameplay
   serialization.
7. Phase 6 incremental game-level simulator scheduling.
8. Phase 7 only if production profiling justifies intra-game workers.

The first two phases are useful even if later concurrency work is postponed.
Phases 2-5 form the minimum architectural path for safely running multiple game
instances concurrently in one process.

## Success measures

- Action admission has one explicit state owner and one complete reset path.
- Stale continuations cannot act after switch, death, reset, game replacement,
  or disconnect.
- Gameplay state is never copied out of a shared registry for mutation.
- Packet transport is not the lock that makes gameplay state safe.
- Separate games make progress concurrently and failures remain isolated.
- Players sharing one level observe one deterministic authoritative order.
- Existing build-103 packet fixtures and semantic simulation traces remain
  stable unless a separately documented behavior correction requires a change.
