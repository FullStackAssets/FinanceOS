# FinanceOS — Section 40: Multi-currency support (items 781–800)

Builder spec for a Retool developer implementing multi-currency handling in the FinanceOS app (App 11). All amounts remain manual entry; exchange rates are user-entered only — no live FX feed anywhere in this section unless item 798's explicit opt-in is built separately (this spec keeps manual entry as the default and only path).

## Tables touched

Existing tables extended: `accounts` gains `currency_code` (text, ISO 4217, default from settings); `transactions` gains `native_amount` (number, amount in the account's currency) and `currency_code` (text, denormalized for display).

New tables (full schema):
- `currencies`: id (pk), code (text, unique, e.g. "USD"), symbol (text, e.g. "$"), symbol_position (text: Before / After), decimal_places (number, default 2), locale (text, e.g. "en-US"), is_sample (bool)
- `exchange_rates`: id (pk), from_code (text), to_code (text), rate (number, multiply from→to), as_of (date, user-entered), note (text), is_sample (bool), updated_at (timestamp)
- `fx_audit`: id (pk), action (text: Create / Update / Delete), rate_id (number), detail (text), acted_at (timestamp), is_sample (bool)

## Shared cross-cutting rules

- Every screen shows the persistent banner: "Manual tracking — no bank connection yet."
- All sample rows carry `is_sample=true` and a visible SAMPLE badge; they are excluded from totals when the sample-data toggle hides them.
- Nothing ever presents as a live bank feed, and nothing in this section presents rates as live — every converted figure is labeled "estimate from user-entered rates."
- Currency symbols render from the `currencies` table, never hard-coded per account.

### 781. Set a home currency in settings used for all totals.

**Components:** Settings page container with a `currencySettingsForm`; `select` (homeCurrencySelect, options from `currencies` query); `text` helper explaining "All net-worth and report totals convert to this currency"; `button` (saveHomeCurrency, primary). A `text` caption under the select shows the current saved value.

**Data:** `getCurrencies` (SELECT code, symbol FROM currencies ORDER BY code); `getSettings` returns the stored `home_currency` value; `saveHomeCurrency` UPDATE on the settings store with the selected code. A `formatHomeTotals` transformer applies the code to all hero totals.

**Logic notes:** Home currency defaults to USD on first run. Changing it re-runs every conversion query on the overview and reports — no stale cached totals. Conversion only runs through `exchange_rates` rows with an as_of date on or before today.

**Acceptance:** (1) Selecting EUR and saving flips all overview totals to EUR with the € symbol; (2) persistence holds across reload; (3) USD remains the default for a fresh settings store; (4) every converted total carries the "estimate from user-entered rates" label.

### 782. Allow per-account currency selection (USD, EUR, GBP, etc.).

**Components:** On the Accounts screen, an `select` (accountCurrencySelect) inside both the add-account and edit-account forms, populated from `getCurrencies`; a currency pill `text` on each account card showing code + symbol. The update-balance form keeps its numeric input unchanged.

**Data:** `getCurrencies` shared query; `addAccount` / `updateAccount` INSERT/UPDATE the new `accounts.currency_code` column. Backfill migration: existing accounts get the settings home currency as their code.

**Logic notes:** Currency is set per account, not per transaction — transactions inherit the account's code at entry time. Changing an account's currency does not rewrite history; it only affects new transactions, and a confirm modal states this explicitly.

**Acceptance:** (1) New account saved with GBP shows a GBP pill and £-formatted balance; (2) editing currency shows the no-history-rewrite warning and requires confirm; (3) account cards display code + symbol for every account; (4) legacy accounts backfill to the home currency, never NULL.

### 783. Record transactions in the account's currency.

**Components:** Add-transaction form gains a read-only `text` (txnCurrencyLabel) showing the selected account's currency code + symbol, bound to `accountSelect.value`; the amount `numberInput` keeps its label "Amount" with the currency appended dynamically ("Amount (EUR)").

**Data:** On account select, `getAccountCurrency` returns `currency_code` for the chosen account; `addTransaction` INSERTs `native_amount` = entered amount and `currency_code` = account code. Legacy `amount` column stays populated in parallel for backward compatibility.

**Logic notes:** Amount is always interpreted in the account's currency — there is no per-transaction currency picker in this build (that arrives via item 792's quick-add parser). Mixed-currency accounts are impossible by construction since currency lives on the account.

