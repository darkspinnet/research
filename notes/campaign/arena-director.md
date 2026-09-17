# Post-teleport arena director boundary

This note isolates the boundary beginning at Cryos marker `1730050752`. It
separates retail content, packaged Lua, build-103 native/client behavior, and
darkspin's current compatibility implementation. That distinction matters:
the current `2s` first-wave timer and deterministic pairs are exact darkspin
behavior, but they are not recovered retail server policy.

## Result

The accepted boss-security teleport lands at marker `174193625`,
`(-347.57553,-224.60829,10.08803)`. That point is `2.76475` units from the
radius-`30` boss-director marker, so authoritative teleport commits entry into
the director volume. The director is marker `1730050752`, authored as
`SpawnPoint_DirectorBoss.Noun` at
`(-350.04453,-223.36415,10.08803)`. Its callback is
`DirectorTrigger_SpawnBoss` and its event is `horde triggered02`.

Exactly two authored listeners register that event:

| Marker | Authored identity | Position | Raw event/callback pair |
| ---: | --- | --- | --- |
| `3741961587` | `SpawnPoint_DirectorHorde.Noun-4` / `SpawnPoint_DirectorHorde.Noun` | `(-355.26929,-207.71077,10.08800)` | `horde triggered02` / `HordeSpawner_Register` |
| `3740600745` | `SpawnPoint_DirectorHorde.Noun-2` / `SpawnPoint_DirectorHorde.Noun` | `(-345.89496,-239.54680,10.16637)` | `horde triggered02` / `HordeSpawner_Register` |

These are the exact spawner identities and loci. They prove two registered
fan-out targets, not one creature per listener, listener delivery order, or
independent wave counters. The missing server director should treat them as
one level-instance-scoped listener set.

## Timing boundary

Let `T0` be acceptance of chunk `144`'s teleporter modifier request. Packaged
Lua chunk `349`, `TeleporterModifier`, proves this arrival sequence:

| Time | Proven operation |
| ---: | --- |
| `T0` | Stop and immobilize the entrant; play `character_teleport_out`. |
| `T0 + 0.500s` | Authoritative `TeleportObject` to marker `174193625`. |
| `T0 + 0.500s` | After an authored zero-duration yield, play `character_teleport_in`. |
| `T0 + 1.000s` | The final `0.500s` wait completes. |

Director contact becomes geometrically true at the authoritative teleport,
`T0 + 0.500s`; it does not need to wait for the final teleport-in presentation
wait. Contact delivery, re-entry suppression, and callback scheduling are
server authority and are not implemented by chunk `349`.

The first-wave timing evidence has two different scopes:

- **Exact darkspin boundary:** `tutorialHordeDirectorDelay` is exactly `2s`.
  The movement handler appends its arena-entry response, then schedules wave
  one with that delay. Spawn animation timestamps are anchored to the incoming
  packet source time plus `2000ms`. A focused build-103 run visibly entered the
  arena, showed the incoming alert, and rendered the first pair at both
  listener loci two seconds later.
- **Retail boundary unresolved:** no packaged Lua chunk, decoded director or
  listener record, client callback, or retained retail packet trace contains
  that `2s` delay. The design record has `waveOverride=4` and
  `challengeOverride=0`, but no recovered delay field. Accordingly, earlier
  notes calling `2s` "authored" overstate the evidence. It is the exact tested
  reconstruction timing, not an exact recovered retail-server timer.

The current `1.5s` clear-to-next-wave delay is likewise darkspin policy only.
Neither delay should be placed in Lua or client code when the server director
is reconstructed.

## Wave count, pool, and selection policy

The director's `SpawnPointDef` stores `challengeOverride=0` at decoded payload
offset `0x9bf` and `waveOverride=4` at `0x9c3`. IDA resolves those fields at
definition offsets `+0x14` and `+0x18`. Four requested waves are therefore
content-proven. The zero challenge override means the level does not supply an
arena-specific challenge value; it does not mean a zero spawn budget.

