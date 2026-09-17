# Campaign action-bar artwork

## Result

The black campaign artwork is not an ability-availability, attention-flag, or
cooldown-overlay failure. Build 103 selects the right character ability
descriptors and exposes the right slots, but the icon resource passed by the
descriptor-to-Scaleform path is the remaining bad input. Existing evidence
localizes the defect to that resource handoff; it does not record the numeric
descriptor `icon` field or the final Scaleform string, so it cannot yet say
whether that identity is zero, merely wrong, or valid but absent from the
loaded movie. The smallest conclusive next step is the focused trace described
below. A packet or Fang substitution should not be guessed before that trace.

The strongest classification evidence is the 2026-07-22 11:30 run. On a fresh
campaign HUD, Blitz produced indexes `2` and `3`; Sage produced index `3`; and
the third selected hero produced indexes `2` and `3`. The third hero was later
verified as Goliath Alpha, and the visible Energy Sentinel basic, Shockwave,
and Zetawatt Beam labels agree with Goliath's content definition. Those labels
come from the same non-null descriptor used for the icon. A null descriptor
would make `sub_421600` return without calling `SetAbilityData`; a descriptor
for the wrong ability would not produce this noun/index/label agreement.
Therefore the observed state is the third alternative: the correct descriptor
reaches the Scaleform population path, but its icon resource identity does not
produce artwork.

The 12:18 playtest adds no contrary icon evidence. It confirms ordinary ability
cooldown processing while the campaign HUD remains in the same setup family.
Its later missing squad-portrait cooldown was caused by the unrelated field-4
deadline clock domain.

## Native path and exact `SetAbilityData` inputs

`sub_422820` is the action-bar controller update. It resolves the local
`LabsPlayer`, reads the selected deck/character index from player dword `+12`,
and obtains that character record with `sub_9C23D0(player, player[3])`. It
repopulates ability data when either the selected index or the selected
character identity at record `+92` changes. On that transition it calls
`sub_421600` for action indexes `0`, `2`, `3`, `6`, `7`, and `8`.

For every one of those indexes, `sub_421600(player, selectedCharacter, index)`
performs this exact chain:

1. `sub_9C25C0(player, index, -1)` returns the ability GUID. The `-1` selects
   `player[3]` for ordinary indexes.
2. `sub_9DAC00(guid)` looks up the registered ability descriptor. A null result
   terminates the function and no `SetAbilityData` call occurs.
3. Descriptor dword `+120` is combined with type `796721156` (`0x2F7D0004`)
   and group `161348501` (`0x099DFB95`), then `sub_7AE040` converts that resource
   identity into the first Scaleform argument. This is the icon input.
4. Descriptor dword `+100` is the second Scaleform argument. This is the
   localized ability-label identity.
5. The third argument is the numeric action index.
6. `sub_421500(index)` supplies the fourth argument, the localized key/slot
   label. Indexes `2`, `3`, `6`, `7`, and `8` map to locale resources
   `0x10000011` through `0x10000015`; index `0` has an empty key label.
7. The client calls Scaleform `SetAbilityData(icon, abilityLabel, index,
   keyLabel)`.

`sub_9C25C0` chooses the GUID as follows:

| Action index | Character definition input | Selected character |
|---:|---|---|
| `0` | definition dword `21` | active deck record |
| `2` | definition dword `25` | active deck record |
| `3` | definition dword `29` | active deck record |
| `6` | definition dword `33` | fixed deck record `0` |
| `7` | definition dword `33` | fixed deck record `1` |
| `8` | definition dword `33` | fixed deck record `2` |

Thus `6`/`7`/`8` are deck-position support abilities, not three support
abilities belonging to the deployed hero. With top-level ability boundary
field `21` equal to `5`, `sub_9C2A60` exposes indexes below `5` plus only the
support index matching active deck position: `6` for active index `0`, `7` for
`1`, and `8` for `2`.

The later visibility and enabled-state calls are separate. `sub_422820` calls
`sub_41DF90` with the `sub_9C2A60` result and `sub_41E0A0` with gameplay
admission state. Those calls do not supply or replace the icon.

## Tutorial versus campaign player inputs

The initial `LabsPlayerUpdate` contains both three fixed `0x620`-byte character
records and three reflected character records. Both forms carry the four
identity inputs ultimately consumed by the selected-character lookup: noun,
appearance asset, version, and type. Top-level field `4` supplies the initial
active-character index; later `PlayerCharacterDeploy` changes the selected
character.

The tutorial snapshot that renders Blitz correctly uses:

| Record | Active index | Noun | Appearance asset | Version | Type |
|---|---:|---:|---:|---:|---:|
| Blitz | `0` for the local slot | `0x6367B6CD` | `0x00000000A93AFF21` | `1` | `2` |
| Sage | not active | `0x2CA50A9A` | `0x0000000055D1408F` | `1` | `2` |
| backing third record | not active | Blitz noun/asset/version | same as Blitz | `1` | `6` |

