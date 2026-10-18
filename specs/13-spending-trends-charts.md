# FinanceOS Builder Spec — Section 13: Spending trends & charts (items 241–260)

## Tables touched
- `accounts`: id (pk), name, type (Checking/Savings/Credit/Investment), institution, balance, updated_at, is_sample — existing.
- `transactions`: id (pk), account_id (fk), date, description, category (Housing/Food/Transport/Income/Fun/Bills/Other), amount (negative=spend), is_sample — existing; all trend queries read from this table only.
- No new tables required for this section. Month keys use `to_char(date, 'YYYY-MM')`.

## Cross-cutting rules (apply to every item)
- The persistent banner "Manual tracking — no bank connection yet." appears at the top of the trends screen and every chart view.
- All queries filter `is_sample = {{ sampleToggle.value ? true : false }}` logic: when the global "hide sample data" toggle is off, exclude `is_sample=true` rows from every aggregation.
- Every chart carries the caption "Based on manually entered transactions only."
- Nothing in this section implies live bank data, auto-sync, or merchant feeds.

---

### 241. Build a trends screen with monthly spend totals from manual transactions.
- **Components:** New `trendsScreen` (Container). Header: Text `trendsBanner` (manual-tracking banner), `sampleToggle` Switch (shared global). `monthlyTotalsChart` (Bar chart), `periodSelector` (Segmented control: 3/6/12 months).
- **Data:** Query `q_monthly_totals`: `SELECT to_char(date,'YYYY-MM') AS m, SUM(CASE WHEN amount<0 THEN -amount ELSE 0 END) AS spend FROM transactions WHERE is_sample = ANY(...) AND date >= CURRENT_DATE - make_interval(months => {{periodSelector.value}}) GROUP BY 1 ORDER BY 1`. Transformer `t_monthly_labels` maps `m` to "Jan 26" style labels.
- **Logic notes:** Spend = absolute value of negative amounts only; Income category excluded from spend even if entered positive. Empty months in range render as zero bars, not gaps.
- **Acceptance:** (1) Screen loads with 12 monthly bars summing negative-amount transactions. (2) Changing the period selector re-queries and re-renders. (3) Banner visible above the chart at all times. (4) With no transactions, chart shows zero bars plus an empty-state note, no errors.

### 242. Render a 12-month bar chart of total spend vs total income.
- **Components:** `spendIncomeChart` (Bar chart, grouped mode) on `trendsScreen`; `incomeSpendLegend` (Text).
- **Data:** Query `q_spend_vs_income`: `SELECT to_char(date,'YYYY-MM') AS m, SUM(CASE WHEN amount<0 THEN -amount ELSE 0 END) AS spend, SUM(CASE WHEN amount>0 THEN amount ELSE 0 END) AS income FROM transactions WHERE <sample filter> AND date >= date_trunc('month', CURRENT_DATE) - INTERVAL '11 months' GROUP BY 1 ORDER BY 1`. Two series mapped in the chart component: Spend (red palette), Income (green palette).
- **Logic notes:** Income = positive amounts regardless of category; a refund entered as positive counts as income. Series gaps for months with no rows must be zero-filled via a generated month series LEFT JOIN.
- **Acceptance:** (1) 12 grouped bars render with correct spend/income per month. (2) Totals match the transactions table filtered to the same range. (3) Sample toggle off removes sample rows from both series. (4) Hover/tooltip shows both values per month.

### 243. Add a per-category monthly trend line chart with selectable categories.
- **Components:** `categoryMultiSelect` (Multiselect: Housing, Food, Transport, Income, Fun, Bills, Other; default top 3 by spend), `categoryTrendChart` (Line chart).
- **Data:** Query `q_category_trend`: `SELECT to_char(date,'YYYY-MM') AS m, category, SUM(ABS(amount)) AS total FROM transactions WHERE <sample filter> AND category = ANY({{categoryMultiSelect.value}}) AND amount<0 GROUP BY 1,2 ORDER BY 1`. Transformer pivots rows into one series per category.
- **Logic notes:** Cap selectable categories at 7 to keep lines readable; pivot in a transformer (`t_pivot_category`) keyed by month. Income category uses positive amounts when selected.
- **Acceptance:** (1) Selecting categories adds/removes lines live. (2) Each line's monthly points match filtered transaction sums. (3) Colors match the app-wide category color map. (4) Empty selection shows a prompt, not a broken chart.

