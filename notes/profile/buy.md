# Build 103 Helix `BUY GAME` tab

## Conclusion

Build 103 has three distinct access/upsell signals, and only one controls the
Helix tab:

- `grant_online_access` in the raw `api.account.getAccount` XML controls the
  Helix Store/`BUY GAME` tab.
- `grant_all_access`, parsed into the native account byte at `account+144`, is
  a gameplay-cap bypass. It does not control Helix navigation or the tab.
- `upsell`, parsed into the native account dword at `account+136`, controls a
  quit-menu upsell redirect. It does not control the Helix tab.

For an owned account, the shipped Helix code disables the Store click, hides
the `BUY GAME` label, and redirects an attempted Store route to Profile. It
does **not** remove the `Tab_4` DOM element or close the tab-bar gap. Thus the
most exact answer is "hide the label and disable/redirect the route," not
"remove the tab node," and not "leave it enabled while merely unlocking other
features." Separately, `grant_all_access=1` merely unlocks capped gameplay.

The current server already returns the correct owned-account combination:
`grant_all_access=1`, `grant_online_access=1`, and `upsell=0`. The remaining
way for the literal `BUY GAME` label to survive is a packaged-web timing bug:
the root page fetches the account asynchronously, but its callback does not
notify an already-loaded `helix.html` to apply `hidestore()`.

## Native account parser

The canonical sources are `bin/game/GameBin/Game.c` and
`bin/game/GameBin/Game.idb`. `sub_4AB910` begins at `0x004AB910` and
parses the `<account>` node used by the native client.

| XML field | Account layout | Native evidence | Meaning |
| --- | ---: | --- | --- |
| `upsell` | `+136` / `+0x88`, dword | parse result written at `0x004AC305` (`mov [esi+88h], eax`) | Integer flag; the only confirmed semantic consumer tests it strictly against `1`. |
| `xp` | `+140` / `+0x8c`, dword | lies between the two fields in `sub_4AB910` | Confirms the layout is not a mistaken byte interpretation of `upsell`. |
| `grant_all_access` | `+144` / `+0x90`, byte | decimal parse at `0x004AC3B3`; `cmp eax,1` at `0x004AC3BD`; `sete dl` at `0x004AC3C1`; byte store at `0x004AC3C9` | Strict normalization: the byte is `1` only when the parsed decimal is exactly `1`; every other value becomes `0`. |
| `cap_level` | `+148` / `+0x94`, dword | parsed immediately after the access byte | Gameplay cap data. |
| `cap_progression` | `+152` / `+0x98`, dword | parsed immediately after `cap_level` | Gameplay cap data used with the access byte. |

The parser is called for account-bearing replies from `sub_4AE250`
(`0x004AE250`), `sub_4AFCF0` (`0x004AFCF0`), and `sub_4B0370`
(`0x004B0370`). On session acceptance, `sub_512E50` (`0x00512E50`)
copies the account into the main state block rooted at `sub_4E4E20()+10776`.
The exact copies are:

- `upsell`: source and destination `+0x88` at `0x00512F26` and
  `0x00512F2C`;
- `grant_all_access`: source and destination bytes `+0x90` at
  `0x00512F3E` and `0x00512F44`.

Consequently the main-state addresses are base `+10912` (`+0x2aa0`) for
`upsell` and base `+10920` (`+0x2aa8`) for `grant_all_access`.

## Downstream native uses

### `upsell` (`account+136`)

The only semantic read found in the canonical decompile is in
`sub_441CD0` (`0x00441CD0`), the options/quit command handler. At
`0x00441DCD` it tests:

```text
main_state.upsell == 1
```

If true, it calls `sub_4E50C0` (`0x004E50C0`), which sets a state byte and
calls `sub_429620(..., 5, 0, 0)`. Navigation selector `5` executes the web
command `showStore()`; its string is selected at `0x004297D4`. If `upsell` is
not `1`, the quit handler posts event `197388488` instead. Therefore
`upsell=1` actively routes quitting through the store/upsell UI. It is not an
"owned" value and setting it to `1` would make this problem worse.

Apart from parsing and the account copy, no other semantic consumer of
`account+136` was found. Same-number offsets in unrelated structures were not
counted.

### `grant_all_access` (`account+144`)

Confirmed uses are:

1. `sub_513660` (`0x00513660`) reads the accepted account byte at
   `0x00513733` and passes it to `sub_53F540` at `0x0051373B`.
   `sub_53F540` (`0x0053F540`) formats the byte as decimal text into global
   `unk_143B510`. This is publication/diagnostic state, not Helix tab logic.
