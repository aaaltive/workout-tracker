# Implementation Plan: Special Training Days, Custom Week Cycles, Cardio Upgrades

Planning doc for three upcoming features. Nothing in this branch changes app
behavior yet — this is design + a checklist (see `TODO.md`) to work from in
follow-up branches, one feature per branch/PR, same as the last few rounds
of work.

The app is a single `index.html` (vanilla JS, no build step, `localStorage`
via the `DATA` object / `saveData()`). All three features below extend that
same pattern: new fields on `DATA`/`DATA.settings`, new helper functions,
new render branches, wired up the same way `exerciseOverrides`,
`customExercises`, and `removedExerciseIds` already are.

---

## 1. Special Training Days — ✅ Shipped

> **Implemented status, see `TODO.md` for the full as-built checklist.**
> All three open questions below were resolved per their "recommend yes"
> defaults and implemented as described:
> 1. **Sunday scheduling counts toward streaks** — but only for the
>    specific Sunday that was completed as a special day, not Sundays in
>    general. `scheduledDow(dow, key)` now takes the date key too, so a
>    plain "mark today as rested" Sunday still never breaks/extends a
>    streak (unchanged from before), while a Sunday with
>    `log.specialDayId` set does.
> 2. **Editing after completion is allowed** — an "Edit" button re-opens
>    the notes/activities form pre-filled from the frozen log fields (not
>    the live template), saving back over the same frozen fields.
> 3. **Badges/`totalWorkouts` count automatically** — confirmed, no code
>    needed beyond setting `log.fullyCompleted` the same way a normal
>    workout does.
>
> One addition beyond the original sketch: `log.specialDescription` is
> also frozen at completion (the sketch below only froze `specialTitle`).
> Without it, deleting a template after completing a day with it would
> have kept the title but silently lost the description — freezing both
> keeps the "deleting a template never rewrites history" guarantee
> complete.
>
> The rest of this section is kept as the original pre-build sketch, for
> history.

A special training day **replaces** a specific calendar date's normal
workout with a free-form activity: a title, optional description, and (on
completion) notes plus a list of logged activities.

### Data model

```js
// Reusable templates the user creates once, schedules many times.
DATA.specialDays = [
  { id, title, description, createdAt }
];

// Per-date scheduling: which special day (if any) replaces a given date's
// normal plan. Keyed by calendar date, NOT by weekday/plan-slot — this is
// a one-time assignment, unlike exerciseOverrides/customExercises which
// apply to a recurring weekday template.
DATA.settings.scheduledSpecialDays = { "<dateStr>": "<specialDayId>" };
```

`DATA.logs[dateStr]` gains three fields, written at **completion** time
(not at scheduling time), following the same "freeze what actually
happened" principle used for exercise snapshots:

```js
log.specialDayId        // which template was used (for display/reference)
log.specialTitle        // frozen copy of the title, in case the template is edited/deleted later
log.specialNotes        // free-text notes entered on Today
log.specialActivities   // [{ id, title, description }]
```

Freezing `specialTitle` (and reading from the log rather than re-looking-up
`DATA.specialDays` once completed) means deleting a template later doesn't
erase history — consistent with how removing a stock exercise doesn't
rewrite past logs.

### Helper functions (new)

- `specialDayFor(dateStr)` — look up the scheduled template for a date.
- `scheduleSpecialDay(dateStr, specialDayId)` / `unscheduleSpecialDay(dateStr)`.
- `addSpecialDayTemplate({title, description})` / `deleteSpecialDayTemplate(id)`.

### Today tab (`renderToday`)

Right now `renderToday` branches on `if(!plan)` for rest days before doing
anything else. Add a branch above/alongside that: if
`specialDayFor(key)` is set, render a dedicated special-day view instead of
the normal exercise list or the rest-day card:

- Header: template title as the heading, description underneath if present
  (muted text, same style as `plan.subtitle`).
