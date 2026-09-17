# Build-103 application packet map

This is the canonical map of build 5.3.0.103 gameplay application messages.
It covers the complete `0x7f` through `0xcc` vocabulary, with priority on
packets accepted by the client from the server.

A receiver-valid body is not proof of the missing retail server's sender
policy, ordering, reliability, or lifecycle.

## RakNet transport envelope

The application IDs below are carried inside the legacy RakNet transport. The
implemented build-103 transport contract is:

```c
struct C2S_09_LegacyOpen {
    u8 id = 0x09;
    u8 protocol = 0x0d;
    be64 client_guid;
    u8 offline_magic[16];
    u8 mtu_probe_padding[*];
};
struct S2C_0A_LegacyOpenReply {
    u8 id = 0x0a;
    u8 offline_magic[16];
    be64 server_guid;
    u8 complemented_ipv4[4];
    be16 client_port;
    u8 mtu_padding[*];
};
struct ConnectedDatagram {
    u8 flags; // high bit set; ordinary sender uses 0x84
    le24 datagram_sequence;
    EncapsulatedPacket packet[*];
};
struct EncapsulatedPacket {
    u8 reliability_and_split;
    be16 payload_bit_length;
    optional le24 message_index;
    optional le24 order_index;
    optional u8 order_channel;
    optional {
        be32 split_count;
        be16 split_id;
        be32 split_index;
    } split;
    u8 payload[ceil(payload_bit_length / 8)];
};
struct ACK_or_NACK {
    u8 id; // 0xc0 ACK or 0xa0 NACK
    be16 record_count;
    repeated {
        u8 is_single;
        le24 start;
        if (!is_single) le24 end;
    } record;
};
struct C2S_04_ConnectionRequest {
    u8 id = 0x04;
    u8 offline_magic[16];
    be64 client_guid;
    be64 client_time;
};
struct S2C_0E_ConnectionAccepted {
    u8 id = 0x0e;
    SystemAddress client;
    be16 system_index;
    SystemAddress system_address[10];
    be64 echoed_client_time;
    be64 server_time;
};
struct C2S_11_NewIncomingConnection {
    u8 id = 0x11;
    u8 client_owned_tail[*];
};
```

The Go transport codecs and fixtures cover offline negotiation, ACK/NACK
ranges, all eight RakNet reliability modes, ordered/sequenced indices,
fragmentation metadata, connection acceptance, and application encapsulation.
This document's 78-entry count refers specifically to the attached gameplay
application vocabulary, not these transport control records.

## Transport attachment map

`cProtocolTransport::Attach` asks RakNet for 78 consecutive wire IDs, then
associates them with an explicitly ordered table of internal GMS IDs and
names. Wire ID is therefore table position plus `0x7f`; it is not generally
`0x7f + internal_id`.

```text
7f:00 80:01 81:02 82:03 83:04 84:05 85:06 86:07 87:08 88:09 89:10 8a:11
8b:55 8c:12 8d:13 8e:14 8f:17 90:16 91:18 92:19 93:20 94:21 95:22 96:23
97:24 98:25 99:26 9a:27 9b:28 9c:29 9d:31 9e:32 9f:33 a0:34 a1:35
a2:36 a3:37 a4:38 a5:39 a6:40 a7:15 a8:30 a9:42 aa:43 ab:44 ac:41
ad:45 ae:46 af:47 b0:48 b1:49 b2:50 b3:51 b4:52 b5:53 b6:54 b7:56
b8:58 b9:57 ba:63 bb:59 bc:60 bd:61 be:62 bf:64 c0:65 c1:66 c2:67
c3:68 c4:69 c5:70 c6:71 c7:72 c8:73 c9:74 ca:75 cb:76 cc:77
```

This mapping is exact from the 78-entry attachment table at executable file
offset `0x00d81ba0` and `sub_A8FAC0`. It explains why internal constructor ID
`30` is wire `0xa8`, not wire `0x9d`.

The gameplay receiver `sub_53ADC0` converts the generated wire ID back to the
internal ID and dispatches the following server messages:

```text
06 PartyMergeComplete       -> inline
08 VoteKickStarted          -> sub_537FB0
11 GameStatePacket          -> sub_537CE0
12 ObjectCreate             -> sub_53A760
13 ObjectUpdate             -> sub_538F10
14 ObjectDelete             -> sub_538FD0
15 PlayerCharacterDeploy    -> sub_53ACA0
16 ObjectTeleport           -> sub_5392F0
17 ObjectJump               -> sub_5390F0
18 ObjectPlayerMove         -> sub_5393D0
19 ForcePhysicsUpdate       -> sub_539590
20 PhysicsChanged           -> sub_5396A0
21 LocomotionUpdate         -> sub_539900
22 LocomotionUnreliable     -> sub_539810
23 AttributeDataUpdate      -> sub_5399D0
24 CombatantDataUpdate      -> sub_539AA0
25 InteractableUpdate       -> sub_539B70
26 AgentBlackboardUpdate    -> sub_539C40
27 LootDataUpdate           -> sub_539D10
28 ServerEvent              -> sub_539E90
30 ActionCommandResponse    -> sub_537D90
35 LabsPlayerUpdate         -> sub_539E00
36 ModifierCreated          -> sub_539FE0
37 ModifierUpdated          -> sub_53A090
38 ModifierDeleted          -> sub_53A140
39 SetAnimationState        -> sub_53A1F0
40 SetObjectGFXState        -> sub_53A380
48 GamePrepareForStart      -> sub_537E00
49 GameStart                -> sub_537EC0
55 DirectorState           -> sub_538EE0
56 ObjectivesInit          -> sub_537AA0
57 ObjectivesComplete      -> sub_536EE0
58 ObjectiveUpdated        -> sub_536DA0
63 CombatEvent             -> sub_539F60
64 ReloadLevel             -> sub_537F60
65 GravityForceUpdate      -> sub_53A450
66 CooldownUpdate          -> sub_53A520
68 CrystalMessage          -> sub_53A910
75 ObjectiveAdd            -> sub_537BF0
77 DebugPing               -> sub_538AC0
```

