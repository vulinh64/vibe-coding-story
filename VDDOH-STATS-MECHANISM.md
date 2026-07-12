# Vampires Dawn: Deceit of Heretics — Game Mechanics

*A plain-language explanation of how combat, growth, equipment, and items
actually work under the hood — written for players and modders who want to
understand the systems, not the file format.*

---

## 1. Introduction

This guide explains the core mechanical systems of *Vampires Dawn: Deceit of
Heretics*: how heroes grow, how combat is resolved, how equipment and items
modify your stats, and how monsters are built.

A few things to keep in mind while reading:

- **Vanilla vs. patched behavior.** Some systems described here have known
  bugs in the original game. Where that's the case, this guide describes the
  vanilla (unpatched) behavior first, then notes what a bugfix changes.
- **This is a mechanics guide, not a file-format reference.** It explains what
  the numbers mean and how they interact, not how they're stored on disk.

---

## 2. Core Hero Stats

Every hero has four natural stats that grow with level:

- **Strength** — physical power; drives Attack and part of Defense
- **Spirit** — magical power; drives Resource (Blood/Soul) alongside Vitality
- **Vitality** — toughness; drives Health alongside Strength
- **Speed** — agility; drives Defense, Move, and battle mobility

Speed has a dual role worth calling out: beyond feeding into Defense and Move,
it also determines **battle movement zone** — how many tiles a hero can cover
during their turn in battle. A high-Speed hero isn't just harder to hit and
quicker to act; they can physically reach more of the battlefield.

**Role archetypes**, based on how these stats are distributed across the four
playable heroes:

- **Vince** — the clearest bruiser: high Strength, high Vitality, low Spirit.
- **Romus** — also physical-leaning, but less tanky than Vince.
- **Lara** — the clearest caster: highest Spirit, lower Strength.
- **Manok** — a hybrid/support caster: moderate Strength, higher Spirit than
  Vince/Romus, plus unique spell access.

Broadly: **Vince and Romus** are physical-leaning melee fighters, while
**Lara and Manok** are caster-leaning ranged fighters.

---

## 3. Stat Growth From Level 1 to Level Cap

Each of the four core stats grows according to three settings:

- **Start** — the stat's value at level 1.
- **Level-99 Target** — the stat's theoretical value if a hero reached level
  99. This is a *growth target*, not a hard cap — see below.
- **Growth Curve** — controls how quickly the stat approaches that level-99
  target as the hero levels up.

### Why "Level-99 Target" isn't the real ceiling

Two things keep a hero from ever actually reaching their level-99 target in a
normal playthrough:

1. **The game caps heroes at level 30.** Vanilla heroes never level high
   enough to approach the level-99 number.
2. **Equipment, talents, buffs, and statuses stack on top of natural growth.**
   The level-99 target only describes *natural* stat growth from leveling —
   not the hero's actual stat total once gear and effects are applied.

---

## 4. Derived Combat Stats

Health, Resource, Attack, Defense, Move, and Regen are not stored directly —
they're calculated from the four core stats.

**Health** is weighted mostly toward Vitality, with a smaller Strength
contribution. **Resource** — which is *Blood* for Lara and Vince, and *Soul*
for Romus and Manok — is weighted mostly toward Spirit, with a smaller
Vitality contribution. **Attack** scales directly with Strength. **Defense**
scales with Speed, with a smaller flat contribution from Strength. **Move**
increases gradually with Speed. **Regen** is a small flat baseline.

The actual formulas:

```
Health   = ((Vitality × 70 + Strength × 30) × 12 / 100) + flat HP bonus
Resource = ((Spirit   × 70 + Vitality × 30) × 12 / 100) + flat resource bonus
Attack   = Strength × 5 − 9 + attack bonus
Defense  = Speed × 3 + Strength − 18 + defense bonus
Move     = 2 + Speed / 5 + move bonus
Regen    = 1 + regen bonus
```

All division here is integer division (fractional remainders are dropped),
which is why some totals may look slightly lower than a quick mental
calculation would suggest.

Walking through Lara's level-1 stats (Strength 2, Spirit 3, Vitality 2,
Speed 6, no equipment) shows the formulas at work:

