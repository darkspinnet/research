# Shift-held direct client session

## Implemented launcher entrypoint

Darkspinner now exposes a detached-client action on its Launcher page. The
primary UI remains open to supervise the existing auth and server services,
while a short-lived helper process requests a fresh JWT for the selected local
profile and injects a second `Game.exe` independently. The helper bypasses
the Wails single-instance UI lock through the private `--detached-game` mode,
uses profile-specific trace and writable configuration paths, and never reuses
the primary client's credential. This provides the practical second-session
path without requiring Shift interception or Steam metadata changes.

Build 103 defaults every process to the same writable Windows roaming-AppData
tree. That tree owns WebCookies, temporary web data, preferences, and other
process-written client state, so bypassing only the global process guard is
insufficient for a concurrent login. Every managed launch passes the recovered
native `userDataDir:<path>` argument and uses a stable, profile-named and
hash-stabilized directory beneath
`darkspin/config/profiles/<profile>-<key>`. Darkspinner also redirects
`USERPROFILE`, `APPDATA`, and `LOCALAPPDATA` into that directory: the native
argument alone only moves the Documents-style folders. Attached and detached
launches for the same profile therefore reuse that profile's private settings,
while different profiles never share preferences, web cookies, server packages,
or creature caches. Darkspinner never reads or seeds from the retail Windows
AppData tree. Existing detached configuration is moved forward from the former
`darkspin/client-data/detached` location on first use. Profile preparation also
creates or normalizes `OptionFullScreen 0` in the isolated preferences so both
attached and detached clients start windowed even when an abrupt exit did not
persist an Alt+Enter change.

Build 103 enforces ordinary single-client startup with the named global guard
`Global\\SporeLabs`. Its recovered startup branch skips that guard when the
native `multipleInstances` argument is present. The detached helper therefore
adds `-multipleInstances` only to its child game's arguments; ordinary Launch
behavior retains the retail single-client guard.

Each detached attempt writes a profile-and-UTC-stamped client trace and a
process-and-UTC-stamped helper failure log. On Windows, the managed launch
proxy now recognizes that the launcher already injected and initialized Fang;
it does not initialize Fang a second time, reopen the trace with
`CREATE_ALWAYS`, or attempt to patch the login hooks twice. Wine continues to
use the proxy-owned initialization path because its launcher does not inject
Fang remotely.

Every managed launch also receives a random launch ID and a dedicated failure
result path beneath `darkspin/logs/client-results`. The injecting helper
atomically registers the launch ID and game PID before resuming Game, so
the main launcher tracks normal and failed clients from process creation. A
detached request checks those live registrations and rejects a profile already
owned by an attached or detached process; one profile therefore never shares
its private AppData tree concurrently. After JWT submission, Fang allows eight
seconds for the first owned-account callback. A silent timeout writes
an atomic JSON result containing the launch ID and game PID before closing only
that client. The main launcher watches all pending results, waits for the bound
PID to exit, delays briefly for final file flushes, publishes the recorded
reason through the launcher's inline Last Run status, and renames the result to
`.shown.json`. Detailed launcher and process activity remains file-only instead
of being mirrored into a scrolling interface log. This correlation remains
unambiguous when attached and multiple detached clients overlap.

Standard and detached launch paths do not treat an unrelated Game process
as a conflict. The launcher checks ownership only for the selected profile,
detects a newly attached launch through its exact launch-ID registration, and
offers to close only the conflicting profile's registered PID. Other attached
or detached clients remain untouched.

The remaining Shift and Steam Help ideas below are optional convenience
entrypoints into this same account-selection and detached-launch contract.

## Feasibility

A Shift-held direct `Game.exe` session is technically feasible without
starting the Darkspinner UI, but the detector must be the startup proxy rather
than Fang. Fang cannot observe process startup until something has already
loaded `fang.dll`. The existing Windows Steam integration installs the
Darkspinner-owned `VERSION.dll` proxy beside `Game.exe`; Windows loads
that proxy during executable import resolution, and its `DllMain` worker is the
earliest Dark Spin-owned code in an unmanaged direct launch.

The proxy currently has two paths:

- a managed launch with `DARKSPIN_FANG_DLL` loads Fang and calls
  `RecapInitializeThread`;
- an unmanaged direct launch starts `darkspinner.exe --steam-launch` and exits
  the original client.

