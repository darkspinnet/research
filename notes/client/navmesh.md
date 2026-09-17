# Build 103 BFX navigation image

## Result

Build 103's type `0x1999AE0B` resource is a little-endian, multi-layer
navigation image. It is not the adjacent type `0x2699C284` PhysX collision
mesh. Each of the seven BFX plan layers contains:

1. a fixed `0x13c`-byte layer header;
2. a packed arena of convex, planar, variable-edge polygons;
3. a packed spatial acceleration image.

The polygon graph alone is sufficient for authoritative projection,
connected-component, reachability, and path-corridor queries. A server may
build its own immutable AABB index instead of reproducing BFX's packed
acceleration image.

This report uses file-relative offsets unless it explicitly says that an
offset is relative to a layer. All integers and floats are little-endian.

## Evidence and fixture identity

The primary fixture is:

| Property | Value |
| --- | --- |
| authored resource | `zelems_1!zelems_1.bfx` |
| package | `bin/game/Data/Levels.package` |
| DBPF ordinal/type/group | `118` / `0x1999AE0B` / `0xC8AC4657` |
| decoded file | `bin/game/logs/zelems_1-type-1999ae0b.bin` |
| decoded size | `634772` (`0x9af94`) bytes |
| SHA-256 | `7d3cee6acddbe626f18dbbc46bc5ab753d85a85e04cb8ee577a73a5e986416cc` |

The independent boundary check is `zelems_3`:

| Property | Value |
| --- | --- |
| package ordinal/type/group | `64` / `0x1999AE0B` / `0xC8AC4655` |
| decoded file | `bin/game/logs/zelems_3-type-1999ae0b.bin` |
| decoded size | `735468` (`0xb38ec`) bytes |
| SHA-256 | `e7ace3ba69562cea182f6ec4b5a6c91b3044c64eae6c236512cf75861319e0ca` |

Ordinal 119 is deliberately excluded from the format analysis. Its decoded
file `bin/game/logs/zelems_1-type-2699c284.bin` is 81,842 bytes and starts
`4e 58 53 01 4d 45 53 48`, or `NXS\x01MESH`. That is a cooked PhysX triangle
mesh, not a BFX navigation image.

The client evidence comes from
`bin/game/GameBin/Game.c` and its canonical IDB:

- `0x009EC890` accepts resource type `429501963`, which is `0x1999AE0B`,
  obtains its bytes, and passes them to the BFX resource loader.
- `0x00A3A7D0` and `0x00A47930` are endian conversion walks. They prove the
  variable polygon stride `52 + 24 * (flags & 0x7f)`, the three-float vertex
  field, and the locations of the packed flags.
- `0x00A3AE00` names the three layer shape inputs at header offsets
  `+0x18`, `+0x1c`, and `+0x20`: radius, step height, and height.
- `0x00A46250`, `0x00A462A0`, and `0x00A462E0` treat edge `+0x00` as the
  adjacent polygon and find the reciprocal edge.
- `0x00A3B2D0` and `0x00A3B380` use edge `i`'s vertex and edge
  `(i+1) % edgeCount`'s vertex as the edge segment.
- `0x009F0FF0` implements the fast area-reachability test from the plan-layer
  and graph identities.
- `0x009F58E0` projects both endpoints to areas and calls the path builder;
  the replay/API wrappers name the operation `CreatePolylinePath`.
- `0x00A503F0` maps module IDs to `bfxSystem`, `bfxPlanner`,
  `bfxPlanner3D`, `bfxMover`, `bfxMover3D`, `bfxBuilder`, and
  `bfxBuilder3D`.
- The builder/update family beginning at `0x00A4E5F0` rebuilds per-layer
  graph data, applies obstacle/shape metadata, and recomputes graph links.
  It is runtime builder state, not the disk polygon format.

## File framing

The first 48 bytes have stable boundaries in both fixtures:

