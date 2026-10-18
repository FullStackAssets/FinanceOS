# FinanceOS — Section 34: Savings rules & round-ups (manual)

Items 661–680 of the FinanceOS 1,000-item roadmap. This section covers user-defined manual savings rules (fixed transfers, round-ups, percentage-of-income) that generate confirmable records — money is never moved automatically; every rule instance is a manual bookkeeping record the user approves.

## Tables touched

- `accounts` (existing): id, name, type, institution, balance, updated_at, is_sample
- `transactions` (existing): id, account_id, date, description, category, amount, is_sample
- `goals` (existing): id, goal_name, target, saved, deadline

### New table: `savings_rules`

| Column | Type | Notes |
|---|---|---|
| id | pk | UUID |
| name | text | e.g. "Move $50 to vacation fund each payday" |
| rule_type | text | fixed / roundup / percent_income / goal_transfer |
| amount | number | for fixed rules |
| percent | number | for percent_income rules (0–100) |
| multiplier | number | round-up multiplier (default 1 = next dollar) |
| source_account_id | fk → accounts | account the rule "draws" from |
| target_goal_id | fk → goals, nullable | goal the rule feeds |
| frequency | text | payday / weekly / monthly / per_transaction |
| priority | number | "pay yourself first" ordering |
| is_paused | bool | default false |
| note | text | intent, e.g. "round-ups for the Japan trip" |
| is_sample | bool | sample rows flagged |

### New table: `rule_instances`

| Column | Type | Notes |
|---|---|---|
| id | pk | UUID |
| rule_id | fk → savings_rules | |
| due_date | date | when the instance is due |
| computed_amount | number | simulated amount for this instance |
| status | text | pending / confirmed / skipped / backdated |
| confirmed_at | timestamp, nullable | |
| linked_transaction_id | fk → transactions, nullable | contribution record created on confirm |
| is_sample | bool | |

## Shared cross-cutting rules

- Every screen shows the persistent banner: "Manual tracking — no bank connection yet."
- Sample rows carry `is_sample=true` plus a visible SAMPLE badge; they are excluded from all totals when the sample-data toggle hides them.
- Nothing ever presents as a live bank feed. All rule output is labeled as simulated/manual.
- Round-ups are MANUAL simulations: the spec models what a rule would transfer if the user did it themselves. Confirming a rule instance creates a contribution record, never an actual bank movement.

---

### 661. Define manual savings rules like "move $50 to vacation fund each payday."

**Components:** Rules List page: Table `savingsRulesTable` (columns: name, type pill, amount, frequency, paused badge), Button "New rule" opening Modal `ruleEditorModal` with TextInput name, Select rule_type (fixed / roundup / percent_income / goal_transfer), NumberInput amount or percent, Select source_account_id (from accounts), Select target_goal_id (from goals), Select frequency (payday / weekly / monthly / per_transaction), TextArea note. Save button with validation.

**Data:** Query `insertSavingsRule` (Retool DB insert into `savings_rules`), then `getSavingsRules` refresh. Transformer `ruleSummary` maps rule_type to human-readable label.

**Logic notes:** Fixed rules require amount > 0; percent rules require 1–100. Source account must exist; target goal optional for plain savings moves (creates a transaction only).

**Acceptance:** (1) Creating a fixed rule persists all fields and appears in the list. (2) Validation blocks amount ≤ 0 or percent > 100. (3) SAMPLE rules show the SAMPLE badge. (4) Saving refreshes the list without a page reload.

### 662. Show due rule instances in a confirmation queue — user approves each transfer record.

**Components:** "Confirm queue" Panel: ListView `ruleQueue` bound to pending rule_instances, each row showing rule name, computed_amount, due_date, target goal, and Buttons "Confirm" / "Skip". Counter `pendingCount` in the page header.

**Data:** Query `getPendingInstances` (status='pending' ordered by due_date). On Confirm: transaction `insertRuleContribution` writes a Transfer/Category transaction row + updates rule_instances.status='confirmed', confirmed_at=now, linked_transaction_id; also increments goals.saved via `updateGoalSaved`.

