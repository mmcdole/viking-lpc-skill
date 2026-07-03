# Player Commands, Command Modules, And Command Paths

Use this when adding player-facing verbs beyond simple room/object triggers: command files, command directories granted to players, or guild/member command sets.

## Current References

- `/doc/build/cmd_module` (also linked as `/doc/build/i_command`)
- `/doc/build/cmdpath`
- `/std/cmd_module.c` (`I_COMMAND`)
- Nearby modern `com/` directories in the target area or an actively maintained guild.

## Choosing The Mechanism

- **`add_trigger` on a room or object**: the verb exists only while the player is near that object. No revocation needed — it disappears with the object. Use for room fixtures and simple item verbs. See `hooks.md`.
- **Command path (`add_cmdpath`)**: a whole directory of command files becomes available to a player. Use for command sets granted by a carried item or by guild membership. Requires symmetric revocation.
- **Guild command paths**: the guild master's `query_cmdpaths()` supplies paths to members automatically. See `guilds.md`.

## Command Modules

A command file inherits `I_COMMAND` and exposes:

```c
#include "/path/to/area/header.h"

inherit I_COMMAND;

string *query_action() {
    return ({ "polish" });
}

static int main(string arg) {
    object ply;

    if (!objectp(ply = this_player())) {
        return 0;
    }
    if (!stringp(arg)) {
        return notify_fail("Polish what?");
    }
    write("You polish it to a shine.\n");
    return 1;
}

string short_help() {
    return "polish <item> - polish something you carry";
}
```

- `query_action()` returns the verbs the file implements.
- `main(string arg)` is the entry point; keep it `static` — the command system invokes it through its own dispatch, and `static` prevents arbitrary external calls.
- `help()` / `short_help()` integrate with the help system; provide at least `short_help`.

## Command Paths

A command path is a directory of command `.c` files plus a `CMD_ACCESS.c` that controls access:

```c
int query_access(object who) {
    return objectp(who) && living(who);
}

string describe() {
    return "area tools";
}
```

Every `.c` file in the directory becomes a command for players granted the path. Gate `query_access` on real membership or possession checks when the commands should be restricted — an unconditional `return 1` makes the whole set public to anyone holding the path.

Wizards manage paths at runtime with the `cmdpath` and `comd` commands (e.g. `comd update` after editing command files).

## Granting And Revoking From A Carried Item

The canonical pattern: grant in the item's move handler when it enters a living holder, revoke when it leaves, and route destruction through the same handler so teardown lives in one place:

```c
static void create() {
    ::create();
    add_hook("__move", store_fp("on_move"));
    add_hook("__destroy", store_fp("on_destroy"));
}

static void on_move(object from, object to) {
    if (objectp(from)) {
        from->remove_cmdpath(AREA_DIR_COM_TOOLS);
    }
    if (objectp(to) && living(to)) {
        to->add_cmdpath(AREA_DIR_COM_TOOLS);
    }
}

static void on_destroy() {
    on_move(environment(), 0);
}
```

This covers drop, give, sale, logout, and destruction with one teardown path. See `hooks.md` for the full cleanup discipline.

## Common Mistakes

- Granting a cmdpath from an item without removing it on move and destroy, leaving players with permanent commands.
- `CMD_ACCESS.query_access` returning 1 unconditionally when the command set was meant to be gated.
- Non-`static` `main`, allowing arbitrary external calls into the command.
- Using a command path where a simple `add_trigger` on the object would do.
- Editing command files and forgetting the runtime update step.