- A `<textarea>` for notes (same pattern as the Goals "milestones"
  textarea).
- An activities list builder: each row is a Title input + Description
  input; a "+ Add activity" button appends another blank row (mirrors the
  "+ Add exercise" / "+ Add milestone" inline-form pattern already used on
  Today and in Goals). Needs local editable-list state similar to how the
  Today tab already tracks `editingExId`/`addingExercise` — e.g. an
  in-memory `specialActivityDrafts` array rebuilt on each render.
- A "Complete Special Training Day" button (styled like `.finish-btn`)
  that freezes notes/activities into the log fields above and sets
  `log.fullyCompleted = true`.
- Once completed, show a read-only summary (like the cardio
  summary/edit pattern) with an "Edit" affordance that re-opens the form
  pre-filled from the frozen log fields, rather than from the (possibly
  since-edited) template.

### The Week A/B pill button

Next to `#hdrWeekPill`, add a small button (e.g. "⚡ Special day") that
opens a picker modal listing `DATA.specialDays` (plus a prompt/link to
create one if the list is empty). Selecting one calls
`scheduleSpecialDay(todayKey, id)` and re-renders — Today immediately
switches to the special-day view. While a special day is scheduled but not
yet completed, show an "↩ Back to normal workout" action to
`unscheduleSpecialDay` and revert. Once `fullyCompleted`, don't allow
unscheduling from the UI (would contradict the "frozen history" principle)
— only a Settings-level "delete this log entry" style escape hatch, if ever
needed, not in scope now.

### Calendar / History (`showDayModal`)

Add the same scheduling entry point here for days other than today
(mainly useful for future dates): a button to open the special-day picker
for that date. If a date already has a scheduled or completed special day,
`showDayModal` should render the template/log's title + description +
(if completed) notes and activities instead of the normal exercise list.

### Month grid indicator (`renderHistory`)

The month calendar (`renderHistory`, the `.cal-cell` grid) currently only
colors each day's cell by status — `done`/`rest`/`missed`/`future`, plus a
`today` outline — and renders nothing but the day number inside. Add a
small ⚡ marker on any past/current day whose log has `log.specialDayId`
set, so a special training day is visually distinguishable at a glance
when scanning history:

- Check `log && log.specialDayId` alongside the existing `cls` logic
  (`index.html` around the `if(d > today) cls = "future"; else if
  (dow===0) cls = "rest"; else if(log && log.fullyCompleted) cls = "done";
  ...` block) and add an `is-special` flag independent of `cls` (a special
  day can still be `done` or `missed`).
- Render the bolt as a small overlay (e.g. an absolutely-positioned `⚡`
  in a corner of the cell) rather than replacing the day number — the cell
  is a tight `aspect-ratio:1` box, so this needs a new small CSS rule
  (something like `.cal-cell .special-mark{position:absolute; top:1px;
  right:2px; font-size:9px;}` with `.cal-cell` given `position:relative`).
- Decide whether to also mark **future** scheduled-but-not-yet-completed
  special days (recommend yes, so you can see what's coming up in the
  month view too — just keep the bolt's color/opacity distinct from a
  completed one, e.g. full-opacity once done, dimmer while only
  scheduled).

### Management UI (Settings)

New Settings card, "Special Training Days": list of created templates
(title + one-line description preview) each with a "Delete" button
(confirm first), and a "+ New special training day" button opening a small
modal (Title input, Description textarea, Save) — same modal pattern as
`showNewGoalModal`.

### Open questions to settle before building

- **Sunday scheduling.** `computeStats()`'s streak logic currently treats
  Sunday (`dow===0`) as always-rest and never part of the scheduled-day
  streak math. If a special day can be scheduled on a Sunday, decide
  whether completing it should count toward streaks — recommend yes, but
  it touches `scheduledDow()` inside `computeStats()`.
