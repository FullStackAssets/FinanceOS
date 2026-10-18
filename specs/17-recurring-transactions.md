# FinanceOS Builder Spec — §17 Recurring Transactions (Items 321–340)

## Section overview
A recurring-templates module where each template describes a repeating money pattern (subscription, rent, paycheck). Templates NEVER post transactions on their own: a generator creates due instances as "pending" items, and the user confirms or skips each one individually, which then writes a real `transactions` row. This preserves the manual-entry contract while removing the typing tedium of repeat entries.

## Tables touched
- `accounts` (existing): id, name, type, institution, balance, updated_at, is_sample.
- `transactions` (existing): id, account_id (fk), date, description, category, amount (negative=spend), is_sample.
- NEW `recurring_templates`: id (pk), description (text), category (text: Housing / Food / Transport / Income / Fun / Bills / Other), amount (number, negative=spend), account_id (fk → accounts), frequency (text: weekly / biweekly / monthly / quarterly / yearly / custom), custom_interval_days (number, nullable), start_date (date), end_date (date, nullable), next_due_date (date), status (text: active / paused), last_confirmed_at (timestamp, nullable), trusted (bool, default false), is_sample (bool).
- NEW `recurring_instances`: id (pk), template_id (fk → recurring_templates), due_date (date), status (text: pending / confirmed / skipped), confirmed_amount (number, nullable), confirmed_at (timestamp, nullable), transaction_id (fk → transactions, nullable), is_sample (bool).
- NEW `audit_log`: id (pk), actor (text), action (text), table_name (text), record_id (text), before_json (text, nullable), after_json (text, nullable), created_at (timestamp).

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen, including the recurring-transactions screen and the confirmation queue.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data toggle hides samples, sample templates and their instances are excluded from all lists, totals, and projections.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.
- Recurring instances are NEVER auto-posted — each one requires an explicit user confirmation, which creates the matching `transactions` row.

---

### 321. Build a recurring-transactions screen listing templates with frequency and next date.
- Components: New "Recurring" tab in the app nav; Table `tblRecurringTemplates` (columns: description, category pill, amount, frequency label, next_due_date, status toggle, annual projection); "New template" button opening a modal.
- Data: `qryRecurringTemplates` = `SELECT * FROM recurring_templates WHERE is_sample = {{ !hideSamples.value }} ORDER BY next_due_date ASC`; join accounts for the account chip. Transformer `trNextDue` recomputes next_due_date from frequency + start_date + end_date on load.
- Logic notes: Next date = start_date advanced by the frequency until > today. Paused templates show grayed with "Paused" badge. Sample templates hidden when the toggle is off.
- Acceptance: (1) Screen lists every active template with correct frequency and next date. (2) Banner visible at top. (3) Hiding samples removes sample templates. (4) Empty state with "Create your first recurring template" CTA when none exist.

### 322. Add a create-recurring form: description, category, amount, account, frequency, start date.
- Components: Modal `mdlRecurringForm` with Text Input (description), Select (category, fixed 7-taxonomy), Number Input (amount with sign hint: negative = spend), Select (account from `qryAccounts`), Segmented control (frequency), conditional Number Input (custom interval days), Date Picker (start date, default today), Date Picker (end date, optional).
- Data: `qryCreateTemplate` INSERT into `recurring_templates`; compute `next_due_date` = start_date via transformer; `qryAudit` writes a create entry.
- Logic notes: Amount 0 rejected; category restricted to the fixed taxonomy; custom frequency requires interval_days ≥ 1. Income amounts positive, spends negative — mirror the transaction sign convention.
- Acceptance: (1) Valid form creates a template row with correct next_due_date. (2) Zero amount blocked with inline error. (3) Audit log records the create. (4) New template appears in the list immediately.

### 323. Generate due instances as manual "pending" items the user confirms one by one — never auto-posted.
- Components: "Generate due" button `btnGenerateDue` on the Recurring screen; List `lstPendingInstances` showing due instances with template name, due date, amount.
- Data: JS query `qryGenerateDue`: for each active template with next_due_date ≤ today and no pending instance for that date, INSERT into `recurring_instances` with status='pending'. Runs on screen load and on button press. NO insert into `transactions` happens here — that only occurs on confirmation.
- Logic notes: Idempotent — re-running skips dates that already have an instance row. Templates past end_date generate nothing. This is the hard guardrail: the generator path has no write permission to `transactions`.
- Acceptance: (1) Due templates produce pending instances without touching `transactions`. (2) Re-running adds no duplicates. (3) Ended templates produce nothing. (4) Code review confirms no auto-post path exists.

