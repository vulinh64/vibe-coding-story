# VDDOH Stats Mechanism

This document records the current confirmed understanding of hero stats in
`Vampires Dawn: Deceit of Heretics`.

## Core Hero Stats

Each hero has four natural growth stats:

- Strength
- Spirit
- Vitality
- Speed

The previous working labels were `Power`, `Spirit`, `Vitality`, and `Agility`.
Screenshots and code behavior confirm that the better labels are:

```text
Power   -> Strength
Agility -> Speed
```

Speed affects the battle movement zone, meaning how many tiles/area the hero can
cover in battle.

## Growth Encoding

Each core stat is stored as three values:

```text
Start
Lv99 Target
Growth Curve
```

Meaning:

- `Start` is the low-level / level-1 value.
- `Lv99 Target` is the natural stat value at level 99, not the normal level cap.
- `Growth Curve` controls how quickly the stat approaches the level-99 target.

The target is not an absolute final stat cap for two reasons:

- The game normally caps heroes at level 30, so vanilla heroes never naturally
  reach their level-99 target.
- Equipment, talents, buffs, and statuses can still modify stats after natural
  growth is calculated.

All three natural stat fields are byte-sized in the packed hero data. In the
current patcher, `Start` and `Lv99 Target` are treated as 7-bit values
(`0..127`) because of the packing format; `Growth Curve` is a full byte
(`0..255`). `Level Cap` is also stored as a 7-bit value (`0..127`).

## Growth Formula

The decompiled code calculates a core stat with integer math:

```text
current =
level * (target - start)
* (level * (100 - curve) / 99 + curve)
/ 99 / 100
+ start
```

Readable form:

```text
growthRange = target - start
curveFactor = level * (100 - curve) / 99 + curve

current = start + level * growthRange * curveFactor / 99 / 100
```

All divisions are integer divisions, matching Java ME behavior.

Curve interpretation:

```text
curve = 0    -> very slow early growth; heavily back-loaded
curve = 100  -> roughly linear growth toward the level-99 target
curve > 100  -> stronger early growth, slower later
curve < 100  -> slower early growth, stronger later
```

Important examples discovered while testing:

```text
Start 15, Lv99 Target 110, Curve 0,   Level 30 -> 23
Start 15, Lv99 Target 110, Curve 100, Level 30 -> 43
Start 15, Lv99 Target 110, Curve 200, Level 30 -> 63
```

This explained the confusing Vince test: changing `Start` to `15` and `Target`
to `110` still produced level-30 Strength `23` when the curve was `0`.

Another confirmed test:

```text
Start 15, Lv99 Target 115, Curve 100, Level Cap 50

Strength @ 50 = 65
Spirit   @ 50 = 65
Vitality @ 50 = 65
Speed    @ 50 = 32  (with Speed Target 50)
```

The patched game showed the same raw stat screen values at level 50, confirming
that the level cap patch and the editor's `@ Cap` preview formula match runtime
behavior.

## Derived Base Stats

From the code and level-1 screenshots, these base formulas are confirmed:

```text
Health   = ((Vitality * 70 + Strength * 30) * 12 / 100) + flat HP bonus
Resource = ((Spirit   * 70 + Vitality * 30) * 12 / 100) + flat resource bonus
Attack   = Strength * 5 - 9 + attack bonus
Defense  = Speed * 3 + Strength - 18 + defense bonus
Move     = 2 + Speed / 5 + move bonus
Regen    = 1 + regen bonus
```

`Resource` is Blood for Lara and Vince, and Soul for Romus and Manok.

These formulas are safest for level-1/no-equipment baselines. Runtime values at
higher levels are recalculated from grown Strength/Spirit/Vitality/Speed and can
also include equipment and other bonuses. For example, the level-50 test with
high grown stats produced `780/780` HP/resource and large attack/defense values
in-game, while the editor currently previews only the four core grown stats.
Future editor columns should add `HP @ Cap`, `Resource @ Cap`, `Attack @ Cap`,
and `Defense @ Cap` once those higher-level derived formulas are fully traced.

## Critical Hits

Critical chance and critical damage are packed together in a short-like runtime
field.

Confirmed attack-side formula:

```text
critChance = baseCritChance + Find Weaknesses bonus
critDamageBonus = baseCritDamageBonus + Deadly Might bonus

if critChance > random(0..99):
  damage = damage + damage * critDamageBonus / 100
```

The damage flag `0x10000` is then set so the UI draws the critical-hit marker.

The confirmed hero talent mapping is:

```text
Find Weaknesses -> hero bonus id 3 -> +1% crit chance per talent level
Deadly Might    -> hero bonus id 4 -> +10% crit damage bonus per talent level
```

The base critical damage bonus appears to be `50%`. Therefore vanilla Deadly
Might previews as:

```text
Level 1: 60%
Level 2: 70%
Level 3: 80%
Level 4: 90%
```

