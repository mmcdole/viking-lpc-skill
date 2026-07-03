# Rooms

Use this when creating or reviewing rooms.

## Current References

- `/doc/build/room`
- `/doc/build/object`
- `/doc/build/door`
- `/doc/build/properties`
- `/doc/examples/room/`
- Nearby modern rooms in the target area or a similar modern area, used to infer style rather than copied directly.

## Basic Pattern

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
    add_property(({ "outdoors" }));
    add_exit(AREA_R_SQUARE, "north");
    add_exit(AREA_R_GATE, "south");
    add_reset_object("watchman", AREA_C_WATCHMAN);
}
```

Match the local area. If the area uses a specialized room class, inherit that instead of raw `I_ROOM`.

## Room Setup Checklist

- Include the local header, not a pile of absolute paths.
- Inherit the most local room std class.
- Call `::create()` before room-specific setup.
- Set light if the default darkness is not intended.
- Set a short and long description.
- Add inspectable `add_item` entries for important nouns in the long description.
- Add properties such as `indoors`, `outdoors`, `water`, `no_magic` only when they match documented property usage.
- Add exits using path macros or relative paths.
- Spawn reset content with `add_reset_object(id, file, number, location)` when NPCs/items should return on reset.
- Use `add_trigger` for room commands and `notify_fail` for rejected commands.

## Exits

`add_exit(room, command, exitfunc, distance, affect_long, flags)` is documented in `/doc/build/room`.

Common simple form:

```c
add_exit(AREA_DIR_ROOMS + "side_street", "west");
```

Use `<room.h>` flags such as `RF_HIDDEN` only after checking current `/std/include/room.h` and local examples.

Generated or map-backed rooms often inherit a reusable terrain/template room and add exits:

```c
#include "/path/to/area/header.h"

inherit room AREA_I_TERRAIN_PLAIN;

void create() {
    room::create();
    add_exit(AREA_DIR_VAR + "12/9", "north", 0, 0, 0, 0);
}
```

For hand-written rooms, prefer readable constants and two-argument exits unless extra behavior is needed.

## Interactive Rooms

Use `add_trigger` and a small helper for commands:

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

## Boards And Shops

Bulletin boards inherit the board std class (`/std/boards/board.c`) and are configured with setters such as `set_headline`, `set_subject_length`, `set_notes_dir`, `set_archive_dir`, and `set_remove_time`; they also accept the standard item description functions. Read `/doc/build/boards/boards` before placing one, and keep note/archive directories inside the area's `var/` tree.

Shops are mostly internal machinery (`/std/shop.c`) with a tiny builder surface; the main constraint is economic — see `/doc/build/prices` (shops never pay more than 1000 coins).

## Common Mistakes

- Long descriptions mention objects that cannot be examined.
- Cloned NPCs/items are created manually in `create()` instead of using reset helpers.
- Hardcoded `/d/...` paths are used where the area header has macros.
- `add_item` callback returns `1`, causing `1` to print as the description.
- A room command returns `0` without `notify_fail`, producing vague player feedback.
