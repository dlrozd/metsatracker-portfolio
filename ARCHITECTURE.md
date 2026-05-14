# Architecture

High-level view of how the production vanilla codebase is wired together.

## Goals that shaped the design

1. **Single-file deploy.** The whole app ships as one HTML file. No
   bundler, no module loader at runtime, no service-worker juggling.
   Reduces the operational surface to zero between releases.
2. **Live multi-device sync.** Director on laptop, foreman on phone in
   the field — both must see the same state with ~500ms latency.
3. **Offline-tolerant.** Field connection is patchy. localStorage cache
   + optimistic local writes; Firebase syncs when network returns.
4. **Director-data-loss-proof.** Multiple backup layers: per-event
   snapshots in Firebase, daily auto in Firebase, manual on demand,
   plus a disk auto-backup via File System Access API.
5. **Two human roles** — director (full control) and foreman (read +
   constrained write). Enforced by Firebase rules, not the UI.

## Module layout (`src/js/*`)

```
i18n.js          — RU/FI string table, t() lookup, language detection
app.js           — globals, lifecycle, login, schema migrations
data.js          — sv()/load, Firebase wiring, backup orchestration,
                   client/owner/season/period CRUD
firebase.js      — _applySnap, _fbSkip write-loop guard
calc.js          — cH(), getWtRate(), allocateTaskUnits(), calcSal(),
                   calcSalAll(), calcTaskFinance() — single source of
                   truth for all money math
localBackup.js   — disk backup via FSA + IDB handle
ui/modal.js      — sm()/hm() modal system, toast
ui/employees.js  — employee CRUD UI
ui/tasks.js      — task list, day editing, time-mask input,
                   close-task flow, validation
ui/dashboard.js  — period dashboard
ui/archive.js    — closed-period archive
ui/settings.js   — settings, backup list, rate management
ui/salary.js     — salary modal, manual bonus/penalty
ui/tools.js      — saws / tools maintenance tracker
export/excel.js  — dlXlsx() + _xsBuildSheet styled builder,
                   accounting reports
export/detailed.js — detailed salary reports
export/istu.js   — ISTU government mileage report
export/timesheet.js — timesheet report
export/invoice.js  — single + group invoices, rate sheets,
                     validateTaskForInvoice()
```

`build.js` concatenates these in strict order (i18n first, exports
last), strips file-header comments, wraps everything in an IIFE,
exposes a curated list of functions to `window` so HTML `onclick`
handlers can reach them, and validates the JS syntax before writing
the bundle.

## Data flow

```
user action
   │
   ▼
mutate globals (emps, tasks, wts, pers, sns, clients, owners,
                groupInvoices, tools, toolCycles)
   │
   ▼
sv()  ───────────────► localStorage cache
   │ (debounced 500ms)
   ▼
Firebase RTDB .update(/data, { tasks, employees, ... })
   │
   ▼
all other clients receive a snapshot →
   ▼
_applySnap(fd)  ────► sets _fbSkip = true, applies, re-renders, clears
```

The `_fbSkip` flag stops `sv()` from echoing a remote snapshot back to
Firebase. `sv()` is the only write path the app uses; everything else
goes through it.

## Backup model

Three retention buckets, each with its own key prefix so a shallow
REST scan can categorise without downloading payloads:

| Type        | Key format                       | Retention | Trigger                                          |
| ----------- | -------------------------------- | --------- | ------------------------------------------------ |
| manual      | `YYYY-MM-DD_HHMMSS`              | ∞         | director clicks "Create backup"                  |
| auto-daily  | `YYYY-MM-DD`                     | 14        | first `sv()` per day per director session        |
| auto-event  | `auto_YYYY-MM-DD_HHMMSS`         | 60        | task created / day added / task closed / bulk    |

Snapshot payload: full state (employees, tasks, workTypes, periods,
seasons, clients, owners, tools, toolCycles, groupInvoices) plus
optional hashed PINs and a `meta` block describing trigger and
affected entity. File attachments are stripped before write to keep
payloads small — the restore preview warns about that explicitly.

Restore flow: read the snapshot, show a confirmation modal with a
diff table (current → backup, Δ per entity) and a file-loss warning,
then a director-PIN modal, then commit.

## Calculation pipeline

```
allocateTaskUnits(tk)
    │  returns { totH, totU, perEmp:{eId:{hours,units}}, usesVolProp, st }
    ▼
calcTaskFinance(tk)  ─► invoice + payroll figures + payrollBreakdown
calcSal(eId, period) ─► one employee's salary for the period
calcSalAll(period)   ─► every employee's salary
expInvoice / expTaskSum / expAcct / ...  consume these
```

`allocateTaskUnits` is the single place where the
"hectares = hours × volume ÷ total_hours" rule lives. It used to be
duplicated across five callers before consolidation — drift risk
removed.

## Security model

- **Firebase Auth (Google Sign-In)** with custom claims for `role`
  (`director` or `foreman`) and an email allowlist for the directors.
- **RTDB rules:**
  - `data.*` readable + writable by both roles (the foreman needs to
    log hours from the field).
  - `data.directorPin` / `data.foremanPin` writable only by directors.
  - `backups.*` writable only by directors; readable by both
    (foreman uses the list for visibility; PIN hashes are still safe
    via PBKDF2).
  - `audits.*` writable only on insert (never overwrite); readable by
    directors.
  - **`.validate` rules** under `data/{tasks,employees,clients,owners,
    workTypes,groupInvoices}` enforce types and reasonable ranges as
    defense-in-depth against direct REST writes that bypass the UI.
- **PIN secrets** stored as `{ v, salt, hash, iter }` PBKDF2 records,
  not plaintext. Schema migration upgrades any legacy plaintext PINs
  on first director login. Backups skip the PIN field entirely if it
  somehow hasn't been migrated yet.
- **Validation** lives in three places: the form save path
  (`_validateTaskFields`), the invoice generation path
  (`validateTaskForInvoice`), and the RTDB `.validate` rules.

## Excel export pipeline

Two patterns coexist:

1. **Programmatic styled builder** (`_xsBuildSheet`) — used by the
   customer-facing invoice, the group invoice, and the task summary.
   Builds the worksheet cell-by-cell from a row description so the
   layout is fully under code control (brand band, ALV totals row,
   alternating row stripes, currency number format).
2. **HTML-to-styled-XLSX bridge** (`dlXlsx`) — used by the legacy
   reports (accounting, ISTU, timesheet, detailed). Parses the existing
   HTML strings with `DOMParser`, walks the table reading inline
   styles (`bgcolor`, `color`, `font-weight`, `font-size`,
   `text-align`, `x:num`), and produces a `xlsx-js-style` worksheet
   with the same visual styling. Saves rewriting nine large
   report-builders.

Both pipelines emit real OOXML so Excel opens the files without
"format does not match extension" warnings.

## Tooling and release flow

- `node build.js` — produces `dist/puutahti.html` and validates JS
  syntax. Run before every commit.
- `node test.js` — 86-assertion unit suite over the calculation layer
  and helpers. Runs in <1s.
- `firebase deploy --only hosting` — production deploy.
- Version bumped in six files (`sw.js`, `index.html`, `styles.css`
  banner, `build.js` title, `ui/settings.js` footer, `package.json`).
- Roughly weekly release cadence; 60+ tagged releases.