The level payload provides these horde-legal, difficulty-`1..100` noun records:

- ordinary director configuration: `TutorialBasicPoison.Noun`;
- first-visit configuration: `TutorialBasicDiseased.Noun`,
  `TutorialBasicRanged.Noun`, `TutorialBasicPoison.Noun`,
  `TutorialSloth.Noun`, and `TutorialSpecialOne.Noun`.

This is the exact eligible content, not an exact wave composition. The
recovered data does not supply per-wave counts, costs, a budget, listener
allocation, weighting, or a fixed noun order. It also does not prove whether
the server merges the ordinary record with the five first-visit records or
selects one configuration branch.

Build 103 contains a plausible native selection primitive, but its connection
to this callback remains unproven. Director spawn mode `5` filters a cached
candidate pool by difficulty, chooses uniformly with replacement, and caps the
result at `15` nouns. For accepted candidate index `n` (zero based), its
adjusted total is:

```text
(1 + 0.1*n) * (rawAcceptedCost + candidateCost)
```

If a candidate would exceed the caller's budget, it is removed from the
temporary pool and selection terminates; accepted candidates remain eligible.
The caller-supplied Cryos budget and the call edge from
`DirectorTrigger_SpawnBoss` have not been recovered, so this algorithm is a
candidate retail policy, not permission to derive exact waves from it.

For contrast, darkspin currently hard-codes one object at each listener:

| Wave | Listener `3741961587` | Listener `3740600745` |
| ---: | --- | --- |
| `1` | `TutorialBasicDiseased` | `TutorialBasicRanged` |
| `2` | `TutorialBasicPoison` | `TutorialSloth` |
| `3` | `TutorialBasicRanged` | `TutorialBasicPoison` |
| `4` | `TutorialSloth` | `TutorialSpecialOne` |

Those eight objects are a deterministic compatibility fixture. Live success
proves that build 103 can consume the nouns and four-wave terminal flow; it
does not prove retail count, pairing, or selection order.

## Incoming-alert ordering

The incoming alert is a `ServerEvent` (`0x9b`) whose reflected field `15`,
`clientEventID`, is `0x1d42121d` (`SPID("HordeIncoming")`). The build-103
receiver passes a nonzero field `15` to its local alert dispatcher, which
renders `Horde incoming!`. This is presentation and cannot activate a wave.

The exact current darkspin ordering for the accepted movement response is:

1. normal player movement acknowledgement;
2. immediate `ObjectTeleport` (`0x90`) to marker `174193625`;
3. the two arena health-obelisk packet groups;
4. incoming field-`15` `ServerEvent` in the same response batch;
5. scheduled wave-one object/state packets at `+2.000s`.

A focused run confirms the alert was consumed after entry and before the first
pair appeared. It does not recover retail ordering. In particular, darkspin's
handler currently bypasses chunk `349`, so its same-batch teleport/alert order
must not be combined with the Lua-proven `T0 + 0.500s` teleport timeline and
called a retail packet trace.

There is no packaged Lua reference that constructs this alert, and the inert
client callback does not synthesize it. The original sender and its precise
position relative to director contact, event fan-out, and wave activation
therefore belong to the missing server implementation. The client owns only
display after receipt.

## Authority ledger

