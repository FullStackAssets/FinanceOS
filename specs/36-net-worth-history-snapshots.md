# Section 36 — Net worth history & snapshots (items 701–720)

Builder spec for the FinanceOS Retool app (App 11). Net worth history gives Nic a point-in-time record of net worth computed from his own user-entered account balances — nothing here connects to a bank or imports live data.

## Tables touched

**accounts** (existing, App 11): id (pk), name (text), type (text: Checking / Savings / Credit / Investment), institution (text), balance (number), updated_at (timestamp), is_sample (bool).

**net_worth_snapshots** (NEW): id (pk), snapshot_date (date), total_net_worth (number), total_assets (number), total_liabilities (number), breakdown_json (text — per-account {account_id, name, type, balance} array), note (text, nullable), event_tags (text, comma-separated, nullable), source (text: manual / month_end_prompt / backfill), is_sample (bool), created_at (timestamp), deleted_at (timestamp, nullable — soft delete keeps audit trail).

## Cross-cutting rules

- Every screen in this section shows the persistent banner "Manual tracking — no bank connection yet." alongside the existing app banner.
- All snapshot values derive from user-entered account balances; nothing ever presents as a live bank feed.
- When the global sample-data toggle is OFF, any snapshot flagged `is_sample=true` (built from sample accounts) is hidden from tables, charts, streaks, milestones, and exports; charts show a gap rather than a fabricated point.
- Snapshot rows flagged `is_sample=true` carry a SAMPLE badge in tables and timeline.
- Snapshot creation is a user action (or answers a user action); the app never silently fabricates a snapshot.

---

### 701. Auto-prompt a net-worth snapshot on the last day of each month.

**Components:** Modal component `mdlSnapshotPrompt` (title "End-of-month snapshot", buttons "Take snapshot now" / "Skip this month"), triggered from the Overview screen. Add a date comparison in the app's on-load JS.

**Data:** On app load, JS query `checkSnapshotPrompt`: compare `moment().date()` with last day of current month and `getSnapshots` result for a snapshot this month. Show modal only if it's the last day and no snapshot exists yet this month. "Take snapshot now" runs `insertSnapshot` writing date, computed totals, per-account breakdown, source=`month_end_prompt`.

**Logic notes:** Prompt once per day max — record a `snapshot_prompt_dismissed` temp state with today's date to avoid re-prompting on reload. "Skip this month" suppresses for the rest of the month.

**Acceptance:** (1) Opening the app on the last day of the month with no snapshot yet that month opens the prompt exactly once. (2) "Take snapshot now" writes a snapshot with source=`month_end_prompt` and correct totals. (3) "Skip this month" suppresses the prompt until the next month.

### 702. Store snapshots immutably with date, value, and per-account breakdown.

**Components:** No dedicated UI; the Retool Database `net_worth_snapshots` table is the store. Spec the table for Nic: editable only through the app's delete flow (item 710), not via inline editing.

**Data:** `insertSnapshot` query builds the row: `snapshot_date` (date-only, no time component), `total_assets`, `total_liabilities` (absolute of negative credit balances), `total_net_worth = assets − liabilities`, `breakdown_json` = JSON of every included account with id, name, type, balance, `source`, `is_sample` flag inherited from any included sample account.

**Logic notes:** Immutability = app never issues UPDATE on snapshot rows; the delete flow (710) soft-deletes via `deleted_at`. The breakdown is frozen at creation — later balance edits never rewrite it.

**Acceptance:** (1) Created snapshot rows are never updated by any app query (grep app queries for UPDATE on this table: none except soft delete). (2) `breakdown_json` parses to an array whose balances sum to `total_net_worth`. (3) Editing an account balance afterward does not change the stored snapshot row.

### 703. Display snapshot history as a table and a line chart.

**Components:** Table component `tblSnapshots` (columns: Date, Net worth, Δ vs previous, Note, badges) and Line Chart `chrtNetWorth` (x = snapshot_date, y = total_net_worth) on a new "Net worth history" screen.

**Data:** Query `getSnapshots` orders by `snapshot_date` asc, filters `deleted_at IS NULL`, applies sample-toggle filter. Transformer `snapDelta` computes per-row Δ vs previous snapshot and percent change.

**Logic notes:** Table click selects a snapshot and syncs a chart marker; chart tooltip shows note and source. Empty state: "No snapshots yet — take your first snapshot to start tracking."

**Acceptance:** (1) Table lists snapshots newest-first with correct Δ vs previous row. (2) Line chart plots every visible snapshot in chronological order with hover tooltips. (3) With zero snapshots, the empty-state message shows instead of an empty chart.

### 704. Allow manual snapshots at any time with an optional note.

