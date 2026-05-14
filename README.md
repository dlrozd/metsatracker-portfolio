# MetsäTracker — portfolio summary

> Production work-time, payroll & invoicing system for **Puutähti Oy** — a
> Finnish forestry company. Used daily by the team (director + foreman +
> field workers). 
> Used by the team during the current forestry season.
> Live: <https://metsatracker.web.app>

This repository is a **public summary** of the project — it documents the
scope, architecture, tech stack, and my role. The source code lives in a
private repository because it contains business logic, payroll rates, and
customer references.

---

## What it does

A single web app that runs the day-to-day operations of a forestry crew:

- **Work tracking** — per-day, per-employee start/end/break time, kilometres,
  driver/passenger flags, work-type and difficulty per task.
- **Multi-component payroll calculation** — base earnings (qty × rate),
  *istupalkaa* mileage allowance (km over threshold), driver km pay, trailer
  pay, passenger pay, plus per-period manual bonuses/penalties and a
  foreman's monthly bonus.
- **Hectares-proportional allocation** — when a task records the total
  area and per-employee hours, each employee's hectares share is derived
  from their share of working hours.
- **Invoicing** — single-task invoices and **group invoices** that bundle
  multiple closed tasks for the same client. Each invoice line stores a
  snapshot of rates so historical figures don't drift.
- **Excel exports** (real `.xlsx` via SheetJS / xlsx-js-style):
  invoices for clients, ISTU government report, salary reports for the
  bookkeeper, timesheets, client-rate sheets.
- **Two-tier safety system** — frontend validation on save +
  Firebase Realtime DB `.validate` rules as backend defense in depth.
- **Disaster recovery** — typed auto-backup retention (manual ∞ /
  daily-auto 14 / event-auto 60), event triggers on task create / day
  added / task close / bulk close, full pre-restore preview with diff
  table + file-loss warning. Disk auto-backup via the File System
  Access API to a user-chosen folder.
- **Bilingual UI** (Russian / Finnish).

---

## My role

End-to-end developer of the production system, working directly with the
company owner:

- **Architecture** — modular vanilla-JS codebase (19 modules) bundled
  into a single HTML file via a custom Node.js build script; planned
  React + TypeScript + Vite + Supabase rewrite for the 2027 season.
- **All features** listed above — designed, implemented, tested, shipped.
- **Backend** — Firebase Realtime Database schema, security rules with
  per-field `.validate` constraints, PBKDF2 PIN hashing with migration
  from legacy plaintext, role-based access (director / foreman).
- **DevOps** — build pipeline, version management, release cadence
  (60+ tagged releases), Firebase Hosting deploy.
- **Testing** — 86 unit tests covering calc layer, validation,
  hash round-trips, format helpers.
- **Customer-facing work** — gathering requirements from the owner,
  iterating on UX feedback (bookkeeper-pain points, fast morning data
  entry before fieldwork), shipping fixes for real-world friction.

---

## Tech stack

### Currently in production
- **Vanilla JavaScript (ES5+)** — no framework, no npm runtime
  dependencies; bundled into one self-contained HTML file.
- **Firebase Realtime Database** — live sync, custom security rules,
  per-field `.validate` for type/range/enum guards.
- **Firebase Auth** — Google Sign-In with role claims (director /
  foreman).
- **Firebase Hosting** — production at `metsatracker.web.app`.
- **xlsx-js-style** (SheetJS fork) — real OOXML `.xlsx` exports with
  cell styling.
- **File System Access API** + IndexedDB — disk auto-backup to a
  user-chosen folder.
- **Web Crypto API** — PBKDF2 hashing for PIN secrets.
- **Custom build script** (Node.js) — concatenates modules in strict
  order, wraps in an IIFE, exposes a public API for `onclick` handlers,
  validates JS syntax before writing the bundle.
- **Custom test runner** (Node.js `vm` module) — 86 assertions.

### Planned (autumn 2026 → 2027 season)
- **React + TypeScript** (Vite + Tailwind)
- **Supabase** — Postgres + Auth + Storage + RLS
- **xlsx-js-style** carries over
- Migration plan stages live cutover so the current vanilla codebase
  keeps running until the new stack is feature-complete.

---

## Highlights from recent releases

The CHANGELOG file lists the most interesting recent work.
Notable themes:

- **Group invoices** — multi-task bundling for the same client with a
  per-task extract action so a single task can be pulled out without
  re-issuing the whole invoice.
- **Backup hardening** — typed retention buckets (manual / daily-auto /
  event-auto), pre-restore diff preview with file-loss warning,
  PIN-hash check before writing snapshots to a foreman-readable path.
- **Per-task profit chip** with a tooltip breakdown — base / mileage /
  driver / trailer / passenger / total — ready to feed an upcoming
  profitability dashboard.
- **Real-`.xlsx` exports** for all 9 legacy reports — bookkeeper no
  longer has to round-trip files through "Save as".
- **Keyboard-only HH:MM time entry** with auto-advance on the 4th
  digit + Safari focus-race fix.
- **Defense-in-depth validation** — frontend save + invoice gate +
  RTDB `.validate` rules.

---

## Status

Production system in daily use. Roughly weekly release cadence with
incremental fixes and feature additions driven by real-world friction
reported by the team. Source under active development; no plans to
open-source the implementation.
