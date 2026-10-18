# FinanceOS — 02. Net Worth Tracking (items 21–40)

Builder spec for the FinanceOS Retool app (manual-entry personal finance; NO bank connection).

## Cross-cutting rules (apply to every item)
- Every screen that renders a financial figure shows the persistent banner **"Manual tracking — no bank connection yet."** (exact text, never paraphrased).
- Every seeded row carries `is_sample=true` and a visible **SAMPLE** badge. When the global sample toggle hides samples, sample rows are excluded from all sums, charts, and exports.
- Nothing may be presented, labeled, or implied as a live bank feed. Synced/bank data does not exist.
- Net worth = sum of included account balances. Credit-type accounts contribute as negatives (balance × −1). Archived and sample-excluded accounts do not contribute.

## Tables touched

Existing:
- `accounts`: id (pk), name (text), type (Checking/Savings/Credit/Investment), institution (text), balance (number), updated_at (timestamp), is_sample (bool). Extended below with `include_in_net_worth` (bool, default true).
- `goals`: id (pk), goal_name (text), target (number), saved (number), deadline (date).

New:
- `net_worth_snapshots`: id (pk), snapshot_date (date), snapshot_at (timestamp), total_value (number), breakdown (text, JSON: per-account {account_id, name, type, balance, contribution}), note (text), is_sample (bool), created_by (text).
- `reconciliations`: id (pk), snapshot_id (fk, nullable), account_id (fk), balance_confirmed (number), confirmed_at (timestamp), confirmed_by (text), is_sample (bool).

---

### 21. Compute net worth as the live sum of all account balances, excluding accounts flagged is_sample when the sample toggle is off.

- **Components:** Text component `netWorthValue` (hero number, currency format), plus the shared `bannerManualTracking` text pinned above it.
- **Data:** Transformer `calcNetWorth`: `{{ accountsQuery.data.filter(a => showSamples.value || !a.is_sample).filter(a => a.include_in_net_worth !== false).reduce((s,a) => s + (a.type==='Credit' ? -Math.abs(a.balance) : a.balance), 0) }}`.
- **Logic notes:** Global `showSamples` toggle (temporary state, default false) gates the `is_sample` filter. Exclude archived accounts. Recompute on any `accountsQuery` refresh.
- **Acceptance:** (1) Toggling sample visibility changes the total only by sample rows. (2) A Credit balance of $2,000 reduces the total by exactly $2,000. (3) Manual test: total matches hand-summed balances.

### 22. Record manual net-worth snapshots on demand with a timestamped "Snapshot" button.

- **Components:** Button `btnSnapshot` ("Take snapshot"), TextInput `snapshotNote` (optional), Modal `modalSnapshotConfirm`.
- **Data:** Query `createSnapshot` (Retool DB INSERT into `net_worth_snapshots`): snapshot_date = today, snapshot_at = now, total_value = `calcNetWorth.value`, breakdown = JSON of per-account contributions, note, is_sample = any included account is sample. On success, refresh `snapshotsQuery`.
- **Logic notes:** Block the button when no accounts are included (total undefined); disable while the mutation runs. Timestamp stored in America/New_York.
- **Acceptance:** (1) Clicking saves one row with correct total and timestamp. (2) Refresh persists the row. (3) Empty note allowed; button disabled with zero accounts.

### 23. Chart net-worth history from saved snapshots on a line graph labeled with the manual-entry banner.

- **Components:** Chart `chartNetWorthHistory` (line, x = snapshot_date, y = total_value), Text `chartCaption` reading "Manual data only — snapshots you recorded."
- **Data:** `snapshotsQuery` (SELECT * FROM net_worth_snapshots ORDER BY snapshot_date; filtered by sample toggle). Transformer sorts ascending and maps to chart series.
- **Logic notes:** Banner caption sits directly under the chart title, not buried in a tooltip. One data point still renders (single dot + label).
- **Acceptance:** (1) Each snapshot appears as a point on the line. (2) Caption text is exact. (3) Sample snapshots vanish from the chart when samples are hidden.

### 24. Break net worth down by account type (Checking/Savings/Credit/Investment) in a stacked bar view.

- **Components:** Chart `chartNetWorthByType` (100%-style stacked bar, one bar per snapshot date, segments = the four types).
- **Data:** Transformer `netWorthByType` groups each snapshot's JSON breakdown by account type, summing contributions (Credit negative). Colors fixed per type across the app.
- **Logic notes:** Negative Credit segment renders below zero where supported; fallback renders Credit as its own bar with minus labeling. Unknown types fall into "Other".
- **Acceptance:** (1) Four segments per snapshot sum to the snapshot total. (2) Colors match the legend and other charts. (3) Credit shows as a negative segment.

### 25. Show credit balances as negative contributions to net worth with a clear minus sign.

