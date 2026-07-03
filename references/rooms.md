# Rooms

Use this when creating or reviewing rooms, doors, boards, and shops.

## Contents

- Basic Pattern
- Exits
- Reset Objects
- Items, Triggers, And Canned Refusals
- Dynamic Descriptions
- Doors
- Boards And Shops
- Content Rules That Apply To Rooms
- Common Mistakes

## Basic Pattern

> Docs: `/doc/build/room`; examples `/doc/examples/room/`

```c
#include "/path/to/area/header.h"

inherit AREA_I_ROOM;

void create() {
    ::create();
    set_light(1);
    set_short("A Narrow Market Street");
    set_long("You are on a narrow market street between tall timber houses. " +
             "Canvas awnings hang over the stalls, and the smell of smoke " +
             "and baked bread drifts through the crowd.");
    add_item(({ "street", "market street" }),
             "The cobbled street is worn smooth by years of traffic.");
    add_item(({ "awning", "awnings", "canvas awnings" }),
             "The faded awnings keep the worst of the rain from the stalls.");
    add_property("outdoors");
    add_exit(AREA_R_SQUARE, "north");
    add_exit(AREA_R_GATE, "south");
    add_reset_object("watchman", AREA_C_WATCHMAN);
}
```

Match the local area. If the area uses a specialized room class, inherit that instead of raw `I_ROOM`.

Checklist: include the local header; inherit the most local room class; call `::create()`; set light (rooms default to 0 = darkness — `set_light(1)` is normal); set short and long; `add_item` every noun the long description mentions; add documented properties only (`indoors`, `outdoors`, `water`, `no_magic`, `no_wander`, ... — the in-game property database via `help lookupp` is authoritative); add exits via header macros or relative paths; spawn reset content with `add_reset_object`.

## Exits

> Docs: `/doc/build/room` (`add_exit`, `change_exit`, `remove_exit`)

```c
add_exit(room, command, exitfunc, distance, affect_long, flags)
```

- `room` — destination file; relative paths like `"./room2"` work.
- `command` — the direction verb.
- `exitfunc` — optional function (name or pointer) called on exit; **return 1 to stop the move**. It receives the destination room string.
- `distance` — action-point cost, ≥1 (feature currently unused; keep ≤10).
- `affect_long` — extra line added to the room's long description for this exit. Caveat: it is applied permanently even if the exit is later removed — use a function pointer that checks the exit still exists if it must change.
- `flags` — from `<room.h>`; `RF_HIDDEN` makes the exit work but not show as obvious. Combine flags with `|`.

`remove_exit(direction)` removes one. `change_exit(direction, index, value)` edits one in place, with `<room.h>` indexes `EX_ROOM`, `EX_FUNC`, `EX_DIST`, `EX_FLAGS` (useful for toggling visibility). `query_commands()` returns the exit verbs.

Common simple form: `add_exit(AREA_DIR_ROOMS + "side_street", "west");`

Generated or map-backed rooms often inherit a reusable terrain/template room and add exits:

```c
inherit room AREA_I_TERRAIN_PLAIN;

void create() {
    room::create();
    add_exit(AREA_DIR_VAR + "12/9", "north", 0, 0, 0, 0);
}
```

For hand-written rooms, prefer readable constants and two-argument exits unless extra behavior is needed.

## Reset Objects

> Docs: `/doc/build/room` (`add_reset_object`)

```c
add_reset_object(id, file, number, location)
```

Clones `file` at create and each reset **if no object with `id` still exists anywhere** (not just in the room). `number` (default 1) clones several; `location` (default this room) places them elsewhere. `file` can also be a function pointer called with `id`, returning the object to place — the hook for daemon-generated content. `query_reset_object(id)` and `remove_reset_object(id)` manage the list.

Use this instead of hand-cloning NPCs/items in `create()`/`reset()`.

## Items, Triggers, And Canned Refusals

> Docs: `/doc/build/room` (`add_item`, `add_neg`), `/doc/build/object` (`add_item` forms, `add_trigger`)

`add_item` in rooms follows the shared object API — including the verb/action forms — see `items.md` for all four forms. A description function must return a string, not 1. Defining `int do_get(string arg)` in the room lets players take room items without a separate trigger.

Room commands use triggers:

```c
add_trigger("pull", store_fp("do_pull"));

int do_pull(string arg) {
    if (arg != "lever") {
        return notify_fail("Pull what?");
    }
    write("You pull the lever.\n");
    say(this_player()->query_name() + " pulls the lever.\n");
    return 1;
}
```

If an action depends on an object in player inventory, guard with `objectp(ply = this_player())`, validate `arg`, use `present(arg, ply)`, and keep error messages specific.

