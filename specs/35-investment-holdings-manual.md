# FinanceOS — Builder Spec: Investment holdings (manual) (items 681–700)

## Section title
Investment holdings (manual) — manually-entered portfolio tracking for Investment-type accounts: holdings, user-entered prices, allocation, lots, dividends, and stale-price warnings. NO live quotes, NO broker connection, NO automatic price feeds of any kind.

## Tables touched
Existing:
- `accounts`: id (pk), name (text), type (text: Checking / Savings / Credit / Investment), institution (text), balance (number), updated_at (timestamp), is_sample (bool)
- `transactions`: id (pk), account_id (fk), date (date), description (text), category (text), amount (number, negative=spend), is_sample (bool)
- `goals`: id (pk), goal_name (text), target (number), saved (number), deadline (date)

New:
- `holdings`: id (pk), account_id (fk → accounts.id), symbol (text, e.g. "AAPL"), name (text, e.g. "Apple Inc."), shares (number), cost_basis_per_share (number), asset_class (text: stocks / bonds / crypto / other), notes (text, user thesis), goal_id (fk → goals.id, nullable), is_cash_position (bool, default false), is_sample (bool), created_at (timestamp)
- `lots`: id (pk), holding_id (fk → holdings.id), shares (number), cost_basis_per_share (number), acquired_date (date), is_sample (bool) — individual tax lots when per-lot tracking is on
- `price_snapshots`: id (pk), holding_id (fk → holdings.id), price (number), as_of_date (date), is_sample (bool), entered_at (timestamp) — every manual price update; the latest row per holding is the "last manual price"

## Cross-cutting rules (apply to every item below)
1. Every screen shows the persistent banner: "Manual tracking — no bank connection yet."
2. All sample rows carry `is_sample=true` and a visible SAMPLE badge; the sample-data toggle excludes them from totals when off.
3. Nothing ever presents as a live bank feed or live market data. All prices carry the label "User-entered" and an as-of date. No ticker widgets, no auto-refresh, no implied real-time quotes.
4. Prices are typed in by the user only — buys, sells, dividends, and price updates are all manual records the user confirms.

---

### 681. Support Investment-type accounts with holdings: symbol, shares, cost basis, notes.

**Components:** Accounts screen "New account" form adds type option "Investment"; holdings section on the account detail page — Table `tblHoldings` (columns: symbol, name, shares, cost basis, asset class, notes icon); Button "Add holding" opens Modal `mdlHolding` with Text Inputs (symbol, name, shares, cost_basis_per_share), Select `selAssetClass` (stocks/bonds/crypto/other), Text Area `notes`, and a toggle for cash position.

**Data:** Queries `createAccount` (INSERT accounts with type='Investment'), `createHolding` (INSERT holdings), `getHoldings` (SELECT * FROM holdings WHERE account_id = {{selectedAccount.id}}). Symbol uppercased via JS on submit.

**Logic notes:** Cash positions store `is_cash_position=true`, shares=1, cost_basis_per_share = dollar amount. Symbol validated non-empty; shares and cost basis > 0.

**Acceptance:** (1) Creating an Investment-type account adds it to the Accounts list with correct type label. (2) Adding a holding persists symbol, shares, cost basis, asset class, and notes. (3) A cash position records correctly and displays as cash. (4) Sample toggle hides sample holdings from the table.

### 682. Allow manual price updates per holding with an as-of date.

**Components:** Button "Update price" per holdings-table row opens Modal `mdlPrice` with Number Input `numPrice`, Date Input `dtAsOf` (defaults to today), and a Text note "Price is user-entered, never a live quote."

**Data:** Query `recordPrice` — INSERT INTO price_snapshots (holding_id, price, as_of_date). Transformer `latestPrice` picks MAX(as_of_date) per holding_id. Success toast confirms the update.

**Logic notes:** Price must be > 0; as_of_date cannot be in the future. If the holding has multiple entries for the same date, the newest entered_at wins.

**Acceptance:** (1) Submitting a valid price and date inserts a price_snapshots row. (2) Future dates are rejected with an error message. (3) Zero/negative prices are rejected. (4) The holdings table shows the new last manual price immediately after update.

### 683. Compute holding value as shares × last manual price.

**Components:** Holdings table derived columns `value` and `totalPortfolioValue` KPI at the top of the Investments screen.

**Data:** Transformer `holdingValues`: join holdings → latest price_snapshots row → value = shares × price. Query `getHoldingsWithPrices` feeds the table; `sumValues` reducer for the KPI.

