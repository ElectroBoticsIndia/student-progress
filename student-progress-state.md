# ElectroBotics student app — session log, 19–26 Sep 2026

Work on `index-firebase.html` (the Firebase student-progress app). Final build in this
session: **2026-09-26b**. The build id now prints under the Sign in button, so a stale
GitHub Pages cache is visible at a glance rather than being mistaken for an unfixed bug.

---

## Do these first

Three of the features below are inert until a one-off action is taken. None of them
announce themselves, so it is worth doing them in this order.

| # | Action | Without it |
|---|---|---|
| 1 | **Settings → Repair · Fee History** — seed, then done | Months before a fee change have no rate of their own |
| 2 | **Settings → Repair · Level Progress** — run once | The "Nearing level end" chip stays empty |
| 3 | **Settings → Fee Bands by Level** — fill the grid | Every student reads "No band" on Payments |
| 4 | **Settings → Student Detail · History From** — set Assignments to Sep 2026 | The student page still counts untracked months |
| 5 | Inventory app → turn **off** *Log component sales in the student app* | Completes the `inv_sales` cutover |

---

## Features

### Migration code removed
Every migration helper is gone — the cards on Leads, Students, Payments, Attendance and
Assignments, the Settings toggle, the CSV importer, and the supporting CSS. A stale
`showMigration:true` left in `settings/app` is now simply ignored.

