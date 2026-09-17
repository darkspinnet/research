# Entangling Rush contact presentation

The 2026-09-06 report showed the running animation continuing at the enemy and the ground burst arriving too early. Indexed chunk 388 (`Abilities/0xD6383F9E.lua`, SHA-256 `1fc40a4d4faa86898e4f48935cc8caec73f4da0a2f1fa315366ca1b49253321b`) proves `delayAfterHit=0`, `relaxAnimTime=1`, and `predictRelaxAnim=false`. Its `onFinishCharge` calls `nLocomotion.Stop`, then emits `entangling_rush_cast` at `GetTargetPosition` and applies roots around that point.

The server now switches from `entanglingrush_loop` to `entanglingrush_end` at arrival, emits the ground event there rather than at departure, and resets animation authority after the one-second ending animation. The effect and root query share the captured target position. Damage and the existing two-unit stopping offset remain unchanged.
