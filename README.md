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

**Load** opens a modal with two ways to bring in a roster:

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

### Setup status

Setup is complete and verified end to end — sign-up, cloud save, and read-back all work:

- Web app registered, config baked into `index.html`.
- Realtime Database `lineplanner-7a8af-default-rtdb` created.
- Email/Password sign-in enabled.
- Rules live and confirmed: anonymous access denied, signed-in read/write allowed.

`database.rules.json` in this repo matches what's live. To re-publish it after an edit:

```sh
firebase deploy --only database
```

> Note: if a database instance is ever missing or unreachable, saving a new shift hangs
> on the Setup screen rather than failing — the app awaits a database read that never
> resolves.

### Accounts

No accounts exist yet — the first person to use the app creates one from the login
screen. Anyone can self-register; to keep it closed, create accounts yourself in the
console and lock sign-up down there.

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
| `roster/associates` | Master associate list used for name autocomplete |
| `settings/coreAssociates` | Core team members and per-lead notes |
| `activity` | Audit log (written even when the Activity tab is hidden) |

## What was removed from the original

- **Crescent staff** — the supplemental-agency position type, its pink styling, the
  "Convert" button, and the associated `crescent` core-team flag. Every position is now
  a normal slot that counts toward the line's fill.
- **Gmail / Power Automate roster sync** — replaced by the manual **Load** flow.
- **Analytics, Roster & Badge Check, and Activity tabs** — hidden behind feature flags.
- The **Waitlist** panel is now the **Roster** panel.
