# Alert & PEP Compliance Dashboard — Handoff

Status as of this document: **`AlertDashboard_vXX_3_2.html` is the active working version.** Everything below describes what it is, how it got here, and what to know before touching it further.

## What this is

A single-file, offline-capable HTML dashboard for tracking Alert/PEP incident compliance milestones (Action Plan, 3-Month Assessment, 6-Month Assessment, Corrections Required, Extension Applied) against their due dates. No build step, no server, no external network calls — open the `.html` file in a browser and it runs. XLSX parsing (SheetJS) and both typefaces are embedded as base64 inside the file for this reason.

Users import a `.csv` or `.xlsx` export from the source system, and the dashboard pivots it, computes compliance status per record, and presents an executive-facing overview, a sortable/filterable data grid, and a reviewer performance summary. Filtered CSV export, a standalone read-only HTML export, and print/PDF are all built in.

## File inventory

| File | Role |
|---|---|
| `AlertDashboard_vXX_2_3.html` | Original production baseline. **Untouched.** Kept as the reference for "what did this replace." |
| `AlertDashboard_vXX_3.html` | Visual-only executive redesign of vXX_2_3. Same data engine and functionality, new design system (see below). Superseded by vXX_3_2 for actual use, but kept as a checkpoint. |
| `AlertDashboard_vXX_3_2.html` | **Current version.** Built on vXX_3, adds the functional changes described below. This is the file to open, test, and deploy. |
| `Anonymized_2026-09-04-19-41-17.csv` | Current sample dataset (136 records, long-format). Used for testing; not itself part of the app. |
| `HANDOFF.md` | This file. |

Two earlier files (`Anonymized_Data.csv`, `CorrectionTest_vXX_2_3.xlsx`) were deleted directly by the repo owner during this work and are gone from the branch.

## vXX_3: the executive redesign

vXX_2_3 was functionally solid but visually generic — stock icon set, default system fonts, SaaS-template color palette, four-card KPI row over one under-filled chart. vXX_3 is a full presentation-layer rebuild on the *same* data engine (parsers, date handling, status math, filters, exporters carried over unchanged):

- **Typography** — Public Sans (US Web Design System) and Source Serif 4, subset and embedded as base64 WOFF2 (~60 KB total). Keeps the zero-dependency/offline property while rendering identically on every machine.
- **Palette** — institutional navy + muted brass, deep desaturated status colors (not bright SaaS red/green/amber), flat rectangular status chips instead of full pills.
- **Overview tab** — replaced the single half-empty bar chart with four panels: Compliance Trend (by month, against a 90% threshold), Overdue Aging (30/60/90/90+ day buckets), Milestone Performance (per-milestone on-time rate), and the original Workstation Volume chart (now carrying a per-unit rate).
- **Identity** — a drawn icon set and departmental seal on one 24×24 grid, replacing the stock icon library.
- **Print/PDF** — new print stylesheet: letterhead masthead, all tabs emitted, repeating table headers, fixed column widths (an early version clipped the Status column off the page — fixed and verified).

Two latent bugs from the production build were also fixed here: `.bg-warning-light` was referenced by the compliance-icon logic but never defined, and that same logic wrote inline amber styles that outranked the CSS class and got stuck once the compliance rate moved back out of the 75–89% band.

## vXX_3_2: functional changes since vXX_3

These were built iteratively, each round tested against the real dataset before commit. In order:

### 1. Collapsible Detailed Data Grid rows
Records load collapsed — only ISB #, Workstation, Last Routing, Date Sent, and overall Status are visible. Selecting a row expands an inline panel below it showing all five milestones (Action Plan, 3-Month, 6-Month, Corrections Required, Extension Applied). Multiple rows can be expanded independently. Expansion state lives in `state.expandedRows` (a `Set` keyed by ISB #), reset on new file import, otherwise persistent across filtering/sorting/pagination.

**This expansion state also drives the Filtered CSV export and the print/PDF output**: a collapsed record's milestone columns export/print blank; only expanded records carry the detail. This was a deliberate design principle established early in the work — what's on screen is what leaves the building, in every export format.

### 2. Neutral "Complete" label
A completed milestone's sub-badge now reads "Complete" in dark gray/bold (`.text-complete`, `var(--ink-soft)`) instead of green (on-time) or red (late) — completion is a neutral fact, not a compliance judgment. The small status dot next to it still shows on-time (green) vs. late (red) at a glance; only the word itself went neutral.

### 3. Corrections Required (4th milestone)
The source data's `Correction(s) Required` task type was previously unrecognized by the CSV parser and silently dropped. Now parsed into `corrDue`/`corrComp`/`corrStatus` in both long- and wide-format import, using the same `getTaskStatus()` logic as every other milestone (due-date math, no special vocabulary).

**Override behavior (deliberately different from every other milestone):** an *incomplete* Corrections Required task overrides the record's overall status unconditionally — even if a different milestone (e.g. 6-Month Assessment) is separately, genuinely overdue. The record shows **"Corrections Pending"** in an amber pill. Rationale: a submission already kicked back for rework is a fundamentally different state than a milestone still working through its first review, and it's more operationally relevant than an unrelated deadline.