### 244. Show average daily spend for the current month vs last month.
- **Components:** Two `Statistic` components (`avgDailyThisMonth`, `avgDailyLastMonth`) and a delta Text `avgDailyDelta` with up/down arrow.
- **Data:** Query `q_avg_daily`: `SELECT SUM(CASE WHEN date >= date_trunc('month',CURRENT_DATE) THEN -amount ELSE 0 END)/NULLIF(EXTRACT(DAY FROM CURRENT_DATE),0) AS this_m, SUM(CASE WHEN date >= date_trunc('month',CURRENT_DATE)-INTERVAL '1 month' AND date < date_trunc('month',CURRENT_DATE) THEN -amount ELSE 0 END)/NULLIF(EXTRACT(DAY FROM date_trunc('month',CURRENT_DATE)-INTERVAL '1 day'),0) AS last_m FROM transactions WHERE amount<0 AND <sample filter>`.
- **Logic notes:** Denominator = elapsed days in current month (not full month length) vs full days in last month. Guard division by zero on the 1st of the month at 00:00 edge.
- **Acceptance:** (1) Values equal manual recomputation from the transactions table. (2) Delta arrow points up when this month is higher. (3) On the 1st, shows "not enough data yet" rather than dividing by zero.

### 245. Display top 10 merchants/descriptions by spend from manual entries.
- **Components:** `topMerchantsTable` (Table, 10 rows): columns Description, Transactions (count), Total spend. Note text: "from your manual entries — descriptions you typed."
- **Data:** Query `q_top_descriptions`: `SELECT description, COUNT(*) AS n, SUM(-amount) AS total FROM transactions WHERE amount<0 AND <sample filter> GROUP BY 1 ORDER BY total DESC LIMIT 10`.
- **Logic notes:** Group by exact description string (no normalization — normalization is a separate AI feature). Case-sensitive grouping is acceptable v1; note it in help text.
- **Acceptance:** (1) Exactly 10 rows, ranked by total spend desc. (2) Count and total match the filtered transactions. (3) No bank/merchant logos or names beyond what the user typed.

### 246. Add a weekday vs weekend spend comparison chart.
- **Components:** `weekdayWeekendChart` (Bar chart, two bars) + `weekdayPatternText` (Text summary, e.g. "You spend 34% more on weekends.").
- **Data:** Query `q_weekday_weekend`: `SELECT CASE WHEN EXTRACT(DOW FROM date) IN (0,6) THEN 'Weekend' ELSE 'Weekday' END AS grp, SUM(-amount) AS total, COUNT(*) AS n FROM transactions WHERE amount<0 AND <sample filter> AND date >= CURRENT_DATE - INTERVAL '90 days' GROUP BY 1`. Transformer computes the percent difference.
- **Logic notes:** Window = last 90 days to smooth noise; DOW uses the app timezone (America/New_York default, user setting). Per-day averages (total/number of weekend vs weekday days in window) are the fair comparison, not raw totals.
- **Acceptance:** (1) Chart shows per-day average for weekend vs weekday. (2) Summary text states the direction and percent correctly. (3) Changing the sample toggle updates both bars.

### 247. Show spend by account as a stacked area chart.
- **Components:** `spendByAccountChart` (Area chart, stacked) with `accountLegend` driven by `q_accounts_list`.
- **Data:** Query `q_spend_by_account`: `SELECT to_char(t.date,'YYYY-MM') AS m, a.name AS acct, SUM(-t.amount) AS total FROM transactions t JOIN accounts a ON a.id=t.account_id WHERE t.amount<0 AND t.<sample filter> AND a.<sample filter> GROUP BY 1,2 ORDER BY 1`. Transformer pivots to one series per account.
- **Logic notes:** Join excludes sample accounts when samples hidden. Accounts with no spend in a month get zero (LEFT JOIN against month series), not a dropped series.
- **Acceptance:** (1) Stacked total per month equals the monthly spend total from item 241. (2) Each account has a stable color across sessions. (3) Toggling samples off removes sample-account layers.

### 248. Render a cumulative spend curve within the current month.
- **Components:** `cumulativeSpendChart` (Line chart) on `trendsScreen`; `todayMarker` reference line at the current date.
- **Data:** Query `q_cumulative_spend`: `SELECT date, SUM(SUM(-amount)) OVER (ORDER BY date) AS cum FROM transactions WHERE amount<0 AND <sample filter> AND date >= date_trunc('month',CURRENT_DATE) GROUP BY date ORDER BY date`. Transformer fills missing days by forward-carrying the last value.
- **Logic notes:** Curve is monotonically non-decreasing; x-axis spans the 1st to today (not the full month) to avoid implying future spend. Future-dated "scheduled" transactions are excluded.
- **Acceptance:** (1) Final point equals the month-to-date spend total. (2) Curve never decreases. (3) Days with no transactions show a flat segment, not a gap.

