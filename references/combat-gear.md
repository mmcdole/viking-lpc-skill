# Weapons, Armour, And Combat Gear

Use this when creating or reviewing weapons, armour, damage types, materials, resistances, protections, wear/wield modifiers, or stat-boosting gear. For generic item behavior (weight, value, ids, get/drop), see `items.md` first.

## Contents

- Weapons
- Armour And Slot Types
- Recipe: A Ring
- Recipe: Stat Gear (Modifiers)
- Recipe: Gear That Grants Commands While Worn
- Damage Types
- Resistance, Protection, Vulnerability
- Materials
- Values And Prices
- Daemon-Generated Gear
- Common Mistakes

## Weapons

> Docs: `/doc/build/weapon`, `/doc/build/weapon.list`; source `/std/weapon.c`; examples `/doc/examples/weapon/`

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

Core setters:

- `set_class(n)` — weapon class; damage per hit is random in `0..n-1`. Recommended range 1–20; the std library refuses classes above 20 and zero-weight weapons. Weight must be at least 1.
- `set_type(type)` — weapon subtype. Only `"twohanded"` changes core behavior; guilds check common types (`axe`, `knife`, `sword`, `longsword`, `mace`, `whip`, `club`, `flail`, `broadsword`, `scimitar`, `rapier`, `shortsword`), so pick a real one.
- `set_damage_type(type)` — string or **array** of strings; with an array each hit picks one at random. Must be from the damage-type list below. Do not confuse with `set_type()`.
- `set_read(str)` — text returned when the weapon is read.

Special behavior:

- `set_hit_func(ob)` — `ob` (object or function pointer) gets `weapon_hit(target)` called on every strike. The **return value is added to the weapon's wc for that hit**; returning the string `"miss"` makes the weapon miss. Inside the callback use `query_wield()` for the wielder, not `this_player()`.
- `un_wield(int dead)` — define in the weapon; called when wielding stops. `dead` is true when unwielding because the wielder died.
- `weapon_kill(target, dtype, weapon_name, target_name, target_gender)` and `damage_noise(target, damage, dtype)` — define these for custom kill/damage messages. Message the hitter, the target, the target's party, and the room. For `weapon_kill` vs NPCs `target` is usually 0; find the hitter via `query_wield()`.
- `not_suitable_as_weapon()` — return 1 to make attacks with this weapon deal no damage (decorative/story weapons).

Breakage:

- Default break chance is 1/1000 per use; `set_break_chance(n)` sets it to n/1000. Doc guideline: iron sword ≈ 5, iron club ≈ 2, wooden sword ≈ 40, wooden staff ≈ 80, glass sword ≈ 150. A chance of 100 breaks on average every 10th hit — keep it low.
- `set_break_msg(str)` — message to the player on break; `set_broken_desc(str)` — description of the broken weapon.

Hooks: weapons fire `__wield` and `__remove` (both blocking: a callback returning 1 blocks the action silently — producing a message is your job) and `__weapon_hit`, `__damage_type`. See `hooks.md`.

Calibration (from `/doc/build/weapon.list` — class/value/weight): knife 5/8/1, curved knife 7/15/1, hand axe 9/25/2, long sword 10/700/3, unique frost-sword tier 15–17/2000/3, dragonslayer tier 18/5000/4 (must be very hard to get), wc 20 reserved for rare magic weapons worth 10000+. Anything wc 18+ should require killing a specific hard-to-reach monster.

## Armour And Slot Types

