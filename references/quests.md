# Quests

Use this when designing, implementing, or reviewing a quest, miniquest, or reward-giving scenario.

## Design Rules

> Docs: `/doc/build/quest`

- Quests are puzzles; they need not be dangerous or long. Easy quests are welcome — they simply earn fewer questpoints.
- The quest must work **every reset**, and a mortal must not be able to break it for other mortals (don't let a key item be destroyed, hoarded, or left in a solved state).
- Nothing counts as a quest until QC approves and installs it; the quest name cannot be set on regular players before that.
- A quest is solved when *you* decide it is: one file (the registered quest file) detects completion and grants credit.

## Granting Credit

> Docs: `/doc/lfun/player/set_quest`, `/doc/lfun/player/query_quests`, `/doc/lfun/player/query_questmapping`

```c
if (!ply->query_quests("battlering")) {
    int qp = ply->set_quest("battlering");
    /* qp = points granted; -1 means granted-with-0-qp; 0 means already set or illegal */
}
```

- `set_quest(name)` returns the dynamic qp granted (from `QUEST_D`), `-1` if the quest grants 0 qp, and `0` if the player already has it or the name is not an approved quest.
- Always guard with `query_quests(name)` (returns true/false for one name). For listing, prefer `query_questmapping()` — bare `query_quests()` (no argument) is a compat hack returning a `#`-delimited string.
- **Testing:** wizards and testcharacters can `set_quest` any string, approved or not — develop and test the full grant path before QC registration.
- Anti-cheat: the daemon knows which file is allowed to set each quest (`QUEST_D->is_valid_file(questname, filename)`); grant from the registered `File` only.

## Registration (QC Submission)

> Docs: `/doc/build/quest`

Submit these fields to QC:

- `QuestName` — the key in the player's quest mapping (visible via `stat`).
- `File` — the file that sets the quest; absolute path, no `.c` suffix.
- `QuestShort` — shown in the adventurers' guild quest list; max 50 characters.
- `QuestHint` — where the quest is, roughly what level it suits, how to start; `@`-prefixed continuation lines.
- `QP` — decided by QC, not by you.

## Questpoints Are Dynamic

> Docs: `/doc/build/qpsystem`; daemon `/secure/daemons/quest_d.c`; defines `<quest.h>`

Each quest has a static QP (set by QC) and a dynamic QP computed by `QUEST_D` from how often quests get solved relative to their weight — popular quests drift down, neglected ones drift up. `set_quest` grants the *current dynamic* value. Daemon API: `query_quests()` (names sorted by current qp), `query_quest_data(name)` (array indexed by `QUESTFILENAME`, `QUESTNAME`, `QUESTHINT`, `QUESTNORMALQP`, `QUESTCURRENTQP` from `<quest.h>`), `is_valid_quest(name)`, `query_current_qp(name)`, `query_normal_qp(name)`, `query_total_qp()`.

## Reward Calibration

> Docs: `/doc/build/quest_rewards`

Rewards are `gold+equipment value / total ep`, given partly **while solving** and partly **at completion**; give gold as a mix of coins and equipment:

| tier | value/ep | effort | max alignment shift |
|---|---|---|---|
| major quest | 20000 / 30000 | 3–5 hours, large area | ±200 |
| medium quest | 10000 / 15000 | 1–3 hours | ±100 |
| easy quest | 3000 / 5000 | about an hour | ±50 |
| scenario (unregistered) | 0–5000 / 0–5000 | free-form | — |

Easy quests and scenarios are called **miniquests** here. (The docs mention `/doc/build/miniquest_obj` and `/room/miniquest_room`; both are gone — treat them as historical references, follow the reward table instead.) Ignore the room-count guidance in the doc: never count rooms; invest depth in each room and only add one when the current rooms can't hold the next idea.

## One-Time Rewards

> Docs: `/doc/lfun/player/set_flag`, `/doc/lfun/player/test_flag`, `/doc/lfun/player/clear_flag`; sfuns `/doc/sfun/bit/`

Players may receive a quest's rewards only once. The quest mapping (`query_quests`) gates the qp; for the material rewards use player flags:

```c
if (!ply->test_flag(MY_QUEST_REWARD_FLAG)) {
    ply->set_flag(MY_QUEST_REWARD_FLAG);
    /* pay out gold/equipment/ep */
}
```

Flags are owned: unless you are an arch you can only set flags you own — ask an archwizard to allocate one. Re-solving must be impossible or reward-free.

## Structure Pattern

The tested shape for a quest implementation:

- Quest state that must survive within a reset lives in the area's objects; anything that must survive across resets lives in a daemon or on the player (flags, the quest itself).
- One registered quest file owns completion detection and calls `set_quest` + rewards; every other object feeds it through normal gameplay (items handed in, monsters killed, triggers pulled).
- Keep hints discoverable in-game (an NPC, a readable, the guild hint) — see the no-deadly-traps and telegraphing rules in `SKILL.md` § QC Ground Rules.

## Common Mistakes

- A quest a mortal can jam for others (carrying off the only key item, leaving a lever in the solved position past reset).
- Granting from a file other than the registered `File` — the daemon check fails.
- Skipping the `query_quests` guard, double-granting rewards, or gating rewards on nothing.
- Rewards outside the calibration table, or alignment swings above the tier cap.
- Assuming a fixed qp value — qp is dynamic; never hardcode it in reward math.
- Testing only as a wizard and concluding registration works (wizards can set anything).
