# Screenshots

Drop UI screenshots into this folder. Suggested set (and why each one
sells the project):

| Filename | Shows | Why it matters |
|---|---|---|
| `01-dashboard.png` | Period dashboard with employee earnings + summary cards | Hero shot — first thing a reviewer sees |
| `02-task-card.png` | Closed task with the finance chip (💶 net/gross + 💰 profit) | Demonstrates per-task financial visibility + group-invoice integration |
| `03-day-editor.png` | A day's entries with start/end/break inputs | Shows the keyboard-only HH:MM time entry |
| `04-group-invoice.png` | Group invoice preview with banner + line table | Customer-facing real .xlsx + multi-task bundling |
| `05-invoice-excel.png` | Generated `.xlsx` opened in Excel/Numbers | Proves styled output, no format-mismatch warning |
| `06-backup-restore.png` | Pre-restore preview modal with diff table | Disaster recovery story |
| `07-archive.png` | Closed-period archive with totals | Long-term recordkeeping |
| `08-settings-rates.png` | Client rates / work types with codes | Shows pricing flexibility |

## Anonymisation checklist

Before committing screenshots, check each frame for:

- [ ] Real employee names → mask with a redaction tool or replace with
      `Employee 1`, `Employee 2`. Names are personal data under GDPR.
- [ ] Real client names (other than Puutähti Oy itself) → mask or
      replace with `Client A`, `Client B`.
- [ ] Real owner names (forest plot owners — `Erikoissijoitusrahasto …`,
      `Mielikki Silva Ky`, etc.) → mask or replace.
- [ ] Real palsta IDs → can stay (they're public cadastral references).
- [ ] Real Euro amounts → can stay if they're reasonable order of
      magnitude; mask if they reveal confidential pricing.
- [ ] Real km / hours / hectares → fine as-is.
- [ ] Browser tabs, system tray, notification badges that leak
      anything sensitive → crop.
- [ ] Director email / Google avatar in the top bar → crop or mask.

Crop close to the relevant UI; full-screen shots add noise.

PNG > JPG for UI screenshots. Keep file size under ~500 KB each — if
they're bigger, scale the screenshot to 1600px wide before saving.
