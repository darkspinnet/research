# Sage-to-Quadra authored edge

## Conclusion

There is no authored chunk-215-to-chunk-502 handoff, and therefore no authored
delay owned by `PlayerUnlockedSecondCreature` or by the second-creature job.
The two chunks have distinct owners:

- Cryos level trigger marker `3912233898` owns
  `nTutorial_IntroSecondCreatureUnlock.main` (chunk `215`).
- The `TutorialSpecialOne_Intro` AI definition owns
  `FirstAggro_SpecialOne` (chunk `502`) as that object's first-aggro ability.
  The engine's per-object first-aggro scheduler activates it when the Intro
  Special One reaches accepted first aggro.

Consequently the exact relative gap is not a content-authored constant. If
`t_enter` is acceptance of the second-creature trigger and `t_aggro` is the
engine's activation of the Intro Special One's first aggro, then chunk `215`
finishes no earlier than `t_enter + 0.5s` (plus its synchronous player loop,
notification, and job deletion), while chunk `502` begins at `t_aggro`.
The available sources do not establish that `t_aggro` follows chunk `215`'s
completion at all, much less establish a positive delay between them. A same
tick start after completion, a later start, or overlap with chunk `215`'s
half-second wait all require the still-unrecovered server/director scheduling
edge. Co-location of the two markers is not a callback or timing edge.

Once activated, chunk `502` owns only its own relative presentation timeline:
at offset `0` it starts a `4.291667s` cinematic and hides its agent; at `2s` it
shows the agent, repeats the visible state through the shared first-aggro
template, and plays `character_teleport_in`; at `3.291667s` the animation wait
ends; at `4.291667s` the final delay ends and the ability deactivates.

## Evidence

### `content.db`

Read-only `darkrun db` queries against
`bin/darkspinner/darkspin/cache/content.db` establish the following authored
links.

- `marker.id=17742` is `SecondCreature_trigger.Noun-1` /
  `TriggerZone.Noun`, authored marker ID `3912233898`, at
  `(259.345673, 81.391434, 25.088488)`.
- `level_event.id=6098` is its `triggerVolume.luaCallbackOnEnter` callback,
  `nTutorial_IntroSecondCreatureUnlock.main`, with radius `25`,
  `is_server_only=1`, and `is_trigger_once_only=0`.
- `level_script.id=133` links that event, and only that event in this evidence,
  to `lua_chunk_id=215` in Cryos level `9`.
- Chunk `215` is resource `13753`,
  `0x24F78AA1/0xE6FA7F3F.lua`, size `1033`, SHA-256
  `e98b1e855e64904689451bd79635c3487a7a20e29f1b943885c9648b61f8e30a`.
- The separate AI marker `TutorialSpecialOne_Intro.Noun` is
  `marker.id=17718`, authored marker ID `3112855115`, at
  `(259.345642, 81.391426, 25.088490)`. It belongs to the AI marker set, not
  the audio trigger set. Its position differs from the trigger by less than
  `0.00004` world units, proving intentional co-location but not invocation.
- Chunk `502` is resource `14061`, `Abilities/0xC57828E2.lua`, size `1237`,
  SHA-256
  `da18208fc597c67ce3251de30d12ad694ae4d644af0e1cb79b5383cc12ec6d8d`.
  Its one recorded dependency is
  `Abilities!template_ability_firstaggro.lua`, resolved directly to chunk
  `848` by the instruction-checked module alias. This identifies chunk `502`
  as a first-aggro specialization, not a level callback or master tutorial
  director.

The existing decoded-content notes add the field-level link omitted by the
normalized tables: `TutorialSpecialOne_Intro` replaces the ordinary tutorial
AI first-aggro ability with `FirstAggro_SpecialOne`, while retaining
`FirstAggro_BeamIn_Tutorial` for first alert. Chunk `215` contains the separate
job sequence: wait `0.5s`, enumerate players, call `UnlockSecondCreature`,
broadcast the `PlayerUnlockedSecondCreature` client event, and delete the job.
It contains no Quadra object, AI, aggro, or ability activation operation.

### `Game.c`

Build-103 native registration and implementations keep the ownership domains
separate.

- `nPlayer.UnlockSecondCreature` is registered to `sub_A050C0`
  (`Game.c:1426766`). It resolves the simulation player and increments
  the integer at player offset `+4984` only while the prior count is at most
  `1`. It does not resolve an AI object, post an aggro event, activate an
  ability, or schedule a delay.
- `nObject.OverrideFirstAggro` is registered to `sub_A02CF0`
  (`Game.c:1425061`). It writes the supplied object ID to the target AI
  runtime at offset `+1412`.
- `nObject.IgnoreFirstAggro` is registered to `sub_A02D70`
  (`Game.c:1425084`). Through `sub_9E3CC0` it sets AI byte `+1365` and
  posts engine event `672` (`Game.c:1399017`).

Neither first-aggro control native references the second-creature player field
or chunk-215 callback, and the unlock native does not reference the AI
first-aggro state. These functions expose controls on the engine-owned
first-aggro lifecycle; they do not supply a Sage-completion-to-Quadra timer.

## Confidence

| Finding | Confidence | Basis |
| --- | ---: | --- |
| Chunk `215` is owned by the server-only second-creature trigger | `0.99` (very high) | Exact `marker -> level_event -> level_script -> lua_chunk` database chain. |
| Chunk `502` is owned by `TutorialSpecialOne_Intro`'s first-aggro slot and activated by the AI first-aggro lifecycle | `0.96` (high) | Decoded AI field link, chunk name/dependency, and matching native first-aggro controls. |
| Chunk `215` has an authored `0.5s` initial wait and chunk `502` has the `0..4.291667s` internal timeline above | `0.99` (very high) | Instruction-level chunk notes and direct content identities. |
| No exact completion-to-activation delay or even strict post-completion ordering is recoverable from these sources | `0.97` (high) | Distinct authored owners, no call/dependency/native bridge, and only spatial co-location. The missing server/director scheduler could impose an external order not represented by these sources. |

No recap evidence was used. No Go source was changed for this determination.
