# NPCs And Monsters

Use this when creating or reviewing NPCs, monsters, and wandering livings.

Each section carries the distilled API and names its doc page. Verify against the doc when you have access; otherwise this is sufficient.

## Basic Pattern

> Docs: `/doc/build/monster`; source `/std/monster.c`, `/std/living.c`; examples `/doc/examples/`

```c
#include "/path/to/area/header.h"

inherit AREA_I_CITIZEN;

static void create() {
    ::create();
    set_gender(1 + random(2));
    set_race("human");
    set_name("market guard");
    set_short("a market guard");
    set_long("This guard watches the market with practiced patience.");
    add_id(({ "guard", "watchman" }));
    set_level(18 + random(3));
    set_hp(350 + random(100));
    set_wc(8);
    set_ac(10);
    set_al(300);
    add_skill("resist_slash", 15);
    load_chat(4, ({
        "*say Keep moving.",
        "*peer",
    }));
    load_a_chat(8, ({
        "*yell Guards!",
        store_fp("warn_attacker"),
    }));
}

static void warn_attacker(object foe) {
    if (!objectp(foe)) {
        return;
    }
    command("say You should have walked away, " + foe->query_name() + ".");
}
```

Use the local area std class when available; modern NPCs often inherit area-specific bases instead of raw `I_MONSTER`. Monsters are usually spawned via `add_reset_object()` in the room (see `rooms.md`) so they return each reset.

Some modern NPC bases defer behavior wiring to a post-create step (a `create_done()`-style callback scheduled with `call_out(..., 0)`) and reject configuration setters after setup completes. Check the base class: configure in `create()`, and let the base wire behavior afterward if it does so.

## Core Setters

> Docs: `/doc/build/monster`

- `set_level(l)` — sets hp/ep as for a player of level l, ac 0, wc of bare hands. **Call this before** `set_hp`/`set_sp`/`set_ac`/`set_wc`, or your overrides get clobbered (`mult_hp`/`mult_sp` change unexpectedly).
- `set_hp(n)`, `set_sp(n)`, `set_ep(n)` (ep can only be set lower than current), `set_wc(n)` (damage per hit is `0..wc-1`), `set_ac(n)`.
- `set_al(n)` — alignment; negative evil, positive good. Player-scale reference: ≤ -1000 demonic, -999..-200 evil, -199..-40 nasty, -39..39 neutral, 40..199 nice, 200..899 good, 900..1299 saintly, ≥ 1300 paladine.
- `set_race(r)` — always set it; weapons and effects check `query_race()`. Common live values (most frequent first): human, undead, animal, demon, elf, dragon, orc, troll, dwarf, gnome, bird, goblin, vampire, bear, giant, horse, cat, wolf, kobold, insect, golem, ogre, rat, elemental, spider, ghost (see `/doc/build/races`).
- `set_gender(n)` — 0 neuter, 1 male, 2 female.
- `command(cmd)` — force the NPC to perform a command.

## Behavior Options

> Docs: `/doc/build/monster`

- `set_aggressivity(a)` — 0 = peaceful until attacked; otherwise the per-heartbeat chance to attack a player, auto-modified by relative alignment and power (a small good monster rarely jumps a big good player).
- `set_wimpy()` — flee below 20% of max hp; `set_wimpy_hp(i)` — flee below i hp; `set_wimpy_percent(i)` — flee below i% of max.
- `set_combat_arr(arr)` — custom unarmed combat messages: `({ dmg_limit, "what", "how", genitive_flag, ... })` repeating, with damage limits increasing monotonically.
- `set_random_pick(i)` — i% chance per heartbeat to pick up items; auto-wields a picked weapon if better, tries to wear picked armour.
- `set_no_corpse(1)` — no corpse on death (overrides `set_corpse_object`); `set_corpse_object(file)` — custom corpse.
- `set_init_ob(ob)` — `ob->monster_init(monster)` is called from the monster's `init`; return 1 to suppress an aggressive attack for that player.

## Equipment

> Docs: `/doc/build/monster` (`add_eq`)

Basic: `add_eq(file)` clones the file into the monster and wears/wields it if it is armour or a weapon. Modern areas typically use quality-driven daemon helpers instead:

```c
if (!clonep(this_object())) {
    return;
}
AREA_D_WEAPON->add_weapon(75);
AREA_D_ARMOUR->add_armour(50);
```

Follow the local pattern; never give a prototype/master object live inventory (guard with `clonep`).