```c
struct BfxEnvelope {                 // file +0x00, 0x18 bytes
    uint32_t zero0;                  // +0x00 = 0
    uint32_t envelopeVersion;        // +0x04 = 2
    uint32_t bytesAfterEnvelope;     // +0x08 = fileSize - 0x18
    uint32_t opaqueResourceToken;    // +0x0c; not CRC-32
    uint32_t zero1;                  // +0x10 = 0
    uint32_t zero2;                  // +0x14 = 0
};

struct BfxImageHeader {              // file +0x18, 0x18 bytes
    uint32_t imageVersion;           // +0x00 = 0x00010000
    uint32_t imageBytes;             // +0x04 = fileSize - 0x24
    uint32_t zero0;                  // +0x08 = 0
    uint32_t zero1;                  // +0x0c = 0
    uint32_t headerBytes;            // +0x10 = 0x1c
    uint32_t layerCount;             // +0x14 = 7
};
```

The first `BfxLayerHeader` begins at file offset `0x30`. Each following layer
begins at `previousLayerOffset + previous.layerBytes`. The final equality is:

```text
lastLayerOffset + lastLayer.layerBytes == fileSize
```

`opaqueResourceToken` differs between `zelems_1` (`0x777cf98b`) and
`zelems_3` (`0x63bcca30`). It does not equal standard CRC-32 or Adler-32 over
the obvious payload ranges. Treat it as opaque until its producer is found;
do not reject otherwise structurally valid content solely because a server
cannot recompute it.

Confidence: high for all offsets, sizes, versions, and layer walking; low for
the purpose of `opaqueResourceToken`.

## Layer header and footprint inputs

```c
struct Vec3f {
    float x;
    float y;
    float z;
};

struct BfxLayerHeader {              // 0x13c bytes before polygon arena
    uint32_t headerBytes;            // +0x000 = 0x1c
    uint32_t planLayer;              // +0x004 = 0..6
    uint32_t polygonArenaBytes;       // +0x008
    uint32_t layerBytes;              // +0x00c, start-to-start stride

    float buildScale;                // +0x010 = 2.0 in both fixtures
    float buildQuantum;              // +0x014 = 0.15 or 0.30
    float radius;                    // +0x018
    float stepHeight;                // +0x01c
    float height;                    // +0x020

    Vec3f boundsMin;                 // +0x024
    Vec3f boundsMax;                 // +0x030

    uint8_t zeroTable[0x80];         // +0x03c
    uint8_t opaqueSignature[0x80];   // +0x0bc
};                                  // polygon arena starts at +0x13c
```

The 128-byte signature is identical in every tested layer of both files. Its
SHA-256 is
`3584fdaac76c1a3264063106be12852cb51de030d85d803102ca026528158375`.
It is a useful corruption/version signature, but its algorithmic purpose is
not recovered.

The seven shape tuples are identical in both levels:

| Plan layer | Build quantum | Radius | Step height | Height |
| ---: | ---: | ---: | ---: | ---: |
| 0 | 0.15 | 0.71 | 0.50 | 1.75 |
| 1 | 0.15 | 1.06 | 0.50 | 3.00 |
| 2 | 0.15 | 1.24 | 0.50 | 2.50 |
| 3 | 0.30 | 1.41 | 0.50 | 4.00 |
| 4 | 0.30 | 1.59 | 0.50 | 5.00 |
| 5 | 0.30 | 1.77 | 0.50 | 3.50 |
| 6 | 0.30 | 3.89 | 0.50 | 9.00 |

These are separately baked navigation meshes. The client query APIs take a
`PlanLayer`; they do not pass an arbitrary footprint into polygon projection.
For a known gameplay archetype, use its authored plan layer. For an arbitrary
server footprint, a conservative fallback may choose a layer whose radius
and height are both at least the requested values. Reject if no such layer
exists. Do not select by radius alone: the height ordering is intentionally
non-monotonic.

Confidence: high for the shape field names because `0x00A3AE00` emits those
exact names and reads the corresponding runtime offsets; medium for the
names `buildScale` and `buildQuantum`.

## Polygon arena

The arena begins at `layerOffset + 0x13c` and contains no count or pointer
table. Walk it until exactly `polygonArenaBytes` bytes have been consumed.

