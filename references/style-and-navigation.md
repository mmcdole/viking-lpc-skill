# Style And Navigation

## Mudlib Layout

Current, useful roots:

- `/std/`: core inherits such as `object.c`, `item.c`, `room.c`, `monster.c`, `daemon.c`, `weapon.c`, `armour.c`, `container.c`, and modules under `/std/mod/`.
- `/std/include/`: standard headers. `mudlib.h` and `std.h` provide common `I_*` constants.
- `/doc/build/`: current builder docs for room, object, monster, item, weapon, armour, container, headers, logging, hooks, etc.
- `/doc/examples/`: current examples to compare against.
- `/doc/concepts/`: LPC and object model background.
- `/doc/lfun/`: callable object APIs by subsystem.
- `/doc/hooks/`: hook names and behavior.
- Modern complete areas: use as source material when current docs are not enough, then generalize the pattern to the target area.
- `/attic/doc/`: old docs. Read only when current docs/code do not answer the question, then verify against current code.

## Area Organization

Modern areas centralize paths and inherit names in a single header, built so the area is relocatable:

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
- Inherit through constants when a local header provides them.
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
- Match local indentation and brace style. Existing modern areas are not uniform.

## Current Documentation To Prefer

- Headers: `/doc/build/header`
- Inherit overview: `/doc/build/inherit_tree`
- Object basics: `/doc/build/object`
- Logging: `/doc/build/logging`
- Properties: `/doc/build/properties`
- Hooks: `/doc/hooks/hook`, `/doc/hooks/hooklist`
- LPC: `/doc/concepts/lpc`, `/doc/learn_lpc/`

## Documentation Culture

Well-maintained areas keep living documents; follow the convention when editing one:

- `CHANGES`: reverse-chronological, dated author entries with `-` bullets, often naming new header macros. Add an entry for substantive changes.
- `README`: onboarding for other wizards (what the area is, local tooling, where docs live).
- `QC.*` files: the quality-control review dialogue with resolutions marked inline. Read them — they encode what reviewers care about (headers over hardcoded paths, shared guard helpers, cleanup symmetry).
- `help/` man pages for area tooling; add one when adding a command.
- Quests are registered as data plus a grant call: quest metadata lives in a data file, and the object grants with `ply->set_quest(name)` guarded by `!ply->query_quests(name)` — see `/doc/build/quest` and `/doc/build/qpsystem`.

## Quality Checks

- Read `QC.Notes` or `QC.NOTES` in the area if present.
- Check area `CHANGES`, `README`, and `TODO` for intent.
- For runtime/compile problems, current logging docs point to `~/log/compile.err`, `~/log/runtime.err`, and domain logs. Use in-mud commands such as `errors`, `err`, `deb`, and `dbg` when available.
