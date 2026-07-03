# LPC Basics

Use this when the task is about LPC syntax, object model, inheritance, messaging, or explaining VikingMUD code.

Each section carries the distilled facts and names its doc page. Verify against the doc when you have access; otherwise this is sufficient.

## LPC Differences That Matter Here

> Docs: `/doc/concepts/lpc`, `/doc/concepts/objects`, `/doc/learn_lpc/INDEX`

- LPC is C-like, but objects are loaded/compiled by the driver and have no `main`.
- `create()` is the normal object setup hook.
- `reset(int flag)` is called on resets: `flag` is 0 on the first call (object load) and 1 on every later periodic reset. Many objects use it to refresh content.
- `object->function(args)` calls a function in another object.
- Common types: `int`, `string`, `object`, `mixed`, arrays with `*`, and mappings.
- There is garbage collection; do not manage memory manually.
- `sscanf` behavior differs from C; verify parsing patterns in local examples.

## Inheritance

> Docs: `/doc/build/inherit_tree`; constants in `/std/include/mudlib.h` and `/std/include/std.h`

Major inherit constants (define via `#include <mudlib.h>`, plus `<std.h>` for `DROP`/`GET`/room `EXITS` defines):

- `I_OBJECT`: generic base object.
- `I_ITEM`: carryable item.
- `I_ROOM`: room.
- `I_MONSTER`: monster/living NPC; `I_WMONSTER`, `I_RMONSTER`, `I_RWMONSTER`: wandering/random variants.
- `I_DAEMON`: daemon/service; `I_SAVE_DAEMON` for daemons with managed persistence.
- `I_WEAPON`, `I_ARMOUR`, `I_CONTAINER`, `I_FOOD`, `I_DRINK`, `I_TORCH`.
- `I_COMMAND`: player command module.
- `I_GUILD`: the leveling/advancement guild base.
- Also: `I_DOOR`, `I_KEY`, `I_MONEY`, `I_SHOP`, `I_CORPSE`, `I_TREASURE`, `I_REGISTRY`, and module inheritables such as `I_HOOK`.

Never hardcode mudlib paths (`inherit "/std/room";` fails QC); when an area-specific standard class exists, prefer it over raw `I_*`.

Other standard include files (per `/doc/build/header`, which documents these — not area headers): `<levels.h>` wizard levels, `<daemons.h>` extra daemons (postfixed `_D` vs `D_` in mudlib.h), `<colours.h>` colour codes, `<origin.h>` `ORIGIN_LOCAL`/`ORIGIN_CALL_OTHER` for security checks, `<limits.h>`, `<type.h>`, `<feelit.h>` feeling bits, `<door.h>`, `<room.h>`, `<armour.h>`.

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
inherit base I_DAEMON;
inherit util AREA_I_UTIL;

static void create() {
    base::create();
    util::create();
}
```

## Function Pointers

> Docs: `/doc/sfun/types/store_fp`, `/doc/sfun/types/call_fp`, `/doc/sfun/types/functionp`

Use `store_fp("function_name")` for callbacks used by hooks, triggers, item descriptions, and chats:

```c
add_trigger("repair", store_fp("do_repair"));
add_hook("__init", store_fp("check_player"));
add_item("statue", store_fp("query_statue_desc"));
```

`store_fp("func")` binds the function to `this_object()`; the two-argument form `store_fp(obj, "func")` binds another object. Function pointers here are plain arrays, so the literal `({ obj, "func" })` form is equivalent. MudOS/LDMud-style closures (`#'func`, `lambda`, `fp()`) do not exist in this mudlib — do not use them.

Callback return values matter. For descriptions, return a string. For triggers, return `1` when handled; use `notify_fail("Message.")` and return `0` when the command should fail. `notify_fail()` itself always returns 0, so `return notify_fail("Message.");` is the standard failure idiom.

## Messaging And format_message

> Docs: `/doc/sfun/string/format_message`

- `write(str)` — to `this_player()`; `say(str)` — to the room except `this_player()`; `tell_object(ob, str)` — to one living; `tell_room(room, str)` — to everyone in a room.
- `format_message(str, passive, active, self)` substitutes names and pronouns. `$` codes draw from the active party (default `this_player()`), `#` codes from the passive party. Letters: `N`/`n` name, `G` name genitive, `O`/`o` objective (him/her/it), `P`/`p` possessive (his/her/its), `R`/`r` pronoun (he/she/it); capitalized = capitalized output.

```c
format_message("$N smiles at #N and chuckles to $oself.", who)
/* "Auronthas smiles at Kniggit and chuckles to himself." */
```

Many std facilities run strings through it: monster chats and spell messages, drink messages, `add_neg` other-messages, armour `wear_msg` properties. It is not fast — don't use it in hot paths.

Never generate messages that fake game output or another player's speech (QC rule).

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

Prefer early returns to deeply nested logic. Validate `previous_object()`, `this_player()`, and cloned objects before using them. Use `objectp`, `stringp`, `intp`, `sizeof`, `living(ob)`, `clonep(ob)`.

## QC Ground Rules

> Docs: `/doc/build/GUIDELINES` — read it fully once; these recur in reviews

- No deadly traps; danger must be telegraphed.
- Don't send objects/monsters outside your own area unless harmless.
- No messages that impersonate the game or other players.
- Healing must cost ≥ 4 gp/hp on average and be limited per reset (see `items.md`).
- Setting is the distant past — no modern technology.
- Teleporting items: rare, predefined destinations, ≥ 50 sp per use, never to a player.
- Armour slot types and caps are fixed (see `combat-gear.md`).
- Nothing goes live without QC approval; indent properly; never hardcode mudlib paths.
