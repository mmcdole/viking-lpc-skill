# Hooks, Triggers, And Callbacks

Use this when adding or reviewing `add_hook`, `remove_hook`, `hook`, `bhook`, `add_trigger`, function-pointer descriptions, movement callbacks, inventory callbacks, or cleanup code.

Each section carries the distilled facts and names its doc page. Verify against the doc when you have access (per-hook pages live under `/doc/hooks/__*`); otherwise this is sufficient. Also useful: nearby modern hook users in the target area — extract the lifecycle pattern; do not copy area names or paths.

## Hook Kinds

> Docs: `/doc/hooks/hooks.about`, `/doc/hooks/hook`; source `/std/mod/hook.c`

Hooks named `__b...` are blocking hooks. Other hooks are notification hooks.

- Notification hooks are fired with `hook("__name", ({ args... }))`; all callbacks run and return values are collected but usually ignored.
- Blocking hooks are fired with `bhook("__bname", ({ args... }))`; nonzero callback results can block the action.
- Use only globally registered hook names from `/doc/hooks/hooklist` or valid custom names in the form `__<euid>_<name>`.
- Custom blocking hooks should follow the `__<euid>_b<name>` convention.

Read the specific hook doc before writing the callback. The callback signature must match that hook's standard arguments, plus any custom arguments bound at `add_hook` time.

## Hook Catalog

> Docs: `/doc/hooks/hooklist` (verified against the live list); per-hook pages under `/doc/hooks/`

Hooks marked `*` also have a blocking `__b<name>` variant. Hooks are inherited down the class tree (a monster has all living and object hooks). Callback signatures differ per hook — check the per-hook doc when available.

- **All objects** (`I_OBJECT`): `__destroy`, `__enter_inv`, `__leave_inv`, `__init`, `__move`, `__reset`, `__short`, `__long`*
- **Livings** (`I_LIVING`) add: `__attack`*, `__add_hit_exp`*, `__add_money`, `__choose_killer`, `__choose_target`, `__damage_dealt`, `__damage_done`, `__die`, `__kill`, `__hit_player`, `__reduce_hit_point`, `__fight_beat`, `__peace_beat`, `__flee`*, `__move_player`*, `__give`, `__heart_beat`, `__eat`*, `__drink`*, `__drink_alco`, `__wear`*, `__wield`*, `__remove`*, `__open`, `__close`, `__receive`, `__receive_feeling`*, `__do_feeling`, `__feeling_occurred`, `__query_skill`, `__set_alignment`, `__restore_spell_points`, and blocking-only guards `__btest_dark`, `__btoodrunk`, `__btoosoaked`, `__btoostuffed`, `__bnotify_attack`, `__bnotify_attacking`, `__bbusy_next_round`
- **Monsters** add: `__beat_stopped`, `__catch_tell`; wandering monsters add `__wander`*, `__wander_done`, `__wander_fail`
- **Players** add: `__quit`, `__reenter`, `__linkdead`*, `__look`, `__invis`, `__vis`, `__quest`, `__query_exit`, `__query_visible_exits`, `__set_dead`, `__add_alignment`, `__bal_title`, `__bprompt`, `__short`*
- **Items** add: `__break`*, `__drop`*, `__get`*, `__wear_out`*, `__info`; armour: `__wear`*, `__remove`*; containers and doors: `__open`*, `__close`*; corpses: `__set_alignment`; food: `__eat`*; drink: `__drink`*; weapons: `__weapon_hit`*, `__wield`*, `__remove`*, `__damage_type`
- **Rooms**: `__feeling_occurred`, `__perform_move`*
- **Channel daemon**: `__chat`

This catalog is a snapshot — when you have mudlib access, confirm against the live `/doc/hooks/hooklist`.

Blocking-hook silence caveat: for armour `__wear`/`__remove` and weapon `__wield`/`__remove`, a callback returning 1 blocks the action but **no message is produced automatically** — telling the player why is the coder's responsibility (per `/doc/build/armour` and `/doc/build/weapon`).

## Adding Hooks

> Docs: `/doc/lfun/object/add_hook`, `/doc/lfun/object/remove_hook`, `/doc/lfun/object/query_hook`

Common local style:

```c
add_hook("__move", store_fp("on_move"));
add_hook("__destroy", store_fp("on_destroy"));
```

Adding a hook to another object:

```c
to->add_hook("__heart_beat", store_fp("on_heartbeat"));
```