## Spells

> Docs: `/doc/build/monster` (spell functions), `/doc/build/damage`

Built-in spell casting, checked once per combat round:

- `set_chance(c)` — percent chance per round to cast.
- `set_spell_dam(d)` — damage is random `0..d-1`.
- `set_spell_dam_type(t)` — string or array (random pick per hit) from the damage-type list in `combat-gear.md`; defaults to `"blunt"`. Invalid types create system logs.
- `set_spell_mess1(m)` — message to others in the room; `set_spell_mess2(m)` — message to the victim. Both support `format_message()` codes with `$` = monster, `#` = victim (see `lpc-basics.md`).

For boss-type NPCs, derive hit points from intended fight design (raid size × expected damage per second × fight length) rather than picking a big number, and check `/doc/build/properties` and the area header for combat opt-out properties before hand-rolling special defenses.

## Chats

> Docs: `/doc/build/monster` (chat functions)

`load_chat(chance, msgs)` fires outside combat; `load_a_chat(chance, msgs)` in combat. `chance` is a 1–100 per-heartbeat percentage — keep it low or the NPC gets noisy. Array entries:

- Plain string — emitted as-is (formatted with `format_message()`).
- `"*command"` — the monster performs the command.
- `"!command"` — as `*`, but the attacker's real name is appended as argument (a_chat).
- A function pointer — called with the attacking object as argument (a_chat).

## Reactions To What The NPC Sees And Hears (Catch Talk)

> Docs: `/doc/build/monster_response`, `/doc/build/monster` (catch talk)

`add_response(ob, action, match, response, chance)` makes the NPC react to text and feelings it perceives:

- `action` — string (or array) to match in incoming messages; `match` — extra substring that must also match (e.g. `"at you."`).
- `response` — a plain string (told to the room), `"!command"` (the NPC performs the command — requires `set_soul()` in `create()` for feelings), `"#function"` (calls that function with the matched string), or an array mixing all three.
- `chance` — 1–100, default 100.
- Responses support `$N/$n` (name of the player who triggered), `$O/$o` (objective), `$P/$p` (possessive), `$R/$r` (pronoun).

```c
add_response(this_object(), "smiles", "", "!say $R seems happy today.");
add_response(this_object(), ({ "points", "glares" }), "at you.",
             "!say Leave me alone, $N.", 75);
```

A feeling-specific form matches feeling types (`noise`, `touch`, `aggressive`, `friendly`) with filters (`equal`, `present_or`, `present_and`, negations, or a custom filter function) — read `/doc/build/monster_response` before using it. Lower-level: `set_match(ob, funcs, types, matches)`.

## Wandering Monsters

> Docs: `/doc/build/monster` (wandering section), `/doc/build/walking_monster`

Inherit `I_WMONSTER` (or `I_RWMONSTER`) instead of `I_MONSTER`, then:

- `set_wandering_chance(percent)` — chance to move per tick.
- `set_wandering_time(seconds)` — seconds between move attempts (min 2; also starts the wandering).
- `set_leaves_area(bool)` — whether it may leave its creator's area.
- `set_wandering_route(commands)` — fixed patrol route (array of direction commands).
- `set_wandering_start(room)` — starting room.
- `set_wandering_hook(func)` — function receiving the current environment, returning the direction command; **it must itself respect the `no_wander` room property**, since it overrides the regular restrictions.

Wandering monsters add hooks `__wander` (blocking variant available), `__wander_done`, `__wander_fail`.

## Hooks And Reactions

> Docs: `/doc/hooks/hooklist` and per-hook pages under `/doc/hooks/`

NPCs commonly hook lifecycle and combat events:

```c
add_hook("__init", store_fp("check_reward"));
add_hook("__kill", store_fp("claim_reward"));
add_hook("__wander_done", store_fp("calm_down"));
```

Use exact names from the hooklist (bad hooks are logged), match the documented callback signature, and keep handlers small and guarded. See `hooks.md` for the catalog and cleanup discipline.

## Common Mistakes

- Calling `set_hp` before `set_level`, accidentally resetting values.
- Forgetting `set_race`, or `add_id` for natural player references.
- Overusing chat chances; chats every few heartbeats get muted by players.
- Using `"!..."` responses without `set_soul()`.
- Giving inventory to the blueprint instead of clones.
- Aggressive monsters that can wander out of the area (`set_leaves_area` left on, or a wandering hook ignoring `no_wander`).
- Adding hooks without checking exact hook names.