The simulator's client-to-server dispatcher `sub_9C1580` accepts only internal
IDs `9` (player status), `29` (action command), `67` (crystal drag), `76`
(loot drop), and `77` (debug ping). Other mode-family consumers live outside
these two switches and are documented separately; absence from this list alone
is not treated as proof that a UI mode ignores a message.

## Go contract coverage

`server/raknet/contracts.go` contains one direction-aware descriptor for every
entry in the 78-opcode vocabulary. `codec_coverage_test.go` requires every
active S2C or bidirectional entry to have a typed encoder fixture and every
active C2S or bidirectional entry to have a typed decoder fixture. Unattached
and exhaustively unconsumed entries remain descriptors and are not given
fabricated bodies.

The complete optional create prefix, all 23 object reflection fields, all 18
reliable locomotion fields, and all 26 ServerEvent fields have generic typed
contract encoders in addition to narrower gameplay recipes. Unknown or
unhandled incoming messages retain name, full payload, endpoint, source time,
and client time in the ignored-packet log.

## Evidence and notation

| Status | Meaning |
| --- | --- |
| Exact | Complete client-consumed wire layout recovered or capture-fixed. A field may remain explicitly unnamed when the client only stores or ignores it. |
| Unattached | Vocabulary entry exists, but build 103 installs no GMS callback. |
| Unconsumed | Transport name/ID exists, but exhaustive build-103 analysis finds no receiver or sender for this entry. |
| C2S | Client request, not a server-to-client message. |

Integers and IEEE-754 floats are little-endian unless noted. `bytes[N]` is a
fixed byte image, `bytes[*]` is a receiver-ignored trailing range, and
`variant` is a discriminator-selected body.

```c
struct SparseReflection {
    repeated { u8 field_index; bytes[field_width(field_index)] field; };
    u8 end = 0xff;
};

struct MaskReflection {
    mask field_mask;
    bytes[*] fields_in_ascending_bit_order;
};
```

These encodings are not interchangeable.

## Complete vocabulary