**Components:** Button `btnSnapshotNow` ("Take snapshot") in the Net worth history screen header; modal `mdlManualSnapshot` with TextArea `txtSnapshotNote` (optional, 280 chars) and date display of today.

**Data:** `insertSnapshot` with `snapshot_date = today`, `source='manual'`, `note` from the text area (nullable). After insert, `getSnapshots` refetches and the new row highlights briefly.

**Logic notes:** Manual snapshots on the same date as an existing snapshot are allowed but flagged with an "(already have one today)" hint so Nic doesn't accidentally double-snapshot; totals use current balances.

**Acceptance:** (1) Button + note flow writes a snapshot dated today with the entered note stored verbatim. (2) Empty note stores NULL, not empty string. (3) Creating a snapshot does not disturb existing rows.

### 705. Compare any two snapshots with a per-account change breakdown.

**Components:** Two Select components `selSnapA`, `selSnapB` (populated from `getSnapshots`), Button `btnCompare`, Table `tblCompare` with columns Account, Type, Balance @A, Balance @B, Δ, %.

**Data:** Transformer `cmpSnapshots` joins the two `breakdown_json` arrays on account name (accounts may have been renamed — match by name with account_id fallback), computes Δ and percent per account plus totals row.

**Logic notes:** If A and B are the same snapshot, show "Select two different snapshots." Accounts present in one snapshot only are shown with the missing side marked "—" and a footnote explaining renames/deletions.

**Acceptance:** (1) Choosing two different snapshots renders per-account Δs that sum to the total net-worth difference. (2) Same-snapshot selection shows the guidance message, not garbage. (3) Renamed/missing accounts display gracefully with the "—" marker.

### 706. Annotate snapshots with life events ("sold car", "bonus").

**Components:** In the snapshot detail drawer: Multi-select Tags component `tagsEvents` with free-text entry (presets: sold car, bonus, raise, moved, debt paid, windfall) and TextArea `txtEventNote`.

**Data:** `updateSnapshotEvents` query — the single allowed UPDATE on this table, limited to `event_tags` and `note` columns only (never values). Events render as chips on the table row, chart tooltip, and timeline.

**Logic notes:** Event tags are editable because annotation is inherently post-hoc; financial values remain immutable. Keep a short preset list plus custom entry; cap at 10 tags per snapshot.

**Acceptance:** (1) Adding an event tag stores it in `event_tags` and renders it as a chip wherever the snapshot appears. (2) Editing tags/notes never alters `total_net_worth` or `breakdown_json`. (3) Custom (non-preset) tags save and display correctly.

### 707. Show annualized growth rate between the first and latest snapshot.

**Components:** Statistic components `statAnnualizedGrowth` and `statTotalGrowth` on the Net worth history header; tooltip explaining the formula.

**Data:** Transformer `growthRate` takes first and latest visible snapshots: `years = daysBetween/365.25`, `CAGR = (latest/first)^(1/years) − 1`. Handles first ≤ 0 by showing "—" with tooltip "Growth rate needs a positive starting value."

**Logic notes:** Show both absolute change and CAGR. Negative starting net worth makes CAGR misleading — display total change and months elapsed instead, with the formula footnote.

**Acceptance:** (1) With ≥2 snapshots and positive start, CAGR matches the standard formula to 2 decimals. (2) With negative/zero start value, rate shows "—" and the tooltip explains why. (3) With <2 snapshots, the stat shows "Not enough history."

### 708. Export snapshot history to CSV.

**Components:** Button `btnExportSnapshots` on the Net worth history screen; exports respect the sample toggle (excluded rows never export).

**Data:** JS query `exportSnapshotsCSV` builds CSV rows: snapshot_date, total_net_worth, total_assets, total_liabilities, note, event_tags, source, sample flag. Uses Retool's `utils.exportData` with filename `financeos-net-worth-history-YYYY-MM-DD.csv`.

**Logic notes:** Export includes a header comment line stating "Manual tracking — figures entered by hand, no bank connection." Deleted snapshots never export. Date format ISO (YYYY-MM-DD) for spreadsheet compatibility.

**Acceptance:** (1) Exported CSV contains exactly the visible (non-deleted, toggle-respecting) snapshots. (2) Amounts export as plain numbers (no currency symbols). (3) First line of the file is the manual-tracking disclaimer.

### 709. Display snapshots on a timeline with event annotations.

**Components:** Timeline component `tlNetWorth` (custom list rendering vertical timeline) — each node: date, net-worth value, Δ chip, event chips, note excerpt. Clicking a node opens the snapshot detail drawer.

**Data:** Reuses `getSnapshots`; transformer `timelineNodes` shapes rows into timeline items sorted newest-first, flagging months with no snapshot as gaps.