**Logic notes:** Holdings with no price snapshot show "—" instead of a value and are excluded from the portfolio total. Cash positions use cost_basis_per_share as their price fallback.

**Acceptance:** (1) Value column equals shares × last manual price for every row with a price. (2) Holdings without a price show "—", not 0. (3) Portfolio total matches the sum of row values. (4) Updating a price changes the row value and total on refresh.

### 684. Show portfolio allocation as a donut chart from manual entries.

**Components:** Chart component `chtAllocation` (type: donut) on the Investments screen, grouped by asset_class, with legend and per-slice percent labels.

**Data:** Transformer `allocationData` groups `holdingValues` by asset_class, summing value per class. Cash positions roll into a "cash" slice.

**Logic notes:** Slices sized by market value (shares × last price), not cost basis. Empty portfolio shows a placeholder message ("Add a holding and price to see allocation") instead of a broken chart.

**Acceptance:** (1) Slice sizes match computed per-class values ÷ portfolio total. (2) Labels show asset class name and percent. (3) Empty state renders a friendly message, not an error. (4) All values derive from manual entries; no external price source is queried.

### 685. Display unrealized gain/loss per holding from cost basis vs manual price.

**Components:** Holdings table columns `gainLoss` (currency) and `gainLossPct` (percent), colored green/red. Summary row shows total unrealized gain/loss.

**Data:** Transformer `gainLoss`: per holding, total_cost = shares × cost_basis_per_share; gain = value − total_cost; pct = gain / total_cost. For multi-lot holdings, cost = Σ lot shares × lot cost basis.

**Logic notes:** Cost basis is the weighted-average user-entered purchase price, not a live value. Loss rows render red, gains green, zero neutral.

**Acceptance:** (1) Gain/loss equals (last price − cost basis) × shares per row. (2) Percent equals gain ÷ total cost. (3) Negative values render in red with a minus sign. (4) Table total gain/loss equals the sum of row values.

### 686. Record buys and sells as manual transactions linked to the holding.

**Components:** Buttons "Record buy" / "Record sell" on each holding row open Modal `mdlTrade` with Date, Shares (number), Price per share, and optional note. The modal shows the resulting transaction preview before confirm.