| ID | Name | Direction | Status | Client context |
| ---: | --- | --- | --- | --- |
| `0x7f` | HelloPlayerRequest | C2S | Exact | Sent after `Connected`; account and optional playgroup. |
| `0x80` | HelloPlayer | S2C | Exact | Assigns gameplay slot and endpoint. |
| `0x81` | ReconnectPlayer | S2C | Exact | Selects a client game state on reconnect. |
| `0x82` | Connected | S2C | Exact | Empty handshake trigger; client replies `0x7f`. |
| `0x83` | Goodbye | Unattached | Exact | Vocabulary entry exists, but build 103 installs no GMS callback. |
| `0x84` | PlayerJoined | S2C | Exact | One-byte slot; adds/binds a gameplay player. |
| `0x85` | PartyMergeComplete | S2C | Exact | Eight-byte timestamp through the shared timestamp body. |
| `0x86` | PlayerDeparted | S2C | Exact | One-byte slot; removes a gameplay player. |
| `0x87` | VoteKickStarted | S2C | Exact | Fixed two-byte prompt state; the client ignores byte zero and resolves byte one as the target player slot. |
| `0x88` | PlayerStatusUpdate | C2S | Exact | Status plus progress; build 103 installs no S2C gameplay dispatch case. |
| `0x89` | GameAborted | Unattached | Exact | Vocabulary entry exists, but build 103 installs no GMS callback. |
| `0x8a` | GameStatePacket | S2C | Exact | Two clocks, state, game type, and trailing mode word. |
| `0x8b` | DirectorState | S2C | Exact | Complete seven-field boss, horde, captain, and completion reflection. |
| `0x8c` | ObjectCreate | S2C | Exact | Complete create mask followed by all 23 sparse base-object fields. |
| `0x8d` | ObjectUpdate | S2C | Exact | Object ID followed by the same complete 23-field base-object reflection. |
| `0x8e` | ObjectDelete | S2C | Exact | Deletes one or more object IDs. |
| `0x8f` | ObjectJump | S2C | Exact | Fixed object ID, jump position/direction, and four currently unnamed physics parameters copied into physics state. |
| `0x90` | ObjectTeleport | S2C | Exact | Immediate object relocation. |
| `0x91` | ObjectPlayerMove | S2C | Exact | Authoritative movement/turn goal. |
| `0x92` | ForcePhysicsUpdate | S2C | Exact | Fixed object ID plus position, Euler rotation, and scale replaces the complete transform. |
| `0x93` | PhysicsChanged | S2C | Exact | Object ID plus enabled byte toggles the physics representation. |
| `0x94` | LocomotionUpdate | S2C | Exact | Complete 18-field reliable locomotion reflection, including lob and projectile parameters. |
| `0x95` | LocomotionUnreliable | S2C | Exact | Partial-goal correction. |
| `0x96` | AttributeDataUpdate | S2C | Exact | Terminated indexed float-attribute updates. |
| `0x97` | CombatantDataUpdate | S2C | Exact | Sparse current HP and mana. |
| `0x98` | InteractableUpdate | S2C | Exact | Sparse use count, use limit, and interaction ability. |
| `0x99` | AgentBlackboardUpdate | S2C | Exact | Sparse target and AI combat state. |
| `0x9a` | LootDataUpdate | S2C | Exact | Complete ten-field crystal/item/rarity/instance/DNA reflection. |
| `0x9b` | ServerEvent | S2C | Exact | Complete 26-field presentation/event envelope. |
| `0x9c` | ActionCommandMsgs | C2S | Exact | Complete native size switch and every reachable constructor/caller are mapped. |
| `0x9d` | PlayerDamage | Unattached | Exact | Name/internal ID exist; no constructor or callback exists in build 103. |
| `0x9e` | LootSpawned | Unattached | Exact | Name/internal ID exist; no constructor or callback exists in build 103. |
| `0x9f` | LootAcquired | Unattached | Exact | Name/internal ID exist; no constructor or callback exists in build 103. |
| `0xa0` | SystemMessage | Unattached | Exact | Name/internal ID exist; no constructor or callback exists in build 103. |
| `0xa1` | LabsPlayerUpdate | S2C | Exact | Complete outer mask plus 24-field player, 124-field character, and two-field crystal reflections. |
| `0xa2` | ModifierCreated | S2C | Exact | Creates modifier instance. |
| `0xa3` | ModifierUpdated | S2C | Exact | Mutates timing, stack count, and bound state of one modifier instance. |
| `0xa4` | ModifierDeleted | S2C | Exact | Deletes modifier instance. |
| `0xa5` | SetAnimationState | S2C | Exact | Sets/reset animation state. |
| `0xa6` | SetObjectGFXState | S2C | Exact | Sets graphics state. |
| `0xa7` | PlayerCharacterDeploy | S2C | Exact | Binds squad index to controlled object. |
| `0xa8` | ActionCommandResponse | S2C | Exact | Types 1, 2, 4, 8, and 16 mutate action admission; other values are no-ops. |
| `0xa9` | ChainVoteMsgs | S2C | Exact | Fixed 337-byte result selection, countdown, and cash-out routing subtypes. |
| `0xaa` | ChainLevelResultsMsgs | Unconsumed | Exact | Build 103 has no registration, dispatcher case, reader, constructor, or sender for internal ID 43. |
| `0xab` | ChainCashOutMsgs | S2C | Exact | Fixed subtype-zero 712-byte cash-out presentation; unconsumed ranges remain explicitly reserved. |
| `0xac` | ChainPlayerMsgs | C2S | Exact | Six exact chain-vote/cash-out request variants; build 103 installs no S2C receiver. |
| `0xad` | ChainGameMsgs | S2C | Exact | Subtypes `0`, `1`, and `3` select active-game states 11, 13, and 6; other subtypes are drained. |
| `0xae` | ChainGameOverMsgs | S2C | Exact | In `cGameOverState`, bodyless subtypes `0` and `1` are no-ops; other subtypes are drained. |
| `0xaf` | QuickGameMsgs | S2C | Exact | Subtype `0` selects state 2; every other subtype is drained without mutation. |
| `0xb0` | GamePrepareForStart | S2C | Exact | Level, markerset, player mask/index. |
| `0xb1` | GameStart | S2C | Exact | Starts prepared level index. |
| `0xb2` | CheatMessage | Unconsumed | Exact | Vocabulary/attachment entry exists, but build 103 has no constructor, receiver registration, or dispatcher case for internal ID 50. |
| `0xb3` | ArenaPlayerMsgs | C2S | Exact | All three client constructors: bodyless result/lobby transitions and the six-byte AcceptMission selection. |
| `0xb4` | ArenaLobbyMsgs | S2C | Exact | Subtype `3` plus a fixed 1,279-byte lobby/results snapshot. |
| `0xb5` | ArenaGameMsgs | S2C | Exact | Complete state-gated subtype union: `5`, `7`, `8`, `9`, and `10`. |
| `0xb6` | ArenaResultsMsgs | S2C | Exact | Subtype `4` plus the fixed 1,274-byte, six-player results body. |
| `0xb7` | ObjectivesInit | S2C | Exact | Replaces selected-objective vector. |
| `0xb8` | ObjectiveUpdated | S2C | Exact | Updates one objective/player state. |
| `0xb9` | ObjectivesComplete | S2C | Exact | Final records plus four separately stored, intentionally unnamed result bytes for each of four players. |
| `0xba` | CombatEvent | S2C | Exact | Complete sparse damage/healing presentation and combat-text input. |
| `0xbb` | JuggernautPlayerMsgs | C2S | Exact | Both client constructors emit exactly one discriminator byte: subtype `0` or `1`. |
| `0xbc` | JuggernautLobbyMsgs | Unconsumed | Exact | No sender construction or receiver registration exists for internal ID 60. |
| `0xbd` | JuggernautGameMsgs | S2C | Exact | Subtype `3` consumes a packed seven-byte mode record; subtype `4` selects state 18. |
| `0xbe` | JuggernautResultsMsgs | S2C | Exact | Results state consumes one discriminator byte; subtype `2` is an explicit no-op. |
| `0xbf` | ReloadLevel | S2C | Exact | Empty in-place teardown/reload. |
| `0xc0` | GravityForceUpdate | S2C | Exact | Object ID followed by the two sparse `cGravityForce` vectors. |
| `0xc1` | CooldownUpdate | S2C | Exact | Ability/global cooldown clock. |
| `0xc2` | CrystalDragMessage | C2S | Exact | Fixed 24-byte editor drag command for world drop, positioned drop, or slot move. |
| `0xc3` | CrystalMessage | S2C | Exact | Fixed crystal HUD queue record: acquired, move accepted, or move rejected. |
| `0xc4` | KillRacePlayerMsgs | C2S | Exact | Bodyless subtype `1` and subtype `3` with a flag plus four uninitialized constructor bytes. |
| `0xc5` | KillRaceLobbyMsgs | Unconsumed | Exact | No sender construction or receiver registration exists for internal ID 70. |
| `0xc6` | KillRaceGameMsgs | S2C | Exact | Subtype `7` consumes a seven-byte exit record; subtype `8` selects state 20. |
| `0xc7` | KillRaceResultsMsgs | S2C | Exact | Every subtype consumes one byte only; subtype `5` is explicitly tested and remains a no-op. |
| `0xc8` | TutorialGameMsgs | S2C | Exact | Tutorial state consumes subtype `0` plus cumulative XP; active generic gameplay consumes bodyless subtype `0` as a state-2 transition. |
| `0xc9` | CinematicMsgs | S2C | Exact | Camera cinematic around point/radius. |
| `0xca` | ObjectiveAdd | S2C | Exact | Appends one objective record. |
| `0xcb` | LootDropMessage | C2S | Exact | Drops persistent inventory item. |
| `0xcc` | DebugPing | Both | Exact | Diagnostic timestamp. |

## Handshake, party, and state

