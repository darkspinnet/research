# Quadra build-103 perception contract

## Result

`TutorialSpecialOne_Intro.Noun` does **not** author an 18-unit perception
perimeter. Its build-103 `NonPlayerClass` has both spatial inputs set to zero:

| Field | Native offset | Quadra value |
| --- | ---: | ---: |
| `aggroRange` | `+0x38` | `0.0` |
| `alertRange` | `+0x3c` | `0.0` |

The native perception helper uses
`max(nonPlayerClass.aggroRange, nonPlayerClass.alertRange)`, so Quadra's exact
authored perception radius is **0.0 world units**. In practical terms this noun
has no positive, content-authored radial first-aggro perimeter. The current
generic `tutorialEnemyAlertRadius = 18` movement sphere is not exact for
Quadra and should not be described as build-103-authored evidence.

The engine event which enters the normal first-aggro path is
**`Aggro_Trigger`**. Its registered callback adds the hostile object with flag
`2`; on the first aggro-list insertion the engine posts timed AI stimulus
**`0x20` (decimal 32)** for `10.0` seconds and marks first aggro consumed. That
stimulus is evaluated by the agent brain whose Quadra AI definition points its
first-aggro slot at `FirstAggro_SpecialOne`.

This does not prove an omitted Sage/director edge which emits
`Aggro_Trigger`. It proves that a generic positive movement radius is not that
edge. An external engine/director event or explicit alert is still required to
activate Quadra from its authored zero range.

## Packaged content

The five same-instance resources for SPID base
`TutorialSpecialOne_Intro` (`0x0a88b6a3`) were decoded from
`AssetData_Binary.package`. The relevant records are:

- type `0x76a8f7d8`, the noun, which explicitly links
  `TutorialSpecialOne_Intro.NonPlayerClass` and
  `TutorialSpecialOne_Intro.AIDefinition`;
- type `0x30728ce7`, the `NonPlayerClass` record, which owns `BurstShot` and
  stores zero at fixed offsets `0x38` and `0x3c`;
- type `0xeeeb0e31`, the AI definition, whose first-aggro field at `+0x28` is
  SPID `0x4bbf19fb`, `FirstAggro_SpecialOne`.

The native property schema independently labels `NonPlayerClass +0x38` as
`aggroRange` and `+0x3c` as `alertRange`. Both schema defaults are also `0.0`.
There is no float encoding for `18.0` (`00 00 90 41`) anywhere in the five
decoded same-instance Quadra resources. The AI record differs from ordinary
`TutorialSpecialOne` only in identities and the first-aggro slot; it does not
add a Quadra-specific positive perception override.

Focused artifact identities:

| Artifact | SHA-256 |
| --- | --- |
| decoded type `0x30728ce7` (`quadra.phase`, historical diagnostic filename) | `1eaeab2587b99774a201bbcf638814ca3458f46b5d36dae695e9988757dc82f9` |
| decoded noun | `b97f8663d51aa8482af8cd32992c6bf73ef495f4ef07cc82649730f91322154d` |
| decoded AI definition | `b6dee6ac376b1af6361a84b369fc2a4405d16e2d025079651fdf4f30fdcba3a9` |

## Native radius proof

Build-103 `sub_9E4040` (`0x009e4040`) supplies the circle used by the Lua
binding `nAgent.InPerceptionCircle`:

1. It obtains the agent's perception center.
2. It initializes the radius to `20.0`.
3. If the object has a `NonPlayerClass` reference at object offset `+0x80`, it
   resolves that record and replaces the radius with the larger float at
   `+0x38` or `+0x3c`.

Quadra is in step 3, not the fallback case: its noun explicitly links the
same-instance `TutorialSpecialOne_Intro.NonPlayerClass`. Both linked fields are
zero, so the result is `max(0.0, 0.0) = 0.0`. The `20.0` constant is only the
missing-`NonPlayerClass` fallback and is not Quadra's radius.

`sub_A0AEA0` (`0x00a0aea0`) implements the exposed test as:

```text
perceptionRadius > distance(point, perceptionCenter) - suppliedPointRadius
```

The comparison is strict. With the normal omitted/supplied-zero point radius,
Quadra's `0.0` never admits a point, including one exactly at the center.