The label is cosmetic only — for compliance-rate/tile counting purposes, a Corrections Pending record still resolves to On-Schedule or Overdue **by the correction's own due date**, via a new `getComplianceBucket(item)` helper. A correction due in the future counts On-Schedule; one whose due date has passed counts Overdue. The pill and CSV/print output always say "Corrections Pending" either way — only the internal counting differs.

### 4. Extension Applied (5th milestone)
Same pipeline treatment as Corrections Required (`extDue`/`extComp`/`extStatus`), but **does not override** other milestones — it participates in the normal earliest-incomplete-by-due-date sequencing. When it does end up being the milestone driving the record's status, the pill shows **"Extension Applied"** in blue (new `--info` design tokens), bucketed by its own due date via the same `getComplianceBucket()` mechanism.

One additional rule, added after the fact: if a record has an Extension Applied task with **no due date at all** (this happens in the real data — an extension was logged but no new due date was recorded) and no other milestone is tracked either, the record now shows "Extension Applied" instead of falling back to a bare "N/A". This is driven by a separate `extFound` boolean (set at parse time, independent of whether a due date exists) so the signal isn't lost when `getTaskStatus()` would otherwise return N/A for lack of a due date.

### 5. Total Overdue rename
The "Critical Overdue" tile was renamed to "Total Overdue" ("Past scheduled due date"). The underlying count has no day-based severity threshold — one day late counts the same as 900 days late — so "Critical" implied a threshold that didn't exist and sat inconsistently next to the Overdue Aging chart, which *does* bucket by severity (30/60/90/90+ days).

### 6. CSV column reorder
In the Filtered CSV export, `Overall Status` was moved from the last column to right after `Last Routing`: `ISB Number, Workstation, Last Routing, Overall Status, Date Created, Date Sent To Field, [milestone detail columns...]`. The on-screen grid's column layout is unchanged — this only affects the CSV export.

## Design decisions worth knowing before extending this further

- **`getOverallStatus(item)`** is the single source of truth for a record's displayed status. Order of evaluation: (1) if no milestone has a real value, return `'N/A'` unless Extension Applied was recorded with no due date, in which case return `'Extension Applied'`; (2) if Corrections Required is incomplete, unconditionally return `'Corrections Pending'`; (3) otherwise, the earliest-incomplete milestone by due date drives the result (tie-break order: Action Plan → 3-Month → 6-Month → Corrections → Extension); if every milestone is complete, the last-due one represents the record.
- **`getComplianceBucket(item)`** is the counting/charting counterpart — it resolves `'Corrections Pending'` back to `corrStatus` and `'Extension Applied'` back to `extStatus` for every place that needs a real On-Schedule/Overdue/N/A value: the summary tiles, Compliance %, Compliance Trend chart, Overdue Aging chart, Workstation Volume chart, and both reviewer rollups (screen table and CSV export). **Any new chart or metric that reads `item.overallStatus` directly instead of going through this helper will silently miscount Corrections Pending / Extension Applied records** — this is the one thing most likely to bite a future change.
- **`getDrivingMilestone(item)`** (used only by the Overdue Aging chart's days-past-due figure) mirrors `getOverallStatus()`'s selection logic and must be kept in sync with it if that logic changes again.
- Pill colors: `.pill.success` (green, On-Schedule), `.pill.danger` (red, Overdue), `.pill.pending` (amber, Corrections Pending), `.pill.info` (blue, Extension Applied), `.pill.neutral` (gray, N/A).
- Sidebar filters exist for all five milestones (Action Plan, 3-Month, 6-Month, Corrections Required, Extension Applied), each independently filterable by On-Schedule/Overdue/N/A/All — these filter on the milestone's own raw status, not the overridden overall-status label.

## Known caveats / things not to be surprised by

- The real dataset currently has only 2 records with a Corrections Required task and 1 with an Extension Applied task (which itself has no due date). The override logic has been verified against synthetic records run through the actual import/render pipeline (not just the underlying functions in isolation), but hasn't been exercised at volume against real data.
- Wide-format CSV import (one row per ISB, rather than one row per task) has no reliable way to detect "an Extension Applied task exists but has no due date" — it can only infer presence from a filled-in date column. This is a pre-existing limitation of the wide-format path (shared with the "Extended" superseding-assessment logic), not new.
- The Correction(s) Required and Extension Applied task types have no "Extended" variant of their own in the source data (unlike 3-Month/6-Month Assessment, which can be superseded by an "Extended" row) — the CSV export headers reflect this (no "Extended" column for either).

## Where to pick this up

Branch `claude/push-file-repo-lls55d` has all of the above committed and pushed; no pull request has been opened. If the next step is deploying vXX_3_2 to replace the production file, or opening a PR against a default branch, that hasn't been done yet and would need explicit sign-off first.
