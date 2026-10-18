# FinanceOS — Income Tracking (items 261–280)

Builder spec for the Income view. All figures are manual entries or labeled SAMPLE data.

## Tables touched

Existing: `transactions` (id, account_id, date, description, category, amount, is_sample), `accounts` (id, name, type, institution, balance, updated_at, is_sample), `budgets`.

New tables:
- `income_sources`: id (pk), source_label (text: paycheck/freelance/gifts/other), description (text), is_sample (bool)
- `expected_income`: id (pk), source_label (text), expected_amount (number), expected_date (date), frequency (text: one-time/weekly/biweekly/monthly), received (bool), received_at (timestamp nullable), note (text), is_sample (bool)
- `income_goals`: id (pk), source_label (text nullable), target_amount (number), month (text, YYYY-MM), note (text), is_sample (bool)
- `income_tax_tags`: id (pk), transaction_id (fk), withholding_amount (number), note (text), is_sample (bool)

## Cross-cutting rules

- The persistent banner "Manual tracking — no bank connection yet." renders on the income screen.
- Every row flagged `is_sample=true` shows a SAMPLE badge and is excluded from totals when the sample-data toggle hides samples.
- Nothing on the income screen presents figures as a live bank feed.

## Item specs

### 261. Build an income view listing all transactions in the Income category by month.

- **Components:** New tab "Income" in the app nav; MonthPicker (single month, defaults to current); Table `incomeTable` (columns: date, description, source_label, account, amount); summary Stat boxes above the table.
- **Data:** Query `qIncomeByMonth`: `SELECT t.*, a.name AS account_name FROM transactions t JOIN accounts a ON t.account_id = a.id WHERE t.category='Income' AND t.date >= {{monthStart}} AND t.date < {{monthEnd}} AND t.is_sample = {{sampleToggle ? true : false OR false}}` — order by date desc. Transformer `tIncomeTotal` sums amounts.
- **Logic notes:** Amounts for Income rows are positive; the add-transaction form's income toggle already enforces sign. When samples are hidden, exclude `is_sample=true` rows from both table and totals.
- **Acceptance:** (1) Table shows only Income-category rows for the selected month. (2) Changing the month re-queries within 1s. (3) Totals match the manual sum of visible rows.

### 262. Add income source labels (paycheck, freelance, gifts, other) as sub-tags.

- **Components:** Select `selIncomeSource` on the add/edit transaction form (visible only when category=Income), options from `income_sources`; Badge in `incomeTable` row showing source_label.
- **Data:** Column `source_label` (text, nullable) added to `transactions` (or a join table `transaction_income_tags` if schema change is restricted — spec uses a nullable column). Query `qIncomeSources` lists distinct labels. Default options seeded: paycheck, freelance, gifts, other; user can add custom labels via a small "manage" drawer.
- **Logic notes:** Source label is a sub-tag, not a category: category stays "Income" for budget math. Unknown/legacy income rows show "—" badge. Custom labels are plain text, max 40 chars.
- **Acceptance:** (1) Saving an Income transaction stores the selected source label. (2) Rows with no label render an em-dash badge, not blank space. (3) New custom labels appear in the dropdown immediately.

### 263. Show total recorded income per month with a 6-month average.

- **Components:** Two Stat components: `statIncomeMonth` ("This month: $X") and `statIncomeAvg` ("6-month average: $Y").
- **Data:** Query `qIncomeMonthly` returns per-month totals for the last 6 months: `SELECT date_trunc('month', date) AS m, SUM(amount) AS total FROM transactions WHERE category='Income' AND date >= {{sixMonthsAgo}} GROUP BY 1`. Transformer `tIncomeAvg` computes average of the 6 totals.
- **Logic notes:** Months with zero income rows count as 0 in the average (data gap is signal). Respect the sample toggle in both stats. Round display to whole dollars; keep cents in tooltip.
- **Acceptance:** (1) Stats update when the sample toggle changes. (2) Average equals the mean of the 6 monthly totals shown in the breakdown. (3) A month with no income rows displays $0, not a blank.

### 264. Display income by source as a bar chart.

- **Components:** BarChart `chIncomeBySource` (x = source_label, y = total), period selector (this month / last 6 months).
- **Data:** Query `qIncomeBySource`: `SELECT source_label, SUM(amount) AS total FROM transactions WHERE category='Income' AND date >= {{periodStart}} GROUP BY 1 ORDER BY total DESC`. Transformer normalizes null labels to "Unlabeled".
- **Logic notes:** Reuse the same category color palette as the app-wide charts (Income = the fixed Income color). Sort bars descending. Bar click drills to the filtered income table (item 261) pre-filtered by source.
- **Acceptance:** (1) Chart totals equal the income table total for the same period. (2) Clicking a bar filters the table to that source. (3) Unlabeled rows appear under one "Unlabeled" bar.