```c
struct S2C_80_HelloPlayer {
    u8 player_type;
    u8 gameplay_index;
    u8 ipv4_network_order[4];
    u16 port;
}; // 8, Exact

struct S2C_81_ReconnectPlayer { u32 game_state; }; // 4, Exact
struct S2C_82_Connected {};                        // 0, Exact
struct S2C_83_Goodbye { unreachable; };
struct S2C_84_PlayerJoined { u8 slot; };
struct S2C_85_PartyMergeComplete { u64 timestamp; };
struct S2C_86_PlayerDeparted { u8 slot; };
struct S2C_87_VoteKickStarted {
    u8 unused;
    u8 target_player_slot;
}; // 2, Exact
struct C2S_88_PlayerStatusUpdate {
    u32 status;
    f32 progress;
}; // 8, Exact
struct S2C_89_GameAborted { unreachable; };
struct S2C_8A_GameState {
    u64 game_time;
    u64 elapsed_time;
    u8 game_state;
    u32 game_type;
    u32 mode_word;
}; // 25, Exact
```

The Beam Out request is the `0x88` instance
`{ status = 0x20, progress = 1.0 }`. `sub_538C30` is the sole build-103
constructor and always emits this eight-byte shape toward the server.

## Director, object, and component state

```c
struct S2C_8B_DirectorState {
    u8 mask;
    if (mask & 0x01) bool is_boss_spawned;
    if (mask & 0x02) bool is_boss_horde;
    if (mask & 0x04) bool is_captain_spawned;
    if (mask & 0x08) bool is_boss_complete;
    if (mask & 0x10) bool is_horde_spawned;
    if (mask & 0x20) u32 boss_object_id;
    if (mask & 0x40) u32 active_horde_wave_handle;
}; // Exact

struct S2C_8C_ObjectCreate {
    u32 object_id;
    u16 create_mask;
    // mask fields 0..9, in ascending order when selected:
    optional u32 noun;
    optional vec3 position;
    optional f32 rotation_x_degrees;
    optional f32 rotation_y_degrees;
    optional f32 rotation_z_degrees;
    optional u64 asset_id;
    optional f32 scale;
    optional u8 team;
    optional bool collision_enabled;
    optional bool player_controlled;
    SparseReflection object_tail;
}; // Exact

struct S2C_8D_ObjectUpdate {
    u32 object_id;
    SparseReflection object_fields;
};
struct S2C_8E_ObjectDelete { u32 object_id[]; }; // length divisible by 4
struct S2C_8F_ObjectJump {
    u32 object_id;
    vec3 jump_position;
    vec3 jump_direction;
    f32 jump_parameter[4];
}; // 44, Exact; copied to physics offsets +440 through +476

struct S2C_90_ObjectTeleport {
    u32 object_id;
    vec3 position;
    quat orientation;
}; // 32, Exact

struct S2C_91_ObjectPlayerMove {
    u32 object_id;
    u32 goal_flags;
    vec3 goal_position;
    vec3 facing;
    vec3 external_velocity;
    vec3 external_force;
    f32 allowed_stop_distance;
    f32 desired_stop_distance;
    vec3 target_position;
    u32 target_object_id;
}; // 80, Exact

struct S2C_92_ForcePhysicsUpdate {
    u32 object_id;
    vec3 position;
    vec3 rotation_euler;
    vec3 scale;
}; // 40, Exact
struct S2C_93_PhysicsChanged {
    u32 object_id;
    bool is_enabled;
}; // 5, Exact

struct S2C_94_LocomotionUpdate {
    u32 object_id;
    SparseReflection locomotion_fields;
}; // Exact

struct S2C_95_LocomotionUnreliable {
    u32 object_id;
    vec3 partial_goal_position;
}; // 16, Exact

struct S2C_96_AttributeDataUpdate {
    u32 object_id;
    repeated { u8 attribute_index; f32 amount; };
    u8 end = 0xff;
};

struct S2C_97_CombatantDataUpdate {
    u32 object_id;
    u8 mask;
    if (mask & 0x01) f32 hit_points;
    if (mask & 0x02) f32 mana_points;
};

struct S2C_98_InteractableUpdate {
    u32 object_id;
    u8 mask;
    if (mask & 0x01) i32 times_used;
    if (mask & 0x02) i32 uses_allowed;
    if (mask & 0x04) u32 ability;
};

struct S2C_99_AgentBlackboardUpdate {
    u32 object_id;
    u8 mask;
    if (mask & 0x01) u32 target_object_id;
    if (mask & 0x02) bool is_in_combat;
    if (mask & 0x04) u8 stealth;
    if (mask & 0x08) bool is_targetable;
    if (mask & 0x10) u32 attacker_count;
};

struct S2C_9A_LootDataUpdate {
    u32 object_id;
    u16 mask;
    if (mask & 0x001) i32 crystal_level;
    if (mask & 0x002) u64 item_id;
    if (mask & 0x004) u32 rigblock_asset;
    if (mask & 0x008) u32 suffix_asset;
    if (mask & 0x010) u32 prefix_asset;
    if (mask & 0x020) u32 secondary_prefix_asset;
    if (mask & 0x040) i32 item_level;
    if (mask & 0x080) i32 rarity;
    if (mask & 0x100) u64 instance_id;
    if (mask & 0x200) f32 dna_amount;
}; // Exact
```

The `0x8c` object-tail vocabulary is complete:

```text
0  u8 team                         1  bool player-controlled
2  u32 input stamp                 3  u8 player index
4  vec3 linear velocity            5  vec3 angular velocity
6  vec3 position                   7  quat orientation
8  f32 scale                       9  f32 marker scale
10 u32 last animation state        11 u64 last animation time
12 u32 move/idle override          13 u32 graphics state
14 u64 graphics-state start        15 u64 new graphics-state start
16 bool visible                    17 bool collision
18 u32 owner ID                    19 u8 movement type
20 bool disable repulsion          21 u32 interactable state
22 u32 source marker ID
```

A create prefix is not a complete object: combatant, attributes, interactable,
loot, locomotion, and blackboard arrive separately.

The reliable `0x94` vocabulary is:

```text
0  u64 lob start time              1  f32 previous lob speed modifier
2  LobParameter lob parameters     3  ProjectileParameter projectile parameters
4  u32 goal flags                  5  vec3 goal position
6  vec3 partial goal position      7  vec3 facing
8  vec3 external linear velocity   9  vec3 external force
10 f32 allowed stop distance       11 f32 desired stop distance
12 u32 target object ID            13 vec3 target position
14 vec3 expected geo collision     15 vec3 initial direction
16 vec3 offset                     17 i32 reflected last update
```