> Docs: `/doc/build/armour`, `/doc/build/armour.list`, `/doc/build/GUIDELINES` (rule 8); caps table `/std/include/armour.h`

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
    add_property("steel");
}
```

- `set_ac(n)`: a random value up to `n` is subtracted from each hit. The AC of all worn pieces sums.
- `set_type(t)` sets the **slot**. Only one piece per type can be worn. Default type is `"armour"`.
- `set_light(n)` makes the armour shine like a lamp of strength n.

The complete documented slot table, with each slot's max AC:

| type | max ac | body slot |
|---|---|---|
| `armour` | 4 | torso/body (default) |
| `legging` | 2 | legs |
| `helmet` | 1 | head |
| `shield` | 1 | carried shield |
| `cloak` | 1 | back/shoulders |
| `glove` | 1 | hands |
| `boot` | 1 | feet |
| `ring` | 1 | finger |
| `amulet` | 1 | neck |
| `tabard` | 0 | over-armour (decorative) |
| `belt` | 0 | waist (decorative) |
| `magic` | 12 | special; arch-tier approval only |

`tabard` and `belt` are valid wearable slots that give **no protection** — use them for decorative or effect-only gear. An arbitrary type string is accepted but gives no protection and no slot conflict; strongly avoid. Wearing refuses armour whose AC exceeds its type cap.

Calibration (from `/doc/build/armour.list` — ac/value/weight/type): helmet 1/75/1/helmet, ring of protection 1/200/1/ring, leather jacket 1/30/1/armour, leather armour 2/100/3/armour, chainmail 3/300/4/armour, plate 4/700/6/armour. No armour can be weightless. If weight is one higher than the table, cut price by 1/3; one lower, double it.

Wear/remove messaging — the std armour honours these **properties** (with `#s` replaced by the short description, and `format_message()` codes like `$N`/`$r`/`$o` for the wearer — codes in `SKILL.md` § Messaging):

```c
add_property("wear_msg", "You slip #s onto your finger and feel steadier.\n");
add_property("wear_other_msg", "$N slips #s onto $p finger.\n");
add_property("remove_msg", "You pull #s off your finger.\n");
add_property("remove_other_msg", "$N removes #s.\n");
```

To force-wear from code, call `wear()` in the armour (`wear(1)` for silent); same for `remove()`.

Hooks: armour fires `__wear` and `__remove`; both blocking — a callback returning 1 blocks silently, messaging is your responsibility. See `hooks.md`.

## Recipe: A Ring

A ring is ordinary armour: follow the armour pattern above with `set_type("ring")` (max AC 1, one worn at a time), `set_ac(1)` (or 0 for effect-only rings), `set_weight(1)`, a value near the calibration table (ring of protection: 200), and a material property matching the description (e.g. `add_property(({ "metal", "copper" }))`). Amulets (`set_type("amulet")`), cloaks (`"cloak"`), boots, gloves, and belts follow the identical pattern with their own type and AC cap.

## Recipe: Stat Gear (Modifiers)

> Docs: `/doc/build/modifiers`; example `/doc/examples/modifier/`

Modifiers are temporary additions/subtractions to a stat, skill, or property. They are never saved — they vanish on quit — which makes them ideal for equipment effects.

The four player stats are the properties `"str"`, `"int"`, `"con"`, `"dex"`. Any property/skill name works as a target, e.g. `"prot_magic"`, `"magic_resistance"`, `"carry_modifier"`.

For gear, use the built-in wrappers — the modifier is applied on wield/wear and removed on unwield/remove automatically, with an auto-generated id:

```c
/* in a weapon */  set_wield_modifier("str", 2, 4);
/* in armour  */   set_wear_modifier("dex", 1, 5);
```

For non-gear effects (spells, studying a tome, potions), use the living's modifier API directly:

```c
void set_modifier(string id, string prop, string class, int value, int maxvalue)
void add_modifier(string id, int value)     /* adjust an existing modifier */
mixed query_modifier(string id)             /* ({ prop, class, value, mv }) */
void remove_modifier(string id)
int  query_tmp_prop(string prop)            /* current total across modifiers */

/* e.g. a study-once tome: */
this_player()->set_modifier("tome:int", "int", "special", 3, 6);
```

Classes and stacking rules (the balance model):