## First-aggro event proof

The engine registers the string event `Aggro_Trigger` at `0x00f9b8f0` to
callback `sub_A22620` (`0x00a22620`). When invoked, the callback:

1. rejects an inapplicable client/disabled object;
2. validates hostility/team eligibility;
3. requires an agent brain;
4. calls `sub_9E4640(blackboard, targetID, 5.0, 2, "Trigger Volume")`.

On an empty aggro list, flag `2` takes the `a4 & 6` first-entry branch in
`sub_9E4640`. At `0x009e47d7` it pushes `0x20`, then at `0x009e47da` calls the
timed-stimulus function with a `10.0`-second lifetime. It sets AI byte `+0x555`
(`+1365`) and inserts the target. The agent-brain tick `sub_A21800` reads the
current stimulus mask and supplies it to the behavior-tree evaluator.

That runtime path joins the packaged AI definition:

```text
Aggro_Trigger
  -> sub_A22620 hostile trigger callback
  -> sub_9E4640 first aggro-list insertion
  -> timed stimulus 0x20 / 32
  -> agent brain behavior-tree evaluation
  -> firstAggroAbility = FirstAggro_SpecialOne
```

Two nearby paths must not be conflated with this event:

- Lua `nObject.AlertObject` calls `sub_9E4870` and posts stimulus `0x200`
  (decimal `512`) for `10.0` seconds. This is an explicit alert path, not the
  normal `Aggro_Trigger` callback.
- Lua `nObject.IgnoreFirstAggro` sets the consumed byte and **removes** mask
  `0x2a0` (decimal `672`). It does not post event 672. The mask includes
  `0x20`, `0x80`, and `0x200`, which is consistent with clearing the family of
  first-aggro stimuli.

## Assessment of the current 18-unit sphere

The current server constant is declared in `server/tutorial_encounter.go` and
is passed to Quadra's swept `sim.FirstAggroEntered` test in
`server/gameplay_udp.go`. Its existing live justification came from an
ordinary opening-enemy create observed at a goal 17.6 units from a different
marker. That observation neither measures retail Quadra's authored fields nor
identifies an engine threshold: it exercised darkspin's own 18-unit gate.

Consequently:

- `18` is not present in Quadra's packaged noun, `NonPlayerClass`, or AI
  definition;
- `18` is not the native missing-class fallback (`20`);
- Quadra's linked class overrides the fallback with exact radius `0`;
- movement through an 18-unit sphere cannot be claimed to reproduce the
  build-103 `Aggro_Trigger` contract for this noun.

No replacement server behavior is proposed here because this task is an
evidence note only. The safe modeling conclusion is to keep the missing
director/event producer explicit rather than treating the generic 18-unit
sphere as recovered retail behavior.

## Reproduction artifacts

Generated diagnostics are under `bin/game/logs/quadra-perception`; the IDA user
directory was `bin/game/ida_user`. The final headless IDA log is
`ida_quadra_perception.log`, SHA-256
`d0b270d6476cb72487dcf459122a020a84bc3f2fd61be71866ad49decb7ad447`.
It records the fixed-range disassembly for the radius schema, event
registration, callback, first-entry stimulus, explicit alert, and ignore-mask
paths.

## Confidence

| Finding | Confidence | Basis |
| --- | ---: | --- |
| Quadra's authored `aggroRange` and `alertRange` are both `0.0` | `0.99` | Exact decoded record offsets plus independent native property labels/defaults. |
| Quadra's effective content-backed perception radius is `0.0`, not `18` or fallback `20` | `0.98` | Explicit noun-to-`NonPlayerClass` link and native `max(+0x38,+0x3c)` selection. |
| Normal `Aggro_Trigger` first entry posts stimulus `0x20` | `0.99` | Exact event registration, callback arguments, and disassembly at `0x009e47d7..0x009e47da`. |
| `AlertObject` posts `0x200` and `IgnoreFirstAggro` removes `0x2a0` | `0.99` | Separate native call sites and exact immediate masks. |
| The missing external producer/order of Quadra's `Aggro_Trigger` is recovered | `0.00` | Neither these records nor the known Sage chunk identify that producer. |