**Logic notes:** Confirm is idempotent — disable the button while the chain runs. Skipped instances are retained for history, never deleted.

**Acceptance:** (1) Pending instances list exactly those with status pending. (2) Confirming creates one contribution record and marks the instance confirmed. (3) Double-clicking Confirm does not create duplicate records. (4) Skipping hides the row and records status='skipped'.

### 663. Support round-up rules: round each spend to the next dollar, difference to a goal (recorded manually).

**Components:** Rule editor extension: when rule_type='roundup', show NumberInput `multiplier` (default 1) and target goal selector. Rules table shows computed "avg round-up per txn" column. Toggle `roundupActive` enables/disables the rule's per-transaction computation.

**Data:** Transformer `computeRoundups`: for each spend transaction (amount < 0) since the rule was created, roundup = ceil(|amount| × multiplier) − |amount| × multiplier; skips exact-dollar amounts (round-up 0).

**Logic notes:** Round-ups apply only to spends (negative amounts), never income. Dollar round-up = (ceil(|amount| × multiplier) − |amount| × multiplier) / multiplier. Only positive differences count; a $10.00 spend yields $0.00.

**Acceptance:** (1) A $4.25 spend with multiplier 1 yields $0.75. (2) Exact-dollar spends yield $0.00 and create no instance. (3) Multiplier 2 on $4.25 yields $0.25 ((ceil(8.5)−8.5)/2). (4) Income transactions are never rounded up.

### 664. Display a round-up preview per transaction before the user confirms it.

**Components:** Transactions list extension: inline "round-up" chip on each spend row showing the simulated amount (e.g. "+$0.75 → Japan trip"), with a "preview details" Popover showing the math (spend, rounded target, difference, target goal). Preview appears only when an active roundup rule exists.

**Data:** Transformer `roundupPreview` joins spend transactions with the active roundup rule (rule_type='roundup', is_paused=false) and computes the difference per row. No database writes.

**Logic notes:** Preview is purely computed — it must match the amount that item 662's confirm would create, so both use the same `computeRoundups` transformer.

**Acceptance:** (1) Each spend row shows a chip when a roundup rule is active. (2) Chip amount equals ceil(|amt|)−|amt| for multiplier 1. (3) Popover shows the arithmetic. (4) No chip appears when all roundup rules are paused.

### 665. Add percentage-of-income rules ("save 10% of each paycheck") as manual checklists.

**Components:** Rule editor: rule_type='percent_income' reveals NumberInput percent and a Select of income categories/tags to base it on. Confirmation queue rows for these rules render as a Checklist item: "10% of $2,400 paycheck = $240 → confirm". Progress ring per month showing confirmed vs expected.

**Data:** Transformer `percentInstances`: for each Income transaction in the month (category='Income'), instance amount = |amount| × percent/100. Query `getIncomeThisMonth` feeds it.

**Logic notes:** Base income excludes SAMPLE rows when the sample toggle hides them. Percent stored 0–100, displayed with one decimal.

**Acceptance:** (1) 10% on a $2,400 income yields a $240.00 instance. (2) Non-income transactions never generate instances. (3) Percent > 100 is rejected at save. (4) The monthly progress ring counts confirmed/total instances.

### 666. Show rule performance: total saved per rule per month.

**Components:** "Rule performance" Panel: BarChart `rulePerfChart` (x = month, series per rule, y = total saved) plus Table `rulePerfTable` (rule, month, confirmed count, total). MonthRange selector defaults to last 6 months.

**Data:** Query `getRulePerformance`: join rule_instances (confirmed) to savings_rules, group by rule_id and month(confirmed_at), sum computed_amount. Transformer pivots for the chart.

**Logic notes:** Only status='confirmed' rows count toward totals; skipped rows excluded. SAMPLE rules flagged but included unless hidden by the toggle.

**Acceptance:** (1) Chart bars equal the sum of confirmed instance amounts per rule per month. (2) Skipped instances do not appear in totals. (3) Changing the month range re-queries correctly. (4) A rule with zero confirmed instances shows $0, not a blank.

