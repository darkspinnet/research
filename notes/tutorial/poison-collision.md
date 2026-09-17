# TutorialPoisonCloud build-103 collision contract

## Scope and conclusion

This note uses only the build-103 `content.db`, payloads extracted with
`darkrun db bget`, decoded resources from the shipped packages, and
`bin/darkspinner/GameBin/Game.c`. Recap material was not inspected.

The short contract is:

- `TutorialBasicDiseased` has an authored footprint radius of **1.4**. Its
  authored target bounds are an axis-aligned **1 x 1 x 1 box**, from
  `(-0.5, -0.5, 0)` through `(0.5, 0.5, 1)`. The footprint radius is not added
  to the projectile radius by the projectile collision code.
- `TutorialPoisonCloud` uses `Ability_Fireball.Noun` as its projectile object.
  Its Lua collision `radius` is **1.0 at every scale tier**. The native shape
  branch is box-shaped, not a radial sphere test: the fireball noun supplies
  X/Y dimensions of 1/1 and native code replaces the third half-extent with
  the Lua radius, producing projectile half-extents **(0.5, 0.5, 1.0)** (full
  extents **1 x 1 x 2**) for this ability.
- Flight is not assigned a predictive intercept time for a moving target. At
  initialization the native projectile aims at the target's position at that
  moment. It advances through the simulation and tests the current physics
  scene each update. The authored travel budget is **12 distance units** and
  the authored speed table is **{6, 8, 10}**. Thus the unmodified, straight
  nominal range times are **2.0, 1.5, or 1.2 seconds**, according to the
  selected scale tier; target velocity does not select a different root or
  flight time.
- Dynamic-object collision is **overlap first, sweep second**. It is not a
  discrete simulation-overlap-only test. The overlap catches a projectile
  already intersecting an eligible object at the beginning of the update; if
  that finds none, a box sweep covers the update displacement.
- An eligible object result sets the object-hit flag and hit object ID. A
  world collision or crossing the stored endpoint sets a separate impact flag
  with no object ID. Either impact flag wins only while the monitored remaining
  range is still strictly positive. If actual accumulated travel reduces the
  remaining range to zero or below first, `WaitForProjectile` returns the
  range-expiry fallback `(object ID 0, false)`. Therefore an impact observed on
  the same poll that exhausts the distance budget loses the tie to fallback.

## Proven content evidence

### Content database identity and bytecode

`content.db` identifies the shipped packages as follows:

- `AssetData_Binary.package`: package row 1, 13,515 resources, SHA-256
  `faf7b72a27b7f6b6f2c497c9aa5379bd2f20b1e8fe7b59b0d3798a2232774f2b`.
- `ServerData.package`: package row 3, 1,101 resources, SHA-256
  `845ef186dcc752071fff8fa43385a0bcd88e14ff8230001bc26b75f44a30892f`.

The `TutorialPoisonCloud` compiled Lua is ServerData ordinal 627,
`content_source_resource.id=14143`, group `0x7153BBB1`, instance
`0x66D99FC9`, decoded size 1,721, and decoded SHA-256
`076202c42dec21692943d05402a1d42ef47a78ec224ba056fd947ad9e126eee3`.
It was extracted directly from `server_data.decoded_payload` with:

```text
darkrun db server_data bget decoded_payload where content_source_resource_id=14143 \
  --decode zlib --output bin/game/logs/TutorialPoisonCloud-build103.luac \
  --config bin/darkspinner/darkspin.toml
```

The resulting bytecode disassembly proves these assignments (constant and
instruction indices are included so this does not depend on a source-level
decompiler):

- PCs 67-68 assign `distance = 12`.
- PCs 69-75 assign `speed = {6, 8, 10}`.
- PCs 76-82 assign `radius = {1, 1, 1}`.
- PCs 104-109 assign `object = nUtil:GetAsset("Ability_Fireball.Noun")`.

The same chunk also proves the ability identity and use: it creates
`nAbility_TutorialPoisonCloud` from `nAbility_Projectile_Template`, registers
the name `TutorialPoisonCloud`, and uses the diseased attack animation.

### TutorialBasicDiseased noun

The case-folded package hash of `TutorialBasicDiseased` is `0x52C73D4D`.
`content.db` resolves five AssetData resources with that instance. The noun is
ordinal 3720, type `0x76A8F7D8`, decoded size 729, decoded SHA-256
`7fa30e9983ea108ef0462c91c4b7da8870c0982439dd37c94c58ae9c7d19211c`.
The companion ordinal 3718 explicitly names `TutorialPoisonCloud`, proving the
ability-to-actor association independently of the Lua animation name.

In decoded ordinal 3720:

- offset `0x14` is little-endian float `0x3FB33333`, i.e. approximately
  **1.4**, the actor footprint/collision-radius field;
- offsets `0x38..0x40` are minimum bounds `(-0.5, -0.5, 0)`;
- offsets `0x44..0x4C` are maximum bounds `(0.5, 0.5, 1)`;
- the resource names `TutorialBasicDiseased`, its non-player class, and
  `DefaultPhysics.prop`.

Those min/max fields prove the authored **1 x 1 x 1 axis-aligned target box**.
They also show why treating 1.4 as a circle to be Minkowski-summed with the
projectile radius would conflate two distinct noun fields.

### Projectile noun and radius

