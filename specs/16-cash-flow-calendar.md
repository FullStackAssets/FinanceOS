# FinanceOS Builder Spec — §16 Cash Flow Calendar (Items 301–320)

## Section overview
A forward-looking cash-flow calendar that projects daily balances from recorded account balances plus planned items (bills due, expected income, recurring templates). All projections are computed client-side from manual data and are always labeled as estimates — never bank forecasts.

## Tables touched
- `accounts` (existing): id, name, type (Checking / Savings / Credit / Investment), institution, balance, updated_at, is_sample.
- `transactions` (existing): id, account_id (fk), date, description, category, amount (negative=spend), is_sample.
- NEW `planned_cash_items`: id (pk), item_type (text: bill / expected_income / what_if), label (text), account_id (fk, nullable), planned_date (date), amount (number, negative=outflow), frequency (text: once / weekly / biweekly / monthly / custom_interval, nullable), linked_bill_id (fk, nullable), linked_template_id (fk, nullable), confirmed (bool, default false), is_sample (bool, default false), created_at (timestamp). Drives the calendar and projection engine.
- NEW `cash_flow_settings`: id (pk), low_balance_floor (number, default 0), projection_horizon_days (number, default 90), include_sample_accounts (bool, default false), updated_at (timestamp). Single row per user profile.

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen, including the calendar.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample toggle hides samples, sample accounts and items are excluded from the projection, totals, and exports.
- Nothing ever presents as a live bank feed; all projections are labeled "estimate from your manual entries — not a guarantee."

---

### 301. Build a monthly calendar showing expected income and bills per day from manual entries.
- Components: Custom month-grid component `calCashFlow` (6 rows × 7 cols), month/year navigation (`btnPrevMonth`, `btnNextMonth`, `lblMonthTitle`), day cells rendering item chips.
- Data: Query `qryPlannedItems` reads `planned_cash_items` where planned_date within the visible month and confirmed or item_type in (bill, expected_income); query `qryBillsDue` expands bills into due dates for the month. Transformer `trCalendarDays` merges bills (negative, red chip) and expected income (positive, green chip) per day.
- Logic notes: Bills come from the bills table via planned_cash_items rows of type bill; income from expected-income entries. Past days show recorded transactions instead of planned items (dimmed).
- Acceptance: (1) Every bill due in the month appears on its due date. (2) Expected income entries appear on their planned dates. (3) Chips show label + signed amount; tapping a day opens the day popover (item 307).

### 302. Compute a projected daily balance line from recorded balances plus planned items.
- Components: Line chart `chartProjectedBalance` above the calendar; stat `statStartingBalance` showing the projection anchor.
- Data: Transformer `trProjection` starts from SUM(accounts.balance) for included accounts (as-of today), then walks forward day by day adding planned_cash_items amounts in date order. Query `qryStartingBalances` fetches current recorded balances once.
- Logic notes: Projection anchor = recorded balances, not a bank figure. Only items on/after today affect the forward line; historical days use recorded transactions (item 317). Recompute on any planned-item change.
- Acceptance: (1) Day-1 value equals the sum of included account balances. (2) Each day's value = prior day + that day's net planned items. (3) Line updates instantly when a planned item is added, moved, or deleted.

### 303. Mark planned items distinctly from recorded transactions on the calendar.
- Components: Day-cell chips with style variants; legend row `legendPlannedVsActual` under the calendar.
- Data: No new queries — `trCalendarDays` tags each chip with `source: 'planned' | 'recorded'`.
- Logic notes: Planned chips use dashed borders and a clock icon; recorded chips use solid borders. Tooltips read "Planned — not yet recorded" vs "Recorded manual entry". Colors alone never carry the meaning (accessibility: icon + text label "P").
- Acceptance: (1) Every chip is visually distinguishable by source at a glance. (2) Legend explains both styles. (3) Screen-reader labels announce the source.

