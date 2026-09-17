# Generated item power budget

Campaign drops and campaign cash-out rewards now select complete packaged affix combinations under a shared weighted ceiling. Existing stored items, fixed shop stock, explicit developer grants, native attribute calculations, and combat formulas are unchanged.

The budget is `65 + 0.6 * min(item level, 100) + 15 * normalized rarity`, plus 10 for unique-family items. Normalized rarity is Basic 0, Uncommon/Unique 1, Rare/Rare Unique 2, and Epic/Epic Unique 3. This is a ceiling, not a promise that every discrete packaged roll spends exactly the same amount.

## Pricing

The whole item's base stats and affixes are evaluated together, including native rarity allocations and the extra-stat-count bonus. Flat stats are priced before integer truncation and divided by the native item-level scale; this preserves level progression while preventing percentage rolls from growing exponentially. Negative stats do not refund budget. Modifier-vector channels that are neither displayed by the profile nor consumed by authoritative combat are metadata and contribute no player-power cost.

| Stat | Budget points |
| --- | --- |
| Strength, Dexterity, Mind | 5 per point / level scale |
| Health | 1 per point / level scale |
| Power | 1.5 per point / level scale |
| Physical/Energy Defense | 0.75 per point / level scale |
| Critical Rating | 0.5 per point / level scale |
| Flat physical/energy damage | 8 per point / level scale |
| Minimum/maximum weapon damage | 12 per endpoint point / level scale |
| Direct attack damage | 24 per point / level scale |
| Attack speed and general/weapon/direct damage bonuses | 2 per percentage point |
| Cooldown reduction, Overdrive duration, channel reduction | 3 per percentage point |
| Life/mana steal | 8 per percentage point |
| Immunity flags | 100 per flag |
| Other native percentage attributes | 1.5 per percentage point |

These weights are initial balance policy, not recovered retail tuning or an exact hero-independent DPS equivalence. Multiplicative speed/damage and sustain deserve particular attention during real play.

## Complete rolls

The seeded selector retains native level, class, science, basic eligibility, unique-family, and rarity-slot restrictions. It backtracks over complete suffix/prefix combinations; an expensive affix may remain when cheaper companions fit. Secondary prefixes must be distinct. It never clears a required slot to get under budget. Because packaged affixes cannot be scaled continuously, the selector raises the whole-item ceiling through bounded 25%, 50%, 100%, 200%, and 300% feasibility bands only when no complete combination exists under the tighter band. This preserves the rarity's required slots and prevents an impossible discrete budget from aborting a campaign Cash Out.

Affix magnitudes are packaged content, not individually writable stats. Consequently, reducing an over-budget roll means selecting a cheaper eligible affix combination, potentially changing its stat mix or name. No client assets or tooltip formulas are patched.

## Verification limits

Source review and formatting only; builds, compilation, and tests were not run. Snapshot evidence from a live Cash Out proved that rejecting non-profile modifier-vector channels could eliminate every eligible roll, so those channels are now excluded from player-power pricing and discrete combinations use the smallest feasible bounded ceiling. Validate fresh drops in the production path before treating the initial weights as play-balanced.
