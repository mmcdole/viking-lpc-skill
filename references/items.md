# Items, Containers, Food, Drink, And Light Sources

Use this when creating or reviewing carryable items, containers, food, drink, torches, and object descriptions. For weapons and armour, see `combat-gear.md`.

## Contents

- Basic Item Pattern
- Description And IDs
- Weight, Value, Get, Drop
- Containers
- Food
- Drink
- Light Sources
- Auto-Load Items
- Daemon-Generated Items
- Common Mistakes

## Basic Item Pattern

> Docs: `/doc/build/item`, `/doc/build/object`; source `/std/item.c`; examples `/doc/examples/item/`

```c
#include "/path/to/area/header.h"

inherit AREA_I_ITEM;

static void create() {
    ::create();
    set_name("brass token");
    set_short("a brass token");
    set_long("It is a small brass token stamped with the city seal.");
    add_id(({ "token", "city token" }));
    set_weight(1);
    set_value(10);
    add_property("metal");
}
```

For simple local objects, match the local inherit. Some modern objects inherit core specialized classes directly (e.g. `I_TORCH`) when that is the correct behavior.

## Description And IDs

> Docs: `/doc/build/object` (the full `add_item`/`add_id`/trigger API lives on `/std/object.c` and is shared by items, rooms, and NPCs)

- `set_name(n)` — base name; also sets short and a default long.
- `set_short(s)` — room/inventory display; also defaults long to short + ".\n".
- `set_long(l)` — examine text.
- `add_id(n)` / `remove_id(n)` — alternate player-visible names; string or array. Include the adjective+noun forms players will try from the short description.
- `set_read(str)` — text for `read <id>`. `set_info(str)` — hidden info.
- `id(s)` returns 1 for any added id or item-id.

`add_item` adds inspectable sub-parts, optionally with actions. Four documented forms:

```c
add_item(id, desc);                              /* look-only            */
add_item(({ ids1, desc1, ids2, desc2 }));        /* several at once      */
add_item(ids, desc, verbs, action);              /* verbs invoke action  */
add_item(id, desc, ({ verb1, act1, verb2, act2 }));  /* several actions  */
```

`desc` and `action` may be strings or function pointers (`store_fp("func")`). A string action is printed; a function action is called with the id as argument. A description **function must return a string** — returning 1 prints "1".

Triggers add room/inventory verbs scoped to the object's presence:

```c
add_trigger("rub", store_fp("do_rub"));

static int do_rub(string arg) {
    if (!stringp(arg) || !id(arg)) {
        return notify_fail("Rub what?");
    }
    write("The token warms briefly in your palm.\n");
    say(this_player()->query_name() + " rubs a brass token.\n");
    return 1;
}
```

`remove_trigger(verb)` removes one. Triggers disappear with the object — no cleanup needed.

## Weight, Value, Get, Drop

> Docs: `/doc/build/item`

- `set_weight(int|function)` / `query_weight()` — never change the weight of an item while a player carries it.
- `set_value(int|function)` / `query_value()`.
- `set_get(int|function)` — 1 = can be picked up (default), 0 = cannot. You can also define `get()` for side effects on pickup.
- `set_drop(int|function)` — note the inverted sense: **0 = can be dropped (default), 1 = cannot**. Or define `drop()`; worn/wielded items are removed/unwielded there.

Keep values consistent with `/doc/build/prices`: no shop pays more than 1000 coins; items should never be worth more than 5000.

## Containers

> Docs: `/doc/build/container`; source `/std/container.c`

- `set_max_weight(n)` / `query_max_weight()` — carrying capacity.
- `set_can_open(status)` — whether it can be opened/closed at all (initially closed); `set_open()` / `set_closed()`; `query_open()` / `is_open()`.
- Containers fire blocking `__open`/`__close` hooks; local container classes often add a `prevent_enter(obj)` guard — return 1 to refuse an insertion (write your own refusal message).
- The property `prevent_insert` on a **drink or item** stops players putting it inside containers.

