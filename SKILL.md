---
name: viking-lpc
description: Use when writing, reviewing, modernizing, or explaining VikingMUD-style LPC code — rooms, NPCs and monsters, items, weapons and armour, daemons, guilds, quests, player commands, hooks, triggers, area headers, and mudlib conventions.
---

# VikingMUD LPC

## Mudlib Access

All mudlib paths in this skill are absolute from the mud's root (`/`), the way wizards see them in-game: `/std/`, `/doc/`, `/d/`, `/players/`. Adapt to whichever access mode applies:

1. **Local checkout**: a directory whose root contains `/std/`, `/doc/`, `/secure/`, `d/`, and `players/` (often a directory named `lib/`). Mud-absolute paths map directly onto that root. Read docs and nearby code freely.
2. **In-game access only**: the user is a wizard working inside the mud (ed, ftp). You cannot browse mudlib files yourself — ask the user to paste specific, targeted files: the area header, one nearby example, or a named doc page. Keep requests few and precise.
3. **No access**: rely on the reference files in this skill; they carry enough distilled API to write correct standard code. Flag anything you could not verify against live docs so the user can check in-game.

Never block on reading docs. The references here are sufficient for standard work; the mudlib docs are for verification and edge cases.

## Source Priority

1. Read nearby code in the target area first (or ask the user for one representative file).
2. When accessible, use current docs under `/doc/build`, `/doc/examples`, `/doc/concepts`, `/doc/lfun`, `/doc/sfun`, and `/doc/hooks`.
3. Apply the practices in this skill: local headers, path/class/daemon macros, focused std layers, guarded callbacks, service daemons, and hook cleanup.
4. If the guidance is ambiguous, inspect a modern area with similar behavior and generalize the pattern; never copy its names, paths, macros, or content into unrelated work.
5. Treat `/attic/doc` and `/attic/doc/oldexamples` as historical context only; do not copy their APIs without checking current code/docs.

## Workflow

Before creating or changing LPC:

1. Locate the target domain/area header and path macros (or ask the user to paste the header).
2. Inspect the inherited std class and one or two similar local examples when available.
3. Load only the relevant reference file:
   - Rooms, exits, doors, boards, shops: `references/rooms.md`
   - NPCs/monsters, spells, chats, responses, wandering: `references/npcs.md`
   - Items, containers, food, drink, light sources, auto-load: `references/items.md`
   - Weapons, armour, slot types (ring/amulet/cloak/...), damage types, materials, resistances, stat gear and modifiers, command-granting gear: `references/combat-gear.md`
   - Daemons, services, registries, persistence, area bootstrap, logging: `references/daemons.md`
   - Guilds: `references/guilds.md`
   - Player commands, command modules, command paths: `references/commands.md`
   - Hooks, triggers, callbacks, and cleanup: `references/hooks.md`
   - Quests, miniquests, questpoints, reward calibration: `references/quests.md`
   - LPC syntax/object basics, messaging/format_message, QC ground rules: `references/lpc-basics.md`
   - Repo navigation, area/style conventions: `references/style-and-navigation.md`

   Reference sections carry distilled API facts locally and open with a `> Docs:` pointer naming the mudlib page they came from — use the local facts to write code, and the pointer to validate when mudlib access exists.
4. Prefer small, idiomatic LPC that calls inherited setup (`::create()`) and uses existing constants/macros instead of hardcoded paths.
5. After editing, verify by reading the final file and, when possible, checking mud compile/runtime logs or existing project validation commands.

## House Style

- Include the local area header when one exists.
- Use path, class, room, object, and daemon macros from headers instead of repeated string paths.
- In `create()`, call `::create()` before local setup unless the inherited class clearly documents otherwise.
- Prefer `static void create()`; it is the std-library canon, though plain `void create()` is common in area code — match the local file.
- Use `objectp`, `stringp`, `intp`, `sizeof`, and early returns for guard clauses.
- Use `store_fp("func")` for hooks, triggers, function descriptions, and chat callbacks.
- For hooks, distinguish notification hooks from blocking `__b...` hooks, and clean up hooks added to players or other external objects.
- Avoid copying old attic examples until current docs and live code confirm the pattern.

## Key Current Docs

- Build docs: `/doc/build/room`, `/doc/build/door`, `/doc/build/monster`, `/doc/build/monster_response`, `/doc/build/item`, `/doc/build/object`, `/doc/build/weapon`, `/doc/build/weapon.list`, `/doc/build/armour`, `/doc/build/armour.list`, `/doc/build/container`, `/doc/build/food`, `/doc/build/drinks`, `/doc/build/logging`, `/doc/build/guild`, `/doc/build/cmd_module`, `/doc/build/cmdpath`, `/doc/build/damage`, `/doc/build/material`, `/doc/build/resistance`, `/doc/build/protection`, `/doc/build/modifiers`, `/doc/build/prices`, `/doc/build/quest`, `/doc/build/registry`, `/doc/build/castle`, `/doc/build/GUIDELINES`
- `/doc/build/header` documents the standard include files (`<mudlib.h>`, `<levels.h>`, ...), not area headers — area headers are a modern-area convention (see `references/style-and-navigation.md`).
- Concepts: `/doc/concepts/lpc`, `/doc/concepts/objects`, `/doc/concepts/oop`
- Examples: `/doc/examples/room/`, `/doc/examples/item/`, `/doc/examples/weapon/`, `/doc/examples/modifier/`, `/doc/examples/unique/`, `/doc/examples/autoload/`
- Hooks: `/doc/hooks/hooks.about`, `/doc/hooks/hook`, `/doc/hooks/hooklist`, and specific hook pages
- Lfuns/sfuns: `/doc/lfun/` (object APIs by subsystem, including `/doc/lfun/daemons/`), `/doc/sfun/` (e.g. `types/store_fp`, `string/format_message`)
