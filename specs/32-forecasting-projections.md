# FinanceOS — Section 32: Forecasting & projections

Section: Forecasting & projections (roadmap items 621–640). All projections are extrapolations of Nic's manual entries, clearly labeled as estimates — never predictions of real accounts.

## Tables touched

Existing tables used (full app schema from specs.md App 11):
- accounts: id (pk), name (text), type (text: Checking / Savings / Credit / Investment), institution (text), balance (number), updated_at (timestamp), is_sample (bool)
- transactions: id (pk), account_id (fk), date (date), description (text), category (text: Housing / Food / Transport / Income / Fun / Bills / Other), amount (number, negative=spend), is_sample (bool)
- budgets: id (pk), category (text), month (text), limit_amount (number)
- goals: id (pk), goal_name (text), target (number), saved (number), deadline (date)

NEW table — forecasts (stores generated forecast runs so accuracy can be compared over time):
- forecasts: id (pk), forecast_type (text: month_end_spend / next_month_spend / goal_completion / cash_position_90d / annual_spend / big_purchase_impact / runway / debt_payoff / bills_total / net_worth_trajectory), created_at (timestamp), assumption_notes (text), assumptions_json (text, JSON blob), results_json (text, JSON blob: low/base/high values), is_sample (bool)

NEW table — forecast_assumptions (sandbox scenario inputs):
- forecast_assumptions: id (pk), name (text), income_change_monthly (number), new_bill_name (text), new_bill_amount (number), notes (text), created_at (timestamp)

Cross-cutting dependency: sections 17 (recurring transactions) and 15 (bills) supply the recurring templates and bill templates used by cash-position and bills forecasts; section 31 (AI spending insights) may share the low/base/high presentation pattern. Bills data source: assume a `bills` table (id, name, amount, due_day, is_sample) defined in section 15's spec.

## Shared cross-cutting rules

- Every screen shows the persistent banner "Manual tracking — no bank connection yet."
- Sample rows carry is_sample=true + a SAMPLE badge, and are excluded from totals and forecasts when the sample toggle hides them.
- Nothing ever presents as a live bank feed.
- Every forecast figure is an extrapolation of manual entries, labeled "estimate from your manual entries — not a guarantee."
- Forecasts recompute on data change; stored forecast runs (forecasts table) are point-in-time snapshots for accuracy review.

---

### 621. Project month-end spend per category from current daily run rate.

Components: Forecast screen section with a bar/table per category; Date range display; Toggle "include sample data" bound to the global sample flag; Text "projected" labels.

Data: `txThisMonth` query: transactions where date >= first-of-month (respecting is_sample filter). Transformer `runRate`: per category, spend-to-date / elapsed-days * days-in-month.

Logic notes: daily run rate = (sum of negative amounts) / days elapsed (min 1 to avoid divide-by-zero on day 1). Projected = runRate * daysInMonth. Handle empty categories as $0, not null. Days in month from JS Date.

Acceptance: (1) with $300 Food spend by day 10 of a 30-day month, projected Food = $900; (2) hiding samples removes sample transactions from the projection; (3) every figure carries the estimate label; (4) zero-transaction categories show $0 projected.

### 622. Forecast next month's total spend from the 3-month manual average.

Components: Stat cards (low/base/high or single figure); Method note "3-month average of manual entries"; "Save forecast" button writing to forecasts table.

Data: `txByMonth` query: last 3 full calendar months of transactions, grouped by month. Transformer `avg3`: mean of monthly spend totals.

Logic notes: only full months count — current partial month excluded. If fewer than 3 months of history exist, average over available months and flag "limited history." Spend = sum of negative amounts only (income excluded).

Acceptance: (1) with monthly spends 2000/2400/2200, forecast = 2200; (2) current partial month is never included; (3) 1-month history shows "limited history — 1 month" warning; (4) saving writes a forecasts row with assumption_notes "3-month manual average."

### 623. Project savings-goal completion dates from contribution velocity.

Components: Goals list with per-goal projected completion date; Progress bars showing on-track / behind pace; Velocity text "saving $X/mo → done ~Month Year."

Data: `goalContributions` — derive from transactions tagged to goals (assume section 12 contribution records: goal_contributions table with goal_id, amount, date) or from goal.saved delta over time. Transformer computes monthly velocity = total contributed / months elapsed.