Known reliable variants are:

```c
struct LocomotionLobFields {
    field[0] u64 start_time_ms;
    field[1] f32 previous_speed_modifier;
    field[2] LobParameter parameter; // exact 84-byte nested image
    end;
};
struct LocomotionProjectileFields {
    field[3] ProjectileParameter parameter;
    optional field[14] vec3 expected_geometry_collision;
    field[17] i32 reflected_last_update;
    end;
};
```

These prove receiver shapes, not retail batching or `0x94` versus `0x95`
selection policy.

## Events, actions, modifiers, and combat

```c
struct S2C_9B_ServerEvent {
    SparseReflection event_fields; // 26 registered fields
};
struct S2C_9D_PlayerDamage { unattached; };
struct S2C_9E_LootSpawned { unattached; };
struct S2C_9F_LootAcquired { unattached; };
struct S2C_A0_SystemMessage { unattached; };

struct S2C_A2_ModifierCreated {
    u32 target_object_id;
    u32 modifier_guid;
    u32 instance_id;
    u32 duration_ms;
    u32 overdrive;
    u32 stack_count;
    u64 start_ms;
    u32 source_object_id;
    bool is_bound;
}; // 37, Exact
struct S2C_A3_ModifierUpdated {
    u32 target_id;
    u32 instance_id;
    i64 start_milliseconds; // -1 preserves the existing start time
    u32 stack_count;
    bool is_bound;
}; // 21, Exact
struct S2C_A4_ModifierDeleted {
    u32 target_object_id;
    u32 instance_id;
}; // 8, Exact

struct S2C_A5_SetAnimationState {
    u32 object_id;
    u32 animation_state;
    u64 timestamp_ms;
    bool is_overlay;
    f32 scale;
    u32 source_echo_gate; // native sender uses zero
}; // 25, Exact
struct S2C_A6_SetObjectGFXState {
    u32 object_id;
    u32 gfx_state;
    u64 timestamp_ms;
}; // 16, Exact
struct S2C_A7_PlayerCharacterDeploy {
    u8 player_index;
    u32 creature_index;
    u32 object_id;
}; // 9, Exact

struct S2C_A8_ActionCommandResponse {
    u8 sync_stamp;
    u8 response_type; // accepted=1, rejected=2, released=4, pursuit=8, clear-user-data=16
    u8 reserved_02_03[2];
    u32 object_id;
    u32 ability_index;
    u8 reserved_0c_0f[4];
    u64 source_start_ms;
    u64 source_commit_ms;
    u64 source_end_ms;
    u64 global_cooldown_ms;
    u8 reserved_30_33[4];
    u32 user_data;
}; // 56, Exact

struct S2C_BA_CombatEvent {
    u8 mask;
    fields selected from {
        u16 flags; f32 delta_health; f32 absorbed_amount;
        u32 target_object_id; u32 source_object_id; u32 ability_id;
        vec3 damage_direction; i32 integer_hp_change;
    };
}; // Exact

struct S2C_C1_CooldownUpdate {
    u32 object_id;
    u64 ability_key;
    i64 duration_ms;
    i64 source_start_ms;
    i64 global_cooldown_ms;
}; // 36, Exact
```

Recovered `0x9b` fields are simple-swarm effect `0`, slot `1`, removal `2`,
hard-stop `3`, force-attached `4`, critical `5`, asset `6`, primary object `7`,
secondary object `8`, attacker `9`, position `10`, facing `11`, orientation
`12`, target point `13`, text `14`, client event ID `15`, and loot tail
`17..25`. Field `16` is the player-exclusion mask. `nEvent.NotifyPlayer`
writes the exact expression `~(1 << player_index)` at structure offset `+96`;
ordinary broadcast events retain the zero default and omit the sparse field.

`0x9c` is a C2S discriminated action family. Its complete build-103 tail-size
switch is `0:0`, `1:12`, `2:16`, `3:24`, `4:24`, `5:4`, `6:1`, `7:44`,
`8:44`, `9:20`, `10:24`, `11:4`, and `12:0`. Types `0`, `1`, and `2` have
legacy size-switch entries but no constructor reaches the sole sender in this
build, so the server rejects them as reserved. Type `6` is the one-byte
Overdrive request used by `LABS_REQ_OVERDRIVE`. Type `13` is rejected before
the sender because the native serializer supports only `0..12`. Known
requests include ground movement, stop,
targeted/ground ability, interact, crystal pickup, equipment pickup, and
cancellation. Each accepted request requires a terminal `0xa8` or an
explicitly proven response-free path.

The `0xa8` receiver switch has cases only for:

| Type | Client mutation |
| ---: | --- |
| `1` | Accept the matching action, install definition/timing, and clear active/queued admission. |
| `2` | Reject/cancel the matching active or queued action. |
| `4` | Release the matching accepted action and its timing state. |
| `8` | Transfer the active definition into pursuit and start object/point locomotion. |
| `16` | Clear the retained last-action user data. |

Type `0` and all other values return without mutation. They are not valid
success acknowledgements. A type-5 squad switch performs its local switch path
without reserving the generic ability slots, so successful deploy replication
is response-free; rejection uses matching type `2`.

## Player and squad reflection

```c
struct S2C_A1_LabsPlayerUpdate {
    u8 player_slot;
    u16 outer_mask;
    if (outer_mask & 0x1000) SparseReflection player_fields;
    if (outer_mask & 0x0001) SparseReflection character_0;
    if (outer_mask & 0x0002) SparseReflection character_1;
    if (outer_mask & 0x0004) SparseReflection character_2;
    if (outer_mask & 0x0008) LabsCrystalReflection crystal_0;
    if (outer_mask & 0x0010) LabsCrystalReflection crystal_1;
    if (outer_mask & 0x0020) LabsCrystalReflection crystal_2;
    if (outer_mask & 0x0040) LabsCrystalReflection crystal_3;
    if (outer_mask & 0x0080) LabsCrystalReflection crystal_4;
    if (outer_mask & 0x0100) LabsCrystalReflection crystal_5;
    if (outer_mask & 0x0200) LabsCrystalReflection crystal_6;
    if (outer_mask & 0x0400) LabsCrystalReflection crystal_7;
    if (outer_mask & 0x0800) LabsCrystalReflection crystal_8;
}; // Exact

struct LabsFixedCrystal {
    u32 noun;
    u16 crystal_level;
    u8 reserved[10];
}; // 16 bytes in player field 13

struct LabsCrystalReflection {
    u8 mask; // bit 0 noun, bit 1 crystal level
    if (mask & 1) u32 noun;
    if (mask & 2) u16 crystal_level;
};
```

