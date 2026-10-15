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
- [x] Confirm whether swim distance should count toward the existing
      mile-based badges (`5 Mile Club`, etc.). **Resolved:** no — split
      into fully separate per-activity badge tracks instead (Run/Bike in
      miles, Swim in km), see section 3 below.
- [x] Confirm "max speed" wording/units for swim (mph-style vs. a pace
      framing). **Resolved:** pace framing — "Best 100m pace" (min+sec),
      see section 3 below.

## 1. Special Training Days — ✅ Shipped (`claude/special-training-days` branch, merged here)

All three "Decisions carried over from the plan" questions were resolved
per their recommended defaults — see the implementation note at the top
of section 1 in `IMPLEMENTATION_PLAN.md`, including the one addition
beyond the original sketch (`log.specialDescription` also frozen, not
just `specialTitle`).

**Data model**
- [x] `DATA.specialDays = []` (templates: `{id, title, description, createdAt}`)
- [x] `DATA.settings.scheduledSpecialDays = {}` (`dateStr -> specialDayId`)
- [x] `log.specialDayId` / `log.specialTitle` / `log.specialDescription` /
      `log.specialNotes` / `log.specialActivities` fields, written at
      completion time
- [x] Migration guards in `loadData()`, matching the existing pattern for
      `exerciseOverrides`/`customExercises`; also added `specialDays`/
      `scheduledSpecialDays` to the "Reset all data" fresh-state object
      (it was already missing several other fields for older features -
      not fixed here, out of scope, but didn't make it worse)

**Helpers**
- [x] `specialDayFor(dateStr)` — single source of truth: returns frozen
      log fields once completed (`frozen:true`), else the live template
      if one's scheduled, else `null`
- [x] `scheduleSpecialDay(dateStr, id)` / `unscheduleSpecialDay(dateStr)`
- [x] `addSpecialDayTemplate(title, description)` /
      `deleteSpecialDayTemplate(id)` — deleting also clears any
      not-yet-completed scheduled dates pointing at it, so `specialDayFor`
      never has to handle a dangling reference

**Today tab**
- [x] New render branch in `renderToday()` for `specialDayFor(key)`
      (`renderSpecialToday`), short-circuiting before the normal
      plan/rest-day logic
- [x] Title + description display
- [x] Notes `<textarea>`
- [x] Activities list builder (title + description per row, "+ Add
      activity" to append, "✕ Remove" per row) — backed by an in-memory
      `specialDraft` state var (mirrors `editingExId`/`addingExercise`),
      read from the live DOM before every re-render so typed-but-unsaved
      text in existing rows survives adding/removing a row
- [x] "Complete Special Training Day" button → freezes notes/activities,
      sets `fullyCompleted`, runs `checkNewBadges()`
- [x] Edit flow for a completed special day (re-open frozen notes/
      activities, not the live template)
- [x] "⚡ Special day" button next to `#hdrWeekPill` (styled as a
      `.weekpill` for contrast on the dark header, shown regardless of
      whether today has a week pill or is a rest day) → picker modal →
      `scheduleSpecialDay`
- [x] "↩ Back to normal workout" to unschedule (only shown before
      completion)

**Calendar / History**
- [x] Scheduling entry point inside `showDayModal` for other dates
      (`⚡ Use a special training day` / `⚡ Change special training day`)
- [x] `showDayModal` renders the special-day template/log instead of the
      normal exercise list when one is scheduled/logged for that date
- [x] Month calendar grid (`renderHistory` / `.cal-cell`): a small ⚡
      marker (`.special-mark`) on any day with a completed special day
- [x] Scheduled-but-not-yet-completed days also get the marker, at 45%
      opacity (`.special-mark.dim`) to distinguish from completed
- [x] A completed special day on a Sunday now shows "done" (green) in the
      calendar instead of "rest" (grey) — a normal rested-but-unmarked
      Sunday is unaffected

**Settings / management**
- [x] New "Special Training Days" card: list + "+ New special training
      day" modal (Title, Description, Save)
- [x] Delete template (with confirm)

**Decisions carried over from the plan** (all resolved, see implementation
note in `IMPLEMENTATION_PLAN.md`)
- [x] Sunday scheduling + streak counting — `scheduledDow(dow, key)` now
      checks the specific date's `specialDayId`, not just day-of-week
- [x] Special-day completion counts toward `totalWorkouts`/badges —
      confirmed free, verified in testing

**Testing (jsdom)**
- [x] Create template, schedule on Today via the header button, fill
      notes + 2 activities (verified surviving a mid-entry re-render),
      complete, verify all fields frozen correctly in the log
- [x] Calendar shows a full-opacity bolt + "done" styling for the
      completed day; a separately-scheduled future day shows a dim bolt
- [x] History modal shows frozen title/description/notes/activities, no
      reschedule button once completed
- [x] Deleting the template afterward leaves the completed day's History
      and Today display completely unaffected (frozen fields win)
- [x] Scheduling a future date via History, reopening the picker, "Change"
      flow
- [x] "Back to normal workout" correctly reverts Today to the normal
      Monday workout view, with the header button reappearing
- [x] Streak math: a completed special Sunday extends a streak (Sat+Sun+Mon
      = 3); a plain completed-but-not-special Sunday does not (stays 1)
- [x] Regression: all tabs render with no errors, both mocked and
      real-clock dates; re-ran the week-cycle and cardio test suites
      against this same branch with no regressions

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

## 3. Cardio Upgrades — ✅ Shipped (`claude/cardio-upgrades` branch, merged here)

Scope grew substantially from the original sketch — see the
implementation note at the top of section 3 in `IMPLEMENTATION_PLAN.md`
for the two decisions (swim kept separate from miles; "Best pace" not
"Max speed") and the new badge-system work that wasn't originally planned
(per-activity cumulative + sustained-weekly-average badges, 45 badges
total up from the original 18).

**Fix while here**
- [x] `computeStats()` cardio aggregation bug: reads `l.cardio.type`/
      `milesFromEntry(l.cardio)` as if `log.cardio` were a flat entry
      instead of a dict keyed by exercise id — fixed as part of the
      array-ification below

**Data model**
- [x] `log.cardio[id]` becomes an array of entries (was a single object)
- [x] `cardioEntriesFor(log, id)` read-helper that normalizes old
      single-object entries to a 1-element array (also used by
      `estimateWorkoutSeconds` and `showDayModal`, which had the same
      flat-entry assumption)

**Per-type UI/fields**
- [x] Run/Bike: Max speed input (unit label follows the mi/km selection
      live)
- [x] Run/Bike: avg speed + avg pace (`M:SS /mi` or `/km`) in the summary
- [x] Swim: unit `<select>` changed from mi/km to m/yd
- [x] Swim: **Best 100m pace** input (min + sec, not "Max speed" — see
      decision above)
- [x] Swim: avg pace per 100 m (`M:SS /100m`) in the summary, normalizing
      yards → meters first (verified: 1000 yd / 17 min → 1:52 /100m)

**Helpers**
- [x] `formatPaceSeconds(totalSeconds, unitLabel)`
- [x] `formatAvgSpeed(distance, durationMin, unitLabel)` /
      `formatAvgPace(distance, durationMin, unitLabel)`
- [x] `cardioDistanceFor(entry)` — unit-aware conversion to miles
      (Run/Bike) or km (Swim) for `cardioTotals`/badge aggregation
- [x] `trailing13WeekAverage(weekTotals)` for the sustained-average
      badges

**Badges (new scope)**
- [x] Replaced the 4 combined "X Mile Club" badges + kept "Triple
      Threat" as-is
- [x] `buildCardioBadges()` generates 30 badges: 5/10/25/50/100
      cumulative-total tier × {Run mi, Bike mi, Swim km}, plus the same 5
      tiers × 13-week rolling average × the same 3 activities
- [x] `computeStats()` returns `cardioTotals` and `cardioWeeklyAvg` per
      activity type

**Multiple entries per day**
- [x] Stacked list of summary cards per cardio item (one per logged
      entry), each with its own Remove button
- [x] "+ Add another cardio entry" button; hidden while that item's form
      is open, with a Cancel button to back out without saving

**Testing (jsdom)**
- [x] Log a run with max speed, verify avg speed (10.0 mph) + avg pace
      (6:00 /mi) + max speed all display correctly
- [x] Log a swim in meters and in yards, verify both produce correct
      meters-normalized `/100m` pace
- [x] Log two entries (Run + Swim, then Run + Bike) under the same cardio
      slot same day, verify both persist independently
- [x] Remove entries down to zero, verify `log.done[id]` resets to false
      and the form reappears
- [x] `computeStats()` per-activity totals verified (Run totals correct,
      Bike stays 0, `totalMiles` = Run+Bike only)
- [x] Badge generation verified (45 total, old combined badges gone, new
      ids present, a tier badge actually earns when its threshold is met)
- [x] 13-week rolling average verified against a hand-computed expected
      value (3 weeks of 10mi runs ÷ 13 ≈ 2.31 mi/wk)
- [x] Migration: an old pre-upgrade single-object `log.cardio[id]` entry
      still renders and counts toward the duration estimate
- [x] Cancel button on the "add another" form
- [x] Duration estimate sums multiple entries' durations correctly
- [x] History (`showDayModal`) shows all entries for a day, not just one
- [x] Regression: all tabs render with no errors, both mocked and
      real-clock dates

## 4. Follow-up requests (post-launch) — ✅ Shipped

Smaller requests that came out of hands-on testing of the three features
above, each on its own branch, merged back here.

**Add-exercise week-scope options** (`claude/add-exercise-scope-options`)
- [x] The "+ Add exercise" week-selector now shows on every day (previously
      only on `hasAlt:true` days), with a new **"Just today"** option
      alongside Every week/A only/B only
- [x] One-time exercises are stored in `log.oneTimeExercises` (frozen to
      that date, not `settings.customExercises[planSlot]`), so they never
      recur on future weeks
- [x] `oneTimeExercisesFor`/`addOneTimeExercise`/`removeOneTimeExercise`/
      `oneTimeItemsFor` helpers; counted in progress/duration; shown with a
      "ONE-TIME" badge instead of "ADDED"; visible in `showDayModal` history
- [x] Tested: selector appears on a `hasAlt:false` day, one-time item
      doesn't appear on a future occurrence of that day, counts toward
      progress, hard-deletes cleanly, "Every week"/week-specific options
      still work (regression)

**Per-week day swap** (`claude/per-week-swap`)
- [x] Settings "Swap workout days" card gained a week selector so the same
      swap mechanic (already shipped) could target week 3+, not just A/B
- [x] Tested: swap in one week doesn't affect another; "Restore original
      day order" scoped to the viewed week; regression on A/B swap + legacy
      single-`dayMap` migration

**Cross-week day swap** (`claude/cross-week-day-swap`)
- [x] Generalized further so the two sides of a swap can belong to
      *different* weeks (e.g. Monday of Week 2 with Wednesday of Week 3),
      not just within one week — per the user's explicit example
- [x] Settings card now shows two independent Week+Day pickers ("Day A" /
      "Day B") instead of one week selector driving both sides; the
      "Current order for" viewing/reset section is unchanged
- [x] No new data model — both sides read/write their own
      `dayMapForWeek(weekKey)` and exchange plan-slot values; same-week
      swaps (`weekA === weekB`) handled as one map mutation so neither
      side's change is lost
- [x] Tested: same-week swap still works exactly as before (regression);
      the user's literal example (Week B Monday ↔ Week 3 Wednesday)
      verified both in the stored day-map and via an actual render showing
      the swapped content; no-op guard for swapping a cell with itself;
      full tab-render regression
