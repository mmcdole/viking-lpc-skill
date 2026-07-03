# Style And Navigation

## Mudlib Layout

Current, useful roots:

- `/std/`: core inherits such as `object.c`, `item.c`, `room.c`, `monster.c`, `daemon.c`, `weapon.c`, `armour.c`, `container.c`, and modules under `/std/mod/`.
- `/std/include/`: standard headers. `mudlib.h` and `std.h` provide common `I_*` constants; see `/doc/build/header` for the full catalog of standard include files (`<levels.h>`, `<colours.h>`, `<origin.h>`, ...). Note: that doc covers **standard include files**, not area headers — the area-header convention below is a modern-area idiom.
- `/doc/build/`: current builder docs for room, object, monster, item, weapon, armour, container, logging, damage, materials, modifiers, doors, quests, etc. `/doc/build/GUIDELINES` holds the QC ground rules — read it once (distilled in `lpc-basics.md`).
- `/doc/examples/`: current examples to compare against (`room/`, `item/`, `weapon/`, `modifier/`, `unique/`, `autoload/`, `container/`, `guild/`, `boards/`).
- `/doc/concepts/`: LPC and object model background.
- `/doc/lfun/`: callable object APIs by subsystem (including `/doc/lfun/daemons/`).
- `/doc/sfun/`: driver/simulated efuns (`types/store_fp`, `string/format_message`, ...).
- `/doc/hooks/`: hook names and behavior.
- Modern complete areas: use as source material when current docs are not enough, then generalize the pattern to the target area.
- `/attic/doc/`: old docs. Read only when current docs/code do not answer the question, then verify against current code.

## Area Organization

Modern areas centralize paths and inherit names in a single header, built so the area is relocatable. This is a convention extracted from well-maintained areas, not a core doc:

- Use an include guard and pull in the mudlib headers the area needs.
- Define one root macro and compose every other path from it, never repeating a literal path:

```c
#ifndef AREA_H
#define AREA_H
#include <mudlib.h>

#define AREA_DIR         ("/path/to/area/")
#define AREA_DIR_OBJ     (AREA_DIR + "obj/")
#define AREA_DIR_DAEMONS (AREA_DIR + "daemons/")
#define AREA_C_TOKEN     (AREA_DIR_OBJ + "token")
#define AREA_D_LOG       (AREA_DIR_DAEMONS + "logd")
#endif
```

- Pick a short unique area prefix and use role prefixes consistently: `<PREFIX>_DIR_*` directories, `<PREFIX>_I_*` local inherits, `<PREFIX>_C_*` cloneables, `<PREFIX>_R_*` rooms, `<PREFIX>_D_*` daemons, `<PREFIX>_P_*` property strings.
- Keep all gameplay tuning (prices, levels, capacities, chances) as named constants in the header — no magic numbers in logic files. Convenience function-macros (e.g. a logging macro wrapping the log daemon) live here too.
- Sub-area headers include the parent header and add narrower constants under their own short prefix.

A typical area directory layout mirrors the mudlib, relative to the area root: `std/` (local base classes), `daemons/`, `obj/`, `rooms/`, `com/` (commands), `help/`, `etc/` (data files), `var/` (runtime/generated state, save files), `log/`, plus `README`, `CHANGES`, and QC notes at the top.

## Extracted Coding Conventions

- One include block at the top, usually the local area header.
- Inherit through constants when a local header provides them; never hardcode `/std/` paths (QC rule).
- Use named inherited labels for multiple inheritance when needed:

```c
inherit base I_DAEMON;
inherit util AREA_I_UTIL;

static void create() {
    base::create();
    util::create();
}
```

- Call `::create()` in normal single-inherit objects and rooms.
- Build descriptions with wrapped string concatenation aligned under the first string.
- Use path macros for clones and reset objects.
- Use daemon constants for shared services.
- Use hooks with `store_fp("function_name")`; keep handlers `static` unless external callers need them.
- Guard public helpers with `objectp`, `stringp`, and early returns.
- Name private object state with a leading underscore (`_faction`, `_rentals`) and initialize it in `create()`.
- Base classes may mark setters `nomask` and reject calls after setup completes; respect those timing contracts rather than working around them.

## Service-Oriented Area Conventions

- Keep service daemons focused: one daemon for persistent rentals/state, one for object discovery/location tracking, one for logging, one for filtering or statistics if needed.
- Wrap `log_file()` behind an area daemon when the area logs many related actions.
- Use `restore_object()` and `save_object()` only in daemons or objects that clearly own persistent state.
- Put player-facing commands under an area command directory and grant the command path from a carried item or room when appropriate.
- When an item adds hooks or command paths to a player, remove them in move/destroy cleanup.
- Match local indentation and brace style. Existing modern areas are not uniform, but unindented code fails QC.

## Documentation Culture

Well-maintained areas keep living documents; follow the convention when editing one:

- `CHANGES`: reverse-chronological, dated author entries with `-` bullets, often naming new header macros. Add an entry for substantive changes.
- `README`: onboarding for other wizards (what the area is, local tooling, where docs live).
- `QC.*` files: the quality-control review dialogue with resolutions marked inline. Read them — they encode what reviewers care about (headers over hardcoded paths, shared guard helpers, cleanup symmetry).
- `help/` man pages for area tooling; add one when adding a command.

## Quality Checks

- Read `QC.Notes` or `QC.NOTES` in the area if present.
- Check area `CHANGES`, `README`, and `TODO` for intent.
- Check the change against `/doc/build/GUIDELINES` (distilled in `lpc-basics.md`).
- For runtime/compile problems, current logging docs point to `~/log/compile.err`, `~/log/runtime.err`, and domain logs. Use in-mud commands such as `errors`, `err`, `deb`, and `dbg` when available.
