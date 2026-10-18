# FinanceOS — Rollover & budget rules (items 181–200)

Builder spec for the Retool FinanceOS app (App 11). Manual-entry personal finance; NO bank connection. This section adds month-to-month rollover, envelope budgeting, rule-based budgets, payday cycles, budget health, and audit logging of rule changes.

## Tables touched

- `budgets` (existing): id, category, month, limit_amount.
- `transactions` (existing): id, account_id, date, description, category, amount (negative=spend), is_sample.
- `goals` (existing): id, goal_name, target, saved, deadline.
- NEW `budget_rollovers`: id (pk), category (text), from_month (text, e.g. "2026-09"), to_month (text), rollover_amount (number, signed — negative when overspend is deducted), capped_amount (number, amount actually applied after cap), cap_applied (bool), created_at (timestamp), is_sample (bool).
- NEW `budget_rules`: id (pk), category (text), rule_type (text: percent_of_income / fixed_addon / cap), rule_value (number), base_category (text, e.g. "Income"), rollover_opt_in (bool, default false), rollover_cap (number, nullable), cycle (text: monthly / semi_monthly / biweekly, default monthly), paused_month (text, nullable), enabled (bool, default true), updated_at (timestamp), is_sample (bool).
- NEW `envelopes`: id (pk), category (text), month (text), funded_amount (number), spent_amount (number), swept_amount (number, default 0), source_note (text), is_sample (bool).
- NEW `audit_log` (dependency for item 200; full spec in section 38): id (pk), actor (text), action_at (timestamp), table_name (text), record_id (text), action (text), before_json (text), after_json (text), entry_method (text: manual/import/sample/AI-suggested).

## Cross-cutting rules

- Persistent banner "Manual tracking — no bank connection yet." on every screen, including all new rollover/envelope/rule views.
- All seeded/demo rows carry is_sample=true with a visible SAMPLE badge; the global "hide sample data" toggle excludes them from every total, chart, rollover computation, and health score in this section.
- Nothing in this section ever presents figures as a live bank feed; rollover and projection labels always read as manual-entry calculations.
- Rollover computations run only over non-sample transactions when the sample toggle hides samples.

### 181. Add an optional rollover rule per category carrying unused budget to next month.

- Components: On the budgets screen, per-category row add a Switch `swRollover_{category}` bound to `budget_rules.rollover_opt_in` for that category/month, plus a Number Input `numRolloverCap_{category}` (optional max) enabled only when the switch is on. A "Rollover settings" drawer holds the switch + cap + explanatory text.
- Data: Query `getBudgetRules` (SELECT from budget_rules WHERE category and month). On toggle, upsert into budget_rules (rollover_opt_in true/false). Transformer `rolloverCalc`: unused = limit_amount − actual_spend (actual from transactions this month, spend categories only); if unused > 0 and opt-in, propose rollover.
- Logic notes: Opt-in is per category, default off (see 183). Rollover is computed at month close or on demand via "Apply rollovers" — never automatically mid-month. Income category excluded.
- Acceptance: (1) Toggling the switch persists rollover_opt_in for that category. (2) With opt-in off, month close creates no rollover row. (3) With opt-in on and $40 unused, a budget_rollovers row for next month is created with rollover_amount=40.

### 182. Show rollover amounts as a separate segment on next month's budget bars.

- Components: On the budget-vs-actual bar component, add a stacked segment: base bar = limit_amount, overlaid segment = rollover from budget_rollovers for that category/month in a distinct color (e.g., hatched blue) with a Tooltip "Includes $X rolled over from September".
- Data: Query `getRolloverForMonth` (SELECT category, capped_amount FROM budget_rollovers WHERE to_month = current). Transformer merges onto budget rows: effective_limit = limit_amount + capped_amount.
- Logic notes: The segment is visually distinct from the base limit; legend explains the two segments. If rollover is negative (overspend deduction, item 190), the segment renders as a red reduction below the base.
- Acceptance: (1) A category with $200 limit + $40 rollover shows a bar totaling $240 with the $40 segment in the rollover color. (2) Tooltip names the source month and amount. (3) Categories without rollovers render unchanged.