**Logic notes:** Timeline groups by year with sticky year headers. Sample snapshots show the SAMPLE badge inline. Nodes with event tags render the tag chips beneath the value.

**Acceptance:** (1) Timeline renders every visible snapshot in reverse-chronological order with correct Δ chips. (2) Event tags appear as chips under their snapshot node. (3) Clicking a node opens the detail drawer for that snapshot.

### 710. Allow deleting a snapshot with confirmation and audit logging.

**Components:** Delete icon button per table row opening confirm Modal `mdlDeleteSnapshot` ("Delete snapshot of <date>? This keeps an audit record.") with Confirm/Cancel.

**Data:** `softDeleteSnapshot` sets `deleted_at = now()` — never a hard delete. `logAudit` writes to the audit log table (item 738/Section 38 area): action=`snapshot_deleted`, snapshot id + date. Deleted rows are excluded from all queries via `deleted_at IS NULL`.

**Logic notes:** Soft delete preserves the immutable history claim while letting Nic fix mistakes. Provide an "Include deleted" admin view showing deleted rows with restore button (restore clears `deleted_at`, logs the restore).

**Acceptance:** (1) Confirming deletion hides the snapshot everywhere but the row remains in the DB with `deleted_at` set. (2) An audit-log entry records the deletion with snapshot id and date. (3) Restore re-shows the snapshot and logs the restore.

### 711. Show asset vs liability split per snapshot over time.

**Components:** Stacked bar or area Chart `chrtAssetsLiabilities` on the Net worth history screen: x = snapshot_date, two series (assets positive green, liabilities red).

**Data:** `getSnapshots` supplies `total_assets` and `total_liabilities`; transformer shapes them into the chart series. Net-worth line can overlay as a toggle.

**Logic notes:** Liabilities stored positive for charting (`total_liabilities` column), net worth = assets − liabilities. Tooltip shows both components plus net on hover.

**Acceptance:** (1) Chart renders two stacked/area series matching each snapshot's stored asset and liability totals. (2) Toggling the net-worth overlay adds a line matching `total_net_worth`. (3) Tooltip numbers equal the stored column values.

### 712. Overlay goal target lines on the net-worth chart.

**Components:** Toggle `tglGoalLines` on `chrtNetWorth`; when on, the chart renders horizontal reference lines for each goal (goals table: goal_name, target).

**Data:** Query `getGoals` (existing table) supplies targets; transformer `goalLines` maps them to Plotly/retool chart reference-line config with goal name labels.

**Logic notes:** Only goals with a deadline in the future (or no deadline) draw lines; completed goals (saved ≥ target) draw dashed. Lines are labeled with goal name + target amount so the crossover point is visible.

**Acceptance:** (1) Enabling the toggle draws one labeled horizontal line per active goal at its target value. (2) Goals with saved ≥ target render dashed. (3) Disabling the toggle removes all lines cleanly.

### 713. Add a "snapshot completeness" check warning about stale account balances.

**Components:** Alert callout `calloutCompleteness` above the snapshot button; shows "2 of 5 accounts updated in the last 7 days" style summary.

**Data:** Transformer `completenessCheck` compares each non-sample account's `updated_at` against a 7-day threshold; counts fresh vs stale. Sample accounts listed separately.

**Logic notes:** Threshold is 7 days by default. Stale list names the accounts so Nic can update balances before snapshotting; the check never blocks — it warns. Sample accounts are excluded from the "stale" count but noted.

**Acceptance:** (1) With all accounts updated recently, the callout shows a green "All balances fresh" state. (2) With stale accounts, it lists them by name. (3) The warning never blocks snapshot creation.

### 714. Support quarterly snapshot reminders as an alternative to monthly.

**Components:** Select `selSnapshotCadence` in Settings (options: Monthly / Quarterly / Off), stored in a `settings` temp state persisted to a `app_settings` table row.

**Data:** `checkSnapshotPrompt` reads the cadence: monthly → last day of month; quarterly → last day of Mar/Jun/Sep/Dec; off → never prompt. Manual snapshots remain available regardless.

**Logic notes:** Changing cadence mid-quarter takes effect immediately; the "already snapshotted this period" check uses the active cadence's period boundaries.

**Acceptance:** (1) Quarterly mode prompts only on the last days of Mar, Jun, Sep, Dec. (2) Off mode never prompts. (3) Switching cadence does not delete or alter existing snapshots.

### 715. Display the longest snapshot streak (consecutive months).

**Components:** Statistic `statStreak` ("Longest streak: 8 months") with a flame/badge icon; tooltip lists the streak's start–end dates.

