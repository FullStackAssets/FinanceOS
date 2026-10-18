# FinanceOS — Month-End Review Flow (items 721–740)

Builder spec for a guided, self-serve month-end review inside the FinanceOS Retool app. The review walks the user through 6 steps — spend, budgets, bills, goals, net worth, notes — over their own manually entered data, saves the completed review as an immutable monthly record, and supports pausing, early completion, streaks, and a one-page PDF summary.

## Tables touched

- **transactions** (existing): `id, account_id (fk), date, description, category, amount (negative=spend), is_sample`
- **budgets** (existing): `id, category, month, limit_amount`
- **goals** (existing): `id, goal_name, target, saved, deadline`
- **accounts** (existing): `id, name, type, institution, balance, updated_at, is_sample`
- **month_reviews** (new): `id (pk), month (text, e.g. "2026-10", unique), status (text: In progress / Completed / Preliminary), current_step (number), started_at (timestamp), completed_at (timestamp, nullable), total_income (number), total_spend (number), net_worth_snapshot (number), retrospective_note (text), adjustments_json (text, JSON of budget/goal adjustments made), snapshot_refs (text, JSON of source record IDs), archived (bool), is_sample (bool)`
- **audit_log** (assumed existing from section 38): `id (pk), action (text), detail (text), created_at (timestamp)` — used by item 740.

## Cross-cutting rules

- Every screen shows the persistent banner "Manual tracking — no bank connection yet."
- All sample rows carry `is_sample=true` + a SAMPLE badge; sample rows are excluded from every total when the sample toggle hides them.
- Nothing ever presents as a live bank feed — every figure is manual entry or SAMPLE.
- The review is a guided self-check checklist over the user's own data, never an automated month-close: every value shown is presented as "your entered data," and nothing is modified without the user confirming.

---

### 721. Build a guided month-end review: 6 steps covering spend, budgets, bills, goals, net worth, notes.

**Components:** `btnStartReview` (Button, "Start month review"), `wizardReview` (Wizard with 6 steps: Spend, Budgets, Bills, Goals, Net worth, Notes), `txtBanner` (persistent banner text), `lblMonth` (label bound to review month). **Data:** `qryMonthReview` (get or create `month_reviews` row for target month); `qryMonthTotals` (transactions this month by sample filter). **Logic notes:** The wizard opens only from an explicit "Start review" action; target month defaults to the just-ended month; each step has a per-step "Mark step complete" toggle stored in `current_step`; quitting preserves `In progress` status. **Acceptance:** (1) Review launches only via the explicit start button. (2) All 6 steps are navigable in order with per-step completion tracking. (3) `month_reviews` row is created on first start with `status=In progress`.

### 722. Step 1: review total spend vs income with category highlights.

**Components:** `statIncome`, `statSpend` (Statistic cards), `tblCategorySpend` (Table: category, spend, income, vs prior month), `chkStep1` (Checkbox, "I've reviewed this"). **Data:** `qryStep1Totals`: `SELECT SUM(CASE WHEN amount<0 THEN -amount END) spend, SUM(CASE WHEN amount>0 THEN amount END) income, category FROM transactions WHERE date >= {{monthStart}} AND date < {{nextMonth}} {{sampleFilter}} GROUP BY category`. **Logic notes:** `monthStart` computed in JS as the first day of the target month in the user's timezone; category highlights = top-3 spend categories and largest month-over-month deltas; SAMPLE badge on any row where `is_sample=true`. **Acceptance:** (1) Totals match the transactions table filtered by month and sample toggle. (2) Top-3 categories highlighted. (3) Checking the confirmation box advances `current_step`.

### 723. Step 2: grade each budget (hit/missed) and optionally adjust next month's limits.