### 324. Show a confirmation queue where each due instance is approved or skipped.
- Components: "Confirm" queue section `secConfirmQueue` at top of the Recurring screen; each row has Approve (check) and Skip (×) buttons; row shows description, due date, expected amount, account chip, template-name tag.
- Data: `qryPendingInstances` joins instances to templates where status='pending'. Approve → `qryConfirmInstance`: INSERT into `transactions` (date=due_date, description, category, account_id, amount, is_sample copied from template), UPDATE instance status='confirmed', confirmed_at=now, transaction_id set; Skip → UPDATE status='skipped'. Both advance the template's next_due_date.
- Logic notes: Confirming is_sample templates writes is_sample=true transactions so samples never pollute real totals. Skipped instances never touch `transactions`.
- Acceptance: (1) Approve creates exactly one transaction row. (2) Skip creates none and marks the instance skipped. (3) next_due_date advances in both cases. (4) Queue empties when all due instances are handled.

### 325. Support frequencies: weekly, biweekly, monthly, quarterly, yearly, custom interval.
- Components: Frequency segmented control in the template form: Weekly / Biweekly / Monthly / Quarterly / Yearly / Custom; conditional "every N days" number input for Custom.
- Data: Transformer `trAdvanceDate(date, frequency, customDays)`: weekly +7d, biweekly +14d, monthly +1 month (clamp day to month end, e.g. Jan 31 → Feb 28), quarterly +3 months, yearly +12 months, custom +N days. Stored in `frequency` / `custom_interval_days`.
- Logic notes: Month-end clamping documented in the help tooltip ("a 31st template lands on the last day of shorter months"). Yearly Feb 29 → Feb 28 in non-leap years.
- Acceptance: (1) Each frequency advances dates correctly from a known start. (2) Jan 31 monthly lands on Feb 28/29. (3) Custom N=10 advances 10 days. (4) Stored frequency round-trips through edit without change.

### 326. Allow editing a template with an option to apply to future instances only.
- Components: Edit action per template row → reuses `mdlRecurringForm` prefilled; checkbox `chkFutureOnly` ("Apply to future instances only — keep already-confirmed history as-is"), default checked.
- Data: `qryUpdateTemplate` UPDATEs the template row. If future-only: only template fields change; existing confirmed/skipped instances and their transactions are untouched, and next_due_date is recomputed from the new frequency. If unchecked: additionally UPDATE pending (not confirmed) instances' expected amounts to match.
- Logic notes: Confirmed history is immutable under both options — the audit trail of what actually happened must not be rewritten. Changing the amount with future-only checked leaves past confirmations at their original amounts.
- Acceptance: (1) Editing with future-only checked leaves confirmed instances and their transactions unchanged. (2) Unchecked updates pending instances too. (3) next_due_date recomputes. (4) Audit log records the edit with before/after values.

### 327. Display the annual cost projection of each recurring template.
- Components: "Annual" column on the template table; tooltip on hover showing the math.
- Data: Transformer `trAnnualCost`: occurrences per year = 52 (weekly), 26 (biweekly), 12 (monthly), 4 (quarterly), 1 (yearly), 365/custom_interval_days (custom). Annual = amount × occurrences, prorated if end_date falls within the year. Spend shown negative, income positive.
- Logic notes: Projection only — labeled "projected from your template". Templates with an end date before year-end prorate by remaining occurrences. Paused templates show the projection grayed with "(paused)".
- Acceptance: (1) $10 weekly shows −$520/yr. (2) Custom 10-day shows −$365×amount correctly. (3) Ended-mid-year template prorates. (4) Value matches the tooltip's shown math.

### 328. Add a pause/resume control per recurring template.
- Components: Toggle or Pause/Resume button `btnPauseResume` on each template row; paused rows render dimmed with a "Paused" badge.
- Data: `qryTogglePause` sets status='paused'/'active'. Generator (`qryGenerateDue`) skips paused templates entirely. Audit log records the pause/resume.
- Logic notes: Pausing does not delete pending instances already generated — the user can still confirm or skip those. Resume recomputes next_due_date from today forward (missed dates during the pause do not backfill; user can backdate via "create single" if needed).
- Acceptance: (1) Paused template generates no new instances. (2) Existing pending instances remain actionable. (3) Resume recomputes next_due_date ≥ today. (4) Audit log has the pause/resume entry.

