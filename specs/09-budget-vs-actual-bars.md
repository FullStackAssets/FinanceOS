# FinanceOS Builder Spec — Section 09: Budget vs actual bars (items 161–180)

## Section title
Budget vs actual bars — per-category spend bars comparing this month's budget limits against actuals computed from manually entered transactions, plus projections, drill-downs, and export.

## Tables touched
No new tables. Uses existing:
- `budgets`: id (pk), category (text), month (text `YYYY-MM`), limit_amount (number)
- `transactions`: id (pk), account_id (fk), date (date), description (text), category (text), amount (number, negative=spend), is_sample (bool)

Month-over-month comparison (item 170) derives last month's rows from `transactions` + `budgets`; no schema change.

## Cross-cutting rules
- Every screen shows the persistent banner **"Manual tracking — no bank connection yet."**
- Sample rows carry `is_sample=true` + a visible SAMPLE badge; when the sample toggle hides samples, all actuals, projections, and totals exclude them.
- Nothing ever presents as a live bank feed; all figures are labeled as manual entries or sample data.

---

### 161. Compute actuals per category from this month's manually entered transactions only.

- **Components:** JS query `actualsByCategory` (runs on Budgets screen mount); temp state `actuals` (object keyed by category).
- **Data:** `SELECT category, SUM(ABS(amount)) AS spent FROM transactions WHERE amount < 0 AND date >= {{monthStart}} AND date < {{monthEnd}} [AND is_sample = false when samples hidden] GROUP BY category`. Income category uses `amount > 0` and no ABS.
- **Logic notes:** Month window from a `currentMonth` temp state (`YYYY-MM-01`). Only committed manual rows count; drafts/pending rows are excluded. Categories with no transactions get spent = 0.
- **Acceptance:** (1) Bar view matches `SUM(ABS(amount))` of this month's manual spend per category. (2) Future-dated or last-month transactions never appear. (3) Toggling sample visibility recomputes totals within one render cycle.

### 162. Render per-category budget vs actual bars with spent portion filled.

- **Components:** ListView `budgetBars` bound to a merged budget+actual array; inside each row a Progress component (value = spent/limit × 100) with the spent segment filled, plus Text labels for category, "$X of $Y".
- **Data:** Transformer `budgetBarRows` merges `budgets` rows for `currentMonth` with `actuals` from item 161; rows sorted by spent/limit descending.
- **Logic notes:** Bar max = limit; overflow (spent > limit) renders the fill at 100% with an overflow treatment defined in item 176. Categories with a budget but no spend show an empty bar; categories with spend but no budget show an "unset" row (item 145 cross-ref).
- **Acceptance:** (1) Every budgeted category renders one bar. (2) Fill proportion equals spent ÷ limit. (3) Empty-spend categories show 0% fill, not an error.

### 163. Color bars green under 80%, amber 80–100%, red over 100% of budget.

- **Components:** Progress `barColor` dynamic style on `budgetBars` rows; a legend Text row (green/amber/red swatches with labels).
- **Data:** Reuses `budgetBarRows`; JS helper `barColor(pct)`: `pct < 0.8 → '#22c55e'`, `< 1.0 → '#f59e0b'`, else `'#ef4444'`.
- **Logic notes:** Thresholds apply to spend categories; Income category bars use a neutral blue so "over 100%" of an income goal reads as good, not red. Colors are never the sole indicator — each bar also shows the text percent.
- **Acceptance:** (1) 79% renders green, 80% amber, 100.1% red. (2) Legend is visible on the screen. (3) Income row never renders red for exceeding target.

### 164. Show remaining amount ("$42 left") on each category bar.

- **Components:** Text `remainingLabel` inside each `budgetBars` row, right-aligned.
- **Data:** `remaining = limit - spent`; formatted via `{{ '$' + Math.abs(remaining).toFixed(2) }}` with "left" or "over" suffix.
- **Logic notes:** Positive → "$42 left"; negative → "$18 over" (styled by item 176). Zero → "On budget". Rounding to cents; values under $0.005 display as $0.00.
- **Acceptance:** (1) Label matches limit − spent exactly. (2) Over-budget rows show "over" wording, never "left". (3) Updates instantly when actuals recompute.

### 165. Exclude sample transactions from actuals when the sample toggle hides them.

- **Components:** Reuses the global `hideSamples` boolean temp state and sample-toggle chip.
- **Data:** The `actualsByCategory` query appends `AND is_sample = false` when `hideSamples` is true; transformer re-runs on toggle change.
- **Logic notes:** Default state hides samples (item 346 cross-ref). The query must not permanently filter — toggling back restores sample rows. All downstream bars, projections, and exports consume the same filtered actuals object.
- **Acceptance:** (1) With samples hidden, a sample-only category shows $0 spent. (2) Toggling visibility recomputes every bar. (3) No screen shows sample-inflated actuals while the toggle is off.

### 166. Add a drill-down from any bar to the filtered transactions behind it.