The `add_hook` docs also allow `({ obj, "func" })`. Use that form for lightweight objects when `store_fp` does not attach.

Extra arguments are prepended before standard hook arguments:

```c
obj->add_hook("__move", store_fp("track_move"), "label");
```

The callback receives `"label"` first, then the standard `__move` arguments.

If passing an array as one single custom argument, wrap it in an array:

```c
obj->add_hook("__move", store_fp("track_move"), ({ ids }));
```

## Cleanup Rule

If your object adds hooks to another object, especially a player, it must remove them when no longer applicable. Two named patterns cover nearly every case:

**Teardown lives in the move handler.** Put all removal in the `from` branch and all installation in the `to` branch of your `__move` handler, then route destruction through the same handler by calling it with `to == 0` from the `__destroy` handler. One teardown path then covers drop, give, sale, logout, and destruction.

**Re-target guard.** When a hook can be re-pointed at a new target object, tear down the previous target before hooking the new one, and clear the reference on destroy:

```c
public void set_attuned(object item) {
    if (objectp(_attuned)) {
        _attuned->remove_hook("__short");
    }
    if (objectp(item)) {
        item->add_hook("__short", store_fp("on_attuned_short"));
    }
    _attuned = item;
}
```

Carried-key/item cleanup pattern (move + destroy):

```c
public void on_key_move(object from, object to) {
  if (objectp(from)) {
    from->remove_hook("__bmove_player");
    from->remove_hook("__move_player");
  }
  if (objectp(to) && living(to)) {
    to->add_hook("__bmove_player", store_fp("on_bmove_player"));
    to->add_hook("__move_player", store_fp("on_move_player"));
  }
}

public void on_key_destroy() {
  on_key_move(environment(), 0);
}
```

Modern carried items also remove command paths and heart-beat hooks from the old holder in `on_move`, then remove any item-specific secondary hooks in `on_destroy`. For granting and revoking command paths from a carried item, see `commands.md`.

Do not add player hooks from a plain `I_DAEMON`; the `add_hook` docs warn that daemons do not have a safe `on_destruct` cleanup path. Use an object with cleanup behavior or put the hook on a carried object that removes it on move/destroy.

## Common Hook Patterns

Object movement:

```c
add_hook("__move", store_fp("on_move"));

static void on_move(object from, object to) {
  if (objectp(from)) {
    from->remove_hook("__heart_beat");
  }
  if (objectp(to) && living(to)) {
    to->add_hook("__heart_beat", store_fp("on_heartbeat"));
  }
}
```

Inventory logging:

```c
add_hook("__enter_inv", store_fp("on_enter_inv"));
add_hook("__leave_inv", store_fp("on_leave_inv"));
```

Room enter/leave behavior:

```c
add_hook("__enter_inv", store_fp("on_enter"));
add_hook("__leave_inv", store_fp("on_leave"));
```

Short/long annotations:

```c
item->add_hook("__short", store_fp("on_attuned_short"));

public string on_attuned_short() {
  return "[attuned]";
}
```

Blocking action:

```c
add_hook("__bwear_out", store_fp("prevent_wear"));

public int prevent_wear() {
  return 1;
}
```

## Triggers Versus Hooks

Use `add_trigger` for player commands handled by a room or object:

```c
add_trigger("sharpen", store_fp("do_sharpen"));

static int do_sharpen(string arg) {
  if (!stringp(arg) || arg != "blade") {
    return notify_fail("Sharpen what?");
  }
  return 1;
}
```

Use hooks for lifecycle events, movement, inventory changes, combat events, short/long augmentation, wear/wield/remove, or blocking built-in behavior.

## Callback Guardrails

- Callback functions should be small and match the documented standard arguments.
- Guard every object argument with `objectp` before use.
- Use `living(obj)` when attaching player/living hooks.
- For blocking hooks, return nonzero only when intentionally blocking.
- For notification hooks, return values are usually ignored; use explicit side effects.
- Use exact hook names; invalid hooks are logged to `HOOK`.
- Check `query_hook("__name")` before adding a hook only when duplicate hooks would be harmful, such as one-off short-description annotations.

## Common Mistakes

- Adding hooks to players and never removing them.
- Using notification hooks when blocking behavior is required.
- Returning nonzero from a blocking hook accidentally.
- Registering a typoed hook name.
- Passing an array as custom args without wrapping it correctly.
- Hooking a blueprint/master object when only clones should receive the behavior.