### 304. Show a low-balance warning on days the projection dips below a user-set floor.
- Components: Number input `numLowBalanceFloor` in calendar settings drawer; warning icon overlay on affected day cells; alert banner `alertLowBalance` listing the first breach date.
- Data: Transformer `trProjection` flags days where projected balance < `cash_flow_settings.low_balance_floor`; query `qryCashFlowSettings` loads/saves the floor.
- Logic notes: Floor defaults to 0. Warning is advisory only — it never blocks entry. Amber for within 10% above floor is optional; red when below.
- Acceptance: (1) Setting a floor of $500 flags every projected day under $500. (2) Affected day cells show a warning icon. (3) Clearing the floor to 0 only flags negative projections.

### 305. Allow dragging a planned bill to a new date to model paying early or late.
- Components: Draggable bill chips on `calCashFlow` (HTML5 drag on desktop, long-press drag handle on iPhone); drop targets on day cells.
- Data: On drop, query `qryMovePlannedItem` updates `planned_cash_items.planned_date`; `trProjection` recomputes immediately. Undo toast restores the prior date.
- Logic notes: Only planned (unconfirmed) items are draggable; recorded transactions are fixed. Dragging updates the linked bill's planned date, not the bill template's due day. A "reset to due date" action restores the original.
- Acceptance: (1) Dropping a bill chip on another day moves it and the projection line shifts. (2) The chip shows "moved from <original date>" until reset. (3) Undo restores the prior date within 10 seconds.

### 306. Display a 90-day forward cash-flow chart based on recurring manual patterns.
- Components: Area/line chart `chart90Day` with toggle `segHorizon` (30 / 60 / 90 days).
- Data: Transformer `trForward90` expands recurring `planned_cash_items` (frequency weekly/biweekly/monthly) and recurring templates for the next N days, then accumulates from the current recorded balance. Query `qryRecurringPlanned` supplies the recurring rows.
- Logic notes: Expansion is pure date math — no auto-posting; these are projections only. Monthly items land on the same day-of-month (clamped to month end). Label the chart "90-day estimate from your manual entries — not a guarantee."
- Acceptance: (1) Chart renders 90 daily points from today. (2) Recurring bills and income repeat at their frequencies. (3) Horizon toggle re-renders without reload.

### 307. Add per-day detail popover listing that day's planned and recorded items.
- Components: Popover `popDayDetail` anchored to the tapped day cell; list `lstDayItems` grouped by Planned / Recorded; quick actions per row.
- Data: Transformer `trDayDetail` filters merged day data for the selected date; queries reuse `qryPlannedItems` and the recorded-transactions query for that date.
- Logic notes: Past dates show only recorded items; future dates show planned items; today shows both. Each row shows label, amount, source badge, and (for planned) confirm/edit actions.
- Acceptance: (1) Tapping any day opens the popover with that day's items. (2) Planned and recorded sections are visually separated. (3) Empty days show "No items — add a planned bill or income."

### 308. Support multiple accounts in the projection with a per-account toggle.
- Components: Account multi-select `selProjectionAccounts` (checkbox list with balances) in the calendar toolbar; "All" / "None" quick actions.
- Data: `trProjection` filters the starting balance sum and per-day account-specific items by selected account ids. Planned items with null account_id apply to the aggregate.
- Logic notes: Default = all non-sample checking/savings accounts. Toggling re-anchors the projection instantly. Per-account daily lines are optional via a second chart series toggle.
- Acceptance: (1) Deselecting an account removes its balance from the anchor and its items from the flow. (2) Selection persists per session. (3) Headline numbers update on every toggle change.

### 309. Show the projected end-of-month balance as a headline number.
- Components: Hero stat `statMonthEndProjection` ("Projected month-end") with subtext of net planned change.
- Data: Reads the last day value from `trProjection`; subtext = end value − starting balance.
- Logic notes: Labeled "Projected" with the estimate disclaimer inline. Turns amber when below the low-balance floor, red when negative.
- Acceptance: (1) Number equals the projection's final day value. (2) Subtext shows +/− net change vs today. (3) Estimate disclaimer is visible adjacent to the number.