### 667. Allow pausing rules during tight months with one tap.

**Components:** Rules table row action: Button `pauseBtn` toggling to "Resume" (icon + label), with a paused Status pill on the row. Bulk action: "Pause all" / "Resume all" buttons with confirm modal. Paused rules dimmed (opacity 0.6).

**Data:** Query `toggleRulePaused` updates `savings_rules.is_paused`. Transformer `activeRules` filters is_paused=false for instance generation.

**Logic notes:** Pausing stops new instance generation immediately but preserves existing pending instances (user can still confirm or skip them). No delete.

**Acceptance:** (1) One tap flips is_paused and updates the row instantly. (2) Paused rules generate no new instances. (3) Existing pending instances remain confirmable. (4) "Pause all" requires a confirm and pauses every active rule.

### 668. Support "pay yourself first" ordering of rules by priority.

**Components:** Rules table: NumberInput or drag-handle column `priority` (1 = first). Confirmation queue sorted by priority ascending, then due_date. "Reorder" mode: up/down arrow buttons per row.

**Data:** Query `updateRulePriority` writes the priority. Transformer `sortedQueue` applies the sort. On rule creation, default priority = max(existing)+1.

**Logic notes:** Priorities are unique per user — on conflict, renumber siblings. Queue display groups by priority rank with a "Pay yourself first" header.

**Acceptance:** (1) Queue lists priority-1 rules before priority-2. (2) Changing priority re-sorts the queue live. (3) Duplicate priorities are auto-renumbered. (4) New rules append at the lowest priority by default.

### 669. Display a rules calendar showing upcoming rule instances.

**Components:** Calendar component `rulesCalendar` (month view) with event dots per instance, colored by status (pending amber, confirmed green, skipped gray). Clicking a day opens a Drawer listing that day's instances with confirm/skip buttons. Toggle "show confirmed" to declutter.

**Data:** Query `getInstancesInRange` (due_date between calendar start/end). Transformer maps rows to calendar events {date, title, color}.

**Logic notes:** Recurring rules pre-generate instances for the next 90 days (scheduled generation job or on-load backfill). Fixed/payday rules anchor to the user's payday setting.

**Acceptance:** (1) Every pending instance in the visible month renders a dot. (2) Clicking a day lists its instances with working confirm/skip. (3) Status colors match the legend. (4) Toggling "show confirmed" hides confirmed dots.

### 670. Add rule notes explaining the intent ("round-ups for the Japan trip").

**Components:** Rule editor TextArea `note` (280 chars, counter). Rules table: note shown as muted subtext under the rule name; full note in a Tooltip on hover. Confirmation queue rows display the note in italic under the amount.

**Data:** `note` column on `savings_rules`, carried through `getSavingsRules` and joined into queue queries.

**Logic notes:** Notes are optional; empty notes render nothing (no placeholder clutter). Notes are included in CSV export (item 678).

**Acceptance:** (1) A saved note displays under the rule name. (2) Editing the note persists and updates everywhere. (3) Empty notes show no placeholder text. (4) Note text appears in the confirmation queue row.

### 671. Never auto-move money: rules only create records the user confirms.

**Components:** Global "Safety" banner on the rules page: "Rules never move money — every instance needs your tap." Confirmation queue is the ONLY path to creating a contribution record; there is no "auto-confirm" setting anywhere in the app. Audit log view (item 680) proves every record has a confirmed_at timestamp tied to a manual action.

**Data:** No query auto-confirms: `insertRuleContribution` is only callable from the Confirm button handler. Code review checklist item in the spec: grep the app JSON for any scheduled/auto trigger on confirm logic — none allowed.

**Logic notes:** This is a design constraint, not just UI: the data layer must not contain any cron/trigger that flips status to confirmed. Edge case: backdated catch-ups (item 673) also require an explicit confirm tap.