### 265. Add expected-income entries for upcoming paychecks, marked as planned not received.

- **Components:** Section "Expected" above the received-income table; Button "Add expected income" opening a modal (source_label, expected_amount, expected_date, frequency, note); "Mark received" per-row action.
- **Data:** New `expected_income` table (schema above). Query `qExpectedIncome` lists unreceived rows ordered by expected_date. On "Mark received," an `INSERT INTO transactions` creates the matching Income row (is_sample=false, source_label copied) and sets `received=true, received_at=now()`.
- **Logic notes:** Planned rows are visually distinct (dashed border, "PLANNED" pill) and are NEVER included in received-income totals. Frequency drives a "generate next" helper that pre-fills the following entry's date.
- **Acceptance:** (1) Planned entries do not change any received-income total. (2) "Mark received" creates exactly one transaction row and flips the planned row. (3) Past-due planned rows show an amber "overdue" chip.

### 266. Show a planned-vs-received income comparison per month.

- **Components:** Grouped BarChart `chPlannedVsReceived` (months on x-axis; two series: planned, received); month detail strip showing variance.
- **Data:** Query combines `qIncomeMonthly` (received) with `qExpectedMonthly`: `SELECT date_trunc('month', expected_date) AS m, SUM(expected_amount) AS planned FROM expected_income GROUP BY 1`. Transformer `tPlanVariance` computes received − planned per month.
- **Logic notes:** A planned entry marked received in a later month counts in the later month's received series (no double count). Months with neither show as zero bars, not gaps.
- **Acceptance:** (1) For a month where all planned income was received, bars are equal. (2) Variance strip shows "$X short" / "$X over" correctly signed. (3) Sample-toggle state applies to both series.

### 267. Support marking income as recurring with a frequency (weekly, biweekly, monthly).

- **Components:** On the Income form and expected-income modal: Checkbox "Recurring" + Select frequency (weekly/biweekly/monthly); on the Income tab, a "Recurring" filter chip and a small recurrence icon on tagged rows.
- **Data:** Columns `is_recurring` (bool, default false) and `frequency` (text nullable) on `transactions` and `expected_income`. For expected entries, a "Generate series" button inserts N upcoming planned rows (default next 6 occurrences) using date arithmetic per frequency.
- **Logic notes:** Recurrence is a manual template helper, not auto-posting: generated rows are planned entries the user still confirms (item 265 flow). Editing one generated row does not alter siblings. Biweekly = +14 days, monthly = same day-of-month (clamp to month end).
- **Acceptance:** (1) "Generate series" creates exactly 6 planned rows with correct dates. (2) Recurring rows show the recurrence icon; non-recurring do not. (3) Deleting one generated row leaves the others untouched.

### 268. Display net income after manually tagged tax withholdings.

- **Components:** Stat `statNetIncome` ("Net after tax: $X") beside gross total; per-row "tag withholding" action opening a small modal (withholding_amount, note).
- **Data:** New `income_tax_tags` table (schema above), one row per tagged transaction. Query `qIncomeWithTax`: `SELECT t.*, COALESCE(w.withholding_amount,0) AS tax FROM transactions t LEFT JOIN income_tax_tags w ON w.transaction_id=t.id WHERE t.category='Income' AND ...`. Transformer `tNetIncome` = SUM(amount − tax).
- **Logic notes:** Withholding is a manual tag, never computed from a tax table. Net = gross − tagged withholdings only; untagged rows contribute full amount. The tag is informational — it does not create a separate transaction.
- **Acceptance:** (1) Tagging $200 withholding on a $1,000 paycheck shows net $800. (2) Untagged income counts at full value in net. (3) Deleting the tag restores gross in the net figure.

### 269. Add an income notes field for details like "overtime" or "bonus."

- **Components:** TextArea `taIncomeNote` on the add/edit income form (placeholder: "overtime, bonus, …"); note icon indicator on income table rows that have one; tooltip/popover showing the note on hover or tap.
- **Data:** Column `note` (text, nullable) on `transactions` (shared with general transaction notes; income rows use the same field). Included in the row payload; truncated to 60 chars in the indicator tooltip, full text in the row drawer.
- **Logic notes:** Notes are freeform, max 500 chars. Notes are searchable via the global notes/memo search. Empty notes store NULL, not empty string, so the indicator logic is `note IS NOT NULL`.
- **Acceptance:** (1) Saving a note on an income row persists and re-renders after refresh. (2) Rows without notes show no indicator. (3) The note appears in CSV exports.

### 270. Show income deposited per account to track where money lands.