```
Health  = ((2 × 70 + 2 × 30) × 12 / 100) = 24
Blood   = ((3 × 70 + 2 × 30) × 12 / 100) = 32
Attack  = 2 × 5 − 9                       = 1
Defense = 6 × 3 + 2 − 18                  = 2
Move    = 2 + 6 / 5                       = 3
```

These are the safest formulas to reason about at level 1 with no equipment
equipped — at higher levels, the same formulas apply, but they're fed by the
*grown* stat values, and equipment/bonuses add on top separately.

### Confirmed level-1 baseline stats

| Hero | Strength | Spirit | Vitality | Speed | Health | Resource | Attack | Defense | Move | Regen |
|---|---|---|---|---|---|---|---|---|---|---|
| Lara | 2 | 3 | 2 | 6 | 24 | 32 (Blood) | 1 | 2 | 3 | 1 |
| Vince | 3 | 1 | 3 | 6 | 36 | 19 (Blood) | 6 | 3 | 3 | 1 |
| Romus | 3 | 1 | 2 | 6 | 27 | 15 (Soul) | 6 | 3 | 3 | 1 |
| Manok | 3 | 2 | 2 | 6 | 27 | 24 (Soul) | 6 | 3 | 3 | 1 |

---

## 5. Combat Resolution

Physical attacks resolve through a specific sequence of checks, and **which
checks apply depends on the attacker's facing.**

### Miss vs. Evade — two different failure states

- **Missed** — the attack failed an accuracy/hit check.
- **Evaded** — the attack connected, but the defender's evasion/reflex check
  succeeded.

These are tracked separately, and only one can happen per attack.

### Facing determines which checks are even possible

| Facing | Miss check | Evade check | Result |
|---|---|---|---|
| Back attack | Skipped | Skipped | Always connects |
| Side attack | Performed | Skipped | Can miss, cannot be evaded |
| Front attack | Performed | Performed | Can miss, or be evaded if the hit lands |

This matches what the in-game tooltips imply: attacking from the side denies
the enemy any chance to evade (though you can still whiff the attack), and
attacking from the back is a guaranteed hit.

### Resolution order

For any attack that isn't a back attack, the game checks in this order:

1. **Miss check** — if it fails, the attack stops here as a Miss.
2. **Evade check** (front attacks only) — if it succeeds, the attack stops
   here as an Evade.
3. **Damage is applied.**
4. **Critical hit is rolled** (see below).

Because all of these checks are rolled against a 0–99 random range, treat
displayed percentages as gameplay probabilities that can have minor
off-by-one behavior at the extreme ends (0% and 100%).

### Hit Chance

Similar to evasion, hero hit/accuracy appears to use a small flat baseline —
currently estimated at **5%** base hit rate, though this value is less firmly
confirmed than the evasion and crit numbers below and may need further
in-game verification. As with evasion, talents and bonuses likely add on top
of this baseline.

### Evasion

Base evasion for any hero is a flat 5%, plus 2% per level of the **Reflexes**
talent:

| Reflexes level | Evasion |
|---|---|
| 1 | 7% |
| 2 | 9% |
| 3 | 11% |
| 4 | 13% |

### Critical hits

All four heroes share the same base critical stats:

- **Base critical chance:** 5%
- **Base critical damage bonus:** +50%

Two talents scale these:

- **Find Weaknesses** — +1% critical chance per talent level
- **Deadly Might** — +10% critical damage bonus per talent level

So vanilla Deadly Might scales like this:

| Deadly Might level | Critical damage bonus |
|---|---|
| 1 | 60% |
| 2 | 70% |
| 3 | 80% |
| 4 | 90% |

**Important:** critical damage bonus is *bonus* damage, not a total damage
multiplier. A 90% bonus means a critical hit deals your normal damage *plus*
90% more — roughly 190% of normal damage after rounding, not a flat 90% of
normal damage.

---

## 6. Resistances and Elemental Damage

Elemental resistance and status resistance are **two different systems**, and
conflating them leads to wrong expectations.

### Elemental damage resistance = flat reduction, not a shield

For skills that deal raw elemental damage (Fire, Ice, Light, Shadow, Blood),
resistance reduces the incoming damage — it does not grant immunity, even at
100 resistance.

**Example:** a level-4 Fireball with 150 base damage, hitting a target with
100 Fire resistance, still deals 50 damage. The resistance behaved like a flat
150-point reduction, not a percentage shield that would fully block the hit.