- **Editing a completed special day.** Allow re-opening the notes/activity
  list after `fullyCompleted` to fix a typo, same as cardio's Edit flow?
  Recommend yes, same UX as cardio.
- **Badges.** Do special training days count toward `totalWorkouts` /
  streak badges the same as a normal workout? Recommend yes — they already
  will, for free, as long as `log.fullyCompleted` is set the same way.

---

## 2. Custom Week Periodicity (beyond Week A/B) — ✅ Shipped

> **Implemented status, see `TODO.md` for the full as-built checklist.**
> Two decisions were made/changed during implementation, both confirmed
> with the user first:
> 1. **Stable key vs. display name, split apart.** The `labels[]` array
>    sketched below would have let renaming a week (e.g. "A" → "Alpha")
>    silently break its match against the stock `PLAN` data's hardcoded
>    `week:"A"` tags. Shipped instead as `weekCycle.keys` (stable,
>    generated once) + `weekCycle.names` (freely renameable display text
>    per key). All rendering goes through a `weekName(key)` helper.
> 2. **New weeks are seeded as a full copy, not left sparse.** The user
>    asked for a new week to clone the week two slots back (`count-2`;
>    e.g. a 3rd week copies week 1, a 4th copies week 2) using each
>    exercise's current effective sets/reps/weight, rather than starting
>    with only "every week" exercises as originally sketched below. New
>    weeks default to a numbered display name ("Week 3", "Week 4", ...).
>
> The rest of this section is kept as the original pre-build sketch, for
> history.

Today, `weekLetterFor(d)` returns `"A"` or `"B"` from ISO-week-number
parity (or a manual `DATA.settings.weekOverride` of `"auto"/"A"/"B"`).
Every `PLAN` item is tagged `week: "A" | "B" | "both"`, and `itemsFor`
filters on `it.week==="both" || it.week===weekLetter`.

### Data model

```js
DATA.settings.weekCycle = {
  count: 2,            // number of weeks in the rotation; 2 = today's behavior
  labels: ["A","B"],   // one label per week, user-renameable, length === count
  override: "auto"     // "auto" | one of labels — replaces weekOverride
};
```

Default value keeps existing behavior byte-for-byte: `count:2`,
`labels:["A","B"]` reproduces today's `weekLetterFor`/`weekOverride`
exactly.

### Generalizing the week lookup

- Replace `weekLetterFor` with `weekLabelFor(d)`: compute
  `getWeekNumber(d) % weekCycle.count`, map to `weekCycle.labels[idx]`,
  respecting `weekCycle.override` when it isn't `"auto"`.
- Item tagging: treat `"both"` as a permanent synonym for "every week in
  the cycle" (so existing `PLAN` data needs zero rewriting) — i.e.
  `it.week==="both" || it.week===weekLabelFor(d)`. New custom exercises
  can be tagged with any current `weekCycle.labels` entry.

### Settings UI

New "Training week cycle" card:

- A number input/stepper for week count (reasonable bound, e.g. 2–6).
- An editable text field per label (defaults to "A", "B", "C", "D", ...
  letters, or let the user fully rename, e.g. "Push", "Pull", "Legs",
  "Recovery").
- The existing Week-override segmented control
  (`Auto / Force A / Force B`) becomes **dynamically generated** from
  `weekCycle.labels` instead of the current hardcoded 3-button markup.

The "+ Add exercise" week-applies selector on Today (currently a
hardcoded `Every week / Week A only / Week B only` 3-button `.seg`) also
needs to generate its buttons from `weekCycle.labels` dynamically.

### The big caveat — say this to the user before building

**Increasing week count does not create new stock exercise content.** The
built-in `PLAN` only ever tags items `"A"`, `"B"`, or `"both"` in the
source data. Setting `count: 4` means weeks 3 and 4 will show only the
`"both"`-tagged (every-week) stock exercises for each day, plus whatever
the user explicitly adds via "+ Add exercise" tagged to week 3/4. Nothing
auto-splits the existing A/B content into four variants. If the user wants
real 4-week-cycle stock programming, that's a separate, much bigger task
(re-authoring `PLAN` itself) — flag this explicitly before starting so
expectations are set correctly.