```c
struct BfxPolygonDisk {              // fixed prefix, 0x34 bytes
    uint32_t runtime0;               // +0x00, zero in static fixtures
    uint32_t runtime1;               // +0x04, zero in static fixtures
    uint32_t runtime2;               // +0x08, zero in static fixtures
    uint32_t runtime3;               // +0x0c, zero in static fixtures
    Vec3f center;                    // +0x10
    float boundingRadius;            // +0x1c
    uint32_t userData;               // +0x20, 0xffffffff in fixtures
    uint32_t runtime4;               // +0x24, zero in static fixtures
    uint32_t meta0;                  // +0x28
    uint32_t meta1;                  // +0x2c
    uint16_t visitStamp;             // +0x30, transient
    uint16_t graphSlot;              // +0x32, low 10 bits used at runtime
    BfxEdgeDisk edge[edgeCount];      // +0x34
};

struct BfxEdgeDisk {                 // 0x18-byte stride
    uint32_t neighborOffset;          // +0x00; layer-relative, or 0
    Vec3f vertex;                    // +0x04
    uint32_t flags;                  // +0x10
    uint32_t traversalCost;           // +0x14
};
```

Decode the polygon metadata as:

```c
edgeCount      = meta0 & 0x7f;
graphIdentity  = (meta0 >> 7) & 0xffff;
planLayer      = (meta0 >> 23) & 0x1f;
isDynamic      = (meta0 >> 30) & 1;
polygonBytes   = 0x34 + 0x18 * edgeCount;
```

In both disk fixtures, `graphIdentity` is the unloaded sentinel `0xffff`,
`isDynamic` is zero, and the embedded plan layer agrees with the containing
layer. The loader assigns runtime graph identities. `visitStamp` and
`graphSlot` are also runtime bookkeeping and are zero in these two static
resources.

`center` is the polygon bounding-sphere center, not a vertex average.
`boundingRadius` equals the maximum three-dimensional distance from `center`
to any vertex to within `1.9e-6` across all 7,658 tested polygons.

### Adjacency and portals

`neighborOffset == 0` means a boundary edge. A nonzero value is relative to
the start of the containing `BfxLayerHeader`, not the polygon arena and not
the current polygon. It must land on the fixed prefix of another polygon.

For polygon `p`, edge `i` represents the directed segment:

```text
p.edge[i].vertex -> p.edge[(i + 1) % edgeCount].vertex
```

If it has a neighbor, exactly one edge in that neighbor points back to
`p`. The neighbor's two portal endpoints are byte-for-byte the reverse of
the source endpoints in both validation fixtures. This is the corridor
portal representation; there is no separate static portal array.

`traversalCost` is zero on every boundary edge and nonzero on every linked
edge in both fixtures. Client code at `0x00A3A950` derives the reciprocal
cost from polygon-center distance, per-polygon cost nibbles, and build scale.
Use the stored positive integer as the deterministic static A* edge cost.

The two observed edge flag values are `0xffff0000` and `0xffff8000`.
Code around `0x00A46400` and the path builder consumes further bitfields,
but their complete policy meaning is not proven. A first static server
implementation should preserve the field, accept the two observed values,
and not invent different traversal behavior for bit `0x8000` until a fixture
or client call site proves it.

Confidence: high for all record boundaries, offset bases, adjacency,
portal endpoints, and cost zero/nonzero semantics; medium for the
human-readable meaning of individual `meta1` and edge flag bits.

## Geometry and coordinate system

Coordinates are ordinary build-103 `Vec3f` values with `x`, `y`, `z` order
and Z as elevation. For example, `zelems_1` layer 0 spans:

```text
min = (-637.875, -162.675064, 0.421402)
max = ( 980.625,  746.325378, 33.575020)
```

`zelems_3` layer 0 spans:

```text
min = (-338.175018, -3204.525146, 0.396182)
max = (3373.424561,   277.875092, 35.574287)
```