### 329. Show which recurring items have no confirmation in the last 60 days as stale.
- Components: "Stale" badge on template rows where last_confirmed_at is NULL or older than 60 days; a "Needs attention" filter chip above the table.
- Data: Transformer `trStaleness`: stale = status='active' AND (last_confirmed_at IS NULL OR last_confirmed_at < now() − 60 days). Templates with future next_due_date beyond 60 days are not flagged — staleness means instances were due and never handled.
- Logic notes: The refined rule: stale only if there EXISTS a pending or skipped instance older than 60 days, or a template whose next_due_date is >60 days past with no confirmation. Prevents false flags on quarterly/yearly templates.
- Acceptance: (1) Template with 90-day-old unconfirmed instance shows Stale. (2) Quarterly template with next due in 20 days does not. (3) Confirming clears the badge. (4) Filter chip isolates stale rows.

### 330. Support end dates on recurring templates (e.g., a 12-month subscription).
- Components: Optional End date picker `dtEndDate` in the template form; "Ends <date>" label on rows with an end date.
- Data: `end_date` column; validation end_date ≥ start_date. Generator stops creating instances with due_date > end_date. 90-day timeline (item 334) truncates at end_date. Annual projection (item 327) prorates.
- Logic notes: Editing a template to add an end date leaves already-generated future pending instances in place (user clears them via skip) but generates no new ones past the end. A template whose end_date passed shows an "Ended" badge and stops appearing in the due queue.
- Acceptance: (1) 12-month subscription template generates no instances past month 12. (2) end_date < start_date blocked. (3) Ended templates show "Ended" and leave the queue. (4) Prorated annual projection matches remaining occurrences.

### 331. Add a "confirm all due" bulk action for trusted templates.
- Components: "Confirm all due" button in the queue header, enabled only when at least one pending instance belongs to a trusted template; per-template `trusted` star toggle in the template form ("I trust this template — allow bulk confirm").
- Data: `qryConfirmAllDue`: loops pending instances where template.trusted=true, performing the same confirm steps as item 324 (insert transaction, mark confirmed, advance next_due_date), wrapped so each instance is individually audited. Non-trusted templates' instances stay in the queue.
- Logic notes: Bulk confirm never touches untrusted templates — the user must still review those one by one. A confirmation summary toast reports how many were posted. Undo is via normal transaction delete (audit-logged).
- Acceptance: (1) Only trusted templates' instances confirm. (2) Each posts exactly one transaction. (3) Summary toast shows the count. (4) Untrusted instances remain pending.

### 332. Display recurring spend as a share of monthly recorded income.
- Components: Summary tile `tileRecurringShare` on the Recurring screen: "Recurring: $X / $Y income (Z%)" with a small progress bar.
- Data: Transformer: monthly recurring spend = Σ |amount| of active spend templates × occurrences-per-month (weekly ×4.33, biweekly ×2.17, monthly ×1, quarterly ×0.33, yearly ×0.083, custom ×30.4/interval). Monthly recorded income = Σ positive `transactions` amounts this month (samples excluded when toggle hides them). Guard divide-by-zero → "no income recorded yet".
- Logic notes: Labeled "based on your entered transactions". Uses occurrences-per-month averages, not exact calendar math — documented in the tooltip.
- Acceptance: (1) Tile shows correct percent from sample data. (2) Zero-income month shows the fallback text, not NaN. (3) Hiding samples recomputes. (4) Paused templates excluded.

### 333. Tag confirmed instances with the template name for traceability.
- Components: Template-name tag shown on each confirmed instance row and on the resulting transaction row (small chip).
- Data: On confirm, the created `transactions` row gets description = template description (or template description + " — " + original suffix) and `recurring_instances.template_id` → `transaction_id` links back. A "view template" link on the transaction detail opens the template.
- Logic notes: Template renames apply to future confirmations only — historical transactions keep the name at confirm time (audit integrity). The instance row permanently records template_id + confirmed_amount even if the template is later deleted (FK set null on delete, name snapshotted into a `template_name_snapshot` column).
- Acceptance: (1) Confirmed transaction shows the template tag. (2) Clicking it opens the template. (3) Renaming the template leaves old transactions unchanged. (4) Deleting a template keeps instance history readable.

### 334. Show a timeline of upcoming recurring instances for the next 90 days.
- Components: Horizontal timeline strip `tmUpcoming90` on the Recurring screen: dots per instance grouped by week, color by category; click a dot → instance detail popover (template, due date, amount, account).
- Data: JS query `qryUpcoming90`: for each active template, project next_due_date forward by frequency while ≤ today+90 days and ≤ end_date, producing virtual instance entries (not DB rows). Exclude paused templates; sample templates excluded when the toggle hides them.
- Logic notes: Virtual — nothing is written until the generator creates real pending instances at due time. Timeline is a planning view only, labeled "projected — confirm each when due".
- Acceptance: (1) 90-day window shows all active templates' upcoming dates. (2) End dates truncate the projection. (3) Paused templates absent. (4) Clicking a dot opens the detail popover.