### 310. Clearly label all projections as estimates from manual data, not bank forecasts.
- Components: Caption component `capEstimateDisclaimer` under every projection chart/number; info tooltip on the calendar header.
- Data: Static text; no queries.
- Logic notes: Exact wording: "Estimate from your manual entries — not a bank forecast." Appears on the calendar, 90-day chart, month-end headline, and what-if panel. Never abbreviated away on small screens (wraps, never hidden).
- Acceptance: (1) Every projection surface shows the disclaimer. (2) Wording matches exactly. (3) Disclaimer persists in exported/shared images of the calendar.

### 311. Add a "what-if" mode: add a hypothetical expense and see the impact.
- Components: Toggle `tglWhatIf`; form `frmWhatIf` (label, amount, date, account); what-if chips rendered in purple on calendar and projection; "clear all" button.
- Data: What-if rows are `planned_cash_items` with item_type='what_if' and confirmed=false, excluded from all other screens via the type filter. `trProjection` includes them only when the toggle is on.
- Logic notes: What-if items never convert to real transactions automatically; a "convert to planned bill" action exists with user confirmation. Deleting all what-ifs restores the baseline projection.
- Acceptance: (1) Adding a $2,000 expense on the 15th drops the projection from that date forward. (2) Toggling what-if mode off restores the baseline instantly. (3) What-if chips are visually distinct (purple, "WHAT-IF" tag).

### 312. Display paycheck-to-paycheck coverage: bills due before next expected income.
- Components: Coverage card `cardPaycheckCoverage` listing bills due between today and the next expected income date, with total and a coverage ratio.
- Data: Transformer `trCoverage` finds the next `planned_cash_items` row of type expected_income with planned_date ≥ today, then sums bill-type items due before that date.
- Logic notes: Ratio = expected income ÷ bills due. Green ≥ 1.2, amber 1.0–1.2, red < 1.0 with "shortfall of $X" text. If no expected income is planned, show "Add your next expected income to see coverage."
- Acceptance: (1) Card lists exactly the bills due in the window with amounts. (2) Ratio math is shown, not just a color. (3) Empty state guides the user to add expected income.

### 313. Highlight the lowest projected balance day of the month.
- Components: Special highlight ring on the min-balance day cell in `calCashFlow`; annotation dot on `chartProjectedBalance`.
- Data: `trProjection` computes min over the visible month's daily values and exposes `minDate`/`minValue`.
- Logic notes: If multiple days tie, highlight the earliest. The highlight updates when items move. Tooltip: "Lowest projected balance: $X on <date>."
- Acceptance: (1) Exactly one day per month view carries the highlight. (2) Highlight moves correctly after dragging a bill. (3) Value shown matches the projection data.

### 314. Support weekly and biweekly pay schedules in the projection engine.
- Components: Frequency select on expected-income entries: once / weekly / biweekly / monthly; pay-schedule presets in the income form.
- Data: `trForward90` / `trProjection` expand weekly (+7 days) and biweekly (+14 days) from the start date across the horizon. Stored as `planned_cash_items.frequency`.
- Logic notes: Biweekly anchor = the entered start date; expansion never drifts. Monthly uses same-day-of-month clamping. A preview list shows the next 6 generated dates before saving.
- Acceptance: (1) Weekly income appears every 7 days in the projection. (2) Biweekly income appears every 14 days from the anchor. (3) Preview list matches the generated dates exactly.

### 315. Show a confidence note: projections degrade without regular manual updates.
- Components: Confidence banner `bannerProjectionConfidence` with levels High / Medium / Low and explanatory text.
- Data: Transformer `trConfidence` checks: days since any account `updated_at` (stale > 30 days → Low), count of planned items in horizon (< 3 → Medium), sample exclusion state.
- Logic notes: High = all included accounts updated within 14 days and ≥3 planned items. Medium = one condition missed. Low = any account stale > 30 days. Text: "Projections degrade without regular manual updates — update your balances to improve accuracy."
- Acceptance: (1) Level recomputes on data changes. (2) Each level shows the reason (e.g., "2 accounts not updated in 30+ days"). (3) Never claims accuracy it cannot verify.

