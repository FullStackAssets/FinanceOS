# FinanceOS Builder Spec — §06 Search, Filter & Sort (Items 101–120)

## Section overview
Full search, filter, and sort layer for the Transactions screen, plus cross-screen filter interactions (overview drill-down, budgets cross-filter). All filtering runs over the Retool DB `transactions` table via one parameterized query (`qryTransactionsFiltered`) driven by component state, so every filter combines with AND logic and updates the table, totals footer, and exports consistently.

## Tables touched
- `transactions` (existing): id, account_id (fk), date, description, category (Housing / Food / Transport / Income / Fun / Bills / Other), amount (negative=spend), is_sample. Note: items 117 (notes search) and 421+ (tags) assume a `notes` text column on `transactions` — add it here if not already present: `notes (text, nullable)`.
- NEW `saved_filters`: id (pk), filter_name (text), filter_json (text: serialized filter state), created_at (timestamp), is_sample (bool, default false). Seed with two sample rows ("Dining out this month", "All Housing 2026") marked is_sample=true.

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen, including the transactions screen above the filter bar.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data filter hides samples, they are excluded from the table, counts, totals, and exports.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.

---

### 101. Add a search box that matches transaction descriptions case-insensitively.
- Components: Text Input `txtSearch` above the transactions table, placeholder "Search descriptions…", prefix search icon, clear (×) suffix button.
- Data: `qryTransactionsFiltered` adds `WHERE LOWER(description) LIKE '%' || LOWER({{ txtSearch.value.trim() }}) || '%'` when search text is non-empty. Debounce input (200 ms) via the component's debounce setting.
- Logic notes: Trim whitespace; empty string = no filter. Match is substring, case-insensitive. Unicode-safe via LOWER on both sides.
- Acceptance: (1) Typing "coffee" matches "Coffee" and "COFFEE". (2) Table, count, and footer totals update live. (3) Clearing restores the unfiltered set.

### 102. Add a category filter dropdown covering Housing, Food, Transport, Income, Fun, Bills, Other.
- Components: Select `selCategory`, options exactly Housing / Food / Transport / Income / Fun / Bills / Other plus "All categories" (empty value default).
- Data: Appends `AND category = {{ selCategory.value }}` when a category is chosen.
- Logic notes: Single-select only. Category list is fixed taxonomy — no custom categories without a schema change. Default = all.
- Acceptance: (1) Selecting "Food" shows only Food rows. (2) "All categories" clears the constraint. (3) Combines with search/date filters via AND.

### 103. Add an account filter so the table can show a single account's activity.
- Components: Select `selAccount` populated from `qryAccounts` (label = name + institution, value = id), plus "All accounts" default.
- Data: Appends `AND account_id = {{ selAccount.value }}` when set.
- Logic notes: Hidden sample accounts still selectable when samples visible. When a single account is selected, item 88's running-balance column logic can reuse this state.
- Acceptance: (1) Choosing an account limits rows to that account. (2) Empty = all accounts. (3) Selection persists across other filter changes.

### 104. Add a date-range picker with presets: this month, last month, last 90 days, custom.
- Components: Segmented control `segDatePreset` (This month / Last month / Last 90 days / Custom) + Date Range `dtrCustom` shown only when Custom is selected.
- Data: Transformer `trDateRange` maps the preset to {start, end} (ISO dates); query uses `AND date BETWEEN {{ trDateRange.value.start }} AND {{ trDateRange.value.end }}`.
- Logic notes: "This month" = first day of current month through today. "Last 90 days" = today − 89 days through today. Custom validates start ≤ end; swap or error message otherwise.
- Acceptance: (1) Each preset produces the correct range. (2) Custom range filters exactly. (3) Invalid custom range shows an inline error, not an empty crash.

### 105. Add an amount-range filter with min and max fields.
- Components: Two Number Inputs `numMinAmt`, `numMaxAmt` in a collapsible "More filters" row; allow negatives (spend is negative).
- Data: Appends `AND amount >= {{ numMinAmt.value }}` / `AND amount <= {{ numMaxAmt.value }}` when each is non-null.
- Logic notes: Min > max shows inline validation "Min must be ≤ max". Blank = unbounded. Works on signed amounts (e.g., min −50, max −10 finds spends between $10 and $50).
- Acceptance: (1) Min/max bound rows correctly. (2) Blank fields impose no bound. (3) Min > max blocks the query with a clear message.

### 106. Add a spend-vs-income filter toggle.
- Components: Segmented control `segFlow` (All / Spend / Income).
- Data: Spend appends `AND amount < 0`; Income appends `AND amount > 0`; All appends nothing. Zero-amount rows are excluded from both (validation elsewhere prevents zeros).
- Logic notes: Sign convention is the single source of truth — negative = spend, positive = income.
- Acceptance: (1) "Spend" shows only negative amounts. (2) "Income" shows only positive. (3) "All" shows both.

