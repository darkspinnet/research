# Ability effect corrections from the 0.7.27 reports

The September 5 UTC reports described oversized Roar of Derision, absent Zetawatt Beam, screen color flashes during Missile Barrage impacts, and incorrect Gravity Well death explosions. The following corrections change server presentation only. No shipped game files or Fang gameplay hooks change.

## Roar of Derision

Authoritative `content.db` chunk 428, resource 13979, `Abilities/0x426D3542.lua`, SHA-256 `47fb06d62ae5b0bd6776a427c046c4cb02eba9bf501f5a6bbfdcce3f661f69c0`, emits `roar_of_derision_cast.ServerEventDef` from `onFinishCharge`, with only `asset` and the caster's resulting `position`. The charge template (chunk 574) does not emit that cast event at startup. The server previously sent it at startup with the caster object ID, which selects the object's graphics transform in native `sub_506D90` instead of independent world placement. The correction emits fields 6/10 at charge completion before the taunt and buff presentation, and removes the startup emission. No arbitrary effect-size multiplier is introduced; the live visual size still needs confirmation.

## Zetawatt Beam

Authoritative chunk 172, resource 13705, `Abilities/0x15072D5A.lua`, SHA-256 `2084a1cc3ba678077de0b4776972099ccdcc892cade754d1811fbed391e90410`, emits `cyber_randomAbility_1.ServerEventDef` with `objectId=caster` and `targetPoint=endpoint` (prototype 0.1, instructions 108–121). The previous server event had object ID zero and used `position` for the endpoint. The beam now sends fields 6/7/13, retaining the caster and the existing 35-unit endpoint. Damage geometry and timing are unchanged.

## Missile Barrage

The retained Missile Barrage chunk matches authoritative chunk 287, resource 13830, SHA-256 `7eec6b13c37c59b91d862c45fc47ecf5ac222821c933565de58181692dc2cd80`. Its descending projectiles use the projectile template's impact presentation. The server sends exactly `(0,0,-1)` as impact facing on both ground and target impacts.

Canonical `sub_506D90` prioritizes nonzero facing over explicit orientation and calls `sub_7B1E00` with the facing and world-up. The latter normalizes their cross product without a zero-length guard. A vertical facing parallel to world-up produces invalid matrix entries. `PositionedEffectMessage` now omits facing for exactly vertical, finite directions and sends field 12 with a normalized quaternion rotating local +Y to the requested +Z or -Z direction. Canonical `sub_4DE210` confirms the quaternion-to-matrix convention. This retains vertical impact orientation without the singular conversion; ordinary facing payloads are unchanged. A real-client replay is still required to confirm the reported color flashes are eliminated.

## Gravity Wells

The report log identifies attacked `ZelemGravityOrb.Noun` fixtures. They previously fell through the size-based generic destructible explosion selection. Retained GravityOrbPassive bytecode matches authoritative chunk 830, resource 14407, SHA-256 `2e6b90f9c9803e6d363103df94902e5e3d32befa3e6d1942e58a47cfa2540e84`. Its deactivation clears gravity, removes the owned effect, emits `gravity_orb_fizzle.ServerEventDef` with a world position, and marks the orb for deletion when it has not already exploded.

Killed wells now select that fizzle rather than a scenery explosion and use the next scheduled deletion (one millisecond) rather than retaining a destructible corpse. Object deletion also releases attached presentation. Natural expiry now emits the fizzle at the orb position instead of binding it to the orb and caster, so it survives the following object deletion. Existing expiry cleanup continues to hard-stop its owned effect before the fizzle.

## Verification

Production source paths, authored bytecode arguments, native event transforms, and packet field selection were reviewed; Go formatting and `git diff --check` completed. No builds were run at the user's request. No tests were created, changed, or run, per repository policy. Live-client visual confirmation remains outstanding for all four corrections.