### 316. Add calendar export (ICS) of bill due dates entered manually.
- Components: Button `btnExportICS` ("Export bills (.ics)") on the calendar toolbar; month-range selector for the export.
- Data: JS query `qryBuildICS` generates VEVENT entries from bill-type `planned_cash_items` in the selected range: DTSTART = planned_date, SUMMARY = label + amount, DESCRIPTION includes "Manual tracking — no bank connection yet." Downloads via a data URI.
- Logic notes: Only confirmed planned bills are exported; what-if items excluded. Times are all-day events. Filename `financeos-bills-YYYY-MM.ics`.
- Acceptance: (1) Downloaded file imports into Apple/Google Calendar with correct dates. (2) Each event shows label and amount. (3) What-if and unconfirmed items are absent.

### 317. Display past months' actual daily balances reconstructed from transactions.
- Components: Mode toggle `segPastFuture` (Past actuals / Projection) on the calendar; past-day cells show reconstructed balances.
- Data: Transformer `trPastBalances` starts from the earliest known recorded balance and walks forward applying recorded `transactions` amounts per day per account. Query `qryHistoricalTxns` fetches the needed range.
- Logic notes: Reconstruction is an estimate — labeled "reconstructed from your manual entries; gaps possible where balances weren't updated." Days before the first known balance show "unknown," never fabricated.
- Acceptance: (1) Past days show a daily balance derived from transactions. (2) Days pre-dating known balances show "unknown." (3) The reconstruction label is visible in past mode.

### 318. Allow excluding sample accounts from the projection.
- Components: Toggle `tglExcludeSamples` (default ON) in calendar settings; sample-account chips shown greyed with SAMPLE badge when included.
- Data: `trProjection` and all calendar queries add `AND (is_sample = false OR {{ tglExcludeSamples.value }} = false)` semantics via the global sample-visibility state.
- Logic notes: Default excludes samples so projections reflect real money only. When samples are visible, their chips carry SAMPLE badges and the estimate disclaimer notes sample inclusion.
- Acceptance: (1) Default projection ignores sample accounts and their items. (2) Toggling samples on adds them with badges. (3) Totals update instantly on toggle.

### 319. Show a side-by-side planned vs actual cash flow at month end.
- Components: Comparison panel `pnlPlannedVsActual` with two columns (Planned / Actual) and a variance column; available for closed months via month picker.
- Data: Transformer `trMonthCompare` aggregates planned_cash_items (planned_date in month) vs recorded transactions (date in month) by category, plus totals and net.
- Logic notes: Variance = actual − planned, with red/green coloring and a plain-language note ("Bills came in $120 higher than planned"). Only months fully in the past are comparable; the current month shows "in progress."
- Acceptance: (1) Each category row shows planned, actual, variance. (2) Totals reconcile with the transactions table. (3) Current month is labeled in-progress, not final.

### 320. Add an iPhone-optimized week-strip view of the cash-flow calendar.
- Components: Horizontal week strip `stripWeek` (7 day cards, swipeable between weeks) shown below 480px width; each card shows net planned/recorded amount and low-balance dot.
- Data: Reuses `trCalendarDays` and `trProjection`; no new queries. Week navigation via swipe gestures and prev/next chevrons.
- Logic notes: Cards are ≥44px touch targets; the strip replaces the month grid on small screens (month grid hidden via responsive visibility). Tapping a card opens the same day popover (item 307). Safe-area insets respected.
- Acceptance: (1) At 390px width the week strip renders with no horizontal page scroll. (2) Swiping moves exactly one week. (3) Low-balance and lowest-day highlights appear on the strip.