Logic notes: projected date = (target − saved) / monthlyVelocity. If velocity is 0, show "no contributions yet — no date." If projected date > deadline, show red "behind pace" indicator.

Acceptance: (1) goal $5000 target, $2000 saved, $500/mo velocity → projected date 6 months out; (2) zero velocity shows "no contributions yet"; (3) behind-pace goals highlighted red; (4) sample goals excluded when samples hidden.

### 624. Forecast cash position 90 days out using recurring templates and bills.

Components: 90-day cash line chart (Retool Chart); Table of projected daily balance events; Assumption notes field.

Data: `recurringTemplates` (section 17), `bills` (section 15). Start from sum of liquid accounts (Checking + Savings) minus credit balances if selected.

Logic notes: iterate 90 days, apply each template occurrence and bill due date with its amount/sign. Starting cash = manual liquid balances. Warn prominently that stale balances (>30 days old) make this unreliable (links to item 637). Show min projected balance callout "lowest point: $X on date."

Acceptance: (1) a $100 weekly recurring outflow reduces projected cash by ~$1200 over 90 days; (2) monthly bills applied on their due_day each month; (3) estimate label present; (4) lowest-point callout matches the computed minimum day.

### 625. Show forecast ranges (low/base/high) instead of single-point predictions.

Components: Range bar or "low – high" stat per forecast type; Tooltip explaining how the range is derived; Method picker where relevant (e.g. ±10% band vs percentile).

Data: computed in transformer from base forecast: low = base * 0.85, high = base * 1.15 by default; where history exists, use 25th/75th percentile of past monthly totals.

Logic notes: ranges must never be presented as confidence intervals of real accounts — they are bands on manual-entry extrapolations. Round to whole dollars. High must be ≥ low; guard against negative bounds on spend (floor at 0).

Acceptance: (1) base $1000 shows $850 – $1150 default band; (2) with 6+ months history, range reflects actual 25th/75th percentiles; (3) spend ranges floored at $0; (4) label "range, not a prediction" visible.

### 626. Label every forecast "estimate from your manual entries — not a guarantee."

Components: Reusable Text component or shared module placed adjacent to every forecast figure; consistent styling (italic, muted).

Data: none — static copy. Optionally a shared Retool module `forecastDisclaimer` reused across screens.

Logic notes: disclaimer must appear within the same viewport region as each forecast figure (no hidden footers). Builder checklist item: every new forecast screen in this section includes the label before sign-off.

Acceptance: (1) every subsection 621–640 spec screen includes the exact label; (2) copy matches character-for-character; (3) label visible without scrolling on desktop and iPhone layouts; (4) CSV exports (item 638) include the disclaimer in assumption notes.

### 627. Compare past forecasts to actuals to show accuracy over time.

Components: Accuracy table (forecast date, forecast type, predicted, actual, % error); Line chart of error trend; "Clear saved forecasts" option.

Data: `savedForecasts` query from forecasts table; `actuals` computed from transactions for the same period. Transformer joins by forecast_type + period.

Logic notes: only compare forecasts whose target period has fully elapsed. % error = |actual − base| / actual. If actual is 0, show "n/a". Accuracy improves as a motivator: show "within 10% of actual" badges.

Acceptance: (1) a forecast of $2200 with actual $2000 shows 10% error; (2) forecasts for future periods excluded from accuracy table; (3) badges appear for ≤10% error; (4) clearing saved forecasts requires a confirm step.

### 628. Project annual spend per category from year-to-date manual data.

Components: Annual projection table by category; Year progress bar ("62% of year elapsed"); Toggle to annualize vs. use seasonal adjustment (links to item 633).

Data: `txYTD` query: transactions from Jan 1 to today. Transformer: per category, ytdSpend / daysElapsed * 365.

Logic notes: simple annualization assumes flat seasonality; show a note suggesting the seasonality view (item 633) for categories with spiky history. Exclude Income category from spend projection.

Acceptance: (1) $6000 Food spend over 200 days → ~$10,950 annual; (2) Income category excluded; (3) year-progress bar matches elapsed days / 365; (4) sample toggle respected.

### 629. Forecast the impact of a planned big purchase on monthly budgets.

