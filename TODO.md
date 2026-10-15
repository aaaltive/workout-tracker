# TODO: Special Training Days, Custom Week Cycles, Cardio Upgrades

Checklist form of `IMPLEMENTATION_PLAN.md`. One feature per branch/PR,
same as prior work. Check items off as they land.

## Before starting

- [x] Confirm the "no auto-generated content" caveat for custom week
      counts (section 2 of the plan) — increasing week count only affects
      every-week + user-added exercises, not the stock A/B plan content.
      **Resolved:** user asked instead for new weeks to be seeded as a
      full copy of an earlier week (see section 2 below) rather than
      starting sparse — implemented that way.
- [ ] Confirm whether swim distance should count toward the existing
      mile-based badges (`5 Mile Club`, etc.).
- [ ] Confirm "max speed" wording/units for swim (mph-style vs. a pace
      framing).

## 1. Special Training Days

**Data model**
- [ ] `DATA.specialDays = []` (templates: `{id, title, description, createdAt}`)
- [ ] `DATA.settings.scheduledSpecialDays = {}` (`dateStr -> specialDayId`)
- [ ] `log.specialDayId` / `log.specialTitle` / `log.specialNotes` /
      `log.specialActivities` fields, written at completion time
- [ ] Migration guards in `loadData()` + wherever logs are read, matching
      the existing pattern for `exerciseOverrides`/`customExercises`

**Helpers**
- [ ] `specialDayFor(dateStr)`
- [ ] `scheduleSpecialDay(dateStr, id)` / `unscheduleSpecialDay(dateStr)`
- [ ] `addSpecialDayTemplate(...)` / `deleteSpecialDayTemplate(id)`

