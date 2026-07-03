# LPC Basics

Use this when the task is about LPC syntax, object model, inheritance, or explaining VikingMUD code.

## Current References

- `/doc/concepts/lpc`
- `/doc/concepts/objects`
- `/doc/concepts/oop`
- `/doc/learn_lpc/INDEX` and lessons
- `/doc/build/inherit_tree`
- `/doc/build/object`
- `/std/object.c`
- `/std/include/mudlib.h`, `/std/include/std.h`

## LPC Differences That Matter Here

- LPC is C-like, but objects are loaded/compiled by the driver and have no `main`.
- `create()` is the normal object setup hook.
- `reset(int flag)` is called on resets: `flag` is 0 on the first call (object load) and 1 on every later periodic reset. Many objects use it to refresh content.
- `object->function(args)` calls a function in another object.
- Common types include `int`, `string`, `object`, `mixed`, arrays with `*`, and mappings.
- There is garbage collection; do not manage memory manually.
- `sscanf` behavior differs from C; verify parsing patterns in local examples.

## Inheritance

Current major inherit constants are defined in `/std/include/mudlib.h` (plus domain headers for area-local classes):

- `I_OBJECT`: generic base object.
- `I_ITEM`: carryable item.
- `I_ROOM`: room.
- `I_MONSTER`: monster/living NPC.
- `I_WMONSTER`, `I_RMONSTER`, `I_RWMONSTER`: wandering/random variants.
- `I_DAEMON`: daemon/service; `I_SAVE_DAEMON` for daemons with managed persistence.
- `I_WEAPON`, `I_ARMOUR`, `I_CONTAINER`, `I_FOOD`, `I_DRINK`, `I_TORCH`.
- `I_COMMAND`: player command module.
- `I_GUILD`: the leveling/advancement guild base.
- Also available: `I_DOOR`, `I_KEY`, `I_MONEY`, `I_SHOP`, `I_CORPSE`, `I_TREASURE`, `I_REGISTRY`, and module inheritables such as `I_HOOK`.

When there is an area-specific standard class, prefer it over raw `I_*`.

## Create Pattern

Single inheritance:

```c
#include "/path/to/area/header.h"

inherit AREA_I_ITEM;

static void create() {
    ::create();
    set_name("token");
    set_short("a stamped brass token");
    set_long("It is a small brass token stamped with the city seal.");
    add_id(({ "brass token", "city token" }));
    set_weight(1);
    set_value(10);
}
```

Multiple inheritance:

```c
#include "/path/to/area/header.h"

inherit base I_DAEMON;
inherit util AREA_I_UTIL;

static void create() {
    base::create();
    util::create();
}
```

## Function Pointers

Use `store_fp("function_name")` in this mudlib for callbacks used by hooks, triggers, item descriptions, and chats:

```c
add_trigger("repair", store_fp("do_repair"));
add_hook("__init", store_fp("check_player"));
add_item("statue", store_fp("query_statue_desc"));
```

`store_fp("func")` binds the function to `this_object()`; a two-argument form `store_fp(obj, "func")` binds another object. Function pointers here are plain arrays, so the literal `({ obj, "func" })` form is equivalent. MudOS/LDMud-style closures (`#'func`, `lambda`, `fp()`) do not exist in this mudlib — do not use them.

Callback return values matter. For descriptions, return a string. For triggers, return `1` when handled; use `notify_fail("Message.")` and return `0` when the command should fail. `notify_fail()` itself always returns 0, so `return notify_fail("Message.");` is the standard failure idiom.

## Defensive Checks

Follow modern local code:

```c
if (!objectp(ply)) {
    return 0;
}
if (!stringp(arg)) {
    notify_fail("Repair what?");
    return 0;
}
```

Prefer early returns to deeply nested logic. Validate `previous_object()`, `this_player()`, and cloned objects before using them.
