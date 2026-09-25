# Line Planner

A single-file, cloud-enabled staffing board for production lines. Set up your lines for
a shift, load the day's roster, and drag people onto positions as they arrive.

This is a trimmed-down, general-use version of the Crescent Staffing Planner, with the
site-specific pieces removed so it can be dropped into any site.

## Quick start

Open `index.html` in a browser. That's the whole app — no build step, no install.

1. **Setup** — pick the date and shift, add your lines (letter, lead(s), how many
   associates each needs), then **Start Staffing**.
2. **Staffing** — click **Load** in the Roster panel to bring in today's roster, then
   drag names onto line positions as people come in.
3. Changes auto-save two seconds after you stop typing.

## The Roster panel

The right-hand panel holds everyone who is present but not yet assigned. Drag a name
onto a position to place them; drag them back to unassign.

**Load** opens a modal with three ways to bring in a roster:

- **From roster** (the default) — the active associates from the **Associates** tab whose
  scheduled days include this date. Inactive associates are never included. Tick the box
  to pull everyone active regardless of their scheduled days.
- **Upload file** — an `.xlsx`, `.xls`, or `.csv`. Column headers are detected
  automatically:
  - Name: either a `Full Name` column, or `First Name` + `Last Name`.
  - Employee ID (optional): `Employee ID`, `Emp ID`, `Badge ID`, etc. Supplying IDs
    populates name autocomplete elsewhere in the app.
- **Paste names** — one name per line. `Last, First` is flipped to `First Last`, and
  list numbering (`1.`, `2)`) is stripped.

Then choose **Replace roster** (fresh list for the shift) or **Append** (late arrivals).

Loading is safe to repeat: anyone already assigned to a line, or already in the panel,
is skipped rather than duplicated.

Other controls: **Quick Add** adds one name at a time with autocomplete, and **+ Add**
adds a blank row you can type into.

## Associates

The **Associates** tab holds the site roster. **Upload Roster Spreadsheet** reads the
recruiting roster workbook as it stands — no reformatting first:

- The header row is found by scanning for **Employee Name**, so the title and leads
  lines above it don't matter.
- Day columns (`THUR`, `FRI`, `SAT`, `SUN`, and any other weekday spelling) become that
  person's **scheduled days**. An `x` in the cell counts.
- `Start Date`, `Phone Number`, `CRM #`, `Background`, `language`, `last 4 SSN`,
  `Recruiter`, `Assigned` and `Notes` are read when present.
- Running totals and section labels in the middle of the list (`HEADCOUNT SCHEDULED`,
  `UNDER(-)/OVER`, `Pending I-9`, …) are skipped, and the summary says how many.
- Tabs named for exits — **NCNS**, **Terminate**, **Declines** — are read as name-only
  lists and flip those people to inactive.

Re-uploading is safe and is the expected way to work: people already on file are updated
in place, new people are added, and **nobody is ever deleted**. Upload it once or a dozen
times a day. People are matched on CRM number first and on name second, so a row that
gains a CRM number later still lands on the same person.

### Status

The first column is a **status** dropdown, Active or Inactive. It is the switch that
decides who reaches the staffing roster: the Roster panel's **From roster** option only
pulls active associates. A status set here survives re-uploads — the spreadsheet never
un-deactivates someone.

### Profile

Clicking a name opens that associate's page: start date, CRM number, phone, background,
language, recruiter, notes, the days they're scheduled for, and their attendance week by
week with an all-time tally at the top.

## Attendance

The **Attendance** tab is one week at a time, keyed by the week ending (Sunday).
Present/absent comes off the saved staffing sheets rather than being typed in. A day is
only worked out once a staffing sheet exists for it — until then it stays blank, so an
upcoming week doesn't read the roster's scheduled-days pattern as a wall of absences:

- No staffing sheet saved for that day yet → blank (`—`), and the column header says
  *no sheet*
- Sheet saved, on a line that day → **present**
- Sheet saved, scheduled that day but not placed → **absent**
- Sheet saved, not scheduled → `off`

Confirmations and hand-set overrides work on any day, sheet or no sheet, so you can log
who confirmed and record a known PTO day before the shift is built.

Use ⇄ on a cell to override it by hand and ↺ to hand it back to the sheet. Absences take
a reason — **Call-Off**, **NCNS**, **Excused** or **PTO** — and every associate has a
free-text note for the week.

The **C** column is the confirmation the site used to track with an X on the roster
spreadsheet. The strip across the top turns it into a metric: confirmed, showed, show
rate, and confirmed no-shows. **Export Week** writes the whole grid to `.xlsx`.

## Scorecard

The **Scorecard** tab reproduces one week block of the Korpack scorecard — the KPI panel
on the left, a column per working day, and a week TOTALS column.

| Row | Where it comes from |
| --- | --- |
| Required | Every spot created on that day's lines |
| Working - Direct Only | How many of those spots were filled |
| Confirmed | Confirmations logged on the Attendance tab |
| New Starts, Lines Cut | The staffing sheet |
| Variance, Daily Direct People, Daily Fill % | Calculated |
| Forecasted, Sent Home, 2nd Day Returns, Surveys Completed, Early Leaves, DNRs | Editable fields |

Rows marked `AUTO` fill themselves in and can still be typed over; the ↺ that appears
hands the cell back. A day with no saved sheet and no confirmations stays blank rather
than reading as a zero, so it doesn't drag the averages down.

**Setup** sets the site name, the shift label, and which days get a column (Thu/Fri/Sat/Sun
by default).

