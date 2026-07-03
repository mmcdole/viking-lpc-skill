# Guilds

Use this when creating, extending, or reviewing a guild. Guilds are multi-object subsystems, not single files: expect a guild master, guild rooms, member commands, and often boards, shadows, and spells/skills.

## Current References

- `/doc/build/guild` — the guild-master interface; read it fully before starting.
- `/doc/lfun/daemons/guild_d` — the central guild daemon API.
- `/std/guild.c` (`I_GUILD`) — the leveling/advancement guild base; a distinct concept from custom guilds.
- `/doc/build/modifiers` — the `guild` modifier class for guild spells/specials.
- Existing modern guilds under `/d/*/guild/` — the best source for structure; generalize, do not copy.

## Three Interfaces To Coordinate

Building a guild means implementing or calling three separate interfaces:

1. **Guild-master callbacks** (implemented in your guild-master object, per `/doc/build/guild`). All optional; the important ones:
   - Combat/PK policy: `allowed_to_attack`, `allowed_to_be_attacked`, `playerkilling_allowed`.
   - Member lifecycle: `member_enters`, `member_exits`, `member_linkdies`, `member_reenters`, `member_present`.
   - Presentation: `member_extra_look`, `member_finger_info`, `skill_name`, `weapon_noise`.
   - Integration: `query_cmdpaths()` (grants command paths to members — see `commands.md`), `nchat_filter`, `query_guild_aliases` (mail aliases).

2. **Player-object grant API** (called on the player object):
   - `set_guildlevel(newlevel)` — must be called from the guild-master file; returns the new level on success, the current value on failure.
   - `query_guildlevel()`.
   - `query_player_weapon(n)` — overrides the wielded weapon; the docs warn about recursion, read them before using it.

3. **The guild daemon `D_GUILD`** (`/secure/daemons/guild_d.c`):
   - `into_guild(player, id)` and `remove_from_guild(player)` handle join/leave plumbing — always route membership changes through the daemon.
   - Guild registration (`set_guild(...)` and the in-game guild-set editor) is arch-level; coordinate with an arch to register or retune a guild.

## Structure Of A Modern Guild

Existing guilds typically organize as an area: a guild header with path macros, guild rooms, a guild master, a `com/` command directory granted via `query_cmdpaths()`, boards for members, and member-facing objects. Guild shadows are common for per-member persistent behavior, and skills/spells integrate through `add_skill`/`set_spell` on members plus the `guild` modifier class for temporary boosts.

Before writing anything, read one or two actively maintained guilds end to end and map their division of responsibilities; then reuse the structure, not the content.

## Common Mistakes

- Calling `set_guildlevel` from an object other than the guild-master file (it fails).
- Bypassing `D_GUILD` for membership changes.
- Confusing `I_GUILD` (the leveling/advancement base) with the custom-guild master interface.
- Granting guild command paths without gating them on membership in the path's access check.
- Designing guild balance (levels, costs, specials) without checking how existing guilds tune the same numbers.