- **Components:** Table `tblNetWorthBreakdown` with a `contribution` column; custom column mapper renders Credit rows as "−$X".
- **Data:** Contribution = `a.type==='Credit' ? -Math.abs(a.balance) : a.balance`.
- **Logic notes:** Never rely on color alone; the minus sign is explicit text. A Credit account with a positive stored balance still contributes negatively (abs applied).
- **Acceptance:** (1) Every Credit row shows a leading minus. (2) Hero total equals sum of signed contributions. (3) Screen-reader label reads "minus".

### 26. Allow excluding individual accounts from net worth via a per-account include/exclude toggle.

- **Components:** Toggle `toggleIncludeNW` inside each account row (list view or drawer); tooltip "Include in net worth".
- **Data:** New column `include_in_net_worth` (bool, default true) on `accounts`; query `updateAccountInclude` persists the toggle; `calcNetWorth` filters on it.
- **Logic notes:** Default true for all existing rows (backfill on migration). Excluded accounts show a muted "excluded" tag in the breakdown.
- **Acceptance:** (1) Toggling off removes the account's contribution instantly. (2) Setting persists across reload. (3) Excluded accounts remain visible with an "excluded" tag.

### 27. Display net-worth change since the previous snapshot with an up/down indicator.

- **Components:** Statistic `statNWChange` next to the hero number: delta amount + ▲/▼ icon + percent.
- **Data:** Transformer compares latest two rows of `snapshotsQuery` (respecting sample filter): delta = latest − previous.
- **Logic notes:** With fewer than two snapshots, show "—" and the hint "Take a second snapshot to see change." Zero delta shows "no change".
- **Acceptance:** (1) Delta equals latest minus previous exactly. (2) Up/down icon matches sign. (3) Single snapshot shows the empty hint, not $0.

### 28. Annotate net-worth snapshots with a note field explaining what changed (e.g., "paid off card").

- **Components:** TextInput `snapshotNote` (max 200 chars) in the snapshot modal; note shown under each row in `tblSnapshots`.
- **Data:** `note` column on `net_worth_snapshots`; included in `createSnapshot` and editable via `updateSnapshotNote`.
- **Logic notes:** Note is optional; empty notes render as "—". Notes export with the snapshot (see item 29).
- **Acceptance:** (1) Note saves and displays on the snapshot row. (2) Editing the note does not alter the stored total. (3) Note appears in CSV export.

### 29. Export net-worth history to CSV with an is_sample column preserved.

- **Components:** Button `btnExportSnapshots` ("Export CSV").
- **Data:** Query `exportSnapshots` selects snapshot_date, total_value, note, is_sample; `utils.exportData` writes CSV with header row. Prepend a banner row comment "Manual tracking — no bank connection yet." as the first CSV line.
- **Logic notes:** Column order fixed: snapshot_date, total_value, note, is_sample. UTF-8 encoding. Sample toggle applies to which rows are included; the column itself is always present.
- **Acceptance:** (1) CSV opens in a spreadsheet app with correct columns. (2) is_sample values are true/false per row. (3) Banner line present at top of file.

### 30. Add a net-worth goal line overlay drawn from a linked savings goal's target.

- **Components:** Select `selectNWGoal` (goal picker) on the net-worth chart header; Chart `chartNetWorthHistory` gains a dashed goal-line series.
- **Data:** Transformer appends a constant series at `selectedGoal.target` across the chart's date range. Goal stored in a `nw_goal_id` app variable (temporary state).
- **Logic notes:** Line labeled with the goal name + target (e.g., "Emergency fund — $10,000"). Clearing the picker removes the line. Works on both line and area variants.
- **Acceptance:** (1) Goal line renders at exactly the goal's target value. (2) Label names the linked goal. (3) No goal selected → no overlay.

### 31. Show a table of all past net-worth snapshots with date, value, and note.

- **Components:** Table `tblSnapshots` (columns: snapshot_date, total_value formatted as currency, note, SAMPLE pill when is_sample, actions: edit note, delete).
- **Data:** `snapshotsQuery` ordered by snapshot_date desc, respecting the sample toggle.
- **Logic notes:** Delete requires confirmation and is audit-logged (see audit section). Values are immutable except the note.
- **Acceptance:** (1) Every snapshot row appears with correct date/value/note. (2) Sample rows carry the SAMPLE pill. (3) Totals in the table match the chart.

### 32. Recalculate net worth instantly when any account balance is edited, without page reload.

- **Components:** None new; wiring on the Accounts screen's balance editor.
- **Data:** On success of `updateAccountBalance`, trigger `accountsQuery` refresh; `calcNetWorth` is reactive to `accountsQuery.data`, so the hero number, breakdown table, and type chart update in the same tick.
- **Logic notes:** No `location.reload()` or full page re-query of unrelated tables. Optimistic update acceptable but must reconcile with the DB response.
- **Acceptance:** (1) Editing a balance updates the hero number within 1 second. (2) No page reload occurs. (3) Breakdown and stacked bar update in the same refresh.

### 33. Flag accounts not updated in 30 days with a "stale balance" badge on the net-worth breakdown.