**Acceptance:** (1) Adding €45 to a EUR account stores native_amount=45, currency_code=EUR; (2) the form label shows "Amount (EUR)" when a EUR account is selected; (3) `amount` mirrors `native_amount` for existing reports; (4) no transaction can be saved without a resolved currency code.

### 784. Support manual exchange-rate entries with as-of dates.

**Components:** "Exchange rates" admin panel: `table` (ratesTable: from, to, rate, as_of, note); `form` with `select` fromCode, `select` toCode, `numberInput` rateInput (step 0.0001), `datePicker` asOfDate, `textInput` rateNote; `button` saveRate (primary) and per-row edit/delete `button`s.

**Data:** `getRates` (SELECT from exchange_rates ORDER BY as_of DESC); `saveRate` INSERT/UPDATE; `deleteRate` with confirm modal. Validation transformer rejects rate ≤ 0, from_code = to_code, and future as_of dates.

**Logic notes:** Rates are directional pairs (EUR→USD stored separately from USD→EUR); the save form offers a "also create inverse" checkbox that inserts the reciprocal. Duplicate (from, to, as_of) entries are blocked — editing updates the existing row.

**Acceptance:** (1) Saving 1 EUR = 1.08 USD with as-of 2026-10-01 creates the row; (2) zero/negative rates and same-currency pairs are rejected with an inline error; (3) the inverse checkbox creates the reciprocal row automatically; (4) editing an existing pair/date updates rather than duplicating.

### 785. Convert foreign balances to home currency for net worth using the latest manual rate.

**Components:** Overview hero `statistic` (netWorthHome) showing the converted total; a `text` subcaption "Converted with your rates as of {date}"; the existing native-currency balances remain on account cards unchanged.

**Data:** `getHomeNetWorth` transformer: for each account, find the latest `exchange_rates` row where from_code = account currency, to_code = home currency, as_of ≤ today; multiply balance × rate; sum. If no rate exists for a currency, that account is listed as "unconverted" in a footnote and excluded from the sum.

**Logic notes:** Latest rate means max as_of, not max updated_at. Accounts already in the home currency convert at 1.0. Sample accounts follow the same path but are excluded when the sample toggle is off.

**Acceptance:** (1) A €1,000 account at 1.08 contributes $1,080 to net worth; (2) an account whose currency has no rate shows an "unconverted" footnote and is excluded; (3) changing home currency re-runs the conversion; (4) native balances on account cards never change as a result of conversion.

### 786. Show both native and converted amounts on foreign-currency rows.

**Components:** Accounts table and Transactions table each gain a computed column: `text` cells rendering two lines — native amount (bold) and converted home amount (muted, smaller) via a `dualAmount` transformer. Home-currency rows show only the native line.

**Data:** `dualAmount(row)` transformer: looks up the latest rate for row.currency_code → home currency; returns {native: formatted native, converted: formatted home or null}. Formatting uses the `currencies` table for symbol and decimals.

**Logic notes:** When no rate exists, the converted line reads "no rate" in muted italic instead of a number — never blank, never zero. The converted line always appends the word "est." to reinforce estimate labeling.

**Acceptance:** (1) A GBP account row shows "£500" and below "$635 est."; (2) home-currency rows show a single line; (3) missing rates render "no rate" rather than 0; (4) every converted line includes the "est." marker.

### 787. Label converted figures as estimates from user-entered rates.

**Components:** A shared `text` caption component pattern ("est. from your rates") placed under every converted figure: net-worth hero, dual-amount table cells, report totals, export headers. A single `estimateLabel` JS snippet returns the exact string so wording never drifts.

**Data:** `estimateLabel(asOfDate)` returns `est. from your rate of ${asOfDate}`; consumed by the net-worth caption, `dualAmount`, report transformers, and the export header builder.

**Logic notes:** The label is mandatory wherever a converted number appears — it is part of the component, not optional helper text. If a converted figure is rendered anywhere without the label, that is a defect per this spec.

