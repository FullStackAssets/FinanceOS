# FinanceOS — Section 12: Goal contributions (items 221–240)

## Tables touched

Existing: `goals` (id, goal_name, target, saved, deadline, is_sample), `accounts`, `transactions`, `budgets`.

New table — `goal_contributions`: id (pk), goal_id (fk → goals.id), date (date), amount (number, signed: positive = contribution in, negative = withdrawal out), source_account_id (fk → accounts.id, nullable), note (text), is_sample (bool). Index on (goal_id, date).

New table — `audit_log` (shared): id (pk), actor (text), timestamp (timestamp), table_name (text), record_id (text), action (text: create/edit/delete), before_value (text/JSON), after_value (text/JSON), entry_method (text: manual/import/sample).

New table — `contribution_plans` (item 223): id (pk), goal_id (fk), amount (number), day_of_month (number), active (bool), note (text), is_sample (bool).

New table — `roundup_rules` (item 237): id (pk), account_id (fk, nullable), linked_transaction_id (fk, nullable), round_to (number, default 1), roundup_amount (number), goal_id (fk), date (date), confirmed (bool), is_sample (bool).

## Cross-cutting rules

- Every screen carries the persistent banner "Manual tracking — no bank connection yet." at top (dismissible per session only, restored on load).
- Every row with is_sample=true shows a visible "SAMPLE" badge; the global "hide sample data" toggle excludes sample rows from all totals, charts, and goal math.
- Goal `saved` is a derived total: sum of signed contribution amounts per goal (excluding samples when hidden). The Goals screen recomputes on every contribution change — no stale cached totals.
- Nothing auto-moves money: contributions, withdrawals, plans, and round-ups only create records the user confirms. Nothing here ever presents as a live bank feed.

---

### 221. Record each contribution as a dated entry with amount and source account.

- **Components:** "Contribute" button on each goal card opens a Modal (`contributeModal`) with a Date picker (default today), Number input (amount, min 0.01), Select (source account, from `accounts` list, excludes archived), Text input (note). Save button writes to `goal_contributions` and triggers recompute of goal `saved`.
- **Data:** Query `insertContribution` (Retool DB insert), then `recalcGoalSaved`: `UPDATE goals SET saved = (SELECT COALESCE(SUM(amount),0) FROM goal_contributions WHERE goal_id = ? AND is_sample = ?) WHERE id = ?`.
- **Logic notes:** Positive amounts = contributions; withdrawals use the separate withdrawal flow (item 229) but are stored as negative amounts in the same table. Source account balance is reduced manually via the accounts screen (item 205) — the spec links both steps in the confirmation toast. Sample accounts produce is_sample=true rows automatically.
- **Acceptance:** (1) Saving a contribution creates exactly one dated row with the chosen account. (2) Goal `saved` increases by the amount immediately. (3) Missing date or amount blocks save with a field-level error.

### 222. Add a quick-contribute keypad with common amounts for iPhone entry.

- **Components:** Inside `contributeModal`, a Button group row: $10, $25, $50, $100, $250, plus a "custom" toggle revealing the Number input. Keypad buttons are 44px+ touch targets in a 3-column grid. Tapping a preset sets the amount and highlights it.
- **Data:** Presets are a local constant array; selected preset stored in `quickAmount` temp state. The same `insertContribution` query is reused — only the amount source differs.
- **Logic notes:** One-handed iPhone use: keypad sits above the fold, Save is a full-width primary button reachable by thumb. Presets do not replace the custom field — both write identical rows. Default selected preset: none, to avoid accidental taps.
- **Acceptance:** (1) Tapping a preset fills the amount and shows a selected state. (2) Custom entry still works and isn't overwritten by preset taps afterward unless re-tapped. (3) All targets ≥44px at 390px width.

### 223. Support scheduled contribution plans (e.g., $200 on the 1st) as manual reminders, not auto-transfers.

