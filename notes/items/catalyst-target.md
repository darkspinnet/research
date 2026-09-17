# Catalyst pickup command admission

## Result

Build 103 does not use the equipment type-11 contract for catalysts. A
campaign catalyst takes type 9 only when the resolved object already has
`cLootData` at object `+744` and its noun definition has a non-null field at
`+148`. The representative packaged catalyst nouns satisfy both construction
conditions without an interactable component:

- their serialized noun class is `0xcd482419` (`-850910183`), the crystal
  class that makes `sub_9CBCD0` allocate `cLootData` during object construction;
- their serialized noun field `+148` is nonzero, so `sub_9D9770` returns true;
- their three intrinsic-interactable definition slots at noun `+136/+140/+144`
  are all zero. `sub_A191B0` therefore creates no intrinsic interactable state,
  and `sub_A190A0` supplies no type-11 ability for this class.

The current sparse `0x9a` update is not what creates `cLootData`; it updates
field zero on the component that the noun constructor already allocated. This
means an intermittent no-send on a correctly resolved representative catalyst
is not explained by the missing equipment `0x98` update. Adding
`cInteractableData`/`PickUpLoot` would bypass the proven type-9 design and is
not supported.

There are two evidence-backed no-send boundaries instead:

1. The automated load-test picker calls
   `FindGoodMeleePosition(actor, candidate)` before it classifies any nearby
   object as loot or crystal. Failure prevents assignment to `ClosestCrystal`
   or `ClosestLoot`, so `nAction.PickUpCrystal`/`InteractWithObject` is never
   called and no action can be staged. This gate is presentation/test AI, not
   part of a manual click.
2. A manual click that reaches `sub_44CAF0` can satisfy the noun predicate but
   still have `sub_4DFB40` decline to install the pending type-9 action because
   of the shared local action state. `sub_44CAF0` discards that return code.
   Once the pending action is installed, the normal update sends it before the
   subsequent status/finish logic. Lob completion is not consulted by this
   sender and cannot by itself cancel a staged type 9.

The strongest current hypothesis for automatic/intermittent misses is the
landing/reachability boundary, not noun construction. Darkspin chooses a
compatibility destination exactly one unit from the player rather than the
native source-centered navmesh sample. During the half-second lob, and after a
landing beside the actor or an obstacle, `FindGoodMeleePosition` may fail its
collision, ground projection, or exact-reachability checks. There is also an
unclosed replication gap: native `sub_A2EF00` changes object movement type to
`4`, while darkspin publishes only the three `cLocomotionData` fields and no
object field-19 movement-type delta or final object-position delta. This is an
implementation candidate only after a live trace proves which client state is
wrong; the original authoritative sender is absent from this executable.

An accepted pickup's scheduled publisher and one-second simulator must share
the same pending inventory and pickup objects. Copying either object into the
scheduled step before release preserves the empty pre-assignment slot, causes
collection encoding to reject the simulator's valid assignment, and rolls the
reservation back so the same catalyst can be accepted repeatedly.

## Exact client admission path

All line references are to canonical
`bin/game/GameBin/Game.c`.

1. `sub_44A7C0` at lines `196663-196684` resolves the clicked object. When
   object `+744` is non-null it returns true only when component float `+64`
   (`mDNAAmount`) is `<= 0`. It does not call the generic interactable checks
   on this branch. A catalyst constructed with `cLootData` and baseline DNA
   zero passes.
2. `sub_44CFA0` at lines `198901-198954` calls `sub_44CAF0` for a non-combat
   target that passes `sub_44A7C0`; otherwise it issues ground movement. A
   visible approach is therefore not proof that a pickup command was staged.
3. `sub_44CAF0` at lines `198612-198652` requires object `+744` and
   `sub_9D9770(object noun)` for its type-9 branch. It copies target object ID,
   current target XYZ, and selector `-1` into the 20-byte type tail and calls
   `sub_4DFB40`. Only the fallback branch calls `sub_9D8A40` and builds type
   11.