The small Z range relative to X/Y and the marker/world transform call sites
establish Z-up. Do not flatten the geometry to XY during projection: a few
polygons are steep. Instead, derive a plane normal from the ordered vertices
and perform point-in-convex-polygon and closest-edge calculations in that
plane.

Every polygon in both fixtures is convex and planar. Maximum absolute
distance from its fitted plane is below `1.5e-4`; most are exactly horizontal.
Vertex winding is consistent within a polygon, but the server query does not
need to assume a particular normal sign.

Confidence: high.

## Spatial acceleration image

The spatial image immediately follows the polygon arena:

```c
struct BfxSpatialPrefix {            // 0x1c bytes
    Vec3f boundsMin;                 // +0x00
    Vec3f boundsMax;                 // +0x0c
    uint32_t treeImageBytes;          // +0x18
};

struct BfxTreeImage {                // treeImageBytes bytes
    uint32_t partitionOffset;         // +0x00
    float partitionBound0;            // +0x04
    float partitionBound1;            // +0x08
    uint8_t packedNodes[];            // +0x0c
};
```

The exact size identity is:

```text
layerBytes
  = 0x13c
  + polygonArenaBytes
  + 0x1c
  + treeImageBytes
```

Packed tree nodes follow the traversal used at `0x00A3A700`:

```c
// tag bit 31 set: a 4-byte leaf
polygonOffset = tag & 0x7fffffff;

// tag bit 31 clear: a 12-byte internal node
axis         = (tag >> 28) & 7;      // observed 0..2
farChildByte = tag & 0x0fffffff;     // relative to this node
nearChild    = this + 12;
bound0       = *(float *)(this + 4);
bound1       = *(float *)(this + 8);
```

Leaf polygon offsets use the same layer-relative identity as adjacency
offsets. The image contains partitioned packed trees; the first root and
child encoding are proven, but the top-level purpose of
`partitionOffset/partitionBound*` is not yet certain.

Reproducing this image is unnecessary for authoritative server behavior.
After validating its size and leaf offsets, build an immutable R-tree/BVH
over each polygon's bounding sphere or AABB. This avoids coupling the parser
to an only partially named acceleration policy while retaining identical
geometry and graph semantics.

Confidence: high for the prefix boundary, total size, node encodings, and
leaf offset base; medium for the partition fields.

## Cross-level boundary validation

The following values are obtained by walking the declared sizes and record
strides, not by searching for plausible floats:

### `zelems_1`

| Layer | Layer offset | Layer bytes | Arena bytes | Arena offset | Polygons | Edges | Directed links | Spatial bytes |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | `0x000030` | 102696 | 93260 | `0x00016c` | 569 | 2653 | 1202 | 9120 |
| 1 | `0x019158` | 102160 | 92788 | `0x019294` | 565 | 2642 | 1192 | 9056 |
| 2 | `0x032068` | 102204 | 92816 | `0x0321a4` | 566 | 2641 | 1200 | 9072 |
| 3 | `0x04afa4` | 81868 | 74368 | `0x04b0e0` | 448 | 2128 | 958 | 7184 |
| 4 | `0x05ef70` | 81868 | 74368 | `0x05f0ac` | 448 | 2128 | 958 | 7184 |
| 5 | `0x072f3c` | 81904 | 74356 | `0x073078` | 451 | 2121 | 962 | 7232 |
| 6 | `0x086f2c` | 82024 | 74284 | `0x087068` | 463 | 2092 | 962 | 7424 |

### `zelems_3`

| Layer | Layer offset | Layer bytes | Arena bytes | Arena offset | Polygons | Edges | Directed links | Spatial bytes |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | `0x000030` | 122860 | 111424 | `0x00016c` | 694 | 3139 | 1444 | 11120 |
| 1 | `0x01e01c` | 122860 | 111328 | `0x01e158` | 700 | 3122 | 1456 | 11216 |
| 2 | `0x03c008` | 121504 | 110116 | `0x03c144` | 691 | 3091 | 1434 | 11072 |
| 3 | `0x059aa8` | 94628 | 85848 | `0x059be4` | 528 | 2433 | 1104 | 8464 |
| 4 | `0x070c4c` | 94604 | 85824 | `0x070d88` | 528 | 2432 | 1104 | 8464 |
| 5 | `0x087dd8` | 96192 | 87236 | `0x087f14` | 539 | 2467 | 1124 | 8640 |
| 6 | `0x09f598` | 82772 | 74952 | `0x09f6d4` | 468 | 2109 | 950 | 7504 |

