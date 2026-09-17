# Game 5.3.0.103: progression and campaign

## Scope

These static-analysis findings apply to:

```text
bin/game/GameBin/Game.exe
SHA-256 3C7248A4C6614450290A66C22AB7FCDA86052B29A7B555E834D7C23F9B73EE5B
```

“Confirmed” here means directly observable in this executable's strings and
x86 control flow. Semantic names are descriptions of behavior, not recovered
debug symbols. Supporting batch-analysis output is under
`bin/game/logs/ida-progress-*.log`.

## Campaign resource

- `ChainLevels.ChainLevels` is stored at `0x010252A4`.
- The resource-registration/data reference is at `0x00F72E60`.
- The adjacent extracted asset
  `bin/server/data/chainlevels/ChainLevels.ChainLevels.xml` contains 72
  ordered entries.
- `bin/server/data/difficultytuning/DifficultyTuning.DifficultyTuning.xml`
  independently contains 72 health, 72 damage, and 72 expected-avatar-level
  values.

The 72-entry agreement confirms that the chain is a progression axis, not
merely a list of the 24 unique story maps.

## Threat label calculation

The client and the structurally matching reference packet model use a
one-based non-tutorial level index:

```text
minor = ((levelIndex - 1) % 4) + 1
major = ((levelIndex - 1) / 4) + 1
```

Combined with the 72-entry data arrays, this yields labels 1-1 through 18-4.

## Planet screen and permanent map gating

The planet UI uses `Planet_Room.swf`, `SP_UI/cPlanetScreen`,
`InitForFirstPlanet`, and `InitForNewPlanet`. The main initialization function
is `sub_527DD0` (`0x00527DD0` through approximately `0x0052A2A7`). Its object
fields include:

| Offset | Observed use |
| --- | --- |
| `+0x30` | level resource/noun |
| `+0x34` | level index |
| `+0x38` | star level |
| `+0x3C` | time remaining |
| `+0x40` | progression within the active chain run |

Near `0x005296F4`, the normal path:

1. obtains global account/UI state;
2. bypasses the comparison when an unlock-all/test flag at `+0x2AA8` is set;
3. reads the permanent progression scalar at `+0x2AB0`;
4. adds one;
5. compares candidate level index `[planetScreen+0x34]` against that value;
6. derives the locked presentation state when the candidate is greater.

The account parser also contains the `chain_progression` descriptor. Together
these instructions confirm the compatibility rule:

```text
candidateLevelIndex <= chainProgression + 1
```

Thus an account at progression 0 can select index 1, and completing index N is
expected to expose index N+1.

## New-player HTTP state

`sub_4A8DB0` builds a game request containing `new_player_progress` and,
optionally, `new_player_inventory`. `sub_4B6E60` submits it.

`sub_5187E0` stores new-player progress in global account state at `+0x2A6C`
and submits the update. Its callers expose this sequence:

| Transition | Function / observed association |
| --- | --- |
| 0 -> 1000 | `sub_52D230`, initial spaceship/room state |
| 1000 -> 2000 | `sub_516F50` |
| 3000 -> 4000 | `sub_41A570` |
| 4000 -> 5000 | `sub_419510` |
| 5000 -> 6000 | `sub_4C08A0`, launch of `SP_Editor` |
| 6000 -> 6500 | `sub_52D230`, `NewPlayerUnlockedMapRoom` |
| 6500 -> 6800 | `sub_527810`, planet/room initialization |
| 6800 -> 8000 | `sub_527810`, planet/room initialization |
| 8000 -> 9000 | planet-screen/progress paths, including a branch in `sub_527DD0` |

The same room controller references `NewPlayerArrivedCollectionRoom` and
`NewPlayerUnlockedEditorRoom`, demonstrating that the numeric field controls a
staged collection/editor/map onboarding flow.

`sub_518860` manages `new_player_inventory`, sets it to 1 after a
creature/collection condition, and submits it with a zero progress argument.
Its exact player-facing meaning is not yet confirmed.

## Hero unlock request

- `sub_4A90B0` constructs `api.creature.unlockCreature`.
- `sub_4B6F70` is its submit wrapper.
- `sub_40E8B0` resolves the selected creature-template noun and triggers the
  request from the UI.
- `sub_407540` fills cashout UI data including `numCreatureUnlocks`, images,
  classes, genetic types, and text.

This confirms that hero acquisition includes a server-authorized selection
request and that cashout UI can present one or more creature unlock choices.
It does not establish the starter hero count or level milestone schedule.

## Unconfirmed by this pass

- clean-account DNA, XP, hero count, and part count;
- the identity and timing of tutorial hero grants;
- XP and DNA reward formulas;
- which server event advances `chain_progression`;
- exact semantics of `new_player_inventory`;
- whether band 18's zero expected-avatar-level entries are sentinel values;
- the five-planet active-run cap, which currently comes from the structurally
  matching legacy packet implementation rather than a decisive build-103
  instruction sequence.