**Data:** Queries `recordTrade`: INSERT transactions (account_id=holding's account, date, description="Buy 10 AAPL @ 150", category="Other" or a dedicated "Transfer" category, amount=∓(shares×price)); UPDATE holdings SET shares = shares ± qty. If lot tracking is enabled, INSERT/UPDATE `lots` rows (sells consume oldest lots first).

**Logic notes:** Selling more shares than held is rejected. Buys increase shares and reweight the average cost basis; sells reduce shares but leave per-share cost basis unchanged for remaining shares.

**Acceptance:** (1) A buy creates a transaction record and increases the holding's shares. (2) A sell decreases shares and creates the matching transaction. (3) Overselling is blocked with an error. (4) The transaction description identifies the holding symbol.

### 687. Show dividend entries as Income transactions tagged to the holding.

**Components:** Button "Record dividend" on each holding row opens Modal `mdlDividend` with Date, Amount, and Note. Dividends list section per holding (mini-table of dividend records).

**Data:** Query `recordDividend`: INSERT transactions with category="Income", description="Dividend — {symbol}", amount=positive; a `holding_id` text reference in the description or an optional fk column. Transformer `dividendsByHolding` aggregates total dividends per holding.

**Logic notes:** Dividends never change shares or cost basis. They count as income in month-end reports and are excluded from unrealized gain/loss math.

**Acceptance:** (1) A dividend creates an Income-category transaction linked to the holding. (2) The holding's per-holding dividend list shows the new entry. (3) Shares and cost basis are unchanged by dividends. (4) Total dividends per holding equal the sum of its dividend records.

### 688. Add a holdings table with symbol, shares, price, value, gain/loss.

**Components:** Table `tblHoldings` on the Investments screen with columns: symbol, name, shares, last price (+ as-of date sub-line), value, gain/loss, gain/loss %. Row selection drives the detail panel below (notes, lots, dividends).

**Data:** Single transformer `holdingsTableData` joining holdings + latest price + computed value and gain/loss. Sortable by value desc default.

**Logic notes:** All derived columns come from `holdingValues` / `gainLoss` transformers — no duplicate formulas. Price cells show "User-entered · {as_of_date}" tooltip.

**Acceptance:** (1) Every required column renders for every holding. (2) Sorting by value works ascending and descending. (3) Selecting a row loads its detail panel. (4) Rows without prices show "—" in price/value/gain-loss columns.

### 689. Support cash positions inside investment accounts.

**Components:** Holdings form includes a "This is a cash position" toggle; cash rows render with a cash icon and no gain/loss columns (show "—").

**Data:** `holdings` rows with is_cash_position=true. Price fallback = cost_basis_per_share. Value = shares(1) × price. Excluded from allocation chart's security slices (own "cash" slice) and from gain/loss math.

**Logic notes:** Cash positions never show unrealized gain/loss and never trigger stale-price badges. Multiple cash positions per account allowed (e.g., USD cash + sweep).

**Acceptance:** (1) Creating a cash position records is_cash_position=true. (2) Cash rows show no gain/loss. (3) Cash appears as its own slice in the allocation chart. (4) Cash values count toward portfolio total.

### 690. Display portfolio value history from manual price snapshots.

**Components:** Line chart `chtPortfolioHistory` on the Investments screen; Date range selector (3M / 1Y / All).

**Data:** Transformer `portfolioHistory`: for each snapshot date, sum shares-held × price for all holdings (shares as of that date approximated from current shares; note the approximation). Cash positions included at face value.

**Logic notes:** Only dates with at least one price snapshot are plotted; gaps are not interpolated. X-axis shows as-of dates; points labeled "user-entered prices."

**Acceptance:** (1) Chart points equal the sum of per-holding values at each snapshot date. (2) Date-range selector filters correctly. (3) Empty history shows a placeholder, not an error. (4) No live market data is fetched or implied.

### 691. Label all prices as user-entered; never imply live market data.

**Components:** A shared Text component / caption style "Entered by you · {as_of_date}" placed under every price field, chart title, and value KPI. Tooltip on price columns: "Manually entered — not a live quote."

**Data:** No new queries; reuse as_of_date from price_snapshots and entered_at timestamps.

**Logic notes:** The words "live", "real-time", "quote", or ticker-style flashing prices must not appear anywhere in the Investments screens. Design review checklist item.

**Acceptance:** (1) Every displayed price shows a "user-entered" label with as-of date. (2) No component implies real-time or live data. (3) The Investments screen header carries the manual-tracking banner. (4) QA pass finds zero "live quote" language in the section.

### 692. Add investment goal linking (e.g., "retirement" goal fed by portfolio value).

**Components:** Holdings form / holding detail includes Select `selGoal` (options from goals table, plus "None"). Goals screen shows a "Linked portfolio value" line on linked goals.

**Data:** `holdings.goal_id` fk; query `updateHoldingGoal`. Transformer `goalPortfolioValue`: sum of holding values per goal_id. Goals screen optionally offers "Use portfolio value as saved amount" — user-confirmed button that writes the computed value into goals.saved.

**Logic notes:** Linking is informational by default; overwriting goals.saved requires an explicit user tap with a confirm dialog. Unlinking clears goal_id.

**Acceptance:** (1) Selecting a goal persists goal_id on the holding. (2) The goal shows the linked portfolio value. (3) "Use as saved amount" updates goals.saved only after confirmation. (4) Unlinking removes the linkage without deleting data.

### 693. Show asset-class tags per holding (stocks, bonds, crypto, other) user-assigned.

**Components:** Select `selAssetClass` in the holding form (stocks / bonds / crypto / other); Tag/badge in the holdings table asset-class column; Filter dropdown on the Investments screen to filter holdings by class.

**Data:** `holdings.asset_class` text column. Query `updateAssetClass`. Filter applied in `holdingsTableData` transformer.

**Logic notes:** Asset class is free user choice; no validation against symbol. "Other" is the default for new holdings.

**Acceptance:** (1) Asset class saves and displays as a tag. (2) Class filter shows only matching holdings. (3) Allocation chart groups by the same classes. (4) Changing a class updates the chart on refresh.

### 694. Support multiple lots per symbol with individual cost bases.

**Components:** Holding detail panel shows a lots sub-table (acquired date, shares, cost basis per share, value at cost). Toggle "Track lots" per holding; "Add lot" button opens a lot entry modal.

**Data:** `lots` table (holding_id, shares, cost_basis_per_share, acquired_date). When enabled, holding-level shares = Σ lot shares; cost basis = weighted average. Sells consume oldest lots first (FIFO), updating lot shares.

**Logic notes:** Enabling lot tracking on an existing holding converts its current position into one lot. Disabling collapses lots back into the weighted-average holding row.

**Acceptance:** (1) Adding a lot stores shares, cost basis, and date. (2) Holding shares equal the sum of lot shares. (3) A sell reduces the oldest lot first. (4) Gain/loss uses per-lot cost bases correctly.

### 695. Display a rebalancing view: current vs target allocation percents.

**Components:** Investments screen "Rebalance" tab: table of asset classes with current % (from manual prices), Number inputs for target %, and a "difference" column; a horizontal bar visual comparing current vs target.

**Data:** Transformer `rebalanceData`: current % per class from `allocationData`; targets stored in localStorage or a `rebalance_targets` table (class, target_pct). Difference = target − current.

**Logic notes:** Targets must sum to 100%; validation warns otherwise. The view suggests nothing actionable beyond "buy/sell" amounts — it never executes anything (manual-entry only).

**Acceptance:** (1) Current % matches the allocation chart. (2) Target inputs persist between sessions. (3) Non-100% targets trigger a validation warning. (4) Difference column equals target − current.

### 696. Add contribution tracking: total invested vs current value.

**Components:** KPI cards on the Investments screen: "Total invested" (cost basis sum), "Current value", "Net gain/loss".

**Data:** Transformer `contributionSummary`: total_invested = Σ (shares × cost_basis) across holdings (+ lots when enabled); current_value from `holdingValues`. Net = current − invested.

**Logic notes:** Dividends and sells are excluded from "total invested" (sells reduce both sides). Cash positions count at face value in both.

**Acceptance:** (1) Total invested equals Σ shares × cost basis. (2) Current value matches the portfolio total. (3) Net gain/loss equals current − invested. (4) Recording a buy changes total invested accordingly.

### 697. Show a "prices stale" badge when no manual update in 30 days.

**Components:** Badge "Stale price" (amber) on holding rows and a summary banner "N holdings have prices older than 30 days" on the Investments screen with an "Update prices" shortcut.

**Data:** Transformer `staleCheck`: days_since = today − MAX(as_of_date) per holding; stale if > 30 or no snapshot at all.

**Logic notes:** Cash positions are exempt. The badge is informational only — no blocking behavior.

**Acceptance:** (1) Holdings with prices older than 30 days show the badge. (2) Holdings with no price snapshot show the badge. (3) Updating a price clears the badge. (4) Cash positions never show the badge.

### 698. Export holdings to CSV with cost basis and last price.

**Components:** Button "Export holdings CSV" on the Investments screen. Generates and downloads a CSV file.

**Data:** Transformer `holdingsExport`: rows of symbol, name, shares, cost_basis_per_share, last_price, as_of_date, value, gain_loss, asset_class, notes. JS builds the CSV blob; export header includes the manual-tracking banner text and export date.

**Logic notes:** Sample rows included only when the sample toggle is on (flag them in a column). File name: `financeos-holdings-YYYY-MM-DD.csv`.

**Acceptance:** (1) Clicking export downloads a CSV file. (2) CSV contains all required columns including cost basis and last price. (3) Sample-row handling matches the toggle state. (4) The file header stamps the export as manual data with the date.

### 699. Add investment notes per holding ("thesis", "why I bought").

**Components:** Text Area `notes` in the holding form and an editable notes panel in the holding detail view. Notes render as plain text with a "last edited" timestamp.

**Data:** `holdings.notes` text column; query `updateHoldingNotes`. No rich text — plain text only.

**Logic notes:** Notes are private to the app; no sharing. Character limit 2000.

**Acceptance:** (1) Notes save and persist on the holding. (2) Notes display in the detail panel. (3) Editing notes updates the last-edited indicator. (4) Empty notes show a "Add your thesis…" placeholder.

### 700. Exclude investment volatility from budget actuals; show it only in net worth.

**Components:** No new UI — a documented rule enforced in the budget actuals transformer.

**Data:** Budget actuals transformer filters transactions to categories/spend only; investment buy/sell/dividend transactions (identified by category "Transfer" or description prefix "Buy "/"Sell "/"Dividend —") are excluded from budget actuals. Net worth transformer includes investment account balances (from `holdingValues` totals) in the net-worth sum.

**Logic notes:** Dividends appear as income in month reports but not as budget "actuals" for spending categories. Document the exclusion in the budget screen's info tooltip.

**Acceptance:** (1) Buy/sell transactions do not change any budget actual. (2) Investment account value is included in the net-worth total. (3) The budget info tooltip explains the exclusion. (4) Removing the exclusion filter in a test query changes actuals, proving the filter is what excludes them.

---

## Verification
- Every item 681–700 has a `### N.` subsection above (grep-verified).
- All subsections use manual-entry data only; no live quotes, no broker feeds.
- Evidence file: `.evidence/35.txt` (one line per item).