**Export XLSX** writes the week in the workbook's own geometry — KPI panel in A/B, row
labels in C, days in D onward, totals in the last column, same blank spacer rows — plus a
flat `Scorecard_Flat` sheet for pivoting. **Export CSV** writes the flat version alone.

## Email Prep

**Email Prep** on the Staffing toolbar builds the day's headcount-confirmation email from
the live board, following the layout of the roster workbook's *Email HC Confirmation*
tab: total headcount, then ORDERED / FILLED / NOT FILLED / Comments per line, then the
totals row, with the day's confirmed-vs-showed underneath.

Each line takes a description (e.g. `Chomps - Auto`) that saves onto the line, so it
carries over to the next day's email. Comments are per-send. Recipients, site name and a
subject template (`{site} {day} {m}/{d} attendance`) save as defaults.

**Copy Body** copies the table as rich HTML where the browser allows it, so it pastes
into Outlook as a table rather than as runs of spaces; **Open in Mail** hands the whole
thing to your mail client.

> Recipients start empty. Paste the distribution list in once and **Save as default** —
> it isn't baked into the source.

## Feature flags

Three pages from the original are hidden by default. To bring one back, flip its flag
near the top of the `<script type="text/babel">` block in `index.html`:

```js
const FEATURES = {
    analytics: false,   // Analytics tab
    rosterAdmin: false, // "Roster & Badge Check" tab (badge scan, DNR/status upload)
    activity: false,    // Activity / audit-log tab
};
```

`rosterAdmin` also controls the two **Scan Badge** buttons on the Staffing screen.

The code for these pages is still present, so turning a flag on is all that's needed.

## Firebase

Cloud sync uses Firebase Realtime Database plus Email/Password auth, against project
**`lineplanner-7a8af`**. The config is already in `index.html`.

If Firebase can't be reached the app falls back to `localStorage` and keeps working
offline; a yellow banner tells you when that's happening.

### About the API key in `index.html`

GitHub secret scanning flags the `AIza…` string in `index.html` as a Google API key.
**It is meant to be public** — a Firebase web app cannot work without shipping its
config to the browser, so rotating it accomplishes nothing; the replacement would be
equally visible. The key is also restricted to Firebase APIs only, so it can't be used
against other Google services.

What the public key *does* allow is creating an account in this project. So being
signed in cannot be the thing that grants access to data — see below.

### Access control

`database.rules.json` grants access to a signed-in user who **has a verified
`@employbridge.com` email** or **whose UID is listed under `/allowedUsers`**. A stranger
can still register an account, but it gets them nothing: every path denies them.

**@employbridge.com teammates need no approval.** They sign up in the app, get a
verification email, click the link, and are in. Until they verify, the app shows a
"Verify your email" screen instead of the planner. Verification is required because
Email/Password sign-up doesn't prove someone owns the address they typed. The domain
lives in two places that must match: the rules and `COMPANY_EMAIL_DOMAIN` in `index.html`.

To add anyone outside that domain, have them sign up in the app first, then run:

```sh
echo 'true' | firebase database:set /allowedUsers/<their-uid> --project lineplanner-7a8af
```

Find UIDs in the Firebase console under **Authentication → Users**. To revoke someone,
delete their `/allowedUsers` entry and their account (for a company-domain user,
deleting or disabling the account is what revokes access).

`/allowedUsers` is not writable by app users at all — only via console/CLI, which bypass
rules. That means an approved user cannot approve anyone else.

To re-publish rules after editing:

```sh
firebase deploy --only database
```

### Worth doing

- **Firebase App Check** if you want requests attested as coming from your real app.

### Setup status

Verified end to end: sign-up, cloud save, and read-back all work. Realtime Database
`lineplanner-7a8af-default-rtdb` is live, Email/Password sign-in is enabled, and the
allowlist rules are deployed and confirmed — anonymous denied, non-allowlisted accounts
denied, allowlisted accounts read/write.

> Note: if a database instance is ever missing or unreachable, saving a new shift hangs
> on the Setup screen rather than failing — the app awaits a database read that never
> resolves.

### Accounts

The login screen supports sign-in, self-service sign-up, and password reset. Auth is
only enforced when the Firebase SDK loads and the config is valid; otherwise the app
runs ungated in local-only mode.

## Deploying

The app is a static file, so anything that serves HTML works. With Firebase Hosting:

```sh
firebase deploy --only hosting
```

## Data layout

Realtime Database nodes:

| Node | Contents |
| --- | --- |
| `staffing/<date>_<shift>` | One saved shift: lines, positions, roster panel, indirect slots |
| `roster/associates` | Master associate list: status, scheduled days, start date, contact and recruiting fields |
| `attendance/<weekEnding>` | One week of attendance: per-day status, reason and confirmation, plus weekly notes |
| `scorecard/<weekEnding>` | Scorecard entry for a week: manual fields and overrides of derived rows |
| `settings/planner` | Scorecard setup (site, shift label, days tracked) and Email Prep defaults |
| `settings/coreAssociates` | Core team members and per-lead notes |
| `activity` | Audit log (written even when the Activity tab is hidden) |

## What was removed from the original

- **Crescent staff** — the supplemental-agency position type, its pink styling, the
  "Convert" button, and the associated `crescent` core-team flag. Every position is now
  a normal slot that counts toward the line's fill.
- **Gmail / Power Automate roster sync** — replaced by the manual **Load** flow and the
  Associates tab's roster upload.
- **Analytics, Roster & Badge Check, and Activity tabs** — hidden behind feature flags.
- The **Waitlist** panel is now the **Roster** panel.