| Boundary | Proven owner | What is and is not recovered |
| --- | --- | --- |
| Teleporter out/teleport/in | Packaged Lua chunk `349` invoking authoritative natives | Exact `0.5s / 0s / 0.5s` sequence is proven. Lua does not publish `horde triggered02`, select enemies, or send the incoming alert. |
| Director geometry and fan-out topology | Level content | Radius `30`, callback/event names, two listeners, their marker IDs and positions, `challengeOverride=0`, and `waveOverride=4` are proven. Content does not implement dispatch or waves. |
| Boss-marker client callback | Build-103 client native `DirectorTrigger_SpawnBoss -> sub_9FACF0` | It checks the authority/entrant branch, obtains the inline director address, and returns false on every path. It has no event fan-out, wait, spawn, state mutation, or packet send. |
| Listener callback | Missing server authority | `HordeSpawner_Register` occurs in neither the executable string corpus nor any of 1,029 packaged Lua chunks. Listener registration/delivery and lifetime are server-owned. |
| Candidate selection and creation primitives | Build-103 native simulation code | Mode `5` budget selection and `ActivateMinionSpawn`/`ActivateLieutenantSpawn` exist, but no recovered edge connects them to the Cryos callback. They are primitives, not proof of composition. |
| Spawn presentation | Packaged Lua chunk `655`, `SpawnModifier` | It stops locomotion, applies scoped `Immobilized=1`, adds `generic_spawn.ServerEventDef`, plays `horde_beam_in`, and waits `0.5s`; deactivation removes the effect and resets animation. The attaching/removal owner and object-create ordering are missing server behavior. |
| Incoming alert display | Build-103 client | Field `15 = 0x1d42121d` is receiver-proven. Alert construction and send order are not client behavior. |
| Horde state | Missing server authority, possibly replicated to client reflection | Client fields `mActiveHordeWaves` and `mbHordeSpawned` have initialization/getter accesses but no recovered local gameplay writer. They do not reveal the retail clear predicate. |

The missing server director must therefore own authoritative contact delivery,
once/re-entry policy, atomic fan-out to the two phase-scoped listeners, choice
of first-visit versus ordinary configuration, challenge/budget and RNG,
per-listener spawn allocation, object creation, spawn-modifier attachment and
release, initial aggro, active-wave tracking, clear-to-next-wave scheduling,
incoming-alert construction/order, cancellation and late-join reconstruction,
and terminal completion. None of those behaviors should be assigned to chunk
`141` (its `3.75s` job is a proven no-op), `ActivateHordeSpawn` (a build-103
no-op), chunk `349`, or the inert client boss callback.

## Evidence anchors

- `bin/game/logs/cryos-route/design.markerset`, SHA-256
  `651b17394e29be66a3e05c64649ff648e3274d9c9293b9008edd1f1c2ba95376`:
  raw marker identities and callback/event pairs.
- `bin/game/logs/cryos-route/tutorial-level.bin`, SHA-256
  `f97e3491d780e20514f0703e83cc5e6da5ebda0f07831c0eefaad2afab7ba10a`:
  ordinary and first-visit noun records.
- `bin/game/logs/ida-wave-override.log`, SHA-256
  `a98bf557d84b2e6ff3a627cd0983c91c81738af35b08eabacb83cbdfedcafd1`:
  `waveOverride` registration and payload audit.
- `bin/game/logs/ida-spawnpoint-fields.log`, SHA-256
  `6559e004c4bd257106a279ca1b0432aeac5f22dbf7bf3fb3353e2159aee5441f`:
  challenge/wave field offsets and client-consumer audit.
- `bin/game/logs/ida-director-xrefs.log`: boss callback registration, missing
  listener string, native spawn primitives, and mode-`5` selection audit.
- `bin/game/logs/ida-horde-state.log`, SHA-256
  `20d73bbc10296c21267dae1b4949165c9c8d2ab6a947c906fca42c733eb67da5`:
  reflected director fields and access scan.

## Remaining retail capture targets

1. Retain a retail trace from chunk `349`'s `ObjectTeleport` through the first
   enemy create; measure director-contact-to-alert and contact-to-wave times.
2. Recover the server callback or protocol state that supplies the Cryos
   challenge/budget and chooses the ordinary or first-visit noun configuration.
3. Recover listener delivery/allocation order, creation-to-`SpawnModifier`
   ordering, initial aggro, clear predicate, and cancellation/late-join state.

