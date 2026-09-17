# Build-103 stat reference

| Name | Status | Reference | Confidence | Formula |
| --- | --- | --- | --- | --- |
| Strength (`STR`, 0) | Implemented | build-103 enum; profile projection | High | Raw stat; Ravager primary |
| Dexterity (`DEX`, 1) | Implemented | build-103 enum; profile projection | High | Raw stat; Sentinel primary |
| Mind (`MIND`, 2) | Implemented | build-103 enum; profile projection | High | Raw stat; Tempest primary |
| MaxHealthIncrease (3) | Not implemented | build-103 enum | Low | |
| Health / MaxHealth (`HLTH`, 4) | Partial | profile native path | High | `baseHealth + (Strength - 10) * 5` |
| Power / MaxMana (`MANA`, 5) | Partial | profile native path | High | `basePower + Mind` |
| DamageReduction (6) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| Dodge / PhysicalDefense (`PDEF`, 7) | Partial | profile native path | High | `baseDodge + Dexterity * 6` |
| PhysicalDamageReduction (8) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| Resist / EnergyDefense (`EDEF`, 9) | Partial | profile native path | High | `baseResist + Mind * 6` |
| CriticalRating (`CRTR`, 10) | Implemented | `sub_9E58E0`; `server/sim/critical.go` | High | `min(1, rating / (RatingConversion[difficulty-1] * 100))` |
| NonCombatSpeed (11) | Not implemented | build-103 enum | Low | |
| CombatSpeed (12) | Not implemented | build-103 enum | Low | |
| DamageBuff (13) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Multiplier starts at `1 + DamageBuff` |
| Silence (14) | Not implemented | build-103 enum | Low | |
| Immobilized (15) | Not implemented | build-103 enum | Low | |
| DefenseBoostBasicDamage (16) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Basic adds `value * (PhysicalDefense + EnergyDefense)` |
| PhysicalDamageIncrease (17) | Not implemented | build-103 enum | Low | |
| PhysicalDamageIncreaseFlat (18) | Not implemented | build-103 enum | Low | |
| AutoCrit (19) | Implemented | `sub_9E58E0`; `server/sim/critical.go` | High | Positive value guarantees critical |
| BehindDirectDamageIncrease (20) | Not implemented | build-103 enum | Low | |
| BehindOrSideDirectDamageIncrease (21) | Not implemented | build-103 enum | Low | |
| CriticalDamageIncrease (`CRTD`, 22) | Implemented | `sub_9E5B10`; `server/sim/critical.go` | High | `damage * (2 + increase)` |
| AttackSpeedScale (`ATTSP`, 23) | Implemented | `sub_438F00`; `server/game/timing.go` | High | Basic duration `/ (1 + max(scale, -0.9))` |
| CooldownScale (`COOL`, 24) | Implemented | `sub_438F00`; `server/game/timing.go` | High | Duration `* max(0, 1 - reduction)` |
| Frozen (25) | Not implemented | build-103 enum | Low | |
| ProjectileSpeedIncrease (`PROS`, 26) | Implemented | WaitForProjectile path; `server/game/timing.go` | High | `speed * (1 + max(increase, 0))` |
| AoeResistance (`AOERES`, 27) | Profile implemented | More Stats wire key; build-103 enum | Medium | |
| EnergyDamageBuff (28) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added for energy source |
| Intangible (29) | Not implemented | build-103 enum | Low | |
| HealingReduction (30) | Implemented | `sub_9E5D10`; `server/game/healing.go` | High | `max(0, healing * (1 - reduction))` |
| EnergyDamageIncrease (31) | Not implemented | build-103 enum | Low | |
| EnergyDamageIncreaseFlat (32) | Not implemented | build-103 enum | Low | |
| Immune (33) | Not implemented | build-103 enum | Low | |
| StealthDetection (34) | Not implemented | build-103 enum | Low | |
| LifeSteal (`LFSTL`, 35) | Implemented | Soul Ravager Support packaged bytecode; `server/gameplay/combat.go` | High | Accepted applied damage `* LifeSteal`, then target healing reduction and maximum-health cap |
| RejectModifer (36) | Not implemented | build-103 enum; client spelling | Low | |
| AoEDamage (`AOEDMG`, 37) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added when descriptor has `0x8` |
| TechnologyTypeDamage (38) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added for damage type 0 |
| SpacetimeTypeDamage (39) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added for damage type 1 |
| LifeTypeDamage (40) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added for damage type 2 |
| ElementsTypeDamage (41) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added for damage type 3 |
| SupernaturalTypeDamage (42) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added for damage type 4 |
| TechnologyTypeResistance (43) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| SpacetimeTypeResistance (44) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| LifeTypeResistance (45) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| ElementsTypeResistance (46) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| SupernaturalTypeResistance (47) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| MovementSpeedBuff (`MOV`, 48) | Partial | projectile locomotion path; More Stats wire key | Medium | Projectile speed `* (1 + buff)` |
| ImmuneToDebuffs (49) | Not implemented | build-103 enum | Low | |
| BuffDuration (`BUFD`, 50) | Partial | `sub_9DE220`; `server/game/timing.go` | High | Buff duration `* (1 + increase)` |
| DebuffDuration (51) | Partial | `sub_9DE220`; `server/game/timing.go` | High | Debuff duration `* (1 - reduction)` |
| ManaSteal (52) | Profile implemented | build-103 enum | Medium | |
| DebuffDurationIncrease (53) | Profile implemented | build-103 enum | Medium | |
| EnergyDamageReduction (54) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| Incorporeal (55) | Not implemented | build-103 enum | Low | |
| DoTDamageIncrease (56) | Not implemented | build-103 enum | Low | |
| MindControlled (57) | Not implemented | build-103 enum | Low | |
| SwapDisabled (58) | Not implemented | build-103 enum | Low | |
| ImmuneToRandomTeleport (59) | Not implemented | build-103 enum | Low | |
| ImmuneToBanish (60) | Profile implemented | build-103 enum | Medium | |
| ImmuneToKnockback (61) | Profile implemented | build-103 enum | Medium | |
| AoERadius (62) | Implemented | packaged ability Lua; `server/zone/ability` | High | Proven consumers use `base radius * (1 + increase)`; authored template paths that calculate but ignore the scaled radius remain unscaled |
| PetDamage (63) | Implemented | packaged pet ability definitions; `server/gameplay/combat.go` and pet-specific attack runtimes | High | Pet ability damage uses Pet Damage as its primary attribute through the native ability coefficient formula |
| PetHealth (64) | Implemented | `sub_9E2B40`; `server/gameplay/ability_summon.go` | High | Pet maximum and current health `* (1 + increase)` at spawn |
| CrystalFind (65) | Implemented | `sub_A184E0`; `server/gameplay/interaction.go` | High | Crystal chance `* (1 + increase)` |
| DNADropped (66) | Implemented | `sub_9CABF0`; `server/zone/loot/loot.go` | High | DNA amount `* (1 + increase)` |
| RangeIncrease (67) | Implemented | `sub_9DE710`; `server/game/range.go` | High | Authored admission range `* (1 + increase)` only for range-enabled, non-Basic descriptors; projectile travel and effect radii remain separate |
| OrbEffectiveness (68) | Implemented | build-103 pickup consumer; `server/gameplay/interaction.go` | High | Health and mana restoration `* (1 + increase)` |
| OverdriveBuildup (69) | Profile implemented | build-103 enum | Medium | |
| OverdriveDuration (70) | Implemented | `sub_4E2740`; `server/game/timing.go`; `server/gameplay/overdrive.go` | High | Overdrive drain rate `* (1 - increase)`, so duration is `maximum energy / projected drain` |
| LootFind (71) | Implemented | `sub_9CE140`; `server/gameplay/interaction.go` | High | Equipment chance `* (1 + increase)` |
| Surefooted (72) | Profile implemented | build-103 enum | Medium | |
| ImmuneToStunned (73) | Profile implemented | build-103 enum | Medium | |
| ImmuneToShocked (74) | Not implemented | build-103 enum | Low | |
| ImmuneToSleep (75) | Profile implemented | build-103 enum | Medium | |
| ImmuneToTaunted (76) | Profile implemented | build-103 enum | Medium | |
| ImmuneToTerrified (77) | Profile implemented | build-103 enum | Medium | |
| ImmuneToSilence (78) | Profile implemented | build-103 enum | Medium | |
| ImmuneToCursed (79) | Profile implemented | build-103 enum | Medium | |
| ImmuneToPoisonOrDisease (80) | Profile implemented | build-103 enum | Medium | |
| ImmuneToBurning (81) | Profile implemented | build-103 enum | Medium | |
| ImmuneToRooted (82) | Profile implemented | build-103 enum | Medium | |
| ImmuneToSlow (83) | Profile implemented | build-103 enum | Medium | |
| ImmuneToPull (84) | Profile implemented | build-103 enum | Medium | |
| DoTDamageDoneIncrease (`DOTDMG`, 85) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added when descriptor has `0x4` |
| AggroIncrease (86) | Profile implemented | build-103 enum | Medium | |
| AggroDecrease (87) | Profile implemented | build-103 enum | Medium | |
| PhysicalDamageDoneIncrease (88) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added when descriptor has `0x40` |
| PhysicalDamageDoneByAbilityIncrease (89) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added for non-basic descriptor `0x40` |
| EnergyDamageDoneIncrease (90) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added when descriptor has `0x80` |
| EnergyDamageDoneByAbilityIncrease (91) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added for non-basic descriptor `0x80` |
| ChannelTimeDecrease (92) | Partial | `sub_438810`; `server/game/timing.go` | High | Channel duration `* max(0, 1 - decrease)` |
| CrowdControlDurationDecrease (93) | Partial | `sub_9DE220`; `server/game/timing.go` | High | Non-DoT crowd-control duration `* (1 - reduction)` |
| DoTDurationDecrease (94) | Partial | `sub_9DE220`; `server/game/timing.go` | High | DoT duration `* (1 - reduction)` |
| AoEDurationIncrease (`AOEDUR`, 95) | Implemented | More Stats wire key; packaged ability bytecode; `server/game/timing.go` | High | Multiplies proven numeric-loop bounds and Plasma Sentinel Support's continuous lifetime by `1 + increase` |
| HealIncrease (`HEAL`, 96) | Implemented | `sub_9E5550`; `server/game/healing.go` | High | Healer multiplier starts at `1 + increase` |
| OnLockdown (97) | Not implemented | build-103 enum | Low | |
| HoTDoneIncrease (98) | Implemented | `sub_9E5550`; `server/game/healing.go` | High | Added when descriptor has `0x1000` |
| ProjectileDamageIncrease (99) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added when descriptor has `0x2000` |
| DistributeDamageAmongSquad (100) | Not implemented | build-103 enum | Low | |
| DeployBonusInvincibilityTime (101) | Profile implemented | build-103 enum | Medium | |
| PhysicalDamageDecreaseFlat (102) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| EnergyDamageDecreaseFlat (103) | Hero incoming damage implemented | `server/combat/defense.go`; `damage-mitigation.md` | Server policy | Applied after critical and named ability reductions; see mitigation note for gates and stacking |
| MinWeaponDamage (104) | Implemented | `sub_4CE0D0`; `server/game/gameplay_session.go` | High | Added after the equipped-weapon multiplier and before the minimum percentage |
| MaxWeaponDamage (105) | Implemented | `sub_4CE0D0`; `server/game/gameplay_session.go` | High | Added after the equipped-weapon multiplier and before the maximum percentage |
| MinWeaponDamagePercent (106) | Implemented | `sub_4CE0D0`; `server/game/gameplay_session.go` | High | Multiplies the weapon-adjusted minimum after its flat addition |
| MaxWeaponDamagePercent (107) | Implemented | `sub_4CE0D0`; `server/game/gameplay_session.go` | High | Multiplies the weapon-adjusted maximum after its flat addition; maximum is clamped to minimum |
| DirectAttackDamage (`ATTD`, 108) | Implemented | `sub_9E4EF0`; `server/game/damage.go` | High | Flat add unless DoT or HoT |
| DirectAttackDamagePercent (`ATTDP`, 109) | Implemented | `sub_9E4F40`; `server/game/damage.go` | High | Added to non-DoT multiplier |
| GetHitAnimDisabled (110) | Not implemented | build-103 enum | Low | |
| XPBoost (111) | Implemented | `server/game/gameplay_session.go`; `server/gameplay/combat.go` | Medium | Applied to per-player NPC XP awards |
| InvisibleToSecurityTeleporters (112) | Not implemented | build-103 enum | Low | |
| BodyScale (113) | Not implemented | build-103 enum | Low | |
| Percent normalization | Implemented | `sub_9C99A0` / `sub_9C9A30`; `server/game/part_catalog.go` | High | Listed percent attributes use stored value `* 0.01` at runtime |
| Ability primary coefficient | Implemented | `sub_9E4E60`; `server/game/damage.go` | High | `raw * (1 + (primary + 1) * coefficient)` |
| Mana cost | Implemented | ability native path; server gameplay | High | `authoredCost + primary * manaCoefficient` |
| Damage range selection | Implemented | build-103 random wrapper; `server/sim` | Medium | Inclusive integer `[minimum, maximum]` |
| Equipped weapon damage | Implemented | `sub_9CB0B0`; bundled `creatureprofile.html`; `server/game/part_catalog.go` | High | Weapon-backed base range uses `trunc(authored range * levelScale * 0.588235)`; a nonpositive item modifier falls back to `1` |