### 249. Add year-over-year monthly comparison using manual history.
- **Components:** `yoyChart` (Bar chart, grouped by year) + `yoyYearSelector` (Select: available years from data).
- **Data:** Query `q_yoy`: `SELECT EXTRACT(YEAR FROM date)::int AS y, EXTRACT(MONTH FROM date)::int AS mo, SUM(CASE WHEN amount<0 THEN -amount ELSE 0 END) AS spend FROM transactions WHERE <sample filter> GROUP BY 1,2 ORDER BY 1,2`. Transformer pivots to one series per selected year (max 3 years).
- **Logic notes:** Only months with data in at least one selected year render; partial current year is labeled "YTD". Requires ≥2 years of data to be meaningful — otherwise show guidance text.
- **Acceptance:** (1) Same calendar month aligns across year series. (2) Current-year months after today are excluded. (3) With <2 years of history, chart area shows the guidance note instead of a misleading single series.

### 250. Display the single largest transaction per month in a "biggest hits" list.
- **Components:** `biggestHitsList` (ListView or Table): columns Month, Description, Category, Amount. Sorted newest first.
- **Data:** Query `q_biggest_hits`: `SELECT DISTINCT ON (to_char(date,'YYYY-MM')) to_char(date,'YYYY-MM') AS m, date, description, category, -amount AS amt FROM transactions WHERE amount<0 AND <sample filter> ORDER BY 1, amt DESC`. Limit to last 12 months in the component.
- **Logic notes:** Ties broken by most recent date. Income transactions never appear (spend only). Clicking a row navigates to the transactions screen filtered to that month (deep link).
- **Acceptance:** (1) One row per month, the true max spend. (2) Row click opens the filtered transactions view. (3) Sample toggle respected.

### 251. Show category share as a 100% stacked bar per month.
- **Components:** `categoryShareChart` (Bar chart, 100% stacked mode) on `trendsScreen`.
- **Data:** Query `q_category_share`: `SELECT to_char(date,'YYYY-MM') AS m, category, SUM(-amount) AS total FROM transactions WHERE amount<0 AND <sample filter> GROUP BY 1,2`. Chart component normalizes each month's segments to 100%.
- **Logic notes:** Months with zero spend render as an empty/gray bar with a "no spend" tooltip, not a missing bar. Segment order fixed by the category sort order setting.
- **Acceptance:** (1) Each month's segments sum to 100%. (2) Segment values match `q_category_trend` for the same month. (3) Colors follow the category color map.

### 252. Add a "spending velocity" gauge: current pace vs budget pace.
- **Components:** `velocityGauge` (Gauge chart 0–150%) + `velocityText` ("On pace" / "X% over pace").
- **Data:** Transformer `t_velocity` over two queries: month-to-date spend (`q_mtd_spend`) and total monthly budget (`q_month_budget_total` = SUM(limit_amount) for current month from budgets). velocity = (spend / days_elapsed) / (budget / days_in_month) × 100.
- **Logic notes:** Budget pace assumes even daily spend — note this limitation under the gauge. If no budgets are set, gauge shows "set budgets to enable pace" instead of a number. Days-elapsed uses local today.
- **Acceptance:** (1) Gauge reads 100% when pace exactly matches budget pace. (2) No-budget state shows guidance, not 0%. (3) Value recomputes when a transaction is added.

### 253. Exclude sample data from trend charts when the sample toggle is off.
- **Components:** No new UI — this is the data-layer contract. Global `sampleToggle` Switch (already on overview; mirrored on trends screen header).
- **Data:** Every query in this section includes `AND is_sample = false` when `{{sampleToggle.value === false}}`; when true, includes all rows. Implement as a shared transformer `t_sample_clause` returning the SQL fragment, referenced by all trend queries.
- **Logic notes:** The toggle state persists per session (localStorage). Charts must re-run queries on toggle change — set all trend queries to run on `sampleToggle.change`.
- **Acceptance:** (1) With samples hidden, no `is_sample=true` row contributes to any chart. (2) Toggling re-renders every chart on the screen. (3) A small "samples hidden/shown" indicator sits next to the toggle.

### 254. Add chart tooltips showing exact amounts and transaction counts.
- **Components:** Chart-level tooltip config on every chart in this section (Retool chart "Tooltip" settings): custom template.
- **Data:** Queries in this section already return counts where needed; add `COUNT(*)` to `q_monthly_totals`, `q_spend_vs_income`, `q_category_trend` and expose via the tooltip template: `{{ self.x }} · ${{ self.y }} · {{ self.customData.count }} transactions`.
- **Logic notes:** Tooltip amounts formatted with the user's currency/number settings. For stacked charts, tooltip shows the segment value plus the month total.
- **Acceptance:** (1) Hovering any bar/point shows amount and count. (2) Amount matches the underlying query row. (3) Formatting follows the currency setting (symbol, decimals).

