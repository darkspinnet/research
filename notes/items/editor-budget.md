# Editor launch DNA-budget authority (build 103)

## Result

The object consumed by `sub_4B9FA0` / `sub_724E70` is the editor history
snapshot owned by the base editor application. It is a 44-byte, ref-counted
object with vtable `0x00FF9BAC`. The snapshot is normally created by
`sub_730340`, populated by `sub_71E0B0`, stored in the history vector at editor
object `+0x1A8`, and replayed by the undo/redo dispatchers through main editor
vtable slot 55.

The effective shipped value of `editorDefaultInitialBudget`
(`0xB1B76A19`) is **0**. The derived `Editors` app selects property-list
resource `0xF86BD1D6`; that exact decoded resource does not contain
`0xB1B76A19`. `sub_71F170` therefore uses its fallback DWORD at
`0x011BED1C`, which is loader-zeroed image storage. The first history snapshot
then reads category 0 back from the budget manager and records zero at snapshot
offset `+8`. Undo/redo later calls `sub_724E70`, which deliberately restores
that recorded zero to category 0. The launch path is preserving its snapshot;
it is not independently inventing a second budget policy.

This makes the normal fix a targeted content correction: add
`editorDefaultInitialBudget` with the intended nonzero value to the actual
default Editors property list (and to any proven alternative editor config
that replaces it). `sub_71D080` applies the property before `sub_7305B0`
creates the initial history snapshot, so the corrected value will be captured
at `+8` and replayed. No Fang behavior rewrite is required or appropriate.

## Darkspinner implementation status

The initial content-patch implementation was removed after live startup
testing proved that adding `0xB1B76A19 = 100` to the default Editors property
list made build 103 repeatedly request `api.creature.resetCreature` for the
same hero and prevented login. Persistently resetting that creature did not
change the client decision, proving the account record was not the cause.

A fresh vanilla Steam copy established the authoritative package as 1,080,117
bytes with SHA-256
`fa3b3682edd6eb11defa12f60cf61d0b68c135d8941233f5046facbb846d08e5`.
It matched the untouched install-side copy byte for byte. The rewritten
973,973-byte package had SHA-256
`59f59db3a8fa2a0d1d230ec60aba067f327227cfafd4e371cf9096ca57141f50`
and caused the reset loop. Replacing it with the vanilla package restored
normal JWT login and inventory loading without any reset request. The
Darkspinner integrity manifest now treats the vanilla hash as the sole source
of truth.

The recovered property reader remains useful evidence, but `100` is not a
safe authored value for this property in isolation. Darkspinner must preserve
the shipped Editors package until the relationship between the category-0
budget, base creature complexity, and the displayed `current/maximum` DNA
values is recovered.

## Ownership and vtables

- `sub_4BDB00` at `0x004BDB00` allocates `0x28E0` (10,464) bytes with tag
  `"SP_App"`, calls `sub_4BCFE0`, and registers the result as `"Editors"`.
- `sub_4BCFE0` at `0x004BCFE0` constructs the derived application after the
  base constructor `sub_725250`.
- Derived main vtable `0x00FD98D0`, slot 55, points to `sub_4B9FA0`
  (`0x004B9FA0`). Its entry is at `0x00FD99AC`.
- Base main vtable `0x00FFA1B8`, slot 55, points to `sub_724E70`
  (`0x00724E70`). Its entry is at `0x00FFA294`.
- Counting from the auxiliary/interface-table address `0x00FFA184` gives the
  previously observed base slot number 68. Both coordinates identify the same
  callback.
- Derived main slot 69 at `0x00FD99E4` points to `sub_4B9FE0`; base main slot
  69 at `0x00FFA2CC` points to `sub_71F170`. Counting from `0x00FFA184` gives
  base slot 82.
