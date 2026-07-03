# Area Structure And Headers

Use this when creating a new area, writing or extending an area header, laying out area directories, or maintaining area documentation (README, CHANGES, QC files).

## Area Headers

> Convention extracted from well-maintained modern areas, not a core doc. Note: `/doc/build/header` documents the standard include files (`<mudlib.h>`, `<levels.h>`, ...), not area headers.

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
- When adding an object or daemon to an established area, add its constant to the header and match the existing naming and alignment.

## Directory Layout

A typical area directory mirrors the mudlib, relative to the area root: `std/` (local base classes), `daemons/`, `obj/`, `rooms/`, `com/` (player commands granted via command paths — see `commands.md`), `help/`, `etc/` (data files), `var/` (runtime/generated state, save files), `log/`, plus `README`, `CHANGES`, and QC notes at the top.

## Living Documents

Well-maintained areas keep living documents; follow the convention when editing one:

- `CHANGES`: reverse-chronological, dated author entries with `-` bullets, often naming new header macros. Add an entry for substantive changes.
- `README`: onboarding for other wizards (what the area is, local tooling, where docs live).
- `QC.*` files: the quality-control review dialogue with resolutions marked inline. Read them — they encode what reviewers care about (headers over hardcoded paths, shared guard helpers, cleanup symmetry).
- `help/` man pages for area tooling; add one when adding a command.

## Reviewing Changes In An Area

- Read `QC.Notes` or `QC.NOTES` in the area if present.
- Check area `CHANGES`, `README`, and `TODO` for intent.
- Check the change against `/doc/build/GUIDELINES` (distilled in `SKILL.md` § QC Ground Rules).
