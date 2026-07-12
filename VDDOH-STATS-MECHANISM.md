# Vampires Dawn: Deceit of Heretics — Game Mechanics

*A plain-language explanation of how combat, growth, equipment, and items actually work under the hood — written for players and modders who want to understand the systems, not the file format.*

---

## 1. Introduction

This guide explains the core mechanical systems of *Vampires Dawn: Deceit of Heretics*: how heroes grow, how combat is resolved, how equipment and items modify your stats, and how monsters are built.

A few things to keep in mind while reading:

- **Vanilla vs. patched behavior.** Some systems described here have known bugs in the original game. Where that's the case, this guide describes the vanilla (unpatched) behavior first, then notes what a bugfix changes.
- **This is a mechanics guide, not a file-format reference.** It explains what the numbers mean and how they interact, not how they're stored on disk.

---

## 2. Core Hero Stats

Every hero has four natural stats that grow with level:

- **Strength** — physical power; drives Attack and part of Defense
- **Spirit** — magical power; drives Resource (Blood/Soul) alongside Vitality
- **Vitality** — toughness; drives Health alongside Strength
- **Speed** — agility; drives Defense, Move, and battle mobility

Speed has a dual role worth calling out: beyond feeding into Defense and Move, it also determines **battle movement zone** — how many tiles a hero can cover during their turn in battle. A high-Speed hero isn't just harder to hit and quicker to act; they can physically reach more of the battlefield.

**Role archetypes**, based on how these stats are distributed across the four playable heroes:

- **Vince** — the clearest bruiser: high Strength, high Vitality, low Spirit.
- **Romus** — also physical-leaning, but less tanky than Vince.
- **Lara** — the clearest caster: highest Spirit, lower Strength.
- **Manok** — a hybrid/support caster: moderate Strength, higher Spirit than Vince/Romus, plus unique spell access.

Broadly: **Vince and Romus** are physical-leaning melee fighters, while **Lara and Manok** are caster-leaning ranged fighters.

---

## 3. Stat Growth From Level 1 to Level Cap

Each of the four core stats grows according to three settings:

- **Start** — the stat's value at level 1.
- **Level-99 Target** — the stat's theoretical value if a hero reached level 99. This is a *growth target*, not a hard cap — see below.
- **Growth Curve** — controls how quickly the stat approaches that level-99 target as the hero levels up.

### Why "Level-99 Target" isn't the real ceiling

Two things keep a hero from ever actually reaching their level-99 target in a normal playthrough:

1. **The game caps heroes at level 30.** Vanilla heroes never level high enough to approach the level-99 number.
2. **Equipment, talents, buffs, and statuses stack on top of natural growth.** The level-99 target only describes *natural* stat growth from leveling — not the hero's actual stat total once gear and effects are applied.

---

## 4. Derived Combat Stats

Health, Resource, Attack, Defense, Move, and Regen are not stored directly — they're calculated from the four core stats.

**Health** is weighted mostly toward Vitality, with a smaller Strength contribution. **Resource** — which is *Blood* for Lara and Vince, and *Soul* for Romus and Manok — is weighted mostly toward Spirit, with a smaller Vitality contribution. **Attack** scales directly with Strength. **Defense** scales with Speed, with a smaller flat contribution from Strength. **Move** increases gradually with Speed. **Regen** is a small flat baseline.

The actual formulas:

```
Health   = ((Vitality × 70 + Strength × 30) × 12 / 100) + flat HP bonus
Resource = ((Spirit   × 70 + Vitality × 30) × 12 / 100) + flat resource bonus
Attack   = Strength × 5 − 9 + attack bonus
Defense  = Speed × 3 + Strength − 18 + defense bonus
Move     = 2 + Speed / 5 + move bonus
Regen    = 1 + regen bonus
```

All division here is integer division (fractional remainders are dropped), which is why some totals may look slightly lower than a quick mental calculation would suggest.

Walking through Lara's level-1 stats (Strength 2, Spirit 3, Vitality 2, Speed 6, no equipment) shows the formulas at work:

```
Health  = ((2 × 70 + 2 × 30) × 12 / 100) = 24
Blood   = ((3 × 70 + 2 × 30) × 12 / 100) = 32
Attack  = 2 × 5 − 9                       = 1
Defense = 6 × 3 + 2 − 18                  = 2
Move    = 2 + 6 / 5                       = 3
```

These are the safest formulas to reason about at level 1 with no equipment equipped — at higher levels, the same formulas apply, but they're fed by the *grown* stat values, and equipment/bonuses add on top separately.

