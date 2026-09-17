# 1-1 final leader identity and presentation (build 103)

## Result

The original low-band `zelems_1` final leader is implementation-ready as the
concrete noun **`ZelemSpecialHaster_Captain.Noun`**, not as a generic special
proxy. Its noun hash is `0xd86fdcdd` and its packaged instance is
`0xfc016279`.

The strongest supported presentation policy is:

1. create that noun at the authored director-boss anchor, marker row `126389`,
   `(948.3736, 674.0089, 0.0880)`;
2. retain its noun-authored display name `Illust the Accelerator` and its two
   class-authored affixes in exact order:
   `Swift.NPCAffix`, then `Aura_Swift.NPCAffix`;
3. let those affixes localize to the visible line `Swift Aura, Swift`;
4. use the noun-linked `ZelemSpecialHaster.AIDefinition` and phase rather than
   an invented boss AI;
5. publish ordinary object, combatant, attribute, modifier, and director state.
   Do not invent a boss-name, health-bar, mutation-name, or effect packet.

The identity is exact for difficulty band `1-24`, which is the authored band
used by the original 1-1 special entry. The surviving walkthrough independently
shows the same name, affix order, Haster model, large target HUD, and adds.

What remains absent is the retail server operation that selects this special
entry for row `126389`. The level's reflected `boss` pool is empty and the boss
marker contains no direct noun reference. This missing selector does not make
the noun a guess: the level pool, the noun's own class data, and the recorded
encounter all converge on the same identity. It only leaves the authoritative
spawn transaction and its higher-band reuse policy unresolved.

## Provenance

The authoritative runtime database is
`bin/darkspinner/darkspin/cache/content.db`. Its manifest reports
`source_build=103`, `content_release=build-103-content`, recipe `35`, and input
fingerprint
`b088d6767e4485d90ff108c4eb091dd729b5cc8604b99b459ccc94cd1223b3ec`.

Primary immutable sources used here are:

| Source | SHA-256 |
| --- | --- |
| `bin/game/Data/AssetData_Binary.package` | `faf7b72a27b7f6b6f2c497c9aa5379bd2f20b1e8fe7b59b0d3798a2232774f2b` |
| `bin/game/Data/ServerData.package` | `845ef186dcc752071fff8fa43385a0bcd88e14ff8230001bc26b75f44a30892f` |
| `bin/game/GameBin/Game.exe` | `3c7248a4c6614450290a66c22ab7fcda86052b29a7b555e834d7c23f9b73ee5b` |
| `bin/game/GameBin/Game.c` | `1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb` |
| `bin/video/walkthrough/1-1/1-1.mkv` | `363f22ac2dc1d541623c55103505df42d651763b193a0285a8d40917a8831fda` |

Focused decoded copies are under `bin/game/logs/1-1-boss`. They are inspection
artifacts; `content.db` and the packages remain authoritative.

## Exact identity chain

### Level pool and final anchor

`content.db.level_director_entry.id=833` belongs to level `56`, `zelems_1`.
It is configuration ordinal `1`, now identified by reflection order as the
`special` pool, entry ordinal `0`, with noun
`ZelemSpecialHaster_Captain.Noun`, inclusive difficulty `1-24`, and
`is_horde_legal=1`.

The final cluster is marker set `1793`,
`zelems_1_design_spawners.Markerset`. Its relevant rows are:

| Row | Authored ID | Role |
| ---: | ---: | --- |
| `126384-126387` | `2145860737`, `2145860735`, `2145860738`, `2145860736` | four `boss triggered -> HordeSpawner_Register` add anchors |
| `126388` | `258231375` | colocated `LevelExitPoint` |
| `126389` | `223774364` | `SpawnPoint_DirectorBoss.Noun-876848874`, boss listener/support-unlock anchor |

The marker noun is an anchor/controller noun, not Illust's combat noun. The
four horde markers are add positions, not alternate leader identities.

### Noun and class override

The package stores two resources with instance `0xfc016279`:

| DB resource / ordinal | Type | Decoded size / SHA-256 | Exact relevant data |
| --- | --- | --- | --- |
| `6980` / `6979` | `0x76A8F7D8` noun | `723`; `e3aca654bb5584109b31a56b2cd56a48c82dc9e302a1372b50949e847ccf8fc0` | `ZelemSpecialHaster_Captain.NonPlayerClass`, `ZelemSpecialHaster.CharacterAnimation`, `ZelemSpecialHaster.AIDefinition`, `builtins!sphere`, `DefaultPhysics.prop`, `npcCreature` |
| `6979` / `6978` | `0xD117AFCA` class attributes | `514`; `5cb418c83b4e68dd477da258f07a820726abcd6b70308d18845613fca8af8634` | literal `Illust the Accelerator`, `AssetStrings!0x9214772A`, parent/base `ZelemSpecialHaster.ClassAttributes`, then `Swift.NPCAffix` and `Aura_Swift.NPCAffix` |

