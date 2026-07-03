# Items

Use this when creating or reviewing carryable items, food, drink, containers, torches, and object descriptions. For weapons and armour, see `combat-gear.md`.

## Current References

- `/doc/build/item`
- `/doc/build/object`
- `/doc/build/container`
- `/doc/build/food`
- `/doc/build/drinks`
- `/doc/build/prices`
- `/doc/examples/item/`
- `/doc/examples/autoload/`
- `/std/item.c`, `/std/container.c`
- Nearby modern items and item-generation daemons in the target area or a similar modern area.

## Basic Item Pattern

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

For simple local objects, match the local inherit. Some modern objects inherit core specialized classes directly, e.g. `I_TORCH`, when that is the correct behavior.

## Description And IDs

- `set_name` gives the object its base name and default descriptions.
- `set_short` controls room/inventory display.
- `set_long` controls examine text.
- `add_id` adds alternate player-visible names.
- `add_item` can add inspectable sub-parts to an object.
- `set_read` is for text read with `read <id>`.
- `set_info` is for hidden info.

Use ids that players will naturally try, including adjective+noun forms from the short description.

## Weight, Value, Get, Drop

From `/doc/build/item`:

- `set_weight(int|function)` and `query_weight()`
- `set_value(int|function)` and `query_value()`
- `set_get(int|function)` for whether the item can be picked up.
- `set_drop(int|function)` for whether the item can be dropped.

Do not change an item weight while it is carried unless the local code already handles that safely.

Keep values consistent with `/doc/build/prices`: no shop pays more than 1000 coins, and items should never be worth more than 5000.

## Specialized Items

Choose the most specific inherit:

- Weapon: `I_WEAPON` or local weapon class — see `combat-gear.md`.
- Armour: `I_ARMOUR` or local armour class — see `combat-gear.md`.
- Container: `I_CONTAINER` or local container class.
- Food/drink: `I_FOOD`, `I_DRINK`.
- Light source: `I_TORCH`.

Read the matching `/doc/build/*` page and one local example before setting combat or consumption values.

## Auto-Load Items

Items that should survive in a player's inventory across logout implement the auto-load pair (see `/doc/examples/autoload/`):

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

Modern areas often create families of items through daemons. Keep that behavior centralized:

- `AREA_D_CONTAINER->create_container(quality)`
- `AREA_D_WEAPON->add_weapon(quality)`
- `AREA_D_ARMOUR->add_armour(quality)`

Use daemon helpers when creating random loot or standardized area gear. Hand-code unique/story items when fixed behavior and fixed descriptions matter.

## Interactive Items

Use triggers or `add_item` actions rather than raw ad hoc command wiring:

```c
add_trigger("rub", store_fp("do_rub"));

int do_rub(string arg) {
    if (arg != "token" && arg != "brass token") {
        return notify_fail("Rub what?");
    }
    write("The token warms briefly in your palm.\n");
    return 1;
}
```

For function descriptions, return a string, not `1`.

## Common Mistakes

- Missing ids for words in the short description.
- Using raw `I_ITEM` when an inherited specialized class already exists.
- Forgetting `::create()`.
- Setting unrealistic weight/value compared to nearby items.
- Duplicating randomized item generation that an area daemon already centralizes.