The player reflection has registered fields `0..23`: data setup `0`, current
deck index `1`, queued deck index `2`, three fixed character images `3`, player
index `4`, team `5`, online ID `6`, status `7`, progress `8`, controlled object
`9`, Overdrive energy `10`, charged flag `11`, DNA `12`, nine 16-byte crystal
records `13`, eight crystal-bonus flags `14`, avatar level `15`, avatar XP `16`,
chain progression `17`, camera lock `18`, locked Overdrive `19`, locked
crystals `20`, ability boundary `21`, locked deck minimum `22`, and deck score
`23`. The initial form
also embeds three fixed `0x620`-byte character images in player field `3`,
three nested character reflections, and nine selected crystal records.
Character fields `0..12` are version, noun, asset, creature type, deploy
deadline, ability points, nine ability ranks, current/max health, current/max
mana, gear score, and flattened gear score. Character fields `13+i`, for
attribute index `i=0..110`, are sparse `f32` part attributes. The nested
character reflector therefore has exactly 124 possible fields. The separate
attribute reflector's minimum/maximum weapon-damage fields `111` and `112` do
not fit in this nested character range and are not serialized by the recovered
character writer.

The fixed `0x620` character image and sparse character reflection now carry
the selected account creature's actual gear score and its floor-flattened
score. They also carry all 111 computed part attributes, including the selected
hero template's minimum and maximum weapon damage at indices `101` and `102`.
The fixed attribute image is discontiguous: indices `0..73` begin at `0x0b8`,
index `74` is at `0x1e4`, indices `75..97` begin at `0x1ec`, and indices
`98..110` begin at `0x24c`. The historical gear score `300.0`, attack/cooldown
scales `1.0`, and weapon range `1..5` came from reference-server defaults, not
the selected build-103 account hero, and are no longer fabricated.
Crystal-record fields remain documented separately. No deck-score aggregation
formula exists in the local evidence set: build 103 only copies field 23 and
increments it through the tutorial's bounded 0-to-2 unlock path, while the
preserved server source comments out a call to an unimplemented
`Squad::GetScore`. Any retail scoring policy is therefore an external server
boundary rather than an outstanding client decode.

The recovered reference writer places character type/ability fields at
`0x3b8/0x3c8/0x3cc`, but clean build-103 client experiments proved those
reference offsets stale by `0x30`. Build 103 accepts and consumes them at
`0x388/0x398/0x39c`; the encoder retains the live-verified offsets.

Each crystal mask reflection has field `0` noun (`u32`) and field `1` crystal
level (`u16`); unlike the player and character sparse reflectors it has no `0xff`
terminator. A full initial snapshot uses outer mask `0x1fff`: three characters,
all nine crystals, and the player reflection. The former `0x17ff` mask omitted
crystal slot eight and was not a complete player snapshot.

An incremental full-inventory refresh uses outer mask `0x1ff8`: player field
`13` first copies all nine fixed crystal records, then outer fields `3..11`
apply the nine nested crystal reflections. The nested noun callback resolves
each transmitted noun ID into the resource handle stored in the fixed record.
Advertising only player bit `0x1000` leaves the raw noun ID in that handle slot;
the catalyst tooltip later treats it as a pointer and faults on hover.

## Chain, game mode, and objectives

```c
struct S2C_A9_ChainVote {
    u8 subtype;
    variant {
        case 0: struct {
            u32 current_level;
            u32 chain_level_index;
            u32 star_or_reward_tier;
            f32 time_remaining;
            u8 planets_represented;
            u32 preview_enemy_noun[6];
            u8 reserved_029[16];
            u32 first_presentation[3];
            u8 reserved_045[4];
            u32 unlock_ordinal;
            u8 reserved_04d[140];
            u32 next_presentation[3];
            u8 reserved_0e5[4];
            u32 next_level;
            u8 reserved_0ed[100];
        } selection_or_result; // 337 bytes
        case 1: f32 countdown_seconds;
        case 2: bool route_to_cashout;
    };
}; // Exact
struct S2C_AA_ChainLevelResults { unconsumed; };
struct ChainCashOutReward {
    u8 reserved_00[8];
    u32 rigblock;
    u32 suffix;
    u32 prefix;
    u32 prefix_2;
    i32 level;
    i32 rarity;
    i32 roll;
}; // 36 bytes
struct S2C_AB_ChainCashOut {
    u8 subtype;
    if (subtype == 0) struct {
        i32 planets_completed;
        i32 dna;
        u8 reserved_008[12];
        i32 starting_xp[4];
        i32 final_xp[4];
        i32 gold_medals[4];
        i32 silver_medals[4];
        i32 bronze_medals[4];
        i32 upper_rarity_boundary[4];
        i32 lower_rarity_boundary[4];
        u8 cashout_bonus_granted[4];
        ChainCashOutReward reward[4][4];
    } cashout_snapshot; // 712 bytes
    else bytes[*] ignored;
}; // Exact
struct C2S_AC_ChainPlayer {
    u8 subtype;
    variant {
        case 0: bytes[0] request_vote_data;
        case 1: { u8 choice; u32 selected_record_id; };
        case 2: bytes[0] select_cashout;
        case 4: bytes[0] request_cashout_data;
        case 7: u8 player_slot;
        case 8: bool player_flag;
    };
};
struct S2C_AD_ChainGame {
    u8 subtype;
    variant {
        case 0: bytes[0]; // state 11
        case 1: bytes[0]; // state 13 / Game Over
        case 3: bytes[0]; // state 6
        default: bytes[*] ignored;
    };
}; // Exact in active gameplay
struct S2C_AE_ChainGameOver {
    u8 subtype;
    if (subtype == 0 || subtype == 1) bytes[0];
    else bytes[*] ignored;
}; // Exact in cGameOverState; accepted subtypes perform no mutation
struct S2C_AF_QuickGame {
    u8 subtype;
    if (subtype == 0) bytes[0]; // state 2
    else bytes[*] ignored;
}; // Exact

struct S2C_B0_GamePrepareForStart {
    u32 level;
    u32 markerset;
    u32 player_mask;
    u32 level_index;
}; // 16, Exact
struct S2C_B1_GameStart { u32 level_index; }; // 4, Exact

struct ObjectiveRecord {
    u32 objective_id;
    u8 state[4];
    u32 token[4][3];
}; // 56, Exact
struct S2C_B7_ObjectivesInit {
    u8 count;
    ObjectiveRecord record[count];
};
struct S2C_B8_ObjectiveUpdated {
    u32 objective_id;
    u8 player_index;
    u8 medal;
    u32 voiceover;
    bool is_shown;
    u32 token[3];
}; // 23, Exact
struct S2C_B9_ObjectivesComplete {
    u8 count;
    ObjectiveRecord record[count];
    struct {
        u8 result_field_0;
        u8 result_field_1;
        u8 result_field_2;
        u8 result_field_3;
    } player_result[4];
}; // Exact; trailing bytes are stored independently but not interpreted here
struct S2C_CA_ObjectiveAdd { ObjectiveRecord record; };
```

