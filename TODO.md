# TODO: Special Training Days, Custom Week Cycles, Cardio Upgrades

Checklist form of `IMPLEMENTATION_PLAN.md`. One feature per branch/PR,
same as prior work. Check items off as they land.

## Before starting

- [ ] Confirm the "no auto-generated content" caveat for custom week
      counts (section 2 of the plan) — increasing week count only affects
      every-week + user-added exercises, not the stock A/B plan content.
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

## 2. Custom Week Periodicity

**Data model**
- [ ] `DATA.settings.weekCycle = {count:2, labels:["A","B"], override:"auto"}`
      default, migration guard
- [ ] Decide storage for item week-tags beyond `"both"`/current
      labels (no change needed if just adding more labels to pick from)

**Core logic**
- [ ] `weekLabelFor(d)` replacing `weekLetterFor` (mod-`count` rotation +
      override)
- [ ] Update every call site of `weekLetterFor` (`renderToday`,
      `showDayModal`, anywhere else `grep`-confirmed at build time)
- [ ] Update `itemsFor` filter to use the new label

**Settings UI**
- [ ] "Training week cycle" card: count input, per-label text fields
- [ ] Dynamically generate the week-override segmented control from
      `weekCycle.labels` (replacing the hardcoded Auto/A/B buttons)

**Today tab**
- [ ] Dynamically generate the "+ Add exercise" week-applies selector
      from `weekCycle.labels`

**Edge cases**
- [ ] Shrinking `count` doesn't delete out-of-range-tagged exercises
      (they just stop showing)
- [ ] `isDayMapDefault`-style helper if needed for a "reset week cycle to
      default" action in Settings

**Testing**
- [ ] jsdom pass: default 2-week behavior unchanged byte-for-byte after
      the refactor (regression-critical)
- [ ] 4-week cycle: verify rotation, override, and that `"both"`-tagged
      items still show every week
- [ ] Add a custom exercise tagged to week 3/4, verify it only shows on
      that week
- [ ] Regression: all tabs render, History previews unaffected

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
