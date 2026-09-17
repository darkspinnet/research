# Build-103 Kill Race packet family

Build `5.3.0.103` attaches internal GMS IDs `69`-`72` to wire IDs
`0xc4`-`0xc7` in this order:

| Wire | Internal | Attached name | Proven build-103 direction |
| --- | ---: | --- | --- |
| `0xc4` | 69 | `KillRacePlayerMsgs` | C2S only |
| `0xc5` | 70 | `KillRaceLobbyMsgs` | Unconsumed |
| `0xc6` | 71 | `KillRaceGameMsgs` | S2C only |
| `0xc7` | 72 | `KillRaceResultsMsgs` | S2C only |

“Only” above is an exhaustive build-103 finding, not a claim about the retired
server. The evidence is the canonical `Game.idb`, checked against
`Game.c`. The IDB-wide `sub_A8FC00` constructor scan found exactly two
family call sites, both internal ID `69`. The `sub_A8FDD0` wire-lookup scan
found no ID-`69` lookup, one ID-`70` removal, four ID-`71` installs plus four
removals, and one ID-`72` install plus one removal.

## Transfer-ready layouts

Integers are little-endian. Offsets are payload offsets after the RakNet wire
opcode.

```c
/* 0xc4 / internal 69: exact client constructors */
struct C2S_C4_KillRacePlayer_ResultsEntered {
    uint8_t subtype;                    /* +0x00 = 1 */
};                                      /* fixed 1 byte */

struct C2S_C4_KillRacePlayer_UIAction {
    uint8_t subtype;                    /* +0x00 = 3 */
    uint8_t constant_one;               /* +0x01 = 1; meaning not recovered */
    uint8_t uninitialized_stack[4];     /* +0x02, copied but never initialized */
};                                      /* fixed 6 bytes */

/* 0xc5 / internal 70: no build-103 constructor or installed receiver */
struct KillRaceLobby_Unconsumed {
    uint8_t unresolved[];
};

/* 0xc6 / internal 71: exact bytes consumed by every installed callback */
struct S2C_C6_KillRaceGame_Exit {
    uint8_t subtype;                    /* +0x00 = 7 */
    uint8_t opaque_exit[7];             /* +0x01; forwarded byte-for-byte */
};                                      /* receiver consumes 8 bytes */

struct S2C_C6_KillRaceGame_SelectResults {
    uint8_t subtype;                    /* +0x00 = 8 */
};                                      /* receiver consumes 1 byte */

struct S2C_C6_KillRaceGame_Other {
    uint8_t subtype;                    /* +0x00, any value except 7 or 8 */
    uint8_t ignored_tail[];             /* never read by build 103 */
};

/* 0xc7 / internal 72: exact bytes consumed by the results-state callback */
struct S2C_C7_KillRaceResults {
    uint8_t subtype;                    /* +0x00; subtype 5 is explicitly tested */
    uint8_t ignored_tail[];             /* no subtype reads another byte */
};
```

The four trailing bytes of C2S `c4/3` are not inferred fields. At
`sub_438380` (`0x00438380`), the constructor reserves six bytes, writes the
one-byte subtype, then copies five bytes beginning at a local for which only
the first byte was assigned. IDB instructions `0x004383EC`-`0x00438425`
confirm that no instruction initializes the next four bytes. A compatible
encoder should zero them deliberately rather than reproduce the information
leak; zero is a conservative normalization, not captured retail semantics.

For `0xc6` and `0xc7`, the stated sizes are consumed sizes. The message reader
does not compare total payload length after those reads, so a longer packet's
tail is ignored rather than decoded. No dynamic length, count, string, or
reflection reader is reachable in this family.

## `0xc4` senders

### Subtype 3: UI action

`sub_438380` is an event callback referenced once from the
`SP_UI/cKillRaceLobbyUI` object table. It returns false without sending unless:

1. the event object at argument `+8` equals exact type hash `0x287259f6`;
2. its virtual method at `+0x1c` returns exact event hash `0x08feb1ee`;
3. `sub_535CC0` reports the gameplay transport attached.

It then creates internal ID `69`, fixed size `6`, writes subtype `3`, and
writes the five-byte local described above through `sub_A87AC0`. The event
hash names are not recovered; calling this a lobby UI action is exact from its
owner, while a more specific button meaning would be inference.

### Subtype 1: results-state entry

`sub_452D90` is the `+0x1c` entry method in the exact
`SP_SporeLabs/cKillRaceResultsState` vtable `off_FD2A2C`. After the ordinary
state-entry setup it checks `sub_535CC0`. On success it first installs the
`0xc7` callback, then creates internal ID `69`, fixed size `1`, and sends
subtype `1`. No other field is present.

There are no internal-69 receive lookups or registrations anywhere in the
IDB, and no other internal-69 constructors.

## `0xc5` lobby finding