### 183. Add a "use it or lose it" default with rollover as an explicit opt-in per category.

- Components: Budget rule drawer shows a Radio group `rgRolloverMode`: "Use it or lose it (default)" vs "Rollover unused to next month". The default selection is the first option for every category until the user changes it.
- Data: budget_rules.rollover_opt_in defaults to false at row creation; the upsert on first open writes the default. Query `getRolloverDefaults` confirms all categories default false on a fresh month.
- Logic notes: "Use it or lose it" means unused budget simply expires — no budget_rollovers row is written at close. Changing to rollover mode is explicit (user taps the radio), satisfying the opt-in requirement; the change is audit-logged (item 200).
- Acceptance: (1) A new category/month defaults to "use it or lose it" with no rollover row created at close. (2) Switching to rollover mode writes rollover_opt_in=true and is logged. (3) The UI labels the default clearly so users understand unused funds expire.

### 184. Display a rollover history log per category and month.

- Components: In the category detail view (or budget row expand), a Table `tblRolloverHistory` with columns: From month, To month, Rollover amount, Cap applied (yes/no), Created at. Filtered by the selected category.
- Data: Query `getRolloverHistory` (SELECT * FROM budget_rollovers WHERE category = selected ORDER BY from_month DESC). Respects the sample-data toggle via is_sample filter.
- Logic notes: Each row is immutable once created (no edits) — corrections are new rows with a note, preserving history. Negative rows (overspend deductions) show in red with a minus sign.
- Acceptance: (1) Selecting Food shows every rollover in/out for Food, newest first. (2) Amounts, months, and cap flags match the stored rows. (3) Sample-flagged rows hide when the sample toggle is off.

### 185. Cap rollover accumulation at a user-set maximum per category.

- Components: In the rollover settings drawer, Number Input `numRolloverCap` ("Max rollover carried, $") with placeholder "No cap". Display text under the budget bar when capped: "Rollover capped at $X (had $Y)".
- Data: Stored in budget_rules.rollover_cap. Rollover computation: capped_amount = MIN(unused, cap) when cap is not null; cap_applied = (unused > cap). Existing accumulated rollover is NOT re-capped retroactively — cap applies to each month's new rollover amount.
- Logic notes: Cap is per category, user-set, optional. A cap of 0 means no rollover accumulates (equivalent to opt-out but explicit). Changing the cap is audit-logged.
- Acceptance: (1) With cap $100 and $150 unused, capped_amount=100 and cap_applied=true. (2) With no cap set, full $150 rolls over. (3) The capped notice appears on the budget bar.

### 186. Add envelope-style budgeting view where each category is a funded envelope.

