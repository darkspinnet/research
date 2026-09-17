# 1-1 Wanderer and Spike population policy

## Walkthrough correlation

The indexed live recording in `bin/video/walkthrough/1-1` visibly confirms
small packs immediately after deployment and repeated mixed packs appearing
across island/bridge traversal. This supports proximity/traversal-driven
population and rejects both an empty map and a single unconditional opening
roster. It does not recover exact locus selection, noun rolls, group counts,
budgets, polling cadence, or difficulty because the recording exposes only
one edited run.

## Result

The 395 ungated `SpawnPoint_Director*` controls in level 1-1 are candidate
loci consumed as the player traverses the level. They are not 395 enemies and
they are not all populated when the level opens. Wanderer loci are Pac-Man-dot
style proximity hits governed by a short hit counter and tuned spawn odds;
Spike loci are proximity hits at irregular intervals which request a
challenge-budgeted combat group. The server must therefore retain per-run
director state and react to locus hits. A level-opening scan which turns every
marker, or one marker from every set, into an enemy is inconsistent with the
director's documented design.

This determines the policy shape, but the available retail client, content DB,
and public designer material do not recover every build-103 constant. In
particular, they do not identify the exact six nouns selected for a given 1-1
run, the final fifth-hit fallback semantics, the exact lieutenant-count
probability table, or packet ordering within a spawned group. Those details
must not be invented from marker count or marker-set order.

## Evidence boundary

The conclusions below combine three different evidence classes. They are kept
separate because none is a complete server implementation.

1. Dan Kline's 2011 *The AI Director of Game* presentation is primary
   design evidence from Game's AI Director designer. Its slides and
   speaker notes describe the final Wanderer algorithm, Spike challenge
   budgeting, route sections, performance feedback, and point-hit execution.
   The accompanying post also answers questions about the shipped tuning.
2. Build-103 content and client code establish 1-1's concrete marker inventory,
   difficulty-banded noun rows, spatial registration, and client-side selector
   behavior. They do not contain an observed authoritative server spawn call
   for these marker kinds.
3. The recovered C++ `game_server` is an incomplete reference. Its
   Wanderer loop is explicitly temporary, takes only the first marker of each
   A/B/C set, and never handles Spike loci. It is negative evidence against
   copying that behavior, not evidence of the retail policy.

No available trace is a retail server trace. Existing local traces describe
Darkspin's current implementation and cannot establish retail population
timing.

## Policy summary

| Question | Supported policy | Build-103 detail still missing |
| --- | --- | --- |
| Locus selection | Evaluate Wanderer and Spike candidates when traversal/proximity hits them. Wanderer selection uses hit position within a short sequence; Spikes are placed at irregular route intervals. | Exact spatial radius, hit-reset boundary, reuse rule, and tie-breaking when several loci enter range together. |
| Budget | Wanderers use tuned hit probabilities and clump odds, not a budget proportional to the 341 markers. Spikes use a numeric challenge budget derived from predicted monster difficulty, pack size, lieutenant count, route section, and player performance. | Exact build-103 challenge table and 1-1 initial/section targets. |
| Count | A successful Wanderer decision produces a small clump, described as 1-4. Spike count is the composition which satisfies the selected challenge cell; the presentation's table ranges through 16 minions and 7 lieutenants but marks oversized cells as forbidden. | Exact random branch semantics for Wanderer bonus counts and exact allowed Spike cells after final tuning. |
| Noun choice | At level start the director chooses a six-enemy bucket. Wanderers are normally minions, sometimes agents according to player count, and very rarely lieutenants. Spikes center on lieutenants with supporting enemies; lieutenant count has a separately tuned probability. | The six nouns chosen for a particular run, A/B/C bucket assignment in build 103, and any tutorial/first-run override. |
| Difficulty effects | Mission difficulty selects the content rows eligible for nouns. Within a run, section intensity and recent performance choose Hard/Medium/Skip Spike outcomes. Player count affects exceptional Wanderer roles. | Exact mapping from 1-1 difficulty/UI state to the numeric content difficulty and final performance thresholds. |
| Ordering | Candidate evaluation follows player traversal/hit order, not marker-set serialization. Population is emitted only after a decision succeeds. | Order of nouns/positions/object-create packets inside one clump or Spike group. |
| Opening or proximity | Proximity/traversal-driven. Level opening may select the enemy bucket and initialize state, but does not populate all candidate loci. | Whether an initial player placement immediately intersects a locus in retail. |

## Wanderer policy

The presentation describes Wanderer loci as dense Pac-Man dots. The director is
a polling state machine "looking to hit a spike or a wanderer point," and the
algorithm is expressed in terms of each Wanderer point *hit*. This is direct
evidence for player-driven spatial activation.

The final algorithm shown in the deck is:

- 18% on the first eligible hit;
- 25% on hits two through five; and
- tuned 12% / 6% / 2% branches associated with clumps of 2 / 3 / 4
  Wanderers.