Components: Form: purchase name, amount, category, month; Impact table showing affected budget(s) with before/after remaining; "what-if" bar chart.

Data: `budgetsForMonth` + `spendMTD` queries; user input only — nothing is written to transactions (what-if only).

Logic notes: after = budgetLimit − (spendMTD + purchaseAmount). If after < 0, show red "over budget by $X". Explicit "this is a what-if — nothing was recorded" banner. Discard/clear button resets inputs.

Acceptance: (1) $800 Food purchase against $500 remaining shows "over budget by $300"; (2) no transaction row is created; (3) what-if banner visible; (4) clearing the form restores the plain budget view.

### 630. Show a "runway" estimate: months of expenses covered by liquid accounts.

Components: Big stat "X.X months of runway"; Inputs: liquid accounts multi-select (default Checking + Savings); Monthly expense figure (auto from 3-month average, editable).

Data: `liquidTotal` = sum of selected account balances; `monthlyExpense` = 3-month manual average spend (item 622 logic) or manual override.

Logic notes: runway = liquidTotal / monthlyExpense. If monthlyExpense is 0, show "no spend data." Label as estimate; note that irregular annual bills are not smoothed — link to bills forecast (item 632).

Acceptance: (1) $6000 liquid / $2000 monthly → 3.0 months; (2) account selection changes the total immediately; (3) manual override of monthly expense recomputes; (4) estimate label present.

### 631. Project debt-payoff dates from manual payment entries.

Components: Debt list (Credit accounts from item 641) with projected payoff date each; Inputs: extra payment per month (optional); Avalanche vs snowball selector.

Data: `debts` (accounts type=Credit with balance, APR from section 33), `debtPayments` from manual transactions tagged to debt. Transformer amortizes: balance decreases by (payment − interest) monthly.

Logic notes: interest = balance * APR/12 monthly. If minimum payment ≤ monthly interest, show "never pays off at this rate" warning. Payoff date = months until balance ≤ 0. All manual-entry based — no lender data.

Acceptance: (1) $1000 at 24% APR paying $100/mo shows correct amortization schedule; (2) payment ≤ interest shows "never pays off" warning; (3) avalanche/snowball changes extra-payment allocation order; (4) estimate label present.

### 632. Forecast next month's bills total from bill templates.

Components: Bills list with next-month projected total; Per-bill row: name, expected amount, due date; "Amounts are from your templates" note.

Data: `bills` table (section 15): id, name, amount, due_day, is_sample. Project each bill once into next month.

Logic notes: bills with variable amounts (e.g. utilities) use the manual template amount; show "uses template amount" tag where no history exists. Exclude bills marked inactive if the section-15 spec supports it.

Acceptance: (1) 5 bills totaling $850 → next month forecast $850; (2) variable-amount bills tagged "template amount"; (3) sample bills excluded when samples hidden; (4) estimate label present.

### 633. Display seasonality: which months historically cost more, from manual history.

Components: 12-month heatmap or bar chart of historical monthly spend; "Most expensive months" callout list; Min-history warning.

Data: `txByMonthAll` — all history grouped by calendar month (Jan–Dec aggregates across years).

Logic notes: require at least 12 months of history spanning all 12 calendar months for a full view; otherwise show "partial history — X of 12 months." Rank months desc by average spend. Purely descriptive of manual entries — not a prediction.

Acceptance: (1) December averaging highest shows as top month; (2) partial history (<12 months) shows the warning; (3) chart uses average-per-month, not single-year data; (4) sample data excluded when hidden.

### 634. Allow adjusting forecast assumptions (income change, new bill) in a sandbox.

Components: Sandbox panel: income change (+/− $/mo), new bill (name + amount), one-time expense; "Apply to forecasts" recomputes; Reset button; forecast_assumptions save/load.

Data: reads forecast_assumptions table; sandbox inputs are temporary state variables until saved.

Logic notes: sandbox adjustments layer onto items 622/624/630 forecasts. Clearly badged "SANDBOX — not your real data." Saving a scenario stores it; loading restores inputs. Deleting a scenario requires confirm.

Acceptance: (1) +$500 income improves cash-position and runway figures; (2) sandbox badge visible on all affected figures; (3) reset restores base forecasts; (4) scenarios persist via forecast_assumptions table.

