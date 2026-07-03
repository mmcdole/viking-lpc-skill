# Player Commands, Command Modules, And Command Paths

Use this when adding player-facing verbs beyond simple room/object triggers: command files, command directories granted to players, or guild/member command sets.

Each section carries the distilled API and names its doc page. Verify against the doc when you have access; otherwise this is sufficient.

## Choosing The Mechanism

- **`add_trigger` on a room or object**: the verb exists only while the player is near that object. No revocation needed — it disappears with the object. Use for room fixtures and simple item verbs. See `hooks.md` and `items.md`.
- **Command path (`add_cmdpath`)**: a whole directory of command files becomes available to a player. Use for command sets granted by a carried/worn item or by guild membership. Requires symmetric revocation.
- **Guild command paths**: the guild master's `query_cmdpaths()` supplies paths to members automatically. See `guilds.md`.

## Command Modules

> Docs: `/doc/build/cmd_module` (also linked as `/doc/build/i_command`); source `/std/cmd_module.c`

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

- `query_action()` returns the verbs the file implements. If omitted, the verb defaults to the filename minus `.c` — the common case needs no `query_action` at all.
- `main(string arg)` is the entry point; `arg` is whatever the player typed after the verb. Keep it `static`: the player object calls `_main()` (defined in `I_COMMAND`), which dispatches to `main` — `static` prevents arbitrary external calls. This is the security model.
- `help()` and `short_help()` integrate with the help system. `help()` may return either a text string or a file path — a leading `/` means "more() this file to the player". Provide at least `short_help`.

## Command Paths

> Docs: `/doc/build/cmdpath`

A command path is a directory of command `.c` files plus a `CMD_ACCESS.c` that controls access:

```c
int query_access(object who) {
    return objectp(who) && living(who);
}

string describe() {
    return "areatools";   /* one word; used as the verbgroup name */
}
```

Every `.c` file in the directory becomes a command for players granted the path (non-command files like READMEs and help files may live there too). Gate `query_access` on real membership or possession checks when the commands should be restricted — an unconditional `return 1` makes the whole set public to anyone holding the path.

Runtime management: `cmdpath` views/adds/removes searched paths; `comd dump [path]` shows loaded paths/commands; `comd update <path>` refreshes the daemon's image. After editing a module you only need `comd update` when you (1) added or removed modules, (2) changed the verbs a module responds to, or (3) changed `short_help()`.

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

For gear that grants commands **only while worn/wielded**, grant in the `__wear`/`__wield` hook and revoke in `__remove` — but still revoke in `__move`/`__destroy` too, since a worn item can leave its holder without a remove event. Full recipe in `combat-gear.md`.

## Common Mistakes

- Granting a cmdpath from an item without removing it on move and destroy, leaving players with permanent commands.
- `CMD_ACCESS.query_access` returning 1 unconditionally when the command set was meant to be gated.
- Non-`static` `main`, allowing arbitrary external calls into the command.
- Using a command path where a simple `add_trigger` on the object would do.
- Editing command files and forgetting `comd update` after adding/removing modules or changing verbs.
