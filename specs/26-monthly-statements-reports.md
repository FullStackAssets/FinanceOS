# Section 26 — Monthly statements & reports (items 501–520)

## Tables touched

**Existing:** accounts, transactions, budgets, goals (Retool Database, all with `is_sample` flags per App 11).

**NEW — monthly_statements:**
- id (pk), month (text, format YYYY-MM, unique), closed (bool, default false), closed_at (timestamp, nullable)
- total_income (number), total_spend (number), savings_rate (number, 0–1), budget_score (number, 0–100), net_worth (number)
- category_breakdown (text, JSON: {category: {actual, budget, variance}})
- summary_text (text), retro_note (text, nullable), goals_snapshot (text, JSON)
- is_sample (bool), created_at (timestamp), updated_at (timestamp)

**NEW — report_templates:**
- id (pk), name (text: e.g. "Landlord proof of rent", "Simple monthly summary"), template_body (text, placeholder fields), created_at (timestamp)

## Cross-cutting rules

- Every screen shows the persistent banner: "Manual tracking — no bank connection yet."
- Sample rows carry `is_sample=true` + SAMPLE badge and are excluded from totals when the sample toggle hides them.
- Nothing ever presents as a live bank feed.
- Reports are generated from manual entries only and every report header is labeled "Generated from manual entries" with the report date.

---

### 501. Build a month-review screen: income, spend, savings rate, budget score, top categories.

**Components:** MonthPicker (Select, month options from distinct transaction months), KPIStat hero strip: kpiIncome, kpiSpend, kpiSavingsRate, kpiBudgetScore (Statistic components), topCategoriesList (ListView of top 5 spend categories with bars), btnCloseMonth.

**Data:** qryMonthTx → `SELECT * FROM transactions WHERE to_char(date,'YYYY-MM')={{monthSel.value}}` (apply sample filter from global toggle); transformer calcMonth aggregates income (amount>0), spend (amount<0, abs), savingsRate, per-category totals; budgetScore = % of categories under budget.

**Logic notes:** Handle no-transaction months: show "No entries yet" empty state, savings rate N/A not 0/0 error. Amounts rounded to 2 decimals; spend displayed positive.

**Acceptance:** (1) Changing month re-renders all KPIs from its transactions. (2) Empty month shows empty state, no NaN. (3) SAMPLE badge appears when sample data included; hidden-sample mode excludes them from totals. (4) Banner visible at all times.

### 502. Compute savings rate as (income − spend) ÷ income from manual data.

**Components:** Reuses kpiSavingsRate from 501; tooltip explaining formula: "Savings rate = (income − spend) ÷ income, from your manual entries."

**Data:** JS transformer: `const inc=..., sp=...; savingsRate = inc>0 ? (inc-sp)/inc : null;`

**Logic notes:** Guard division-by-zero when income=0 → display "—". Negative rate possible when spend>income — show red. Round to 1 decimal %. This is purely client-side on the user's entered data; no assumed income.

**Acceptance:** (1) Known test data (income $4,000, spend $3,200) yields 20.0%. (2) Zero income shows "—", never Infinity/NaN. (3) Negative savings displays in red with minus sign. (4) Value matches manual calculator check on sample data.

### 503. Show a month-over-month comparison table for the last 6 months.

**Components:** moMTable (Table) with columns: Month, Income, Spend, Savings rate, Budget score, Δ vs prior month; monthSel drives the "current" anchor.

**Data:** qryLast6Tx → transactions from last 6 distinct months (relative to selected); transformer builds per-month aggregates + delta arrows.

**Logic notes:** Months with no data appear as grayed rows with "—". Anchor = selected month; table shows 5 prior. Sort descending by month.

**Acceptance:** (1) Table always shows 6 rows (current + 5 prior), placeholders for empty months. (2) Δ column shows correct signed change vs prior month. (3) Clicking a row switches the month-review screen to that month. (4) Sample-toggle exclusion applies to all rows.

### 504. Generate a written month summary in plain language ("You spent $X on Food, $Y over budget").

**Components:** summaryCard (HTML/Text component), btnCopySummary, btnRegenerate.

**Data:** JS transformer buildSummary(tx, budgets): composes sentences — top category spend, largest over-budget category ("$Y over budget"), savings-rate line, income line.

