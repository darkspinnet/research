# Juggernaut packet family (5.3.0.103)

Scope: application bodies for internal IDs 59-62 / wire opcodes
`0xbb`-`0xbe`. Sizes below exclude the one-byte wire opcode and the RakNet
envelope. Multibyte fields are little-endian. **Exact** means directly observed
in `Game.c` and/or the canonical IDB; **inference** is called out
explicitly.

## Contract summary

| Wire | ID | Observed direction | Body contract | Live state gate |
|---|---:|---|---|---|
| `0xbb` | 59 | C2S only | exactly one emitted byte: `0` or `1` | sender context: Juggernaut results (18) or lobby (17) |
| `0xbc` | 60 | none | no evidenced body | none |
| `0xbd` | 61 | S2C only | subtype 3: intended 8 bytes; subtype 4: intended 1 byte | Dungeon (6), Observer (7), Cinematic (8), Spectator (9) |
| `0xbe` | 62 | S2C only | subtype 2: intended 1 byte | Juggernaut results (18) |

`sub_A8FC00(..., internal_id, capacity)` constructs an outgoing message;
`capacity` is allocation capacity, not an emitted field or body length.
`sub_A87AC0` appends the actual body bytes. `sub_A8FDD0` translates an internal
ID for receiver attach/detach. Transport vtable slots `+44` and `+48` attach
and detach a callback, respectively.

## Transfer-ready wire declarations

```c
#pragma pack(push, 1)

/* Internal 59, wire 0xbb, C2S. Both observed messages have body size 1. */
typedef struct C2S_JuggernautPlayer {
    uint8_t subtype; /* +0: exact observed values 0 and 1 */
} C2S_JuggernautPlayer;

enum C2S_JuggernautPlayerSubtype {
    JUGGERNAUT_PLAYER_RESULTS_STATE_ENTER = 0, /* contextual name: inference */
    JUGGERNAUT_PLAYER_LOBBY_UI_EVENT       = 1, /* contextual name: inference */
};

/* Internal 61, wire 0xbd, S2C, subtype 3, body size 8. */
typedef struct S2C_JuggernautGameSubtype3 {
    uint8_t  subtype;  /* +0 = 3 */
    uint8_t  field_01; /* +1; copied to shared object +2 in Dungeon */
    uint16_t field_02; /* +2; copied to shared object +4 in Dungeon */
    uint16_t field_04; /* +4; copied to shared object +6 in Dungeon */
    uint16_t field_06; /* +6; copied to shared object +8 in Dungeon */
} S2C_JuggernautGameSubtype3;

/* Internal 61, wire 0xbd, S2C, subtype 4, body size 1. */
typedef struct S2C_JuggernautGameSubtype4 {
    uint8_t subtype; /* +0 = 4; requests transition to state 18 */
} S2C_JuggernautGameSubtype4;

/* Internal 62, wire 0xbe, S2C, subtype 2, body size 1. */
typedef struct S2C_JuggernautResultsSubtype2 {
    uint8_t subtype; /* +0 = 2; explicit handler branch has no mutation */
} S2C_JuggernautResultsSubtype2;

#pragma pack(pop)

/*
 * Internal 60 / wire 0xbc has no evidenced endpoint or body declaration in
 * this build. Do not infer an opaque/dynamic struct from its assigned name.
 */
```

The subtype-3 field meanings are unknown. Calling its seven trailing bytes an
“exit record” is only an inference from the effects in three callbacks, not a
recovered protocol name.

## Exact endpoint evidence

### `0xbb` / internal 59

There are exactly two constructor call sites among all 36 IDB xrefs to
`sub_A8FC00`, and both append exactly one byte:

- `sub_437CB0`, constructor call `0x00437d25`: after checking event hashes
  `0x287259f6` and `0x08feb1ee` and requiring `sub_535CC0(network)` to succeed,
  it emits byte `1`. The callback is installed by the state-17
  `SP_UI/cJuggernautLobbyUI` setup. The event's human-readable meaning is
  unknown.
- `sub_452710`, constructor call `0x004527fc`: on entry to
  `cJuggernautResultsState` (state 18), after the same network-presence gate and
  after registering the `0xbe` receiver, it emits byte `0`.