### Status effects are a separate system

Statuses like Poison, Bleeding, Blaze, Sleep, and Blind use their own
chance/immunity-style resistance checks — completely separate from elemental
damage reduction. A fire-based skill might *cause* Blaze, but once it's
applied as a status, it's governed by status resistance rules, not the Fire
damage formula.

**Rule of thumb:**

- Fire / Ice / Light / Shadow / Blood *damage* → flat damage reduction
- Status effects (Poison, Bleeding, Sleep, Blind, etc.) → chance/immunity
  resistance

### Default immunities for bloodless enemies

Some enemies and characters lack blood entirely — skeletal or otherwise
bloodless units. By default, these are immune to blood-dependent status
effects (such as Poison and Bleeding), since those effects rely on having
blood to begin with.

### A known vanilla bug: resistance overflow

Status resistance is accumulated internally before being clamped to the
0–100 range players see. Because of how that accumulation works, pushing a
resistance high enough past 100 can cause it to **overflow and wrap around to
0** instead of staying capped at 100.

In practice: a character sitting at 100 resistance to some status effect, who
then equips gear that adds *more* of that same resistance, can end up **more**
vulnerable to that status than before — not less. This is a real, reproducible
vanilla bug, not just a display quirk.

**Practical takeaway for players/modders:** if a character already has 100
resistance to something, be cautious about stacking more of the same
resistance type on top — it can backfire.

---

## 7. Equipment

Equipment layers bonuses on top of a hero's natural stats — it's important to
keep "hero base stats" and "equipment bonuses" mentally separate, since the
in-game stat screen shows them combined.

**Example:** a hero's stat screen might show Attack 17 / Defense 7 while
equipped, but their true base stats (with all items removed) are Attack 6 /
Defense 3. The remaining +11 Attack and +4 Defense are entirely from
equipment — not part of the hero's identity.

In general:

- **Hero base editing/tuning** should focus on Strength, Spirit, Vitality,
  Speed, and their growth curves.
- **Equipment tuning** should focus on Attack, Defense, HP/Resource,
  Movement, Regen, resistances, and on-hit effects.

### A known vanilla quirk: equipment Attack-bonus overwrite

Most equipment bonuses stack normally — if you equip two pieces of armor that
each add resistance, both bonuses apply. But **non-weapon Attack bonuses
behave differently**: instead of adding together, a second piece of gear with
an Attack bonus can **overwrite** the first one's contribution rather than
stacking with it.

**Concrete example (Vince, level 50):**

| Loadout | Attack | Defense | Bleed resistance |
|---|---|---|---|
| Weapon only | 446 | 196 | 0 |
| + War Plate Mail | 456 | 261 | 30 |
| + Strong Helmet | 446 | 291 | 30 |
| + Aaron's Shoes | 446 | 311 | 30 (+80 Sleep) |

Notice that Attack goes from 446 up to 456 after adding War Plate Mail — but
drops back to 446 once Strong Helmet is added, even though nothing was
unequipped. Defense and resistances continued to stack normally the whole
time. This is a real vanilla quirk: **non-weapon equipment Attack bonuses do
not stack with each other; the most recently scanned item's bonus wins.**
Weapon damage itself is unaffected and always stacks correctly.

**Practical takeaway:** if you're optimizing a build, don't assume every
piece of gear with an "Attack bonus" line is additive with every other piece.
Test the actual stat screen total rather than adding tooltip numbers together.

### Runes

Category-7 items (runes) apply different effects depending on whether they're
socketed into a weapon or a piece of armor, plus a shared "common" effect that
applies either way. A few examples:

- **Shadow Rune III** — Weapon: +12 Shadow damage, 4% Blind chance. Armor:
  +30 Anti-shadow, Anti-blind.
- **Status Rune II** — Common: +1 to all four core stats. Weapon: 4% Sleep
  chance. Armor: 20% Anti-sleep.
- **Blood Rune III** — Common: +5 Vitality, +3 Regen. Weapon: 6% Bleeding
  chance. Armor: 30% Anti-bleeding.
- **Rune of Nyr** — Common: +1 Spirit, +5 Max HP. Weapon: +3 Fire damage, 2%
  Weak chance. Armor: +8 Anti-fire damage, 30% Anti-weak.

