# Weapons, Armour, And Combat Gear

Use this when creating or reviewing weapons, armour, damage types, materials, resistances, protections, or wield/wear modifiers. For generic item behavior (weight, value, ids, get/drop), see `items.md` first.

## Current References

- `/doc/build/weapon` and `/doc/build/weapon.list`
- `/doc/build/armour` and `/doc/build/armour.list`
- `/doc/build/damage`
- `/doc/build/material` and `/doc/build/materials`
- `/doc/build/resistance`, `/doc/build/protection`
- `/doc/build/modifiers`
- `/doc/build/prices`
- `/doc/build/combat_issues`
- `/std/weapon.c`, `/std/armour.c`, `/std/include/armour.h`
- Nearby modern gear and gear-generation daemons in the target area.

## Weapons

```c
#include "/path/to/area/header.h"

inherit AREA_I_WEAPON;

static void create() {
    ::create();
    set_name("broadsword");
    set_short("a notched broadsword");
    set_long("The blade is notched from years of service.");
    add_id(({ "sword", "notched broadsword" }));
    set_type("broadsword");
    set_class(10);
    set_weight(4);
    set_value(300);
    set_damage_type("slash");
}
```

Key facts:

- `set_class(n)` sets the weapon class. The std library refuses classes above 20 (and zero-weight weapons). Calibrate against `/doc/build/weapon.list`: a knife is around wc 5, a long sword around wc 10, strong uniques 15–18, and wc 20 is reserved for rare, expensive magic weapons.
- `set_type(type)` sets the subtype. Only `"twohanded"` changes core behavior, but guilds check common types (sword, axe, knife, mace, club, and similar), so pick a real one.
- `set_damage_type(type)` is validated against the message daemon's damage-type list; an invalid type raises an error. See the damage-type list below.
- `set_hit_func(fp)` adds special on-hit behavior; `set_break_chance`, `set_break_msg`, and `set_broken_desc` control breakage.
- `set_wield_modifier(prop, value, maxvalue)` grants stat modifiers while wielded; see Modifiers below.
- Weapons fire `__wield` and `__remove` hooks; see `hooks.md`.

## Armour

```c
inherit AREA_I_ARMOUR;

static void create() {
    ::create();
    set_name("chainmail");
    set_short("a chainmail hauberk");
    set_long("Interlocked steel rings, oiled and well kept.");
    add_id(({ "mail", "hauberk", "chainmail hauberk" }));
    set_type("armour");
    set_ac(3);
    set_weight(8);
    set_value(500);
}
```

Key facts:

- `set_ac(n)`: a random value up to `n` is subtracted from each hit.
- Each armour type has a maximum AC, defined in `/std/include/armour.h`; wearing refuses armour above its type cap. Current caps: `armour` 4, `legging` 2, `magic` 12, and 1 each for `shield`, `glove`, `helmet`, `cloak`, `boot`, `amulet`, `ring`. Types not in the table give no protection.
- Only one piece per type can be worn; the AC of all worn pieces sums.
- Calibrate against `/doc/build/armour.list`: helmet ac 1, leather ac 1–2, chainmail ac 3, plate ac 4.
- `set_wear_modifier(prop, value, maxvalue)` grants stat modifiers while worn.
- Armour fires `__wear` and `__remove` hooks.

## Damage Types

The canonical list is enforced by the message daemon (see `/doc/build/damage`): `acid`, `bite`, `blunt`, `chop`, `claw`, `cold`, `drain`, `electricity`, `fire`, `impact`, `magic`, `pierce`, `poison`, `slash`.

- A lowercase type affects a single hit location on the target.
- A capitalized variant (`Fire`, `Cold`, ...) is "area-of-effect" in the sense of hitting **all hit locations on that one target** — it wears out, burns, freezes, or shatters equipment across the body before damage reduction is applied. Dragonbreath is the canonical example. Capitalized types still check the lowercase `resist_`/`prot_` properties.
- `Drain` is special: it passes through everything, cannot be protected against, and must never be dealt by players — reserve it for monsters/effects where damage must go through.
- Use `blunt` for normal weapons (clubs, maces); `impact` is for supernatural effects (vacuum, meteors).

Invalid damage types create system logs or errors — always use a listed type. Monster spells (`set_spell_dam_type`) accept an array of types; each hit picks one at random. Do not confuse `set_damage_type()` with `set_type()` on weapons.

## Materials

Materials are properties (`add_property("steel")`, `"leather"`, `"silk"`, ...) with per-damage-type wear behavior documented in `/doc/build/material`. Set a material consistent with the description; it affects how the item degrades under different damage types.

## Resistance And Protection

Two parallel property families, per damage type:

- `resist_<type>` (e.g. `resist_fire`): a percentage chance to reduce that damage to one third.
- `prot_<type>` (e.g. `prot_fire`): a percentage chance to prevent that damage entirely.
- Umbrella properties exist for groups: `resist_normal`, `resist_special`, `resistance`, and the `prot_` equivalents.

Read `/doc/build/resistance` and `/doc/build/protection` before assigning values; keep them modest and consistent with nearby gear.

## Modifiers

`set_wield_modifier` / `set_wear_modifier` grant temporary stat or property boosts. Modifiers have three classes — `guild`, `item`, and `special` — and the documented balance rule is `value + maxvalue <= 10`. Read `/doc/build/modifiers` before adding one.

## Values And Prices

Per `/doc/build/prices`: no shop pays more than 1000 coins, and items should never be worth more than 5000. Price gear consistently with its class/AC tier in the `weapon.list` and `armour.list` tables.

## Daemon-Generated Gear

Modern areas centralize randomized gear in daemons (quality-driven factories that compute class/AC/weight/value from a quality score and optionally equip an NPC). Use the area's gear daemons for random loot and standard NPC equipment; hand-code unique items only when fixed behavior and descriptions matter. See `daemons.md`.

## Common Mistakes

- Weapon class above 20 or armour AC above its type cap — silently refused or rejected at wear time.
- Invalid or unlisted damage types.
- Missing `set_type` on armour, leaving it giving no protection.
- Values inconsistent with the documented price tables.
- Modifiers exceeding the `value + maxvalue <= 10` rule.
- Duplicating randomized gear generation that an area daemon already centralizes.
