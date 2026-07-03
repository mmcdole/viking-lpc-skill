---
name: viking-lpc
description: Use when writing, reviewing, modernizing, or explaining VikingMUD-style LPC code — rooms, NPCs and monsters, items, weapons and armour, daemons, guilds, quests, player commands, hooks, triggers, area headers, and mudlib conventions.
---

# VikingMUD LPC

This file carries what every LPC task needs: access modes, workflow, the LPC essentials, coding conventions, and the QC ground rules. Task-specific APIs live under `references/` — load the file the task calls for (see Reference Index).

## Mudlib Access

All mudlib paths in this skill are absolute from the mud's root (`/`), the way wizards see them in-game: `/std/`, `/doc/`, `/d/`, `/players/`. Adapt to whichever access mode applies:

1. **Local checkout**: a directory whose root contains `std/`, `doc/`, `secure/`, `d/`, and `players/` (often a directory named `lib/`). Mud-absolute paths map directly onto that root. Read docs and nearby code freely.
2. **In-game access only**: the user is a wizard working inside the mud (ed, ftp). You cannot browse mudlib files yourself — ask the user to paste specific, targeted files: the area header, one nearby example, or a named doc page. Keep requests few and precise.
3. **No access**: rely on the reference files in this skill; they carry enough distilled API to write correct standard code. Flag anything you could not verify against live docs so the user can check in-game.

Never block on reading docs. The references here are sufficient for standard work; the mudlib docs are for verification and edge cases.

## Workflow

1. Locate the target area's header and path macros, or ask the user to paste the header (`references/areas.md` covers header conventions).
2. Inspect the inherited std class and one or two similar local examples when available.
3. Load the reference file matching the task from the index below.
4. Write small, idiomatic LPC that calls inherited setup (`::create()`) and uses header macros and existing constants instead of hardcoded paths.
5. If the guidance is ambiguous, inspect a modern area with similar behavior and generalize the pattern; never copy its names, paths, macros, or content into unrelated work. Treat `/attic/doc` as historical context only; do not copy its APIs without checking current code/docs.
6. When mudlib access exists, verify distilled facts against the current docs — `references/mudlib-map.md` maps the doc tree.
7. After editing, verify by reading the final file and, when possible, checking compile/runtime logs (`~/log/compile.err`, `~/log/runtime.err`, domain logs) or in-mud commands (`errors`, `err`, `deb`, `dbg`).

## Reference Index

- Rooms, exits, doors, boards, shops: `references/rooms.md`
- NPCs/monsters, chats, responses, spells, wandering: `references/npcs.md`
- Items, containers, food, drink, light sources, auto-load: `references/items.md`
- Weapons, armour, slot types (ring/amulet/cloak/...), damage types, materials, resistances, stat gear and modifiers, command-granting gear: `references/combat-gear.md`
- Daemons, services, registries, persistence, area bootstrap, logging: `references/daemons.md`
- Guilds: `references/guilds.md`
- Player commands, command modules, command paths: `references/commands.md`
- Hooks, triggers, callbacks, and cleanup: `references/hooks.md`
- Quests, miniquests, questpoints, reward calibration: `references/quests.md`
- Area headers, area directory layout, README/CHANGES/QC documentation culture: `references/areas.md`
- Mudlib doc tree and std roots, for verification when access exists: `references/mudlib-map.md`

Reference sections carry distilled API facts locally and open with a `> Docs:` pointer naming the mudlib page they came from — use the local facts to write code, and the pointer to validate when mudlib access exists.

## LPC Essentials

### Language Notes

> Docs: `/doc/concepts/lpc`, `/doc/concepts/objects`

- LPC is C-like, but objects are loaded/compiled by the driver and have no `main`.
- `create()` is the normal object setup hook.
- `reset(int flag)` is called on resets: `flag` is 0 on the first call (object load) and 1 on every later periodic reset. Many objects use it to refresh content.
- `object->function(args)` calls a function in another object.
- Common types: `int`, `string`, `object`, `mixed`, arrays with `*`, and mappings.
- There is garbage collection; do not manage memory manually.
- `sscanf` behavior differs from C; verify parsing patterns in local examples.

### Inheritance

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

Other standard include files: `<levels.h>` wizard levels, `<daemons.h>` extra daemons (postfixed `_D` vs `D_` in mudlib.h), `<colours.h>` colour codes, `<origin.h>` `ORIGIN_LOCAL`/`ORIGIN_CALL_OTHER` for security checks, `<limits.h>`, `<type.h>`, `<feelit.h>` feeling bits, `<door.h>`, `<room.h>`, `<armour.h>`. The full catalog is in `/doc/build/header` (which documents these standard includes — not area headers; see `references/areas.md`).

