# FinanceOS Build Spec — 01 · Overview Dashboard (items 1–20)

Section of the FinanceOS 1000-item roadmap: "Overview dashboard".
Screens: the Overview screen (screen 1 of App 11), Retool app `FinanceOS` in the fullstackassets organization.

## Tables touched

- accounts: id (pk), name (text), type (text: Checking/Savings/Credit/Investment), institution (text), balance (number), updated_at (timestamp), is_sample (bool)
- transactions: id (pk), account_id (fk), date (date), description (text), category (text: Housing/Food/Transport/Income/Fun/Bills/Other), amount (number, negative=spend), is_sample (bool)
- budgets: id (pk), category (text), month (text YYYY-MM), limit_amount (number)
- goals: id (pk), goal_name (text), target (number), saved (number), deadline (date)
- net_worth_snapshots (NEW): id (pk), snapshot_date (date), total_value (number), per_account_json (text, JSON map of account id → balance), note (text), is_sample (bool), created_at (timestamp)
- bills (NEW): id (pk), bill_name (text), category (text: Bills/Housing/Other), amount (number), due_day (number 1–31), account_id (fk, nullable), is_active (bool), is_sample (bool)

## Cross-cutting rules (apply to every item below)

1. Every screen renders the persistent banner "Manual tracking — no bank connection yet." No screen may render a financial figure without the banner or a SAMPLE indicator present.
2. Any row with is_sample=true shows a SAMPLE badge/pill and is excluded from all totals when the global sample toggle is off (default: off).
3. Nothing may be presented as a live bank feed; labels read "manual entry", "entered", or "recorded" — never "synced" or "live".
4. Shared transformer `sampleFilter`: `{{ (rows) => showSamples.value ? rows : rows.filter(r => !r.is_sample) }}` applied to every query result before aggregation.

---

### 1. Add an overview screen showing net worth computed as the sum of all account balances, clearly marked as manual-entry data.

- Components: Text component `nwHero` (large number format), Text `nwLabel` ("Net worth — manual entries"), container `heroCard`.
- Data: query `qAccounts` (SELECT * FROM accounts ORDER BY sort_order); transformer `tNetWorth`: `sampleFilter(qAccounts.data).reduce((s,a)=>s+a.balance,0)`. Credit balances are stored as negatives (e.g. −1,240) so the sum is signed automatically.
- Logic notes: Net worth = Σ balance over included, non-sample accounts (when samples hidden). Show "as of" from item 10 beside it. Format with currency, 2 decimals on desktop, compact ($1.2k) when layout toggle is compact (item 19).
- Acceptance: (1) Hero number equals the sum of visible account balances. (2) Toggling samples off removes sample accounts from the sum. (3) Label "manual entries" is adjacent to the number.

### 2. Show a month-to-date spend vs budget progress bar per top-level category on the overview screen.

- Components: ListView `catBars` rendering one row per category; per row a ProgressBar `bar` and Text `pctLabel`.
- Data: queries `qTxnsThisMonth` (transactions WHERE date_trunc('month',date)=current month), `qBudgetsThisMonth` (budgets WHERE month=current YYYY-MM); transformer `tCatSpend`: group sample-filtered txns by category, spend = Σ(−amount) for amount<0, Income excluded from spend bars.
- Logic notes: Bar percent = spend ÷ limit, capped at 100% visual fill; overflow shown by red fill + "+$X over" text. Categories with no budget show "no budget set" state, not a zero bar.
- Acceptance: (1) Each category's bar matches its month-to-date spend ÷ budget. (2) Hiding samples recomputes actuals. (3) Bar turns red exactly when spend > limit.

### 3. Display each savings goal's progress bar with saved/target amounts on the overview screen.