`ObjectivesComplete` reads its trailing 16 bytes as four byte fields per
player, not as four semantic 32-bit words. The equivalent byte count had hidden
that distinction in earlier notes; `sub_536EE0` stores each field into a
separate four-player client array.

The subtype-zero `0xa9` selection and post-level result uses share the same
337-byte client record. Their sender policy differs, but their client-consumed
wire layout is exact. `0xab` has an exact presentation layout; its reserved
bytes and reward authority remain explicitly outside the recovered contract.

## Remaining mode and utility families

```c
struct S2C_B2_CheatMessage { unconsumed; };
struct C2S_B3_ArenaPlayer {
    u8 subtype;
    variant {
        case 0:
        case 1: bytes[0];
        case 2: { u8 accepted = 1; i32 deck_id; } accept_mission;
    };
}; // Exact
struct ArenaRoundResult {
    u32 outcome;
    u32 field_04;
    u64 creature_resource[3];
    f32 health_percent[3];
    f32 mana_percent[3];
}; // 56
struct ArenaPlayerResult {
    u64 player_id;
    u32 avatar_id;
    u8 team;
    u32 kills;
    u32 deaths;
    f32 damage_dealt;
    f32 damage_taken;
    f32 healing_dealt;
    f32 healing_received;
    ArenaRoundResult round[3];
}; // 205
struct ArenaResultsBody {
    u32 selector_a;
    u32 selector_b;
    u32 count_a;
    u32 count_b;
    i32 round_time_seconds;
    u32 dna_amount[6];
    ArenaPlayerResult player[6];
}; // 1274
struct S2C_B4_ArenaLobby {
    u8 subtype = 3;
    u8 lobby_flag;
    u32 lobby_word;
    ArenaResultsBody results;
}; // 1280, Exact
struct S2C_B5_ArenaGame {
    u8 subtype;
    variant {
        case 5: bool alternate_state; // false -> state 15, true -> state 16
        case 7: bytes[0];             // state 7
        case 8: bytes[0];             // state 6
        case 9: u32 player_word_1524;
        case 10: u32 player_word_1528;
        default: bytes[*] ignored;
    };
}; // Exact; recognized subtype depends on the currently registered state
struct S2C_B6_ArenaResults {
    u8 subtype = 4;
    ArenaResultsBody results;
}; // 1275, Exact; other subtypes are drained after the discriminator
struct C2S_BB_JuggernautPlayer {
    u8 subtype; // exactly 0 or 1
}; // Exact
struct S2C_BC_JuggernautLobby { unconsumed; };
struct S2C_BD_JuggernautGame {
    u8 subtype;
    variant {
        case 3: {
            u8 field_01;
            u16 field_02;
            u16 field_04;
            u16 field_06;
        };
        case 4: bytes[0]; // state 18
        default: bytes[*] ignored;
    };
}; // Exact
struct S2C_BE_JuggernautResults {
    u8 subtype; // subtype 2 is explicitly handled; every value is a no-op
    bytes[*] ignored_tail;
}; // Exact
struct S2C_BF_ReloadLevel {};
struct S2C_C0_GravityForceUpdate {
    u32 object_id;
    repeated {
        u8 field;
        variant {
            case 0: vec3 force;
            case 1: vec3 force_for_mover;
        };
    };
    u8 end = 0xff;
}; // Exact; sub_53A450 resolves object +748 and class hash 0x758f9056
struct C2S_C2_CrystalDrag {
    i32 source_slot;
    i32 operation; // 0 world, 1 world at position, 2 slot
    vec3 world_position;
    i32 destination_slot;
}; // 24, Exact

struct S2C_C3_CrystalMessage {
    u8 subtype;
    variant {
        case 0: {
            i32 slot;
            u32 noun_asset;
            u32 ignored_09_0c;
            i32 crystal_color; // five-way noun color from sub_9D9870, not stat identity
            i32 selected_subtype; // sender-owned; client ignores it
            u8 ignored_15_1c[8];
        };
        case 2: {
            u8 ignored_01_14[20];
            i32 source_slot;
            i32 destination_slot; // -1 means dropped
        };
        case 3: u8 ignored_01_1c[28];
    };
}; // 29, Exact; other nonzero subtypes are drained without a HUD action

struct C2S_C4_KillRacePlayer {
    u8 subtype;
    variant {
        case 1: bytes[0];
        case 3: { u8 is_selected = 1; u8 uninitialized_stack[4]; };
    };
}; // Exact
struct S2C_C5_KillRaceLobby { unconsumed; };
struct S2C_C6_KillRaceGame {
    u8 subtype;
    variant {
        case 7: bytes[7] exit_record;
        case 8: bytes[0]; // state 20
        default: bytes[*] ignored;
    };
}; // Exact; other subtypes drain only the discriminator
struct S2C_C7_KillRaceResults {
    u8 subtype; // subtype 5 is explicitly tested; every value is a no-op
    bytes[*] ignored_tail;
}; // Exact; receiver consumes only the discriminator
struct S2C_C8_TutorialGame {
    receiver_context {
        tutorial_state: {
            u8 subtype;
            if (subtype == 0) i32 cumulative_xp;
            else bytes[*] ignored;
        };
        active_generic_gameplay: {
            u8 subtype;
            if (subtype == 0) bytes[0]; // state 2
            else bytes[*] ignored;
        };
    };
}; // Exact; the same opcode has state-dependent body consumption
struct S2C_C9_Cinematic {
    i64 duration_ms;
    vec3 focus;
    f32 radius;
}; // 24, Exact
struct S2C_CC_DebugPing { u64 timestamp; }; // 8, Exact
```