**Logic notes:** Deterministic template, no AI: e.g. "You spent $1,240 on Food, $140 over budget. Your savings rate was 18%." Skip categories with zero spend. Over-budget only reported when variance > 0.

**Acceptance:** (1) Summary names the actual top-spend category from the data. (2) Figures match the KPI strip exactly. (3) Copy button copies plain text. (4) Empty month yields "No entries this month." not gibberish.

### 505. Add a "close the month" action that snapshots budgets, actuals, and net worth.

**Components:** btnCloseMonth (Button, confirm via Modal), closeMonthModal showing snapshot preview.

**Data:** qryCloseMonth → INSERT INTO monthly_statements (month, closed=true, total_income, total_spend, savings_rate, budget_score, net_worth=sum(accounts.balance), category_breakdown=JSON, goals_snapshot=JSON of goals {goal_name,saved,target}); net worth uses accounts table at close time.

**Logic notes:** Idempotent guard: if a row for the month exists and closed=true, disable button ("Already closed"). Confirmation modal prevents accidents. Snapshot is a frozen copy — later edits to transactions don't change it.

**Acceptance:** (1) Closing writes exactly one monthly_statements row with matching totals. (2) Already-closed month disables the button. (3) Category breakdown JSON parses and matches the on-screen table. (4) Re-running close does not duplicate rows.

### 506. Display closed months as read-only with a "reopen" option logged in audit.

**Components:** closedBadge ("Closed — read-only"), btnReopen (opens confirm modal), monthReviewScreen in read-only mode (inputs disabled, banner note).

**Data:** qryReopen → UPDATE monthly_statements SET closed=false; INSERT INTO audit_log (action='month_reopened', month, at=now()); (audit_log table assumed from section 38 — reference by name; if not yet built, note as dependency.)

**Logic notes:** Reopen is the only mutation allowed on closed months; it flips closed flag only — the snapshot values stay frozen, so re-closing preserves original figures. Reopen requires confirmation.

**Acceptance:** (1) Closed month renders read-only: all edit controls disabled. (2) Reopen writes an audit row with timestamp. (3) Snapshot totals unchanged after reopen→reclose cycle. (4) Reopen button hidden from read-only view users (see section 39 roles).

### 507. Show per-category report cards with actual, budget, variance, and trend.

**Components:** catCardGrid (GridView of cards), each card: category name, actual, budget, variance pill (green under / red over), sparkline (trend from qryCatHistory), link to transactions.

**Data:** qryCatReport → per-category actuals for month + budgets join; qryCatHistory → same category last 6 months for sparkline data.

**Logic notes:** Categories with no budget show "No budget set" instead of variance. Variance = actual − budget. Trend sparkline uses prior 6 months' actuals; single data point → flat line.

**Acceptance:** (1) Card figures match transactions table for the month. (2) Over-budget shows red pill, under-budget green. (3) Clicking a card filters the transactions screen to that category/month. (4) No-budget categories show neutral state, not $0 variance.

### 508. Add a quarterly rollup view aggregating three monthly reports.

**Components:** quarterSel (Select: Q1–Q4 + year), rollupTable (Table: category × month columns + quarter total), quarterKPIs strip.

**Data:** qryQuarterTx → transactions WHERE month in quarter's 3 months; transformer aggregates income/spend/savings per month + quarter totals; uses closed snapshots when a month is closed (prefer snapshot over live calc).

**Logic notes:** Mixed state (2 closed + 1 open month): use snapshot for closed, live for open; flag rows with a "live" vs "closed" tag. Savings rate computed on quarter totals, not averaged.

**Acceptance:** (1) Quarter totals equal the sum of its three months. (2) Closed months pull from snapshot; open months from live data, each tagged. (3) Category breakdown matches per-month tables. (4) Empty quarter shows empty state.

### 509. Build an annual report: 12-month category table, net-worth curve, goal completions.

**Components:** yearSel, annualCatTable (12-month × category pivot), netWorthChart (LineChart from monthly snapshots), goalCompletions list (goals where saved≥target during year), btnExportAnnualPDF.

**Data:** qryAnnualTx (year of transactions), qryAnnualSnapshots (monthly_statements for the year), goals history (goals table; completion = saved≥target).