The Lua-referenced `Ability_Fireball.Noun` resolves to AssetData ordinal 3221,
type `0x76A8F7D8`, instance `0x5A5AAFAF`, decoded size 629, decoded SHA-256
`9011e75175f8624f04c666e2a6cb113fb94bfcd7775acdbcac0afd1f7ff85058`.
Its inline physics block beginning at offset `0x204` contains a byte shape
discriminator **1** at `0x205`, followed by dimensions **(1, 1, 3)** at
`0x209`, `0x20D`, and `0x211`. This is the discriminator consumed by the native
type-1 box branch described below. The ability's `radius=1` does not change
X/Y; native code uses half of the noun's first two dimensions and replaces the
third half-extent with the radius.

The decoded package evidence is retained under
`bin/game/logs/poison_collision_noun` and the full read-only package inventory
under `bin/game/logs/poison_collision_assetdata`.

## Proven native evidence

Line numbers refer to `bin/darkspinner/GameBin/Game.c`.

### Launch direction and range monitoring

`sub_A2EBA0` (`0x00A2EBA0`, lines 1460032-1460167) initializes the projectile.
When given a target object, lines 1460112-1460140 read the projectile and target
positions, normalize `targetPosition - projectilePosition`, and multiply that
direction by the configured speed. No target velocity is read and no
intercept quadratic is solved.

`sub_A0FBE0`, the native `WaitForProjectile` binding (lines
1434571-1434702), stores the projectile ID, its current position, and the
configured distance budget in the waiting-thread state (lines
1434692-1434700).

`sub_A08A50`, the wait callback (lines 1429261-1429326), resolves the
projectile each poll and subtracts the Euclidean distance from the previously
recorded projectile position from the remaining budget (lines
1429283-1429296). This is actual simulated travel, not target-relative travel
and not simply wall-clock `distance/speed` when the path or speed changes.

### Projectile collision shape

`sub_A2E020` (`0x00A2E020`, lines 1459490-1459516) constructs the type-1
query half-extents. It halves noun dimensions X and Y (lines 1459510-1459513).
It normally halves Z as well, but if projectile state offset `+124` is nonzero,
that state value replaces Z directly (lines 1459507-1459509). For this content,
the noun's `(1,1,3)` and Lua radius `1` therefore become `(0.5,0.5,1)`.

### Overlap plus sweep

`sub_A2F990` (`0x00A2F990`, lines 1460643-1460809) is the dynamic-object
collision query used from the projectile step:

1. For shape discriminator 1, lines 1460692-1460702 call the physics interface
   with the current position, the box half-extents, and current orientation to
   collect up to 32 overlaps.
2. Candidate objects are filtered by `sub_A2F7F0` (lines
   1460579-1460640): the eight-entry ignore set is checked first, the explicitly
   ignored/source IDs are rejected, and the noun collision/team mode determines
   whether same-team, different-team, or unrestricted candidates are eligible.
3. If no eligible overlap exists, lines 1460749-1460773 make the type-1 swept
   physics call using the same box, orientation, and the frame displacement.
   The accepted sweep record, including contact position, is copied at lines
   1460795-1460798.

Thus "simulation overlap" alone is incomplete: overlap is the first phase,
followed by a continuous sweep for that update.

Static/world collision is handled separately by `sub_A2E0E0`
(`0x00A2E0E0`, lines 1459520-1459648), which follows the same pattern: an
overlap at the current position and then a shape sweep over the displacement.

### Object hit, world/endpoint impact, and fallback

`sub_A2FC20` (`0x00A2FC20`, lines 1460817-1461160) advances this projectile
mode. Its decisive branches are:

- An eligible result from `sub_A2F990` stores the object ID and contact point
  and sets state byte `+16` (lines 1461057-1461076). This is the direct object
  hit.
- If there was no direct object hit and world collision is enabled, a successful
  `sub_A2E0E0` stores the impact point and sets state byte `+17` (lines
  1461080-1461107). This is a collision/impact but has no direct actor ID.
- If a stored endpoint exists, the endpoint branch fires only when
  `stepLengthSquared > distanceToEndpointSquared` (strict `>`, lines
  1461110-1461126). It snaps to the endpoint and also sets `+17`.

Finally, `sub_A08A50` applies the exact ordering visible at lines
1429290-1429309:

1. Subtract this poll's traveled distance.
2. If remaining distance is strictly `> 0`, update the previous position and
   inspect state bytes `+16` and `+17`.
3. If neither flag is set, keep waiting.
4. If either flag is set, return the stored object ID and boolean `true`.
   A direct object hit has the nonzero ID stored by the `+16` branch; a
   world/endpoint impact ordinarily has ID 0 from initialization.
5. If remaining distance is `<= 0`, return object ID 0 and boolean `false`
   without consulting the collision flags. That is range-expiry fallback and
   establishes the exact-tie behavior.

## Inference and limits

- The field names are absent from the decompiler. Calling noun offset `0x14`
  the footprint radius is supported by its position in the noun layout, its
  value, and the native `GetFootprintRadius` binding at lines
  1433618-1433619, but the weak binding body is not present in this C export.
- The target's authored min/max box is proven content. Connecting it to the
  registered target physics geometry is a native-layout inference: the
  projectile routine queries the physics scene rather than manually reading or
  summing the target's 1.4 footprint. Nothing in the inspected collision path
  performs a 2-D circle test with `1.4 + 1.0`.
- Which one of `{6,8,10}` is selected is the normal ability scale-tier choice,
  not a moving-target choice. The inspected sources prove the three authored
  speeds and prove that target velocity does not participate in native launch
  direction or range monitoring; they do not expose a source-level name for
  the tier selector in this path.
- Physics-interface method names are stripped. "Box overlap" and "box sweep"
  are identified from the type-1 discriminator, argument sets, result forms,
  and paired no-displacement/displacement calls. The control-flow conclusion
  (overlap first, swept query second) does not depend on those inferred method
  names.