For all 7,658 polygons and 35,198 edges across these tables:

- the arena walk ends exactly at `arenaOffset + polygonArenaBytes`;
- every edge count is in `[3, 20]`;
- every nonzero neighbor lands on a polygon boundary;
- all 16,050 directed links have exactly one reciprocal link;
- all reciprocal portal endpoint pairs are exact reversed matches;
- every boundary edge has cost zero and every linked edge has positive cost;
- each polygon's embedded plan layer matches its containing layer;
- each layer size equation and the final file-size equation hold.

These invariants are stronger boundary evidence than matching the repeated
header signature alone.

## Authoritative query contract

The replay/API names around `0x00A4A230` through `0x00A4A850` and
`0x00A4B1C0` through `0x00A4B3B0` expose the relevant BFX contract:

```text
GetClosestArea(position, planLayer, pathSpec)
GetClosestReachableArea(position, startArea, pathSpec)
GetClosestReachableAreas(position, startArea, pathSpec, radius, maxCount)
IsAreaReachableFromArea(areaA, areaB, pathSpec)
CreatePolylinePath(startArea?, startPosition, goalArea?, goalPosition,
                   pathSpec, creationOptions)
NavProbe(startPosition, direction, distance, planLayer, pathSpec)
CheckCircleFit / CheckBoxFit / CheckTriangleFit
```

`PathSpec` contains at least:

```c
struct PathSpec {
    uint32_t obstacleMode;
    uint32_t obstacleBlockageFlags;
    uint32_t customGeometryMatchFlags;
    uint32_t linkUsageFlags;
    bool usePathSharingPenalty;
    float pathSharingPenalty;
    float maxSearchDistance;
};
```

Static server queries should expose explicit limits rather than silently use
the client defaults:

```go
type Footprint struct {
    PlanLayer  uint8
    Radius     float32
    StepHeight float32
    Height     float32
}

type ProjectionOptions struct {
    Footprint  Footprint
    MaxDistance float32
    ComponentID uint32 // optional reachability constraint
}

type Projection struct {
    Position    Vec3
    PolygonID   uint32 // layer-relative polygon offset
    ComponentID uint32
    Distance    float32
}

type PathOptions struct {
    Footprint        Footprint
    MaxProjectDistance float32
    MaxSearchDistance  float32
    MaxVisitedPolygons int
}
```

### Layer selection

Resolve `PlanLayer` before querying. If the caller supplies an authored
layer, verify that its header shape is compatible with the asserted
footprint. If it supplies only dimensions, select a conservative compatible
baked layer as described above. Never project on layer 0 and then claim
clearance for a larger footprint.

### Point projection

1. Reject non-finite coordinates and negative/non-finite limits.
2. Query the server-built polygon AABB index with a sphere or expanded AABB
   of `MaxDistance`.
3. For each candidate convex polygon:
   - derive its plane from non-collinear ordered vertices;
   - orthogonally project the point to that plane;
   - if the projected point is inside every directed edge half-space, use it;
   - otherwise use the closest point on every polygon edge segment.
4. Choose the candidate with smallest three-dimensional squared distance.
   Break exact ties by the layer-relative polygon offset for determinism.
5. Require `distance <= MaxDistance`.
6. If a component constraint is present, discard candidates outside it
   before choosing the nearest.

The baked layer already accounts for radius and height. Do not shrink each
polygon again by the same footprint.

### Connected-region identity

The on-disk `graphIdentity` is an unloaded `0xffff` sentinel, so it is not a
stable server region ID. Build connected components per plan layer from
reciprocal nonzero adjacency after parsing. Assign deterministic component
IDs by sorting components on their smallest layer-relative polygon offset.
Store the component ID beside each immutable polygon.

