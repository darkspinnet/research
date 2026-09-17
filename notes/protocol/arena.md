# Arena packet family (build 5.3.0.103)

## Evidence and conventions

`[E]` is direct client evidence from the canonical
`bin/game/GameBin/Game.c`, canonical-IDB names/addresses, or matching
x86 in `Game.exe`. `[I]` is a semantic inference from how an exact field is
used. Integers and floats are little-endian. All structures below are packed.
Sizes are GMS bodies **after** the one-byte wire opcode; add one for the complete
application packet.

The canonical IDB name table places `kGmsArenaPlayerMsgs`,
`kGmsArenaLobbyGMSMsgs`, `kGmsArenaGameGMSMsgs`, and
`kGmsArenaResultsGMSMsgs` at `0x01030160`, `0x01030174`, `0x0103018c`, and
`0x010301a4`. `[E]` Their internal IDs are 51--54 and the GMS wire transform
produces `0xb3`--`0xb6`. `[E]`

A fresh canonical-IDB export was run with the environment IDA user profile and
`ida.exe -A`. It produced `bin/game/logs/ida-arena-name-xrefs-fresh.log` and
`bin/game/logs/ida-arena-api-xrefs-fresh.log`; the latter exhaustively enumerates
the `sub_A8FC00` send-constructor and `sub_A8FDD0` receiver-lookup call sites.
Those results, the IDB-derived canonical C file, and direct executable
disassembly agree on every address below. The installed IDA has no callable
Hex-Rays plugin (`HEX_RAYS_UNAVAILABLE`), so the checked-in canonical C remains
the decompiler evidence. The IDB was not modified.

| Wire | Internal | Name | Direction | Recognized body sizes |
|---|---:|---|---|---|
| `0xb3` | 51 | ArenaPlayer | C2S only | 1, 6 |
| `0xb4` | 52 | ArenaLobby | S2C only | 1280 |
| `0xb5` | 53 | ArenaGame | S2C only | 1, 2, 5, state-dependent |
| `0xb6` | 54 | ArenaResults | S2C only | 1275 |

## Transfer-ready wire layouts

```c
#pragma pack(push, 1)

/* 0xb3 / internal 51, C2S */
typedef struct ArenaPlayerEnterLobby {       /* body size 1 */
    uint8_t subtype;                         /* +0 = 0 */
} ArenaPlayerEnterLobby;

typedef struct ArenaPlayerEnterResults {     /* body size 1 */
    uint8_t subtype;                         /* +0 = 1 */
} ArenaPlayerEnterResults;

typedef struct ArenaPlayerAcceptMission {    /* body size 6 */
    uint8_t subtype;                         /* +0 = 2 */
    uint8_t accepted;                        /* +1 = 1 */
    int32_t deck_id;                         /* +2; selected local PVP squad record ID */
} ArenaPlayerAcceptMission;

/* Shared by 0xb4 subtype 3 and 0xb6 subtype 4. */
typedef struct ArenaRoundResult {            /* size 56 */
    uint32_t outcome;                        /* +0; value 3 marks a winning round */
    uint32_t field_04;                       /* +4 */
    uint64_t creature_resource[3];           /* +8 */
    float health_percent[3];                 /* +32 */
    float mana_percent[3];                   /* +44 */
} ArenaRoundResult;

typedef struct ArenaPlayerResult {           /* size 205 */
    uint64_t player_id;                      /* +0 */
    uint32_t avatar_id;                      /* +8 */
    uint8_t team;                            /* +12; compared with 1 and 2 */
    uint32_t kills;                          /* +13 */
    uint32_t deaths;                         /* +17 */
    float damage_dealt;                      /* +21 */
    float damage_taken;                      /* +25 */
    float healing_dealt;                     /* +29 */
    float healing_received;                  /* +33 */
    ArenaRoundResult round[3];               /* +37 */
} ArenaPlayerResult;

typedef struct ArenaResultsBody {            /* size 1274 */
    uint32_t selector_a;                     /* +0 */
    uint32_t selector_b;                     /* +4 */
    uint32_t count_a;                        /* +8 */
    uint32_t count_b;                        /* +12 */
    int32_t round_time_seconds;              /* +16 */
    uint32_t dna_amount[6];                  /* +20 */
    ArenaPlayerResult player[6];             /* +44 */
} ArenaResultsBody;

/* 0xb4 / internal 52, S2C */
typedef struct ArenaLobbySnapshot {          /* body size 1280 */
    uint8_t subtype;                         /* +0 = 3 */
    uint8_t lobby_flag;                      /* +1 */
    uint32_t lobby_word;                     /* +2, intentionally unaligned */
    ArenaResultsBody results;                /* +6 */
} ArenaLobbySnapshot;

/* 0xb5 / internal 53, S2C */
typedef struct ArenaGameSelectResults {      /* body size 2 */
    uint8_t subtype;                         /* +0 = 5 */
    uint8_t select_results;                  /* +1; 0 -> state 15, nonzero -> 16 */
} ArenaGameSelectResults;

typedef struct ArenaGameStateOnly {          /* body size 1 */
    uint8_t subtype;                         /* +0 = 7 or 8 */
} ArenaGameStateOnly;

typedef struct ArenaGameWord {               /* body size 5 */
    uint8_t subtype;                         /* +0 = 9 or 10 */
    uint32_t word;                           /* +1, intentionally unaligned */
} ArenaGameWord;

/* 0xb6 / internal 54, S2C */
typedef struct ArenaResultsSnapshot {        /* body size 1275 */
    uint8_t subtype;                         /* +0 = 4 */
    ArenaResultsBody results;                /* +1 */
} ArenaResultsSnapshot;

#pragma pack(pop)
```