**Logic notes:** Pivot: rows=categories, columns=months, cells=spend. Net-worth curve uses snapshot net_worth for closed months, live sum for open. Goal completions dated by deadline or saved-reached (use deadline as proxy, labeled).

**Acceptance:** (1) Annual category totals equal the 12 monthly totals. (2) Net-worth curve has ≤12 points, labeled live vs closed. (3) Completed goals list matches goals table. (4) Year with partial data shows gaps honestly, not zeros.

### 510. Highlight best and worst budget months of the year.

**Components:** bestWorstCards: two highlight cards (trophy icon best, warning icon worst), yearSel shared with 509.

**Data:** JS: rank 12 months by budget_score (from monthly_statements or live calc); best=max score, worst=min score.

**Logic notes:** Tie-break: higher savings rate wins. Months with zero budgets excluded from ranking ("no budget set"). Only months with data considered.

**Acceptance:** (1) Best/worst match manual ranking of the 12 scores. (2) Cards show the month, score, and one-line reason (e.g. "5 of 6 budgets met"). (3) Clicking navigates to that month's review screen. (4) No-data year shows empty state.

### 511. Add a "what changed" diff between two selected months.

**Components:** monthA, monthB Selects, diffTable (Table: category, A actual, B actual, Δ, Δ%), summaryLine ("Food up $210, 24%").

**Data:** transformer diffMonths(txA, txB): per-category actuals, delta, % change; guard divide-by-zero when A=0 ("new spend").

**Logic notes:** Sort by absolute delta desc. Income shown separately. % change capped display at ±999% to avoid silly numbers. Same-month selection blocked with hint.

**Acceptance:** (1) Deltas equal B−A per category. (2) New categories (A=0) labeled "new", not ∞%. (3) Selecting same month twice shows a hint and empty table. (4) Figures match the two month-review screens.

### 512. Show income-vs-spend waterfall per month.

**Components:** waterfallChart (custom Plotly via HTML component or BarChart with running totals), monthSel shared.

**Data:** transformer: start at 0 → +income sources (by category, optional grouping) → −spend by top categories → end = net.

**Logic notes:** Keep ≤8 bars: top 5 spend categories + "Other". Positive bars green, negative red, net bar blue. Tooltips show exact amounts.

**Acceptance:** (1) Final bar equals income−spend for the month. (2) Bar order: income first, then spend desc. (3) Tooltips show exact values. (4) Empty month shows single "no data" bar, no crash.

### 513. Include sample-data exclusion note on every report when samples are hidden.

**Components:** sampleNoteBanner (Alert component) rendered at top of every report view when global sample toggle = hidden: "Sample entries are hidden — totals exclude $X sample spend and $Y sample income."

**Data:** transformer always computes both totals: withSamples and withoutSamples; the note reports the excluded amounts.

**Logic notes:** Note appears on month review, quarterly, annual, PDF export, and shared templates. When samples are shown, a subtler note: "Includes sample data." Excluded amounts computed from is_sample=true rows in range.

**Acceptance:** (1) Hidden mode: note visible on every report type with correct excluded figures. (2) Shown mode: note changes wording, no exclusion. (3) PDF export carries the same note. (4) Figures match a direct query on is_sample rows.

### 514. Allow annotating a monthly report with a retrospective note.

**Components:** retroNoteEditor (TextArea) + btnSaveNote on the month-review screen; note display card showing saved text + last-edited timestamp.

**Data:** qrySaveRetro → UPDATE monthly_statements SET retro_note={{editor.value}} WHERE month=...; creates a row if none (upsert pattern via Retool GUI upsert).

**Logic notes:** Editable even on closed months (annotation is metadata, not figures — allowed per 506). 2,000-char limit. Save is explicit (button), not autosave, to avoid accidental writes.

**Acceptance:** (1) Note persists and reloads on month revisit. (2) Works on closed months without altering snapshot figures. (3) Character limit enforced with counter. (4) SAMPLE badge logic unaffected.

### 515. Display goal progress achieved within each month.

**Components:** goalsThisMonth list (ListView): goal name, saved at month start→end, progress bar, "funded this month" amount.