Therefore the direct-start detector already exists at the required process
boundary. Sampling `GetAsyncKeyState(VK_SHIFT) & 0x8000` in
`initialize_proxy`, outside `DllMain`, can select an additional path. Sampling
both `VK_LSHIFT` and `VK_RSHIFT` would preserve which physical key was held.

## Required headless handshake

Shift alone is not enough to create an authenticated second client. Fang needs
a fresh short-lived `DARKSPIN_LAUNCH_JWT`, and the local authentication broker
must know which retained profile owns it. A second launch must not reuse the
first client's token or account binding.

The safe flow is:

1. The proxy detects Shift on its existing worker thread.
2. It starts a small server-owned session helper with inherited anonymous pipe
   handles. The helper, not the DLL, owns profile selection and the existing
   `/api/desktop/start` then `/api/desktop/exchange` flow.
3. The helper returns a bounded session envelope containing the selected
   profile, launch JWT, Fang path, server endpoint, and expiry.
4. The proxy validates the envelope, sets the launch environment in the
   current process, loads Fang, calls `RecapInitializeThread`, clears the JWT,
   and lets the original `Game.exe` continue.
5. Failure leaves the ordinary unmanaged launch behavior available; it must
   never silently enter with the first client's identity.

This avoids a visible Darkspinner process and avoids relaunching the client.
It also keeps HTTP, profile policy, token parsing, and server startup out of
`DllMain` and the C proxy. A named pipe can replace anonymous pipes if the
helper must outlive startup, but it needs per-launch authentication and a
strict local-user ACL.

## Second-profile policy

The least surprising initial policy is an explicit helper default such as
profile 2, with a later broker operation that selects the first retained
profile not currently holding a Blaze session. Automatically cloning the first
profile is unsafe because game membership is account-owned and the server
correctly prevents one active user from occupying two worlds.

The helper may connect to an already-running Darkspinner-owned server, but the
proxy should not make a DLL responsible for starting and supervising the full
server lifetime. If no broker/server is available, it should report a clear
launch failure or invoke a dedicated headless server command.

## Constraints

- A vanilla direct executable without the Darkspinner-owned startup proxy
  cannot load or consult Fang. Achieving that would require a shortcut/helper,
  an installed proxy, or an executable patch; executable patching is forbidden.
- The current workspace policy treats shipped game directories as immutable.
  Research and helper/proxy source changes are safe, but installing or updating
  `GameBin/VERSION.dll` requires the separately approved integration path
  and must not occur as an incidental PvP build step.
- Shift detection is an input convenience, not authentication. The broker
  remains authoritative for profile availability and token issuance.
- The JWT must be cleared immediately after Fang consumes it, matching the
  managed launcher path.

## Suggested implementation slices

1. Add a headless `darkrun session issue --account <profile>` operation that
   writes one length-delimited session envelope to an inherited handle.
2. Add Shift detection and bounded helper IPC to `app/fangproxy/proxy.c`, gated
   only for unmanaged direct launches.
3. Reuse Fang's existing environment contract and initializer; do not add a
   client gameplay hook.
4. Add direct-launch diagnostics under the server-owned logs directory and
   verify two distinct profiles reach two distinct Blaze sessions.

## Reusing Steam's EA Help choice

The second Steam choice is a separate publisher-authored launch option whose
target is `Support/EA Help/Electronic_Arts_Technical_Support.htm`. It does not
start `Game.exe`, so neither the startup proxy nor Fang is loaded. Steam
launch options can name different executables and arguments, which means the
choice could conceptually target `darkspinner.exe --second-session` or a small
headless session helper instead.

There are two materially different ways to expose that entrypoint:

- **Recommended:** install a separate Steam library shortcut named, for
  example, `Game - Second Client`, targeting the headless session
  entrypoint. This is user-owned, survives game verification, does not alter
  the shipped support files, and leaves the publisher's Help choice intact.
- **Exact menu replacement:** patch Steam's local cached app launch metadata so
  the second launch option names the Darkspinner entrypoint. This is technically
  possible but unsupported: Steam owns and refreshes that metadata, so the
  change can disappear after a client/app-info update. Any implementation must
  be explicitly opt-in, stop Steam before editing, make a recoverable backup,
  verify the exact Game app and original Help entry, and restore rather
  than overwrite an unknown configuration.

Replacing the `.htm` file, changing the system-wide HTML association, or
placing an executable under the support filename is not an acceptable route.
Those approaches either mutate shipped content or affect unrelated HTML files.
The existing `VERSION.dll` integration cannot intercept the Help selection
because that DLL is imported only by the game executable.