- Components: New "Envelopes" tab/screen: a Card grid `gridEnvelopes`, one card per spend category showing envelope name, funded amount, spent so far, remaining (funded − spent − swept), and a progress bar. A "Fund envelopes" button opens the funding drawer.
- Data: envelopes table (one row per category/month). Queries: `getEnvelopes` (current month), `fundEnvelope` (INSERT/UPDATE funded_amount), `spendFromEnvelope` triggered by transaction insert (increments spent_amount for the transaction's category).
- Logic notes: Envelopes are an alternative view over the same budget data — funding an envelope does not change budget limit_amount; it is a separate allocation the user manages. Remaining = funded − spent − swept; overspent envelopes show negative remaining in red.
- Acceptance: (1) Each spend category renders as an envelope card with funded/spent/remaining. (2) Funding $300 to Fun updates its card immediately. (3) Adding a $25 Fun transaction decreases that envelope's remaining by $25.

### 187. Support funding envelopes directly from recorded income transactions.

- Components: In the funding drawer, a "Fund from income" section: Select `selIncomeTxn` listing this month's Income-category transactions (description + amount), Number Input for allocation per envelope, and an "Allocate" button.
- Data: Query `getIncomeTxns` (transactions WHERE category='Income' AND month=current, non-sample per toggle). On allocate, UPDATE envelopes SET funded_amount = funded_amount + allocation, source_note = 'Funded from: <income description>'. The income transaction itself is unchanged (funding is an allocation record, not a transfer).
- Logic notes: Partial allocation allowed (allocate $400 of a $1,200 paycheck). Total allocated across envelopes is shown vs the income amount to avoid over-allocating; warn if allocations exceed the income transaction amount.
- Acceptance: (1) Selecting a paycheck and allocating $200 to Fun raises Fun's funded_amount by $200. (2) source_note records which income funded it. (3) A warning appears if total allocations exceed the selected income amount.

### 188. Show envelope balances decreasing as manual spend transactions are added.

- Components: Envelope cards show a live "Remaining" figure and a thin spend bar; a per-envelope mini transaction list (last 5 spends) under each card.
- Data: On transaction INSERT/UPDATE/DELETE for spend categories, a follow-up query adjusts envelopes.spent_amount for that category/month (recompute as SUM of spend transactions for the category/month rather than incrementing, to stay correct on edits/deletes). Transformer `envelopeRemaining` = funded − spent − swept.
- Logic notes: Recompute-from-source (not increment) keeps envelopes consistent when transactions are edited or deleted. Only manual transactions in the envelope's month count; future-dated "scheduled" transactions do not reduce the balance until their date arrives.
- Acceptance: (1) Adding a $30 Food transaction drops the Food envelope remaining by $30 within the same save. (2) Deleting that transaction restores the $30. (3) Editing the amount adjusts the remaining accordingly.

### 189. Add a "sweep leftover" action moving unspent envelope funds to a savings goal.

- Components: Per-envelope "Sweep leftover" button (visible when remaining > 0), opening a confirm modal: Select `selSweepGoal` (goals list), read-only amount = remaining, Confirm button.
- Data: On confirm: INSERT into goal contributions (goals.saved += amount — see section 12 schema), UPDATE envelopes SET swept_amount = swept_amount + amount, and write an audit_log row. The envelope's remaining drops to 0; funded/spent stay as history.
- Logic notes: Sweep is one-way per action but recorded, so it can be traced; a mistaken sweep is corrected by a manual goal withdrawal (section 12), not by editing the sweep. Sweep amount = remaining at confirm time.
- Acceptance: (1) Sweeping $60 from Fun to "Vacation" raises the goal's saved by $60 and zeroes Fun's remaining. (2) The sweep appears in the goal's contribution history with source "envelope sweep". (3) Cancel leaves everything unchanged.

### 190. Display negative rollover handling when a category overspends (deduct from next month).

- Components: On next month's budget bar, a red "overspend deduction" segment/flag: "−$X deducted: overspent in September". The category row shows adjusted effective limit = limit − deduction.
- Data: At month close, if actual > limit and rollover opt-in is on, write budget_rollovers with negative rollover_amount (e.g., −35). Effective limit next month = limit_amount + rollover_amount (negative reduces it).
- Logic notes: Negative rollover only applies when the category opted into rollover; "use it or lose it" categories simply expire overspend with no deduction (documented in the drawer). Deductions are shown explicitly, never silently subtracted.
- Acceptance: (1) Food overspent by $35 with rollover on → next month shows −$35 deduction on Food's bar. (2) With rollover off, no deduction is created. (3) The deduction row appears in the rollover history log in red.

### 191. Add budget rules like "Fun budget = 10% of recorded income."

- Components: In the budget rule drawer, a "Rule-based budget" section: Select `selRuleType` (Percent of income / Fixed add-on), Number Input `numRuleValue` (e.g., 10 for 10%), read-only preview text "Fun budget = 10% of recorded income → $X this month", and an Apply button. A badge (item 192) marks rule-based categories.
- Data: Stored in budget_rules (rule_type='percent_of_income', rule_value=10, base_category='Income'). Transformer `ruleBudgetCalc`: budget = SUM(Income transactions this month, non-sample per toggle) × rule_value / 100. The computed value feeds the budget bar as the effective limit.
- Logic notes: Rules compute from recorded (manual) income only — never assumed or projected income. If no income is recorded yet, the rule budget shows $0 with a note "no income recorded yet". Percent is of gross recorded income for the month.
- Acceptance: (1) Setting Fun = 10% of income with $3,000 recorded income yields a $300 Fun budget. (2) The preview text updates live as income is entered. (3) With $0 recorded income, the rule budget shows $0 and the explanatory note.

### 192. Show which categories use rules vs fixed limits with a badge.

- Components: On each budget row, a small Badge: "RULE" (blue) when budget_rules has an enabled rule for that category/month, "FIXED" (gray) otherwise. Tooltip on the badge shows the rule summary ("10% of recorded income").
- Data: Query `getBudgetRuleFlags` (LEFT JOIN budgets to budget_rules on category/month; flag = rule exists and enabled). Badge hidden text is also exposed for screen readers (see section 47).
- Logic notes: A category can have both a fixed limit_amount and a rule — the rule wins for display and the badge shows RULE with a note "rule overrides fixed $X". Disabling the rule reverts the badge to FIXED.
- Acceptance: (1) Fun with an active rule shows the RULE badge; Housing without one shows FIXED. (2) Tooltip states the rule in plain words. (3) Disabling the rule flips the badge back immediately.

### 193. Recompute rule-based budgets automatically when new income is entered.

- Components: No new UI — the budget bars and envelope funding previews update live. A subtle "updated" flash on the affected budget row when a recompute changes its value.
- Data: The add-transaction success handler triggers `ruleBudgetCalc` refresh for all rule-based categories when the saved transaction's category is Income. Also runs on income edit/delete. Debounced to avoid thrash during bulk imports.
- Logic notes: Recompute uses the same transformer as item 191 — single source of truth. Only categories with enabled rules recompute; fixed budgets never move. The recompute respects the sample toggle (sample income excluded when hidden).
- Acceptance: (1) Recording a $500 income entry raises a 10%-rule Fun budget by $50 without reload. (2) Deleting that income entry lowers it back. (3) Fixed-limit categories are unaffected by income changes.

### 194. Add a payday-based budget cycle option (e.g., 1st–15th, 16th–end).

- Components: In budget settings, a Select `selBudgetCycle`: "Calendar month" (default), "Semi-monthly (1st–15th, 16th–end)", "Biweekly (pick start date)" with a Date picker for the anchor. The choice applies per the scope in item 196.
- Data: Stored in budget_rules.cycle plus a settings key `payday_cycle_anchor`. Transformer `cycleRange(date)` returns {start, end} for the cycle containing a date; all "this month" actuals queries are rewritten to use the cycle range instead of calendar month.
- Logic notes: Semi-monthly splits at the 15th/16th regardless of month length. Biweekly uses the anchor date rolling forward 14 days. Changing the cycle mid-period takes effect immediately and is audit-logged; historical months keep the cycle they were closed under.
- Acceptance: (1) Selecting semi-monthly changes budget actuals to sum the 1st–15th only (during the first half). (2) Biweekly with anchor Oct 2 covers Oct 2–15. (3) The active cycle's date range is displayed (item 195).

### 195. Display budget cycle dates clearly on the budgets screen.

- Components: A header strip on the budgets screen: "Budget cycle: Oct 1 – Oct 15" (or "October 2026" for calendar month), updated by the cycle transformer. Each budget row's actuals tooltip repeats the cycle range.
- Data: `cycleRange(today)` transformer output rendered in a Text component `txtCycleRange`. No new table.
- Logic notes: The label always reflects the cycle currently in force, so users never wonder which days "actuals" cover. On cycle boundaries the label flips at local midnight.
- Acceptance: (1) The header shows the exact start/end dates of the active cycle. (2) Switching cycle type updates the header immediately. (3) Tooltips on actuals reference the same range.

### 196. Support different cycles per category for users paid biweekly.

- Components: In the per-category rule drawer, a Select `selCategoryCycle`: "Use default", "Calendar month", "Semi-monthly", "Biweekly" (+ anchor date input when biweekly chosen). A small note shows the category's effective range.
- Data: Stored per category in budget_rules.cycle (and anchor in a `cycle_anchor` column). Actuals per category are computed with that category's own range: query groups transactions by category, each filtered to its cycle window.
- Logic notes: Mixed cycles mean the budgets screen header shows the default cycle, while individual rows show their own range in a caption ("Oct 2 – Oct 15"). Rollover (items 181–190) uses each category's own cycle boundaries for close timing.
- Acceptance: (1) Fun on biweekly (anchor Oct 2) sums Oct 2–15 while Housing on calendar month sums Oct 1–31. (2) Each row displays its own cycle range. (3) Rollover close for Fun triggers at its cycle end, not month end.

### 197. Add a budget health score (0–100) based on categories under budget.

- Components: A "Budget health" card on the budgets screen and overview: big number 0–100, a colored ring (green ≥80, amber 50–79, red <50), and a one-line explainer ("6 of 8 categories under budget").
- Data: Transformer `healthScore`: score = 100 × (categories with actual ≤ limit) ÷ (categories with a budget set, excluding paused — item 199 — and Income). Only spend categories count; sample handling follows the global toggle.
- Logic notes: Equal weight per category by default (documented). Categories with no budget set are excluded, not scored zero. The score is a manual-data summary, labeled as such — never a credit-style score.
- Acceptance: (1) 6 of 8 categories under budget → score 75, amber ring. (2) Paused categories don't count for or against. (3) The explainer line names the numerator and denominator.

### 198. Show tips for categories repeatedly over budget ("Fun exceeded 3 months running").

- Components: Under each over-budget category row, an info callout `tip_{category}` with the streak message and a "View trend" link opening the category trend (section 13). A "Tips" summary card lists all active streaks.
- Data: Transformer `overBudgetStreaks`: for each category, count consecutive months (up to 12 back) where actual > limit, using that category's cycle. Streak ≥ 2 triggers a tip; wording escalates at 3+ ("3 months running — consider raising the limit or trimming spend").
- Logic notes: Streaks use closed months plus the current in-progress month. Paused months break the streak (not counted). Tips are informational only — no auto-adjustment of budgets.
- Acceptance: (1) Fun over budget in Aug, Sep, and Oct shows "Fun exceeded 3 months running". (2) A category over once shows no tip. (3) "View trend" opens that category's trend view.

### 199. Allow pausing a category's budget for a month (e.g., no Transport while traveling).

- Components: Per-category row menu → "Pause this month" (with confirm), and a "Resume" action on paused rows. Paused rows render grayed with a "PAUSED" badge and are excluded from totals, health score, and rollover.
- Data: budget_rules.paused_month = "2026-10" (current cycle). Queries filter out paused categories from actuals aggregation, health score denominator, and rollover close. Unpausing clears the field (audit-logged).
- Logic notes: Pausing is per month/cycle — it auto-expires when the cycle rolls (no permanent pause). Transactions in a paused category are still recorded and visible in the transactions list; only budget treatment is suspended.
- Acceptance: (1) Pausing Transport removes it from the health score and budget bars for October. (2) Transport transactions still appear in the transaction list. (3) November starts with Transport unpaused automatically.

### 200. Log all rollover and rule changes in the audit log.

- Components: No new UI in this section — entries appear in the audit log viewer (section 38). A "Recent rule changes" mini-list in the budget settings drawer shows the last 10 entries for transparency.
- Data: Every write to budget_rollovers, budget_rules, and envelopes writes an audit_log row: actor ("user"), action_at (now), table_name, record_id, action (create/update), before_json/after_json snapshots, entry_method='manual'. Rollover close writes one entry per category with the computed amounts.
- Logic notes: before/after JSON captures the full row state so changes are reconstructible. Bulk month-close logs each category as its own entry (no collapsed rows). Log writes happen in the same success handler as the data write so they can't be skipped.
- Acceptance: (1) Toggling rollover opt-in creates an audit entry with before=false/after=true. (2) Month close creates one entry per rolled category with amounts. (3) The settings mini-list shows the 10 most recent rule changes with timestamps.