## Food

> Docs: `/doc/build/food`; economy rules also in `/doc/build/GUIDELINES` (rule 6)

```c
inherit AREA_I_FOOD;   /* I_FOOD inherits I_ITEM */

static void create() {
    ::create();
    set_name("honey cake");
    set_short("a honey cake");
    add_id("cake");
    set_weight(1);
    set_value(90);          /* from the cost formula below */
    set_strength(20);       /* hp healed when eaten */
    set_doses(1);
    set_eater_mess("You eat the honey cake. It is sticky and delicious.\n");
}
```

- `set_strength(n)` — hp healed on eating (0 for pure flavour food). Typically should not exceed 50.
- `set_doses(i)` / `set_max_doses(i)` — how many times it can be eaten.
- `set_eater_mess(m)` — message to the eater.
- Food fires the blocking `__eat` hook.

**Healing economy rules (enforced by QC):** sold healing must cost `4*x + x*x/10` coins for `x` hp; at most 3000 hp of healing sold per reset per establishment, at most 200 hp to one customer per reset; healing items must weigh at least 1. Free healing items are only allowed if they heal at most 20, are destructed on use, and at most one is generated per room per reset.

## Drink

> Docs: `/doc/build/drinks`; source `/std/drink.c`

Same economy rules as food. Key setters:

- `set_heal(h)` — sets both `set_hp_heal(h)` and `set_sp_heal(h)`; or set them separately per drink.
- `set_strength(s)` — how drunk it makes the drinker; `set_soft_strength(s)` — how much it soaks. Rule of thumb: a drink healing 40 should have strength/soft_strength around 40.
- `set_full(n)` / `set_maxfull(n)` — drinks remaining / bottle capacity (both default 1). `query_value()` scales with `full/maxfull`.
- `set_drinker_mess(m)` / `set_drinking_mess(m)` — messages to drinker / room; support `format_message()` codes with `$` for the drinker (codes in `SKILL.md` § Messaging).
- `set_empty_container(e)` — name/id of the empty vessel.
- Drinks fire blocking `__drink` (and livings `__drink_alco` for alcohol). Add `prevent_insert` to forbid bagging.

## Light Sources

> Docs: `/doc/build/object` (`set_light`); source `/std/torch.c`

`set_light(n)` on any object makes it shine with strength n and correctly updates the `light` property. Burnable torches inherit `I_TORCH`. Rooms default to light 0 (darkness) — see `rooms.md`.

## Auto-Load Items

> Examples: `/doc/examples/autoload/`

Items that survive in a player's inventory across logout implement the auto-load pair:

```c
public string query_auto_load() {
    return sprintf("%s:%d,%d", source_file_name(), _level, _secret);
}

public void init_arg(string arg) {
    int level, secret;

    if (sscanf(arg, "%d,%d", level, secret) == 2) {
        set_level(level);
        set_secret(secret);
    }
}
```

`query_auto_load()` returns `"<file>:<args>"`; on login the item is cloned and `init_arg` receives the argument string. Encode only what must persist — transient state (charges, cooldowns) can be deliberately dropped. Validate the parsed values; never trust the saved string blindly.

## Daemon-Generated Items

Modern areas often create families of items through quality-driven factory daemons (naming convention and API design in `daemons.md`). Use daemon helpers for random loot and standardized gear; hand-code unique/story items when fixed behavior and descriptions matter. For gear specifics, see `combat-gear.md`.

## Common Mistakes

- Missing ids for words in the short description.
- An `add_item`/description function returning `1` instead of a string.
- Getting `set_drop` backwards (1 means *cannot* drop).
- Forgetting `::create()`.
- Healing food/drink priced below the `4*x + x*x/10` formula or exceeding the per-reset caps.
- Changing weight while carried; weightless items.
- Duplicating randomized item generation that an area daemon already centralizes.