For simple canned refusals, `add_neg` beats a trigger:

```c
add_neg(({ "get", "take" }), ({ "torch", "torches" }), "You can't reach them.");
```

Third argument can be a function pointer instead. Argument matching: `0` matches only no-argument, `""` matches any argument. Optional fourth arg is the message others see, with `#v` = verb, `#s` = argument, plus `format_message()` codes. `query_neg()` / `remove_neg()` manage them.

## Dynamic Descriptions

> Docs: `/doc/build/room` (`add_my_desc`)

`add_my_desc(desc, obj)` appends a temporary line to the room's long description, remembered per contributing object — it disappears when `obj` is destructed (always pass `obj` explicitly). `change_my_desc(desc, obj)` updates it; `remove_my_desc(obj)` removes all from that object. Use a function pointer for descriptions that must vary.

## Doors

> Docs: `/doc/build/door`; include `<door.h>` for message constants

`add_door(...)` (only in `I_ROOM`) creates a door object on an exit. The most useful arguments, in order: `file`, `verb`, `func`, `mp` (same as `add_exit`'s first four), then `ddesc` (affect-long), `dtype` (e.g. `"door"`, `"gate"`, `"portal"`), `dstatus` (open by default?), `ltype` (lock type, not implemented), `lstatus` (locked by default?), `key` (keycode; 0 = no lock), `lside` (1 = this side, -1 = other side, 0 = both), then message array, long desc, and function mapping. Arguments passed as 0 keep defaults. It returns the door object; configure via that:

```c
object door;

door = add_door(AREA_R_CELL, "north", 0, 1);
door->set_door_type("gate");
door->set_keycode("celldoor42");   /* a key object matching this code locks/unlocks */
door->set_def_open(0);
door->set_def_lock(1);
door->set_lock_side(1);
door->add_door_id(({ "gate", "iron gate" }));
door->set_long("A rusted iron gate.");
door->set_msgs(OPEN_MSG, FAIL_TO_PLAYER, "The gate will not budge.\n");
```

- `set_msgs(msg_type, msg_dest, msg)` with types `OPEN_MSG`, `CLOSE_MSG`, `LOCK_MSG`, `UNLOCK_MSG`, `PASS_MSG`, `LONG_MSG` and destinations `TO_PLAYER`, `TO_ROOM`, `TO_ALL`, `FAIL_TO_PLAYER`, `FAIL_TO_ROOM`; or pass a 5-element array to set a whole group. Substitutions: `#E` exit verb, `#T` door type, `#N` player name, `#D` open/closed, `#L` locked/unlocked.
- `add_function(verb, fp)` overrides a door verb (e.g. take over `"open"`); `set_functions(mapping)` sets them all; `remove_function(verb)`.
- Runtime control: `set_open(status, silent)`, `set_locked(status, silent)`; queries `query_open()`, `query_locked()`, `query_keycode()`, `query_other_side()` (`({ room, door })` of the opposite side, 0 if unloaded).
- Both sides need their own `add_door` with matching keycodes for a two-sided door.
- Doors fire blocking `__open`/`__close` hooks.

## Boards And Shops

> Docs: `/doc/build/boards/boards`; source `/std/boards/board.c`, `/std/shop.c`; prices `/doc/build/prices`

Bulletin boards inherit the board std class and are configured with setters such as `set_headline`, `set_subject_length`, `set_notes_dir`, `set_archive_dir`, and `set_remove_time`; they also accept the standard item description functions. Keep note/archive directories inside the area's `var/` tree.

Shops are mostly internal machinery with a tiny builder surface; the main constraint is economic — no shop pays more than 1000 coins.

## Content Rules That Apply To Rooms

> Docs: `/doc/build/GUIDELINES`

- No deadly traps: a player must never die because they didn't know what the next room does; telegraph danger.
- Never move objects/monsters to destinations outside your area unless deliberately harmless.
- Rewards for exploring; avoid mechanisms that cost experience for merely examining things.
- Setting: distant past — no modern objects (no airplanes; use a flying horse).
- Teleporting items/rooms: heavily restricted, predefined destinations only, ≥50 sp per use, never to a specific player.

## Common Mistakes

- Long descriptions mentioning objects that cannot be examined.
- Cloning NPCs/items manually in `create()` instead of `add_reset_object`.
- Hardcoded `/d/...` paths where the area header has macros.
- An `add_item` callback returning `1`, printing "1" as the description.
- A room command returning `0` without `notify_fail`, producing vague feedback.
- Forgetting the room defaults to darkness (`set_light(1)`).
- `affect_long` text lingering after its exit is removed.
- One-sided doors, or mismatched keycodes between the two sides.