4. `sub_9D9770` at lines `1389811-1389832` resolves the noun definition and
   returns exactly whether noun field `+148` is non-null. Adjacent accessors
   `sub_9D97F0`, `sub_9D9870`, and `sub_9D9900` read fields `+0`, `+16`, and
   `+20` from the pointed record. Their UI and crystal-combination call sites
   identify it as the crystal display/color/rarity definition, rather than a
   generic role string or an interactable ability.
5. `sub_9CBCD0` at lines `1378823-1378872` allocates object `cLootData` when
   the noun class word is either `0xcd482419` (crystal) or `0x292fea33`
   (equipment loot). Generic construction calls it from `sub_9D1930` at line
   `1383122`, before the object enters its pool. This is why a valid crystal
   has `cLootData` before the later `0x9a` field-zero delta is decoded.
6. `sub_539D10 -> sub_A1DEC0` remains only a reflection decoder. Like the
   already documented `0x98` decoder, it resolves a component; it does not
   change the noun class or repair a missing construction-time component.

The type-9 tail and final sender remain as previously recovered: logical
action message `29`, wire `0x9c`, 40-byte common header plus 20-byte tail,
reliable ordered arguments `1,3,0`. The last tail byte is selector `-1`; the
three following bytes are padding and should be zero in a deterministic
encoder.

## Representative `content.db` noun evidence

The authoritative Darkspinner runtime database was queried with
`--config bin/darkspinner/darkspin.toml`. `crystal_definition` rows `1`, `2`,
`3`, and `49` select representative red normal/rare/epic attack-speed nouns
and a blue crowd-control-reduction noun. The corresponding base-name FNV-1
instance IDs resolve these `content_source_resource` rows in
`AssetData_Binary.package`:

| Noun | Resource / ordinal | Size | Decoded SHA-256 | Noun `+148` |
| --- | ---: | ---: | --- | ---: |
| `crystal_attackspeed.Noun` | `8009 / 8008` | 642 | `55009a7374186e1081cf2fbe0c1cc1696453c2c4b4d68be859cdd2f8ea1de484` | `0x043b3be8` |
| `crystal_attackspeed_rare.Noun` | `9810 / 9809` | 651 | `c9228d862f12204f7ae66dabb5c1434251c9b11a76109854d2beff11ac55d2ad` | `0x043ac788` |
| `crystal_attackspeed_epic.Noun` | `8184 / 8183` | 651 | `9bb698136d2fd9b0a1658b9cfea3959836175cc8b109d7a9f8b11d95a657c9f9` | `0x043bd2a0` |
| `crystal_ccreduction.Noun` | `13157 / 13156` | 643 | `6ed6119864ba043a2fb9b5180dac30bb1163ec8d7343b054b49012cb0be39a59` | `0x044106a8` |

All four decoded resources have:

- noun class `0xcd482419` at `+0`;
- authored lifetime `120.0` at `+12`;
- zero words at `+136`, `+140`, and `+144`;
- a nonzero crystal-definition reference at `+148`;
- the same footprint inputs at noun `+56/+60/+68/+72`:
  `-0.5, -0.5, 0.22, 0.32`.

`sub_9D70E0` at lines `1387794-1387847` computes the no-shape-resource
footprint radius as
`sqrt(max(abs(+68),abs(+56))^2 + max(abs(+72),abs(+60))^2) * scale`.
At darkspin's scale `1`, each representative catalyst therefore has exact
radius `sqrt(0.5)`, approximately `0.707107`. This radius participates in
`FindGoodMeleePosition`; it is not a reason to add an interactable component.

Decoded copies are retained under `bin/game/logs/catalyst-target/`. The asset
catalog independently tags the attack-speed noun `crystal` and `red`, but the
construction proof above comes from the real type-`0x76a8f7d8` noun payloads,
not catalog tags.

## Action state and cancellation