The speaker notes describe the intended result as clumps of 1-4. The slide says
"bonus 2/3/4 Wanderers," while the notes say the clump itself is 1-4, so the
available source does not prove whether those branches are mutually exclusive
total counts or additional counts. An earlier algorithm used 27% on hits two
through five and explicitly guaranteed the fifth hit. The final slide no
longer states that guarantee. The fifth-hit fallback must therefore remain
unknown rather than being carried forward from the superseded algorithm.

This counter/probability model explains why marker count alone cannot produce a
spawn count. The 341 1-1 Wanderer controls provide many possible hit locations;
only successful decisions create population. The state also implies that
locus identity and counter state must be run-local, and that a consumed hit
cannot simply be reconsidered on every poll.

The intended noun roles are also explicit: minions are the normal Wanderers;
agents occur occasionally and randomly based on player count; lieutenants are
very rare. A global enemy bucket is selected at level start, with A/B/C portions
of a level drawing from the bucket's rows. This makes `WandererA`, `WandererB`,
and `WandererC` section/bucket labels, not instructions to spawn every member of
each marker set.

## Spike policy

Spikes occur at irregular route intervals and are centered on lieutenant-class
enemies. Their group construction is budgeted by a numeric `challenge` value.
The presentation's challenge table prices a combination by total enemies and
lieutenant count; examples include challenge 5 for one minion, 30 for one
lieutenant, 60 for six minions, 110 for six enemies including one lieutenant,
and 320 for sixteen minions. Oversized table cells are colored as combinations
the director will not spawn. The table is evidence of group-composition
budgeting, not permission to use its largest row as 1-1's default count.

The designer described challenge as starting around 50 and rising across a
level, with a late hard Spike able to reach about 320. Medium Spikes were
significantly smaller, roughly half a hard Spike, and easy outcomes smaller
again. The precise shipped values were tuned, so these figures bound the model
but do not uniquely determine 1-1's group at any one locus.

Route progress divides the level into A/B/C narrative sections with increasing
intensity. Recent performance then changes the next Spike among Hard, Medium,
and Skip. Performance was described as a binary doing-well/not-doing-well
decision based on current health and number of living creatures, considering
the prior two Spikes. The deck gives the transition table:

| Previous two | Doing well | Doing poorly |
| --- | --- | --- |
| Hard, Medium | Hard | Medium |
| Hard, Skip | Hard | Medium |
| Medium, Hard | Medium | Skip |
| Medium, Medium | Hard | Skip |
| Medium, Skip | Hard | Medium |
| Skip, Hard | Medium | Skip |
| Skip, Medium | Hard | Medium |

The number of lieutenants is selected with a separate, custom-tuned probability
per lieutenant count. Kline recalled that the shipped table was retuned and
that six- and seven-lieutenant outcomes were never used. This is stronger
evidence than assuming all columns of the illustrative challenge table were
reachable.

The deck says designers placed Spike points everywhere they could, averaging
about ten per level, subject to geometry. Level 1-1 has 54 Spike controls, so a
control cannot be equated with a guaranteed encounter. They form a spatial
candidate field which the director polls; the performance state and Skip
outcome further prove that every Spike marker is not prepopulated at opening.

## Build-103 mapping for level 1-1

Level 56 (`zelems_1`) contains 395 ungated ordinary director controls across 15
marker sets:

| Marker-set ordinal | Class | Weight | Controls |
| ---: | --- | ---: | ---: |
| 23 | `SpawnPoint_DirectorWandererA` | 1 | 106 |
| 24 | `SpawnPoint_DirectorWandererB` | 1 | 141 |
| 25 | `SpawnPoint_DirectorWandererC` | 1 | 94 |
| 26-37 | `SpawnPoint_DirectorSpike` | 1 or 2 | 54 total |

Spike sets 30-32 have weight 2; the others have weight 1. There is no
`level_event` gating these controls. That means they are available to the
director's spatial policy, not that their weights or lack of events specify a
spawn count. The marker classes map to native director kinds 7 (Wanderer) and
8 (Spike).

For content difficulty 1-24, level 1-1 exposes these noun rows:

- minion: `ZelemBasicRanged`;
- special: `ZelemSpecialHaster_Captain`;
- agent: `ZelemBasicRanged`, `ZelemBasicHybrid`, `ZelemBasicRepair`;
- captain: `ZelemSpecialHaster`, `ZelemNomadSnipe`,
  `ZelemNomadWithDrone`; and
- boss: no row.

Difficulty 25-48 and 49-72 substitute the corresponding middle- and high-band
rows recorded in `notes/campaign/1-1/nouns.md`. Difficulty therefore changes eligible
nouns without changing the 395-locus inventory. These rows are candidate role
pools; they do not by themselves say which six nouns the level-opening bucket
selected or which noun a later locus receives.

The public EA launch interview says the AI Director selects six out of 96
possible enemies at the beginning of each level. Kline's deck likewise shows a
six-enemy bucket and explains that the first randomization is the enemy types
for the level. This is compatible with opening-time *selection state*, but not
opening-time population. Build-103's exact bucket assembly and any tutorial
overlay are not present in the evidence examined here.

## Native client findings

The client registers director markers in its spatial store through
`sub_9FB420`. The level-load path subsequently calls `sub_9FD490` (through the
`sub_9FE7C0` thunk) to collect/prune Spike markers relative to special regions.
Both operations prepare candidate geometry; neither creates an enemy.

