# FinanceOS Builder Spec — §27 Charts & Visualization Library (Items 521–540)

## Section overview
Standardized chart layer used by every screen: a shared type registry, one category-color palette, period/sample controls wired identically on each chart, and supporting features (labels, annotations, alt text, downloads, empty states, fullscreen, caching, confidence shading). Charts render via the Retool Chart component (Plotly-based); all figures derive from manual `transactions`, `accounts`, `budgets`, and net-worth snapshots, never from a live feed.

## Tables touched
- `transactions`, `accounts`, `budgets`, `goals` (existing; see App 11 in specs.md).
- NEW `chart_presets`: id (pk), preset_name (text), chart_type (text: bar / line / donut / stacked_bar / area / heatmap), period_key (text: 3m / 6m / 12m / ytd / all), filters_json (text: category/account/period state), created_at (timestamp), is_sample (bool, default false). Seed with 3 sample rows ("Monthly spend trend", "Spend by category", "Net worth area").
- NEW `chart_annotations`: id (pk), chart_key (text), annotate_date (date), note (text), created_at (timestamp), is_sample (bool, default false).

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen, including the chart gallery and fullscreen views.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data toggle hides samples, they are excluded from chart series and totals.
- Nothing ever presents as a live bank feed; all charts aggregate manual entries or labeled samples. Downloads carry a manual-data caption.

---

### 521. Standardize a chart library: bar, line, donut, stacked bar, area, heatmap.
- Components: Retool Chart `chtMain` with a type registry implemented in a JS query `qryChartTypes` returning [{key:'bar',label:'Bar'},{key:'line',...},{key:'donut'...},{key:'stacked_bar'...},{key:'area'...},{key:'heatmap'...}]. Every chart screen reuses the same wrapper container `cntChartShell` (header: title, type select, period select, sample toggle; body: chart).
- Data: Transformer `trChartData` maps a canonical data shape {labels[], series[{name, data[], color}]} to each chart's Plotly config. Heatmap uses intensity values computed per cell.
- Logic notes: One canonical shape means one period/sample code path for all six types. Stacked bar = bar with barmode='stack'. Donut = pie with hole=0.45.
- Acceptance: (1) All six types render from the same transformer. (2) Switching type keeps period/sample state. (3) No chart uses ad-hoc bespoke config outside the registry.

### 522. Apply consistent category colors across every chart in the app.
- Components: App-level variable `varCategoryColors` (Object) mapping category → hex: Housing #5B8DEF, Food #F59E0B, Transport #10B981, Income #22C55E, Fun #A855F7, Bills #EF4444, Other #94A3B8.
- Data: `trChartData` assigns series colors by looking up this map; new/unknown categories fall back to #64748B and log a console warning.
- Logic notes: Single source of truth — change one hex and every chart updates. Color also used by budget bars and legend chips so the taxonomy reads identically everywhere.
- Acceptance: (1) "Food" is the same hex on the donut, stacked bar, and budget screen. (2) Changing the variable updates all charts. (3) Unknown category renders a neutral gray, never fails to render.

### 523. Add a chart-period selector (3/6/12 months, year-to-date, all time).
- Components: Segmented control `segChartPeriod` (3M / 6M / 12M / YTD / All) inside every `cntChartShell`.
- Data: Transformer `trChartPeriod` maps the key to {start, end}: 3M = today − 90 days; 6M/12M analogous; YTD = Jan 1 → today; All = earliest transaction date → today.
- Logic notes: End is always today (no future). "All" caps at data range; if only sample data exists, range uses sample dates too. Changing period re-triggers the chart's data query.
- Acceptance: (1) Each preset produces the expected date window. (2) YTD starts Jan 1 of the current year. (3) Charts re-render within one query cycle of a change.

### 524. Support toggling sample data in/out on each chart.
- Components: Switch `swtChartSamples` ("Include sample data") in every chart shell, default ON.
- Data: Chart queries add `AND (is_sample = false OR {{ swtChartSamples.value }})` per table. `trChartData` also counts sample rows for a "N sample rows included" caption.
- Logic notes: Toggle is per-chart but defaults from the app-level sample preference. Totals and series both exclude samples when off. The SAMPLE badge appears in the chart subtitle when on.
- Acceptance: (1) Toggling off removes sample rows from every series and total. (2) Caption states sample inclusion. (3) Toggle state persists per chart across tab switches.

### 525. Add data labels on bars with exact dollar amounts.
- Components: In the Chart component's Plotly JSON, set `texttemplate: '$%{y:,.0f}'`, `textposition: 'outside'` for bar and stacked bar types; `textinfo: 'label+percent'` for donuts.
- Data: `trChartData` injects the label config only for bar/donut types; line/area/heatmap skip labels to avoid clutter (labels available via hover template instead).
- Logic notes: Labels use accounting format ($1,250, not $1250.00). For small bars, label falls inside or hides below a pixel threshold; totals still exact in the subtitle.
- Acceptance: (1) Bar chart shows exact dollar labels on each bar. (2) Donut shows label + percent. (3) Line/area rely on hover, labels hidden.