Two helpers were deliberately kept because they have live callers: `aaNormDate` (reads a
student's free-text Join Date) and `driveToImageUrl` (the photo field). Their comments were
rewritten so they no longer claim to be part of an importer.

Also removed a dead `compHistModal` left over from the components→`inv_sales` move; its
close handler no longer existed, so its button would have thrown.

### Tutor availability → Students drill-through
Clicking a tutor chip in the availability grid jumps to the Students table filtered to that
tutor **and** that batch, so the table lands on exactly the number shown on the chip. Enter
and Space work too.

When one time slot has its Batch field spelled more than one way (`Sun 9:30 am` vs
`Sun - 9:30 am`), the grid merges them but the Batch dropdown cannot. Rather than silently
show a different count, the drill falls back to tutor-only and shows an amber note naming
the problem. The "free" text is deliberately not clickable.

**Behaviour change:** drill-through now forces the Status filter back to Active. Every
drill source counts active students only, so a table left on "All" used to show more rows
than the bar or chip that was clicked. This affects the Summary chart drills too.

### Pointer cursor
Tutor chips get `cursor:pointer` via CSS. Chart bars need a Chart.js `onHover` handler —
a canvas has no per-bar hover target, so CSS cannot do it. Applied to the three clickable
charts (Level, Country, Tutor); the Enrolment chart is not clickable and keeps the arrow.

### Level-wise fees with history
`feeHistory` on the student record:

```
feeHistory: [
  { from:'2026-06', amount:1500, currency:'INR', level:'Robotics Foundation' },
  { from:'2026-09', amount:2000, currency:'INR', level:'Robotics Intermediate' }
]
```

`Monthly Fee` and `Currency` stay and hold the latest entry, so exports, the students table
and the inventory app are untouched, and **no Firestore rules change was needed**.

The rule: the fee for month M is the latest entry whose `from` is ≤ M. A month before the
first entry uses the first entry, so nothing renders blank.

Six screens switched to the per-month rate: student payments card and drill, Fee payments
Pending column, Fee revenue expected, reminder text, add-payment suggestion, bill line.

**This closed a real hole.** Fee revenue read the student's *current* fee for every month,
so raising a fee would have rewritten every earlier month's expected revenue. Verified: a
student on ₹1,500 through August and ₹2,000 from September now reads ₹2,700 expected for
June–August and ₹3,200 for September.

Form: an *Effective from* month defaulting to next month, the history list with a remove
button per entry, and a warning before saving when backdating (*"This reprices 3 unpaid
month(s): Jul, Aug, Sep"*). Paid months always keep the amount actually received. Level
changes leave the fee alone entirely. Parents see per-month rates with no explanatory line.

### Per-module history floors (student detail only)
Attendance and assignments began on different dates. A separate Settings card sets a floor
per module for **the student detail page only** — Attendance from June, Assignments from
September.

The trends and tutor screens deliberately keep using the global Data Start, so the same
student can legitimately read 75% on their page and lower in trends. Different floors, both
correct under their own rule.

### Four classes a month, capped
`glExpected` returns `min(class days in the window, 4)`. It is a ceiling, not a fixed
number — a student who joins on the 20th still expects only the classes that remained.

Only the expected side is capped. Attendance is whatever actually happened, compensation
classes included, so a five-Sunday month can read **5 of 4 · 125%**. Above 100% is now
possible and correct; the health bands still show it green.

Applied to assignments as well, so both cards on a student's page agree.

### Save settings moved
It used to sit inside the Currency card while actually saving three cards, two of which are
at the bottom of the screen. Now a bar pinned to the bottom of Settings, showing
**● Unsaved changes** when a field it covers is touched, with a line naming what it is
responsible for. The self-contained cards (Frozen rates, the repair tools) keep their own
buttons.

### Fee bands by level
A Settings grid — one row per level, five currency columns labelled India (INR),
Switzerland (CHF), Canada (CAD), UK (GBP), US (USD). Stored in `settings/app.feeBands`.

Keyed by currency rather than region because "international" is not one number: a UK
student is billed GBP and a US student USD.

Each Payments row is scored on the fee for the selected month against the band for their
**current** level:

| Tier | Rule | Tag |
|---|---|---|
| At band | ≥ 100% | green, shows `+₹400` if above |
| Under band | 80–99% | amber, shows the shortfall |
| Well under | < 80% | red, shows the shortfall |
| No band | not configured, or no fee | grey |

A student halfway through Level 2 still on the Level 1 rate is exactly what this surfaces.

### Level progress flags
A `Level 78%` tag on rows at 70% or above, and a chip to filter them.

**The design was driven by read cost.** Progress lives in a subcollection per student, so
reading it live on the Payments screen would have cost roughly 1,250–5,000 reads per load,
against a 50,000/day free-tier ceiling this project has already hit once. Instead
`levelPct` + `levelPctName` are cached on the student record, written at the two moments
the data is already in memory — saving a topic, opening a profile — at **zero extra reads**.

A level change makes the cached figure stale; the code detects that by comparing the stored
level name, so a stale number is ignored rather than shown wrong, and repairs itself next
time that profile is opened.

### Second chip row on Payments
`All · At band · Under band · Well under · No band · Nearing level end`, filtering
**together** with the status chips — so "Pending" plus "Well under" is one click. Clear
resets both rows.

### Tutor breakdown in Fee revenue
Expanding a month now shows a **Country** table and a **Tutor** table beneath it, same
columns, same share denominator. A tutor teaching both regions gets one merged row tagged
*India + international*; switch the region toggle and it narrows. Students with no tutor
group under *Unassigned*. Totals reconcile against the country table in every view.

### `FIREBASE` tag removed from the header.

---

## Bugs found and fixed

### ⚠️ Parent sign-in — a rule that passed `get` refused every `list`
The long one. Every parent was locked out. Chased through three wrong theories before
finding it.

**The cause.** Parent sign-in is a query, not a document fetch:

```js
where('parentEmails','array-contains', <address>)
```

Firestore evaluates a `get` against the real document, but a **list against the query's
potential result set** — it never looks at documents. Every condition must be provable from
the `where()` clauses alone. `listsParent` had three:

```
'parentEmails' in data          ← key existence: NOT provable
data.parentEmails is list       ← type check:    NOT provable
emailLower() in data.parentEmails   ← maps onto array-contains: provable
```

The first two cannot be proven, so the whole query was refused. Fix:

```
function listsParent(data) {
  return isSignedIn() && emailLower() in data.parentEmails;
}
```

Those two guards were mine, added so a student with no `parentEmails` could not "blow up
the expression, because an error here denies the whole query". They caused the exact
failure they were written to prevent.

**Rules for anything a parent touches:** simulate as **list**, not just `get` — the
Playground only offers `list` when Location is a collection path (`/students`), not a
document path. Put nothing in a parent-facing rule that a `where()` clause cannot prove.
And keep `isStaff()` on the **left** of the `||`: an error on the left denies the whole
read, so putting `listsParent` first would break a staff read of any student lacking the
field.

**Still unverified:** `isParentOf` carries the same shape of guards and backs the four
queries a parent runs after signing in. Left alone deliberately rather than changed on
suspicion. If a parent signs in but attendance/assignments/payments/bills are empty, this
is the first place to look.

### Diagnostic blind spots — the same mistake twice
Both cost real time, and both were mine.

1. **The repair tool could not tell "nothing wrong" from "saw nothing".** With zero rows
   read, it printed "Nothing to repair" — identical to success. It now always reports how
   many records it actually read.
2. **`enterParentMode` swallowed the lookup error.** A `permission-denied` from the rules
   was shown to the parent as *"your account isn't linked to any student"*, pointing
   everyone at the data when the problem was the rules. It now distinguishes a failed
   lookup from an empty one and prints the Firestore error code.

Also added a **"Why is an address refused?"** checker that runs both the admin read and the
real sign-in query and names the exact cause — including "the data is correct but the query
is refused", which is what finally cracked it.

### Settings were never loaded on a student's page
`loadSettings` only ran from Payments and Settings, so opening a profile used the built-in
June default rather than the configured Data Start. Pre-existing, and it hit **parents
hardest** — they land straight on their child's page and never visit either screen.
`openProfile` now loads settings, cached, one read.

### An unrelated edit wrote fee history
The student form arrives pre-filled with the current fee and next month, so saving after
changing a phone number stamped a duplicate entry. An entry is now only added when it
actually changes that month's rate. Related: `Monthly Fee` now always syncs to the newest
entry, which is what let the two drift apart.

### Render-before-set — the same ordering bug, twice
1. **`openPaymentForm`** called `payShowCurrency` before setting `payCurrentCtx`, so the
   amount hint was one dialog behind — showing July's fee while suggesting September's.
2. **`openStudentForm`** called `feeLoadForm(s)` before `setCurrencyValue(...)`, so the fee
   preview note read the currency dropdown while it still held the previous value. A CAD
   student's note said `₹60`. **Display only** — the saved `Currency` and `feeHistory`
   entries were correct throughout, confirmed by checking what Save actually writes.

After the second, swept every dialog opener in the file for the same pattern —
`openLeadForm`, `openDueForm`, `openKitForm`, `openBills`, `openReceipt`, `openGlance`,
`openProjectForm`, `openTutorNotes`, `openNoticeForm`, `showChildPicker` and the rest. **No
third instance.** A permanent test now asserts a visible fee note can never disagree with
the currency dropdown.

### Retracted mid-flight
Proposed reordering the `||` in the students rule so `listsParent` came first. That would
have broken staff reads of any student lacking `parentEmails`, because an error on the left
of `||` denies the whole read. Caught before shipping.

---

## Worth remembering

- **`get` and `list` are different tests in Firestore rules.** A rule can pass one and fail
  the other. Simulate as `list` for anything a parent touches.
- **Publishing rules replaces everything.** Never ship a file described as "the student
  rules" — there is one file covering both apps, and a missing block is not a syntax error.
- **Parents must not be in `users`.** That collection is the staff roster only.
- **The project is on the Spark free tier** — 50,000 reads/day, already hit once. Any
  feature that wants per-student data on a list screen needs denormalising, not reading.
- **Gmail dots and `+tags` are not normalised.** `f.bar@gmail.com` and `fbar@gmail.com` are
  one inbox but different strings; store the one they sign in with.
- These tables render `<thead>` inside `<tbody>` in the DOM. Pre-existing across the whole
  app, renders correctly, nothing depends on it.

---

## Testing

33 Playwright tests against a stubbed Firestore, all passing with zero page errors. Notable
additions this session:

- The fee-revenue regression — raise a fee, prove last June's expected revenue does not move
- Five-Sunday months capping expected at 4 while a late joiner still expects 2
- All four fee-band tiers including above-band, with CHF shortfalls shown in CHF
- A stale level percentage being correctly ignored after a level change
- Both Payments chip rows filtering together
- The parent path with the rules query denied, and with it allowed
- A visible fee note never disagreeing with the currency dropdown

The harness gained a hook that lets a test deny one specific query shape, which is what made
the denied-parent path testable at all.

---

## Open items

- `isParentOf` may hit the same get-vs-list wall as `listsParent` — see above
- The Summary "Students by Country" chart still groups on the raw country value
- Trends and tutor screens could be aligned to the student-detail floors later; nothing
  about the current split prevents it

Related project docs: `firestore-rules-notes.md` · `parent-sign-in.md` ·
`firestore-read-budget.md`