The generic client selector in `sub_9FE270` dispatches kind 7 to
`sub_9FDD50`, a budgeted selector with a hard cap of 15 selected nouns, and kind
8 to `sub_9FD260(this, 2, difficulty, -1)`, which selects one noun through the
boss slot. No observed client caller invokes `sub_9FE270` for kind 7 or 8; its
only direct call supplies kind 1 with budget 10. The base 1-1 boss pool is also
empty. Consequently these routines constrain client data semantics but cannot
be promoted into the missing authoritative server policy. In particular:

- the kind-7 cap of 15 is not the documented Wanderer clump count;
- the kind-8 boss lookup does not prove that Spike loci are inert; and
- neither routine establishes when the server sends object creation.

The object-create packet (`0x8c`) is generic, and the recovered client contains
no authoritative sender for director spawns. The director-state packet
(`0x8b`) does not carry a spawn list. Exact intra-group creation ordering is
therefore unobserved.

## Spawn timing and ordering

The supported runtime sequence is:

1. At level opening, initialize director state and choose the run's enemy
   bucket; do not materialize all candidate controls.
2. As player movement hits eligible spatial loci, evaluate them in traversal
   order. Marker-set serialization order is not gameplay order.
3. For a Wanderer hit, advance the short hit sequence and apply the applicable
   spawn/clump decision. On success, select nouns from the run's role bucket.
4. For a Spike hit, use route section and the recent-performance state to pick
   Hard, Medium, or Skip. If not skipped, choose lieutenant count and a group
   composition within the selected challenge target.
5. Emit object creation for the resulting group and update consumed-locus,
   hit-counter, and performance history state.

This sequence does not establish the order of members within a successful
group. Without a retail trace or authoritative server code, the server must not
derive that order from marker ordinal, marker-set order, noun-row order, or the
order of the illustrative challenge table.

The roughly 20-enemy on-screen limit mentioned in the presentation is a global
performance constraint, not a per-locus count. Spikes were already close to
that limit. It can inform eventual population management, but it is not a
license to truncate every requested group to 20 without recovering the actual
despawn/cull policy.

## Rejected opening-time behavior

The recovered reference server's `OnPlayerStart` calls `LoadLevel`, creates
design objects, and then runs a temporary `TestEnemy` loop independently for
WandererA, WandererB, and WandererC. It chooses a random one of six chain nouns,
uses the first marker, then unconditionally `break`s. It has no Spike branch.
That produces at most three Wanderers at opening and is visibly scaffolding:
the source comment immediately before the break says `// temporary`. It
contradicts the documented hit/probability policy and must not be copied.

Likewise, none of these shortcuts is supported:

- one enemy per marker (395 opening enemies);
- one enemy per marker set (15 opening enemies);
- one enemy per Wanderer A/B/C set (the temporary reference behavior);
- treating weight 2 as two enemies; or
- using the empty base boss row to suppress every Spike.

## Remaining evidence needed for exact parity

Exact implementation should wait for at least one of the following: a retail
server trace covering locus entry and object creation, recovered authoritative
server code/configuration, or additional shipped director tuning data. The
highest-value unknowns are the spatial hit radius and tie order, counter reset
and fifth-hit behavior, final clump-count branching, the six-noun bucket
construction, 1-1 challenge targets, lieutenant-count probabilities, and
intra-group position/create order.

## Sources and reproducibility

- Dan Kline, [The AI Director of Game](https://dankline.wordpress.com/2011/06/29/the-ai-director-of-game/), including the linked
  [PDF](https://dankline.wordpress.com/wp-content/uploads/2011/06/ai-director-in-game-gameai-2011.pdf)
  and speaker notes in the linked
  [PPTX](https://dankline.wordpress.com/wp-content/uploads/2011/06/ai-director-in-game-gameai-2011.pptx).
  The downloaded PDF SHA-256 was
  `47292AD0F333F32AD96B9CFCAA9C1817E6232FFA1A66734B6A6B2F26BEF2B6F2`; the
  PPTX SHA-256 was
  `3785A9A64B7831A17A0B929074D6EA3C53E479DA89962887A1A6A0EE624E93F1`.
- EA, [Game Beams Into Stores Today](https://www.ea.com/news/game-beams-into-stores-today), for the six-of-96 level-opening selection statement.
- Canonical client decompilation: `bin/game/GameBin/Game.c`, notably
  `sub_9FB420`, `sub_9FD260`, `sub_9FD490`, `sub_9FDD50`, `sub_9FE270`, and the
  `sub_9FE7C0` thunk.
- Canonical client database: `bin/game/GameBin/Game.idb`.
- Recovered incomplete server:
  `bin/game/logs/recap_server-reference/game_server/source/game/instance.cpp`,
  `OnPlayerStart` and its temporary `TestEnemy` loop.
- Local content derivations and cross-checks: `notes/campaign/1-1/opening.md`,
  `notes/campaign/1-1/director.md`, `notes/campaign/1-1/budget.md`, `notes/campaign/1-1/native.md`, and
  `notes/campaign/1-1/nouns.md`.