**Acceptance:** (1) No setting, toggle, or schedule exists that auto-confirms instances. (2) Every confirmed instance has a non-null confirmed_at from a user click. (3) The safety banner is visible on the rules page. (4) Spec review finds no auto-confirm trigger in queries.

### 672. Show the cumulative effect of all rules on goal timelines.

**Components:** Goals detail Drawer extension: "With rules" projection line on the goal progress chart — a dashed LineChart series showing projected saved balance if all active rules' monthly averages continue. Text callout: "At this pace: funded by Mar 2027 (5 months early)."

**Data:** Transformer `ruleProjection`: per goal, monthly rule average = sum(confirmed amounts, last 3 months)/3; projection = current saved + avg × months until deadline. Query `getGoalRuleTotals`.

**Logic notes:** Paused rules excluded from the projection. Projection is labeled "projection," never as a guarantee. SAMPLE rules excluded when the sample toggle hides them.

**Acceptance:** (1) The dashed projection line starts at current saved and rises by the 3-month average. (2) Paused rules do not contribute. (3) The "funded by" date matches saved + avg×months ≥ target. (4) The chart is labeled as a projection.

### 673. Allow backdating a missed rule instance as a manual catch-up.

**Components:** Skipped/missed instance row action: Button "Catch up" opening Modal with DatePicker `catchupDate` (default today, cannot be future) and NumberInput amount (prefilled with computed_amount, editable). Confirm creates a rule_instance with status='backdated' and the contribution record dated to the chosen date.

**Data:** Query `insertBackdatedInstance` + same contribution chain as item 662 with the chosen date. Validation: date ≤ today.

**Logic notes:** Backdated records are flagged status='backdated' (not 'confirmed') so reports can distinguish. The linked transaction's date equals the chosen catch-up date, keeping month totals honest.

**Acceptance:** (1) Catch-up creates an instance with status='backdated'. (2) Future dates are rejected. (3) The contribution transaction carries the chosen date. (4) Reports distinguish backdated from confirmed rows.

### 674. Display per-goal rule contributions vs ad-hoc contributions.

**Components:** Goal detail Drawer: stacked BarChart `goalSourceChart` — per month, two stacks: "from rules" (teal) vs "ad-hoc" (slate). Legend + totals row: "$320 from rules · $150 ad-hoc".

**Data:** Query `getGoalContributions`: transactions tagged to the goal (via rule_instances.linked_transaction_id for rule-sourced; goal contribution records without a link are ad-hoc), grouped by month.

**Logic notes:** A contribution is "from rules" iff its transaction id appears as linked_transaction_id in rule_instances. Ad-hoc = manual goal contributions with no link.

**Acceptance:** (1) Stacks sum to the month's total goal contributions. (2) Every rule-confirmed contribution lands in the "from rules" stack. (3) Manual contributions with no rule link land in "ad-hoc". (4) Totals row matches the chart.

### 675. Add a "round-up all this week" bulk confirm action.

**Components:** Confirmation queue header: Button "Round-up all this week" (visible only when pending roundup instances exist for the current week), opening a confirm Modal listing each instance with its amount and a total. Confirm-all executes sequentially with a progress indicator.

**Data:** Query `getWeekRoundups` (rule_type='roundup', status='pending', due_date within this week). Loop in a JS query `bulkConfirmRoundups` calling the single-confirm chain per instance, collecting failures.

**Logic notes:** Atomicity per instance: each confirm is its own transaction; a failure mid-batch leaves earlier ones confirmed and reports which failed. Button disabled while running.

**Acceptance:** (1) The button appears only when ≥1 pending weekly roundup exists. (2) The modal lists each instance and the correct total. (3) After confirm, all listed instances are confirmed with linked records. (4) A mid-batch failure reports the failed item without duplicating the rest.

### 676. Show which account each rule draws from with a balance sanity check.

**Components:** Rules table column "Draws from": account name + current balance chip. Warning icon + Tooltip when computed_amount > account balance: "Rule amount exceeds available balance." Confirmation queue rows show the same check inline before the Confirm button.

**Data:** Transformer `balanceCheck`: joins rule_instances to accounts via savings_rules.source_account_id, compares computed_amount to accounts.balance.