- The history-snapshot vtable is `0x00FF9BAC`: destructor
  `sub_720230`, retain `sub_880E50`, release `sub_C758D0`, and refcount query
  `sub_B16DE0`. No recovered RTTI name is reliable enough to assign a stronger
  class name, so this note calls it the editor history snapshot.

The global application manager is constructed by `sub_8641D0` with vtable
`0x0100E448`. Manager slot 8 (`sub_864470`) registers the `"Editors"` app.
Event handler `sub_863670` handles event `13878786` and calls manager slot 16
(`sub_863EB0`) with the application index from payload `+8`; `sub_863A80`
switches the active app through app vfuncs `+44` and `+40`. This layer selects
the application but does not construct the history snapshot or assign its
budget.

## Snapshot layout and input authority

Only offsets through `+21` are consumed by `sub_724E70`; the object has
additional server-event bookkeeping through `+43`.

| Offset | Size | Producer | Consumer/effect | Authority and confidence |
|---|---:|---|---|---|
| `+0` | 4 | constructor | vtable `0x00FF9BAC` | Class ownership, high |
| `+4` | 4 | constructor/refcount methods | lifetime | Refcount, high |
| `+8` | 4 | `sub_71E0B0`: budget-manager vfunc `+8`, category `0` | `sub_724E70`: budget-manager vfunc `+24`, category `0` | Packaged tuning becomes live editor state, then snapshot state; high |
| `+12` | 4 | `sub_71E0B0`: editor object `+856` (`this[214]`) | selects mode 0, 1, 2, or derived mode 3 UI event | Runtime editor-mode/UI selection; high for mechanics, medium for the exact authored UI control |
| `+16` | 4 | `sub_71E0B0`: current editor data `this[183][80]`, or 0 when absent/disabled | `sub_724E70` calls `sub_6F7FC0(this[183], field)` when nonzero | Current selected editor/model resource handle; high for mechanics, medium for semantic name |
| `+20` | 1 | editor object byte `+1234` | restored to byte `+1234` | Runtime editor dirty/history state; high |
| `+21` | 1 | editor object byte `+1235` | restored to byte `+1235` | Runtime editor history/UI state; high |
| `+22..+23` | 2 | unused padding | not read by launch consumer | High |
| `+24..+32` | 12 | normally zero; custom `sub_737290` branch copies a three-DWORD server/model event tuple | not read by `sub_724E70` | Server/model event bookkeeping, high |
| `+36` | 4 | normally zero; custom branch writes `-1` | not read by `sub_724E70` | Server/model event bookkeeping, high |
| `+40` | 4 | normally zero; custom branch retains the source event pointer | not read by `sub_724E70` | Server/model event ownership, high |

`sub_4B9FA0` adds only derived mode-3 handling: if snapshot `+12 == 3`, it
posts UI event `178287184`, then calls `sub_724E70`. The base consumer:

- restores snapshot `+8` to budget category 0;
- maps mode 0, 1, and 2 to UI events `-266747161`, `-266747149`, and
  `1881245250`;
- applies nonzero `+16` through `sub_6F7FC0`;
- restores bytes `+20` and `+21`; and
- publishes `SP::MessageBasicRC<5>` event `84476980`.

The account balance is separate. `sub_4B9FE0` calls `sub_71F170`, clears
categories 10 and 9, and loads category 8 from
`sub_4E4E20() + 10808`. No category-0 snapshot field reads that account
location. Existing-part costs later debit category 0 through `sub_724AC0` /
`sub_6A22E0`; that is consumption, not initial authority.

No packaged UI resource directly supplies snapshot `+8`. UI actions choose
the editor mode/model and cause history capture, but the budget is read from
the already initialized budget manager. The custom server/model event path
does not overwrite `+8`; it calls `sub_71E0B0` first and adds only the
`+24..+40` extension. Thus neither ordinary UI dispatch nor that server event
is an independent category-0 budget source.

## Every constructor and dispatcher

### Constructors and canonical producer