**Acceptance:** (1) Grep of the app shows no converted figure without the estimate label; (2) wording is identical everywhere via the shared snippet; (3) the label includes the rate's as-of date; (4) native-currency figures never carry the estimate label.

### 788. Add a rates history table with edit and delete.

**Components:** Rates panel `table` (ratesTable) with columns Pair, Rate, As of, Note, Updated; row-level `button` editRate and `button` deleteRate; the save form doubles as the editor (populated on edit); a `modal` (confirmDeleteRate) for deletes.

**Data:** `getRates` ordered by pair then as_of DESC; `updateRate` and `deleteRate` queries; `logFxChange` INSERT into `fx_audit` on every create/update/delete with the before/after values in `detail`.

**Logic notes:** Deleting a rate that is the latest for a pair triggers a warning: "Net worth will fall back to the next-newest rate." Edit preserves the row id so audit history stays linked. Soft-delete is not used; the audit row is the record.

**Acceptance:** (1) Editing a rate updates the row and writes an fx_audit entry; (2) deleting the latest rate shows the fallback warning and requires confirm; (3) the audit log captures old and new values; (4) the table always sorts newest-first within each pair.

### 789. Warn when a conversion uses a rate older than 30 days.

**Components:** A `callout` (staleRateWarning, amber) above any converted total that used a rate with as_of older than 30 days: "Rate EUR→USD is 42 days old — consider updating." The rates table shows an amber dot on stale rows.

**Data:** `rateAgeDays` transformer computes today − as_of per rate used; `getStaleRates` flags rows > 30 days. The warning lists each stale pair with its age.

**Logic notes:** The threshold is fixed at 30 days per the roadmap item (not user-configurable in this build). Staleness is computed at render time so it never goes stale itself. The warning links to the rates panel.

**Acceptance:** (1) A 45-day-old EUR→USD rate triggers the amber callout on the overview; (2) a 10-day-old rate shows no warning; (3) stale rows in the rates table show the amber dot; (4) the callout links directly to the rates panel.

### 790. Support per-trip or per-goal currency grouping for travel funds.

**Components:** On the Goals screen, a `select` (goalCurrencySelect) on the add/edit-goal form; goal cards show a currency pill; a `select` (tripFilter) on the Transactions screen filters by goal-linked travel tags (reuses the tags system) to show "trip spend" grouped in the goal's currency.

**Data:** `goals` gains `currency_code` (default home currency); `saveGoal` persists it; `getTripSpend` transformer sums transactions tagged for the trip and converts each via the latest rate into the goal's currency for the grouped view.

**Logic notes:** This is grouping and display only — it does not move money or create FX transactions. A goal in JPY shows its target, saved, and linked trip spend all in JPY with converted "est." lines where the source currency differs.

**Acceptance:** (1) A "Japan trip" goal in JPY shows target/saved in ¥; (2) linked USD transactions appear converted with "est." lines; (3) the trip filter groups spend by trip tag in the goal currency; (4) goals without a currency default to the home currency.

### 791. Display currency symbols correctly per locale.

**Components:** A shared `formatMoney(amount, code)` JS snippet used by every amount display in the app; it reads symbol, position, and decimals from `getCurrencies` and applies locale grouping via `Intl.NumberFormat(locale)`.

**Data:** `getCurrencies` provides code, symbol, symbol_position, decimal_places, locale. The formatter caches the currency metadata in a temporary state variable to avoid per-cell queries.

**Logic notes:** Symbol position (before/after) and decimals come from the table — e.g. JPY uses 0 decimals, EUR in de-DE renders "1.234,56 €". No hard-coded "$" remains in any amount renderer after this item.

**Acceptance:** (1) €1,234.56 renders correctly for EUR/en-US; (2) JPY amounts show 0 decimals; (3) a symbol-after currency renders the symbol after the number; (4) every amount in the app routes through `formatMoney` — spot-check five screens.

### 792. Allow entering amounts with currency codes in quick-add ("45 EUR").

**Components:** Quick-add `textInput` (quickAddInput) accepting free text like "45 EUR coffee" or "12.50 GBP"; a `text` preview line under the input showing the parsed result ("€45.00 → Groceries?") before confirm; `button` quickAddConfirm.