### 107. Add a sample-data filter: show all, hide samples, show only samples.
- Components: Segmented control `segSample` (Show all / Hide samples / Samples only), default "Hide samples" per the app-wide default (item 39/346).
- Data: Hide samples → `AND is_sample = false`; Samples only → `AND is_sample = true`; Show all → no constraint. This state is shared app-wide via `appVars.sampleVisibility`.
- Logic notes: Default hides samples everywhere (totals, charts, exports). "Show all" renders SAMPLE badges on sample rows.
- Acceptance: (1) Default view excludes sample rows and their totals. (2) "Samples only" shows exactly the seeded rows with badges. (3) Switching updates table, count, and footer totals.

### 108. Combine all filters with AND logic and show active filter chips that can be cleared individually.
- Components: Container `cntActiveChips` rendering one Chip per active filter (search text, category, account, date preset, amount range, flow, sample) with an × to clear that filter only.
- Data: Single `qryTransactionsFiltered` builds all WHERE clauses with AND. Clearing a chip resets that control to its default and re-runs the query.
- Logic notes: Defaults (empty search, "All" selections, hide-samples) produce no chip. Chips update instantly on any control change.
- Acceptance: (1) Multiple filters narrow results conjunctively. (2) Each chip clears only its own filter. (3) No active filters = no chips shown.

### 109. Add sorting by date, amount, and description in ascending/descending order.
- Components: Select `selSort` with options: Date ↓ (default), Date ↑, Amount ↓, Amount ↑, Amount by |value| ↓, Description A–Z, Description Z–A.
- Data: `ORDER BY` clause mapped from the selection; tiebreak always `id DESC` for stability.
- Logic notes: Amount sorts on signed value (largest spend = most negative at the bottom of Amount ↑). Absolute-value option helps find biggest transactions regardless of sign.
- Acceptance: (1) Each option reorders rows correctly. (2) Order is stable across re-renders. (3) Sort composes with all active filters.

### 110. Persist the user's last filter set per session so navigation doesn't reset it.
- Components: No new UI; uses `localStorage` via `utils.setValue` / a page-load JS query `qryRestoreFilters`.
- Data: On any filter change, serialize {search, category, account, datePreset, customRange, minAmt, maxAmt, flow} to `localStorage.financeos_tx_filters`. On Transactions screen mount, restore values into components before first query run.
- Logic notes: Sample-visibility is app-global (separate var), not per-session-filter. Corrupt JSON in storage → fall back to defaults silently.
- Acceptance: (1) Leaving to Budgets and back restores the filter set. (2) Reloading the app mid-session restores it. (3) Corrupt stored JSON doesn't break the screen.

### 111. Add saved-filter presets like "Dining out this month" or "All Housing 2026."
- Components: Select `selSavedFilter` ("Saved views…") + Button `btnSaveView` ("Save current view") opening a small modal for the preset name.
- Data: Save writes the current serialized filter state to `saved_filters.filter_json`. Selecting a preset loads it into the filter controls and re-runs the query. Seed two sample presets (is_sample=true).
- Logic notes: Preset names must be unique — prompt to overwrite on collision. Deleting a preset requires confirmation. Sample presets are hidden when samples are hidden.
- Acceptance: (1) Saving then re-applying reproduces the exact filtered view. (2) Two seed presets work out of the box. (3) Duplicate-name save asks to overwrite.

### 112. Show the count of matching transactions next to the filter bar.
- Components: Text `txtMatchCount` right-aligned in the filter bar: "{{ qryTransactionsFiltered.data.length }} transactions".
- Data: Reads the filtered query's row count (query returns full filtered set; pagination is client-side per item 87).
- Logic notes: Count reflects all AND-combined filters. With 10k+ rows, count comes from a separate `SELECT COUNT(*)` with identical WHERE clauses to avoid loading all rows.
- Acceptance: (1) Count updates on every filter change. (2) Reads "1 transaction" / "N transactions" grammatically. (3) Matches the actual rows displayed.

### 113. Make filters collapsible on iPhone to preserve table space.
- Components: Button `btnToggleFilters` ("Filters" with chevron) wrapping the filter controls in collapsible Container `cntFilters`; collapsed by default under 480 px width, expanded on desktop.
- Data: Collapse state in `appVars.filtersCollapsed`; no query impact.
- Logic notes: Active filter chips (item 108) stay visible even when the filter panel is collapsed, so the user always sees what's applied. Chevron rotates on toggle.
- Acceptance: (1) At 390 px width the filter panel starts collapsed with table visible. (2) Chips remain visible while collapsed. (3) Desktop default is expanded.

