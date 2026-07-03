# viking-lpc

An [agent skill](https://code.claude.com/docs/en/skills) for writing, reviewing, and modernizing VikingMUD-style LPC: rooms, NPCs and monsters, items, weapons and armour, daemons, guilds, player commands, hooks, and mudlib conventions.

The skill teaches current mudlib idioms — area headers with path macros, layered std classes, service daemons, hook lifecycle and cleanup, command paths — and points at the live `doc/` tree rather than legacy attic examples.

## Mudlib access

No local mudlib is required. The skill's reference files carry distilled API knowledge (signatures, enforced limits, hook catalog, canonical patterns) sufficient to write standard code on their own, and it adapts to whatever access you have:

- **Local checkout** — a directory containing `std/`, `doc/`, `secure/`, `d/`, `players/`: the agent reads docs and nearby code directly. Mudlib paths in the skill are written mud-absolute (`/doc/build/room`), matching what wizards see in-game; a checkout's root maps to `/`.
- **In-game access only** — you work as a wizard through ed/ftp: the agent asks you to paste specific files (your area header, one nearby example, a doc page) when verification matters.
- **No access** — the agent relies on the skill and flags anything it couldn't verify against live docs.

## Install

Copy this directory to either:

- `~/.claude/skills/viking-lpc/` — available in all projects, or
- `<project>/.claude/skills/viking-lpc/` — available in one project.

The agent loads `SKILL.md` automatically when LPC/VikingMUD work comes up, and pulls in the topic files under `references/` on demand.

## Layout

- `SKILL.md` — entry point: workflow, source priority, house style.
- `references/` — one file per topic: rooms, NPCs, items, combat gear, daemons, guilds, commands, hooks, quests, LPC basics, style and navigation.

Each reference section embeds the distilled API (signatures, enforced limits, calibration tables, recipes) locally and opens with a `> Docs:` line naming the mudlib page it was distilled from — so the agent can work with no mudlib access and validate when it has some.