### 526. Build a net-worth area chart from manual snapshots.
- Components: Chart `chtNetWorth` (area type) on Overview; data source is NEW concept: net-worth snapshots. Introduce table `networth_snapshots`: id (pk), snapshot_date (date), total_net_worth (number), is_sample (bool). "Record snapshot" Button appends one row (sum of account balances at click time).
- Data: Query `qrySnapshots` orders by date; transformer fills missing months by carrying forward the last value (step interpolation), never inventing new data.
- Logic notes: No bank connection means no auto snapshots — chart only shows manually recorded points; gaps are labeled "no snapshot recorded" rather than smoothed over.
- Acceptance: (1) Recording a snapshot adds a point immediately. (2) Gaps carry forward, flagged as estimates. (3) Area fill is Income-green above zero, red below.

### 527. Build a budget-burndown line per category within the month.
- Components: Chart `chtBurndown` (line) on Budgets screen; Select `selBurndownCategory` picks the category.
- Data: Query aggregates daily cumulative spend per category for the current month. Transformer adds the ideal linear burndown line: (limit / daysInMonth) × day. Red markers on days spend exceeded the ideal line.
- Logic notes: Ideal line uses days elapsed, not flat average. Future days show ideal only (dashed); past days show actual (solid) vs ideal. Categories with no budget show spend line only.
- Acceptance: (1) Ideal line is linear across the month. (2) Actual crosses show red markers. (3) Future days don't plot actuals.

### 528. Add a sankey-style flow: income → categories → savings.
- Components: Chart `chtFlow` using Plotly `sankey` type via the Chart component's Plotly JSON (Retool supports raw Plotly traces).
- Data: Query sums income (positive amounts) per account for the period → nodes: "Income" → each category node → "Savings". Link values are absolute dollars; width ∝ value.
- Logic notes: Only periods with recorded income render a meaningful flow; if total income = 0, show the empty state instead of a single node. Savings node = income − spend, labeled "Unspent (savings)" and never a bank balance.
- Acceptance: (1) Link widths proportional to dollars. (2) Zero-income period shows empty state. (3) Hover shows exact $ per link.

### 529. Show a histogram of transaction sizes to spot unusual spending.
- Components: Chart `chtTxnSizes` (bar/histogram) on Transactions screen; range slider `rngBin` (bin width $10/$25/$50) and toggle "spend only".
- Data: Transformer bins |amount| into N bins (default 20); counts per bin. Outlier bins (count = 1 and value > 3× median) get the red highlight color.
- Logic notes: Income excluded when "spend only" is on. Bin count adapts: sqrt(n) rule with min 5, max 40. Hover shows bin range and count.
- Acceptance: (1) Bins computed from absolute amounts. (2) Singleton extreme bins highlighted red. (3) Bin-width control re-bins live.

### 530. Add sparklines in table rows for category and account trends.
- Components: In the transactions table, add a custom column "Trend" rendering a mini SVG sparkline via a mapped value (Retool HTML column type). Same for the accounts table (balance trend over last 6 snapshots).
- Data: Transformer `trSparkline(rowId)` computes last-12-period values for the row's category/account and returns an inline SVG path string (no external lib needed).
- Logic notes: Sparklines are ~80×24 px, no axes, line only. Clicking a sparkline drills to the full chart. Sample rows render sparkline in a faded style.
- Acceptance: (1) Each row shows a trend sparkline. (2) SVG renders without external assets. (3) Click navigates to the category/account chart.

### 531. Support comparing two custom periods on the same chart.
- Components: Date Range `dtrCompareA` and `dtrCompareB` in the chart shell ("Compare" mode toggle `swtCompare`), plus a "vs" legend.
- Data: Chart query runs twice (period A, period B); transformer aligns both to day-index (1..N) so different lengths still overlay. Series names suffixed "(A)" and "(B)".
- Logic notes: Comparison overlays by relative day, not calendar date, with a caption stating exact ranges. B uses dashed/dimmed variants of the same colors.
- Acceptance: (1) Two periods overlay on one chart. (2) X-axis is day-index with a caption naming both ranges. (3) Turning compare off restores the single-period view.

### 532. Add an "annotate chart" feature placing notes on specific months.
- Components: "Annotate" Button `btnAnnotate` opens a Modal `mdlAnnotate` (Month picker, note text, save). Uses `chart_annotations` table.
- Data: On save, INSERT into chart_annotations; the chart transformer adds Plotly `annotations` at the month's x position with the note text, and a marker dot on the series.
- Logic notes: Annotations belong to a chart_key + month; deleting an annotation removes its marker. Notes are manual — no auto-generated annotations.
- Acceptance: (1) Saved note appears as a marker + label on the month. (2) Notes persist per chart across sessions. (3) Delete removes marker.