- Components: ListView `goalBars` bound to goals; per row: Text `goalName`, ProgressBar `goalBar`, Text `goalAmounts` ("$1,240 of $5,000").
- Data: query `qGoals` (SELECT * FROM goals ORDER BY deadline NULLS LAST); transformer `tGoalPct`: `Math.min(100, Math.round(saved/target*100))`.
- Logic notes: Division-by-zero guard: target ≤ 0 → show "set a target" instead of a bar. Goals flagged is_sample show the SAMPLE badge and are excluded when samples hidden. Paused goals (see item 214) are excluded from this overview strip.
- Acceptance: (1) Bar width equals saved ÷ target. (2) Amounts render as "saved of target". (3) Zero-target goals do not divide by zero.

### 4. Render a persistent banner reading "Manual tracking — no bank connection yet." pinned to the top of the overview screen.

- Components: Container `bannerTop` with position sticky (top: 0, z-index high), Text `bannerText`, icon `infoIcon`. Background: amber tint (#fef3c7) with dark text for contrast.
- Data: none (static).
- Logic notes: Banner is the first rendered element in the DOM order on every main screen, not just Overview — implement as a shared module/fragment reused across screens. Dismissible-per-session option (item 342) must never remove it from the Overview on first load.
- Acceptance: (1) Exact text "Manual tracking — no bank connection yet." visible at top on load. (2) It stays pinned while scrolling. (3) Same component instance reused on all screens.

### 5. Add a sample-data toggle chip on the overview screen that hides or shows rows flagged is_sample.

- Components: Switch or segmented-control Chip `showSamples` (default OFF), placed in the overview header next to the banner.
- Data: none directly; the switch drives the global temp-state `showSamples.value`, consumed by `sampleFilter` everywhere.
- Logic notes: Toggle triggers re-run of dependent queries (qAccounts, qTxnsThisMonth, qGoals, etc.) via success handlers or dependency chain. When ON, a secondary "Sample data visible" pill appears next to the banner so mixed figures are always labeled.
- Acceptance: (1) Default OFF on first load. (2) Turning ON includes is_sample rows in every overview total. (3) A visible indicator shows when samples are included.

### 6. Show account balance cards on the overview with institution name, account type, and updated_at timestamp.

- Components: GridView/ListView `acctCards`; per card: Text `acctName`, Tag `acctType` (Checking/Savings/Credit/Investment with icon), Text `acctInstitution`, Text `acctBalance`, Text `acctUpdated` ("updated 2h ago"), SAMPLE badge when applicable.
- Data: `qAccounts`; transformer `tAcctCards` adds `ageLabel` from updated_at.
- Logic notes: Sort order follows the user's drag order from the Accounts screen (item 48); default: Checking, Savings, Credit, Investment then name. Credit cards show balance in red with a leading minus.
- Acceptance: (1) Each card shows name, institution, type tag, balance, updated-at. (2) SAMPLE badge appears exactly when is_sample=true. (3) Card order matches the Accounts screen order.

### 7. Add a quick-add transaction button on the overview screen that opens the add-transaction form prefilled with today's date.

- Components: Button `btnQuickAdd` ("+ Add transaction", primary style, 44px+ touch target); Modal `modalTxn` containing the shared add-transaction form component.
- Data: on click, set temp-state `txnFormDefaults` = { date: today (local), account: last-used account, category: null }; the form reads these as initial values.
- Logic notes: Modal uses the same form definition as the Transactions screen (single source of truth, item 61). After save, the modal shows the confirmation toast (item 73) and refreshes overview queries without full page reload.
- Acceptance: (1) Date field is prefilled with today's local date. (2) Saving refreshes overview totals. (3) Form is the same component used on the Transactions screen.

### 8. Show the three largest spending categories this month with amounts on the overview screen.

- Components: Container `topCats` with three stat tiles (category icon + name + amount), header "Top spending this month".
- Data: reuse `tCatSpend` from item 2; transformer `tTop3`: sort spend desc, take 3.
- Logic notes: Ties broken alphabetically for determinism. If fewer than 3 categories have spend, show only those (no empty placeholders). Income category is excluded. Respects sample toggle.
- Acceptance: (1) Exactly the top-3 spend categories by amount render. (2) Amounts equal month-to-date spend per category. (3) Empty months show a "no spending recorded yet" state.

### 9. Display a cash-flow summary tile for the current month: total income minus total spend from manual transactions.

- Components: Stat tile `cashFlowTile`: big number (net), sub-lines "Income $X" / "Spent $Y".
- Data: transformer `tCashFlow`: income = Σ amount where amount>0, spend = Σ(−amount) where amount<0, both sample-filtered for current month.
- Logic notes: Net = income − spend; negative net shows in red with "spent more than earned" caption (ties to item 276's monthly flag). All figures labeled manual.
- Acceptance: (1) Net equals income minus spend from this month's manual transactions. (2) Sample toggle changes the result. (3) Negative net is visually distinct.

### 10. Add an "as of" timestamp on the overview showing when balances were last manually updated.

- Components: Text `asOf` under the net-worth hero: "Balances as of Oct 18, 2026 9:41 AM — entered manually".
- Data: transformer `tAsOf`: MAX(updated_at) over sample-filtered accounts.
- Logic notes: If any account is stale (>30 days, item 33), append "· 2 accounts stale". Timezone: user's locale (settings). This timestamp is about balance recency, not data sync.
- Acceptance: (1) Timestamp equals the latest account updated_at. (2) It updates immediately after any balance edit. (3) Stale-account count appears when applicable.

### 11. Render a 6-month spend trend sparkline on the overview using only manually entered transactions.

- Components: Chart `spendSpark` (line, height ~64px, no axes labels except min/max), caption "manual entries only".
- Data: query `qTxns6mo` (last 6 full months + current MTD); transformer `tSpark`: bucket spend by month, return [{month:'Apr', spend:1234}].
- Logic notes: Spend only (amount<0 → −amount). Months with no transactions render as 0 (flat point, not a gap). Sample-filtered. Tooltip shows "May: $1,234 (42 transactions)".
- Acceptance: (1) Six points, one per month, spend-only. (2) Zero months render as 0. (3) Caption "manual entries only" is present.

### 12. Show upcoming bills due in the next 7 days as a list tile on the overview screen.

- Components: Container `billsTile` with ListView `upcomingBills`: rows show bill name, due date chip ("in 3 days" / "tomorrow"), amount, account chip.
- Data: query `qBills` (bills WHERE is_active); transformer `tUpcomingBills`: compute next occurrence date from due_day within the next 7 days; exclude bills already marked paid this month (paid flags live in transactions via matching bill tag or a bill_payments table — see section 15 spec).
- Logic notes: Bills with variable amounts show the estimated default with "est." suffix. Overdue bills (due date passed, unpaid) appear in red at the top of this tile.
- Acceptance: (1) Only bills due within 7 days and unpaid appear. (2) Due chips show correct day counts. (3) Overdue bills are red and first.

### 13. Add a net-worth delta tile comparing this month's manual snapshot to last month's.

- Components: Stat tile `nwDelta`: signed amount + up/down arrow + "% change", caption "vs last month's snapshot".
- Data: query `qSnapshots` (net_worth_snapshots ORDER BY snapshot_date DESC LIMIT 2); transformer `tDelta`: current − previous; pct = delta ÷ |previous|.
- Logic notes: If no previous snapshot exists, show "Take your first snapshot" with a button that runs item 22's action. Delta is computed from snapshots, never from live balances, so the tile is stable between snapshots. Manual-entry banner applies (snapshots table labeled manual).
- Acceptance: (1) Delta equals current snapshot minus previous. (2) Up/down indicator matches the sign. (3) Missing previous snapshot shows the onboarding state, not $0.

### 14. Display a budget-health ring on the overview summarizing categories under/over budget this month.

- Components: Chart `healthRing` (donut): segments = under budget (green), near limit 80–100% (amber), over (red), no budget (gray). Center label "5 of 7 on track".
- Data: reuse item 2's per-category spend vs limit; transformer `tHealth`: count categories per band.
- Logic notes: "On track" = spend < 80% of limit. Categories with no budget set are excluded from the ring but counted in a footnote ("2 categories have no budget"). Income excluded.
- Acceptance: (1) Segment counts match the underlying category bands. (2) Center label shows on-track count. (3) Colors also carry text labels (never color-only).

### 15. Show the most recent five transactions on the overview with category, description, and signed amount.

- Components: ListView `recentTxns` (5 rows): category icon, description, date, signed amount (red − / green +), SAMPLE pill where applicable, account chip.
- Data: query `qRecent5` (transactions ORDER BY date DESC, id DESC LIMIT 5, sample-filtered).
- Logic notes: Row click navigates to the Transactions screen filtered to that transaction (deep link). Future-dated "scheduled" transactions (item 72) show a "scheduled" tag instead of appearing as past spend.
- Acceptance: (1) Exactly the 5 most recent transactions render. (2) Signed amounts use red/green with explicit signs. (3) Click-through lands on the filtered Transactions view.

### 16. Add an "Update balances" shortcut on the overview that jumps to the accounts screen.

- Components: Button `btnUpdateBalances` (secondary style) in the accounts section header.
- Data: none (navigation only).
- Logic notes: Navigates to the Accounts tab with the balance-edit affordance focused on the first stale account (stale = updated_at > 30 days). This is a shortcut, not a bulk edit — balances are still updated one account at a time with updated_at stamped (item 45).
- Acceptance: (1) Button navigates to the Accounts screen. (2) The most-stale account's edit control is highlighted. (3) No balance is changed by the shortcut itself.

### 17. Render goal deadlines as countdown chips on the overview for goals with a deadline set.

- Components: In each goal row from item 3, a Chip `deadlineChip`: "42 days left" (neutral), "7 days left" (amber), "overdue" (red).
- Data: transformer `tCountdown`: days = ceil((deadline − today)/86400000).
- Logic notes: Goals without a deadline show no chip (not "no deadline" text clutter). Overdue goals keep their progress bar and show red "overdue by N days". Chips respect the sample toggle via the parent goal row.
- Acceptance: (1) Chip shows only when deadline is set. (2) Day counts are correct for the user's timezone. (3) Overdue state is red and distinct.

### 18. Show a per-account balance bar comparing Checking, Savings, Credit, and Investment totals.

- Components: Chart `typeBars` (horizontal bar, 4 bars, fixed category colors: Checking blue, Savings green, Credit red, Investment purple).
- Data: transformer `tTypeBars`: group sample-filtered accounts by type, Σ balance (credit negative → bar extends left).
- Logic notes: Negative credit totals render as leftward bars with a minus sign, visually distinct from asset bars. Empty type groups show a faint zero bar with the type label so all four types are always visible.
- Acceptance: (1) Four bars, one per account type, totals match account sums. (2) Credit renders negative with a minus sign. (3) Colors are consistent with the app's category/type palette (item 122).

### 19. Add an overview layout toggle between compact cards and expanded detail for iPhone screens.

- Components: Segmented control `layoutToggle` ("Compact" / "Detailed") in the overview header; temp-state `overviewLayout`.
- Data: none; the toggle switches container visibility and CSS density classes.
- Logic notes: Compact: stat tiles only, compact number format ($1.2k), sparkline hidden. Detailed: full cards, exact amounts, sparkline and top-categories visible. Default: Detailed on ≥768px, Compact on <768px (auto-detected once, then user override persists per session, item 110's pattern).
- Acceptance: (1) Toggle switches layouts without data reload. (2) Compact uses $1.2k formatting. (3) Choice persists for the session on mobile widths.

### 20. Include a footer note on the overview restating that all figures are manual entries or labeled sample data.

- Components: Text `footerNote` pinned at the bottom of the overview scroll: "All figures are manual entries or labeled sample data. FinanceOS has no bank connection."
- Data: none (static).
- Logic notes: This footer is distinct from the top banner (item 4) — it closes the screen with the same honesty statement. In print/statement views (item 100, 359) this text is included in the printed output.
- Acceptance: (1) Footer text is present at the bottom of the overview. (2) Wording names both manual entries and sample data. (3) It appears in printed exports of the overview.