Opaque game-mode bodies are intentionally not declared with a guessed subtype:
adjacent families make a discriminator plausible, but that is insufficient
evidence to shift every unknown body by one byte.

The active gameplay state registers internal IDs `45`, `47`, `73`, `53`,
`61`, and `71` against the Chain, Quick, Tutorial, Arena, Juggernaut, and Kill
Race game callbacks respectively. This registration order is the authority
for the state-specific `0xad`, `0xaf`, `0xc8`, `0xb5`, `0xbd`, and `0xc6`
layouts above; it also prevents confusing internal IDs with their non-linear
attached wire IDs.

Internal ID `50` / wire `0xb2` has no `sub_A8FC00` sender construction, no
`sub_A8FDD0` receiver registration, and no gameplay dispatcher case in build
103. `CheatMessage` is therefore a reserved vocabulary entry, not an opaque
packet whose body should be guessed.

## Client requests that require server responses

```c
struct C2S_7F_HelloPlayerRequest {
    u64 user_id;
    optional u64 playgroup_id;
}; // exactly 8 or 16
struct C2S_88_BeamOut { u32 status; f32 progress; };
struct ActionCommon {
    u8 action_type;
    u8 action_sync;
    u16 reserved;
    u32 input_sync;
    u32 controlled_object_id;
    vec3 source_position;
    quat source_orientation;
}; // 40 bytes

struct C2S_9C_ActionCommand {
    ActionCommon common;
    variant by common.action_type {
        case 0:  bytes[0] reserved_unreachable;
        case 1:  bytes[12] reserved_unreachable;
        case 2:  bytes[16] reserved_unreachable;
        case 3:  { u32 reserved; vec3 goal; u32 goal_flags; u32 trailing; } move;
        case 4:  bytes[24] stop_tail;
        case 5:  u32 creature_index;
        case 6:  u8 overdrive_request;
        case 7:
        case 8:  {
            u32 target_object_id;
            vec3 cursor;
            vec3 target_position;
            u32 ability_index;
            i32 rank;
            u32 request_flag;
            u32 user_data;
        } ability;
        case 9:  {
            u32 target_object_id;
            vec3 position;
            i8 selector;
            u8 padding[3];
        } catalyst;
        case 10: bytes[24] ignored_uninitialized_cancel_tail;
        case 11: u32 target_object_id;
        case 12: bytes[0] dance;
    };
};
struct C2S_CB_LootDrop { u64 persistent_item_id; };
```

The type-4 stop and type-10 cancel constructors write only the common header.
Their nominal 24-byte tails are copied from uninitialized stack storage and no
recovered consumer reads them. The server therefore authenticates the common
header and ignores those tail bytes. The complete reachable constructor set is
type `3/4` movement, type `5` squad switch, type `6` Overdrive, type `7/8`
ability, type `9` catalyst, type `10` cancel, type `11` interactable, and type
`12` dance.

## Interaction safety boundary

```text
create
-> component baselines
-> presentation/locomotion
-> client selection and admission
-> exact C2S request
-> authoritative validation
-> terminal response or proven response-free completion
-> acquisition/effect/state mutation
-> object deletion
-> abort/disconnect/reload cleanup
-> same-process next-session validation
```

If any admission, completion, cancellation, or teardown step is unknown, the
default server must not publish that object as interactive. Fang `/reset` is a
diagnostic recovery aid, not a substitute for this contract.

## Inbound incomplete-packet tracing

The gameplay dispatcher can log every incoming packet in a currently
incomplete client message family before another handler consumes or rejects
it. No catalogued family remains in that special set. If later analysis
demotes a family, add it back until its exact contract is recovered.

The default ignored-opcode path also logs the complete hexadecimal payload,
including a hypothetical `0xb2` sender despite its exhaustive unconsumed
classification. This preserves evidence for an unexpected client sender without pretending
that the opcode is implemented. Exact inbound messages `0x7f`, `0x88`, `0x9c`,
`0xcb`, and `0xcc` do not produce the incomplete warning.

## Priority recovery queue

1. Complete equipment, catalyst/crystal, orb, DNA, and obelisk transactions
   across `0x8c`, `0x94`, `0x98`, `0x9a`, `0x9b`, `0x9c`, `0xa8`, and delete.
2. Finish all fields/nested masks in `0x8c`, `0x94`, `0x9b`, and `0xa1`.
3. Recover chain result/cash-out `0xa9` through `0xae`.
4. Recover PvP subtypes after campaign interactions survive abort, reconnect,
   reload, and same-process restart.

## Sources

- `bin/game/GameBin/Game.c` and `Game.idb`: canonical build-103
  receiver, registration, reflection, and callback evidence.
- `server/raknet/types.go`: current opcode vocabulary.
- `server/raknet/application.go`: current encoders and explicit gaps.
- `notes/design/architecture/raknet-gameplay-exchange.md`: framing, handler
  addresses, clocks, and recovered sequences.
- `notes/protocol/interaction.md`: interaction lifecycle audit.
- `bin/server/darkspin/logs/traces`: machine-readable runtime captures.