The exact `SP_SporeLabs/cKillRaceLobbyState` entry method `sub_452A90` creates
the lobby presentation state but installs no internal-70 callback.
`sub_452B40` lazily creates `SP_UI/cKillRaceLobbyUI`; neither it nor the UI
object installs one. The exit method `sub_452C30` nevertheless maps internal
ID `70` and calls the transport's callback-removal slot (`vtable +0x30`).

Thus build 103 contains a cleanup for a receiver it never installs. The IDB
has no internal-70 constructor, sender, callback install, subtype reader, or
body reader. This is stronger than “body unknown”: wire `0xc5` is unconsumed
in this build.

## `0xc6` receivers and state gates

Every installed internal-71 callback has the same complete subtype contract:

```c
switch (read_u8(+0x00)) {
case 7:
    read_bytes(+0x01, opaque_exit, 7);
    GThread::OnExit(opaque_exit);        /* shared sub_44B580 */
    break;
case 8:
    pending_state = 20;
    sub_4EF8B0(2, 1.0f);
    break;
default:
    break;                              /* no more bytes, no mutation */
}
```

The seven-byte helper `sub_44B580` reads one `u32`, one `u16`, and one `u8`
only as copy widths; it immediately reconstructs the same packed seven bytes
on the stack and passes them to `GThread::OnExit`. No consumer in this path
assigns meanings to offsets `+0x01`-`+0x07`.

Four state-entry methods install the callback through transport vtable
`+0x2c`, each only when `sub_535CC0` says the gameplay transport is attached:

| Entry / owner evidence | Callback storage | Callback | Subtype-8 destination |
| --- | ---: | --- | ---: |
| `sub_44AD80`; six-family gameplay entry | `state+0x1c` | `sub_44DFB0` | `state+0x30 = 20` via `sub_44B290` |
| `sub_44E100`; exact `SP_SporeLabs/cCinematicState` vtable | `state+0x1c` | `sub_44EE20` | `state+0x54 = 20` via `sub_44E560` |
| `sub_44EEF0`; exact `mixmode_observer` setup | `state+0x10` | `sub_44F3D0` | `state+0x28 = 20` via `sub_44B520` |
| `sub_451A70`; main active-state vtable `off_FD2900` | `state+0x60` | callback at `0x00451380` | `state+0x44 = 20` via `sub_450B10` |

`sub_451A70` has one additional outer gate: it skips all setup when the byte
at `virtual_method_+0x14(state)+0x1c` is nonzero. When setup runs, it installs
all mode-family callbacks, including internal `71`. The other three rows are
their respective state-entry paths and have no Kill-Race-specific predicate
beyond the transport-attached check.

The four internal-71 removals are exact at `sub_44B950`, `sub_44BC60`,
`sub_44E680`, and `sub_450BB0`; each maps `71` and calls transport vtable
`+0x30`. They are lifecycle cleanup, not additional receivers.

No installed callback accepts a different Kill Race subtype. In particular,
there is no hidden field behind subtype `8`: all four helpers mutate the
pending-state word and destroy the read context without another read.

## `0xc7` receiver

`sub_452F50`, the `+0x08` method of
`SP_SporeLabs/cKillRaceResultsState`, creates one delegate whose callback is
`sub_452EB0`. Results-state entry `sub_452D90` installs that delegate for
internal ID `72` at transport vtable `+0x2c`, gated only by
`sub_535CC0`. Results-state exit `sub_452960` maps `72` and removes it through
transport vtable `+0x30`.

`sub_452EB0` reads exactly one byte. It explicitly compares subtype `5`; that
branch only constructs and destroys a second read-context wrapper and performs
no callback or state mutation. Every other subtype falls through with the
same externally visible result. No subtype reads a second byte.

There is no internal-72 constructor or client sender.

## Exhaustive negative findings

- The generic gameplay dispatcher `sub_53ADC0` has no cases `69`-`72`; the
  valid `0xc6`/`0xc7` receivers above are state-owned registrations.
- The simulator C2S dispatcher `sub_9C1580` accepts only internal IDs `9`,
  `29`, `67`, `76`, and `77`; it supplies no reverse-direction evidence for
  this family.
- IDB-wide constructor callsites contain two internal-69 calls and zero
  internal-70, `-71`, or `-72` calls.
- IDB-wide wire lookups contain zero internal-69 calls. Internal `70` occurs
  once, solely in removal. Internal `71` occurs in four installs and four
  removals. Internal `72` occurs in one install and one removal.
- No `0xc4` S2C callback, `0xc5` callback in either direction, `0xc6` C2S
  sender, or `0xc7` C2S sender exists in build 103.
- There are no additional Kill Race subtype switches, payload reflections,
  string readers, array readers, length fields, or callbacks beyond those
  listed above.

The direction labels formerly shown as “Both” for this family should therefore
not be transferred into the canonical packet map; they were vocabulary-level
placeholders, not build-103 callsite evidence.