### 114. Add a "clear all filters" one-tap button.
- Components: Button `btnClearFilters` ("Clear all", secondary style) in the filter bar, enabled only when at least one filter is active.
- Data: Resets all filter controls to defaults (empty search, "All" selections, current-month preset, blank amounts, All flow; sample visibility stays at the app-global setting) and re-runs the query; also clears the saved-filter selection.
- Logic notes: One tap, no confirmation — it's non-destructive (only resets view state). Disabled state prevents no-op clicks.
- Acceptance: (1) One tap restores the default filtered view. (2) Button disables when nothing is active. (3) Does not change the global sample-visibility toggle.

### 115. Highlight matched search text within descriptions.
- Components: Custom column rendering in the transactions table for the description cell.
- Data: JS mapper wraps the matched substring in a `<mark>` span: `description.replace(new RegExp(escapeRegExp(search), 'gi'), '<mark>$&</mark>')`, rendered as HTML in the column.
- Logic notes: Escape regex metacharacters in the search term. Escape HTML in the description first to prevent injection, then apply the mark. Only highlights when a search term is active.
- Acceptance: (1) Matched text shows a yellow highlight in every visible row. (2) Special characters (e.g., "C++") don't break rendering. (3) No highlight when the search box is empty.

### 116. Filter budgets screen categories by tapping a category on the overview.
- Components: On the Overview category donut/bars, set click event → `utils.setValue(appVars.budgetCategoryFocus, category)` then navigate to Budgets.
- Data: Budgets screen reads `appVars.budgetCategoryFocus` on mount and pre-filters/highlights that category row (scrolls to it); cleared when the user changes the filter or leaves the screen.
- Logic notes: Deep-link equivalent: `/budgets?category=Food` behavior via the same var for shared views (item 404). Null var = no focus.
- Acceptance: (1) Tapping "Food" on overview opens Budgets focused on Food. (2) Clearing the focus restores the full list. (3) Works from both the donut and the top-3 categories tile.

### 117. Add a notes/memo search that scans the transaction notes field.
- Components: Checkbox `chkSearchNotes` ("Include notes") beside the search box, default off.
- Data: When checked, the search clause becomes `AND (LOWER(description) LIKE … OR LOWER(notes) LIKE …)`. Requires the `notes` text column on `transactions` (see Tables touched).
- Logic notes: Notes may be NULL — use `COALESCE(notes, '')`. Match highlighting (item 115) applies to descriptions; add a notes indicator (item 94) so users know where the hit came from.
- Acceptance: (1) With the box checked, a memo-only term finds the row. (2) Unchecked, memo-only terms find nothing. (3) NULL notes never cause errors.

### 118. Support filtering by day of week to spot weekend spending patterns.
- Components: Multi-select Checkbox Group `chkWeekday` (Mon–Sun) in "More filters".
- Data: Appends `AND EXTRACT(DOW FROM date) IN ({{ selectedDays }})` when any day is chosen (Postgres DOW: 0=Sunday).
- Logic notes: Pairs with the weekday-vs-weekend chart (item 246) — a "Weekend only" preset selects Sat+Sun. Empty selection = no constraint.
- Acceptance: (1) Selecting Sat+Sun shows only weekend transactions. (2) Combines with category/date filters. (3) "Weekend only" one-tap preset works.

### 119. Add an "over budget categories only" quick filter on the transactions screen.
- Components: Toggle `tglOverBudgetOnly` ("Over-budget categories only") in "More filters".
- Data: Transformer `trOverBudgetCats` computes this month's per-category actuals vs `budgets.limit_amount` (same logic as item 161); filter appends `AND category IN ({{ overBudgetCategories }})`.
- Logic notes: Categories with no budget set are excluded from "over budget". Recomputes when budgets or transactions change (item 180). Empty result = friendly empty state ("No categories over budget 🎉").
- Acceptance: (1) Toggle shows only transactions in over-budget categories. (2) Updates live when a budget is edited. (3) Empty state renders when nothing is over budget.

### 120. Export the currently filtered transaction set to CSV.
- Components: Button `btnExportFiltered` ("Export CSV") in the filter bar.
- Data: JS query `qryExportFilteredCSV` re-runs the filtered SELECT (all rows, not just the visible page), builds CSV with columns date, description, category, account, amount, notes, is_sample, data-entry method, and triggers `utils.downloadFile`.
- Logic notes: Export honors every active filter including sample visibility; exports containing sample rows get the SAMPLE column value and the "Contains sample data — not real finances." banner row per items 351–352 and 486. Filename: `financeos-transactions-YYYYMMDD-HHmm.csv`.
- Acceptance: (1) Exported rows exactly match the filtered table. (2) `is_sample` column present; sample rows watermarked. (3) File opens cleanly in spreadsheet apps (UTF-8, quoted fields).