- Three classes: `guild` (guild spells/specials), `item` (bonuses while equipment is used — what the wield/wear wrappers use), `special` (short-lived boosts).
- Totals across classes are cumulative. **Within** a class, values sum but are capped by the **lowest maxvalue** present in that class (negative values always apply). So a sword (+3, mv 6) plus a helmet (+2, mv 4) gives +4, not +5, until the helmet is removed.
- Hard rule: `value + maxvalue <= 10`, and that ceiling is for rare cases. Any `item`-class modifier with `value + maxvalue > 4` must be on a unique.

Think of `maxvalue` as "the cap my object imposes on its whole class": generous maxvalue = plays well with other gear; low maxvalue = strong alone but suppresses stacking.

## Recipe: Gear That Grants Commands While Worn

Give the wearer a command path (a directory of command files — see `commands.md`) in the `__wear` hook and remove it in `__remove`. Armour `__wear`/`__remove` are blocking hooks: return 0 to allow the action.

```c
inherit AREA_I_ARMOUR;

static void create() {
    ::create();
    /* ... normal ring/amulet setup ... */
    add_hook("__wear", store_fp("on_wear"));
    add_hook("__remove", store_fp("on_remove"));
    add_hook("__move", store_fp("on_move"));
    add_hook("__destroy", store_fp("on_destroy"));
}

static int on_wear(int silent) {
    object ply;
    if (objectp(ply = environment()) && living(ply)) {
        ply->add_cmdpath(AREA_DIR_COM_RING);
    }
    return 0;                       /* 0 = allow the wear */
}

static int on_remove(int silent) {
    object ply;
    if (objectp(ply = environment()) && living(ply)) {
        ply->remove_cmdpath(AREA_DIR_COM_RING);
    }
    return 0;
}
```

A worn item can also leave its holder without a remove event (drop, give, sale, destruction), so revoke in `__move`/`__destroy` too: implement `on_move(from, to)` revoking the cmdpath from `from`, and route `on_destroy` through it — the carried-item pattern in `commands.md`, minus the grant-on-`to` branch (granting stays in `on_wear`).

For a single verb, skip the command path and use `add_trigger("verb", store_fp("do_verb"))` on the item — the trigger only works while the item is present, so no revocation is needed. Gate the trigger's handler on `query_worn()` if it must require wearing. See `commands.md` for command files, `CMD_ACCESS.c`, and the carried-(not-worn)-item variant of this pattern.

## Damage Types

> Docs: `/doc/build/damage`

The canonical list (enforced — the message daemon rejects others, invalid types create system logs): `acid`, `bite`, `blunt`, `chop`, `claw`, `cold`, `drain`, `electricity`, `fire`, `impact`, `magic`, `pierce`, `poison`, `slash`.

- A lowercase type affects a single hit location on the target.
- A capitalized variant (`Fire`, `Cold`, ...) hits **all hit locations on that one target** — it wears out, burns, freezes, or shatters equipment across the body before damage reduction. Dragonbreath is the canonical example. Capitalized types still check the lowercase `resist_`/`prot_` properties.
- `Drain` passes through everything, cannot be protected against, and must never be dealt by players — reserve it for monsters/effects where the damage must go through. Drain never damages items.
- `magic` should not be used on its own; it means the `magic_resistance` skill helps against the attack.
- `blunt` is for normal weapons (clubs, maces); `impact` is for supernatural effects (vacuum, meteors, falls).

Where damage types appear:

```c
weapon->set_damage_type("fire");                       /* weapons */
monster->set_spell_dam_type(({ "fire", "impact" }));   /* monster spells; array = random pick per hit */
who->hit_player(60, "Cold", hitter);                   /* direct damage: (damage, type, hitter) */
```

## Resistance, Protection, Vulnerability

> Docs: `/doc/build/resistance`, `/doc/build/protection`, tail of `/doc/build/damage`

Per damage type, three property families on **livings**:

