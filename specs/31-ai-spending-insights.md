# FinanceOS — 31. AI spending insights (items 601–620)

**Section:** AI spending insights — pattern detection, plain-language summaries, and a scoped "ask about my money" chat, all computed from the user's own manual entries.

**Tables touched (base, from App 11):**
- accounts: id (pk), name (text), type (text: Checking / Savings / Credit / Investment), institution (text), balance (number), updated_at (timestamp), is_sample (bool)
- transactions: id (pk), account_id (fk), date (date), description (text), category (text: Housing / Food / Transport / Income / Fun / Bills / Other), amount (number, negative=spend), is_sample (bool)
- budgets: id (pk), category (text), month (text), limit_amount (number)
- goals: id (pk), goal_name (text), target (number), saved (number), deadline (date)

**New table:**
- insights: id (pk), insight_type (text: weekly_summary / unusual_spend / trim_suggestion / budget_miss / pace_compare / recurring_detect / leak_pattern / budget_suggestion / bill_rise / recap_draft / year_review / habit_impact), title (text), body (text), week_of (date), period_start (date), period_end (date), evidence_filter (text, JSON encoding the transaction filter used), source_note (text, default "Derived from manual entries — not bank analysis"), feedback (text, nullable: up / down), status (text: new / read / dismissed), created_at (timestamp), is_sample (bool)

**Shared cross-cutting rules:** every screen shows the persistent banner "Manual tracking — no bank connection yet."; sample rows carry is_sample=true + SAMPLE badge and are excluded from totals when the sample toggle hides them; nothing ever presents as a live bank feed. Insights are computed from the user's own manual entries; they are framed as patterns and observations, never financial advice. Every insight links to the transactions that produced it.

### 601. Generate a weekly plain-language spending summary from manual transactions.
Components: Insights inbox (ListView1) with "Weekly summary" card pinned; Text component `summaryBody` (markdown); Button `regenSummary`; banner component. Data: `qryWeekTxns` (transactions WHERE date >= week start AND is_sample=false unless toggle on); transformer `tWeeklySummary` aggregates total spend, top category, count, largest single transaction. Logic: narrative template in JS — total, top category share, largest item, one-week-over-week comparison. Runs on app load Sundays and on demand. Acceptance: 1) Body names the week's total spend and top category with figures matching transaction sums. 2) Mentions largest single transaction by description. 3) Shows "derived from manual entries" note. 4) Regenerate reflects newly added transactions.

### 602. Highlight the top 3 unusual spends of the week with comparisons to norms.
Components: Insight card `unusualSpendsCard`; Table `tblUnusual` (description, amount, norm, multiplier); Button `viewEvidence`. Data: `qryCatNorms` (90-day per-category median and stdev of spend amounts); transformer `tAnomalies` scores weekly transactions by z-score. Logic: flag amounts > median + 2×stdev or 3× category median; rank by score, take 3. Each stores evidence_filter JSON. Acceptance: 1) Card lists exactly 3 entries with category norm shown (e.g., "3.2× your usual Food spend"). 2) No flags when the week is within normal bands (empty state text). 3) Clicking an entry opens the filtered transaction list. 4) Sample transactions excluded when toggle hides samples.

### 603. Answer "where did my Fun money go?" with a description-level breakdown.
Components: Insight card `funBreakdownCard`; nested Table `tblFunDesc` grouped by description; total row. Data: `qryFunTxns` (category='Fun', current month, exclude samples per toggle); transformer groups by description, sums, sorts desc. Logic: show per-description spend, share of Fun total, count; top line + "other" bucket for items under 5%. Acceptance: 1) Breakdown sums equal the month's Fun category total. 2) Each row links to its underlying transactions. 3) "Other" bucket covers small items. 4) Description is empty-state friendly when no Fun spends exist.

### 604. Suggest which category to trim to hit a savings target, based on history.
Components: Insight card `trimSuggestCard`; NumberInput `targetSavings`; Table `tblTrimOptions` (category, typical spend, cut needed). Data: `qryCatMedians` (3-month per-category medians); transformer `tTrimPlan` computes which categories, reduced by what %, reach the target. Logic: prefer discretionary categories (Fun, Food); show two options: one-category cut and split cut. Never advise debt/income actions. Acceptance: 1) Suggestion names a concrete category and dollar amount (e.g., "Cut Fun by $120/mo to hit $500"). 2) Math uses 3-month medians, verifiable from data. 3) Shows what the reduced category budget would be. 4) Framed as pattern observation, not financial advice.

