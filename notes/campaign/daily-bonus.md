# Campaign Daily Bonus

Build 103 treats the Daily Bonus as a cash-out rarity modifier with a rolling
account cooldown. It is not the separately authored Daily objective and no
recovered client path applies it to ordinary equipment, Catalyst, DNA, or
capsule drops.

## Recovered client contract

The account parser at `Game.c:270774-270784` reads
`cashout_bonus_time` as a 32-bit account scalar. The profile response in
`bin/game/logs/web-118.html:659-679` forwards that field to
`updatedcashouttimer` only for the current player's profile. The exact timer in
`bin/game/logs/helix.html:286-310` computes:

```text
elapsed = current Unix seconds - cashout_bonus_time
remaining = 79,200 - elapsed
available = cashout_bonus_time == 0 or remaining <= 0
```

Thus the shipped profile uses a rolling 22-hour cooldown. It does not use a
midnight/global reset and it does not wait a full 24 hours. A zero value means
the first bonus is immediately available.

The exact `ChainCashOut (AB 00)` body field at `0x084 + playerIndex` becomes
the SWF property `cashoutBonusGranted[playerIndex]`
(`Game.c:147674-147693`). The client presents this server-supplied flag but
does not calculate eligibility or mutate the account from it.

Contemporary Beta 7 patch notes state that the first completed Game Threat
of the day has a `1.5x` chance of earning rare item rewards. They do not claim a
Catalyst, passive-affix, or global drop-rate modifier. This independently fits
the profile timestamp and cash-out flag.

## Server policy and formula

Eligibility is checked immediately before Cash Out generation and revalidated
inside the durable reward transaction. A successful claim writes the current
Unix timestamp to `cashout_bonus_time`, stores the granted flag with the
idempotent result event, grants the selected parts, and advances progression in
one repository save. A failed save restores every mutation. Concurrent
cash-outs regenerate against the winning claim state instead of granting the
bonus twice. Repeated `AC 04` requests replay the stored receipt and flag.

The exact medal contribution remains the client-proven average:

```text
weighted = bronze + 3*silver + 6*gold
ordinary rare chance = floor(weighted / planets completed)
daily rare chance = min(100, floor(3*weighted / (2*planets completed)))
```

The same integer becomes the Rarified-or-better portion of every one-based
Cash Out roll. The existing conservative Purified fallback remains:

```text
purified chance = 0 before completing 5-1
purified chance = min(10, floor(rare chance / 5)) afterward
```

Applying the recovered `1.5x` factor before integer truncation avoids losing a
half-point accumulated across a multi-planet medal average. Applying it to the
combined Rarified-or-better band also scales the nested Purified fallback while
retaining its ten-percent cap. Those two calculation-placement details are an
educated server fallback: the retired server formula has not been recovered.

Long chains already stack with the bonus by retaining medal totals and issuing
one independently rolled reward per completed planet, up to the client's four
reward records. For per-roll rare chance `r` and `n` rewards, the resulting
chance of at least one Rarified-or-better item is `1-(1-r)^n`; this is a derived
probability, not an additional server multiplier.

## Other daily surfaces

- Profile/Helix: displays the exact 22-hour countdown from the persisted last
  claim timestamp.
- Cash Out: receives the durable bonus flag and boosted rarity boundaries.
- Objectives Log: its slot-zero Daily objective is live-session content with
  no expiry field and does not consume `cashout_bonus_time`.
- In-level drops: no build-103 client, content, or patch-note evidence connects
  the Daily Bonus to equipment, Catalyst, DNA, or capsule generation.
- Chain rewards: reward count, reward level, accumulated medals, and the Daily
  Bonus compose in Cash Out; no additional daily chain multiplier was found.

## Remaining authority gaps

The client proves the timestamp/cooldown and presentation fields, while the
patch notes prove the `1.5x` headline. The exact retired-server rounding order,
whether Purified odds were scaled separately, and whether claim time was
recorded at terminal mission completion or durable Cash Out remain unavailable.
Cash Out commit is the conservative claim boundary because that is where the
item becomes durable and where both recovered bonus fields are named.