**Components:** `tblBudgetGrades` (Table: category, limit, actual, status badge Hit/Missed), `numAdjustLimit` (Number input per row for next month's limit), `chkStep2`. **Data:** `qryBudgetGrades` joins `budgets` for the review month with actuals from transactions; `qryApplyAdjustments` upserts `budgets` rows for next month from edited values. **Logic notes:** grade = Hit if actual ≤ limit; adjustments never touch the current month's rows — they write only to the next month's `budgets`; edits preview as diffs before apply. **Acceptance:** (1) Every budget in the review month shows an accurate Hit/Missed grade. (2) Next-month limits are editable and saved only to the next month. (3) A diff preview is shown before applying.

### 724. Step 3: confirm all bills were marked paid; flag any missed.

**Components:** `tblBills` (Table: bill label, due date, amount, paid checkbox, status badge), `alertMissed` (Alert, red count of unpaid), `chkStep3`. **Data:** `qryBillsThisMonth` (bills table/query rows for the month, if defined in section 15). **Logic notes:** "missed" = unpaid and due date < today; user confirms each bill paid/not-paid manually — the spec never infers payment from a bank feed; the count feeds the review summary. **Acceptance:** (1) All bills for the month are listed with their paid status. (2) Missed bills are flagged in red with a count alert. (3) Confirming the step requires an explicit checkbox.

### 725. Step 4: review goal contributions and set next month's contribution targets.

**Components:** `tblGoalReview` (Table: goal, saved, target, contributions this month, progress bar), `numNextTarget` (Number input per goal for next month's contribution target), `chkStep4`. **Data:** `qryGoalContributions` (sum of transactions tagged to each goal this month); `qrySaveTargets` updates `goals` (target column or a `next_month_target` field). **Logic notes:** targets are forward-looking only — they never rewrite history; if the user has zero goals the step is skipped per item 733. **Acceptance:** (1) Each goal shows this month's contributions and progress. (2) Next-month targets are editable and persist. (3) Step auto-skips cleanly when no goals exist.

### 726. Step 5: take or confirm the month-end net-worth snapshot.

**Components:** `statNetWorth` (Statistic: sum of `accounts.balance`), `btnConfirmSnapshot` / `btnRetakeSnapshot` (Buttons), `tblSnapshotAccounts` (Table of per-account balances), `chkStep5`. **Data:** `qryNetWorth`: `SELECT SUM(balance) FROM accounts {{sampleFilter}}`; writes `net_worth_snapshot` into the `month_reviews` row. **Logic notes:** snapshot is taken at user confirmation time, not automatically — the user reviews per-account balances first; retaking overwrites only before completion; SAMPLE rows excluded when the toggle hides them. **Acceptance:** (1) Snapshot equals the sum of displayed account balances. (2) Retake is possible before review completion, locked after. (3) Value is stored on the `month_reviews` row.

### 727. Step 6: write a month retrospective note saved with the closed month.

**Components:** `txtRetrospective` (Text Area, 500-char guidance prompt), `btnFinishReview` (Button, primary, enabled only when all steps checked), `chkStep6`. **Data:** `qrySaveReview` updates `month_reviews` (`retrospective_note`, `completed_at`, `status=Completed` or `Preliminary`). **Logic notes:** note is optional but prompted; finishing is a single explicit action; after finish the row is locked for editing except per item 739's immutability rule. **Acceptance:** (1) Note saves to the month's review row. (2) Finish requires all six step confirmations. (3) `completed_at` timestamp is recorded on finish.

### 728. Show a review progress bar and allow pausing/resuming the flow.

**Components:** `prgReview` (Progress bar, value = completed steps / 6), `btnPauseReview` (Button), `cardResume` (Card shown on the overview screen when a review is `In progress`: month, steps done, "Resume" button). **Data:** `qryPauseReview` sets `current_step` and keeps `status=In progress`; `qryResume` reopens the wizard at `current_step`. **Logic notes:** progress persists across sessions via the `month_reviews` row — closing the browser never loses progress; pause is a no-op state (nothing auto-runs while paused). **Acceptance:** (1) Progress bar reflects the true fraction of completed steps. (2) Resume returns to the exact saved step. (3) Progress survives app reload.

### 729. Carry budget adjustments from the review directly into next month's budgets.

**Components:** `tblAdjustmentPreview` (Table: category, old limit, new limit, delta), `btnApplyAdjustments` (Button). **Data:** `qryApplyAdjustments`: upsert `budgets` rows for next month keyed by (category, month). **Logic notes:** adjustments are collected during Step 2 and applied as one batch at finish; application writes `adjustments_json` into the review row for the audit trail; re-running the review never double-applies — existing next-month rows are overwritten, not duplicated. **Acceptance:** (1) Next month's budget rows match the review's adjusted values exactly. (2) `adjustments_json` records what was applied. (3) Re-applying is idempotent.

### 730. Display last month's retrospective note at the start of the next review.

**Components:** `cardLastNote` (Card, first screen of the review wizard: last month's note text + completion date), `btnDismissNote`. **Data:** `qryLastReview` selects the most recent completed `month_reviews` row before the target month. **Logic notes:** if no prior review exists, the card shows a friendly empty state ("No previous review yet"); the note is read-only here — editing happens only inside its own month. **Acceptance:** (1) The wizard opens showing the prior month's note. (2) Empty state renders when there is no prior review. (3) The note shown is read-only.

### 731. Add a review completion badge per month on the calendar.

**Components:** `calendarReview` (Calendar component, monthly view, custom event rendering), `tagReviewBadge` (badge styled ✓ / Preliminary). **Data:** `qryReviewCalendar` selects `month, status` from `month_reviews` for the visible range. **Logic notes:** badge appears on the month's last day cell; Completed = green check, Preliminary = amber dot, In progress = grey ring; clicking the badge opens that month's review summary. **Acceptance:** (1) Each month with a review row shows the correct status badge. (2) Badge click opens the matching review. (3) Months with no review show no badge.

### 732. Generate a one-page month-end summary PDF from the review.

**Components:** `btnExportSummary` (Button, "Export summary PDF"), `pdfSummary` (PDF export component rendering a fixed one-page template). **Data:** `qrySummaryData` pulls the review row, totals, grades, snapshot, and retrospective note; template stamps the manual-tracking banner and export date. **Logic notes:** one-page template: month header, income/spend, budget grades, net worth, goals, retrospective note; exports are client-side only (no data leaves the browser); SAMPLE badge watermarked when samples included. **Acceptance:** (1) PDF contains all six review sections on one page. (2) Banner text and export date appear on the PDF. (3) Export works with no server round-trip for the file itself.

### 733. Skip logic: hide goal steps if the user has no goals.

**Components:** conditional visibility on the Goals wizard step and goal widgets (`hidden` = `qryGoalCount.data.count === 0`). **Data:** `qryGoalCount`: `SELECT COUNT(*) FROM goals`. **Logic notes:** skip is evaluated at review start and re-evaluated if goals change mid-review; progress math adjusts (e.g., 5 steps instead of 6); step numbering shown to the user stays contiguous. **Acceptance:** (1) With zero goals, the Goals step does not appear and the flow still completes. (2) With ≥1 goal, the step appears. (3) Progress denominator updates to match the visible step count.

### 734. Show streaks of consecutive completed reviews.

**Components:** `statStreak` (Statistic: current streak), `lblStreakDetail` (text: "5 months in a row"). **Data:** `qryStreak`: fetch completed reviews ordered by month desc; compute in JS the longest trailing run of consecutive calendar months with `status=Completed` (Preliminary does not count). **Logic notes:** a missed month resets the current streak to 0; timezone-safe month arithmetic on `YYYY-MM` keys; streak counts only completed reviews, never in-progress ones. **Acceptance:** (1) Streak increments only for consecutive completed months. (2) A skipped month resets it to 0. (3) Preliminary reviews do not extend the streak.

### 735. Allow completing the review early (from the 25th) with a "preliminary" label.

**Components:** `btnEarlyFinish` (Button, visible only on/after the 25th of the review month), `tagPreliminary` (amber "PRELIMINARY" badge). **Data:** `qryEarlyFinish` sets `status=Preliminary` instead of `Completed`. **Logic notes:** early finish requires all steps checked, same as full finish; a preliminary review can be re-opened and re-finished as Completed after month-end — re-finish flips the status and records the final `completed_at`; the badge shows wherever the review is displayed. **Acceptance:** (1) Early finish is available only from the 25th. (2) Preliminary reviews show the label everywhere. (3) Re-finishing after month-end converts status to Completed.

### 736. Remind on the 1st if last month's review wasn't completed.

**Components:** `alertReviewReminder` (Alert banner on the overview screen, dismissible), `btnGoToReview` (Button inside the alert). **Data:** `qryReminder`: on the 1st–7th of the month, check whether last month has a `Completed` review; if not, show the alert. **Logic notes:** the reminder is an in-app banner only — no email, no push; dismissing hides it for the current day (persists via localStorage); it never blocks other app use. **Acceptance:** (1) Alert appears on the 1st when last month's review is incomplete. (2) It disappears once the review is completed. (3) Dismiss hides it until the next day.

### 737. Display review insights: "3rd month under budget on Food."

**Components:** `listInsights` (List view of generated insight strings). **Data:** `qryInsights` pulls the trailing 6 months of review rows and budget actuals; a transformer generates templated insights. **Logic notes:** insight templates: N-th consecutive month under/over budget per category, biggest swing vs prior month, goal streaks; only insights supported by data are shown (no invented claims); SAMPLE rows are excluded when the toggle hides them. **Acceptance:** (1) Each insight matches the underlying monthly data. (2) No insight appears without supporting data. (3) Insights update as the sample toggle changes.

### 738. Support reviewing with a partner in read-only shared mode.

**Components:** `btnShareReview` (Button, generates a share link or PDF), `viewShared` (read-only review summary layout), `tagReadOnly` (badge: "Shared — read only"). **Data:** shared view is served from the same review row but all inputs disabled; no new table required — sharing is via the exported PDF (item 732) or a read-only Retool share. **Logic notes:** partner cannot edit anything — inputs, finish, and adjustment buttons are all disabled in shared mode; sharing a review shares only that month's review data, not full transaction history; the read-only badge is always visible in shared mode. **Acceptance:** (1) Shared view renders with every input disabled. (2) Badge clearly marks read-only mode. (3) No edit action is possible in shared mode.

### 739. Archive each completed review immutably with its snapshot references.

**Components:** `btnArchive` (runs automatically on finish; no user control needed), `viewArchive` (read-only rendering of an archived review). **Data:** on finish, `qryArchiveReview` sets `archived=true` and writes `snapshot_refs` (JSON: account IDs + balances, transaction ID list hash, budget row IDs). **Logic notes:** archived reviews cannot be edited by any app query — edits are blocked by a guard query that rejects writes where `archived=true`; only `status`/`completed_at` of a Preliminary→Completed re-finish may change before archiving locks. **Acceptance:** (1) After finish, every edit path for that review is rejected. (2) `snapshot_refs` captures the source records. (3) The archived review renders read-only with its references listed.

### 740. Log review completion and all adjustments in the audit log.

**Components:** none new — writes to `audit_log` via `qryAuditReview`. **Data:** on review finish, insert one row: action="review_completed", detail = month + status + adjustments summary; on adjustments apply, insert action="budgets_adjusted" with the per-category diffs. **Logic notes:** logging is part of the finish transaction — if the audit insert fails, the finish is rolled back and the user sees an error; log detail never contains account balances beyond what the review already shows. **Acceptance:** (1) Every completed review produces exactly one review_completed audit row. (2) Every applied adjustment set produces one budgets_adjusted row with diffs. (3) A failed audit write blocks the finish rather than silently skipping.
