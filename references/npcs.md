# NPCs And Monsters

Use this when creating or reviewing NPCs, monsters, and wandering livings.

## Current References

- `/doc/build/monster`
- `/doc/build/walking_monster`
- `/doc/build/monster_response`
- `/doc/build/races`
- `/doc/build/damage`
- `/doc/hooks/`
- `/std/monster.c`, `/std/wmonster.c`, `/std/living.c`
- Nearby modern NPCs and local NPC std classes in the target area or a similar modern area.

## Basic Pattern

```c
#include "/path/to/area/header.h"

inherit AREA_I_CITIZEN;

static void create() {
    ::create();
    set_gender(1 + random(2));
    set_race("human");
    set_name("market guard");
    set_short("a market guard");
    set_long("This guard watches the market with practiced patience.");
    add_id(({ "guard", "watchman" }));
    set_faction("good");
    set_level(18 + random(3));
    set_hp(350 + random(100));
    set_wc(8);
    set_ac(10);
    add_skill("resist_slash", 15);
    load_chat(4, ({
        "*say Keep moving.",
        "*peer",
    }));
    load_a_chat(8, ({
        "*yell Guards!",
        store_fp("warn_attacker"),
    }));
}

static void warn_attacker(object foe) {
    if (!objectp(foe)) {
        return;
    }
    command("say You should have walked away, " + foe->query_name() + ".");
}
```

Use the local area std class when available. Modern NPCs often inherit area-specific base classes instead of raw `I_MONSTER`.

Some modern NPC bases defer behavior wiring to a post-create step (a `create_done()`-style callback scheduled with `call_out(..., 0)`) and reject configuration setters after setup completes. Check the base class before adding hooks in `create()`: configure in `create()`, and let the base wire behavior afterward if it does so.

## Setup Checklist

- Include the local header.
- Inherit the local NPC base or `I_MONSTER`.
- Call `::create()`.
- Set name, short, long, ids, race, gender where appropriate.
- Set level before overriding hp/sp/ac/wc, because `set_level` initializes several combat values.
- Use `set_faction`, alignment, skills, resistances, money, and inventory consistently with nearby NPCs.
- Use `load_chat` and `load_a_chat` sparingly; high chances become noisy.
- Use daemon helpers for generated equipment when the area provides them.
- Add hooks only when needed and point them at small guarded functions.

## Equipment

Current docs describe `add_eq(file)` for basic monsters. Modern areas often use daemon helpers:

```c
if (!clonep(this_object())) {
    return;
}
AREA_D_WEAPON->add_weapon(75);
AREA_D_ARMOUR->add_armour(50);
```

Follow the local pattern. If using `clonep`, avoid giving prototype/master objects live inventory.

## Hooks And Reactions

NPCs often use hooks:

```c
add_hook("__init", store_fp("check_reward"));
add_hook("__kill", store_fp("claim_reward"));
add_hook("__wander_done", store_fp("calm_down"));
```

Read `/doc/hooks/hooklist` and the specific hook doc before adding a new hook. Use exact hook names; bad hooks are logged.

## Combat And Chat

`load_chat(chance, messages)` applies outside combat. `load_a_chat(chance, messages)` applies in combat. Message entries can be commands prefixed with `*`, target commands prefixed with `!`, plain strings, or function pointers depending on the documented API.

Keep spell and damage types aligned with `/doc/build/damage`; invalid damage types create system logs. See `combat-gear.md` for the damage-type list and the single-location vs whole-body (capitalized) distinction.

For boss-type NPCs, derive hit points from intended fight design (raid size × expected damage per second × fight length) rather than picking a big number, and check `/doc/build/properties` and the area header for combat opt-out properties before hand-rolling special defenses.

## Common Mistakes

- Calling `set_hp` before `set_level`, then accidentally resetting values.
- Overusing chat chances.
- Forgetting `add_id` for natural player references.
- Giving inventory to the blueprint object instead of clones.
- Adding hooks without checking exact hook names.