Important wording:

```text
Critical damage bonus is bonus damage, not total damage multiplier.
```

So `90%` means a critical hit deals:

```text
damage + damage * 90 / 100
```

or roughly `190%` of normal damage after integer truncation.

## Miss And Evasion

Physical attacks have two separate failure states:

```text
MISSED = accuracy/hit check failed
EVADED = evasion/reflex check failed
```

The combat result flags are:

```text
0x4000 = MISSED
0x8000 = EVADED
```

Hero runtime hit/evasion appears to be packed in `var_short_f`:

```text
high byte = hit / connect chance
low byte  = evasion / reflex chance
```

The hero-side base evasion formula is:

```text
evasionChance = 5 + Reflexes bonus
```

Reflexes is the passive hero talent with hero bonus id `5`:

```text
Reflexes -> +2% evasion per talent level
```

So vanilla Reflexes gives:

```text
Level 1:  7% evasion
Level 2:  9% evasion
Level 3: 11% evasion
Level 4: 13% evasion
```

Facing controls which checks are allowed.

For a physical attack:

```text
Back attack:
  skips MISS check
  skips EVADE check
  result: always connects

Side attack:
  performs MISS check
  skips EVADE check
  result: can MISS, cannot be EVADED

Front attack:
  performs MISS check
  performs EVADE check
  result: can MISS, or can be EVADED if the hit roll succeeds first
```

This matches gameplay text: attacking from the side prevents the enemy from
evading, but the attack can still miss; attacking from the back always hits.

The order is important:

```text
if not back attack and targetHitChance < random(0..99):
  MISSED
  stop

if front attack and evasionChance > random(0..99):
  EVADED
  stop

apply damage
roll critical hit
```

Because the game uses comparisons against `random(0..99)`, many displayed
percent-like values should be treated as gameplay probabilities with possible
off-by-one quirks around `0` and `100`.

## Elemental Resistance And Damage

Elemental resistance is not the same as status immunity. For skills that deal
raw elemental damage, the game subtracts or mitigates against the incoming damage
value rather than turning `100` resistance into full immunity.

This applies to damage-bearing skill elements:

```text
Fire
Frost / Ice
Light
Shadow
Blood
```

Confirmed gameplay implication:

```text
Fireball level 4 base damage: 150
Target fire resistance:       100
Final fire damage:             50
```

So a target with `100` Fire resistance can still take damage from a `150` damage
Fireball. The resistance behaves like a flat damage reduction in this case, not
like a 100% percentage shield.

This does not mean status effects work the same way. Statuses such as Poison,
Bleeding, Blaze, Sleep, Blind, and similar debuffs use status application and
status resistance checks instead of elemental damage reduction. For example,
Blaze may be caused by a fire-based skill or environment, but once treated as a
status it belongs to the status-resistance system, not the Fire damage-reduction
formula.

This matters for editor wording:

```text
Fire/Frost/Light/Shadow/Blood damage -> damage reduction / protection value
Status effects                       -> chance/immunity-style status blocking
```

## Confirmed Level-1 Base Values

These values were confirmed from in-game screenshots with no relevant stat items
equipped.

```text
Lara:
  Strength 2
  Spirit   3
  Vitality 2
  Speed    6
  Health   24
  Blood    32
  Attack   1
  Defense  2
  Move     3
  Regen    1

Vince:
  Strength 3
  Spirit   1
  Vitality 3
  Speed    6
  Health   36
  Blood    19
  Attack   6
  Defense  3
  Move     3
  Regen    1

Romus:
  Strength 3
  Spirit   1
  Vitality 2
  Speed    6
  Health   27
  Soul     15
  Attack   6
  Defense  3
  Move     3
  Regen    1

Manok:
  Strength 3
  Spirit   2
  Vitality 2
  Speed    6
  Health   27
  Soul     24
  Attack   6
  Defense  3
  Move     3
  Regen    1
```

## Formula Checks

Lara:

```text
Health = ((2 * 70 + 2 * 30) * 12 / 100) = 24
Blood  = ((3 * 70 + 2 * 30) * 12 / 100) = 32
Attack = 2 * 5 - 9 = 1
Defense = 6 * 3 + 2 - 18 = 2
Move = 2 + 6 / 5 = 3
```

Vince:

```text
Health = ((3 * 70 + 3 * 30) * 12 / 100) = 36
Blood  = ((1 * 70 + 3 * 30) * 12 / 100) = 19
Attack = 3 * 5 - 9 = 6
Defense = 6 * 3 + 3 - 18 = 3
```

Romus:

```text
Health = ((2 * 70 + 3 * 30) * 12 / 100) = 27
Soul   = ((1 * 70 + 2 * 30) * 12 / 100) = 15
Attack = 3 * 5 - 9 = 6
Defense = 6 * 3 + 3 - 18 = 3
```

Manok:

