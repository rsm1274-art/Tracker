# Alert & PEP Compliance Dashboard — Handoff

Status as of this document: **`AlertDashboard_vXX_3_3.html` is the active working version.** Everything below describes what it is, how it got here, and what to know before touching it further.

## What this is

A single-file, offline-capable HTML dashboard for tracking Alert/PEP incident compliance milestones (Action Plan, 3-Month Assessment, 6-Month Assessment, Corrections Required, Extension Applied) against their due dates. No build step, no server, no external network calls — open the `.html` file in a browser and it runs. XLSX parsing (SheetJS) and both typefaces are embedded as base64 inside the file for this reason.

Users import a `.csv` or `.xlsx` export from the source system, and the dashboard pivots it, computes compliance status per record, and presents an executive-facing overview, a sortable/filterable data grid, and a reviewer performance summary. Filtered CSV export, a standalone read-only HTML export, and print/PDF are all built in.

A full walkthrough of every feature — import formats, the five milestones, the overall-status precedence rules, filters, charts, and exports — lives in `Compliance_Dashboard_Manual.html`. Read that before making changes if you didn't write the status logic yourself; it documents exactly what the current code does, not what it's meant to do.

## File inventory

| File | Role |
|---|---|
| `AlertDashboard_vXX_3.html` | Executive visual redesign baseline (new typography, palette, drawn icon set, print stylesheet). Same data engine as the original production file. Still carries the Compliance Trend chart; no Corrections Required or Extension Applied tracking. Kept as a checkpoint. |
| `AlertDashboard_vXX_3_2.html` | Adds collapsible Detailed Data Grid rows and the neutral "Complete" sub-badge label on top of vXX_3. Still carries the Compliance Trend chart; no Corrections Required or Extension Applied tracking. Kept as a checkpoint — **not** the version that later gained Corrections/Extension support (that work happened in parallel on an unmerged branch and landed directly in vXX_3_3, not here). |
| `AlertDashboard_vXX_3_3.html` | **Current version.** Combines two lines of work: (1) Corrections Required and Extension Applied as fully tracked milestones with their own override/precedence rules, and (2) this round's layout change — the Compliance Trend chart was removed (its month-over-month rate didn't reconcile with the Overall Compliance KPI tile) and Workstation Volume & Status now spans the full left column of the Overview grid to fill the space that left, instead of stretching Overdue Aging horizontally. This is the file to open, test, and deploy. |
| `Anonymized_Data_Corrections.csv` | Sample dataset (136 records, long-format — one row per task, grouped by ISB #). Carries real Corrections Required and Extension Applied rows, used to verify that logic. Not itself part of the app. |
| `Compliance_Dashboard_Manual.html` | Standalone reference manual for vXX_3_3 — import format detection, milestone/status vocabulary, the overall-status precedence ladder, and every tab, filter, and export explained against the actual code. |
| `HANDOFF.md` | This file. |

Several earlier files (`AlertDashboard_vXX_2_3.html`, `AlertDashboard_vXX_2_4.html`, `Anonymized_Data.csv`, `CorrectionTest_vXX_2_3.xlsx`, `CorrectionTest_vXX_2_3.csv`) were deleted directly by the repo owner over the course of this work and are gone from `main`.

## vXX_3: the executive redesign

vXX_2_3 (now deleted) was functionally solid but visually generic — stock icon set, default system fonts, SaaS-template color palette, four-card KPI row over one under-filled chart. vXX_3 is a full presentation-layer rebuild on the *same* data engine (parsers, date handling, status math, filters, exporters carried over unchanged):

- **Typography** — Public Sans (US Web Design System) and Source Serif 4, subset and embedded as base64 WOFF2 (~60 KB total). Keeps the zero-dependency/offline property while rendering identically on every machine.
- **Palette** — institutional navy + muted brass, deep desaturated status colors (not bright SaaS red/green/amber), flat rectangular status chips instead of full pills.
- **Overview tab** — originally four panels: Compliance Trend (by month, against a 90% threshold), Overdue Aging (30/60/90/90+ day buckets), Milestone Performance (per-milestone on-time rate), and Workstation Volume & Status. The Compliance Trend panel was later removed in vXX_3_3 — see below.
- **Identity** — a drawn icon set and departmental seal on one 24×24 grid, replacing the stock icon library.
- **Print/PDF** — print stylesheet with a letterhead masthead, all tabs emitted, repeating table headers, fixed column widths.

Two latent bugs from the production build were also fixed here: `.bg-warning-light` was referenced by the compliance-icon logic but never defined, and that same logic wrote inline amber styles that outranked the CSS class and got stuck once the compliance rate moved back out of the 75–89% band.

## vXX_3_2: functional changes since vXX_3

1. **Collapsible Detailed Data Grid rows.** Records load collapsed — only ISB #, Workstation, Last Routing, Date Sent, and overall Status are visible. Selecting a row expands an inline panel below it showing every tracked milestone. Multiple rows can be expanded independently. Expansion state lives in `state.expandedRows` (a `Set` keyed by ISB #), reset on new file import, otherwise persistent across filtering/sorting/pagination — and it also drives the Filtered CSV export and print/PDF output: a collapsed record's milestone columns export/print blank, only expanded records carry the detail.
2. **Neutral "Complete" label.** A completed milestone's sub-badge reads "Complete" in dark gray/bold (`.text-complete`) instead of green (on-time) or red (late) — completion is a neutral fact, not a compliance judgment. The small status dot next to it still shows on-time (green) vs. late (red) at a glance.

## vXX_3_3: Corrections Required, Extension Applied, and the Overview layout

### Corrections Required (4th milestone)

The source data's `Correction(s) Required` task type is now parsed into `corrDue`/`corrComp`/`corrStatus` in both long- and wide-format import, using the same `getTaskStatus()` logic as every other milestone (due-date math, no special vocabulary).

**Override behavior (deliberately different from every other milestone):** an *incomplete* Corrections Required task overrides the record's overall status unconditionally — even if a different milestone (e.g. 6-Month Assessment) is separately, genuinely overdue. The record shows **"Corrections Pending"** in an amber pill. Rationale: a submission already kicked back for rework is a fundamentally different state than a milestone still working through its first review, and it's more operationally relevant than an unrelated deadline.

The label is cosmetic only — for compliance-rate/tile counting purposes, a Corrections Pending record still resolves to On-Schedule or Overdue **by the correction's own due date**, via `getComplianceBucket(item)`. The pill and CSV/print output always say "Corrections Pending" either way — only the internal counting differs.

### Extension Applied (5th milestone)

Same pipeline treatment as Corrections Required (`extDue`/`extComp`/`extStatus`), but **does not override** other milestones — it participates in the normal earliest-incomplete-by-due-date sequencing. When it does end up being the milestone driving the record's status, the pill shows **"Extension Applied"** in blue (`--info` design tokens), bucketed by its own due date via `getComplianceBucket()`.

One additional rule: if a record has an Extension Applied task with **no due date at all** and no other milestone is tracked either, the record shows "Extension Applied" instead of falling back to a bare "N/A". This is driven by a separate `extFound` boolean (set at parse time, independent of whether a due date exists) so the signal isn't lost when `getTaskStatus()` would otherwise return N/A for lack of a due date.

**This is also the rule most likely to surprise someone reading the grid:** a record can carry an "Extension Applied" task-type row in the source data and still *not* show the Extension Applied label — it only drives the overall status while it's the earliest-due incomplete milestone and there's no open Corrections Required on the same record. See the precedence ladder in `Compliance_Dashboard_Manual.html` (§04) before assuming a missing label means a parsing bug.

### Compliance Trend chart removed

The Overview tab's Compliance Trend panel (month-over-month on-schedule rate) was removed in this round — its figure didn't reconcile with the Overall Compliance KPI tile (54% on the tile vs. 100% on the trend chart, in the case that prompted this), and it was retired rather than patched in place. `renderTrendChart()`, its call sites, and its dedicated CSS were all deleted rather than left dead.

### Workstation Volume & Status now spans two rows

With the Compliance Trend panel gone, Workstation Volume & Status (previously bottom-left) now spans the full height of the left column — both chart rows — via `grid-row: span 2` on its `.chart-panel`, instead of stretching Overdue Aging to fill the freed horizontal space. Its `.bar-chart-container` also lost a hardcoded `max-height: 320px` cap so it actually fills the taller card rather than leaving whitespace beneath it. Overdue Aging and Milestone Performance keep their original sizes, top-right and bottom-right.

### Total Overdue rename

The "Critical Overdue" tile was renamed to "Total Overdue" ("Past scheduled due date"). The underlying count has no day-based severity threshold — one day late counts the same as 900 days late — so "Critical" implied a threshold that didn't exist and sat inconsistently next to the Overdue Aging chart, which *does* bucket by severity (30/60/90/90+ days).

### CSV column reorder

In the Filtered CSV export, `Overall Status` sits right after `Last Routing`: `ISB Number, Workstation, Last Routing, Overall Status, Date Created, Date Sent To Field, [milestone detail columns...]`. The on-screen grid's column layout is unchanged — this only affects the CSV export.

### Overdue Aging "Oldest" is now a link to that record

The Overdue Aging card's footer already showed the single oldest overdue record's days-past-due figure (via `getDrivingMilestone`/`getOverdueDays`, which mirror `getOverallStatus()`'s milestone selection — this was already correct and needed no change). It now also shows that record's ISB # as a button. Clicking it (`jumpToRecord(isb)`) switches to the Detailed Data Grid tab, pages to and expands the record, scrolls it into view, and briefly flashes the row (`.jump-flash` / `rowJumpFlash` keyframe). If the record has fallen outside the currently active filters (`state.data`), it shows a notification instead of jumping to nothing.

### Export Dashboard filename

`exportStandaloneDashboard()`'s downloaded file now reads `Alert_PEP_Compliance_<date>.html`, previously `Compliance_Dashboard_<date>.html`.

## Design decisions worth knowing before extending this further

- **`getOverallStatus(item)`** is the single source of truth for a record's displayed status. Order of evaluation: (1) if no milestone has a real value, return `'N/A'` unless Extension Applied was recorded with no due date, in which case return `'Extension Applied'`; (2) if Corrections Required is incomplete, unconditionally return `'Corrections Pending'`; (3) otherwise, the earliest-incomplete milestone by due date drives the result (tie-break order: Action Plan → 3-Month → 6-Month → Corrections → Extension); if every milestone is complete, the last-due one represents the record.
- **`getComplianceBucket(item)`** is the counting/charting counterpart — it resolves `'Corrections Pending'` back to `corrStatus` and `'Extension Applied'` back to `extStatus` for every place that needs a real On-Schedule/Overdue/N/A value: the summary tiles, Compliance %, Overdue Aging chart, Workstation Volume & Status chart, and both reviewer rollups (screen table and CSV export). **Any new chart or metric that reads `item.overallStatus` directly instead of going through this helper will silently miscount Corrections Pending / Extension Applied records** — this is the one thing most likely to bite a future change.
- **`getDrivingMilestone(item)`** (used only by the Overdue Aging chart's days-past-due figure) mirrors `getOverallStatus()`'s selection logic and must be kept in sync with it if that logic changes again.
- Pill colors: `.pill.success` (green, On-Schedule), `.pill.danger` (red, Overdue), `.pill.pending` (amber, Corrections Pending), `.pill.info` (blue, Extension Applied), `.pill.neutral` (gray, N/A).
- Sidebar filters exist for all five milestones (Action Plan, 3-Month, 6-Month, Corrections Required, Extension Applied), each independently filterable by On-Schedule/Overdue/N/A/All — these filter on the milestone's own raw status, not the overridden overall-status label.

## Known caveats / things not to be surprised by

- Wide-format CSV import (one row per ISB, rather than one row per task) has no reliable way to detect "an Extension Applied task exists but has no due date" — it can only infer presence from a filled-in date column. This is a pre-existing limitation of the wide-format path (shared with the "Extended" superseding-assessment logic).
- The Correction(s) Required and Extension Applied task types have no "Extended" variant of their own in the source data (unlike 3-Month/6-Month Assessment, which can be superseded by an "Extended" row) — the CSV export headers reflect this (no "Extended" column for either).
- `AlertDashboard_vXX_3.html` and `AlertDashboard_vXX_3_2.html` are checkpoints, not alternates to keep in sync — they predate the Corrections Required / Extension Applied work entirely and will not gain it retroactively. Treat vXX_3_3 as the only file to extend going forward.

## Where to pick this up

All of the above is committed and merged to `main`. If the next step is renaming `AlertDashboard_vXX_3_3.html` to a production filename, retiring the vXX_3 / vXX_3_2 checkpoints, or deploying this somewhere beyond the repo, that hasn't been done yet and would need explicit sign-off first.
