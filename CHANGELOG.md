# Changelog highlights

A curated subset of recent releases — focuses on the work that
shaped the product, not every version-bump. Specific version numbers
omitted because they're business-internal.

## Release theme — Audit pass (most recent)

Five-group structured audit and remediation, prioritised by data-loss
risk and visible business correctness:

### Group A — Data correctness
- Fixed archive view that was showing `0.00 ha` on hectares tasks
  with a volume-proportional allocation (real number visible in the
  live tab) — the call site was reading per-entry `units` directly
  instead of running the volume-prop derivation.
- Extracted `allocateTaskUnits(tk)` as the single source of truth for
  the hours-share allocation rule; removed five inline copies of the
  same loop across `calcSal`, `calcSalAll`, `calcTaskFinance`,
  `expInvoice`, `expTaskSum`, and the close-task preview / commit.
- Module constant `VAT_RATE` replaces five hard-coded `0.255` /
  `1.255` literals.
- JSON re-import now runs `validateTaskForInvoice` over imported
  tasks; a confirmation modal lists offending palstas and lets the
  director keep only valid tasks.
- Local disk backup writes the current schema version instead of a
  stale hard-coded one — re-imports correctly trigger migrations.

### Group B — Backup hardening
- PIN-hash check before writing snapshots — plaintext PIN (pre-migration
  legacy) is omitted from Firebase backups and disk backups, removing a
  potential leak via the foreman-readable `/backups` path.
- `_lastBackupDate` persisted across page reloads — a director who
  refreshes mid-day no longer triggers extra daily Firebase writes.
- Pre-restore confirmation modal: timestamp + type pill (Auto /
  Manual), per-entity diff table (current → backup, Δ), orange
  warning with the count of file attachments that won't be restored
  (snapshots strip files to keep payloads small). PIN modal opens
  only after the user confirms; restore commit is audit-logged.

### Group C — Excel exports
- All nine legacy HTML-as-XLS exports (per-employee accounting,
  all-employees accounting, detailed salary, ISTU government,
  timesheet, work-types reference, client-rates summary, per-client
  price sheet, salary detail per employee) routed through `dlXlsx`
  → real OOXML `.xlsx`. Excel no longer warns about format mismatch;
  the bookkeeper no longer has to round-trip every file through
  Save-As.
- Styling preserved automatically via an HTML-to-styled-XLSX bridge
  that reads inline `bgcolor`, `color`, `font-weight`, `font-size`,
  `text-align`, `x:num`, `colspan`/`rowspan`, and `colgroup` widths
  and re-applies them as SheetJS cell styles. No per-export rewrite
  needed.

### Group D — Profit accounting
- `calcTaskFinance.payroll` extended to include all per-task
  attributable allowances: base earnings + istupalkaa (mileage
  over per-employee threshold) + driver km pay + trailer pay +
  passenger pay. Period-level items (manual bonus, manual penalty,
  monthly foreman bonus) intentionally stay out — they're not
  attributable to a specific task.
- Return shape gains `payrollBreakdown` so the task-card profit
  chip's tooltip can list each component; future profitability
  dashboard can aggregate by period.
- Seven unit tests added covering each allowance category.

### Group E — Backend defense
- Per-field `.validate` rules in Firebase RTDB across `data.tasks`,
  `data.employees`, `data.clients`, `data.owners`, `data.workTypes`,
  `data.groupInvoices`. Type guards, reasonable numeric ranges,
  enum membership for categorical fields (`status`, `difficulty`,
  workType `type`). Rules are designed to be backward-compatible —
  every constraint allows empty strings or absent fields so legitimate
  bulk writes of legacy data don't break.

## Earlier feature waves

### Group invoices
- Bulk task selection on the closed-tasks list with per-group
  visibility (open / closed / invoiced) and a same-client guard.
- Group-invoice preview modal, save, view, revoke flow with
  director-only PIN gate, per-task `extract` action so a single
  task can be pulled out of a bundle without re-issuing the whole
  invoice.
- Visual continuity on the task card — adjacent same-group tasks
  share a thick navy-blue left border and lose vertical margin so
  the column reads as one block.
- Pre-revoke confirmation lists every palsta that will be returned
  to "closed — awaiting invoice".

### Auto-backup system
- Four event triggers (task create / day added / task close /
  bulk close) plus the daily-auto fallback and the manual button.
- Typed retention buckets — manual ∞, daily-auto 14, event-auto 60.
- Shallow REST scan categorises keys by prefix so rotation reads
  only key names, not payloads.
- Surfaces backup-write failures via toast + audit log so a silent
  miss can't be mistaken for success.

### Time-entry UX
- Replaced native `<input type="time">` with a custom HH:MM-masked
  text input. Type `1645⏎` and the field becomes `16:45`, focus
  jumps to the next input.
- Auto-advance on the 4th digit; smart padding on commit (`9` →
  `09:00`, `930` → `09:30`).
- Clamp validation: hours capped at 23, minutes at 59 — invalid
  input is corrected, never silently saved.
- Safari fix — `setTimeout(0)` defers `select()` past the browser's
  click handling so the user can click → type without first having
  to drag-select the existing value.

### Validation
- Mandatory frontend validation on save — palsta, client, owner,
  and `volume > 0` for hectares tasks. Invalid fields get a red
  border + focus ring; the toast lists every missing field in one
  message.
- Invoice generation re-runs the validator against the stored task
  so an old export that pre-dates form validation can't quietly
  produce a malformed invoice. The bulk-invoice flow lists the
  offending palstas before letting the director continue.

### Excel for customer-facing invoices
- Single-task and group invoices rebuilt with a programmatic styled
  builder — METSÄTRACKER green band, blue subtitle, dark-blue ERITTELY
  header, dark-blue table header, alternating row stripes, e8f5e9
  totals row, currency number format. Files open cleanly in
  Excel / Numbers / Mail preview.

### Work-type catalogue
- Each work-type gains a customer-recognisable invoice code (e.g.
  `603 — Varhaisperkaus`). Codes seeded in defaults + a schema
  migration backfills existing data.
- `getWtName(w, lang)` is the single rendering helper used across
  task forms, customer rates, dashboards, archive, and exports.

### Day-level kilometre tracking
- Moved km from a per-task `distanceKm` (with a legacy ×2 multiplier
  for round trip) to a per-day `dy.km` field entered from the
  odometer reading. Old per-task field stays in storage but is
  ignored by the new code paths.