### Edge case: shrinking the cycle

If `count` is later reduced (e.g. 4 → 2) while exercises exist tagged to
now out-of-range labels ("Week 3", "Week 4"), don't delete that data —
those exercises simply stop appearing (no week index maps to them) and
resurface if the count is increased again later.

---

## 3. Cardio Upgrades — ✅ Shipped

> **Implemented status, see `TODO.md` for the full as-built checklist.**
> Both open questions below were resolved with the user before building,
> and the resolution changed scope significantly:
> 1. **Swim is kept fully separate, not folded into `totalMiles`.** Rather
>    than converting swim meters/yards into a mile-equivalent for the
>    existing combined distance badges, the badge system was split into
>    three independent per-activity tracks (Run/Bike in miles, Swim in
>    km), replacing the old combined "X Mile Club" badges entirely.
> 2. **Swim's field is "Best pace," not "Max speed."** Framed as a
>    fastest-100m-split pace (two number inputs, min + sec) rather than a
>    speed value, matching how swimmers actually think about effort.
> 3. **New scope, not originally planned here:** per-activity cumulative
>    distance badges (5/10/25/50/100 mi for Run and Bike, 5/10/25/50/100
>    km for Swim) **and** a second badge tier for sustaining a weekly
>    average at each of those same thresholds over the trailing 13 weeks
>    (~3 months). Implemented as a rolling average (sum of the last 13
>    calendar weeks ÷ 13, missing weeks count as 0) — this is one
>    reasonable reading of "reached a weekly average and held it for 3
>    months," not confirmed against a stricter alternative (every single
>    week individually clearing the bar). Worth a quick look to confirm
>    it matches intent.
>
> The rest of this section is kept as the original pre-build sketch, for
> history.

Current state: a cardio `PLAN` item (`type:"cardio"`) logs one entry per
day to `log.cardio[id] = {type, distance, unit, duration}` (`unit`: `mi`
or `km`), with an Edit flow that deletes and re-shows the form.

**Found while reading this code for the plan:** `computeStats()` currently
computes cardio totals as `milesFromEntry(l.cardio)` and
`l.cardio.type` — i.e. it reads `l.cardio` as if it were a single flat
entry, not a dict keyed by exercise id the way `renderToday` writes it
(`log.cardio[it.id] = {...}`). That look like a pre-existing bug (cardio
miles/types are probably under- or mis-counted for stats/badges today).
Since this feature already needs to touch that exact aggregation to
support multiple entries, fix it in the same pass rather than filing it
separately.

### Data model changes

```js
// Was: log.cardio[id] = {type, distance, unit, duration}
// Now: log.cardio[id] = [ {type, distance, unit, duration, maxSpeed}, ... ]
```

Add a small migration read-helper so old single-object entries (written
before this change) still work:

```js
function cardioEntriesFor(log, id){
  const v = log.cardio && log.cardio[id];
  if(!v) return [];
  return Array.isArray(v) ? v : [v];
}
```

`log.done[id]` keeps its existing meaning: true once at least one entry
exists for that id.

### Per-type fields and calculations

| Type | Distance unit | New input | Computed + displayed |
|------|---------------|-----------|----------------------|
| Run  | mi / km (unchanged) | Max speed (mph or km/h, matches unit) | Avg speed (distance ÷ duration); avg **pace** as `M:SS /mi` (or `/km`) |
| Bike | mi / km (unchanged) | Max speed (mph or km/h) | Avg speed; avg pace `M:SS /mi` (or `/km`) |
| Swim | **meters / yards** (replaces mi/km) | Max speed | Avg **pace per 100 m**, formatted `M:SS /100m` |

New formatting helpers:

- `formatPace(totalSeconds, unitLabel)` → `"M:SS /unitLabel"`, guarding
  `duration === 0`.
- `formatSpeed(distance, durationHours)` → `"X.X mph"` / `"X.X km/h"`.
- Swim pace needs a meters-normalization step first (convert yards to
  meters if the yard unit was chosen) before the `/100m` math.

Example from the request: a 6-mile run in 36 minutes → avg pace
`"6:00 /mi"`.

### Mile-based stats still need to work with swim in meters/yards

`milesFromEntry` (or its replacement) needs a unit-aware branch so swim
entries still roll into `totalMiles`/the mile-based badges
(`5 Mile Club`, etc.):

- `mi` → unchanged
- `km` → unchanged (`×0.621371`)
- `m` → `×0.000621371`
- `yd` → `×0.000568182`

**Open question:** does it even make sense for swim distance to count
toward the same "miles" badges as running/biking? Effort-per-distance
isn't comparable across disciplines. Flag for a product decision — default
plan is "still counts, for consistency," but worth confirming.

### Multiple cardio entries per day

Change the cardio card's logged view from a single summary to a list of
summary cards (one per logged entry), each with its own Edit/Remove
button, plus a "+ Add another" button at the bottom (hidden while a new
entry's form is actively open, same as other add-item patterns in the
app). This lets a user log, e.g., both a bike ride and a swim under the
same "Cardio: run, bike, or swim" slot on the same day.

### UI layout changes

- Add a "Max speed" field to the log form (third row, under Distance/Unit
  and Time), unit-aware label (`mph`/`km/h` for run+bike; left as a plain
  number — unit TBD — for swim, see open question below).
- Swim's unit `<select>` changes from `mi`/`km` to `m`/`yd`.
- The summary line
  (`✅ <type> · <distance> <unit> · <duration> min`) gains avg
  speed/pace, e.g. `✅ Run · 6 mi · 36 min · 10.0 mph · 6:00 /mi`.

**Open question:** is "max speed" meaningful for swimming the way it is
for run/bike (mph-style instantaneous peak), or would "best/fastest pace"
read more naturally to a swimmer? The request asks for the same field
across all three types — building it that way by default, but worth a
quick confirm before polishing copy/labels.

---

## Suggested build order

All three are independent of each other (no shared data/code dependency),
so they can ship in any order as separate branches/PRs, same workflow as
the sets/reps, add/remove, and weight-tracking features already merged:

1. **Cardio upgrades** — self-contained, no architectural changes, most
   bounded scope. Good first PR.
2. **Special training days** — new subsystem, more surface area (Today +
   History + Settings), but no changes to existing data shapes.
3. **Custom week periodicity** — touches the most shared/core code
   (`weekLetterFor`, `itemsFor`, the Settings week-override control, the
   add-exercise week-select), so it's the one most likely to need careful
   regression testing against the existing A/B behavior. Do this last, and
   confirm the "no auto-generated content" caveat with the user before
   starting.

---

## 4. Follow-up requests (post-launch) — ✅ Shipped

Three smaller requests surfaced while the user tested the branch above,
each shipped on its own branch and merged back in (see `TODO.md` section 4
for the full checklist per feature):

- **Add-exercise week-scope options** — the "+ Add exercise" selector now
  appears on every day (not just `hasAlt` days) and gained a "Just today"
  one-time option, stored separately in `log.oneTimeExercises` so it never
  recurs.
- **Per-week day swap** — the existing day-swap mechanic in Settings
  gained a week selector so it could target week 3+, not just A/B.
- **Cross-week day swap** — generalized once more so the two sides of a
  swap can belong to different weeks (e.g. "Monday of week 2 with
  Wednesday of week 3"). The Settings card now has two independent
  Week+Day pickers instead of one week selector driving both sides. No new
  data model: each side reads/writes its own `dayMapForWeek(weekKey)` and
  the two plan-slot values are exchanged between them.