### 605. Explain budget misses in plain language ("Food ran over because of 6 dining-out entries").
Components: Insight card `budgetMissCard` per over-budget category; Text `missExplanation`. Data: `qryBudgetActual` (budgets vs summed transactions this month); transformer `tMissExplain` lists contributing transactions. Logic: compute overage; group offending transactions by description keyword; template sentence names count and top descriptions. Acceptance: 1) Explanation names the overage amount and the top contributing descriptions. 2) Links to the filtered transactions. 3) Only appears for categories actually over budget. 4) Counts are exact (matches query row counts).

### 606. Compare this month's pace to the user's best month on record.
Components: Insight card `paceCard`; Stat components: current pace, best month; Progress bar `paceVsBest`. Data: `qryMonthlySpend` (all months, total spend); transformer finds min-spend full month and projects current month (spend-to-date / days elapsed × days in month). Logic: pace% = projected / best month; note if current month incomplete. Acceptance: 1) Best month identified with month label and amount. 2) Projection formula visible in subtitle. 3) Current pace bar vs best-month line. 4) Months with no data are excluded from "best" selection.

### 607. Surface subscription-like recurring charges detected from manual patterns.
Components: Insight card `recurringDetectCard`; Table `tblRecurring` (description, amount, cadence, occurrences); Button `addToRecurring`. Data: transformer `tDetectRecurring` scans 6 months: same description ±2% amount repeating monthly/weekly. Logic: candidate needs ≥3 occurrences; confidence = regular intervals. Button opens recurring-transaction form prefilled. Acceptance: 1) Table lists detected candidates with cadence and occurrence count. 2) Irregular amounts don't match as recurring. 3) "Add to recurring" prefills the form. 4) User can dismiss a candidate permanently.

### 608. Draft a month-end recap paragraph the user can edit and save.
Components: Modal `recapModal`; TextArea `recapDraft` (editable); Button `saveRecap`; list of saved recaps. Data: `qryMonthStats` aggregate (spend, income, top categories, budget hits/misses, goal contributions); transformer `tRecapText` composes paragraph. Logic: template with figures injected; save writes to insights table as recap_draft. Acceptance: 1) Draft contains real figures matching month aggregates. 2) User can edit before saving. 3) Saved recap retrievable later. 4) Marked "derived from manual entries".

### 609. Suggest a realistic budget per category from 3-month medians.
Components: Insight card `budgetSuggestCard`; Table `tblBudgetSuggest` (category, suggested limit, current limit, delta); Button `applyBudgets`. Data: `qryCatMedians3mo`; transformer rounds medians to nearest $10, floor $25. Logic: apply writes budgets rows for next month (month text format YYYY-MM). Acceptance: 1) Suggestions are medians rounded to $10, verifiable against transaction history. 2) Apply creates/updates budget rows for the upcoming month. 3) Current limits shown alongside for comparison. 4) Empty categories get no suggestion.

### 610. Identify "leak" patterns: small frequent spends adding up (e.g., daily coffee).
Components: Insight card `leakCard`; Table `tblLeaks` (pattern name, avg amount, frequency, monthly total). Data: transformer `tLeaks` groups sub-$15 spends by description keyword, counts per week; flags ≥3/week sustained 4+ weeks. Logic: monthly total = avg × occurrences/30 days; sort by total desc. Acceptance: 1) Patterns list shows frequency and projected monthly total. 2) Only frequent small spends qualify (threshold documented). 3) Links to the underlying transactions. 4) One-off small purchases are not flagged.

### 611. Answer natural-language questions over the user's own data ("how much did I spend on Transport in March?").
Components: Chat box `askMoneyChat` (Messages component); answer panel with figure + evidence link. Data: transformer `tAnswerNL` parses category + month via regex/keyword map, runs parameterized SUM query. Logic: supported intents: category spend by month, total spend by month, top category, largest transaction. Out-of-scope → graceful refusal (see 619). Acceptance: 1) "Transport in March" returns the correct sum verified against a filtered transaction list. 2) Answer always shows its evidence link. 3) Unsupported questions refused gracefully. 4) Response time acceptable without external AI (all local computation).

### 612. Show the data behind every insight with a link to the filtered transactions.
Components: "View data" link on every insight card; drawer `evidenceDrawer` with pre-filtered transaction table. Data: insights.evidence_filter JSON parsed into query params. Logic: clicking sets transaction table filters (category, date range, description keyword); drawer shows row count matching insight's claim. Acceptance: 1) Every insight card has a working "View data" link. 2) Filtered table reproduces the insight's figures. 3) Drawer closes and clears filters. 4) Sample rows in evidence respect the sample toggle.