- **Components:** Goal detail "Plans" tab: list of plans (amount, day of month, active toggle), "Add plan" form (Number input, day-of-month Select 1–28, note). Due plans surface as in-app reminder cards on the Overview and in the notification center.
- **Data:** `contribution_plans` table. Query `duePlans`: plans where active=true and day_of_month <= today AND no contribution recorded for this goal in the current month. Confirming a reminder opens `contributeModal` prefilled with the plan amount.
- **Logic notes:** Plans NEVER auto-post — they are reminders only; the user must tap "Log contribution" to create the row. Day 29–31 avoided by restricting to 1–28 to keep monthly math simple; note field records intent ("payday contribution"). Pausing = active=false, one tap.
- **Acceptance:** (1) A due plan shows a reminder card until the user logs or snoozes it. (2) No contribution row is created without explicit user confirmation. (3) Editing/deleting a plan never touches existing contribution rows.

### 224. Show a contribution calendar marking days contributions were recorded.

- **Components:** Goal detail "Calendar" tab: Retool Calendar component bound to the current month, with day-cell markers. Month prev/next navigation. Tapping a marked day opens a popover listing that day's contributions.
- **Data:** Query `contributionsByDay`: `SELECT date, COUNT(*) c, SUM(amount) total FROM goal_contributions WHERE goal_id = ? AND date BETWEEN ? AND ? GROUP BY date`. Transformer maps results to calendar event objects with dot markers and amounts.
- **Logic notes:** Withdrawals render in red, contributions in green on the same day. Sample rows show a "SAMPLE" chip in the day popover. Empty months show a neutral empty state ("No contributions yet this month").
- **Acceptance:** (1) Every contribution date in the viewed month carries a marker. (2) Tapping a marked day lists exactly that day's entries. (3) Month navigation preserves the selected goal.

### 225. Allow editing or deleting a contribution with the goal total recalculated.

- **Components:** Contribution history list on goal detail: each row has Edit (opens modal prefilled) and Delete (confirm dialog). After either action, a toast confirms and the goal progress bar updates.
- **Data:** `updateContribution` and `deleteContribution` queries (Retool DB update/delete by id), followed by `recalcGoalSaved`. Before/after values captured into `audit_log`.
- **Logic notes:** Deleting is a hard delete of the contribution row (goal totals are derived, so nothing else breaks); the audit log preserves the deleted values for traceability. Editing amount or date re-runs all derived views (calendar, monthly totals, streaks) via query refetch chain.
- **Acceptance:** (1) Edit changes the row and the goal `saved` reflects the new amount immediately. (2) Delete removes the row and reduces `saved` accordingly. (3) Both actions write an audit_log entry with before/after values.

### 226. Display a running total of contributions per goal per month.

- **Components:** Goal detail "History" tab: table grouped by month (YYYY-MM header rows) with per-month subtotal row. Current month pinned at top.
- **Data:** Query `monthlyContributionTotals`: `SELECT strftime('%Y-%m', date) ym, SUM(amount) total, COUNT(*) n FROM goal_contributions WHERE goal_id = ? GROUP BY ym ORDER BY ym DESC`. Transformer splits into per-month groups for the table.
- **Logic notes:** Withdrawals subtract from the month's total (signed amounts). Sample contributions are excluded when the sample toggle hides them, with a footnote noting the exclusion. Months with no contributions are omitted (not zero-filled) to keep the list honest.
- **Acceptance:** (1) Each month's subtotal equals the sum of its rows. (2) The sum of all monthly subtotals equals goal `saved`. (3) Toggling sample visibility updates the totals.

### 227. Add contribution notes ("tax refund", "birthday money").