The names `creature_resource`, `selector_*`, `count_*`, `field_04`, and
`lobby_*` are
deliberately conservative. Their offsets/types are exact; their domain meanings
are not established. `[E]` The `deck_id` field is resolved more
strongly: `ArenaLobby.AcceptMission` treats the SWF argument as a one-based
index into the local 64-byte PVP squad records, stores the chosen record's first
dword as the active PVP deck, and sends that dword. Treat it as the authenticated
player's PVP deck/squad ID, not a mission index. `[E]`
The three 64-bit creature values are resolved through the content lookup, so
“resource” is `[I]`. The remaining descriptive names are exact because the UI
uses `avatar%u.png`, `LABS_PVP_KILLS`, `LABS_PVP_DMGDEALT`,
`LABS_PVP_DMGTAKEN`, `LABS_PVP_HEALDEALT`, and `dnaAmount`. `[E]`

The shipped `PvPResults` Flash resource is `FlashUI.package` ordinal `314`,
type `0x278CF8F2`, instance `0x3D010369` (the case-insensitive FNV-1 hash of
`PvPResults`). Its native `FillPlayerResults` call constructs an exact six-value
array: proven `kills`, opaque integer `field_11`, proven `damage_dealt`,
`damage_taken`, and `healing_dealt`, then opaque float `field_21`. The
ActionScript summary renderer consumes indices `2`--`5` as its four combat
totals. Naming indices `1` and `5` as `deaths` and `healing_received` is `[I]`,
supported by the symmetric client-facing lifetime PVP stat contract; their
positions and types are `[E]`.

The same resource's `ArenaRoundResultsItemRenderer.setData` consumes round
array indices `3/4`, `6/7`, and `9/10` through `health1..3` and `mana1..3`
meters and calls `setPercent` for each. This directly resolves the two
three-float arrays as health and mana percentages. `[E]` Round array indices
`0` and `1` remain `outcome` and opaque `field_04`; the Flash renderer forwards
both to text fields but supplies no semantic labels in ActionScript. `[E]`

## `0xb3` ArenaPlayer: all C2S senders

The whole-program search has exactly three `sub_A8FC00(..., 51, ...)`
constructors. `[E]`