The constructor capacities are 6 and 1, respectively, but the emitted body is
one byte in both cases. No receiver registration for internal 59 exists.

### `0xbd` / internal 61

Every registration is gated by `sub_535CC0(network)`. Each state registers on
entry and unregisters on exit:

| State | Registration | Callback object / function | Unregistration |
|---|---|---|---|
| 6 Dungeon | `sub_451A70` at `0x00451c89` | `this+92` / `sub_4512B0` | `sub_450BB0` at `0x00450cc0` |
| 7 Observer | `sub_44AD80` at `0x0044aeff` | `this+24` / `sub_44DEE0` | `sub_44B950` at `0x0044ba4f` |
| 8 Cinematic | `sub_44E100` at `0x0044e2a6` | `this+24` / `sub_44ED50` | `sub_44E680` at `0x0044e77f` |
| 9 Spectator | `sub_44EEF0` at `0x0044f194` | `this+12` / `sub_44F300` | `sub_44BC60` at `0x0044bcde` |

All four callbacks first read one byte at body offset 0 and implement the same
switch:

- Subtype 3 reads seven more bytes. Dungeon calls `sub_450A90`, which copies
  the packed `u8,u16,u16,u16` fields to shared-object offsets `+2,+4,+6,+8`.
  Observer, Cinematic, and Spectator call `sub_44B580`, which consumes the same
  seven-byte record and then invokes the common exit/cleanup path.
- Subtype 4 reads nothing further, stores state ID 18 in the state-specific
  next-state field, and calls `sub_4EF8B0(2, 1.0)`. State 18 is exactly
  `GameJuggernautResults`.
- Every other subtype consumes only the subtype byte and performs no branch
  action.

There is no count, length field, terminator, or dynamic body in either active
form.

### `0xbe` / internal 62

`sub_452710` registers the sole receiver at `0x004527d0` on entry to
`cJuggernautResultsState` (18), using callback object `this+8`; `sub_452910`
initializes that object with `sub_452870`. `sub_452360` unregisters it at
`0x004523b7` on exit. Registration is gated by
`sub_535CC0(network)`.

`sub_452870` reads one subtype byte. Subtype 2 has an explicit branch that
creates and destroys its read-context wrapper but reads no additional fields
and performs no state mutation. Other subtype values likewise have no action
after the first byte. The semantic purpose of subtype 2 is unknown.

## Decoder permissiveness

`sub_A87BF0` copies `min(requested, remaining)` and advances only by the copied
amount; these handlers do not enforce an exact final body length. Consequently:

- trailing bytes after any recognized or unrecognized subtype are ignored;
- an empty body can leave the initialized subtype at its no-op value;
- a truncated subtype-3 record can leave unread local bytes unchanged or
  uninitialized before the callback uses the record.

Thus 8/1/1 are the intended active body sizes evidenced by the reads, not
strict decoder validation. This permissiveness is exact; accepting malformed
forms as a server contract would be an inference and is not recommended.

## Exhaustive negative findings

- The complete 36-xref scan of `sub_A8FC00` found only the two internal-59
  senders above: none for internal IDs 60, 61, or 62.
- The complete `sub_A8FDD0` caller scan found no attach/detach use for IDs 59
  or 60; exactly four attach/four detach sites for ID 61; and exactly one
  attach/one detach site for ID 62.
- The canonical IDB has no direct xrefs to the four packet-name strings at
  `0x01030218`, `0x01030234`, `0x01030250`, and `0x0103026c`; names alone do
  not establish endpoint behavior.
- Internal 60 / `0xbc` has no sender, receiver registration, callback,
  subtype switch, body read, size operation, or state gate. Its direction and
  size are unknown.
- There is no observed S2C endpoint for `0xbb`, no observed C2S endpoint for
  `0xbd` or `0xbe`, no `0xbd` receiver outside states 6-9, and no `0xbe`
  receiver outside state 18.
- State 17's Juggernaut lobby installs the `0xbb` UI sender callback but
  registers no member of this four-opcode family.
- No active member has a dynamic-length payload, nested subtype switch,
  optional evidenced field, or recognized subtype beyond those listed above.

Direction statements describe endpoints present in this client build. They do
not prove that a server could never send an unregistered opcode or receive an
opcode for which this client has no sender.