**Logic notes:** Manual entry means balances may be stale — the warning is advisory, never blocking (user may have cash elsewhere). SAMPLE accounts flagged.

**Acceptance:** (1) Each rule row shows its source account and balance. (2) An instance exceeding the balance shows a warning. (3) Warning does not block confirming. (4) Balance reflects the latest manual update.

### 677. Support goal-to-goal transfer rules recorded manually.

**Components:** Rule editor: rule_type='goal_transfer' reveals two goal Selects (from_goal, to_goal) + amount/frequency. Confirmation queue rows read "Move $100: Emergency fund → Japan trip". Confirm decrements source goal saved and increments target goal saved, logging one transfer record.

**Data:** New columns on savings_rules: from_goal_id, to_goal_id (nullable fks → goals). Query `confirmGoalTransfer`: two `updateGoalSaved` calls + a transfer memo transaction (category 'Transfer', amount 0 net effect) for the audit trail.

**Logic notes:** Source and target must differ (validated). Net savings unchanged — this is reallocation, and the UI labels it "reallocate," not "save."

**Acceptance:** (1) Editor requires two different goals. (2) Confirm moves the amount between goals' saved totals. (3) A memo record appears in history. (4) Total across both goals is unchanged.

### 678. Export rule history to CSV.

**Components:** Rules page: Button "Export history" with Select `exportScope` (all / this month / per rule) and DateRange. Download triggers via Retool's file download utility. Export includes the manual-tracking banner text and export date in the filename.

**Data:** Query `exportRuleHistory`: joins rule_instances → savings_rules, ordered by due_date desc, columns: date, rule name, type, amount, status, target goal, note, is_sample. Transformer `toCSV` builds the CSV string.

**Logic notes:** SAMPLE rows included with an is_sample column so exports stay honest. Large histories paginated server-side; cap at 5,000 rows with a notice.

**Acceptance:** (1) CSV downloads with the expected columns and correct values. (2) Filename contains the export date. (3) is_sample column marks sample rows. (4) Scope filters (month/rule) change the row set.

### 679. Suggest rules based on surplus months ("you had $300 left in June — automate a rule?").

**Components:** Rules page "Suggestions" card: ListView `ruleSuggestions`, each card showing "June surplus: $300 — suggested rule: move $150/mo to Emergency fund" with Buttons "Create rule" / "Dismiss". Dismissed suggestions never reappear (stored dismissal).

**Data:** Transformer `surplusSuggestions`: income − spend − budget allocations per past month (from transactions + budgets); if surplus > $100 and no rule already captures it, suggest a fixed rule at 50% of surplus to the least-funded goal. Query `getDismissedSuggestions` filters.

**Logic notes:** Suggestions are advisory drafts — "Create rule" opens the rule editor prefilled, still requiring save. Never auto-create. Surplus computed on real rows only when samples hidden.

**Acceptance:** (1) A month with $300 surplus and no matching rule produces a suggestion card. (2) "Create rule" prefills the editor; nothing saves until the user taps Save. (3) Dismissed suggestions stay dismissed. (4) No suggestion appears when surplus ≤ $100.

### 680. Log rule creates, edits, pauses, and confirmations in the audit log.

**Components:** "Rule activity" tab on the rules page: Table `ruleAuditTable` (timestamp, actor "you", action, rule name, detail). Filter by action type. Export button reuses the CSV pattern.

**Data:** New table `rule_audit`: id, rule_id (fk), action (text: created / edited / paused / resumed / confirmed / skipped / backdated), detail (text), created_at (timestamp). Every mutating query in this section chains an `insertRuleAudit` call.

**Logic notes:** Audit writes are best-effort: if the audit insert fails, the primary action still completes but a console warning fires (never silently drop the user action). Audit rows are never editable or deletable from the UI.

**Acceptance:** (1) Creating a rule writes a 'created' audit row. (2) Pausing, confirming, and skipping each write their action. (3) The audit table is read-only in the UI. (4) Filtering by action type works.