| Sender | Gate | Body |
|---|---|---|
| `sub_448350`, `cArenaLobbyState` entry | State entry; no local connected test | subtype 0, one byte |
| `sub_448520`, `cArenaRoundResultsState` entry | `sub_535CC0` connected test | subtype 1, one byte |
| `sub_401C70`, SWF callback `ArenaLobby.AcceptMission` | selected local PVP squad plus `sub_535CC0` | subtype 2, literal byte 1, selected PVP deck/squad ID; six bytes |

Each constructor writes exactly the declared size with `sub_A87AC0` and sends
through the transport virtual at `+4`. `[E]` The subtype-2 fields are emitted as
one byte plus five adjacent packed stack bytes, proving the unaligned dword at
body offset 2. `[E]`

There is no internal-51 receiver lookup/registration anywhere in the client.
`[E]`

## `0xb4` ArenaLobby: registration and subtype

`cArenaLobbyState` (`sub_448250`, vtable `0x00fd2470`) constructs its callback
wrapper in `sub_448650`; the wrapper dispatches to `sub_448470`. `[E]`

- `sub_448350` registers internal 52 at transport virtual `+44`, callback
  `this+8`, when ArenaLobby is entered. `[E]`
- `sub_448040` unregisters it at virtual `+48` on exit, behind the connected
  test. `[E]`
- `sub_448470` reads one subtype byte. Only subtype 3 calls `sub_4480E0`; every
  other value performs no Arena callback. `[E]`
- `sub_4480E0` reads exactly 1279 more bytes to state offset `+13` and sets the
  pending byte at state offset `+12`. Thus subtype 3 is exactly 1280 consumed
  body bytes. `[E]`
- `sub_447FE0` consumes the pending snapshot during the ArenaLobby update:
  `sub_4044F0` opens/populates the lobby from the 1279-byte block and
  `sub_404600` applies the embedded `ArenaResultsBody` at block offset 5
  (packet-body offset 6). `[E]`

Within the lobby prefix, the byte at offset 1 is forwarded as a flag and the
unaligned dword at offset 2 is tested for zero/nonzero by squad-selection UI
logic. Their stronger meanings are unknown. `[E]`

The six embedded 205-byte player records also populate the lobby roster.
`sub_403230` resolves each nonzero 64-bit player ID, reads avatar ID at record
offset `+8` and team at `+12`, and passes the entry to SWF `AddPlayer`. Empty
roster slots therefore use a zero player ID. `[E]` See
[`pvp-flow.md`](pvp-flow.md) for the end-to-end matchmaking and implementation
handoff.

## `0xb5` ArenaGame: five state gates

Internal 53 has five register/unregister pairs. All registration paths are
entered only while `sub_535CC0` reports a live game connection; registration is
transport virtual `+44`, unregistration is `+48`. `[E]`

| Owning state/object | Register / unregister | Callback construction | Recognized subtype behavior |
|---|---|---|---|
| `cStateMachine` size `0x38` (`sub_44B630`) | `sub_44AD80` / `sub_44B950` | `sub_451F70` -> `sub_44DE10` | 5 reads bool -> pending state 15/16; 8 -> state 6 |
| `cCinematicState` (`sub_44F4A0`) | `sub_44E100` / `sub_44E680` | `sub_4520C0` -> `sub_44ECA0` | 5 reads bool -> pending state 15/16 |
| `cSpectatorState` (`sub_44B6C0`) | `sub_44EEF0` / `sub_44BC60` | `sub_452210` -> `sub_44F230` | 5 reads bool -> pending state 15/16; 8 -> state 6 |
| `cStateMachine` size `0x6c` (`sub_4522C0`/`sub_4504D0`) | `sub_451A70` / `sub_450BB0` | `sub_4517E0` -> `sub_4511A0` | 5 reads bool -> pending state 15/16; 7 -> state 7; 9/10 read dword |
| `cPreDungeonState` (`sub_52B2F0`) | `sub_52B180` / `sub_52AF90` | `sub_52B440` -> `sub_52B390` | 9 reads dword |

The state labels above come from the allocation strings and vtables. `[E]`
The two differently sized objects both use the literal allocation name
`SP_SporeLabs/cStateMachine`; distinguishing them by size is therefore
necessary. `[E]`