**Data:** Transformer `streakCalc` sorts visible snapshots by month and walks for the longest run of consecutive calendar months (one snapshot per month counts; multiple in one month don't inflate).

**Logic notes:** Backfilled snapshots (720) count toward streaks — they represent real history. Sample snapshots count only when the sample toggle is on. Gaps break the streak.

**Acceptance:** (1) Streak equals the longest run of consecutive months with ≥1 snapshot each. (2) Multiple snapshots in one month don't extend the streak. (3) Tooltip shows the streak's start and end months.

### 716. Show per-account contribution to net-worth change between snapshots.

**Components:** Bar Chart `chrtContribution` (horizontal, per-account Δ bars) inside the compare view (item 705), plus a "Top movers" list highlighting the 3 largest positive and negative contributors.

**Data:** Reuses the `cmpSnapshots` transformer output; transformer `contribChart` maps per-account Δ to bar series, sorted descending.

**Logic notes:** Bars are colored green/red by sign. Accounts with zero change are hidden from the chart but shown in the table. Contribution percentages shown relative to total net-worth change.

**Acceptance:** (1) Bars sum visually and numerically to the total net-worth change between the two snapshots. (2) Top movers list matches the chart's extremes. (3) Zero-change accounts excluded from the chart but present in the table.

### 717. Add snapshot sharing as an image with the manual-data banner.

**Components:** Button `btnShareSnapshot` on the snapshot detail drawer; generates a share card image via an HTML component rendered to canvas (or Retool's export-image pattern).

**Data:** JS builds a card: net-worth value, date, Δ vs previous, event tags, and the mandatory footer text "Manual tracking — figures entered by hand, no bank connection." Download as PNG.

**Logic notes:** The banner text is baked into the image — it cannot be cropped out without visibly damaging the card (footer sits inside the card border). Sample snapshots export with a SAMPLE watermark.

**Acceptance:** (1) Generated image contains the value, date, and the manual-tracking footer text legibly. (2) Footer cannot be removed via any in-app option. (3) Sample snapshots carry a visible SAMPLE watermark on the image.

### 718. Keep sample accounts out of snapshots when the sample toggle is off.

**Components:** Global toggle `tglSampleData` (existing app-level); snapshot creation reads its state and stores `is_sample` on the snapshot row.

**Data:** On insert: if toggle is off, exclude `is_sample=true` accounts from totals and `breakdown_json`, and mark the snapshot `is_sample=false`. If on, include them and mark `is_sample=true` when any included account is sample. All snapshot queries filter `is_sample=false` when the toggle is off.

**Logic notes:** This keeps real and sample histories cleanly separated — a snapshot taken with samples on never pollutes the real history. The breakdown records which accounts were included so the exclusion is auditable.

**Acceptance:** (1) Snapshot taken with toggle OFF contains zero sample accounts in its breakdown and totals. (2) Toggle OFF hides every `is_sample=true` snapshot from tables, charts, streaks, and exports. (3) Snapshot taken with toggle ON is flagged SAMPLE and excluded when the toggle turns off.

### 719. Display a milestone tracker ($0, $10k, $100k net worth) from snapshot history.

**Components:** Milestone component `msNetWorth` — vertical list of milestones ($0, $10k, $50k, $100k, $250k, $500k, $1M) each showing achieved date (first snapshot ≥ milestone) or "Not yet" with current progress %.

**Data:** Transformer `milestoneCalc` scans visible snapshots chronologically; for each milestone, finds the first snapshot date where `total_net_worth` ≥ milestone. Progress = current net worth / milestone.

**Logic notes:** Milestones are fixed tiers; Nic can add custom tiers via a settings list. Achieved dates come from snapshot dates — backfilled snapshots (720) make early milestones datable.

**Acceptance:** (1) Each achieved milestone shows the correct first-crossing date from snapshot history. (2) Unreached milestones show progress % toward them. (3) Adding a custom tier works and persists.

### 720. Backfill older snapshots manually with a date picker for historical context.

**Components:** Modal `mdlBackfill` with DatePicker `dtpBackfillDate` (any past date), per-account balance inputs pre-filled with current balances (editable), TextArea for note.

**Data:** `insertSnapshot` with the chosen past date, `source='backfill'`, and the entered balances as the breakdown. Validation: date must be in the past and not duplicate an existing snapshot date (warn, allow override).

**Logic notes:** Backfill uses balances as-of that date from Nic's memory/records — the note field should encourage citing the source ("from old bank statement"). Backfilled rows sort into history normally and count for streaks/milestones.

**Acceptance:** (1) Backfill writes a snapshot on the chosen past date with `source='backfill'` and the entered balances. (2) Duplicate-date backfill warns but allows proceeding. (3) Backfilled snapshots appear in tables, charts, streaks, and milestones like any other.