**Data:** `parseQuickAdd` transformer: regex extracts the leading number and an optional 3-letter code; validates the code against `getCurrencies`; defaults to the selected account's currency when no code is given; returns {amount, code, description}.

**Logic notes:** If the parsed code differs from the selected account's currency, the input is rejected with "Account is in USD — enter 45 USD or pick a USD account" (no auto-conversion at entry). Unparseable input shows the expected format as a hint.

**Acceptance:** (1) "45 EUR coffee" on a EUR account parses to €45 with description "coffee"; (2) a code mismatch with the account currency is rejected with guidance; (3) "20 lunch" defaults to the account currency; (4) an unknown code like "XYZ" shows an inline error.

### 793. Show exchange gain/loss notes on foreign accounts between rate updates.

**Components:** On each foreign-currency account card, a `text` line (fxDeltaNote): "+$12.40 since your last rate update (est.)" in green/red; a `tooltip` explaining the figure compares the converted balance at the previous vs current latest rate.

**Data:** `getFxDelta(account)` transformer: converts the current balance with the newest rate and with the previously-newest rate for the same pair; the difference is the note. Requires ≥ 2 rates for the pair; otherwise the line is hidden.

**Logic notes:** This is an informational note only — it never creates a transaction or adjusts the balance. Sign coloring follows profit/loss convention (green positive, red negative) with a text "+" / "−" so color is not the only signal.

**Acceptance:** (1) After updating EUR→USD from 1.08 to 1.10, a €1,000 account shows "+$20.00 since your last rate update (est.)"; (2) a single-rate pair hides the line; (3) the note never alters balances or creates transactions; (4) the tooltip explains the computation.

### 794. Export with both native and home-currency columns.

**Components:** Export dialog gains a `checkbox` (includeConvertedColumns, default checked); the CSV/PDF builders add "Amount (native)", "Currency", "Amount (home, est.)", and "Rate as-of" columns when checked.

**Data:** The export transformer maps each transaction through `dualAmount`-style conversion, stamping the rate's as_of date per row. Header row of the export includes "Converted amounts are estimates from user-entered rates."

**Logic notes:** Rows with no available rate leave the home column blank with "no rate" in the rate-as-of column — never zero-filled. The manual-tracking banner text is included in the export header per existing export rules.

**Acceptance:** (1) CSV export of a EUR transaction shows native, code, converted, and as-of columns; (2) unrated rows show blank converted + "no rate"; (3) the estimate disclaimer is in the file header; (4) unchecking the box produces the legacy single-amount export.

### 795. Add currency formatting settings (decimals, symbol position).

**Components:** Settings page "Currency display" section: per-currency `table` (currencyFormatTable) with editable `numberInput` decimals and `select` symbol position (Before/After); `button` saveCurrencyFormats; a live `text` preview row showing "1,234.56" rendered in each currency's settings.

**Data:** `getCurrencies` full row; `updateCurrencyFormat` UPDATEs decimal_places and symbol_position per code; the preview re-renders through `formatMoney` on every edit.

**Logic notes:** Decimals clamp to 0–4. Changing formats re-renders every amount app-wide since all rendering flows through `formatMoney`. Formats apply to display only — stored values keep full precision.

**Acceptance:** (1) Setting JPY to 0 decimals renders "¥1,235" for 1234.5; (2) symbol-after renders "1 234,56 €"-style output; (3) decimals outside 0–4 are rejected; (4) stored transaction values are unchanged by format edits.

### 796. Support crypto-denominated manual holdings with user-entered prices.

**Components:** Accounts screen: account type gains "Crypto" option; add/edit-crypto-account form with `select` (cryptoCode: BTC, ETH, + custom via currencies table), `numberInput` holdings amount (8 decimals allowed), and `numberInput` userPrice with as-of `datePicker`.

**Data:** Crypto codes live in `currencies` (e.g. BTC, ₿, 8 decimals); the user-entered price is stored as an `exchange_rates` row (BTC→home currency, as_of = price date, note "manual crypto price"). Balances compute as holdings × latest price.