### Create Pattern

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

Multiple inheritance — label each inherit and chain every `create()`:

```c
inherit base I_DAEMON;
inherit util AREA_I_UTIL;

static void create() {
    base::create();
    util::create();
}
```

### Function Pointers

> Docs: `/doc/sfun/types/store_fp`, `/doc/sfun/types/call_fp`, `/doc/sfun/types/functionp`

Use `store_fp("function_name")` for callbacks used by hooks, triggers, item descriptions, and chats:

```c
add_trigger("repair", store_fp("do_repair"));
add_hook("__init", store_fp("check_player"));
add_item("statue", store_fp("query_statue_desc"));
```

`store_fp("func")` binds the function to `this_object()`; the two-argument form `store_fp(obj, "func")` binds another object. Function pointers here are plain arrays, so the literal `({ obj, "func" })` form is equivalent. MudOS/LDMud-style closures (`#'func`, `lambda`, `fp()`) do not exist in this mudlib — do not use them.

Callback return values matter. For descriptions, return a string. For triggers, return `1` when handled; use `notify_fail("Message.")` and return `0` when the command should fail. `notify_fail()` itself always returns 0, so `return notify_fail("Message.");` is the standard failure idiom.

### Messaging And format_message

> Docs: `/doc/sfun/string/format_message`

- `write(str)` — to `this_player()`; `say(str)` — to the room except `this_player()`; `tell_object(ob, str)` — to one living; `tell_room(room, str)` — to everyone in a room.
- `format_message(str, passive, active, self)` substitutes names and pronouns. `$` codes draw from the active party (default `this_player()`), `#` codes from the passive party. Letters: `N`/`n` name, `G` name genitive, `O`/`o` objective (him/her/it), `P`/`p` possessive (his/her/its), `R`/`r` pronoun (he/she/it); capitalized = capitalized output.

```c
format_message("$N smiles at #N and chuckles to $oself.", who)
/* "Auronthas smiles at Kniggit and chuckles to himself." */
```

Many std facilities run strings through it: monster chats and spell messages, drink messages, `add_neg` other-messages, armour `wear_msg` properties. It is not fast — don't use it in hot paths.

Never generate messages that fake game output or another player's speech (QC rule).

### Defensive Checks

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

## Coding Conventions

Conventions extracted from well-maintained modern areas and their QC review dialogues. Apply them to new code; when editing an existing file, match its local style.

- One include block at the top, usually the local area header. Include the header whenever one exists.
- Inherit through header or `I_*` constants; never hardcode `/std/` or `/d/` paths (QC rule).
- In `create()`, call `::create()` before local setup unless the inherited class clearly documents otherwise.
- Prefer `static void create()`; it is the std-library canon, though plain `void create()` is common in area code — match the local file.
- Build long descriptions with wrapped string concatenation aligned under the first string.
- Use path macros for clones and reset objects, and daemon constants for shared services.
- Use `store_fp("func")` for hooks, triggers, function descriptions, and chat callbacks; keep handlers `static` unless external callers need them.
- Guard public helpers with `objectp`, `stringp`, and early returns.
- Name private object state with a leading underscore (`_faction`, `_rentals`) and initialize it in `create()`.
- Base classes may mark setters `nomask` and reject calls after setup completes; respect those timing contracts rather than working around them.
- For hooks, distinguish notification hooks from blocking `__b...` hooks, and clean up hooks added to players or other external objects (`references/hooks.md`).
- Match local indentation and brace style. Existing modern areas are not uniform, but unindented code fails QC.
- Avoid copying old attic examples until current docs and live code confirm the pattern.

## QC Ground Rules

> Docs: `/doc/build/GUIDELINES` — read it fully once when access exists; these recur in reviews

- No deadly traps; danger must be telegraphed.
- Don't send objects/monsters outside your own area unless harmless.
- No messages that impersonate the game or other players.
- Healing must cost ≥ 4 gp/hp on average and be limited per reset (see `references/items.md`).
- Setting is the distant past — no modern technology.
- Teleporting items: rare, predefined destinations, ≥ 50 sp per use, never to a player.
- Armour slot types and caps are fixed (see `references/combat-gear.md`).
- Nothing goes live without QC approval; indent properly; never hardcode mudlib paths.