### 255. Support pinch-to-zoom on iPhone charts for dense histories.
- **Components:** Chart config: enable Retool chart zoom/pan (dataZoom) on `monthlyTotalsChart`, `spendIncomeChart`, `categoryTrendChart`; add `resetZoomButton` (Button, appears only when zoomed).
- **Data:** No query change. Transformer `t_zoom_window` tracks the visible range so the reset button knows when to show.
- **Logic notes:** Zoom is x-axis only (time); y-axis stays fixed to avoid misreading scale. Minimum window = 1 month. Double-tap resets on iOS as a fallback to the button.
- **Acceptance:** (1) Pinch zooms the time axis on iPhone Safari. (2) Reset button restores the full range. (3) Zoom state does not corrupt tooltip values.

### 256. Show a 3-month moving average line over monthly spend bars.
- **Components:** `monthlyTotalsChart` gains a second Line series `ma3` (dashed).
- **Data:** Transformer `t_moving_avg` over `q_monthly_totals` output: for each month index i ≥ 2, ma = mean(spend[i-2..i]); first two months get null (no partial averages — avoids a misleading ramp).
- **Logic notes:** Moving average respects the sample toggle because it derives from the already-filtered query. Label the series "3-mo avg".
- **Acceptance:** (1) Line starts at the 3rd month. (2) Values equal the manual 3-month mean. (3) Line updates when the period selector or sample toggle changes.

### 257. Add a heatmap of spend by day of month.
- **Components:** `dayOfMonthHeatmap` (Heatmap/Grid chart: rows = months, columns = day 1–31) + `heatmapScale` legend.
- **Data:** Query `q_heatmap`: `SELECT to_char(date,'YYYY-MM') AS m, EXTRACT(DAY FROM date)::int AS d, SUM(-amount) AS total FROM transactions WHERE amount<0 AND <sample filter> AND date >= CURRENT_DATE - INTERVAL '6 months' GROUP BY 1,2`. Transformer pivots to a 6×31 matrix, nulls → 0.
- **Logic notes:** Color scale is per-dataset max (relative), labeled "darker = more spend (relative)". Days beyond month length (e.g., Feb 30) render blank, not zero-colored.
- **Acceptance:** (1) Cell value matches the day's transaction sum. (2) Impossible days are blank. (3) Scale legend states the range and that it is relative.

### 258. Display income trend separately from spend trend with a toggle.
- **Components:** `trendModeToggle` (Segmented control: Spend / Income) above a shared `mainTrendChart` (Line chart).
- **Data:** One query `q_trend_mode`: `SELECT to_char(date,'YYYY-MM') AS m, SUM(CASE WHEN {{trendModeToggle.value}}='Income' THEN CASE WHEN amount>0 THEN amount ELSE 0 END ELSE CASE WHEN amount<0 THEN -amount ELSE 0 END END) AS total FROM transactions WHERE <sample filter> GROUP BY 1 ORDER BY 1` — or two queries switched by the toggle; either is acceptable, single query preferred.
- **Logic notes:** Income mode sums positive amounts (all categories); Spend mode sums negative absolute values. Chart title updates: "Monthly income trend" vs "Monthly spend trend".
- **Acceptance:** (1) Toggle switches the series and title without reload. (2) Income totals match the Income-filtered transaction sums. (3) Caption still reads "manual data only" in both modes.

### 259. Add one-tap sharing of a chart as an image (with sample watermark if samples included).
- **Components:** `shareChartButton` (Button, icon Share) on each chart card; `sharePreviewModal` (Modal showing the exported PNG before sharing, with watermark notice when applicable).
- **Data:** JS query `q_export_chart`: uses the chart component's export/download API to PNG, draws the PNG to a canvas overlay adding: banner text "Manual tracking — no bank connection yet.", caption "Based on manually entered transactions", and — only when sample rows are included — a diagonal "SAMPLE DATA" watermark. Then invokes `navigator.share()` with fallback to download.
- **Logic notes:** Watermark condition: `sampleToggle.value === true AND chart dataset contains is_sample rows`. Export resolution 2x for legibility. No server round-trip; all client-side.
- **Acceptance:** (1) Shared image contains the banner + caption. (2) Watermark appears iff sample rows are included. (3) On iOS, the native share sheet opens; on desktop, the PNG downloads.

### 260. Include a "manual data only" caption under every trend chart.
- **Components:** A shared `captionText` Text component style (small, muted) placed under each chart on `trendsScreen`; or a single reusable module `mod_chart_caption`.
- **Data:** Static text: "Manual data only — from transactions you entered. No bank connection." No query.
- **Logic notes:** This is a build-checklist item: add a QA step to the trends screen build notes — "every chart on this screen has the caption component directly beneath it." Captions must survive chart re-renders and zoom states.
- **Acceptance:** (1) Every chart in this section has the caption visible beneath it. (2) Caption text is identical everywhere. (3) Caption remains visible when the chart is zoomed or filtered.
