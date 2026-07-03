# Mudlib Map

Use this when mudlib access exists (local checkout or in-game) and you need to locate current docs, examples, or std sources for verification.

## Roots

- `/std/`: core inherits such as `object.c`, `item.c`, `room.c`, `monster.c`, `daemon.c`, `weapon.c`, `armour.c`, `container.c`, and modules under `/std/mod/`.
- `/std/include/`: standard headers. `mudlib.h` and `std.h` provide the common `I_*` constants; `/doc/build/header` catalogs the standard include files (`<levels.h>`, `<colours.h>`, `<origin.h>`, ...). That doc covers standard include files, not area headers — area headers are a modern-area convention (see `areas.md`).
- `/doc/build/`: current builder docs (key pages listed below). `/doc/build/GUIDELINES` holds the QC ground rules (distilled in `SKILL.md`).
- `/doc/examples/`: current examples to compare against (`room/`, `item/`, `weapon/`, `modifier/`, `unique/`, `autoload/`, `container/`, `guild/`, `boards/`).
- `/doc/concepts/`: LPC and object-model background (`lpc`, `objects`, `oop`).
- `/doc/lfun/`: callable object APIs by subsystem (including `/doc/lfun/daemons/`).
- `/doc/sfun/`: driver/simulated efuns (`types/store_fp`, `string/format_message`, ...).
- `/doc/hooks/`: `hooks.about`, `hook`, `hooklist`, and per-hook pages.
- Modern complete areas: source material when current docs are not enough — generalize the pattern to the target area, never copy names, paths, or content.
- `/attic/doc/` (including `/attic/doc/oldexamples`): historical only. Read when current docs/code do not answer the question, then verify against current code before using anything from it.

## Key Build-Doc Pages

Under `/doc/build/`: `room`, `door`, `monster`, `monster_response`, `item`, `object`, `weapon`, `weapon.list`, `armour`, `armour.list`, `container`, `food`, `drinks`, `logging`, `guild`, `cmd_module`, `cmdpath`, `damage`, `material`, `resistance`, `protection`, `modifiers`, `prices`, `quest`, `registry`, `castle`, `GUIDELINES`.