- **Components:** Breakdown list "By account" on the Income tab: rows of account name → month total; small stacked-bar showing each account's share.
- **Data:** Query `qIncomeByAccount`: `SELECT a.name, SUM(t.amount) AS total FROM transactions t JOIN accounts a ON a.id=t.account_id WHERE t.category='Income' AND t.date >= {{monthStart}} GROUP BY 1 ORDER BY total DESC`. Account picker on the form already determines the deposit account.
- **Logic notes:** Only non-archived accounts appear; accounts with zero income this month are omitted from the list (not shown as $0 rows). Totals respect the sample toggle. Currency shown is the account's native/home currency label.
- **Acceptance:** (1) Per-account totals sum to the month's income total. (2) Moving a transaction to another account updates both rows on refresh. (3) Archived accounts are excluded.

### 271. Add a "days of income left" estimate based on average daily spend.

- **Components:** Stat `statIncomeRunway` ("≈ N days of income left") with an info tooltip explaining the math; warning style when N < 14.
- **Data:** Inputs: current month's received income total (from `tIncomeTotal`), average daily spend = total spend (all non-Income categories, absolute value) over the last 30 days ÷ 30 via query `qAvgDailySpend`. Transformer `tRunwayDays` = floor(income_month_to_date ÷ avg_daily_spend).
- **Logic notes:** This is an estimate from manual data, labeled as such ("estimate from your manual entries"). If avg daily spend is 0, display "—" instead of dividing by zero. Spend uses the same sample-toggle filtering as income.
- **Acceptance:** (1) With $3,000 income and $100/day spend, shows 30 days. (2) Zero-spend history renders "—", not Infinity. (3) The caption under the stat reads "estimate from your manual entries."

### 272. Display irregular income months highlighted against the average.

- **Components:** In the monthly income table/breakdown (item 263 data), rows for months whose total deviates more than 25% from the 6-month average get an amber "irregular" chip; a legend explains the rule.
- **Data:** Reuse `qIncomeMonthly`; transformer `tIrregularFlags` computes mean and flags |total − mean| / mean > 0.25. Requires at least 3 months of data before flagging (avoids noise on new users).
- **Logic notes:** The 25% threshold is fixed and documented in the legend — not a hidden heuristic. Flag is display-only; it never alters totals. Sample-toggle state flows through.
- **Acceptance:** (1) A month at 2× the average is flagged; a month within 10% is not. (2) With fewer than 3 months of data, no flags render. (3) The legend text states the 25% rule.

### 273. Support splitting a paycheck into gross, taxes, and net line items.

- **Components:** On the income form, "Split paycheck" toggle revealing three amount inputs (gross, taxes, net) with a live check that gross − taxes = net; saved as linked line items.
- **Data:** Saved as up to 3 transaction rows sharing a `split_group_id` (UUID text column on `transactions`): gross row (Income, +gross), taxes row (category Bills or Other per user choice, −taxes), net row optional — OR simpler: one Income row for net plus a tax tag (item 268). Spec implements the linked-rows approach: rows carry `split_group_id`, and the income view groups them with an expand/collapse control showing "3 parts".
- **Logic notes:** Validation blocks save unless gross − taxes equals net within $0.01. Deleting one part prompts: delete the whole split or the single part. The taxes part does not count toward income totals (it is spend); the gross part does.
- **Acceptance:** (1) A $1,000/$200/$800 split creates 3 linked rows whose gross part totals $1,000 in income. (2) Math mismatch blocks save with a clear message. (3) Deleting the group removes all 3 rows and logs each deletion.

### 274. Show a year-to-date income total with monthly breakdown table.

- **Components:** Stat `statIncomeYTD` ("$X received in 2026"); Table `tblIncomeYTD` with 12 month rows (month, received total, planned total, variance); year selector.
- **Data:** Query `qIncomeYTD`: per-month received totals for the selected year from `transactions`; planned totals from `expected_income`. Transformer fills missing months with $0 rows so all 12 render.
- **Logic notes:** YTD = sum of completed months + current month to date; future months show $0 received and their planned values. Year boundary uses the transaction date, and the tax-year note (Jan–Dec) is shown once above the table.
- **Acceptance:** (1) The 12 monthly rows sum to the YTD stat. (2) Switching year re-queries. (3) Future months show planned (not received) figures.

### 275. Add income goal tracking (e.g., "$500 freelance this month").

- **Components:** "Income goals" card on the Income tab: list of goals (label, target, actual, progress bar); "Add goal" modal (source_label or "all income", target_amount, month); progress bars green at ≥100%.
- **Data:** New `income_goals` table (schema above). Query `qIncomeGoals` for the selected month; transformer `tGoalProgress` joins actual received totals per source_label (or all income when label is null) and computes actual ÷ target.
- **Logic notes:** Goals are manual targets, not predictions. "All income" goals use the month total; source-specific goals filter by source_label. Over-achievement shows "112%" — no cap at 100%.
- **Acceptance:** (1) A $500 freelance goal with $600 received shows 120% and a green bar. (2) Goals respect the sample toggle. (3) Deleting a goal asks for confirmation and logs the deletion.