### 635. Show forecast confidence as high/medium/low based on data completeness.

Components: Confidence badge next to each forecast; Tooltip: "based on X months of data, last balance update Y days ago."

Data: computed from history length and balance freshness (accounts.updated_at).

Logic notes: high = ≥6 months history AND balances updated within 30 days; medium = 3–5 months or balances 30–90 days old; low = <3 months or balances >90 days old. Deterministic rule — document it in the tooltip.

Acceptance: (1) 8 months history + fresh balances → high; (2) 2 months history → low; (3) tooltip shows the exact inputs used; (4) badge updates when balances are refreshed.

### 636. Project net-worth trajectory 12 months out from snapshot trend.

Components: 12-month net-worth line chart with past (solid) + projected (dashed); Trend stats: monthly delta used.

Data: net-worth snapshots (section 36: net_worth_snapshots table with snapshot_date, net_worth). Linear regression or simple average monthly delta.

Logic notes: projected = last snapshot + avgMonthlyDelta * m. If fewer than 3 snapshots, show "need 3+ snapshots" (links to section 36). Dashed styling distinguishes projection from history. Estimate label mandatory.

Acceptance: (1) +$500/mo trend over 6 snapshots projects +$6000 at month 12; (2) <3 snapshots shows the requirement message; (3) projected line dashed, history solid; (4) sample snapshots excluded when hidden.

### 637. Warn when forecasts rely on stale balances older than 30 days.

Components: Warning banner on forecast screens: "Balances last updated X days ago — forecasts may be unreliable"; Per-account staleness list; "Update balances" link to Accounts screen.

Data: `accounts` updated_at; compute max age across accounts used in the forecast.

Logic notes: 30-day threshold, hard rule. Banner appears on every forecast screen that consumes account balances (624, 630, 636 at minimum). Dismissal is session-only — returns on reload until balances update.

Acceptance: (1) balance updated 45 days ago triggers the banner; (2) fresh balances (<30d) show no banner; (3) banner names the stale account(s); (4) dismissal does not persist across sessions.

### 638. Export forecast tables to CSV with assumption notes.

Components: "Export forecast CSV" button per forecast view; Download via Retool's download action; Filename includes date.

Data: current forecast results + assumption_notes from the forecasts table or the on-screen method text.

Logic notes: CSV first rows: disclaimer line (item 626), assumption notes, blank row, then headers + data. Amounts as plain numbers (no currency symbols) for spreadsheet use. Include generated-at timestamp.

Acceptance: (1) exported CSV opens in a spreadsheet with correct columns; (2) disclaimer and assumptions present in the file; (3) filename like `forecast-cash-position-2026-10-18.csv`; (4) sample-excluded state matches the on-screen toggle.

### 639. Exclude sample data from forecasts when samples are hidden.

Components: global sample toggle (shared with the rest of the app); every forecast query filters is_sample when the toggle is off.

Data: all forecast queries (621–638) add `AND is_sample = false` when samples hidden; stored forecasts rows carry their own is_sample flag.

Logic notes: this is a global invariant, not per-screen logic — implement as a shared query filter or a transformer wrapper applied to every forecast data source. When samples are shown, forecasts include them and every figure gets a SAMPLE badge.

Acceptance: (1) hiding samples changes a forecast figure that included sample transactions; (2) no forecast query bypasses the filter — spot-check all 20 subsections; (3) SAMPLE badge appears when samples are included; (4) saved forecast runs store whether samples were included.

### 640. Add a forecast-vs-actual review step in the month-end flow.

Components: Month-end flow step "Review forecasts" (section 37): table of last month's forecasts vs actuals; "Mark reviewed" checkbox; Notes field for learnings.

Data: forecasts table rows for the elapsed month + actuals from transactions (item 627 logic).

Logic notes: step is informational — marking reviewed stores a reviewed flag (assume month_end_reviews table from section 37, or a boolean on the forecast row). Skippable but tracked: unreviewed months show a "not reviewed" indicator.

Acceptance: (1) review step appears in the month-end flow after the target month ends; (2) forecast-vs-actual figures match item 627's accuracy logic; (3) marking reviewed persists; (4) skipping leaves a "not reviewed" indicator.