`sub_4DFB40` at lines `313901-313931` validates the requested type, and only
for a nonnegative result installs the shared pending targeted-action record at
`byte_1438AC0`. It allocates a new one-byte sequence token, stores the action
type and tail, and invokes `sub_4DF800` immediately only when the validation
result is `1`. `sub_44CAF0` neither tests nor reports a negative result. This
is the exact boundary to break on when the animation/approach occurs without a
packet.

For an installed pending type 9, `sub_4DF800` at lines `313738-313781` calls
`sub_4DF5B0` whenever the sent byte is still false, sets the sent byte true,
and only then evaluates the action-status/finish path. `sub_4DF5B0` at lines
`313578-313615` copies the type-specific tail, builds the common header, and
calls the sole action sender `sub_5370F0`. Its only type-specific special case
is for types `7` and `8`, not type `9`.

The pending record can be cleared before that send by `sub_4DEC70` at lines
`312992-313003`, but only when its incoming one-byte sync stamp equals the
pending record's sequence token. Its main caller is response subtype `2` in
`sub_4D64D0` at lines `306056-306080`; that is the client consumer of
darkspin's `ActionCommandResponseMessage{ResponseType: 2}`. A type-2 response
to the type 9 itself necessarily arrives after the client has sent that type
9, so it cannot explain a trace in which the request never existed. It can
explain a no-send only if a delayed cancellation for another action matches a
new pending token (the token is one byte and wraps) or if some local/incoming
path clears the record in the staging-to-update window. That race remains a
capture question.

Darkspin currently emits matching type-2 responses for rejected campaign
pickups and for pursuit transfer. The relevant client token is common-header
byte `+1`, decoded as `command.Common.Unknown[0]`; it is not the 32-bit input
sync stamp at common offset `+4`. Any capture must compare this byte, the
pending token at `byte_1438AC2`, and response order. Changing rejection or
pursuit response policy without that comparison would risk leaving local
actions pinned.

## Lob completion boundary

Native `sub_A2EF00` at lines `1460176-1460312` performs four relevant writes:

- object movement type byte `+97 = 4`;
- locomotion `lobStartTime` at component `+216/+220` from the simulation clock;
- start/destination and duration at component `+228...+272`;
- the derived reflected lob coefficients.

`sub_A04B40` at lines `1426515-1426535` considers a lob active only when a
locomotion component exists, object movement type is `4`, and unsigned elapsed
simulation milliseconds are below `duration * 1000`. It does not clear
movement type when the duration elapses; it simply returns false. This helper
is used by the full-crystal rejection/relaunch path, not by `sub_44CAF0`,
`sub_4DFB40`, or `sub_4DF5B0`.

Consequences:

- Manual-click type-9 construction has no active-lob or completed-lob gate.
- A staged type 9 is not canceled when the half-second lob ends.
- Lob timing can still affect automated discovery because that code asks for
  a reachable melee point around the catalyst's current client transform.
- A start time ahead of the client's simulation clock underflows the unsigned
  subtraction and appears inactive; a stale start time appears already
  complete. Clock alignment and the client transform at `start+500ms` must be
  captured rather than inferred from the server's stored destination.

Darkspin's current `marshalCampaignCrystalDrop` sends
`ObjectCreate(source) -> cLootData(field 0) -> cLocomotionData(fields 0-2)` and
stores the destination immediately as the authoritative pickup position. It
does not publish object movement type field `19` or a final position update.
The native initializer proves those local mutations exist, but the missing
original sender prevents claiming which belonged in create, an object delta,
or a later snapshot. This gap is directly relevant to reachability and must be
observed before changing the wire.

## `FindGoodMeleePosition`

Runtime `content.db` chunk `705`,
`LuaTestScripts/0x89404DF4.lua`, has bytecode SHA-256
`7a1625acaf4229acfbea77fef0e5e6bd6be8bb0afd2b9357866731c21e4c26a7`.
Prototype `0.7` obtains objects sorted by distance within `20`, then at PCs
`99-105` calls `nLocomotion.FindGoodMeleePosition(actor, candidate)` before
calling `GetType`. A false result skips all classification for that candidate.
Only after a true result do PCs `228-239` assign a `kType_Crystal` candidate
to `ClosestCrystal`; the later in-game prototype checks slot availability and
calls `nAction.PickUpCrystal`.