**Data:** qryGoalContributions (goals table; if section 12's contributions table exists, use it for per-month deltas; else month-over-month saved diff labeled "estimated"). Monthly snapshot stores goals_snapshot JSON at close.

**Logic notes:** Prefer a contributions ledger if built (dependency note to section 12); fallback = diff of saved between month-start/end snapshot — label honestly "estimated from snapshots". Closed months read goals_snapshot.

**Acceptance:** (1) Per-goal "funded this month" sums to total contributions made that month. (2) Closed months show the frozen snapshot values. (3) Method labeled (ledger vs estimated). (4) Progress bars match goals screen.

### 516. Add a bills-paid-on-time rate per month.

**Components:** billsOnTimeStat (Statistic: %), billsTable showing each bill, due date, paid date, on-time pill.

**Data:** Bills data: if section 15's bills table exists, query bills WHERE month; paid_on_time = paid_date ≤ due_date. Rate = on-time ÷ total billed.

**Logic notes:** Dependency on section 15 bills/due-dates tables — note in spec. Bills with no paid_date count as unpaid (late), excluded from numerator. 100% when no bills? Show "—" with note "no bills this month" (avoid misleading 100%).

**Acceptance:** (1) Rate equals on-time ÷ total for the month. (2) No-bills month shows "—", not 100%. (3) Table rows show due vs paid dates. (4) Uses closed snapshot when month is closed.

### 517. Show largest single expenses per month in each report.

**Components:** topExpensesList (ListView, top 10) with description, date, category, amount; link to transaction detail.

**Data:** qryTopExpenses → SELECT * FROM transactions WHERE month AND amount<0 ORDER BY amount ASC LIMIT 10 (abs for display).

**Logic notes:** Income transactions excluded. Transfers between own accounts excluded if a transfer flag/category exists (note as optional refinement). Sample rows marked with SAMPLE badge inline.

**Acceptance:** (1) List sorted largest-first, ≤10 rows. (2) Amounts match transactions table exactly. (3) Clicking a row opens that transaction. (4) Sample exclusion toggle applies.

### 518. Support sharing a report as a PDF with the manual-tracking banner.

**Components:** btnExportPDF (uses section 25's PDF pipeline), reportPrintView (dedicated printable layout: banner, "Generated from manual entries" header, report date, KPIs, tables, sample note).

**Data:** Reuses section 25 export: utils.downloadFile of generated PDF; the report view is a hidden printable container rendered before export.

**Logic notes:** PDF header must include the manual-tracking banner text and generation date. Footer: "Figures entered manually — not a bank statement." Shared link = file download (no hosted public URL unless section 25 provides one).

**Acceptance:** (1) Exported PDF opens and is legible. (2) Banner text appears on page 1. (3) Figures match the on-screen report. (4) Sample-exclusion note included when samples hidden.

### 519. Add report templates: "landlord proof of rent", "simple monthly summary."

**Components:** templateSel (Select of report_templates), templatePreview, btnGenerateFromTemplate, templateEditor (admin: name + body with {{placeholders}}).

**Data:** report_templates table; renderer substitutes {{income}}, {{spend}}, {{rent_paid}}, {{month}} from month data. "Landlord proof of rent": lists rent-category transactions for the month with dates/amounts. "Simple monthly summary": income, spend, savings rate, top 3 categories.

**Logic notes:** Templates are presentation-only; no legal claims ("proof" = transaction list, labeled as manual records). Placeholder reference documented in the editor.

**Acceptance:** (1) Landlord template lists exactly the month's rent-category transactions. (2) Simple summary figures match the month-review screen. (3) Unknown placeholders render as blank, never crash. (4) Templates editable without code changes.

### 520. Archive closed-month reports with immutable snapshots.

**Components:** archiveList (Table of closed months: month, closed date, totals), btnViewSnapshot (read-only snapshot viewer), snapshotJSON viewer.

**Data:** monthly_statements rows WHERE closed=true; viewer renders category_breakdown + goals_snapshot JSON read-only.

**Logic notes:** Immutability: app never UPDATEs snapshot columns after close (enforced by convention + a Retool DB note; reopen only flips the closed flag). Archive retained indefinitely; no delete action exposed.

**Acceptance:** (1) Archived snapshot totals never change after later transaction edits. (2) Viewer is fully read-only (no inputs). (3) Archive lists every closed month, sortable. (4) Export of an archived report reproduces the original figures.
