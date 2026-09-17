# Player defeat presentation

Campaign hero defeat is a player death, not a voluntary squad withdrawal or
the ordinary NPC corpse-deletion path. At the lethal boundary the server stops
locomotion, publishes zero health, retains the lethal combat event's authored
damage descriptors, and plays the playable rig's indexed `gen_player_death`
state. The animation is followed by another authoritative publication through
both the squad-resource and active-combatant channels because the native death
transition can otherwise restore class-default full bars locally. It does not
play a beam effect, run `character_teleport_out`, or hide the body. The defeated
hero remains the client's controlled-object identity until the native
survivor-selection transition appears. When the player chooses a living squad
member, the handoff hides the dead source and deploys the selected replacement
without inventing a second death or warp-out presentation.

Movement and world interaction remain rejected while survivor selection is
pending, including commands queued by the client before the lethal hit. A dead
hero therefore cannot begin an unreachable obelisk pursuit; the obelisk remains
available for the selected living replacement. The server still records the
defeat for health, objectives, statistics, target release, ability cleanup, and
final-squad game-over authority.

The death-selection handoff sends one ordered deployment and arrival sequence:
resource state, controlled-object identity, `PlayerCharacterDeploy`, cooldown,
authoritative teleport, visibility, stopped pose, beam-in effect, and beam-in
animation. It also hides the defeated source without replaying a beam-out
animation. A voluntary switch is different: it first publishes the source's
element-specific beam and `character_teleport_out`, waits the packaged
teleporter's recovered 0.5-second out boundary, and only then hides the source
and publishes replacement deployment and beam-in. A death selection instead
uses one four-second presentation boundary measured from the lethal hit. An
early native squad choice is accepted but its replacement deployment remains
scheduled until that boundary, so different playable rigs cannot shorten or
lengthen the handoff.

The installed `CompiledAnimData.package` contains nine playable death clips.
Each uses the authored `0.033333335`-second frame step:

| Animation | Frames | Duration |
| --- | ---: | ---: |
| `gen_death_pc_gaunt` | 66 | 2.20 s |
| `gen_death_pc_gun_m_firerav` | 73 | 2.43 s |
| `gen_death_pc_gun_m_soulrav` | 65 | 2.17 s |
| `gen_death_pc_gun_m_tc` | 74 | 2.47 s |
| `gen_death_pc_gun_r` | 66 | 2.20 s |
| `gen_death_pc_knives` | 66 | 2.20 s |
| `gen_death_pc_meditron` | 74 | 2.47 s |
| `gen_death_pc_staff_l` | 66 | 2.20 s |
| `gen_death_pc_staff_r` | 66 | 2.20 s |

Soul Ravager's gun clip is the shortest at about 2.17 seconds. Tech Commander
and Meditron tie for longest at about 2.47 seconds. The shared handoff deadline
is therefore `ceil(2.47 s) + 1 s = 4 s`, independent of which clip the native
`gen_player_death` state resolves.

Scheduled gameplay deadlines are atomic once production begins. A lethal hit
may cancel all outstanding enemy actions, including the schedule that delivered
the killing projectile, but that cancellation only prevents later deadlines;
it cannot discard the already-produced health, death-animation, and selection
batch.

Build 103's native `SetAnimationStateToDeath` selector keys off the last-hit
descriptor bits (including source family, critical, and knockback). The
currently indexed playable `CharacterAnimation` resources expose
`gen_player_death` as their death state, while the combat event carries the
source-specific descriptor evidence into the client. The server therefore does
not fabricate element-named player death states absent from those rigs.