The noun authors graphics scale `2.75`; with the proven built-in sphere base
radius `0.5`, its content footprint is `1.375`. Object-create scale should
remain `1.0` so the noun scale is not applied twice.

This is a noun-specific class override, not a final-arena marker override.
Wherever this noun is constructed, its own class data supplies Illust's name
and affixes. The final marker does not contain the string `Illust`, an affix
list, or a replacement name.

The locale key is independently present in all five installed languages. The
English row is `localization_text.id=10602`, table `1882572231`, key
`0x9214772a`, text `Illust the Accelerator`. The literal in the class payload
and the locale row agree exactly.

### Affix records and visible order

The class payload's two affix references resolve to real type-`0x7CA6C6C9`
package records:

| Class reference | Asset hash | DB resource / ordinal / instance | Decoded SHA-256 | Registered modifier | English label |
| --- | ---: | --- | --- | --- | --- |
| `Swift.NPCAffix` | `0x578335f1` | `9462` / `9461` / `0x32a98a5c` | `21e09eb168e53614af85843722bcf684fdf9f0699dfc65a69c9e149285ae127e` | `Aura_Swift_NPCAffixModifier` | locale key `0x347f83f7`: `Swift Aura` |
| `Aura_Swift.NPCAffix` | `0x2d9552df` | `3251` / `3250` / `0x863f5b46` | `c6ec5265249d15fc92a413d37340fd9c5b5893a4fd988d92c932e5634abb8fbe` | `Swift_NPCAffixModifier` | locale key `0x863f5b46`: `Swift` |

The apparently crossed modifier names are authored data, not a transcription
error. Consequently the class order `Swift.NPCAffix,
Aura_Swift.NPCAffix` renders exactly `Swift Aura, Swift`, matching
`frame-840.png`. Do not alphabetize, reverse, randomly roll, or collapse the
two entries.

These are NPC affixes, not equipment `loot_affix` rows and not a mutation roll
inferred from one recording. Both references are hard-coded into Illust's
class-attribute resource.

## Affix runtime behavior

### Direct Swift modifier

`Swift_NPCAffixModifier` is chunk `157`, content resource `13688`, source
`Modifiers/0x97D53520.lua`, decoded size `818`, SHA-256
`e0fb5d7219d3faca3afb5b2dc626579362e1d98a17cde03cbc84b6113a1f366a`.
Its exact static/runtime facts are:

- registration GUID `SPID("Swift_NPCAffixModifier") = 0x3a7a7c2f`;
- `activationType=UniqueIrreplaceable`;
- `deactivationType=OnAgentDestroyed`;
- `saveOnDehydrate=true`;
- `movementSpeedIncrease=1`;
- activation calls `AddAttributeModifier` for
  `nAttributeType.MovementSpeedBuff` with that amount, then waits forever;
- deactivation has an empty Lua body; native modifier teardown still owns
  scoped-attribute cleanup.

This is a movement-speed attribute, not the Haster's ordinary combat haste
ability and not proof of an attack-speed or cooldown multiplier.

### Swift aura modifier

The aura family is chunk `283`, content resource `13825`, source
`Modifiers/0x8B0A4EAE.lua`, decoded size `3,283`, SHA-256
`30d5790360164cb116e6b77c147623ced3555b4ce89753e1a53ee52c9469639e`.
For the Swift branch it constructs and registers
`Aura_Swift_NPCAffixModifier`, GUID
`SPID("Aura_Swift_NPCAffixModifier") = 0x27f4bf41`, with child modifier
`SPID("Swift_NPCAffixModifier")`.

The shared aura factory authors ranked radii `[12, 16, 20]`, attaches a trigger
volume to the owner, and supplies `SimpleAuraFns` with
`IsNonDestructibleFriend(owner)` as the target predicate. It is
`UniqueIrreplaceable`, normally deactivates `OnAgentDestroyed`, has modifier
priority `Aura`, waits forever after attaching the volume, and detaches the
volume plus calls `CleanupAuras` on deactivation.