### 276. Flag months where recorded spend exceeded recorded income.

- **Components:** On the monthly breakdown table (item 274), months with spend > income show a red "over" chip and the deficit amount; a banner on the Income tab when the current month is over.
- **Data:** Query `qMonthlySpend` mirrors `qIncomeMonthly` for non-Income categories (absolute values). Transformer `tDeficitFlags` computes spend − income per month; flag when > 0.
- **Logic notes:** Spend = absolute value of negative amounts in non-Income categories. Comparison uses the same sample-toggle filtering on both sides. The flag is informational; it does not block any action.
- **Acceptance:** (1) A month with $2,500 income and $2,800 spend is flagged with a $300 deficit. (2) A surplus month shows no flag. (3) Toggling sample data re-evaluates flags.

### 277. Exclude sample income from real income totals by default.

- **Components:** Global sample-data toggle chip (shared component, also on Overview); every income stat/table/chart binds its query's `is_sample` filter to this toggle; default state = hide samples.
- **Data:** All income queries (`qIncomeByMonth`, `qIncomeMonthly`, `qIncomeBySource`, `qIncomeByAccount`, YTD, goals) include `AND ({{hideSamples}} = false OR t.is_sample = false)`. Expected-income and income-goal queries apply the same predicate on their own `is_sample` columns.
- **Logic notes:** Default-on exclusion is the safety rule: real totals never silently include sample rows. A visible "samples hidden/shown" indicator sits next to the income totals so the user always knows which mode is active.
- **Acceptance:** (1) With samples hidden, a $500 SAMPLE paycheck does not appear in any income total. (2) Toggling to "show samples" adds it back everywhere consistently. (3) The indicator text always matches the toggle state.

### 278. Add an income CSV export with source tags preserved.

- **Components:** Button "Export income CSV" on the Income tab; export respects the current month/period filter and sample-toggle state.
- **Data:** Export query `qIncomeExport`: date, description, source_label, account_name, amount, note, is_sample, data_entry_method ("manual"/"imported"/"sample"). CSV generated via a JS transformer with UTF-8 BOM for spreadsheet compatibility. Filename: `financeos-income-YYYY-MM.csv`.
- **Logic notes:** Every export is stamped in the first rows with "Manual tracking — no bank connection" and the export date (per export rules). Sample rows included only when the toggle shows them, and carry is_sample=true in the file.
- **Acceptance:** (1) Exported row count matches the visible table row count. (2) source_label column is populated for tagged rows. (3) The file opens correctly in a spreadsheet app with UTF-8 encoding.

### 279. Show per-source reliability: how many of the last 6 months had that source.

- **Components:** "Source reliability" list: each source_label with "4/6 months" and a small 6-dot indicator (filled = month had income from that source).
- **Data:** Query `qSourceReliability`: `SELECT source_label, date_trunc('month', date) AS m FROM transactions WHERE category='Income' AND date >= {{sixMonthsAgo}} GROUP BY 1,2`; transformer `tReliability` counts distinct months per label out of the last 6 calendar months.
- **Logic notes:** A month counts if it has ≥1 income row with that label, regardless of amount. Unlabeled rows aggregate under "Unlabeled." Reliability is descriptive history, not a prediction.
- **Acceptance:** (1) A source present in 4 of the last 6 months shows "4/6" with 4 filled dots. (2) Sources with no recent activity show "0/6". (3) Sample-toggle state applies.

### 280. Log income edits and deletions in the audit log.

- **Components:** No new UI on the Income tab beyond existing edit/delete actions; entries appear in the Audit Log viewer (filterable by table=transactions / expected_income / income_goals).
- **Data:** New `audit_log` table (shared with the audit-log section): id (pk), actor (text), acted_at (timestamp), table_name (text), record_id (text), action (text: create/update/delete), before_json (text), after_json (text), entry_method (text: manual/import/sample). Every income write path (form save, mark-received, split save, goal CRUD, expected-income CRUD) inserts a row in the same Retool query success handler.
- **Logic notes:** Logs are append-only; the income screens never update or delete audit rows. before/after store the full row JSON so a deletion remains reviewable. Bulk operations (e.g., generate series) log one entry per created row with a shared batch id in after_json.
- **Acceptance:** (1) Editing an income amount creates an audit row with old and new values. (2) Deleting an income row creates an audit row retaining the deleted row's JSON. (3) The audit viewer can filter to income tables and shows entries in reverse chronological order.
