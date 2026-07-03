# Guilds

Use this when creating, extending, or reviewing a guild. Guilds are multi-object subsystems, not single files: expect a guild master, guild rooms, member commands, and often boards, shadows, and spells/skills.

Each section carries the distilled API and names its doc page. Verify against the doc when you have access; otherwise this is sufficient.

## Three Interfaces To Coordinate

> Docs: `/doc/build/guild` (guild-master callbacks — read it fully before starting), `/doc/lfun/daemons/guild_d`

Building a guild means implementing or calling three separate interfaces:

1. **Guild-master callbacks** (implemented in your guild-master object; all optional):
   - Combat/PK policy: `allowed_to_attack(attacker, target)` and `allowed_to_be_attacked(attacker, player)` — return 1 to allow without logging to `/log/ATTACK`; `playerkilling_allowed(attacker, target)` — return 1 to exempt from playerkill logging.
   - Member lifecycle: `member_enters(ply)` (login), `member_exits(ply)` (quit), `member_linkdies(ply)`, `member_reenters(ply)`, `member_present(member, other)`.
   - Presentation: `member_extra_look(ply)` (string appended to the player's look), `member_finger_info(name)` (guild-related info only), `skill_name(skill)` (human-readable skill names for this guild), `weapon_noise(hitter, target, dam, dtype)` (overrides standard combat noise; hitter's guild is consulted first, then target's, and defining it suppresses regular messaging).
   - Integration: `query_cmdpaths()` (string or array of command paths granted to members — see `commands.md`), `nchat_filter(who, arg)` (filter/replace member chat), `query_guild_aliases()` (mapping of alias name → array of member names; the guild id is prepended, and an alias literally named `"maintainers"` collapses to just the guild id).

2. **Player-object grant API** (called on the player object):
   - `set_guildlevel(newlevel)` — **must be called from the guild-master file**; returns the new level on success, the current value on failure.
   - `query_guildlevel()` — callable from anywhere.
   - `query_player_weapon(n)` — whatever it returns overrides the wielded weapon; never call `query_wield` on the player inside it (recursion). Returning 0 does nothing.

3. **The guild daemon `D_GUILD`** (`/secure/daemons/guild_d.c`):
   - `into_guild(who)` and `remove_from_guild(who)` handle join/leave plumbing; both return 1 on success — always route membership changes through the daemon.
   - Guild registration in the daemon is arch-level; coordinate with an arch to register or retune a guild (`help guildtune`).

Note: `I_GUILD` (`/std/guild.c`) is the leveling/advancement guild base — a distinct concept from the custom-guild master interface above; don't confuse them.

## Structure Of A Modern Guild

> Best source: actively maintained guilds under `/d/*/guild/` — generalize the structure, never copy names/paths/content

Existing guilds typically organize as an area: a guild header with path macros, guild rooms, a guild master, a `com/` command directory granted via `query_cmdpaths()`, boards for members, and member-facing objects. Guild shadows are common for per-member persistent behavior, and skills/spells integrate through `add_skill`/`set_spell` on members plus the `guild` modifier class for temporary boosts (see Modifiers in `combat-gear.md` — guild-class modifiers follow the same `value + maxvalue <= 10` rule).

Before writing anything, read one or two actively maintained guilds end to end and map their division of responsibilities; then reuse the structure, not the content.

## Common Mistakes

- Calling `set_guildlevel` from an object other than the guild-master file (it fails).
- Bypassing `D_GUILD` for membership changes.
- Confusing `I_GUILD` (the leveling/advancement base) with the custom-guild master interface.
- Granting guild command paths without gating them on membership in the path's access check.
- Recursive `query_player_weapon` implementations.
- Designing guild balance (levels, costs, specials) without checking how existing guilds tune the same numbers.