The dependency index leaves `Lua!AuraUtils.lua` with
`lua_dependency.target_lua_chunk_id=NULL`, but the package-identity audit maps
that authored import to chunk `739`. A retained decompile at
`bin/game/logs/recap_server-reference/game_server/res/data/serverdata/lua/AuraUtils.lua`
(SHA-256
`6b1e1d1a15b604949ebb2fb1098859b97be3834cbff5c7656a61a04f379097ed`)
recovers the helper lifecycle. `SimpleAuraFns(owner, childGUID, predicate,
rank)` creates one private `auraEffectModifierIds` table keyed by target object
ID. Its enter callback rejects a failed predicate and an already-retained
target, snapshots the aura owner, resolves an omitted rank once through
`nAbility.GetRank`, requests the child as
`RequestModifier(target, owner, childGUID, snapshot, rank)`, retains the
returned instance ID, and reports success. Its exit callback reports false for
an absent target; otherwise it calls `MarkForDelete(target, instanceID)`,
removes the table entry, and reports success. `CleanupAuras` deletes every
remaining retained child instance. The helper does not refresh or stack a
child on repeated entry. Campaign rank selection remains separate; rank one
and radius `12` are the low-band fallback until that mapping is recovered.

Neither the direct Swift chunk nor the aura chunk calls `AddEffect`, names a
`ServerEventDef`, or names an animation. The two affix package records likewise
contain no effect asset string. The blue shell/ring visible around Illust and
nearby actors is compatible with aura presentation, but the footage cannot
identify its effect resource or authorize a guessed server event. No explicit
effect packet is part of the recovered policy.

### Distinct Haster combat ability

The noun links AI resource `6406`, ordinal `6405`, type `0xEEEB0E31`, decoded
SHA-256
`d262644cff656cdf285420df2ed3f83b6238fc7c7b5105c349c98b14768de64a`.
It uses `nBehavior_Wander`, `nBehavior_Idle`, and
`ZelemSpecialHaster.Phase`.

Phase resource `6402`, ordinal `6401`, type `0x30728CE7`, decoded SHA-256
`034332692f30f2cb31b92d0baa88fd6769a8faf0307ff992e65eac4aaa928069`,
selects `AllyNeedsBuff -> CastZelemHasteBuff` before
`ZelemHasterAttack`. The combat ability applies `ZelemHasteBuff`; it is
separate from `Swift_NPCAffixModifier` and
`Aura_Swift_NPCAffixModifier`. Do not use the AI buff as a substitute for
either class-authored affix, and do not infer that the `Swift` label describes
the ordinary AI gambit.

## Name and health-bar presentation

There is no boss-name or boss-health-bar application packet in the recovered
contract.

Build 103 loads `HUD_NPCBar.swf` in `sub_41E1E0` (`0x0041E1E0`, C line
`164397`). The update path `sub_4218A0` (`0x004218A0`, C line `167654`):

- resolves the currently presented NPC object locally;
- reads its display text from noun/class content;
- reads current and maximum health through `sub_9D8F50` and `sub_9D8DF0` and
  computes their ratio;
- walks the object's modifier collection through `sub_4E44E0` and
  `sub_4E4730` to build affix text;
- calls Flash method `SetNPCInfo` with nine locally assembled arguments at C
  line `167963`.

Thus `Illust the Accelerator`, `Swift Aura, Swift`, and the wide green bar in
`frame-840.png` are client-local HUD presentation over replicated object state.
The server supplies the noun identity, HP/maximum-HP state, and any required
modifier instances. It does not send those strings or a filled bar image.

Director message `0x8B` is separate. Its fields include
`mbBossSpawned`, `mbBossHorde`, `mbBossComplete`, `mbHordeSpawned`, and
`mBossId`, but no name, affix string, HP, maximum HP, or health-bar flag.
`mBossId` is the exact director reference to the already-created leader; it is
not a name packet. The walkthrough's large bar must not be reproduced by
putting text or health into `0x8B`.

The exact captain base HP and the retail difficulty/captain/affix transforms
remain unresolved. The video's bar length is a ratio at one instant and cannot
calibrate maximum health. Use content-derived stats only when their exact
captain-class source is available; otherwise keep the current explicit
low-band stat fallback labelled as such.

## Packet ordering and dependency boundaries

The client binary is a receiver for these campaign messages and contains no
retail 1-1 sender. The following order is therefore divided between exact
causal requirements and conservative sender order:

| Order | Publication | Status and reason |
| ---: | --- | --- |
| `1` | Commit the authoritative leader object ID, noun, position, components, two affix roles, encounter membership, and director intent atomically. | Server transaction policy; prevents partial visible state. |
| `2` | `0x8C ObjectCreate` for noun hash `0xd86fdcdd`, object scale `1.0`, at row `126389`. | Exact dependency: every later packet naming the object requires it to exist. The receiver-valid enemy create shape is documented in `notes/campaign/1-1/native.md`. |
| `3` | `0x97 CombatantDataUpdate` current HP/mana and `0x96 AttributeDataUpdate` maxima/stats, plus any visibility/object baselines. | Exact data dependency for a meaningful HUD ratio; `0x97` versus `0x96` relative sender order is not recovered. |
| `4` | Establish `Swift_NPCAffixModifier` and `Aura_Swift_NPCAffixModifier` in the leader's modifier state. If the server must replicate creation explicitly, use `0xA2 ModifierCreated` only after `0x8C`. | The 37-byte `0xA2` layout and generic field producers are exact. What remains unknown is the Illust-specific attachment producer: whether construction requests these instances at all, and therefore which source, allocator result, rank, start time, and definition-derived bind state retail supplied. Do not send duplicate guessed instances. |
| `5` | Publish `0x8B DirectorState` selecting the leader ID and whichever active boss/horde flags the authoritative encounter actually owns. | Conservative reference order: publish `mBossId` after the object exists. Field meanings are exact; the retail dirty-mask combination and sender order are not. |
| `6` | Publish `0x99` target state before target-consuming turn, locomotion, first action, or ability packets. | Exact causal dependency shared with ordinary enemies. |

`ModifierCreated` is a packed 37-byte message containing target object,
modifier GUID, instance ID, duration, overdrive/rank state, stack count,
64-bit simulation-clock start time, source object, and the definition-derived
bind byte. The instance ID comes from the 2,048-entry generational modifier
pool. Native modifier creation emits the message before the modifier's
`Activate` callback. Explicit removal runs `Deactivate`, native scoped
cleanup, and then `0xA4 ModifierDeleted`. Those generic field producers are
fully recovered; only the absent Illust-specific attachment request prevents
their concrete values from being derived here.

No evidence orders the two class-authored affixes relative to each other on
the wire. Their **display order** is exact from the class list; their packet
order is not. No capture proves RakNet priority, reliability, ordering channel,
late-join replay, or whether `0x8B` preceded the component and modifier
baselines. Preserve the causal order above without claiming it is a retail
trace.

## Rejected candidates and interpretations

| Candidate | Disposition |
| --- | --- |
| `ZelemSpecialHaster.Noun` | Reject as the final leader noun. It is the ordinary low-band `captain`-pool Haster and carries the base family identity; Illust's exact named class override belongs to `_Captain`. |
| difficulty-eligible generic `special` proxy | Superseded for original 1-1. Entry `833` and the `_Captain` class payload identify Illust directly. Higher-band special entries remain separate later-chain content, not evidence that original 1-1 should change Illust's family. |
| `ZelemBoss.Noun` or another populated boss-pool noun | Reject. `zelems_1` has an empty reflected `boss` array, and no final marker links such a noun. |
| `SpawnPoint_DirectorBoss.Noun-876848874` | Reject as the actor. It is marker row `126389`, the spatial/listener anchor. |
| packet-carried name or health bar | Reject. `0x8B` contains only director state; `0x8C` carries noun identity, and `HUD_NPCBar` resolves name, affix text, and health ratio locally. |
| run-random `Swift Aura, Swift` mutation roll | Reject. Both affixes are literal members of Illust's class-attribute payload and resolve to exact package records/locales. |
| `CastZelemHasteBuff` / `ZelemHasteBuff` as the label source | Reject. That is the base Haster AI's separate combat gambit. The displayed labels resolve through the two NPC-affix records. |
| encounter-specific marker name override | Reject. The exact override is noun/class-specific; the final marker has no Illust string or replacement-name field. |
| visible blue shell as a recovered effect asset | Reject. No reviewed affix record or modifier chunk names an effect/animation, and the video cannot disambiguate overlapping aura, ability, and player effects. |
| exact maximum HP inferred from the long bar | Reject. The HUD displays a ratio, and retail captain/difficulty scaling is not recovered. |

## Remaining fallback boundaries

The following still require server policy or stronger evidence:

- the absent final-marker publisher/selector that chooses entry `833`, its
  retry/idempotency behavior, and policy outside original difficulty `1-24`;
- exact captain base HP and the difficulty/captain/affix transforms applied
  before `0x96`/`0x97` replication;
- campaign rank selection for the aura's `[12,16,20]` radii; use rank one and
  radius `12` only as the low-band fallback;
- whether class-authored affixes instantiate during noun construction or need
  an external modifier request, including the request source/rank and replay
  and teardown ownership; once such a request is identified, the exact `0xA2`
  instance, time, stack, and bind fields follow from the recovered native
  creator;
- any effect resource or server event associated with the aura presentation;
  emit none until one is recovered;
- final add selection/waves, leader/add clear predicate, boss/director dirty
  masks and transition order, success/failure races, multiplayer scaling, and
  late-join/reconnect replay.

These are authority gaps around a now-resolved identity. They are not reasons
to retain the former generic Haster proxy or random-affix policy for original
1-1.