- **Components:** Text input "Note" on the contribute modal (optional, 140-char limit, placeholder "e.g. tax refund, birthday money"). Notes display as a muted subtitle under each contribution row and in the day popover.
- **Data:** `note` column on `goal_contributions`; included in insert and update queries. Notes are searchable via the global notes search (section 22).
- **Logic notes:** Notes are plain text, no formatting; leading/trailing whitespace trimmed on save. Empty notes store NULL, not empty string, for clean exports.
- **Acceptance:** (1) A note saved with a contribution appears on its history row. (2) Editing a contribution can add, change, or clear the note. (3) Notes survive CSV export (item 239).

### 228. Warn when a contribution exceeds the goal's remaining target.

- **Components:** Inline warning banner inside `contributeModal` (amber): "This is $X over your remaining target of $Y." Appears live as the amount is typed. Save remains allowed — the warning is advisory, with the overage amount shown.
- **Data:** Transformer `remainingTarget = goal.target - goal.saved` (recomputed from current `saved`). Warning shows when `amount > remainingTarget && remainingTarget > 0`.
- **Logic notes:** Overshooting is legitimate (goals can be overfunded), so this never blocks. If `remainingTarget <= 0`, the banner instead reads "Goal already fully funded — this adds surplus." Sample-mode math uses sample-inclusive saved.
- **Acceptance:** (1) Typing an amount above the remaining target shows the exact overage. (2) Saving still succeeds. (3) No warning appears when the amount is within the remaining target.

### 229. Support withdrawing from a goal back to an account with a reason field.

- **Components:** Goal detail "Withdraw" button opens `withdrawModal`: Number input (amount), Select (destination account), required Text input "Reason". Confirm writes a negative-amount row.
- **Data:** Insert into `goal_contributions` with amount = -X, note = reason, source_account_id = destination account. `recalcGoalSaved` runs afterward. Audit log records the withdrawal.
- **Logic notes:** Reason is required (empty reason blocks save) — this keeps the history explainable. Withdrawing more than `saved` is blocked with an error ("Can't withdraw more than the goal holds"). The destination account's balance is increased manually via the accounts screen, linked in the confirmation toast.
- **Acceptance:** (1) Withdrawal reduces goal `saved` by the exact amount. (2) Missing reason blocks the save. (3) Over-withdrawal is rejected with a clear message. (4) The withdrawal row appears in history marked as a withdrawal.

### 230. Show net contributions (in minus out) per goal in the detail view.

- **Components:** Goal detail header stat trio: "Contributed in" (sum of positive), "Withdrawn" (abs of negative sum), "Net" (signed total, large). Net equals the goal `saved` value shown on the progress bar.
- **Data:** Query `contributionSums`: `SELECT SUM(CASE WHEN amount>0 THEN amount ELSE 0 END) in_total, SUM(CASE WHEN amount<0 THEN amount ELSE 0 END) out_total FROM goal_contributions WHERE goal_id = ?`. Net = in_total + out_total.
- **Logic notes:** If there are no withdrawals, the "Withdrawn" stat hides to reduce clutter. All three figures respect the sample-data toggle. The numbers are labeled "from your manual entries."
- **Acceptance:** (1) Net always equals goal `saved`. (2) In minus out arithmetic is exact to the cent. (3) Hiding samples updates all three stats consistently.

### 231. Add a streak counter for consecutive months with at least one contribution.

- **Components:** Goal detail badge: flame icon + "N-month streak". Tooltip lists the streak months. Resets are explained ("streak broken in March — no contributions").
- **Data:** Transformer `streakCalc`: from monthly contribution presence (positive-sum months only), walk backward from the current (or last contributing) month counting consecutive YYYY-MM values.
- **Logic notes:** Only months with net positive contributions count; withdrawal-only months break the streak. The current partial month counts if it has a contribution. Sample contributions count only when samples are visible.
- **Acceptance:** (1) Three consecutive contributing months show "3-month streak". (2) A gap month resets the count to the run after the gap. (3) The tooltip names the exact months in the streak.

### 232. Display contributions as a bar chart per month for the selected goal.

