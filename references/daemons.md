# Daemons

Use this when creating or reviewing daemons, shared services, registries, generators, logging helpers, or persistent/service objects.

## Contents

- Core Pattern
- Header Integration
- API Design
- Registries
- Persistence
- Area Bootstrap Objects ("Castles")
- Logging
- Reset And Heartbeat
- Common Mistakes

## Core Pattern

> Source: `/std/daemon.c`, `/std/save_daemon.c`

Basic mudlib daemon:

```c
#include <mudlib.h>

inherit I_DAEMON;

static void create() {
    ::create();
}
```

`/std/daemon.c` defines an empty `reset`, and daemons rarely override it — their content persists. Override `reset` only to restore saved state on the first call (see Persistence below).

Simple area daemon:

```c
#include "/path/to/area/header.h"

inherit I_DAEMON;

public string query_label(int value) {
    switch (value) {
    case 0..19: return "poor";
    case 20..79: return "ordinary";
    default: return "fine";
    }
}
```

For multi-inherit daemons, label each inherit and chain every inherited `create()` (pattern in `SKILL.md` § Create Pattern).

If inheriting an area daemon std class, inspect it before overriding `create`, `reset`, `heart_beat`, or destruction behavior. Some area daemon bases combine utility, hooks, base daemon behavior, and destroy behavior.

## Header Integration

When adding a daemon in an established area, update the local header with a constant:

```c
#define AREA_D_BOUNTY (AREA_DIR_DAEMONS + "bountyd")
```

Match existing naming and alignment in the target area's header.

## API Design

- Prefer `public` query/create/add methods with narrow arguments.
- Keep helper methods `private` or `static` unless callers need them.
- Validate callers and objects with `objectp`, `living`, `stringp`, etc.
- Return `0` on invalid input when the API is used programmatically; use `notify_fail` only when directly handling a player command.
- Centralize randomized generation here if multiple objects need the same item/NPC/material logic. Follow the modern naming convention: `create_<thing>(quality)` returns an object; `add_<thing>(quality, ...)` creates **and** moves/equips it (typically defaulting the target to `previous_object()`).
- Split one service per concern: a rental/state registry, an object-location tracker, a statistics sink, and a logging facade are four daemons, not one.
- Small stateless predicate daemons compose well with `filter_array(arr, "is_valid", D_FILTER, extra)` — collection logic stays declarative and testable.
- Larger areas sometimes split the logic into a reusable std class and keep the daemon file as a thin concrete instance that only sets paths; follow that split where the area already uses it.

## Registries

> Docs: `/doc/build/registry`; source `/std/registry.c` (`I_REGISTRY`)

For list-shaped persistent data (up to roughly 10k entries), inherit `I_REGISTRY` and override `do_save`/`do_restore` (wrapping `save_object`/`restore_object` on your save path). Entries are mappings; the registry assigns a unique `"id"` key.

- `int insert(mapping data)` — returns the assigned id.
- `void update(int id, mapping data)` — merges keys; use `clear_field(id, key)` to remove a key (you cannot set 0 through `update`).
- `void delete(int id)`.
- Queries: `query_all()`, `query_exact(mapping)` (AND, exact), `query_search(mapping)` (AND, substring for strings / exact for ints), `query(function f)` (filter predicate).
- Aggregation: `stats(field)` counts values of a field; `stats_query(field, criteria)` counts with a filter.

Prefer a registry over hand-rolled parallel arrays/mappings when entries have several fields or need searching.

## Persistence

Give every piece of persistent state exactly one owning daemon:

- Call `restore_object(path)` once, in `create()` or on the first `reset` call (`!flag`).
- Call a private `save()` wrapping `save_object(path, 1)` after every mutating operation — not on a timer.
- Keep save files under the area's `var/` directory via a header macro.
- A daemon that restores state but never saves it is a latent bug: state silently resets every reboot. Keep restore and save symmetric.
- Player-carried persistence belongs on the item itself via auto-load (see `items.md`), not in a daemon.

## Area Bootstrap Objects ("Castles")

> Docs: `/doc/build/castle`

An area's entry wiring is a preloaded file (historically a wizard's "castle"): an `I_DAEMON` registered by an arch in `/secure/etc/init_file`, which on load brings up the area's entry room and grafts exits/descriptions into the outside world. Undo in `on_destruct` any wiring you install in another object — add the exit on load, remove it on destruct.

## Logging

> Docs: `/doc/build/logging`

Use `log_file()` or area logging helpers for durable diagnostics. Current logging guidance:

- Log only what you plan to use.
- Always log errors.
- Include input parameters when they matter.
- Do not log secrets or personal information.
- Avoid logging code that can itself crash.

If the area has a logging daemon or logging macro, use it instead of writing ad hoc log paths. A typical facade routes bare names under the area log directory and formats in one place:

```c
public varargs void log(string file, string str, mixed args...) {
    if (!stringp(file = resolve_file(file)) || !stringp(str)) {
        return;
    }
    if (sizeof(args) > 0) {
        str = sprintf(str, args...);
    }
    log_file(file, str + "\n");
}
```

Log one stream per event type, and include who/what identifiers (`query_real_name()`, `source_file_name()`) so the logs are analyzable later.

## Reset And Heartbeat

`reset(int flag)` receives 0 on load and 1 on later periodic resets. In daemons, use it only to restore persistent state on the first call; when inheriting a daemon base that does work in reset, chain it:

```c
static void reset(int flag) {
    base::reset(flag);
    if (!flag) {
        restore_object(AREA_D_BOUNTY_SAVE);
    }
}
```

Do not add `heart_beat` unless the service truly needs periodic work; prefer explicit calls or `call_out` if local code uses them.

## Common Mistakes

- Creating a daemon but not adding a header constant, leading to hardcoded paths everywhere.
- Mixing command UI and service logic in the same daemon.
- Exposing helper functions as public API accidentally.
- Forgetting to call each inherited `create()` in multiple inheritance.
- Restoring state without saving it (or vice versa) — persistence must be symmetric.
- Adding hooks to players from a daemon; daemons have no safe destruct cleanup (see `hooks.md`).
- Logging vague messages without enough state to debug.