### Confirmed level-1 baseline stats

| Hero | Strength | Spirit | Vitality | Speed | Health | Resource | Attack | Defense | Move | Regen |
|---|---|---|---|---|---|---|---|---|---|---|
| Lara | 2 | 3 | 2 | 6 | 24 | 32 (Blood) | 1 | 2 | 3 | 1 |
| Vince | 3 | 1 | 3 | 6 | 36 | 19 (Blood) | 6 | 3 | 3 | 1 |
| Romus | 3 | 1 | 2 | 6 | 27 | 15 (Soul) | 6 | 3 | 3 | 1 |
| Manok | 3 | 2 | 2 | 6 | 27 | 24 (Soul) | 6 | 3 | 3 | 1 |

### When Regen actually applies

Regen — the base stat plus any bonuses from skills and items — determines how much HP is recovered at the end of a battle turn, but only once **both sides** (yours and the enemy's) have fully used up their moves for that turn. Regen doesn't tick per individual action; it ticks once per completed turn cycle.

One consequence of this: if a battle ends *before* a full turn cycle completes — for example, the enemy is defeated mid-turn — the pending Regen tick for that turn is never applied. A victory doesn't retroactively grant the HP that Regen would have recovered if the turn had finished normally.

---

## 5. Combat Resolution

Physical attacks resolve through a specific sequence of checks, and **which checks apply depends on the attacker's facing.**

### Miss vs. Evade — two different failure states

- **Missed** — the attack failed an accuracy/hit check.
- **Evaded** — the attack connected, but the defender's evasion/reflex check succeeded.

These are tracked separately, and only one can happen per attack.

### Facing determines which checks are even possible

| Facing | Miss check | Evade check | Result |
|---|---|---|---|
| Back attack | Skipped | Skipped | Always connects |
| Side attack | Performed | Skipped | Can miss, cannot be evaded |
| Front attack | Performed | Performed | Can miss, or be evaded if the hit lands |

This matches what the in-game tooltips imply: attacking from the side denies the enemy any chance to evade (though you can still whiff the attack), and attacking from the back is a guaranteed hit.

### Resolution order

The game checks in this order:

1. **Miss check** — skipped entirely for back attacks; for side and front attacks, if it fails, the attack stops here as a Miss.
2. **Evade check** — skipped for back and side attacks; for front attacks, if it succeeds, the attack stops here as an Evade.
3. **Damage is applied.**
4. **Critical hit is rolled** (see below).

Back attacks skip both the miss and evade checks entirely — they always reach steps 3 and 4, meaning damage and the critical-hit roll are still applied normally, it's only the miss/evade checks that are bypassed.

Because all of these checks are rolled against a 0–99 random range, treat displayed percentages as gameplay probabilities that can have minor off-by-one behavior at the extreme ends (0% and 100%).

### Physical damage calculation

When a physical attack connects, its damage is reduced by the target's Defense — which itself is a combination of the target's core stats, equipped items, and skill effects. If the target's Defense is greater than the attacker's Attack, the hit deals no damage at all, and the game displays "RESISTED" on screen instead of a damage number.

### Hit Chance

Similar to evasion, hero hit/accuracy appears to use a small flat baseline — currently estimated at **5%** base hit rate, though this value is less firmly confirmed than the evasion and crit numbers below and may need further in-game verification. As with evasion, talents and bonuses likely add on top of this baseline.

### Evasion

Base evasion for any hero is a flat 5%, plus 2% per level of the **Reflexes** talent:

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

**Important:** critical damage bonus is *bonus* damage, not a total damage multiplier. A 90% bonus means a critical hit deals your normal damage *plus* 90% more — roughly 190% of normal damage after rounding, not a flat 90% of normal damage.

### Magical damage

There are five types of magical damage, each carrying a chance to inflict a matching status effect:

- **Fire** — chance to inflict Blaze
- **Light** — chance to inflict Mute
- **Ice** — chance to inflict Paralysis
- **Shadow** — chance to inflict Blind
- **Blood** — some Blood-damage skills can inflict Bleeding on enemies that have blood, and some inflict Frenzy instead

For any skill that deals magical damage, final damage is calculated as:

```
Final damage = Damage − Target resistance
```

If the target has an elemental shield matching the attack's element — for example, a Light shield against a Light-based attack — the attack deals no damage at all, regardless of the raw damage value.

Some weapons deal magical damage rather than physical damage. When they do, the same formula applies (`Damage − Target resistance`), and the hero's Strength or other physical-bonus stats have no bearing on the result.

For Blood damage specifically, only enemies that actually have blood can take damage from it — bloodless enemies (see Section 6) take no Blood damage at all.

---

## 6. Resistances and Elemental Damage

Elemental resistance and status resistance are **two different systems**, and conflating them leads to wrong expectations.

### Elemental damage resistance = flat reduction, not a shield

For skills that deal raw elemental damage (Fire, Ice, Light, Shadow, Blood), resistance reduces the incoming damage — it does not grant immunity, even at 100 resistance.

**Example:** a level-4 Fireball with 150 base damage, hitting a target with 100 Fire resistance, still deals 50 damage. The resistance behaved like a flat 150-point reduction, not a percentage shield that would fully block the hit.

### Status effects are a separate system

Statuses like Poison, Bleeding, Blaze, Sleep, and Blind use their own chance/immunity-style resistance checks — completely separate from elemental damage reduction. A fire-based skill might *cause* Blaze, but once it's applied as a status, it's governed by status resistance rules, not the Fire damage formula.

**Rule of thumb:**

- Fire / Ice / Light / Shadow / Blood *damage* → flat damage reduction
- Status effects (Poison, Bleeding, Sleep, Blind, etc.) → chance/immunity resistance

### Default immunities for bloodless enemies

Some enemies and characters lack blood entirely — skeletal or otherwise bloodless units. By default, these are immune to blood-dependent status effects (such as Poison and Bleeding), since those effects rely on having blood to begin with.

On the party side, **Romus and Manok** — both of whom are Vince's servants — are skeletal-type units, and are therefore bloodless in the same sense.

Bloodless enemies also have a visible tell: when attacked, they don't display the "blood spilling" effect that fleshed enemies show. This isn't just cosmetic — it signals that any skill relying on drawing blood from the target will have no effect. For example, **Bloodsucking** won't grant Lara or Vince any Blood when used against a bloodless enemy, and **Life Leech** is simply wasted, since there's no blood to drain in the first place.

### A known vanilla bug: resistance overflow

Status resistance is accumulated internally before being clamped to the 0–100 range players see. Because of how that accumulation works, pushing a resistance high enough past 100 can cause it to **overflow and wrap around to 0** instead of staying capped at 100.

In practice: a character sitting at 100 resistance to some status effect, who then equips gear that adds *more* of that same resistance, can end up **more** vulnerable to that status than before — not less. This is a real, reproducible vanilla bug, not just a display quirk.

### Statuses that persist after battle

Some statuses don't clear when a battle ends — they carry over into overworld movement, ticking down (or continuing to apply) as the party moves. Poison (see below) is the clearest example: it persists until cured by an Antidote-like item, regardless of whether the party is in or out of battle. Other effects, such as anti-elemental buffs or HP regen effects, can similarly last for a number of ticks tied to party movement on the area map or overworld, rather than expiring the moment battle ends.

### Two special statuses worth knowing

Most statuses wear off naturally after a battle or a set number of turns, but two behave differently:

- **Poisoned** — unlike other statuses, Poison does not wear off naturally. It persists indefinitely, dealing HP damage per tick, until it's cured by an Antidote-like item (Antidote, Serum, Vampire Stone, etc.). If left unchecked, the affected hero will bottom out at 1 HP outside of battle rather than dying outright. This applies to **Vince and Lara only**. In combat, the poison damage is partly offset by the hero's HP regeneration ticking on the same turn — but this offset only applies during battle, not while poisoned outside of combat.
- **Blaze (desert areas)** — in desert areas, the strong sun causes Vince and Lara to suffer perpetual Blaze damage per tick for as long as they remain in the area, only clearing once they leave. This is a different mechanism from the normal Blaze status caused by fire-related skills, which wears off after battle ends or after a set number of turns. Note that this desert Blaze only triggers on actually entering a desert area — such as a dungeon or battle zone located within one — and not simply from the party passing over desert terrain while traveling on the overworld map.

### Death / Rigidity

There's a third special status worth calling out: **Rigor** (as datamined internally) or **Death**, and it behaves differently depending on which side it applies to.

For enemies, Death is straightforward: once an enemy reaches 0 HP, it's removed from the battle.

For party members, it works differently. When a hero reaches 0 HP, they don't die outright — instead they fall into **Rigidity** status, shown on screen as a tombstone in place of the hero's normal sprite. Rigidity can be cured two ways: using a Life Potion item during battle, or simply surviving to the end of the battle, at which point the hero returns with 1 HP — and still receives EXP from the battle despite having fallen. If every hero in the party falls into Rigidity at the same time, it's game over.

Separately, **Vince and Romus** have access to a spell literally called **Death**. As the name suggests, it has a chance to instantly remove enemies within its area of effect from the battle — regardless of their current HP and regardless of their resistance to Shadow-elemental attacks, including on some bosses. This isn't guaranteed on every cast: it's chance-based, and the chance of it triggering increases as the spell is leveled up.

---

## 7. Equipment

Equipment layers bonuses on top of a hero's natural stats — it's important to keep "hero base stats" and "equipment bonuses" mentally separate, since the in-game stat screen shows them combined.

### Equipment slots

Each hero has the following equipment slots:

- **Helmet**
- **Necklace**
- **Ring** — two ring slots per character
- **Body armor**
- **Boot armor**
- **Main weapon**

### Weapons are character-specific

Each hero uses a unique weapon type, and this isn't interchangeable between characters:

- **Vince** — Blade
- **Romus** — Lance
- **Lara** — Staff (has an attack range, unlike melee weapons)
- **Manok** — Crossbow (a ranged weapon)

### Shop upgrades unlock every 5 levels

New, upgraded weapons, armors, necklaces, and rings become available for purchase at the shop every time a hero crosses a multiple-of-5 level threshold. For example, if Vince levels up from 9 to 10, a new upgraded Blade becomes purchasable for him at the shop. This applies per-hero — each character's shop upgrades unlock based on their own level, not the party's overall progress.

### Rune slots

Weapons and body armor each have rune slots, letting you socket in runes (see below) to add extra effects on top of the item's base stats.

**Example:** a hero's stat screen might show Attack 17 / Defense 7 while equipped, but their true base stats (with all items removed) are Attack 6 / Defense 3. The remaining +11 Attack and +4 Defense are entirely from equipment — not part of the hero's identity.

### A known vanilla quirk: equipment Attack-bonus overwrite

Most equipment bonuses stack normally — if you equip two pieces of armor that each add resistance, both bonuses apply. But **non-weapon Attack bonuses behave differently**: instead of adding together, a second piece of gear with an Attack bonus can **overwrite** the first one's contribution rather than stacking with it.

**Concrete example (Vince, level 50):**

> *Note: level 50 is beyond the vanilla level cap of 30, so these specific stat totals are from a modded/raised level cap rather than a normal playthrough. The stat numbers themselves are modded — but the Attack-bonus overwrite behavior they demonstrate is a vanilla game mechanism, not something introduced by modding.*

| Loadout | Attack | Defense | Bleed resistance |
|---|---|---|---|
| Weapon only | 446 | 196 | 0 |
| + War Plate Mail | 456 | 261 | 30 |
| + Strong Helmet | 446 | 291 | 30 |
| + Aaron's Shoes | 446 | 311 | 30 (+80 Sleep) |

Notice that Attack goes from 446 up to 456 after adding War Plate Mail — but drops back to 446 once Strong Helmet is added, even though nothing was unequipped. Defense and resistances continued to stack normally the whole time. This is a real vanilla quirk: **non-weapon equipment Attack bonuses do not stack with each other; the most recently scanned item's bonus wins.** Weapon damage itself is unaffected and always stacks correctly.

### Runes

Category-7 items (runes) apply different effects depending on whether they're socketed into a weapon or a piece of armor, plus a shared "common" effect that applies either way. A few examples:

- **Shadow Rune III** — Weapon: +12 Shadow damage, 4% Blind chance. Armor: +30 Anti-shadow, Anti-blind.
- **Status Rune II** — Common: +1 to all four core stats. Weapon: 4% Sleep chance. Armor: 20% Anti-sleep.
- **Blood Rune III** — Common: +5 Vitality, +3 Regen. Weapon: 6% Bleeding chance. Armor: 30% Anti-bleeding.
- **Rune of Nyr** — Common: +1 Spirit, +5 Max HP. Weapon: +3 Fire damage, 2% Weak chance. Armor: +8 Anti-fire damage, 30% Anti-weak.

The pattern: elemental/status *offense* is a weapon-side effect, and the matching *defense* (Anti-element, Anti-status) is the armor-side mirror of the same rune.

---

## 8. Items and Consumables

Consumables fall into three functional categories:

1. **Direct-use consumables** — apply an immediate effect (HP/resource recovery, status cure, etc.) the moment they're used, with no combat skill involved.
2. **Permanent non-battle consumables** — used outside of battle to permanently raise a stat. Ankh of Life and Ankh of Magic each permanently raise max HP or max Resource by 5 per use.
3. **Combat-only, skill-backed consumables** — used only in battle, and their effect is actually delegated to an existing skill rather than having a unique effect of its own.

**Examples:**

- **Vampire Stone** (direct-use) — restores a very large amount of HP and Resource (effectively a full heal regardless of whether the hero uses Blood or Soul) and cures Poison.
- **Might Potion** (combat-only, skill-backed) — rather than having a unique effect of its own, it triggers an existing skill that applies the "Strong" status. Strong effectively raises the target's Strength — which, per the derived-stat formulas in Section 4, increases both Attack and Health (since Strength feeds into both). The consumable is essentially a delivery mechanism for an existing skill, not a standalone effect.

---

## 9. Group Talents

Group Talents are skills shared by the entire party, rather than tied to an individual hero. There are three:

- **Bloodsucking** — controls how much Blood Lara and Vince can drain per tick from an NPC outside of combat, before that NPC is removed from the game entirely.
- **Stealing** — controls how much Filar (in-game currency) the party can steal from stealable NPCs. Level 1 unlocks stealing from kids, level 2 from women, level 3 from men, and level 4 additionally lets the party steal extra items from them, on top of Filar.
- **Sharp Senses** — controls what the party can find while exploring. Level 1 reveals good hidden alcoves containing an item (shown by a single red arrow pointing toward the spot). Level 2 reveals hidden gold items on the ground. Level 3 reveals extremely well-hidden spots (shown by two red arrows). Level 4 reveals red items on the ground, which are rarer than gold items.

**Spoiler:** Manok leaves the party permanently at the final battle. Since his skill points are lost at that point regardless, it's generally more efficient to spend Manok's skill points on leveling up Group Talents rather than his personal skills.

---

## 10. Monsters

Monster stats follow the **same underlying formulas as hero stats** — a monster's HP, for instance, is derived from its Strength-like and Vitality-like values using the identical weighting as heroes.

**Example:** a low-level monster with STR-like 4 and VIT-like 4 produces an HP value that matches its in-game observed HP exactly, using the same `(Vitality-weighted + Strength-weighted)` formula heroes use.

### Rewards on death

Defeating a monster grants EXP and Filar (currency) equal to that monster's configured values, and — separately — a **Soul Restore** amount. Soul Restore is not part of the EXP/Filar total; it's specifically Soul recovered by Soul-using heroes (Romus and Manok) when the enemy dies, both during and after the battle.

**Example:** four identical monsters, each worth 120 EXP / 120 Filar / 25 Soul Restore, awarded a combined 480 EXP and 480 Filar for the battle — confirming Soul Restore is tracked completely separately from the battle-result totals.

### A known vanilla bug: EXP award rounding

EXP appears to be awarded to the party in chunks of 4. When the remaining EXP to award is only 1–3, the game appears to substitute a different value (the enemy's Filar reward) instead of the correct remaining EXP amount. In practice, this means the *very last sliver* of EXP from a battle can come out wrong — usually a minor discrepancy, but worth knowing if your EXP totals seem slightly off after a battle.

---

## 11. Leveling and EXP Curve

All heroes share the same EXP requirement curve. The EXP needed to go from level *N* to *N+1* is:

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

The curve is uniform for every hero — there's no per-character EXP scaling, only the natural-growth-curve differences described in Section 3.

---

## 12. Appendix: Known Bugs

A consolidated reference of vanilla quirks mentioned throughout this guide, for players and modders who want the short version:

| Bug | Effect | Practical impact |
|---|---|---|
| **Resistance overflow** | Pushing a status resistance past a certain threshold causes it to wrap around and drop to 0 instead of staying at 100 | Stacking extra resistance on top of an already-100 resistance can make a character *more* vulnerable, not less |
| **Equipment Attack-bonus overwrite** | Non-weapon equipment Attack bonuses don't stack with each other — the last one scanned overwrites the others | Equipping multiple Attack-bonus items may not add up the way their tooltips suggest; Defense and resistances are unaffected and stack normally |
| **EXP award rounding** | When the last remaining EXP chunk in a battle-result calculation is 1–3, the game appears to substitute the wrong value | Minor incorrect EXP totals, typically only affecting the last few EXP points of a battle |