Both validation fixtures contain one adjacency component per layer. The
component algorithm is still required for malformed-content rejection,
other levels, and later dynamic/custom links. A static reachability query is
constant time after projection: same layer and same component means
reachable under the default static policy.

If custom BFX links are later imported, apply `linkUsageFlags` and obstacle
policy before computing policy-specific reachability. Do not treat the
PhysX mesh as an implicit link source.

### Path query

1. Project start and goal on the same selected layer.
2. Reject if projection fails or component IDs differ.
3. Run bounded A* on polygon offsets:
   - each nonzero reciprocal edge is a graph arc;
   - use `traversalCost` as the static arc cost;
   - use straight-line center-to-goal distance scaled no higher than the
     minimum observed cost-per-distance as an admissible heuristic, or use
     Dijkstra initially;
   - enforce `MaxSearchDistance` and `MaxVisitedPolygons`.
4. Reconstruct the polygon corridor.
5. Reconstruct each portal from the source edge vertex and its next vertex.
6. For a minimal authoritative implementation, return start, portal
   midpoints, and goal, then remove a midpoint only when a segment remains
   inside the recovered corridor. A later client-parity implementation may
   use a funnel/string-pull after projecting each local corridor section to
   its polygon plane.
7. Return both the polyline and polygon corridor. Admission logic should use
   the corridor/result status; movement presentation may use the smoothed
   polyline.

This avoids an unsafe global XY funnel on steep polygons while still
providing a valid deterministic path.

### Nearby reachable point and fit

Project with a component constraint and enumerate candidates by increasing
distance until one satisfies the selected baked layer. For an extra runtime
circle larger than the baked layer's radius, either select a larger baked
layer or perform an explicit additional distance-to-boundary test. Never
claim larger clearance from projection alone.

The client also exposes circle, box, and triangle fit operations. Those are
separate shape queries and should not be conflated with point projection.

## Parser rejection rules

Reject a resource if any of the following fails:

1. File size is at least `0x30`.
2. Envelope version is 2 and `bytesAfterEnvelope == fileSize - 0x18`.
3. Image version is `0x00010000` and
   `imageBytes == fileSize - 0x24`.
4. Reserved header words are zero, header size is `0x1c`, and layer count is
   nonzero and bounded (build 103 fixtures use 7).
5. The first layer is at `0x30`; each layer starts in bounds; its
   `headerBytes` is `0x1c`; its index equals its ordinal; and `layerBytes`
   advances without overflow.
6. All shape values and bounds are finite; radius and height are positive;
   bounds minimums do not exceed maximums.
7. The `0x80` zero table is zero. The opaque signature should match the
   build-103 signature when strict build-103 parsing is requested.
8. `0x13c + polygonArenaBytes + 0x1c <= layerBytes`.
9. Every polygon has at least 3 and at most 127 edges, its computed record
   size stays inside the arena, and the final record ends exactly at the
   declared arena end.
10. Centers, radii, and vertices are finite; radius is nonnegative; polygons
    have nonzero area and are planar/convex within a documented tolerance.
11. Embedded plan-layer bits equal the containing layer; static fixtures
    must not set the dynamic bit.
12. Every nonzero neighbor offset is layer-relative, lies in the arena,
    lands on a parsed polygon start, is not self-referential, has exactly one
    reciprocal edge, and has reversed identical portal endpoints.
13. Boundary/link cost consistency holds. Reject a linked edge with zero
    cost rather than allowing a zero-cost path cycle.
14. Spatial bounds are finite; `treeImageBytes` satisfies the layer size
    equation; internal node axes are 0..2; child offsets are aligned and
    in bounds; and leaf offsets land on parsed polygon starts. A server may
    ignore the tree's ordering after validating it.
15. The last layer ends exactly at EOF; trailing data is not accepted.

Use checked 64-bit arithmetic for every offset/size calculation before
converting to an index.

## Focused fixture tests