- `sub_71C570` at `0x0071C570` is the standalone 44-byte snapshot constructor.
  It installs vtable `0x00FF9BAC` and zeros `+4..+43`.
- `sub_730340` at `0x00730340` is the canonical history-snapshot producer.
  When its second stack argument is null, `0x007303AC..0x007303E9` allocates
  44 bytes and performs the same initialization inline.
- `sub_71E0B0` at `0x0071E0B0` populates the consumed prefix. In particular,
  `0x0071E0B9..0x0071E0CB` calls the budget-manager getter for category 0 and
  writes its result to snapshot `+8`.
- `sub_730340` inserts the result into the vector at editor `+0x1A8` and
  increments the history index at `+0x1C8`.
- `sub_737290` has the only recovered preconstructed-snapshot path:
  `0x00737652` allocates 44 bytes, `0x00737671` calls `sub_71C570`,
  `0x0073768B` calls `sub_71E0B0`, `0x00737693..0x007376B5` fills the
  server/model extension, and `0x007376B8` submits it to `sub_730340`.

All direct calls to `sub_730340` in build 103 are:

`0x004BC9B7`, `0x004BFAE0`, `0x007305D9`, `0x0073292A`,
`0x00732F15`, `0x00733125`, `0x007350FA`, `0x00735257`,
`0x00735A8D`, `0x007373AD`, `0x007376B8`, and `0x0073783A`.

At every site except `0x007376B8`, the second stack argument is null, so
`sub_730340` performs the inline construction and live-state capture. The
`0x007376B8` site passes the preconstructed object described above. This
exhausts direct calls in the executable, not merely decompiler text matches.

### Indirect launch dispatchers

There are no ordinary absolute calls to `sub_4B9FA0` because callers invoke
main vtable offset `+0xDC` (slot 55).

- Redo/forward dispatcher `sub_7293A0` (`0x007293A0`) increments the history
  index, loads the stored pointer from editor `+0x1A8`
  (`0x007293C3..0x007293D7`), and calls slot 55 at
  `0x00729500..0x00729515`.
- Undo/back dispatcher `sub_72FF90` (`0x0072FF90`) decrements the history
  index, loads the stored pointer from editor `+0x1A8`
  (`0x0072FFD6..0x0072FFEC`), and calls slot 55 at
  `0x0073011B..0x00730130`.
- The derived wrappers `sub_4BD740` and `sub_4BD540` adjust derived editor
  side effects and then enter `sub_7293A0` / `sub_72FF90`. Their direct calls
  are `0x004BD92E` and `0x004BD727`.

The decompiler loses the slot-55 argument in both base dispatchers because of
damaged stack analysis. Machine code proves that the argument is the retained
history entry, not a hidden caller-supplied launch descriptor.

## Shipped tuning resource and zero overwrite

The derived constructor installs default editor property-list instance
`0xF86BD1D6` at editor object `+460`. `sub_71C270` returns that value.
`sub_731900` uses it unless the attached editor configuration at object
`+524` supplies a replacement instance at its `+12`.

The exact shipped default resource is:

- package: `bin/game/Data/Editors.package`
- ordinal: `698`
- type: `0x00B1B104`
- group: `0x40600100`
- instance: `0x00000000F86BD1D6`
- stable identity:
  `resource/000698_00b1b104_40600100_00000000f86bd1d6.bin`
- stored size: 178 bytes
- decoded size: 192 bytes
- compression: RefPack

It was resolved through the proven constructor/resource-manager chain and
decoded to `bin/game/logs/editor-budget/editors-f86bd1d6.bin`. This is not a
repeat of the earlier blind package scan. The decoded property list lacks
`0xB1B76A19`.

`sub_71F170` at `0x0071F170` queries property `0xB1B76A19` through the loaded
property list at editor `+40`, accepts property types 9 and 16, and otherwise
uses DWORD `0x011BED1C`. That address lies beyond the initialized `.data` raw
extent and is loader-zeroed, so the fallback is 0. It then sets budget category
0 and continues through `sub_71EF20`.