2. `sub_527DD0` (`0x00527DD0`) reads the main-state byte at `0x005296F9`.
   The predicate represented by the decompile is:

   ```text
   is_cap_locked = !grant_all_access && item_or_progression_index > cap_progression + 1
   ```

   A true access byte clears the local cap-lock flag unconditionally. This is
   evidence that the field bypasses trial/progression caps; it does not hide
   or navigate Helix.

The apparent `this+144` reads elsewhere in the decompile are mostly unrelated
types. The main-state access above and the accepted-account path are the ones
tied back to the parsed account object.

## Native Helix/profile navigation

`sub_429620` (`0x00429620`) maps native navigation selectors into JavaScript
commands in the embedded web view:

| Selector | JavaScript command | Relevant behavior |
| ---: | --- | --- |
| `0` | `showFriendsList()` | Root friends page. |
| `5` | `showStore()` | Root forwards to `helix.html` with screen `store`. |
| `6` | `showPlayerProfile(%llu)` | The wide format at `0x00FCFCD4`, selected at `0x004297FE`, forwards an account/profile ID to the root page. ID `0` selects `myprofile`; another ID selects `friendsprofile`. |
| `10` | `showLeaderBoards()` | Root forwards to the Helix leaderboards tab. |
| `11` | `showWiki()` | Root forwards to the Helix manual tab. |

The native selector contains no ownership predicate. Ownership enforcement is
deliberately left to the embedded web resources. This also explains why
changing only `grant_all_access` cannot affect the Helix tab.

## Packaged UI and localization resources

DBPF indexes do not preserve authored filesystem paths, so the identities
below are the stable package tuples and synthetic names reported by
`darkrun inspect`. The authored page roles are established by their decoded
contents and iframe links.

### Root web page

- Package: `bin/game/Data/Web.package`
- Ordinal/resource: `13`,
  `000013_dd6233d6_00000000_0000000050fc8df3.bin`
- Tuple: type `0xDD6233D6`, group `0x00000000`, instance
  `0x0000000050FC8DF3`

This page owns `currentonlineaccesslvl`, calls
`api.account.getAccount`, and assigns only the XML element
`grant_online_access` to that variable. It exposes it through
`getcurrentplayeraccesslvl()`. It also defines `showStore()`, which calls
`FRAME3` (`helix.html`) as `showScreen('store')`.

Critically, neither `grant_all_access` nor `upsell` is read here. The account
request is asynchronous and its callback only assigns
`currentonlineaccesslvl`; it does not call `hidestore()` or otherwise refresh
the child page.

### Helix page that creates the tab

- Package: `bin/game/Data/Web.package`
- Ordinal/resource: `161`,
  `000161_dd6233d6_00000000_00000000bbfa789f.bin`
- Tuple: type `0xDD6233D6`, group `0x00000000`, instance
  `0x00000000BBFA789F`
- Authored role: `helix.html`, proven by the root page's
  `src="helix.html"` and this resource's Helix markup.

This resource unconditionally creates:

```html
<div id="Tab_4" ...><span id="Tab_Four_Label"></span></div>
<iframe id="Tab_4_Frame" ... src="fullgame.html" ...></iframe>
```

It localizes `Tab_Four_Label` from key `0x0b56894b` (named `Store` in the
JavaScript map). Its access predicate is loose JavaScript equality against
the string returned by the XML callback:

```text
accesslvl == 1
```

`showScreen()` calls `hidestore()` whenever that predicate is true. For an
explicit `store` request it allows `Tab_4` only when `accesslvl != 1`; owned
accounts instead call `hidestore()` and select `Tab_1` (Profile).

`hidestore()` performs exactly three operations:

1. replaces `Tab_4.onclick` with an empty function;
2. colors `Tab_Four_Label` gray;
3. sets `Tab_Four_Label.style.visibility = 'hidden'`.

It does not hide/remove `Tab_4` or `Tab_4_Frame`. The page's `window.onload`
reads `accesslvl` into a local variable but never acts on it. If the account
callback has not completed before the first `showScreen()`, the value is the
initial empty string, so the label remains visible; the later callback does
not repair it.

### Full-game upsell page

- Package: `bin/game/Data/Web.package`
- Ordinal/resource: `87`,
  `000087_dd6233d6_00000000_000000002fdbba88.bin`
- Tuple: type `0xDD6233D6`, group `0x00000000`, instance
  `0x000000002FDBBA88`
- Authored role: `fullgame.html`, proven by `helix.html`'s iframe source and
  this resource's buy/redeem implementation.