### 613. Mark all insights as derived from manual entries, not bank analysis.
Components: Footer note component `insightSourceNote` on every insight card: "Derived from manual entries — not bank analysis."; same in chat answers and exports. Data: insights.source_note default value. Logic: hardcoded copy, never conditional; included in recap exports. Acceptance: 1) The note appears on every insight card without exception. 2) It appears in chat answers and recap text. 3) Copy is exact. 4) No insight claims bank connectivity.

### 614. Allow thumbs up/down on insights to tune future ones.
Components: Icon buttons `thumbsUp` / `thumbsDown` on each card; toast confirmation. Data: UPDATE insights SET feedback='up'|'down' WHERE id. Logic: future generation weights: 'down'-rated insight types appear less often (skip if 2 consecutive downs); 'up' pins type. User can undo. Acceptance: 1) Feedback persists (survives reload). 2) Two consecutive 'down' votes suppress that insight type next cycle. 3) Undo resets feedback. 4) No feedback required — cards work without votes.

### 615. Generate a "year in review" narrative from 12 months of manual data.
Components: Button `yearReview`; Modal `yearReviewModal` with sectioned narrative (spending, best/worst months, categories, goals). Data: `qryYearStats` (12 monthly aggregates, category totals, goal progress). Logic: template sections with top-3 categories, biggest single month, goal milestones; requires ≥6 months of data else prompt to add more. Acceptance: 1) Narrative figures match 12-month aggregates. 2) Sections cover spending, months, categories, goals. 3) Graceful message when under 6 months of data. 4) Marked "derived from manual entries".

### 616. Suggest bill-negotiation candidates by flagging bills that rose year over year.
Components: Insight card `billRiseCard`; Table `tblBillRise` (bill description, last year avg, this year avg, increase $/%, months compared). Data: transformer `tBillYoY` matches descriptions year-over-year, compares monthly averages. Logic: flag rises ≥10% or ≥$10/mo; needs ≥6 months data each year. Acceptance: 1) Table shows before/after averages with dollar and % change. 2) Only recurring bill-like descriptions considered. 3) Insufficient history → empty state, not false flags. 4) Links to underlying transactions.

### 617. Estimate the impact of cutting a habit ("$5 coffee × 20 days = $100/mo").
Components: Insight card with Calculator: description keyword picker, avg cost (auto from data), days/month input; result panel `habitImpactResult`. Data: `qryHabitAvg` (avg amount for chosen keyword). Logic: impact = avg × days × 12 for yearly; show monthly and yearly figures; save as insight record. Acceptance: 1) Average auto-fills from the user's actual transaction data. 2) Math shown transparently (avg × days). 3) Monthly and yearly projections displayed. 4) Result can be saved to the insights inbox.

### 618. Provide a "ask about my money" chat box scoped strictly to the user's tables.
Components: Chat interface `askMoneyChat` (persistent, bottom-right or dedicated tab); suggestion chips for supported questions; history list. Data: reads only transactions, budgets, goals, accounts, insights tables. Logic: intent parser (local keyword/regex); every answer includes figure + evidence link; scope banner: "I can only answer from your manual entries." Acceptance: 1) Chat answers supported intents with correct figures. 2) It cannot reference any table beyond the five in scope. 3) History persists per session. 4) Scope banner always visible.

### 619. Refuse gracefully when asked about accounts or data not in the app.
Components: Refusal template in chat: "I only see your manual entries in FinanceOS — I can't look up [topic]." + suggestion chips for in-scope questions. Data: intent parser fallback branch. Logic: trigger on keywords (bank, stock price, credit score, external accounts) or unparseable intent; never hallucinate a figure. Acceptance: 1) Off-scope questions get the refusal template, never a fabricated answer. 2) Refusal suggests an in-scope alternative. 3) No error state or crash. 4) Logged as interaction (no PII stored).

### 620. Add an insights inbox where new weekly insights collect for review.
Components: Inbox screen/tab `insightsInbox`: ListView grouped by week, unread badge count, read/dismiss actions, filter by insight_type. Data: qry `SELECT * FROM insights ORDER BY created_at DESC`; weekly generation job inserts new rows status='new'. Logic: opening marks read; dismiss sets status='dismissed'; badge = count of status='new'. Acceptance: 1) New insights appear weekly with unread badge. 2) Read/dismiss actions update status and persist. 3) Filter by type works. 4) Empty inbox shows friendly empty state.