**Logic notes:** Prices are manual entries like any FX rate — no live price feed, no market-data labeling. The 8-decimal support comes from the per-currency decimals setting (item 795). Converted values carry the same "est." labeling.

**Acceptance:** (1) 0.5 BTC at a user price of $60,000 shows $30,000 est. in net worth; (2) no component implies a live price — all labels say user-entered; (3) holdings support 8 decimals; (4) stale-price warnings (item 789) apply to crypto prices too.

### 797. Display a rates-staleness badge on the overview when rates are old.

**Components:** Overview header `badge` (ratesStaleBadge, amber): "Rates 42 days old" — visible when any rate used in current conversions is > 30 days old; clicking navigates to the rates panel. Hidden entirely when all rates are fresh.

**Data:** Reuses `getStaleRates` from item 789; the badge shows the maximum age among used rates. A `tooltip` lists the stale pairs.

**Logic notes:** The badge reflects only rates actually used in the current view's conversions, not every rate in the table. It updates on every overview render so it can clear itself the moment rates are refreshed.

**Acceptance:** (1) With a 45-day-old rate in use, the amber badge shows in the overview header; (2) updating the rate clears the badge on next render; (3) unused stale rates don't trigger the badge; (4) clicking the badge opens the rates panel.

### 798. Never fetch live FX rates without explicit opt-in; manual entry is the default.

**Components:** A `toggle` (liveFxOptIn, default OFF, disabled state labeled "Coming later") on the rates panel; helper `text`: "FinanceOS uses only rates you enter. Live rates are not available in this version." No network query components are wired in this build.

**Data:** No data work — this item is a deliberate non-feature. A `getFxMode` check returns "manual" always; any future live-rate code must branch on the opt-in toggle, which ships off and inert.

**Logic notes:** The toggle exists as a documented placeholder so a future live-feed feature has an explicit consent gate; it performs no action and triggers no fetch. This satisfies the roadmap item by making manual entry the only operative path and recording the user's explicit stance.

**Acceptance:** (1) The toggle defaults off and does nothing when flipped; (2) helper text states manual-only clearly; (3) no FX network calls exist in the app — verify by inspecting queries; (4) the banner "Manual tracking — no bank connection yet." remains on the rates panel.

### 799. Add a currency converter calculator tool inside the app.

**Components:** A "Converter" utility card (modal or side panel): `numberInput` convertAmount, `select` convertFrom, `select` convertTo, `button` convertBtn, result `text` (convertResult) showing "€92.59 est." plus the rate and as-of date used; a `text` history list of the last 5 conversions (session-only).

**Data:** `runConversion` transformer: looks up the latest `exchange_rates` row for the pair (or triangulates via the home currency when no direct pair exists: from→home→to); returns {result, rateUsed, asOf, triangulated: bool}.

**Logic notes:** Triangulated conversions are labeled "via USD" (or the home code) so the user sees the path. The tool is a calculator only — results have no "apply" button and never write to accounts or transactions.

**Acceptance:** (1) 100 USD → EUR at 0.9259 shows "€92.59 est." with rate and date; (2) a pair with no direct rate triangulates via home and says so; (3) no-rate-at-all shows "no rate available"; (4) the tool cannot modify any account or transaction.

### 800. Log rate changes in the audit log.

**Components:** Reuses the rates panel from item 788 — no new UI. The existing `fx_audit` table backs an "FX history" view: `table` (fxAuditTable: action, pair, old → new, acted_at) accessible from the rates panel via a `button` (viewFxHistory).

**Data:** `logFxChange` INSERTs on every create/update/delete of `exchange_rates` (wired in item 788's save/delete queries); `getFxAudit` SELECTs ordered by acted_at DESC. The shared app audit log (section 38) links to these rows by rate_id.

**Logic notes:** Audit rows are append-only — no edit or delete UI exists for `fx_audit`. The detail field stores human-readable before/after ("EUR→USD 1.08 → 1.10 on 2026-10-18"). Sample-rate changes are also logged, flagged is_sample.

**Acceptance:** (1) Creating, editing, and deleting a rate each produce an fx_audit row; (2) the FX history view shows old → new values with timestamps; (3) audit rows cannot be edited or deleted from the UI; (4) entries link into the main app audit log by rate_id.