- **Components:** Event handler on `budgetBars` row click → `utils.openUrl` internal navigation or `setTab` to Transactions with `drillFilter` temp state `{category, monthStart, monthEnd}`; Transactions table applies it on mount.
- **Data:** No new query; Transactions screen's existing query adds `AND category = {{drillFilter.category}}` and the month window when `drillFilter` is set; a filter chip shows "From budget bar: Food ×" to clear it.
- **Logic notes:** Drill-down preserves the sample-visibility setting so the listed rows sum to the bar's actual. Clearing the chip restores the user's previous filter set.
- **Acceptance:** (1) Tapping a bar opens Transactions filtered to that category and month. (2) The visible rows' total equals the bar's spent figure. (3) Clearing returns to the prior filter state.

### 167. Show a projected month-end total per category based on daily run rate.

- **Components:** Text `projectionLabel` under each bar: "Pace: $X by month end".
- **Data:** Transformer: `dailyRate = spent / daysElapsed`; `projected = dailyRate × daysInMonth`; `daysElapsed` counts from the 1st through today (min 1 to avoid divide-by-zero on the 1st).
- **Logic notes:** Labeled as an estimate from manual entries, never a guarantee. On the 1st, projected = today's spend × days in month (documented as low-confidence). Categories with zero spend project $0.
- **Acceptance:** (1) Projection equals spent ÷ elapsed days × days in month. (2) On the 1st of the month no division error occurs. (3) Label includes the word "Pace" or "projected" so it is not mistaken for actuals.

### 168. Display an overall budget vs actual summary bar for the whole month.

- **Components:** A hero Progress `monthSummaryBar` pinned above the per-category list; Text "Total: $X of $Y • Z%".
- **Data:** Transformer sums limits across budgeted spend categories and sums spent across the same set (Income excluded).
- **Logic notes:** Uses the same color thresholds as item 163. Unset-budget categories contribute spend to neither numerator limit nor denominator — the summary reflects budgeted categories only, with a footnote "N categories without budgets excluded".
- **Acceptance:** (1) Totals equal the sum of the per-category bars shown. (2) Income excluded from both sides. (3) Footnote appears when unset categories exist.

### 169. Add a toggle between absolute dollars and percentage-of-budget views.

- **Components:** SegmentedControl `barViewMode` ("$" / "%") above the list; bar labels switch format.
- **Data:** No query change; transformer adds `pct = spent / limit × 100` per row.
- **Logic notes:** In % mode, bars normalize so 100% = full width regardless of dollar size; remaining label shows "21% left". Mode persists per session in temp state (default $). Over-budget rows show ">100%".
- **Acceptance:** (1) Switching modes re-renders labels without re-querying. (2) % mode shows exact percentages to one decimal. (3) Mode choice survives tab switches within the session.

### 170. Show last month's actuals as a faint comparison marker on each bar.

- **Components:** A thin marker line (a 2px-wide Container positioned at `lastMonthSpent / limit × 100`% width) overlaid on each bar; tooltip Text "Last month: $X".
- **Data:** Query `lastMonthActuals` — same shape as item 161 with the month window shifted back one month; joined into `budgetBarRows` as `prevSpent`.
- **Logic notes:** Marker clamps at 100% width if last month exceeded this month's limit. If last month had no data, no marker renders (no "0" marker confusion). Marker color: neutral gray, always labeled.
- **Acceptance:** (1) Marker position equals last month's spend ÷ this month's limit. (2) No marker when last month is empty. (3) Tooltip shows the exact last-month dollar figure.

### 171. Highlight categories projected to exceed budget before month end.

- **Components:** Warning icon + "Over pace" badge on rows where `projected > limit`; an optional filter chip "Over pace only".
- **Data:** Reuses the projection from item 167; transformer adds `overPace = projected > limit`.
- **Logic notes:** Only flags spend categories with a set budget. Badge appears once projected exceeds limit by any amount; no badge when spent is already over (that uses the red bar state instead). Copy stays neutral: "Over pace", not alarmist.
- **Acceptance:** (1) Every row with projected > limit shows the badge. (2) No badge on rows already over budget (red bar covers it). (3) The "Over pace only" filter isolates exactly those rows.

### 172. Add a "days left in month" indicator next to the budget bars.

- **Components:** Text `daysLeftChip` in the screen header: "12 days left in October".
- **Data:** Pure JS: `new Date(y, m+1, 0).getDate() - today.getDate()`.
- **Logic notes:** Counts calendar days remaining including today. On the last day shows "Last day of {month}". Feeds the projection math contextually but is display-only.
- **Acceptance:** (1) Value equals calendar days remaining. (2) Correct on month boundaries (e.g., Oct 31 → "Last day"). (3) Visible without scrolling on iPhone.

### 173. Render the bars in a vertically scrollable iPhone-friendly list.

- **Components:** The `budgetBars` ListView inside a scrollable Container; each row ≥ 64px tall with full-width tap target; sticky header with summary bar + days-left chip.
- **Data:** No data change.
- **Logic notes:** Layout: category name + remaining on line one, bar on line two, projection/pace on line three. Minimum 44px touch targets; no horizontal scrolling at 390px width. Desktop renders the same list in a max-width 720px column.
- **Acceptance:** (1) No horizontal scroll at 390px. (2) All rows reachable by vertical scroll. (3) Header summary stays visible while scrolling (sticky).

