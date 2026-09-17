# Tutorial noun physics projection for `content.db`

## Result

Build 103 has enough source authority to project the tutorial combat nouns into
`content.db` without copying the current Go constants or assigning names to
unknown binary fields. The important distinction is that four different facts
have previously been described as a "radius" or "bounds":

1. `graphicsScale` is the noun float at decoded offset `0x14`. Native
   `GetGraphicsScale` reads this field and multiplies it by the object's runtime
   scale. It is not, in general, the footprint radius.
2. The noun bounds are the two vectors at `0x38..0x4c`. These are local-space
   minimum and maximum bounds. They are source data, but they must not be
   relabelled as an inline PhysX box when the noun names another physics
   geometry.
3. `GetFootprintRadius` is the authoritative gameplay radius used for range and
   projectile launch placement. Native `sub_9D70E0`, called by the Lua binding
   at `0x00A01190`, obtains it from the noun's geometry and runtime object scale.
   It does not return `graphicsScale` directly.
4. A noun can contain separate collision-query geometry. In particular,
   `Ability_Fireball.Noun` has an inline type-1 box with dimensions `(1,1,3)`;
   these are not its noun bounds.

The recovered values are:

| Noun | `graphicsScale` | Footprint radius | Local noun minimum | Local noun maximum | Physics geometry | Projectile launch distance |
| --- | ---: | ---: | --- | --- | --- | ---: |
| `TutorialBasicPoison.Noun` | `1.3` | `1.3` | `(-0.5,-0.5,0)` | `(0.5,0.5,1)` | `builtins!sphere`, `DefaultPhysics.prop` | not applicable: `TutorialPoisonMelee` is melee |
| `TutorialBasicDiseased.Noun` | `1.4` | `1.4` | `(-0.5,-0.5,0)` | `(0.5,0.5,1)` | `builtins!sphere`, `DefaultPhysics.prop` | `1.4` for `TutorialPoisonCloud` |
| `TutorialBasicRanged.Noun` | `1.45` | `1.45` | `(-0.5,-0.5,0)` | `(0.5,0.5,1)` | `builtins!sphere`, `DefaultPhysics.prop` | `1.45` for `TutorialPlasmaLightning` |
| `TutorialSloth.Noun` | `1.3` | `1.3` | `(-0.5,-0.5,0)` | `(0.5,0.5,1)` | `builtins!sphere`, `DefaultPhysics.prop` | not applicable: `TailZap` is melee |
| `TutorialSpecialOne.Noun` | `2.3` | `2.3` | `(-0.5,-0.5,0)` | `(0.5,0.5,1)` | `builtins!sphere`, `DefaultPhysics.prop` | `2.3` for each `BurstShot` projectile |
| `PC_EL_Rogue.Noun` (the tutorial Blitz hero) | `1.6` | `0.8` | `(-0.5,-0.5,0)` | `(0.5,0.5,1)` | `builtins!sphere`, `DefaultPhysics.prop`; authored base sphere radius `0.5` | not applicable to the recovered tutorial enemy abilities |
| `PC_LF_Mage.Noun` (the tutorial Sage hero) | `1.65` | `0.825` | `(-0.5,-0.5,0)` | `(0.5,0.5,1)` | `builtins!sphere`, `DefaultPhysics.prop`; authored base sphere radius `0.5` | not applicable to the recovered tutorial enemy abilities |
| `Ability_Fireball.Noun` | `1` | not used as a caster | `(-0.5,-0.5,-0.25)` | `(0.5,0.5,0.5)` | inline type-1 box; see below | not applicable: this is the launched object |

“Projectile launch distance” is not another AssetData field. The projectile
template creates the projectile at

```text
caster position + caster facing * GetFootprintRadius(caster)
```

Consequently it should be read from `noun_physics.footprint_radius`, not stored
as a second mutable column. The `not applicable` cases must remain absent at
the ability layer rather than becoming a misleading zero.

## Source identities

All eight nouns are type `0x76A8F7D8`, group `0`, in
`AssetData_Binary.package` (`content_source_package.id=1`). The exact source
rows already present in the build-103 database are:

| Noun | Source resource ID | Package ordinal | Instance | Decoded size | Decoded SHA-256 |
| --- | ---: | ---: | --- | ---: | --- |
| `TutorialBasicPoison.Noun` | `6674` | `6673` | `0x09A3AC7F` | `703` | `ab0ed9b141763a95afae20e6699f6e0f07e7a8ad81ffdc0f5c9b0be674336d14` |
| `TutorialBasicDiseased.Noun` | `3721` | `3720` | `0x52C73D4D` | `729` | `7fa30e9983ea108ef0462c91c4b7da8870c0982439dd37c94c58ae9c7d19211c` |
| `TutorialBasicRanged.Noun` | `2668` | `2667` | `0x3AEC5FE2` | `690` | `f3530e2d3f0d7f346b906ace41dac76457fc4c1cab44d5f175aaf69e3362c82c` |
| `TutorialSloth.Noun` | `3340` | `3339` | `0xF5A88155` | `690` | `a2bb19651e33a6536275472e5dd3a5b0f82b15803fc49163d867db03e2cc71a1` |
| `TutorialSpecialOne.Noun` | `3200` | `3199` | `0xCC7ECBE0` | `707` | `7f701d8df5762af189f9d5f6c325fee149d2009dd5a21fde3ff4149fa52d739a` |
| `PC_EL_Rogue.Noun` | `4725` | `4724` | `0xA823E2E9` | `771` | `7aa6ab3a11d78d8ee830d4a2a7bd85e1ea49bf1d3c926d1710c68b67ff0bd4fe` |
| `PC_LF_Mage.Noun` | `11210` | `11209` | `0x1D521F70` | `777` | `0232b342f3c1622b2eabcfefb0b1205abd6c423b9c8ca5888dac7d3467a8a1b1` |
| `Ability_Fireball.Noun` | `3222` | `3221` | `0x5A5AAFAF` | `629` | `9011e75175f8624f04c666e2a6cb113fb94bfcd7775acdbcac0afd1f7ff85058` |

The noun row's source-resource FK is therefore exact; the builder does not need
to rediscover identity from a display string. Focused decoded copies used for
this comparison are under `bin/game/logs/tutorial_physics_assetdata`; the later
Sage addition is retained as `bin/game/logs/sage-noun/PC_LF_Mage.Noun.bin`.

## Exact source fields

### Common noun fields

The following fields are safe builder inputs because their byte positions and
native consumers are known:

| AssetData field | Decoded representation | Projection |
| --- | --- | --- |
| `graphicsScale` | little-endian `float32` at `0x14` | `graphics_scale` |
| local bounds minimum | three `float32` values at `0x38`, `0x3c`, `0x40` | `bound_min_x`, `bound_min_y`, `bound_min_z` |
| local bounds maximum | three `float32` values at `0x44`, `0x48`, `0x4c` | `bound_max_x`, `bound_max_y`, `bound_max_z` |
| physics geometry reference | variable string; `builtins!sphere` for the actors and `null` for Fireball | `geometry_reference` |
| physics properties reference | variable string; `DefaultPhysics.prop` for all seven | `property_reference` |

The offsets are offsets in the decoded type-`0x76A8F7D8` payload, not offsets
in the compressed DBPF member. The strings occur after the fixed noun body and
must be decoded through the noun's string slots; a builder must not search the
payload for arbitrary printable substrings.

`footprint_radius` is a normalized, source-derived field rather than a direct
copy from `0x14`. That difference matters for Blitz: `graphicsScale=1.6`, while
the authored `builtins!sphere` base radius is `0.5` and the recovered footprint
is `0.8`. It also prevents a future noun with a non-spherical or differently
sized geometry from silently treating visual scale as gameplay radius.

The local minimum/maximum vectors are worth retaining even when a geometry
reference is present. They are used by the native fallback footprint branch and
by graphics/culling consumers. They do not authorize replacing
`builtins!sphere` with an axis-aligned collision box.