Detailed effects:

- Subtype 5 helpers `sub_44B1B0`, `sub_44E480`, `sub_44B440`, and
  `sub_4508D0` read exactly one byte. They store `(byte != 0) + 15` and schedule
  a transition: zero selects named ArenaLobby state 15, nonzero selects named
  ArenaRoundResults state 16. `[E]`
- Subtype 8 calls `sub_44B100`, which selects numeric game state 6. `[E]`
- Subtype 7 calls `sub_450950`, which selects numeric game state 7. `[E]`
- Subtypes 9 and 10 call `sub_4509A0`, read exactly one dword, and store it at
  global player-state offsets `+1524` and `+1528`, respectively. `[E]`
- PreDungeon callback `sub_52B390` also reads a discriminator first and accepts
  only subtype 9; `sub_52B030` then reads the dword and stores it at `+1524`.
  Its body is therefore five bytes, not a raw four-byte state variant. `[E]`

The same opcode is intentionally interpreted through the callback registered
by the current state. For example, subtype 8 is ignored by Cinematic,
PreDungeon, and the size-`0x6c` state machine, while subtype 7/9/10 are ignored
by the other gates. `[E]`

## `0xb6` ArenaResults: registration, layout, callback

`cArenaRoundResultsState` (`sub_4482D0`, vtable `0x00fd249c`) constructs its
callback wrapper in `sub_448A50`; the wrapper dispatches to `sub_4489A0`. `[E]`

- `sub_448520` registers internal 54 at virtual `+44`, callback `this+8`, on
  connected entry, immediately before sending `0xb3` subtype 1. `[E]`
- `sub_4481A0` unregisters it at virtual `+48` on connected exit. `[E]`
- `sub_4489A0` reads one subtype byte. Only subtype 4 calls `sub_4488E0`; all
  other values have no Arena callback. `[E]`
- `sub_4488E0` reads exactly 1274 bytes, invokes `sub_4486A0` for localized
  result statistics, passes the same block to `sub_404600`, enables results/UI
  state, and selects named state 16. `[E]`

The shared result record bounds are fixed: six 205-byte player records, three
56-byte rounds per player, and three creature slots per round. `[E]` Header
offset 16 is formatted as minutes/seconds. Header offsets 0 and 4 choose a
team/result selector (the second overrides the first when nonzero), while
offsets 8 and 12 are displayed numeric counts; stronger labels would be
inference. `[E]`

## Exhaustive negative findings

The following are whole-file negative results, not assumptions:

- No internal-51 receiver registration exists; no internal-52, -53, or -54
  `sub_A8FC00` send constructor exists. Therefore the directions are C2S-only
  for `0xb3` and S2C-only for `0xb4`--`0xb6`. `[E]`
- Internal 52 has only the ArenaLobby register/unregister pair; internal 54 has
  only the ArenaRoundResults pair; internal 53 has exactly the five pairs
  listed above. `[E]`
- There are no `0xb3` constructors for subtypes other than 0, 1, and 2; no
  recognized `0xb4` subtype other than 3; and no recognized `0xb6` subtype
  other than 4. `[E]`
- Across all discriminator-based `0xb5` callbacks, the union of recognized
  subtypes is exactly 5, 7, 8, 9, and 10. PreDungeon does not add a raw variant;
  it recognizes the ordinary subtype-9 form. `[E]`
- An unrecognized subtype path reads only the one-byte discriminator and calls
  no Arena handler/state mutation. No exact-payload-length rejection was found,
  so this proves consumed bytes, not that transport-level trailing bytes are
  rejected. `[E]`
- No variable-length Arena field or dynamic count controls a read. Every
  recognized body read is fixed; result loops have compile-time bounds
  6/3/3. `[E]`
- No client evidence assigns stable meanings to the remaining explicitly opaque fields
  retained above. In particular, the selected record dword is not a mission ID;
  it is the selected local PVP deck/squad ID. The result selectors/counts and
  per-round `field_04` should not be renamed without server captures or further
  content evidence. `[E]`