### 174. Show income categories separately from spending categories on the bars view.

- **Components:** Two section headers inside the list: "Spending" and "Income"; Income rows use blue bars with "received of target" wording.
- **Data:** Transformer splits rows on `category === 'Income'`; Income actuals use `SUM(amount)` for `amount > 0` (no ABS).
- **Logic notes:** Income budgets are optional targets; an Income row without a target shows "no target set" instead of a bar. Income never triggers red/over states for exceeding target — exceeding is good.
- **Acceptance:** (1) Income rows render under their own header. (2) Exceeding an income target shows blue/green, never red. (3) Income excluded from the spending summary bar (item 168).

### 175. Add a confetti-free "under budget" state for categories finishing green.

- **Components:** On a category row, when the month is closed or the user marks review complete and `spent < limit`, show a subtle check icon + "Under budget" text in green.
- **Data:** `underBudget = spent < limit`; evaluated at month-end review time or when viewing a past month.
- **Logic notes:** Deliberately no confetti, animation, or sound — a calm check mark only, per the item's explicit constraint. Only applies to closed/past months to avoid premature celebration mid-month.
- **Acceptance:** (1) Past months under budget show the check + label. (2) Current month never shows it (uses live bar states). (3) No animation or confetti assets anywhere in the flow.

### 176. Display the exact overspend amount in red for categories over budget.

- **Components:** Text `overspendLabel` ("$18.40 over") in `#ef4444`, semibold, on rows where `spent > limit`.
- **Data:** `overspend = spent - limit`, formatted to cents.
- **Logic notes:** Replaces the "remaining" label on over-budget rows (never shows both). Bar fill caps at 100% width; the red number carries the magnitude. Screen-reader label includes the dollar figure, not color alone.
- **Acceptance:** (1) Label equals spent − limit to the cent. (2) Rendered in red semibold type. (3) Bar itself never extends past 100% width.

### 177. Allow tapping a bar to edit that category's budget inline.

- **Components:** Row tap (separate from the drill-down affordance — tap bar background = edit; tap a "transactions" icon = drill-down) opens a small Modal `editBudgetModal` with a NumberInput prefilled with the current limit and Save/Cancel.
- **Data:** `UPDATE budgets SET limit_amount = {{newLimit}} WHERE category AND month`; success triggers `actualsByCategory` + `budgetBarRows` refresh and writes an audit-log row (cross-ref item 148).
- **Logic notes:** The two tap targets must be visually distinct to avoid mis-taps: the bar area edits, a chevron/arrow icon drills down. Validation: positive number required.
- **Acceptance:** (1) Editing updates the bar immediately. (2) Drill-down still reachable via its own icon. (3) Invalid (zero/negative) input is rejected with a message.

### 178. Show a week-by-week spend pace line under each category bar.

- **Components:** A 4–5 segment mini bar (tiny Containers) under each category bar, one segment per week of the month, filled proportionally to that week's spend.
- **Data:** Query `weeklySpendByCategory`: `SELECT category, weekOfMonth(date) AS wk, SUM(ABS(amount))` grouped; transformer normalizes each week's spend against the max week for that category.
- **Logic notes:** Weeks defined Mon–Sun; partial first/last weeks included as their own segments. Segments show exact dollars on hover/tap tooltip. Gives the "pace" story behind the projection in item 167.
- **Acceptance:** (1) Segment widths are proportional to weekly spend. (2) Tooltip shows the week's exact total. (3) Renders for all categories with any spend.

### 179. Add a CSV export of budget vs actual for the current month.

- **Components:** Button `exportBudgetCSV` in the Budgets header; uses `utils.downloadFile` with a generated CSV string.
- **Data:** Transformer builds rows: `category, budget_limit, actual_spent, remaining, pct_used, projected, is_sample_included`; filename `financeos-budget-vs-actual-YYYY-MM.csv`.
- **Logic notes:** Export carries the manual-tracking stamp: first line comment or a `data_source` column ("manual"), plus a `SAMPLE` column value per row when samples are included. Honors the current sample-toggle state. UTF-8 with BOM for spreadsheet apps.
- **Acceptance:** (1) CSV columns match the on-screen figures exactly. (2) Includes the sample-data indicator column. (3) Opens correctly in Excel/Sheets with UTF-8 encoding.

### 180. Recompute actuals instantly when a transaction is added, edited, or deleted.

- **Components:** Success handlers on the add/edit/delete transaction queries call `actualsByCategory.trigger()` and `budgetBarRows` re-evaluates; no page reload.
- **Data:** Same query as item 161, re-triggered on every transaction mutation.
- **Logic notes:** Also re-trigger `lastMonthActuals` and `weeklySpendByCategory` so markers and pace lines stay consistent. Debounce rapid successive mutations (e.g., bulk delete) with a 300ms trailing trigger to avoid query storms.
- **Acceptance:** (1) Adding a transaction updates the matching bar within one second, no reload. (2) Deleting the only transaction in a category returns its bar to 0%. (3) Bulk operations trigger a single recompute, not one per row.
