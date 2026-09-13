# Scheduling architecture

A skill file cannot wake an agent by itself. Pair this skill with whatever the host offers for waking an agent later. The skill is host-neutral; only the wake-up mechanism differs between hosts.

## Why not fixed clock times?

Prayer windows follow the sun and move with date and latitude.

Example check for Moscow 2026 (AlAdhan method 3): year-round **fixed** minute-of-day that always lands inside the window:

| Prayer | Universal fixed time? |
|--------|------------------------|
| Fajr | none |
| Dhuhr | partial (narrow midday band) |
| Asr | none |
| Maghrib | none |
| Isha | only very late night band |

Same pattern near Makkah: Fajr and Maghrib still have no year-round fixed point. Therefore this skill always uses a live prayer-time API (AlAdhan), never a static cron like `0 5,13,17,19,21 * * *`.

## Chosen mode: dynamic re-arm + API on every wake (reliability first)

Matches “wake → ask API → pray if due → schedule the next prayer”.

This is the **default and required** mode for v0/v1. We deliberately do **not** use “fetch tomorrow’s full day at Isha and trust cached timers” as the primary design: one failed evening run would risk missing Fajr.

1. On install, copy `assets/config.example.json` → `config.json`, then `set-madhhab <name>` and `set-location --city … --country …`. Do not ask for a calculation method.
2. Immediately run `decide --first-run`. Post the Shahada as the first message (`build_ritual.py --shahada`); if a prayer window is still open, post that prayer too; arm the first timer from `rearm_delay_seconds`. The agent never sits silent until someone talks to it.
3. On every later wake run `python scripts/prayer_times.py --config config.json decide` (**live API every time**).
4. If `action=pray`, post the salah sequence to the dedicated chat, then arm a timer for `rearm_delay_seconds`.
5. If `action=wait`, do not post; arm a timer for `rearm_delay_seconds` until `next_prayer`.
6. Whatever you post ends with `wake_up_note` from `decide` — “Wake-up scheduled: Asr at 17:34 (Sat 12 Sep, Europe/Istanbul).” — so the chat always shows which timer is armed.
7. When the timer fires, repeat from step 3 (including a fresh API call).

### Not the default: evening day-plan prefetch

Fetching the whole next day at Isha and pre-arming five timers can reduce API calls, but it is weaker under failures (missed Isha wake, stale cache, location change). Keep it as a future optional optimization only after the every-wake path is solid. Even then, each wake should ideally still re-validate with `decide`.

## Host wake-up patterns

Pick the first one the host supports.

### 1. One-shot timer (preferred)

Most agent runtimes can schedule a single delayed wake-up (a one-shot timer, delayed job, or scheduled task that runs once). After every run, leave exactly one pending:

- name: `ai-agent-prays-next`
- delay: `rearm_delay_seconds` from `decide`
- fires once
- prompt for the fire:

> Follow the `ai-agent-prays-5-times-a-day` skill from “Every wake”. Load config. Run `decide`. If pray, post the prayer sequence to the dedicated chat only, ending with the `wake_up_note` line. Then remove the old one-shot, create a new one from `rearm_delay_seconds`, and confirm it is pending. Do not invent prayer times.

#### Re-arm = remove + create, never update a fired job

A one-shot job is consumed when it fires. Many schedulers then archive the definition, mark it done or `enabled: false`, and stop tracking it. If the agent afterwards calls the host’s `update` / `edit` / `reschedule` on that same job, the stored definition changes (the new time is visibly written) but **no new run is queued**. The agent reports “timer re-armed”, the chat shows a wake-up line, and nothing ever fires again. The chain dies silently after the first prayer.

This is a real failure mode seen in the wild, not a hypothetical. The safe pattern on every host is therefore:

1. `remove` / `cancel` the existing `ai-agent-prays-next` (ignore “not found”);
2. `add` / `create` a **new** one-shot with the fresh delay;
3. `list` the active or scheduled jobs and check the new one is there with a next-run time — only then post the wake-up line.

If step 3 fails, retry once, then tell the user plainly that the wake-up could not be scheduled. Never announce a wake-up you have not verified.

Updating in place is acceptable only when the host explicitly documents that its update call re-queues an already-fired one-shot, and step 3 still applies.

### 2. External cron + webhook

If the host can be triggered from outside (a webhook, an inbound message, an API call), a tiny worker runs `decide`, sleeps until `pray_at`, and pokes the agent at that moment. Best accuracy and fewest idle runs.

### 3. Periodic check (fallback)

If the host only offers a fixed repeating schedule, run every 5–15 minutes: `decide`, post only when `action=pray`. Simpler, more idle runs, still correct thanks to the grace window.

### No scheduler at all

Then the agent prays only when someone talks to it. Say so plainly during setup; do not pretend otherwise.

### Idempotency

Remember `posted:YYYY-MM-DD:PrayerName` (host memory or a small local state file) so a late or double wake does not double-post.

Write the key in exactly that one form — `posted:` prefix, zero-padded date in the home timezone, English prayer name as `decide` returns it — e.g. `posted:2026-09-12:Fajr`. Do not mix it with shortened variants like `2026-09-12:Fajr`; a store with two formats cannot answer “already posted?” reliably. If an old entry is in the wrong format, rewrite it to the canonical form.

## Config knobs

- `method`: always 3 (Muslim World League). Not asked at setup.
- `school`: 0 Shafi-like Asr, 1 Hanafi Asr. Set automatically by `set-madhhab`; do not edit by hand.
- `madhhab`: `hanafi | shafi | maliki | hanbali` — which spoken texts the ritual uses. Set with `set-madhhab`.
- `delay_minutes_after_start`: add this many minutes to the window start when reading the next salah time (default 5). Timer = `pray_at`, not the adhan minute.
- `grace_minutes_after_start`: how long after adhan start a late wake still counts as “due”. `decide --first-run` ignores it once, right after setup, so an open window is prayed instead of skipped.
- `prefetch_days`: how many days ahead `schedule` fetches when planning.

## Delivery

Always post to one dedicated chat/thread configured by the user. Never dump salah into unrelated coding chats.