It creates the actual buy/redeem content. The first click opens Steam app
`99890` or the old EA Store URL; the second click logs out to redeem a key.
This page contains no ownership test of its own.

### English label

- Package: `bin/game/Data/Locale/en-us/Text.package`
- Ordinal/resource: `45`,
  `000045_02fac0b6_02fabf01_00000000c6321815.bin`
- Tuple: type `0x02FAC0B6`, group `0x02FABF01`, instance
  `0x00000000C6321815`
- Locale group used by the web pages: `0xc6321815`
- Key: `0x0b56894b`
- Text: `BUY GAME`

The same locale resource maps `0x0b568948` to `PROFILE`,
`0x0b568949` to `FRIENDS`, and `0x0b56894a` to `STATS`, confirming that this
is the Helix tab-label group.

## Current server response

`server/game/api.go` serializes all three signals independently:

```xml
<upsell>...</upsell>
<grant_all_access>0-or-1</grant_all_access>
<grant_online_access>0-or-1</grant_online_access>
```

The owned defaults and normalization in `server/sporenet/account.go` force
both access booleans true. The SQLite schema defaults
`is_all_access_granted` and `is_online_access_granted` to `1`, while `upsell`
defaults to `0`. Read-only `darkrun db user get` inspection through the
current `bin/darkspinner/darkspin.toml` configuration (whose user table is in
`bin/darkspinner/darkspin/saves/darkspin.db`) showed profiles 1 and 2 with
exactly those values: both access columns `1`, `upsell=0`.
Therefore their current serialized response is:

```xml
<upsell>0</upsell>
<grant_all_access>1</grant_all_access>
<grant_online_access>1</grant_online_access>
```

One coverage gap is worth recording: current API tests do not assert these
three elements, so a serializer regression could pass unnoticed.

## Minimal fix, separated by layer

### Server-response fix

For any server that currently marks ownership only with
`grant_all_access=1`, the minimal response fix is to also return
`grant_online_access=1` from `api.account.getAccount` and keep `upsell=0`.
Do not overload `grant_all_access`, and do not set `upsell=1`.

For the current darkspin server, no production response change is indicated:
it already emits the correct values. The only justified server-side change
would be a regression test asserting the three elements for a normalized
owned account. That test would protect the contract but would not cure the
packaged page's asynchronous race.

### Packaged web/client compatibility fix

The smallest behavior fix is in the root packaged page's account callback:
immediately after assigning `currentonlineaccesslvl`, notify the loaded Helix
frame to apply owned state (or call a child refresh function which does so).
Conceptually:

```javascript
currentonlineaccesslvl = account.grant_online_access;
if (currentonlineaccesslvl == 1 && window.frames["FRAME3"])
    window.frames["FRAME3"].hidestore();
```

The child should also make its `window.onload` path call `hidestore()` when
the parent value is already `1`. Together those two idempotent checks cover
both load orders. If the desired presentation is no blank tab-bar slot, use
`Tab_4.style.display = 'none'` (and reflow neighboring tabs) rather than the
retail-compatible label-only hiding; that is a UI policy change, not what the
shipped code currently specifies.

### Fang/native compatibility patch

A Fang patch is a fallback for deployments that cannot replace the packaged
web resource. It should intercept native selector `5` in `sub_429620` (the
`showStore()` selection at `0x004297D4`) and redirect it to Profile for a
known owned account, or inject the same child-page notification after account
acceptance. Such a patch must use `grant_online_access` semantics or an
explicit local-server "all users owned" policy; testing only
`account+144/grant_all_access` would conflate two distinct retail fields.

The web-resource fix is smaller and matches the actual owner of the
visibility predicate. No Fang/client binary patch is required to correct the
current server response itself.

## Implemented compatibility repair

Fang now recognizes the exact `accountinfocallback` response and queues a
deferred root-page script only when that response contains
`grant_online_access=1`. The script retries for up to five seconds so either
iframe load order reaches `FRAME3.hidestore()`, preserving the shipped owned
account presentation without editing `Web.package` or conflating
`grant_all_access`. The account-response test also locks `upsell=0` and both
access fields at `1`.

A rebuilt Darkspinner/build-103 client live check confirmed the callback path:
the trace recorded `owned_account_refresh_queued=1` followed by successful
script delivery, and Helix rendered only `PROFILE`, `STATS`, and `HELP`. The
empty fourth tab container remains, matching the recovered retail-owned
presentation, while the `BUY GAME` label and Store route are unavailable.