The native binding is `sub_A0CAF0` at lines `1432371-1432551`. It:

1. resolves both objects and fails immediately if either is absent;
2. adds their `sub_9D70E0` footprint radii;
3. constructs a surface-contact candidate around the target;
4. rejects candidates for obstacle sweep/collision, failed ground projection,
   excessive projection displacement, or failed
   `sub_9E91A0(actor,candidate,0.01)` exact reachability;
5. rotates around the target in configured angular steps until a candidate
   succeeds or the search limit is exhausted;
6. returns boolean false plus three zero coordinates on exhaustion, otherwise
   true and the projected point with `Z + 0.1`.

The helper does not inspect `cLootData`, noun field `+148`, action state, or lob
timestamps. Its dependency is the current object transform, footprint,
collision/navigation scene, and actor reachability. It can therefore explain
automatic no-send before `sub_44CAF0`, but cannot explain a manual click that
has already reached `sub_44CAF0`.

## Implementation-ready decision tree

No behavior change is justified until one failing attempt is placed on this
tree:

1. If `FindGoodMeleePosition` returns false and neither `nAction` nor
   `sub_44CAF0` runs, inspect the catalyst transform, movement type, lob clock,
   and nav projection. A darkspin-only post-landing failure supports correcting
   placement/final-lob replication; it does not support `0x98`.
2. If `sub_44CAF0` runs but `sub_9D9770` is false, record object noun ID,
   resolved noun class, noun `+148`, and object `+744`. For the four nouns above
   this indicates wrong/unresolved construction state, not missing reflected
   crystal level.
3. If the predicate is true but `sub_4DFB40` returns negative, map the exact
   code and the active movement/ability/channel state. Fix the owning
   cancellation or retry policy; do not mutate the catalyst noun.
4. If `sub_4DFB40` installs the record, watch `byte_1438AC0/AC1/AC2`. A matching
   response subtype `2` that clears it before `sub_4DF5B0` identifies an action
   response race. Compare the response token and originating server branch.
5. If `sub_4DF5B0` and `sub_5370F0` run, the problem is below command
   admission (RakNet capture/filtering), not targetability.

## Remaining capture gaps

1. Capture one successful and one failing darkspin catalyst attempt with
   breakpoints at `sub_A0CAF0`, `sub_44A7C0`, `sub_44CAF0`, `sub_9D9770`,
   `sub_4DFB40`, `sub_4DEC70`, `sub_4DF5B0`, and `sub_5370F0`. Record return
   codes and which boundary is last reached.
2. At create, mid-lob, `start+500ms`, and the first failed/successful pickup
   scan, record object noun ID/class, object `+24` XYZ, `+97`, `+664`, `+744`,
   noun `+148`, lob start/duration/destination, client simulation clock, and
   the server's stored source/destination.
3. For every `FindGoodMeleePosition` failure, record both footprint radii, the
   initial and rotated candidates, which virtual collision/projection call
   failed, the projected displacement, and the `sub_9E91A0` result. Repeat on
   open terrain and beside an obstacle.
4. Correlate every inbound `0xa8` action response around the failure with
   response type, byte sync token, object ID, user data, and server log branch.
   Determine whether a response for an earlier action clears the newly staged
   type 9, including after one-byte token wrap.
5. Obtain a retail build-103 catalyst drop capture to settle publication of
   object movement type `4`, final landing position, `0x94`/`0x9a` ordering,
   reliability/channel, and whether a post-lob object delta exists.
6. Repeat a manual click independently of chunk `705`. If manual clicks always
   send while automated pickups fail, the repair scope is conclusively
   placement/reachability rather than command classification.

No server behavior was changed during this investigation.