### 335. Allow converting any single transaction into a recurring template.
- Components: "Make recurring" action on each transaction row (overflow menu) → opens `mdlRecurringForm` prefilled with description, category, amount, account_id from the transaction.
- Data: Prefill only — the user picks frequency and start date (default start = the transaction's date). On save, normal `qryCreateTemplate`. The source transaction is untouched (no linkage created retroactively; future instances are new).
- Logic notes: Converting does not mark the original transaction as recurring-generated; history stays as-entered. Suggested frequency helper: if similar descriptions recur, the form hints "looks monthly" based on date gaps between same-description transactions (suggestion only).
- Acceptance: (1) Action prefills all four fields correctly. (2) Saving creates a working template. (3) Original transaction unchanged. (4) Cancel leaves no trace.

### 336. Add amount-change detection: flag when a confirmed amount differs from the template.
- Components: Warning icon on instance rows where confirmed_amount ≠ template amount at confirm time; template row shows "last confirmed: $X vs template $Y" note.
- Data: On confirm, the user may edit the amount in the queue row before approving; `confirmed_amount` stores what was actually posted. Transformer `trAmountDrift`: flag when |confirmed − template amount| > 0.01. Optional "update template to this amount" one-tap action.
- Logic notes: Common for variable bills (utilities). Drift does not block confirmation — it informs. The "update template" action is itself audit-logged as a template edit.
- Acceptance: (1) Confirming $85 against a $80 template flags the row. (2) Flag persists on the instance history. (3) "Update template" changes the template amount with audit entry. (4) Exact-match confirms show no flag.

### 337. Display recurring templates grouped by category.
- Components: Group-by toggle on the template table: Flat / By category; grouped view renders category headers (Housing, Food, Transport, Income, Fun, Bills, Other) with per-group subtotals (monthly equivalent + annual).
- Data: `qryRecurringTemplates` ordered by category then next_due_date; transformer groups rows client-side. Subtotal uses the same occurrences-per-month math as item 332.
- Logic notes: Categories with no templates are hidden (not shown empty). Income templates group under Income with positive subtotals. Sample grouping follows the sample toggle.
- Acceptance: (1) Grouped view shows all seven categories that have templates, with headers. (2) Subtotals match manual math. (3) Toggle back to flat preserves sort. (4) Empty categories hidden.

### 338. Support skipping a single instance without affecting the series.
- Components: Skip (×) button on each pending instance row; skipped instances move to a collapsible "Skipped" history section with an "Unskip → back to pending" action.
- Data: UPDATE `recurring_instances` status='skipped' (or back to 'pending' on unskip). Template's next_due_date advances past the skipped date — the series continues unaffected. No `transactions` row is ever written for skipped instances.
- Logic notes: Skipping is the correct action for one-off variations (vacation pause on a weekly subscription) vs pausing the whole template. Skipped instances count in the 60-day staleness check (item 329) so ignored items still surface.
- Acceptance: (1) Skip removes the instance from the queue without posting. (2) Next instance generates on schedule. (3) Unskip restores it to pending. (4) Audit log records skip/unskip.

### 339. Show total monthly recurring obligations as a headline number.
- Components: Hero stat `statMonthlyRecurring` at the top of the Recurring screen: "Recurring obligations: $X/mo" with a breakdown tooltip (spend vs income legs).
- Data: Transformer: Σ over active templates of amount × occurrences-per-month (same factors as item 332), split into spend leg (negative amounts) and income leg (positive). Headline shows the net; tooltip shows both legs and the template count.
- Logic notes: Excludes paused and ended templates; excludes sample templates when the toggle hides them. Labeled "projected from your templates — confirm each when due".
- Acceptance: (1) Headline equals manual sum of actives. (2) Pausing a template drops the number immediately. (3) Tooltip legs add to the net. (4) Sample toggle changes the total.

### 340. Log template creates, edits, pauses, and confirmations in the audit log.
- Components: Audit-log viewer (shared component from §38) gains a "Recurring" table filter; each entry shows actor, timestamp, action, template/instance id, before/after values.
- Data: `audit_log` writes on: template create (after_json = new row), template edit (before/after), pause/resume (status change), instance confirm (after includes created transaction_id), instance skip, bulk confirm (one entry per instance + a summary entry). `qryAudit` INSERT with actor='user'.
- Logic notes: Append-only — no edit/delete of audit entries. Bulk confirm writes per-instance entries so each posted transaction is individually traceable. Sample-template actions are logged with is_sample context.
- Acceptance: (1) Each of the five action types produces a log entry. (2) Entries show correct before/after. (3) Log is append-only (no edit UI). (4) Filter by "Recurring" isolates these entries.