`sub_71D080` calls main slot 69 at `0x0071D0C9..0x0071D0D1`, applying this
default. Later startup reaches `sub_7305B0`; its call at `0x007305D9` creates
the first history entry, and `sub_71E0B0` captures the current category-0
value. Because the property is absent, that captured value is zero.
`sub_724E70` subsequently overwrites category 0 with zero on undo/redo because
restoring the captured budget is exactly the snapshot contract.

## Implementation boundary

The supported correction is content/server-content policy, not Fang behavior:

1. Choose the intended editor initial DNA budget as server/content policy.
2. Author `editorDefaultInitialBudget` (`0xB1B76A19`) as an accepted integer
   property type in resource `0xF86BD1D6`.
3. If an attached editor configuration replaces `0xF86BD1D6`, update only
   replacement resources proven to be used by supported flows.
4. Leave snapshot capture and replay intact. They correctly preserve a
   nonzero category-0 value once tuning initializes it.
5. Do not source category 0 from account category 8, and do not patch Fang to
   ignore or rewrite snapshot `+8`.

A server-side launch adapter is only needed if darkspin deliberately supports
an editor flow whose property list cannot carry the policy. In that case it
must initialize category 0 before the first `sub_730340` capture (or supply an
equivalent snapshot value), while preserving the same authority rule. The
build-103 custom server/model event extension is not such an adapter: it does
not modify `+8`.

## Focused tests

1. **Content identity:** inspect ordinal 698 and assert type/group/instance,
   decoded size, and the presence of `0xB1B76A19` with the intended integer
   type/value after the content change.
2. **Default application:** start Editors with no attached replacement config;
   immediately after `sub_71F170`, assert budget-manager category 0 equals the
   authored value.
3. **Initial capture:** break after `sub_71E0B0` from the `0x007305D9` startup
   call and assert snapshot `+8` equals category 0.
4. **Undo/redo replay:** spend DNA, create history entries, undo, and redo;
   at `0x00729515` and `0x00730130`, assert the dispatched entry is from
   editor `+0x1A8`, and after `sub_724E70` category 0 equals that entry's
   `+8`, not zero unless the snapshot really contains zero.
5. **Modes:** repeat for snapshot modes 0, 1, 2, and derived mode 3; verify the
   corresponding UI event and unchanged category-0 restoration.
6. **Optional resource:** test `+16 == 0` and nonzero; only the latter may call
   `sub_6F7FC0`, with no change to budget authority.
7. **Flags:** toggle editor state bytes `+1234/+1235`, capture, replay, and
   verify exact restoration without affecting `+8`.
8. **Server/model extension:** exercise the `sub_737290` branch at
   `0x00737652`; assert `+24..+40` reflect its event payload while `+8` still
   equals the live category-0 getter result.
9. **Account separation:** vary account value at `sub_4E4E20()+10808`; assert
   category 8 changes and category 0/snapshot `+8` do not.
10. **Cost debit:** add/remove a priced editor part and verify
    `sub_724AC0` debits/refunds category 0 relative to the authored initial
    budget.

## Confidence and remaining evidence

- **High confidence:** owning app and vtables; snapshot size/vtable/layout;
  complete direct producer call list; both slot-55 dispatchers; category-0
  getter/setter chain; exact default packaged resource; absent property;
  zero-valued fallback; account-category separation; content-fix ordering.
- **Medium confidence:** human-readable semantic names for `+16`, `+20`, and
  `+21`, because build 103 exposes their effects but not surviving type names.
- **Not claimed:** an exact packaged UI layout/button resource for mode
  selection. The binary proves UI-controlled runtime state feeds `+12`, but
  there is no proven resource indirection from that field to a stable package
  identity. Inventing one or repeating a blind package scan would not improve
  the budget conclusion.
