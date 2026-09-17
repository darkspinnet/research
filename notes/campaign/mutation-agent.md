# Mutation Agent

The community [Mutation Agent](https://gamegame.fandom.com/wiki/Mutation_Agent) description is consistent with immutable build-103 content, but the package is the authority for implementation details. The decoded artifacts used for this audit are retained under `bin/game/logs/mutation-agent`.

## Packaged identity

`MutationAgent.Noun` resolves to `MutationAgent.NonPlayerClass`, `MutationAgent.ClassAttributes`, and `MutationAgent.AIDefinition`. Its class row is resource instance `0x5743EB97`: challenge 5, NPC rank 1, 120 hit points, 75 power points, 10 Strength/Dexterity/Mind, 60 Dodge/Resist, 45 Critical, targetable, and one-player health scaling. The localized display name is `Mutation Agent` and its authored description is `Transforms Game into elite enemies`.

## Recovered behavior

The passive modifier is Lua chunk 183, `Modifiers/0xE25CE90C.lua`, SHA-256 `c023686b9c159cbc2824901d0bd2d61d552b52bd0372efa9758888e8ce0f6551`. It searches within 15 units, maintains one promoted enemy, rejects targets that already carry `EliteModifier` or the temporary transform modifier, applies `MutationAgent_AoE` to itself, and checks the promoted target once per second so a dead or invalid target can be replaced.

The friendly mutation projectile is chunk 790, `Abilities/0x4ADE6F57.lua`, SHA-256 `2d776a8b16da4c21ef729e30fa668fb9175244c63029c8da13abdc3a3178dae7`. It has range 15, speed 10, a 1.7-second hit time, 2.5-second release, zero damage, and selects the nearest valid friendly, living, non-stealthed, non-destructible, different-species actor without `EliteModifier`. The alternate mutation beam is chunk 1011, `Abilities/0x05E1EA8E.lua`, SHA-256 `57534a92cd707acd3891888bc8a6c7ae5a26a798cf4d3072c8dce94aeb636c72`: range 12, hit at 1.7 seconds, mutation at 2.5 seconds, and release at 3 seconds.

The transform is chunk 336, `Modifiers/0xCC52C237.lua`, SHA-256 `e42465133ddfea7864ee9b18a82e1714f23f458f70fd31127c0bcde149084f37`. It plays `npc_mutationagent_infected_grow`, grants random-affix immunity during the one-second transform, then requests `EliteCreateModifier`. That modifier leads to the packaged `EliteModifier`, whose authoritative non-minion bonuses are 75 percent maximum health, 50 percent damage, and 25 percent body scale.

The hostile aura is chunk 592, `Modifiers/0x398F21C7.lua`, SHA-256 `30f6743656916e21f4fff4a80fe1e1088ef8c0d6bcab9ce3a2217db1eac74ac5`. It pulses every two seconds in a six-unit radius, waits 0.15 seconds after presenting `mutant_agent_aoe_smoke.ServerEventDef`, and deals rank-scaled 6/8/10 Generic Energy AoE damage with coefficient zero. The wander behavior is chunk 841, `behaviors/0x86EAB32E.lua`, SHA-256 `045c9a434a09d83fd04428bbe2890fad90392bc9322aa70c99b7eb00b5b6208c`: it searches within 25 units, flees players closer than eight units by four units at 1.5 times movement speed, and checks every 0.5 seconds.

## Spawn evidence and map policy

The content database has exactly one fixed `MutationAgent.Noun` marker: ordinal 17 at `(69.053, -29.086, 0.088)` in `test_AI_zoo_chrono_Quantum.Markerset`, level `test_AI_zoo_chrono`. It has no `level_director_entry`, and no campaign level package resource contains the noun name. `darkrun map` therefore labels only this exact authored actor as `A`, drawing it after common `M` markers; randomized campaign appearances are deliberately not projected as fixed coordinates.

The missing retail authority is the insertion selector: player-count gate, eligible levels, budget, timing, and chosen anchor. The conservative server fallback begins at chain level 5 (2-1), rolls once for each ordinary lieutenant-centered Spike population cluster, and replaces one common escort on success. Its chance rises linearly from 5 percent at level 5 to a 10-percent cap at level 24. Horde waves, named Captain and mini-boss pits, boss adds, and Destructor encounters use separate planners and are explicitly excluded.

The fallback now requires two ordinary escorts in addition to the cluster lieutenant. `MutationAgent.Noun` has class and AI records but no imported render/physics noun, so the controller retains the selected escort's renderable noun while carrying Mutation Agent stats, AI, semantic identity, and packaged hostile-aura smoke presentation. The nearest eligible escort within the authored 15-unit passive radius performs `npc_mutationagent_infected_grow` for one second and enters the shared Elite path without being reclassified as a Captain or wave event. Clusters without a legal mutation target retain their original population instead of spawning an inert controller.