### `Ability_Fireball.Noun`

Fireball has two different sets of dimensions and both should be test-visible:

- noun bounds: minimum `(-0.5,-0.5,-0.25)`, maximum `(0.5,0.5,0.5)`, hence
  size `(1,1,0.75)`;
- inline physics block beginning at decoded offset `0x204`: shape discriminator
  byte `1` at `0x205`, followed by dimensions `1`, `1`, `3` at unaligned
  little-endian float offsets `0x209`, `0x20d`, and `0x211`.

Native projectile collision selects the discriminator-1 box branch and halves
those three dimensions, normally producing half-extents `(0.5,0.5,1.5)`.
`TutorialPlasmaLightning` and `BurstShot` inherit projectile `radius=0`, so
those are their query half-extents. `TutorialPoisonCloud` authors Lua
`radius={1,1,1}`; native code preserves Fireball X/Y and replaces the third
half-extent with that radius, producing `(0.5,0.5,1)` (full dimensions
`1 x 1 x 2`). The ability override belongs to an ability projection, not to
the Fireball noun row.

The same inline block contains a creature-collision radius `1` and an
other-collision radius `0.25`. Those two proven semantic values should be child
shape rows with roles `creature_collision` and `other_collision`. Other floats
in the block are deliberately excluded until their native meaning is proven.

## Proposed schema

Use singular table and column names in line with the repository SQL convention:

```sql
CREATE TABLE noun_physics (
    id INTEGER PRIMARY KEY,
    content_source_resource_id INTEGER NOT NULL UNIQUE
        REFERENCES content_source_resource(id) ON DELETE CASCADE,
    asset_name TEXT NOT NULL UNIQUE COLLATE NOCASE,
    graphics_scale REAL NOT NULL CHECK (graphics_scale > 0),
    footprint_radius REAL NOT NULL CHECK (footprint_radius >= 0),
    bound_min_x REAL NOT NULL,
    bound_min_y REAL NOT NULL,
    bound_min_z REAL NOT NULL,
    bound_max_x REAL NOT NULL,
    bound_max_y REAL NOT NULL,
    bound_max_z REAL NOT NULL,
    geometry_reference TEXT NOT NULL,
    property_reference TEXT NOT NULL,
    source_size INTEGER NOT NULL CHECK (source_size > 0),
    source_sha256 TEXT NOT NULL,
    source_payload BLOB NOT NULL,
    CHECK (bound_min_x <= bound_max_x),
    CHECK (bound_min_y <= bound_max_y),
    CHECK (bound_min_z <= bound_max_z)
);

CREATE TABLE noun_physics_shape (
    id INTEGER PRIMARY KEY,
    noun_physics_id INTEGER NOT NULL
        REFERENCES noun_physics(id) ON DELETE CASCADE,
    content_source_resource_id INTEGER NOT NULL
        REFERENCES content_source_resource(id) ON DELETE CASCADE,
    ordinal INTEGER NOT NULL,
    shape_role TEXT NOT NULL,
    shape_kind TEXT NOT NULL,
    dimension_x REAL,
    dimension_y REAL,
    dimension_z REAL,
    radius REAL,
    source_offset INTEGER NOT NULL,
    UNIQUE (noun_physics_id, ordinal),
    UNIQUE (noun_physics_id, shape_role),
    CHECK (
        (shape_kind='box' AND dimension_x > 0 AND dimension_y > 0
            AND dimension_z > 0 AND radius IS NULL)
        OR
        (shape_kind='sphere' AND radius > 0 AND dimension_x IS NULL
            AND dimension_y IS NULL AND dimension_z IS NULL)
    )
);
```

`noun_physics_shape.content_source_resource_id` intentionally repeats the exact
source FK on each normalized child fact. For these rows it must equal the
parent's source ID; a builder verification enforces that invariant. This keeps
every geometry fact directly traceable without introducing an FK to a guessed
resource for the textual `DefaultPhysics.prop` reference.

The initial child rows are only Fireball's proven inline geometry:

| Ordinal | Role | Kind | Dimensions/radius | Source offset |
| ---: | --- | --- | --- | ---: |
| `0` | `projectile_query` | `box` | dimensions `(1,1,3)` | `0x205` discriminator; data at `0x209` |
| `1` | `creature_collision` | `sphere` | radius `1` | inline physics block |
| `2` | `other_collision` | `sphere` | radius `0.25` | inline physics block |

Do not add synthetic sphere child rows for the six actors. Their source names
`builtins!sphere`; it does not inline the same shape record as Fireball. The
normalized footprint and the retained geometry reference are sufficient until
the built-in geometry record itself is projected from an independently
identified source.

## Builder projection

The builder can remain deterministic and narrow:

1. Resolve the seven exact `content_source_resource` rows by package ID, type,
   group, instance, decoded size, and decoded SHA-256 from the identity table
   above. A missing or duplicate match is a build error.
2. Decode each DBPF member and require type `0x76A8F7D8`. Parse the fixed noun
   body and its defined string slots.
3. Copy `graphicsScale`, local minimum/maximum, geometry reference, and physics
   property reference. Reject non-finite floats, inverted bounds, and a string
   slot that runs outside the decoded payload.
4. Derive the six actor footprints through the recovered geometry semantics,
   producing exactly `{1.3,1.4,1.45,1.3,2.3,0.8}` in table order above. Do not
   use a blanket `footprint_radius=graphics_scale` rule.
5. For Fireball only, require the exact inline block discriminator and decode
   the three proven shape rows. Store no meaning for the remaining unknown
   floats.
6. Insert the source size, decoded SHA-256, and compressed decoded source
   payload using the same provenance convention as `level` and
   `level_marker_set`.

The scope is deliberately a noun/physics projection. Ability-to-noun links and
ability `radius` overrides can later reference `noun_physics.id`; they should
not mutate noun geometry during content building.

## Verification cases

The builder tests should cover all of the following:

1. **Exact identity:** all seven source-resource IDs, ordinals, instances,
   sizes, and decoded hashes match the table above; each FK joins to package
   `AssetData_Binary.package`, type `0x76A8F7D8`, group `0`.
2. **Actor values:** the five enemy rows and Blitz reproduce the exact scales,
   footprints, bounds, `builtins!sphere`, and `DefaultPhysics.prop` values in
   the result table.
3. **Scale is not footprint:** Blitz must assert `graphics_scale=1.6` and
   `footprint_radius=0.8`. This catches the tempting but invalid direct copy of
   offset `0x14`.
4. **Launch derivation:** Poison Cloud, Plasma Lightning, and Burst Shot resolve
   launch distances `1.4`, `1.45`, and `2.3` from their caster noun rows.
   Poison melee and TailZap have no projectile launch record.
5. **Bounds are not shape dimensions:** Fireball asserts noun size
   `(1,1,0.75)` and inline box dimensions `(1,1,3)` simultaneously.
6. **Projectile query dimensions:** radius-zero abilities produce half-extents
   `(0.5,0.5,1.5)`; Poison Cloud's radius-one override produces
   `(0.5,0.5,1)` without changing the stored noun shape.
7. **Collision roles:** Fireball has exactly one `projectile_query` box, one
   radius-`1` `creature_collision` sphere, and one radius-`0.25`
   `other_collision` sphere. No actor receives a fabricated inline shape row.
8. **Provenance integrity:** every child source FK equals its parent source FK;
   deleting a source resource cascades through both projected tables; an
   orphan or mismatched child is rejected by builder validation.
9. **Malformed content:** wrong type, duplicate source identity, truncated
   fixed body, unterminated string slot, non-finite float, inverted bounds,
   wrong Fireball discriminator, or changed decoded hash fails the build with a
   field-specific error.
10. **Determinism:** two builds from the same packages produce identical row
    ordering, values, hashes, and payloads.

These cases are sufficient to replace the tutorial physics constants with a
future database-backed reader without changing gameplay semantics, while
keeping unsupported field meanings out of the schema.