### 533. Ensure charts are readable in dark mode.
- Components: App theme toggle drives `varChartTheme` {bg, grid, text, tooltip} read by `trChartData` when building Plotly `layout`.
- Data: Transformer picks palette: light → white bg / #E5E7EB gridlines / #111827 text; dark → #111827 bg / #374151 gridlines / #F9FAFB text. Category colors stay identical in both.
- Logic notes: Plotly `paper_bgcolor`/`plot_bgcolor` and font colors switch; legend and tooltip inherit. Contrast of smallest label ≥ 4.5:1 against bg in both modes.
- Acceptance: (1) All charts legible in dark mode (gridlines visible, labels readable). (2) Category colors unchanged. (3) Toggle updates instantly without reload.

### 534. Provide chart alt-text summaries for screen readers.
- Components: Each chart gets a visually-hidden Text `txtChartAlt` (aria-label binding) above the Chart component.
- Data: Transformer `trChartAlt(chartKey)` generates a sentence: "Bar chart, spend by category, March 2026. Top category Food at $1,250. 6 categories, total $3,410. Sample data included." Built from the same aggregated data.
- Logic notes: Alt text regenerates on every data change (period/sample/type). It summarizes, not lists every point. Empty state gets "No manual data yet for this chart."
- Acceptance: (1) Screen reader announces a data summary per chart. (2) Text updates when filters change. (3) Empty charts announce the empty state.

### 535. Add one-tap chart image download with the manual-data caption embedded.
- Components: Button `btnChartDownload` (download icon) in each chart shell; uses Retool's `utils.downloadPage` or a canvas snapshot of the chart.
- Data: JS query composes the filename `financeos-{chartKey}-{period}.png` and stamps the caption "Manual entries — no bank connection. Exported {date}." into the image footer or download card.
- Logic notes: Export includes current filters in the caption (period + sample state). PNG only; minimum 1200 px wide for legibility.
- Acceptance: (1) One tap downloads a PNG. (2) Caption with manual-data disclaimer is embedded. (3) Filename reflects chart + period.

### 536. Show empty-state illustrations when a chart has no manual data yet.
- Components: Container `cntChartEmpty` (illustration image from Image component, headline, "Add your first transaction" Button) swapped in when series data length = 0.
- Data: `trChartData` returns {empty: true} when no rows; the shell shows the empty container instead of the Chart component.
- Logic notes: Empty state never fabricates a placeholder chart. CTA routes to the relevant manual-entry form (add transaction / record snapshot / set budget).
- Acceptance: (1) Zero-data charts show illustration + CTA, not blank axes. (2) CTA opens the correct entry form. (3) Adding data swaps back to the chart.

### 537. Add a fullscreen chart view on iPhone with landscape support.
- Components: Fullscreen Button `btnChartFull` opens a Modal `mdlChartFull` set to 100vw/100vh with the chart re-rendered at the larger size; a "Rotate for best view" hint Text shows in portrait.
- Data: Same transformer/query as the source chart; Plotly `layout.autosize` re-fits on modal open and on orientation change (window resize listener).
- Logic notes: Fullscreen is a view of the same chart, not a new one — period/sample/compare state carries over. Modal closes restore scroll position.
- Acceptance: (1) Fullscreen opens the same chart bigger. (2) Landscape orientation widens the plot area. (3) Closing returns to the prior screen state.

### 538. Cache chart data for instant re-render on tab switches.
- Components: App variable `varChartCache` (Object keyed by `${chartKey}|${period}|${sampleOn}`) holding {data, fetchedAt}.
- Data: Chart queries check the cache first: if key exists and age < 5 min, `trChartData` consumes cache; else the query runs and writes the cache. Mutations (add transaction, record snapshot, edit budget) invalidate affected keys.
- Logic notes: 5-minute TTL; invalidation is explicit, not timer-based. Cache stores raw rows, not rendered config, so theme/type changes re-render free.
- Acceptance: (1) Tab switch back re-renders from cache with no query. (2) New transaction invalidates related charts. (3) Stale (>5 min) entries refetch.

### 539. Display confidence shading on projection charts labeled as estimates.
- Components: On projection charts (forecast screen), Plotly traces add an upper/lower band (fill='tonexty', rgba) around the central estimate line.
- Data: Transformer computes band as ±(growing % of value: 5% at month 1 → 25% at month 12) — simple heuristic, documented in the chart subtitle: "Shading shows estimate range, not a guarantee."
- Logic notes: The band never renders without the "ESTIMATE" label chip on the chart and the banner visible. Actuals (manual) are solid; only projections get shading.
- Acceptance: (1) Band widens with horizon. (2) "ESTIMATE" chip always visible. (3) No shading on historical actuals.

### 540. Add a chart gallery screen linking every available visualization.
- Components: New screen "Charts" — Grid View `grdChartGallery` of cards (thumbnail via mini Chart, title, description, "Open" Button). Includes saved `chart_presets` as "My presets" section.
- Data: `qryPresets` lists chart_presets; each card deep-links to its chart with preset filters applied (sets segChartPeriod, swtCompare, etc. via URL/query params on navigation).
- Logic notes: Gallery lists every chart from items 521–539 plus the standard ones; new charts register themselves in a `varChartRegistry` list. Presets are user-saved, editable, deletable.
- Acceptance: (1) Every visualization appears as a card. (2) Opening a card applies the right chart + filters. (3) Presets save/load/delete correctly.