- **Components:** Goal detail "Chart" tab: Retool Bar Chart, x = month (last 12), y = net contributions that month. Bars green for positive months, red for negative (net-withdrawal) months. Hover tooltip shows in/out/net.
- **Data:** Reuses `monthlyContributionTotals`; transformer shapes to chart series. Toggle switches between net view and in-vs-out grouped bars.
- **Logic notes:** Chart caption: "From your manual entries." Empty months in the 12-month window render as zero-height gaps, not interpolated. Sample exclusion follows the global toggle.
- **Acceptance:** (1) Each bar's value matches the monthly table subtotal. (2) Negative-net months render below the axis in red. (3) Tooltip shows in, out, and net for the hovered month.

### 233. Allow bulk-importing past contributions from a CSV with date and amount columns.

- **Components:** Goal detail "Import" button opens a File Dropzone accepting .csv. Preview table shows parsed rows (date, amount, note) with row-level validation errors highlighted. "Import N rows" confirms; progress bar during insert.
- **Data:** JS parses CSV (date, amount, optional note columns; header row auto-detected). Validation: parseable date, numeric non-zero amount, date not in the future (warns but allows). Bulk insert via Retool DB bulk insert; all rows tagged entry_method='import' and is_sample=false. Audit log records the batch.
- **Logic notes:** Amounts are signed — negative values import as withdrawals. A downloadable CSV template (date,amount,note + one example row) is linked from the import dialog. Duplicate detection: rows matching an existing (date, amount, note) triple are flagged "possible duplicate" and can be skipped.
- **Acceptance:** (1) Valid CSV imports all rows with correct dates/amounts. (2) Invalid rows are listed with reasons and excluded. (3) Goal `saved` reflects the import immediately. (4) The batch is undoable via the audit log entry.

### 234. Tag contributions made from sample accounts so they don't pollute real goal math.

- **Components:** When the source account picker has a sample account selected, the contribute modal shows an inline note: "Source is a sample account — this contribution will be flagged SAMPLE." The history row carries the SAMPLE badge.
- **Data:** On save, is_sample is set from the source account's is_sample flag (inherited). All goal-math queries filter `is_sample = false` when the global sample toggle hides samples.
- **Logic notes:** This is inheritance, not user choice — the user cannot mark a sample-sourced contribution as real at entry time. Converting to real happens only via the explicit sample→real conversion flow (section 18), which is audit-logged.
- **Acceptance:** (1) Contributions from sample accounts are created with is_sample=true. (2) With samples hidden, goal progress excludes them. (3) With samples visible, a side-by-side real-vs-sample note appears under the progress bar.

### 235. Add a "catch up" calculator showing what is needed to get back on deadline pace.

- **Components:** Goal detail "Catch up" card (shown when behind pace): "To hit $X by [deadline], contribute $Y/month for the next N months" plus a per-paycheck variant if a payday schedule exists. Input lets the user test a custom monthly amount and see the resulting completion date.
- **Data:** Transformer: `monthsLeft = months between now and deadline` (min 1); `needed = (target - saved) / monthsLeft`. On-pace benchmark = `(target - saved_at_start) / total_months` from goal creation; behind-pace flag when actual monthly average < benchmark.
- **Logic notes:** All figures labeled estimates from manual entries. If the deadline has passed or saved >= target, the card hides. The calculator never writes data — it's read-only planning.
- **Acceptance:** (1) The monthly figure exactly covers the shortfall over the remaining months. (2) Custom-amount input recomputes the completion date live. (3) Fully-funded or deadline-less goals hide the card.

### 236. Show per-goal average contribution size from manual history.