- **Components:** Badge `badgeStale` in `tblNetWorthBreakdown` rows where `updated_at` < today − 30 days.
- **Data:** Transformer `staleFlags` computes days since `updated_at` per account; badge visible when ≥ 30.
- **Logic notes:** Badge text: "stale — not updated in N days". Editing the balance clears it (updated_at refreshes). Sample accounts flagged the same way.
- **Acceptance:** (1) Account untouched 31 days shows the badge with day count. (2) Updating the balance clears the badge. (3) Badge has text, not color alone.

### 34. Let the user set a net-worth target date and show projected trajectory from manual entries.

- **Components:** Date picker `dateNWTarget`, Number input `inputNWTarget`, Chart overlay series `projectedTrajectory` (dotted line).
- **Data:** Transformer computes current net worth, monthly change rate from the last 3 snapshots (fallback: manual entry of expected monthly change), then draws a straight-line projection to the target date. Labels the line "Estimate from your manual entries — not a guarantee."
- **Logic notes:** If fewer than 2 snapshots exist, projection uses a user-entered monthly rate (default $0). Projection never claims to be a forecast from bank data.
- **Acceptance:** (1) Projection line renders from today to the target date. (2) Estimate label is exact. (3) Changing the target or date redraws the line.

### 35. Display net worth in a large-format hero number optimized for iPhone portrait.

- **Components:** Text `netWorthValue` styled ≥ 40px, full-width, centered; compact "$1.2k" abbreviation toggle for small screens.
- **Data:** Same `calcNetWorth` value; formatter `formatHeroCurrency` with abbreviation option.
- **Logic notes:** At 390px width the number never wraps or overflows; negative totals show red with a minus sign. Abbreviation toggle persisted in app settings.
- **Acceptance:** (1) Renders on one line at 390px width. (2) ≥ 40px font, high contrast. (3) Abbreviation toggle switches $1,234,567 ↔ $1.2M.

### 36. Add per-institution net-worth subtotals for users with multiple institutions.

- **Components:** Grouped list `listNWByInstitution`: institution name, subtotal, account count.
- **Data:** Transformer groups included accounts by `institution`, sums signed contributions per group, sorts desc.
- **Logic notes:** Blank institution labeled "No institution". Subtotals respect sample toggle and include/exclude flags. Subtotals sum to the hero total.
- **Acceptance:** (1) Each institution shows one correct subtotal. (2) Group subtotals add up to the hero number. (3) Sample toggle re-sorts and re-sums correctly.

### 37. Compare net worth across custom date ranges selected by the user.

- **Components:** DateRange picker `rangeNWCompare`; two summary cards "Start" and "End" with delta.
- **Data:** Transformer picks the nearest snapshot on/before each end of the range (latest snapshot ≤ date); delta = end − start with up/down indicator.
- **Logic notes:** If no snapshot exists within a range edge, show "no snapshot in range" rather than interpolating. Ranges cannot start in the future.
- **Acceptance:** (1) Delta equals the two selected snapshots' difference. (2) Missing snapshots show the empty message, not a guess. (3) Swapping range ends swaps the cards.

### 38. Show a waterfall chart of net-worth changes between two snapshots by account.

- **Components:** Chart `chartNWWatefall` (waterfall: start total → per-account deltas → end total); snapshot pickers `selectSnapA`, `selectSnapB`.
- **Data:** Transformer diffs the JSON breakdowns of the two snapshots per account_id (accounts present in only one snapshot shown as "added"/"removed").
- **Logic notes:** Bars signed and labeled with amounts; start and end totals as anchor bars. Defaults to the two most recent snapshots.
- **Acceptance:** (1) Anchor bars equal the two snapshot totals. (2) Middle bars sum to the delta. (3) Added/removed accounts labeled as such.

### 39. Prevent sample accounts from inflating real net worth by defaulting the sample filter to exclude them.

- **Components:** Global toggle `showSamples` (temporary state) defaulting to OFF on every app load.
- **Data:** All net-worth queries/transformers filter `is_sample` unless `showSamples.value` is true. Default is set in the app's startup script, not remembered across sessions.
- **Logic notes:** First-run seeding sets all sample accounts is_sample=true so the default-off toggle yields a clean real-only total. A visible chip shows "Sample data hidden" when off.
- **Acceptance:** (1) Fresh load excludes all sample rows from the total. (2) Enabling the toggle adds them back visibly badged. (3) Toggle state resets to off on reload.

### 40. Add a "reconcile" checklist prompting the user to confirm each balance before a snapshot.

- **Components:** Modal `modalReconcile` listing every included account with current balance and a per-row "Confirm" checkbox; "Confirm all & snapshot" button enabled only when all rows are checked.
- **Data:** Confirmations written to `reconciliations` (one row per account per snapshot); `createSnapshot` runs only after all confirmations saved. Unchecking is allowed before the snapshot is taken.
- **Logic notes:** "Skip reconcile" option available but logs `reconciled=false` on the snapshot row. Checklist re-opens if a balance is edited mid-reconcile.
- **Acceptance:** (1) Snapshot button in the modal stays disabled until all rows confirmed. (2) Confirmation rows persist with timestamps. (3) Skipping is explicit and recorded.
