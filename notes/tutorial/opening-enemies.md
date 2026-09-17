# Cryos opening-enemy activation ownership

## Result

The four current darkspin roles `11` through `14` map to fixed records in
`Game_Tutorial_cryos_1_ai.Markerset`. Those records own placement only.
They have no `level_event` row, trigger component, callback, or authored
per-marker contact radius.

All four use `TutorialBasicPoisonNoOrbs.Noun`, which resolves the same
`TutorialBasicPoison.NonPlayerClass` as ordinary tutorial Poison. Its exact
perception inputs are:

```text
aggroRange +0x38 = 0.0
alertRange +0x3c = 0.0
effective native radius = max(0.0, 0.0) = 0.0
```

The native point test is strict. With a zero supplied point radius, even the
marker center is not inside that circle. Consequently neither an `18`-unit nor
an `8`-unit sphere is authored opening-enemy ownership. Both radii in the
current Go path are compatibility policy. The missing authoritative route/AI
owner must insert the hostile player into each object blackboard and request
first aggro; that owner and its spatial or objective boundary are not present
in the recovered level events, packaged Lua, or build-103 client body.

## Marker identities

| darkspin object | AI ordinal | Authored marker ID | Marker name | Position |
| ---: | ---: | ---: | --- | --- |
| `11` | `13` | `1185902860` | `TutorialBasicPoison.Noun-18` | `(197.706924, -162.081482, 0.231011)` |
| `12` | `2` | `837454317` | `TutorialBasicPoison.Noun-19` | `(218.506149, -137.072830, 9.591080)` |
| `13` | `19` | `1999801501` | `TutorialBasicPoison.Noun-34` | `(227.769058, -121.498695, 9.945616)` |
| `14` | `6` | `1096871443` | `TutorialBasicPoison.Noun-35` | `(233.221115, -116.116280, 9.901413)` |

The nearby authored route markers do not close the ownership gap. The
ability markers at `(232.28690,-93.75159,10.52619)` and
`(233.67430,-96.28107,10.15000)` are five-unit lesson triggers, not enemy
activators. The client-owned audio/lesson volumes likewise publish only their
own tutorial callbacks.

## Which objects spawn together

The local build-103 run at `2026-07-19 14:03:56.735` proves that the locally
built darkspin binary used for that capture creates objects `11`, `12`, `13`,
and `14` in one response batch when the player reaches the first marker. The logged `ObjectCreate`
sequence begins with object IDs `0x0b`, `0x0c`, `0x0d`, and `0x0e` before the
scheduled first-aggro transition.

That trace is not retail spawn ownership. The packaged actors begin in
`nBehavior_Invisible`, so the retail walkthrough can prove when beam-in becomes
visible but cannot prove when the server created an invisible object. No
retail server packet capture or recovered server callback currently proves
whether `11`-`14` are created together, created with a larger route population,
or created separately. Do not turn visual beam-in order or marker proximity
into an object-creation grouping claim.

The safe recovered split is therefore:

```text
object creation/grouping       unresolved retail server owner
hostile target insertion       unresolved retail server owner, per object
Invisible deactivation         object behavior tree
first-aggro presentation       FirstAggro_BeamIn_Tutorial
pursuit and attack selection   object AI/combat owner after presentation release
```

## First aggro, pursuit, and attack eligibility

The exact object-local transition is:

```text
target insertion / first-aggro request
  -> old nBehavior_Invisible.Deactivate
       visible = true
       remove Intangible
       remove InvisibleToSecurityTeleporters
  -> FirstAggro_BeamIn_Tutorial.Activate
       visible/unstealthed
       animation = character_teleport_in
       wait 1.291667 s
  -> first-aggro release
       choose phase action
       pursue if out of range
       otherwise accept TutorialPoisonMelee immediately
```

The `1.291667s` wait belongs only to first-aggro presentation. It does not move
the actor and is not an extra locomotion delay after release. There is no
authored `100ms`, `1.1s`, or `1.5s` opening-enemy think/attack grace period in
the recovered content.

For `TutorialPoisonMelee`, initial activation becomes eligible only after the
first-aggro ability has released and the live target is inside the
surface-expanded envelope:

```text
center distance <= attacker footprint 1.3
                 + player footprint 0.8
                 + authored range 0.75
                = 2.85 units
```

If the target is farther away, pursuit begins after first-aggro release and
continues until that envelope is reached. An accepted attack starts
`cry_minn_lf_poison_attack1` immediately, revalidates range and target at the
`0.43s` hit continuation, applies rank-one `1-3` damage there, and owns a
`2.0s` release/cooldown. First-aggro completion makes an in-range attack
eligible; it does not itself apply damage or guarantee that an out-of-range
actor attacks.

## Evidence

- Runtime content DB: level `Game_Tutorial_cryos_1`, marker set row `291`,
  source SHA-256
  `5067ab36c70cf7aa726d755f39eccf9c7bcdec5f54f6d3eca289e5023cc6698f`.
- Decoded `TutorialBasicPoison.NonPlayerClass`: package ordinal `6674`; native
  schema labels fixed offsets `+0x38/+0x3c` as `aggroRange/alertRange`.
- Build-103 native perception: `sub_9E4040` and strict point test
  `sub_A0AEA0`.
- Packaged behavior/ability: `nBehavior_Invisible` chunk `865` and
  `FirstAggro_BeamIn_Tutorial` chunk `509` over the shared first-aggro
  template.
- Local walkthrough contact sheets generated under
  `bin/game/logs/cryos-opening-ownership`; these prove visible presentation
  only.
- Local runtime log:
  `bin/darkspinner/darkspin/logs/darkspinner.log`, create batch beginning at
  `2026-07-19T14:03:56.7355847-07:00`.