After the tutorial Sage reveal, the first two identities remain the same;
field `21` becomes `3` and the visible creature boundary becomes `2`. Those
boundary values affect visibility, not the icon resource selected from the
ability descriptor.

The 11:30/12:18 normal campaign snapshot for Test's selected squad uses the
following values in both its fixed and reflected records:

| Record | Initial active index | Noun | Appearance asset | Version | Type |
|---|---:|---:|---:|---:|---:|
| Blitz Alpha, creature `1` | `0` | `0x6367B6CD` | `0x0000000000000001` | `3` | `2` |
| Sage Alpha, creature `2` | not active | `0x2CA50A9A` | `0x0000000000000002` | `1` | `2` |
| Goliath Alpha, creature `4` | not active | `0xC940B9DF` | `0x0000000000000004` | `1` | `2` |

The campaign encoder obtains those appearance values from
`campaignAppearanceAsset`: persistent creatures use their database creature
IDs, while only synthetic/tutorial bindings use the hard-coded Blitz or Sage
assets. It writes top-level field `4` from `m.Slot`; for the local Test player
that is `0`, so it coincides with the required initial active-character index.
It writes field `21 = 5` and all three character types as `2`.

The current Fang trace at the campaign entry and every observed switch shows
all three selected creature asset lookups and their nested lookups resolving.
Consequently the persistent appearance identities `1`, `2`, and `4` are not
failing at the outer selected-creature lookup. The different Blitz version is
also not a sufficient explanation: Sage and Goliath use version `1` yet show
the same black-artwork failure. Likewise, type `2` is shared with the working
tutorial Blitz and Sage records. The packet comparison leaves the persistent
appearance identity/path as a useful trace discriminator, but does not justify
replacing those IDs when their lookups demonstrably succeed.

## Why the other HUD symptoms are independent

- `PlayAbilityBlink` changes per-index attention flags after availability is
  evaluated. Fang no longer gates or mutates this presentation; debug builds
  may observe the native availability result and return it unchanged. The
  shared movement route now reserves the ability-lesson marker only when the
  gameplay binding is `ModeTutorial`, rather than treating every campaign
  dungeon as eligible. It never supplies descriptor `+120` or calls
  `SetAbilityData`.
- `SetAbilityCooldownPercent` receives index, percent, remaining time, and an
  active flag from `sub_4222F0`. It is called after ability data population and
  cannot turn a missing icon resource into a valid one.
- Squad portrait cooldowns consume nested character-reflection field `4` and a
  gameplay-clock deadline. They belong to a different Scaleform controller and
  are unrelated to action-bar artwork.
- Field `21` controls `SetAbilityVisibility`; changing it from overflow sentinel
  `9` to normal boundary `5` correctly changes which actions are available but
  cannot repair the icon argument already constructed by `sub_421600`.

## Smallest conclusive next trace

Instrument only the `SetAbilityData` call site inside native function
`sub_421600` (`0x00421600`) in `app/fang/fang.c`. Emit one structured record per
call with:

- active deck index `player[3]`, selected-character pointer/identity, and action
  index;
- `sub_9C25C0` result GUID;
- `sub_9DAC00` result pointer and an explicit `is_descriptor_resolved` bit;
- descriptor dwords `+100` (label) and `+120` (icon);
- the fully formatted icon string produced by `sub_7AE040`;
- the four Scaleform argument types and values immediately before
  `sub_54D240`.

Capture this once in the working tutorial when Blitz index `2` appears and once
at normal campaign entry for indexes `0`, `2`, `3`, and `6`; then switch to
Sage and Goliath to capture `7` and `8`. The comparison has an unambiguous
decision table:

| Trace result | Fix |
|---|---|
| GUID sentinel or descriptor null | Correct the corresponding character record's noun/appearance/version/type or initial active index in `LabsPlayerUpdate`; do not touch Scaleform. |
| Descriptor resolves but GUID/label names the wrong ability | Correct the character-definition selection input, most likely the persistent appearance/version projection. |
| Correct GUID and label, but `+120` is zero/wrong | Supply the proven icon resource identity in Fang for that GUID, or correct the packet field that makes the client choose the wrong descriptor variant. |
| Correct nonzero `+120`, but formatted identity differs from tutorial or cannot load | Fix the resource type/group/instance formatting or package-load identity at the Fang boundary. |
| All four arguments match a working tutorial call | Hook the `SetAbilityData` receiver/movie loader next; the packet and native descriptor path are exonerated. |

Given the current noun/index/label agreement, the expected result is the third
or fourth row. That focused hook is preferable to a broad descriptor-lookup
trace and is the minimum evidence needed for an implementation-ready Fang
remap. If the trace instead identifies a packet-selected variant, the packet
fix should preserve the currently resolving persistent appearance asset and
change only the proven mismatching identity field.