- **Components:** Goal detail stat: "Average contribution: $X (N contributions)". Tooltip breaks out median as well.
- **Data:** Query: `SELECT AVG(amount), COUNT(*) FROM goal_contributions WHERE goal_id = ? AND amount > 0 AND is_sample = ?`. Withdrawals excluded from the average.
- **Logic notes:** Average uses positive contributions only — mixing in withdrawals would mislead. Displayed with the count so "$500 average" over 2 contributions isn't mistaken for a habit. Sample toggle respected.
- **Acceptance:** (1) Average equals total positive contributions ÷ count. (2) Withdrawals don't affect the figure. (3) Zero contributions show "—" not $0.

### 237. Support rounding-up style rules applied manually ("add $3 to round this coffee to $5").

- **Components:** On any spend transaction row, a "Round up" action opens a mini-modal: shows the round-up amount (round_to - (spend mod round_to)), goal picker, Confirm. Confirmed round-ups appear in a "Pending round-ups" queue on the Goals screen until logged as contributions.
- **Data:** `roundup_rules` table stores unconfirmed suggestions; confirming writes a normal `goal_contributions` row (note auto-filled "round-up: <description>") and marks the rule confirmed. Nothing is created without the Confirm tap.
- **Logic notes:** Purely manual — the app suggests, the user confirms each one; there is no auto-sweep. Round-up to $1 default, configurable per rule. "Round up all this week" bulk-confirms the queue with per-row confirmation still available.
- **Acceptance:** (1) A $4.20 coffee suggests a $0.80 round-up to the chosen goal. (2) No contribution exists until the user confirms. (3) Confirmed round-ups appear in contribution history with the auto note.

### 238. Display a leaderboard-style ranking of goals by percent complete.

- **Components:** Goals screen "Rankings" toggle: switches the list to a ranked view sorted by saved/target descending, with rank numbers (#1, #2…), percent bars, and a "most improved this month" badge (largest percentage-point gain).
- **Data:** Transformer over `goals` + contribution sums: `pct = saved / target`. Sort desc; ties broken by higher saved. Monthly gain from `monthlyContributionTotals`.
- **Logic notes:** Framed as personal motivation, not competition — no sharing, no external comparison. Paused goals (item 214) are excluded from ranking. Sample goals rank only when samples are visible.
- **Acceptance:** (1) Order is strictly by percent complete descending. (2) Percentages match each goal's progress bar. (3) Paused goals don't appear in the ranking.

### 239. Add an export of all contributions to CSV per goal.

- **Components:** Goal detail "Export CSV" button. Export honors the sample toggle (exports what's visible) and includes columns: date, amount, source_account, note, is_sample, entry_method.
- **Data:** Query contributions ordered by date; JS builds CSV with UTF-8 BOM; filename `financeos-contributions-<goal-name>-<YYYY-MM-DD>.csv`. Export stamped with "Manual tracking — no bank connection" and export date in a header comment row.
- **Logic notes:** If samples are included, the file carries the sample watermark note and the is_sample column. Export action is logged in export history (section 25). Amounts export signed so re-import round-trips correctly.
- **Acceptance:** (1) CSV opens correctly in spreadsheet apps with UTF-8 encoding. (2) Every visible contribution row appears exactly once. (3) Sample rows are identifiable via the is_sample column.

### 240. Log every contribution, edit, and withdrawal in the audit log.

- **Components:** No new UI — the audit log viewer (section 38) picks these up automatically. Each entry shows: timestamp, actor, action (contribution added / edited / deleted / withdrawn), goal name, amount, before/after.
- **Data:** Every insert/update/delete on `goal_contributions` writes an `audit_log` row with table_name='goal_contributions', record_id, action, before/after JSON snapshots, entry_method (manual/import/sample). Bulk imports log one entry per row plus a batch summary entry.
- **Logic notes:** Audit entries are append-only — no edit/delete of log rows. The per-goal history timeline (section 38) filters the log to one goal_id for a full contribution audit trail.
- **Acceptance:** (1) Adding a contribution creates an audit entry within the same action. (2) Edits store before and after values. (3) Withdrawals are logged as a distinct action type. (4) Import batches are traceable to their source file and row count.
