# Fang build and hook boundary

## Build modes

Fang has two compile-time modes selected through the `fangdebug` Go build tag.
The tag is translated into the C preprocessor definition
`FANG_DIAGNOSTICS` by build-tagged cgo files:

- `mage darkspinner:build` compiles Fang with `-tags fangdebug`, retains
  observation hooks, and writes the development runtime to `bin/darkspinner`.
- `mage darkspinner:buildCI` also compiles Fang with `-tags fangdebug` during
  the early reporting phase, strips Go symbol and DWARF tables with `-s -w`,
  and writes the isolated runtime to `bin/darkspinnerci`. Windows hosts produce
  the Windows executable and `darkspinner-win32-v<version>.zip`; Linux hosts
  additionally produce the native executable and
  `darkspinner-linux-v<version>.zip`. The GitHub workflow runs this target on
  Ubuntu 24.04 and replaces both assets on the rolling `latest` release after
  every push to `main`.

Both current Darkspinner targets open the client trace file and install the
observation hooks so `/bug` reports can capture the same evidence from CI
players as from developers.

## Required compatibility hooks

These hooks are currently necessary to connect build 103 to the local runtime:

- hostname, Winsock, and WebKit network redirection;
- legacy TLS verification/plaintext compatibility;
- JWT login submission and legacy launcher handoff;
- configured cinematic skipping;
- packaged-web callback timing needed by profile and owned-account pages.

## Behavior-changing hooks still shared by both builds

These are not diagnostics and therefore remain in both modes. They require
continued review against the rule that normal gameplay should be repaired
through server contracts rather than client overrides:

- packaged-web profile callback replay and owned-account refresh;
- a signed Map Room AVM1 literal correction applied to Scaleform's decompressed
  stream before parsing, exposing unranked 1v1 to a solo Crogenitor while
  retaining the packaged 2v2 and custom-party gates.

Ability attention is no longer on this list. Fang does not override
`PlayAbilityBlink` availability; debug builds may observe its predicate while
returning the client's original result unchanged.

The former non-tutorial first-loot latch write is also removed. Build 103 owns
that one-shot editor tooltip; Fang does not suppress it by mutating simulator
memory.

The former squad portrait deadline translation is removed as well. Fang no
longer rewrites `LabsPlayerUpdate` field `4`; debug builds only observe the
incoming reflection. The correct local-clock deadline remains a server
protocol research item.

## Observation-only hooks

Diagnostic Fang instruments scene and asset resolution, packet
construction and decoding, cooldown clocks, ability descriptors, loot
conversion and formatting, mana-cost calculation, internal alerts, login
errors, cinematic/UI events, and the reserved developer effect-preview
envelope. Ordinary observation wrappers call the original client function and
preserve its result. The effect-preview decoder is intentionally retained in
both current Darkspinner targets during the early reporting phase.
The custom slash-command bridge, chat UI initialization, clipboard paste,
local `/reset`, and `/exit` handling are compatibility hooks shared by both
builds so release players can use ordinary chat, `/bug`, and the server-owned
command set. Opening chat also bootstraps and repairs the native Rooms target
when a checkpoint Continue enters gameplay without traversing the ship-room
flow; the command converter alone is insufficient because the client otherwise
recognizes a Darkspin command but silently drops it before Messaging transport.
Build 103 exposes no player-facing arbitrary-level picker for disconnected
locations. `sub_527C80` binds `PlanetScreen.AcceptMission` to the authored
mission-selection flow, while the internal `SP_Simulator/LevelName` setup in
`sub_4F3160` has no recovered UI or console entry point. Packaged level content
does include test destinations such as `test_AI_zoo`, so Fang owns the valid
`/warp` location list and canonicalizes an exact or unique case-insensitive
partial match before forwarding it to the server. Exact matches take priority;
missing or ambiguous matches fall through to the server's numbered catalog.
Fang additionally maps `/warp <1-37>` to the matching canonical entry in the
special-purpose area list pasted from the level catalog. Bare `/warp` remains
bare across the compatibility hook so the server can return that list; named
matching continues to cover every packaged level, including ordinary campaign
maps that are intentionally absent from the numeric aliases.
Fang likewise owns the curated `/spawn` catalog of packaged targetable combat
nouns, including the six Destructor families, ordinary director actors,
tutorial actors, and known Captains. It accepts an exact noun stem or full
`.Noun` name and canonicalizes a unique case-insensitive partial; the server
independently limits execution to imported classes in an active warped zone.
The `/loc` command uses the same bridge without client-side coordinates; its
reply comes from the authoritative gameplay session position on the server.
Stale held-input recovery and scene-transition combat/input
cleanup are also retained in both current targets to maximize actionable bug
evidence while the project is early.

Sync Snapshot adds observation-only hooks in both Fang build modes. When its
mode is `manual` or `auto`, Fang keeps a bounded ring containing only gameplay
RakNet bytes and client game-object state. It records full datagrams, complete
reconstructed application messages, frame correlation, locomotion state before
and after native receivers, and a dump-boundary image of validated live object
registry slots. The `/ss` command uses the same custom-command
forwarding bridge as `/bug` so the retail parser delivers them to Blaze
Messaging. Each dump request also carries a server UTC nanosecond stamp that
Fang copies into its monotonic boundary marker for replay alignment. The object keyframe runs on the game thread because the native
registry accessor is thread-local. `off` clears the ring, and all hooks preserve
the client's original control flow and return values. See
[`notes/protocol/snapshot.md`](../protocol/snapshot.md) for commands and artifact
layout.

The removed `LootAwarded` chat suppression is not present in either mode.
The client retains its native inventory update and localized notification.