- `resist_<type>` — percent chance that damage of that type is reduced to 1/3 (integer division).
- `prot_<type>` — percent chance that damage of that type is prevented entirely.
- `vuln_<type>` — vulnerability: damage is **doubled before armour counts**.

Umbrella properties expand to groups:

- `resist_normal` → bite, blunt, chop, claw, pierce, slash. `resist_special` → acid, cold, electricity, fire, impact, magic. `resistance` → both groups. (`prot_normal` / `prot_special` / `protected` are the parallel prot set.)

On monsters: `add_skill("resist_fire", 50)` is permanent, `add_tmp_prop("resist_fire", 50)` is temporary — practically the same for NPCs, different for players (tmp_props are not saved).

On **items** the same properties control wear-out instead: `resist_<type>` cuts item damage to 1/3; `prot_<type>` at even value 1 means the item takes **no** damage of that type; `resistant` = resist everything; `protected` = immune to all wear; `artifact` sets `magic` + `protected` (never wears out); `magic` items effectively resist everything.

Keep values modest and consistent with nearby gear.

## Materials

> Docs: `/doc/build/material`, `/doc/build/materials`

Materials are properties (`add_property("steel")`) that drive how items wear out under damage types. Set one consistent with the description.

- `metal` (and its kinds `adamantium`, `bronze`, `copper`, `electrum`, `eog`, `gold`, `iron`, `mithril`, `silver`, `steel`): extra damage from acid; less from slash, claw, pierce. Noble metals conventionally get `resist_acid`/`prot_acid`; `iron` pairs with a `rust` property for flavour.
- `cloth` (kinds `cotton`, `fur`, `silk`, `wool`): extra damage from acid, bite, pierce, claw, slash, and especially chop. Cotton and silk set `flammable`.
- `leather`, `wood`, `paper`: same slash/chop weakness family; wood and paper set `flammable`, paper takes extreme fire damage.
- `stone`: takes less from bite, claw, fire, pierce, slash. `ice`: extra from fire and chop. `fire`: extra from cold. `bone`, `obsidian`, `liquid` also exist.
- `glass`, `crystal`, `ceramic`: set `fragile` by default (destroyed when dropped; extra blunt/impact/pierce damage) — removable manually.
- Meta-properties: `flammable` (extreme fire damage; area Fire likely destroys it), `fragile`, `afire` (also sets `fire` + `warm`), `cold`, `warm`, `poison` (no effect by itself; effects should check `no_poison`).

## Values And Prices

> Docs: `/doc/build/prices`, `/doc/build/GUIDELINES`

No shop pays more than 1000 coins. Items should never be worth more than 5000 — an item worth 5000 is among the best in the game. If you feel an item deserves more, it is too good and should not exist. Price gear consistently with the `weapon.list`/`armour.list` tiers above.

## Daemon-Generated Gear

Modern areas centralize randomized gear in quality-driven factory daemons (naming convention and API design in `daemons.md`) that compute type/class/AC/weight/value from a quality score (typically 0–100, above 100 adding enchantments); the equipping form chooses sensibly, e.g. a one-handed weapon when a shield is worn. Use the area's gear daemons for random loot and standard NPC equipment; hand-code unique items only when fixed behavior and descriptions matter.

## Common Mistakes

- Weapon class above 20, zero weight, or armour AC above its slot cap — refused by the std library or at wear time.
- Setting a nonstandard armour type: it silently gives no protection and no slot conflict. (Omitting `set_type` is safe — default is `armour`.)
- Invalid or unlisted damage types; using `Drain`/`drain` on anything a player can wield.
- Values inconsistent with the documented price tables.
- Modifiers exceeding `value + maxvalue <= 10`, or non-unique `item`-class modifiers above `value + maxvalue > 4`.
- Granting commands in `__wear` without revoking in `__remove`, `__move`, **and** `__destroy`.
- Duplicating randomized gear generation that an area daemon already centralizes.