The pattern: elemental/status *offense* is a weapon-side effect, and the
matching *defense* (Anti-element, Anti-status) is the armor-side mirror of the
same rune.

---

## 8. Items and Consumables

Consumables fall into three functional categories:

1. **Direct-use consumables** — apply an immediate effect (HP/resource
   recovery, status cure, etc.) the moment they're used, with no combat skill
   involved.
2. **Permanent non-battle consumables** — used outside of battle to
   permanently raise a stat. Ankh of Life and Ankh of Magic each permanently
   raise max HP or max Resource by 5 per use.
3. **Combat-only, skill-backed consumables** — used only in battle, and their
   effect is actually delegated to an existing skill rather than having a
   unique effect of its own.

**Examples:**

- **Vampire Stone** (direct-use) — restores a very large amount of HP and
  Resource (effectively a full heal regardless of whether the hero uses Blood
  or Soul) and cures Poison.
- **Might Potion** (combat-only, skill-backed) — rather than having a unique
  effect of its own, it triggers an existing skill that applies the "Strong"
  status. Strong effectively raises the target's Strength — which, per the
  derived-stat formulas in Section 4, increases both Attack and Health (since
  Strength feeds into both). The consumable is essentially a delivery
  mechanism for an existing skill, not a standalone effect.

---

## 9. Monsters

Monster stats follow the **same underlying formulas as hero stats** — a
monster's HP, for instance, is derived from its Strength-like and
Vitality-like values using the identical weighting as heroes.

**Example:** a low-level monster with STR-like 4 and VIT-like 4 produces an
HP value that matches its in-game observed HP exactly, using the same
`(Vitality-weighted + Strength-weighted)` formula heroes use.

### Rewards on death

Defeating a monster grants EXP and Filar (currency) equal to that monster's
configured values, and — separately — a **Soul Restore** amount. Soul Restore
is not part of the EXP/Filar total; it's specifically Soul recovered by
Soul-using heroes (Romus and Manok) when the enemy dies, both during and after
the battle.

**Example:** four identical monsters, each worth 120 EXP / 120 Filar / 25 Soul
Restore, awarded a combined 480 EXP and 480 Filar for the battle — confirming
Soul Restore is tracked completely separately from the battle-result totals.

### A known vanilla bug: EXP award rounding

EXP appears to be awarded to the party in chunks of 4. When the remaining EXP
to award is only 1–3, the game appears to substitute a different value (the
enemy's Filar reward) instead of the correct remaining EXP amount. In
practice, this means the *very last sliver* of EXP from a battle can come out
wrong — usually a minor discrepancy, but worth knowing if your EXP totals seem
slightly off after a battle.

---

## 10. Leveling and EXP Curve

All heroes share the same EXP requirement curve. The EXP needed to go from
level *N* to *N+1* is:

```
EXP needed = 26 × N + 15
```

So going from level 1 to level 2 requires 41 EXP.

### Cumulative EXP thresholds

| Level | Cumulative EXP required |
|---|---|
| 2 | 41 |
| 3 | 108 |
| 4 | 201 |
| 5 | 320 |
| 10 | 1,305 |
| 20 | 5,225 |
| 30 | 11,745 |

The curve is uniform for every hero — there's no per-character EXP scaling,
only the natural-growth-curve differences described in Section 3.

---

## 11. Appendix: Known Bugs

A consolidated reference of vanilla quirks mentioned throughout this guide,
for players and modders who want the short version:

| Bug | Effect | Practical impact |
|---|---|---|
| **Resistance overflow** | Pushing a status resistance past a certain threshold causes it to wrap around and drop to 0 instead of staying at 100 | Stacking extra resistance on top of an already-100 resistance can make a character *more* vulnerable, not less |
| **Equipment Attack-bonus overwrite** | Non-weapon equipment Attack bonuses don't stack with each other — the last one scanned overwrites the others | Equipping multiple Attack-bonus items may not add up the way their tooltips suggest; Defense and resistances are unaffected and stack normally |
| **EXP award rounding** | When the last remaining EXP chunk in a battle-result calculation is 1–3, the game appears to substitute the wrong value | Minor incorrect EXP totals, typically only affecting the last few EXP points of a battle |