```text
Health = ((2 * 70 + 3 * 30) * 12 / 100) = 27
Soul   = ((2 * 70 + 2 * 30) * 12 / 100) = 24
Attack = 3 * 5 - 9 = 6
Defense = 6 * 3 + 3 - 18 = 3
```

## Character Role Implications

The stats support the gameplay roles:

```text
Vince / Romus = physical-leaning melee fighters
Lara / Manok  = caster-leaning ranged fighters
```

More specifically:

- Vince is the clearest bruiser: high Strength, high Vitality, low Spirit.
- Romus is also physical-leaning, but less tanky than Vince.
- Lara is the clearest caster: highest Spirit, lower Strength.
- Manok is a hybrid/support caster: moderate Strength and higher Spirit than
  Vince/Romus, plus unique spell access.

## Equipment and Bonus Separation

Hero base data should be separated from equipment and item bonuses.

Example: Manok originally showed:

```text
Attack 17
Defense 7
```

After removing his items, he showed:

```text
Attack 6
Defense 3
```

So the extra values were equipment bonuses:

```text
Attack bonus  +11
Defense bonus +4
```

This confirms that base hero editing should focus on Strength, Spirit, Vitality,
Speed, and their growth curves. Equipment/item editing should expose attack,
defense, HP/resource, movement, regen, resistance, and on-hit modifiers.

## Confirmed Resistances From Screenshots

From the stat screen:

```text
Lara:
  Confuse resistance 100

Vince:
  Confuse resistance 100

Romus:
  Blood resistance 100
  Poison resistance 100
  Bleed resistance 100
  Confuse resistance 100

Manok:
  Blood resistance 100
  Poison resistance 100
  Bleed resistance 100
  Confuse resistance 100
```

Important correction:

```text
The ?! icon is Confuse / Confusion resistance, not Fear resistance.
```

Romus and Manok are skeletons, so they have no blood to suck and are immune to
poison and bleeding.

## Resistance Overflow Bug

Hero status resistances are accumulated in a Java `byte[]` before they are
clamped to the display/gameplay range.

The relevant runtime pattern is:

```text
resistance[id] = (byte)(resistance[id] + bonus)

if resistance[id] < 0:
  resistance[id] = 0
else if resistance[id] > 100:
  resistance[id] = 100
```

Because the cast to `byte` happens before clamping, values above `127` overflow
into the negative byte range. The later clamp then treats the negative value as
bad and sets it to `0`.

Examples:

```text
100 + 15  = 115  -> valid byte -> clamped to 100
100 + 27  = 127  -> valid byte -> clamped to 100
100 + 28  = 128  -> byte overflow to -128 -> clamped to 0
100 + 100 = 200  -> byte overflow to -56  -> clamped to 0
```

This is a blatant gameplay bug. A hero with natural `100` resistance can become
vulnerable if equipment adds enough extra resistance of the same type.

Concrete example:

```text
Romus has 100 Bleeding resistance.
If equipment adds enough Bleeding resistance to push the temporary sum to 128+,
the byte overflows and the final displayed/runtime resistance becomes 0.
Romus can then bleed.
```

Practical editor rule:

```text
Never let a hero's combined status resistance exceed 127 before clamping.
If the hero already has 100 resistance, avoid adding any extra resistance of the
same type unless intentionally testing the overflow bug.
```

## EXP Curve and Leveling Notes

All heroes share the same EXP threshold curve.

Confirmed first threshold:

```text
Level 1 -> 2 requires 41 EXP
```

The per-level increment follows:

```text
EXP needed from level N to N+1 = 26 * N + 15
```

Confirmed cumulative thresholds:

```text
Level  2:    41
Level  3:   108
Level  4:   201
Level  5:   320
Level 10:  1305
Level 20:  5225
Level 30: 11745
```

There is also a likely EXP award bug in the battle-result code: EXP is awarded
in chunks of 4, but when the remaining EXP is `1..3`, the code appears to award
the remaining Filar value instead of the remaining EXP value. This can make the
final small EXP remainder after a battle incorrect.

## Editor Notes

The hero editor should expose editable natural growth fields:

```text
Strength Start / Lv99 Target / Growth Curve
Spirit Start / Lv99 Target / Growth Curve
Vitality Start / Lv99 Target / Growth Curve
Speed Start / Lv99 Target / Growth Curve
Level Cap
```

It should also show read-only derived base values:

```text
Base HP
Base Resource
Base Attack
Base Defense
Base Move
Base Regen
```

The current editor also shows read-only cap previews:

```text
STR @ Cap
SPI @ Cap
VIT @ Cap
SPD @ Cap
```

These preview the grown core stats at `Level Cap` using the exact integer growth
formula above. They are now the preferred way to tune hero stats instead of
trying to reason from `Lv99 Target` alone.

Variable-length hero arrays such as starting talents, starting equipment, and
bonus/resistance lists should be handled separately and conservatively.