No implementation files are changed by this task. When the parser/query
package is added, these are the focused tests to create:

1. `TestBFXParseZelems1Header`
   - assert file/envelope sizes, seven layers, exact shape tuples, first layer
     at `0x30`, and EOF at `0x9af94`.
2. `TestBFXParseZelems3Boundaries`
   - assert every layer offset, arena size, polygon/edge/link count, spatial
     size, and EOF from the second table.
3. `TestBFXPolygonArenaConsumesDeclaredBytes`
   - walk all fourteen arenas with `0x34 + 0x18*n`; require exact ends.
4. `TestBFXAdjacencyIsReciprocal`
   - require all 16,050 directed links to land on polygon starts and find one
     reciprocal edge.
5. `TestBFXPortalEndpointsReverse`
   - compare both endpoint triples exactly for every reciprocal pair.
6. `TestBFXBoundaryAndLinkCosts`
   - require `(neighbor == 0) == (cost == 0)` for every fixture edge.
7. `TestBFXPolygonGeometry`
   - require finite, nondegenerate, convex, planar polygons and verify the
     stored bounding radius within `2e-5`.
8. `TestBFXProjectionInteriorEdgeAndOutside`
   - use one horizontal fixture polygon; test an interior vertical
     projection, exact edge projection, and rejection just beyond the max
     distance.
9. `TestBFXProjectionSteepPolygon`
   - choose a polygon with `abs(normal.z) < 0.2` and prove projection is 3D,
     not flattened XY.
10. `TestBFXConnectedComponentDeterminism`
    - parse a small synthetic arena with two islands, shuffle polygon input
      order in the index builder, and require component IDs ordered by
      smallest polygon offset.
11. `TestBFXPathUsesReciprocalPortals`
    - choose two fixture polygons separated by several arcs; require the
      corridor to start/end in the projected polygons and every transition
      to use an exact portal pair.
12. `TestBFXClearanceSelectsBakedLayer`
    - prove radius 1.1/height 2.4 selects a compatible layer, while an
      impossible height/radius combination rejects; include the
      non-monotonic height ordering.
13. `TestBFXRejectsMalformedOffsets`
    - mutate a neighbor to arena-relative, mid-record, self, and out-of-file
      offsets and require rejection.
14. `TestBFXRejectsTruncationAndOverflow`
    - truncate at every major boundary and mutate arena/layer/tree sizes near
      `UINT32_MAX`.
15. `TestBFXRejectsPhysXMesh`
    - pass ordinal 119's `NXS\x01MESH` fixture and require an envelope/version
      error before geometry parsing.

## Confidence summary and remaining unknowns

| Claim | Confidence | Basis |
| --- | --- | --- |
| BFX versus PhysX resource identity | High | DBPF type/group/ordinal plus incompatible headers |
| envelope/image/layer boundaries | High | exact size equations in two files and all layers |
| polygon and edge strides | High | exact arena walks plus client endian walker |
| convex vertices and 3D coordinates | High | all fixture polygons plus geometry call sites |
| adjacency offset base and reciprocity | High | all links land only when layer-relative; client reciprocal lookup |
| portal representation | High | client edge access plus exact reversed endpoints |
| plan-layer radius/step/height | High | named client diagnostics and identical fixture tuples |
| component derivation for static server | High | graph adjacency and client reachability identity |
| stored edge cost as static A* weight | Medium-high | boundary/link correlation and client cost builder |
| spatial prefix/node encoding | High | exact size equation and client traversal |
| spatial partition field names | Medium | behavior inferred, top-level partition policy unnamed |
| `buildScale`/`buildQuantum` names | Medium | stable values and builder use, no surviving symbol |
| opaque envelope token/signature purpose | Low | boundaries proven, producer not recovered |
| individual `meta1` and edge flag policy | Low-medium | bit consumers exist, complete enum names absent |

The unknown flag names do not block the static authoritative contract above.
They matter when custom links, dynamic obstacles, or exact mover parity are
added. Preserve them in parsed data and fail closed for unrecognized values
instead of assigning speculative behavior.
