# Power Automate → Line Planner roster sync

Keeps the Line Planner roster in step with the recruiting spreadsheet automatically.
When someone's Column A dropdown is switched to **Active**, the change lands in the tool
without anyone opening it.

This is a draft to build from, not an exported package. Every action, expression and URL
below is copy-paste ready, but you'll need to point the connectors at your own file and
create one service account.

---

## What it does

```
Roster workbook saved in SharePoint/OneDrive
            │
            ▼
  [1] Trigger — file modified (+ hourly safety net)
  [2] Sign in to Firebase          → idToken (valid 1 hour)
  [3] List rows from the Excel table
  [4] Keep rows with a name and a CRM #
  [5] Map each row to a patch
  [6] PATCH one node per associate → roster/associates/<CRM #>
            │
            ▼
   Line Planner picks it up on next load
```

The flow writes the same fields the tool's own **Upload Roster Spreadsheet** button
writes, into the same place, in the same shape. Both paths stay usable — the manual
upload remains the reconciliation tool (see [Known gaps](#known-gaps)).

---

## Before you start

### 1. Licensing

The flow uses the **HTTP** action, which is a **premium** connector. You need Power
Automate Premium (per-user or per-flow). There is no non-premium path to an arbitrary
REST API — the standard connectors can't reach Firebase.

If premium isn't available, the fallback is an Azure Function that does steps 2 and 6,
called from a standard connector. That's a bigger build; ask and I'll draft it.

### 2. Make the roster range an Excel Table

The Excel connector can only read **named tables**, not raw sheet ranges. The roster
sheet has a title and a leads line above the headers, so:

1. Open the workbook in Excel.
2. Select `A3` through the last column and row of the roster (headers **and** data —
   the header row is the one starting with **Status**, **Employee Name**).
3. **Insert → Table**, tick *My table has headers*.
4. **Table Design → Table Name**: `RosterTable`.

Keep the section rows (`HEADCOUNT SCHEDULED`, `UNDER(-)/OVER`, `Pending I-9`, …) *out*
of the table if you can. If they're inside it, step 4 filters them out anyway.

> Adding a table doesn't change how the sheet looks or behaves for the people using it.

### 3. Create a service account for the flow

The database rules are `auth != null`, so the flow has to sign in as somebody. Don't use
a person's account.

1. Firebase console → **Authentication → Users → Add user**.
2. Email `lineplanner-sync@yourdomain.com` (it never receives mail), and a long random
   password.
3. Store the password in **Azure Key Vault** and read it with the Key Vault connector,
   or at minimum keep it in a flow variable and restrict who can edit the flow.

---

## Flow reference

Create a new **Automated cloud flow**.

### [1] Trigger

**SharePoint — When a file is modified (properties only)**

| Field | Value |
| --- | --- |
| Site Address | the site holding the roster |
| Library Name | e.g. `Documents` |

Then a **Condition**: `File name with extension` **is equal to**
`Korpack_Recruting_Roster.xlsx` — so edits to other files in the library don't fire it.

> Use **OneDrive for Business — When a file is modified** instead if the workbook lives
> in someone's OneDrive rather than a SharePoint library.

**Add a second flow** on a **Recurrence** trigger (every 1 hour) running the same steps
2–6. The file-modified trigger can be slow or miss a save when several people are in the
workbook at once; the hourly run guarantees the tool is never more than an hour stale.
Everything below is idempotent, so running twice costs nothing.

### [2] Sign in to Firebase

**HTTP** — rename the action to `Firebase_sign_in` (the expressions below use that name).

| Field | Value |
| --- | --- |
| Method | `POST` |
| URI | `https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=AIzaSyCGuuwSjRyU-p_vXBehE5oVEjPug2MFZpw` |
| Headers | `Content-Type` : `application/json` |

Body:

```json
{
  "email": "lineplanner-sync@yourdomain.com",
  "password": "@{variables('FirebasePassword')}",
  "returnSecureToken": true
}
```

The token comes back as `body('Firebase_sign_in')?['idToken']` and is good for one hour —
plenty for a single run.

> The `key=` value is the app's public Firebase Web API key. It's already in `index.html`
> and is designed to be public; it identifies the project, it doesn't grant access. The
> password is the secret.

### [3] Read the sheet

**Excel Online (Business) — List rows present in a table**

| Field | Value |
| --- | --- |
| Location / Document Library | wherever the workbook lives |
| File | `Korpack_Recruting_Roster.xlsx` |
| Table | `RosterTable` |

Open **Advanced options** and set **DateTime Format** to `ISO 8601`. That makes the
Start Date column come back as a real date string instead of an Excel serial number.

If the roster grows past 256 rows, also set **Top Count** to `5000` and turn on
pagination.

### [4] Drop the rows that aren't people

**Filter array**

- **From**: `@outputs('List_rows_present_in_a_table')?['body/value']`
- **Condition** (switch to advanced mode):

```
@and(
  not(empty(trim(coalesce(string(item()?['Employee Name']), '')))),
  not(empty(trim(replace(replace(coalesce(string(item()?['CRM #']), ''), decodeUriComponent('%0A'), ''), decodeUriComponent('%0D'), ''))))
)
```

This keeps rows that have both a name and a CRM number. The section-label rows have
neither, so they fall out here.

**Rows without a CRM number are skipped** — see [Known gaps](#known-gaps).

### [5] Map each row to a patch

**Select** — rename to `Build_patches`. Switch the **Map** box to text mode (the icon on
the right of the field) and paste:

```
{
  "key": "@{trim(replace(replace(replace(coalesce(string(item()?['CRM #']), ''), decodeUriComponent('%0A'), ''), decodeUriComponent('%0D'), ''), ' ', ''))}",
  "body": {
    "employeeId": "@{trim(replace(replace(replace(coalesce(string(item()?['CRM #']), ''), decodeUriComponent('%0A'), ''), decodeUriComponent('%0D'), ''), ' ', ''))}",
    "crmNumber": "@{trim(replace(replace(replace(coalesce(string(item()?['CRM #']), ''), decodeUriComponent('%0A'), ''), decodeUriComponent('%0D'), ''), ' ', ''))}",
    "fullName": "@{trim(replace(coalesce(string(item()?['Employee Name']), ''), '*', ''))}",
    "firstName": "@{first(split(trim(replace(coalesce(string(item()?['Employee Name']), ''), '*', '')), ' '))}",
    "lastName": "@{trim(substring(trim(replace(coalesce(string(item()?['Employee Name']), ''), '*', '')), length(first(split(trim(replace(coalesce(string(item()?['Employee Name']), ''), '*', '')), ' ')))))}",
    "status": "@{if(equals(toLower(trim(coalesce(string(item()?['Status']), ''))), 'active'), 'active', 'inactive')}",
    "isActive": @{if(equals(toLower(trim(coalesce(string(item()?['Status']), ''))), 'active'), true, false)},
    "scheduledDays": @{json(concat('[', replace(replace(concat(
        if(contains(toLower(coalesce(string(item()?['THUR']), '')), 'x'), '4,', ''),
        if(contains(toLower(coalesce(string(item()?['FRI']),  '')), 'x'), '5,', ''),
        if(contains(toLower(coalesce(string(item()?['SAT']),  '')), 'x'), '6,', ''),
        if(contains(toLower(coalesce(string(item()?['SUN']),  '')), 'x'), '0,', ''),
        '#'), ',#', ''), '#', ''), ']'))},
    "startDate": "@{if(empty(trim(coalesce(string(item()?['Start Date:']), ''))), '', formatDateTime(item()?['Start Date:'], 'yyyy-MM-dd'))}",
    "phone": "@{trim(coalesce(string(item()?['Phone Number']), ''))}",
    "background": "@{trim(coalesce(string(item()?['Background']), ''))}",
    "language": "@{trim(coalesce(string(item()?['language ']), ''))}",
    "last4": "@{trim(coalesce(string(item()?['last 4 SSN']), ''))}",
    "recruiter": "@{trim(coalesce(string(item()?['Recruiter']), ''))}",
    "assigned": "@{trim(coalesce(string(item()?['Assigned']), ''))}",
    "notes": "@{trim(coalesce(string(item()?['Notes']), ''))}",
    "source": "power-automate",
    "updatedAt": "@{utcNow()}"
  }
}
```

**From**: `@body('Filter_array')`

Notes on the expressions:

- **Column names must match your headers exactly**, trailing spaces included. The current
  file has `language ` and `Start Date:` with the punctuation shown. Check the **List
  rows** output in a test run and correct any that differ.
- **`*` stripping** — the recruiters' `*` / `**` markers are removed so names match what
  the tool already stores. It strips asterisks anywhere in the name, which is fine for
  this data.
- **`scheduledDays`** — a day counts when its cell contains an `x` (either case). The
  `#` sentinel is a trick to drop the trailing comma; `[]` comes out for a row with no
  marks. Day numbers are `0`=Sunday … `6`=Saturday, which is what the tool stores.
- **`startDate`** — assumes step 3's ISO 8601 setting. If the output still shows a number
  like `46280`, use this instead:
  `formatDateTime(addDays('1899-12-30', int(item()?['Start Date:'])), 'yyyy-MM-dd')`
- **`status`** — anything that isn't exactly `Active` becomes `inactive`. That's
  deliberate: a typo or a blank shouldn't silently read as active. Set Excel data
  validation on Column A to just `Active` / `Inactive` so it can't drift.

### [6] Write to the tool

**Apply to each** over `@body('Build_patches')`. In **Settings**, turn on **Concurrency
Control** and set the degree of parallelism to **20** — ~90 rows then finish in seconds.

Inside it, one **HTTP** action:

| Field | Value |
| --- | --- |
| Method | `PATCH` |
| URI | `https://lineplanner-7a8af-default-rtdb.firebaseio.com/roster/associates/@{item()?['key']}.json?auth=@{body('Firebase_sign_in')?['idToken']}` |
| Headers | `Content-Type` : `application/json` |
| Body | `@item()?['body']` |

`PATCH` on a child path is a **field-level merge**. It updates only the fields in the
body and leaves everything else on that associate untouched — `createdAt`, `lastLine`,
`lastShift`, and anything else the tool owns. That's why this is safe to run a dozen
times a day.

---

## Testing it

1. **Run step 2 alone first.** A `200` with an `idToken` means the service account works.
   A `400` with `INVALID_LOGIN_CREDENTIALS` means the email or password is wrong;
   `EMAIL_NOT_FOUND` means the user was never created.
2. **Run steps 3–5 with step 6 disabled.** Look at the `Build_patches` output and check
   one row against the spreadsheet — name clean, CRM as the key, `scheduledDays` matching
   the x's, `status` right.
3. **Enable step 6 and flip one person.** Switch a single row's Column A to the other
   value, save the workbook, wait for the run, then open the tool's **Associates** tab
   and confirm that person moved. The tool reads the roster when the page loads, so
   refresh it.
4. **Run it twice with no edits.** Nothing should change — the second run writes the same
   values over the same values.

---

## Known gaps

**Rows without a CRM number are skipped.** The tool keys an associate on their CRM
number, and falls back to a name-derived key only in its own upload path — that fallback
needs a token sort Power Automate expressions can't do. In the current file 92 of 93 rows
have a CRM number, so this affects almost nobody, but give everyone one and the flow
covers the whole roster.

**A person who gains a CRM number later can double up.** If someone was created in the
tool by a manual upload while their CRM cell was blank, they're stored under a
name-derived key (`N-…`). Once the sheet gets their CRM number, the flow writes them
again under the numeric key and you'll see them twice. Fix: run **Upload Roster
Spreadsheet** in the app once — it matches on name and folds the two together. Better:
fill in CRM numbers before anyone is added to the tool.

**The flow doesn't read the NCNS / Terminate / Declines tabs.** The manual upload does,
and flips those people to inactive. The flow relies on Column A instead, which is the
more deliberate signal. If you want the exit tabs covered too, make each one a named
table and add a parallel branch that PATCHes `{"status":"inactive","isActive":false}` —
but note that Column A saying `Active` beats a stale exit-tab entry in the app, and the
flow would not honour that precedence.

**The flow never deletes anyone.** Same as the manual upload. Someone removed from the
spreadsheet stays in the tool at whatever status they last had. Mark them `Inactive` in
Column A rather than deleting the row.

**`scheduledDays` is overwritten, not merged.** If a row's day cells are all blank the
flow writes `[]`, clearing that person's scheduled days. The manual upload keeps the
previous value instead. The flow's behaviour treats the sheet as the source of truth,
which is usually what you want — just be aware they differ.

---

## Security notes

- The database rules are currently `".read": "auth != null"` and `".write": "auth != null"`.
  **Any** signed-in account can read and write **everything**, including the sync account.
  That's the existing posture, not something this flow introduces, but it's worth
  tightening — per-node rules would let the sync account write only `roster/associates`.
- The Firebase Web API key in step 2 is public by design and safe in the flow definition.
  The **password is not** — keep it in Key Vault, or at least out of run history by
  marking the HTTP action's inputs as secure (**Settings → Secure Inputs**).
- Turn on **Secure Inputs** and **Secure Outputs** on step 2 regardless, so the token
  doesn't sit in run history where anyone with flow access can read it.