**Today tab**
- [ ] New render branch in `renderToday()` for `specialDayFor(key)`
- [ ] Title + description display
- [ ] Notes `<textarea>`
- [ ] Activities list builder (title + description per row, "+ Add
      activity" to append)
- [ ] "Complete Special Training Day" button → freezes notes/activities,
      sets `fullyCompleted`
- [ ] Edit flow for a completed special day (re-open frozen notes/
      activities, not the live template)
- [ ] "⚡ Special day" button next to `#hdrWeekPill` → picker modal →
      `scheduleSpecialDay`
- [ ] "↩ Back to normal workout" to unschedule (only before completion)

**Calendar / History**
- [ ] Scheduling entry point inside `showDayModal` for other dates
- [ ] `showDayModal` renders the special-day template/log instead of the
      normal exercise list when one is scheduled/logged for that date
- [ ] Month calendar grid (`renderHistory` / `.cal-cell`): show a small ⚡
      marker on any day with `log.specialDayId` set, so special training
      days are visible at a glance when scanning past months
- [ ] Decide + implement whether scheduled-but-not-yet-completed future
      special days also get a (dimmer) bolt in the month view

**Settings / management**
- [ ] New "Special Training Days" card: list + "+ New special training
      day" modal (Title, Description, Save)
- [ ] Delete template (with confirm)

**Decisions carried over from the plan**
- [ ] Sunday scheduling + streak counting (`scheduledDow()` in
      `computeStats()`)
- [ ] Confirm special-day completion counts toward `totalWorkouts`/badges
      (should be free, just verify)

**Testing**
- [ ] jsdom pass: create template, schedule on Today, complete with notes
      + 2+ activities, verify frozen in log, verify History shows frozen
      data after editing/deleting the template
- [ ] Regression: all tabs still render; normal (non-special) days
      unaffected

## 2. Custom Week Periodicity — ✅ Shipped (`claude/custom-week-cycles` branch, cherry-picked here)

Implemented with one deliberate deviation from the original sketch below:
each week has a **stable internal key** (used for all `week:` tag
matching, generated once, never changes) separate from a **free-form
display name** (`weekCycle.names[key]`), instead of a flat `labels[]`
array doubling as both. This was necessary because renaming a week
(including the original "A"/"B") would otherwise silently break its link
to the stock `PLAN` data's hardcoded `week:"A"/"B"` tags. Verified by
renaming "A" → "Alpha" and confirming A's stock exercises still matched.

**Data model**
- [x] `DATA.settings.weekCycle = {count, keys, names, override}` default
      (`{count:2, keys:["A","B"], names:{A:"A",B:"B"}, override:"auto"}`),
      migration guard in `loadData()`
- [x] Item `week:` tags continue to store the stable key (unchanged
      mechanism — `"both"`, `"A"`, `"B"`, or a generated key like `wk_...`
      for weeks 3+)

**Core logic**
- [x] `weekLabelFor(d)` replacing `weekLetterFor` (mod-`count` rotation +
      override, override validated against current `keys`)
- [x] `weekName(key)` / `weekColor(key)` helpers — all user-facing text
      goes through `weekName()`, never the raw key (keys for weeks 3+ are
      opaque uids)
- [x] Updated every call site (`renderToday`, `showDayModal`, the
      `#hdrWeekPill`/badge rendering, the Today "+ Add exercise" selector)
- [x] `itemsFor` filter needed no change — already generic string
      comparison against whatever key is passed in

**Settings UI**
- [x] "Training week cycle" card: dynamic `Auto / Force <week>` segmented
      control, per-week renameable text inputs, **"+ Add a week"** /
      **"− Remove last week"** buttons, **"Restore 2-week A/B cycle"**
      reset (shown only when non-default)
- [x] **Seeding on add:** a new week clones every exercise specific to the
      week two slots back (`count-2`; falls back to the most recent week
      if there's no "two back" week yet) — stock + already-added custom
      exercises, using each one's *current effective* sets/reps/weight
      baked in as an independent snapshot. New week's default display
      name is numbered ("Week 3", "Week 4", ...), per the user's answer.
- [x] Removing a week doesn't delete its exercises, only hides them
      (consistent with exercise-removal elsewhere); resets `override` to
      `"auto"` if it pointed at the removed week

**Today tab**
- [x] "+ Add exercise" week-applies selector now generates one button per
      current week (via `weekName()`) instead of the hardcoded 3-button
      Every week/A/B set

**Edge cases**
- [x] Shrinking the cycle doesn't delete out-of-range-tagged exercises
- [x] `isWeekCycleDefault()` / `resetWeekCycle()` for the reset action

**Testing (jsdom)**
- [x] Default 2-week A/B behavior matches the original ISO-week-parity
      logic exactly, for two known dates (regression-critical)
- [x] Force override round-trip; add-a-week seeding (verified the exact 7
      cloned Monday exercises); rename-in-isolation preserves stock A/B
      matching + updates display everywhere; force the new week and see
      the clones; remove-last-week resets a stale override; reset to
      default; "+ Add exercise" selector renders N dynamic buttons and
      tags with the stable key, not display text
- [x] Regression: all tabs render with no errors, both mocked and
      real-clock dates

## 3. Cardio Upgrades

**Fix while here**
- [ ] `computeStats()` cardio aggregation bug: reads `l.cardio.type`/
      `milesFromEntry(l.cardio)` as if `log.cardio` were a flat entry
      instead of a dict keyed by exercise id — fix as part of the
      array-ification below

**Data model**
- [ ] `log.cardio[id]` becomes an array of entries (was a single object)
- [ ] `cardioEntriesFor(log, id)` read-helper that normalizes old
      single-object entries to a 1-element array

**Per-type UI/fields**
- [ ] Run/Bike: add Max speed input (unit matches mi/km selection)
- [ ] Run/Bike: compute + display avg speed and avg pace (`M:SS /mi` or
      `/km`)
- [ ] Swim: change unit `<select>` from mi/km to m/yd
- [ ] Swim: add Max speed input
- [ ] Swim: compute + display avg pace per 100 m (`M:SS /100m`),
      normalizing yards → meters first

**Helpers**
- [ ] `formatPace(totalSeconds, unitLabel)`
- [ ] `formatSpeed(distance, durationHours)`
- [ ] Unit-aware mile-equivalent conversion (mi/km/m/yd) for
      `totalMiles`/badge aggregation

**Multiple entries per day**
- [ ] Render a stacked list of summary cards per cardio item (one per
      logged entry), each with Edit/Remove
- [ ] "+ Add another" button (hidden while a new entry form is open)

**Testing**
- [ ] jsdom pass: log a run with max speed, verify avg speed + pace
      display and math
- [ ] Log a swim in yards, verify meters-normalized `/100m` pace
- [ ] Log two entries (e.g. bike + swim) under the same cardio slot same
      day, verify both persist and `log.done[id]` stays true
- [ ] Verify `computeStats()` totals/badges after the aggregation fix
- [ ] Regression: all tabs render, existing single-entry logs from before
      this change still display correctly
