# FinanceOS Builder Spec — §18 Manual-entry banner & sample data (Items 341–360)

## Section overview
The honesty layer of FinanceOS: a persistent "Manual tracking — no bank connection yet." banner on every screen, plus full lifecycle management for clearly-labeled sample data (seed, hide, inventory, convert, delete, export watermarking). This section guarantees no screen can ever render financial figures without the banner or a sample indicator present.

## Tables touched
- `accounts`, `transactions`, `budgets`, `goals` (existing; all carry `is_sample bool`).
- NEW `app_settings`: id (pk), setting_key (text, unique: banner_position, sample_toggle_default, hide_samples), setting_value (text), updated_at (timestamp). Single-row-per-key store for global toggles (banner top/footer, hide-sample default).
- NEW `audit_log`: id (pk), actor (text), action (text), table_name (text), record_id (text), detail (text), created_at (timestamp). Append-only; used for sample-data delete/convert events (§358) and banner-setting changes.
- Shared sample-seed rows: 3 accounts, 15 transactions, 4 budgets, 2 goals — all `is_sample=true` (existing seed spec in App 11).

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen, including the transactions screen above the filter bar.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data filter hides samples, they are excluded from the table, counts, totals, and exports.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.

---

### 341. Pin the "Manual tracking — no bank connection yet." banner on every main screen.
- Components: Shared module `modBanner` containing a Callout `bannerManual` (text exactly "Manual tracking — no bank connection yet.", icon info-circle, full-width, non-dismissible in module default). Import `modBanner` at the top of every app page: Overview, Transactions, Budgets, Accounts, Goals, Bills, Trends, Reports, Recurring, Settings, Onboarding.
- Data: No query; static text component.
- Logic notes: Banner is part of the page template, not page content — it renders above all scroll containers so it cannot be scrolled out of view. z-index above sticky headers.
- Acceptance: (1) Banner visible at top of all 11 main screens. (2) Exact text match, no typos. (3) Banner survives filter changes, drawer opens, and tab switches.

### 342. Make the banner dismissible per session but restored on next app load.
- Components: Icon Button `btnDismissBanner` (×) inside `modBanner`; temp state `stBannerDismissed` (bool, default false, session-scoped).
- Data: `btnDismissBanner` click sets `stBannerDismissed` to true; banner hidden when `stBannerDismissed.value === true`.
- Logic notes: Temp state resets on app reload by definition (not persisted to `app_settings`). Dismissal applies per browser session only. Slim one-line reminder remains in the header ("manual tracking" link) while dismissed so the state is discoverable.
- Acceptance: (1) Dismissing hides the banner for the session. (2) Full page reload restores it. (3) Dismissal never persists across logins or devices.

### 343. Add a settings toggle to move the banner from top to a slim footer strip.
- Components: Settings → Data & display: Segmented Control `segBannerPosition` (Top / Slim footer strip); `modBanner` container `ctrBannerFooter` pinned to bottom of each page.
- Data: `qryGetSetting('banner_position')` / `qrySetSetting` against `app_settings`; `modBanner` renders in header slot when value = "top", footer slot when "footer".
- Logic notes: Footer strip is a slim 28px bar, same text, pinned above the bottom tab bar on mobile. Switching applies instantly to all screens via the shared module. Default = "top".
- Acceptance: (1) Toggle moves banner on all screens immediately. (2) Choice persists across reloads via `app_settings`. (3) Footer variant never overlaps tab bar or content (safe-area padding).

### 344. Seed the app with clearly-labeled sample accounts, transactions, budgets, and goals on first run.
- Components: Onboarding step "Sample data choice"; Button `btnSeedSamples` ("Start with sample data").
- Data: Query `qrySeedSamples` runs multi-statement INSERTs into accounts (3), transactions (15), budgets (4), goals (2) — every row `is_sample=true`. Guard: `SELECT COUNT(*) FROM accounts WHERE is_sample=true` — skip if seeds already exist.
- Logic notes: Seeding is opt-in during onboarding (item 363) and idempotent; re-running never duplicates. Sample amounts are realistic but clearly modest (e.g., "Sample Checking" $2,450).
- Acceptance: (1) One tap creates the full labeled dataset. (2) Re-running creates zero duplicates. (3) Every seeded row has `is_sample=true`.

### 345. Mark every seeded row with is_sample=true and a visible "SAMPLE" badge.
- Components: Badge `bdgSample` (amber pill, text "SAMPLE") rendered on every row/card in accounts list, transactions table, budgets list, goals list, and overview tiles when `is_sample` is true.
- Data: All list queries select `is_sample`; row rendering condition `{{ row.is_sample }}` shows the badge.
- Logic notes: Badge is non-removable while `is_sample=true` — it is a function of the flag, not a separate field. Applies to detail drawers too (account detail, transaction drawer, goal detail).
- Acceptance: (1) Every sample row in every list shows the badge. (2) Real rows never show it. (3) Badge visible in detail views, not just lists.

### 346. Add a global "hide sample data" toggle that filters samples from all screens and totals.
- Components: Header chip `chipHideSamples` (toggle, label "Hide sample data") present in the app header on all screens; persisted default in `app_settings` (`hide_samples`).
- Data: Every data query appends `AND ({{ !chipHideSamples.value }} OR is_sample = false)`; transformer `trNetWorth`, budget actuals, goal progress, trend charts all consume the filtered sets.
- Logic notes: Toggle is global and sticky across screens and sessions. When ON, samples vanish from tables, tiles, charts, totals, and exports. Default OFF on first run so the sample dashboard is visible (item 372).
- Acceptance: (1) Toggling instantly re-renders every screen without samples. (2) Net worth, budget actuals, and goal bars all exclude samples. (3) Setting persists across reloads.

### 347. Provide a one-tap "delete all sample data" action with confirmation.
- Components: Settings → Data management: Danger Button `btnDeleteSamples` ("Delete all sample data"); confirmation Modal `mdlConfirmDeleteSamples` requiring typed confirmation "DELETE".
- Data: `qryDeleteSamples` runs `DELETE FROM transactions WHERE is_sample=true; DELETE FROM budgets WHERE is_sample=true; DELETE FROM goals WHERE is_sample=true; DELETE FROM accounts WHERE is_sample=true;` in one transaction; then `qryLogAudit('samples_deleted', counts)`.
- Logic notes: Deletes only `is_sample=true` rows — real user data is untouched even if names match. Counts per table shown in the confirmation ("This will delete 24 sample rows"). Action is irreversible; undo is via re-seeding (§344).
- Acceptance: (1) Confirmation requires typing DELETE. (2) Zero real rows affected (verified by pre/post count of `is_sample=false`). (3) Audit log records per-table counts and timestamp.

### 348. Show a sample-data inventory screen listing all sample rows by table.
- Components: Settings → Data management → "Sample data inventory": Accordion `accSampleInventory` with sections per table (Accounts, Transactions, Budgets, Goals); each a Table of that table's `is_sample=true` rows with counts in the section header.
- Data: Four queries `qrySamplesByTable_*` = `SELECT * FROM <table> WHERE is_sample=true`.
- Logic notes: Empty state per section: "No sample rows in this table." Header shows live total ("24 sample rows"). Links from each row jump to the row's home screen.
- Acceptance: (1) Counts match `SELECT COUNT(*) WHERE is_sample=true` per table. (2) Every sample row in the app appears exactly once. (3) Screen updates immediately after seed/delete/convert.

### 349. Prevent sample transactions from affecting budget actuals when hidden.
- Components: Budget bars on Budgets screen and overview spend-vs-budget tile.
- Data: Budget-actual transformer `trBudgetActuals` computes actuals from `qryTransactionsThisMonth` with the global sample filter applied — when `chipHideSamples` is ON, `WHERE is_sample=false`.
- Logic notes: Actuals must always be derived from the same filtered transaction set the user sees on the Transactions screen — no second code path. When samples are visible, they count (and are badged); when hidden, they don't.
- Acceptance: (1) With samples hidden, adding/deleting a sample transaction does not move any budget bar. (2) With samples visible, it does (and the bar's tooltip notes samples included). (3) Monthly totals on overview match the budgets screen exactly.

### 350. Add a "replace samples with my data" guided flow that deletes samples then opens onboarding.
- Components: Settings → Data management: Button `btnReplaceSamples` ("Replace samples with my data"); 3-step guided Modal `mdlReplaceSamples`: (1) review what will be deleted (counts), (2) confirm, (3) done → launches onboarding step "add accounts".
- Data: Reuses `qryDeleteSamples` from §347, then sets onboarding progress state and navigates to the accounts onboarding step.
- Logic notes: Only offered when at least one sample row exists. After completion the user lands in onboarding with a blank slate; the "start with sample data" choice is not re-offered silently.
- Acceptance: (1) Flow deletes all samples then opens onboarding at account creation. (2) Cancel at any step leaves data untouched. (3) Completing twice in a row is harmless (second run finds nothing to delete).

### 351. Watermark exported CSVs that include sample rows with a SAMPLE column value.
- Components: All CSV export actions (transactions, budgets, accounts, goals, net-worth history).
- Data: Export transformer `trExportCSV` includes the `is_sample` column on every row (header `is_sample`, values `TRUE`/`FALSE`); never strips the column even when the user deselects columns (§491 keeps it mandatory).
- Logic notes: Watermarking is structural, not decorative: any spreadsheet opened from the export self-identifies sample rows. Applies to filtered exports too (§120, §481).
- Acceptance: (1) Every CSV export contains an `is_sample` column. (2) Sample rows read TRUE, real rows FALSE. (3) Column cannot be deselected in the column picker.

### 352. Show a banner variant on exports: "Contains sample data — not real finances."
- Components: Export dialogs and the generated file headers.
- Data: Transformer logic: if any exported row has `is_sample=true`, prepend CSV comment line `# Contains sample data — not real finances.` and set the PDF statement header subtitle to "Contains sample data — not real finances." (§485, §490).
- Logic notes: The variant is additive — the standard "Manual tracking — no bank connection" stamp (§486) still appears. If no sample rows are included, the variant line is omitted (no false alarm).
- Acceptance: (1) Exports with ≥1 sample row carry the variant text. (2) Pure-real exports do not. (3) PDF statements show it under the main banner line.

### 353. Allow converting a sample row to real by clearing its flag after user review.
- Components: Per-row action "Mark as real" in transactions drawer, account detail, goal detail, and the sample inventory screen (§348).
- Data: `qryConvertSample` = `UPDATE <table> SET is_sample=false WHERE id={{ id }} AND is_sample=true`; then `qryLogAudit('sample_converted', table, id)`.
- Logic notes: One row at a time — no bulk convert without review (bulk "mark all real" exists separately at §57 for post-replacement). The SAMPLE badge disappears immediately and the row enters real totals. Requires the row to be currently visible (samples not hidden).
- Acceptance: (1) Converting one row removes its badge and includes it in totals. (2) Audit log records the conversion with timestamp. (3) Converting a row twice is a no-op.

### 354. Display a first-run explainer card: what manual tracking means and its limits.
- Components: Onboarding welcome step: Card `crdManualExplainer` with headline "You enter the numbers — the app does the math", body text covering: no bank connection, balances updated by hand, figures are only as current as the last update, sample data is clearly labeled.
- Data: Static content; no query.
- Logic notes: Shown before the sample-data choice (§363) so the choice is informed. Also reachable later from Help → "Manual tracking explained" (§962). Plain language, no jargon; max ~120 words.
- Acceptance: (1) Card is the first content on the welcome step. (2) Copy explicitly states no bank connection and manual updates. (3) Accessible from help after onboarding.

### 355. Add a persistent "how to update balances" help link next to the banner.
- Components: Link `lnkUpdateBalancesHelp` ("How to update balances") rendered inline beside the banner text in `modBanner` on the Overview and Accounts screens.
- Data: Opens help article modal `mdlHelpUpdateBalances` (routine guide per §966): open Accounts → tap account → Update balance → enter amount.
- Logic notes: Link sits inside the banner module so it follows the banner's position setting (§343). On the footer-strip variant it appears as a small "?" icon to save space.
- Acceptance: (1) Link visible next to the banner on Overview and Accounts. (2) Opens the correct help article in one tap. (3) Present in both top and footer banner positions.

### 356. Show sample vs real totals side by side on the overview when samples are visible.
- Components: Overview hero: paired tiles `tileNetWorthReal` / `tileNetWorthSample` and `tileSpendReal` / `tileSpendSample`, shown only when `chipHideSamples` is OFF and at least one sample row exists.
- Data: Transformers compute both aggregates: `SUM(balance) WHERE is_sample=false` vs `=true`, same for month spend. Labels: "Your entries" vs "Sample data".
- Logic notes: Side-by-side display prevents the classic confusion of blended totals. When samples are hidden, the sample tiles disappear and the real tiles take full width.
- Acceptance: (1) With samples visible, both totals render labeled. (2) Real total excludes every sample row. (3) Hiding samples collapses to real-only layout.

### 357. Prevent goal progress from counting sample contributions when samples are hidden.
- Components: Goals screen progress bars and overview goal tiles.
- Data: Goal progress computed from `goal_contributions` (or the `saved` rollup) filtered by the global sample toggle; contribution rows from sample accounts (§234) carry `is_sample=true` and are excluded when hidden.
- Logic notes: Same single-filter principle as §349 — goal math consumes the filtered set, never a separate unfiltered query. When samples are visible, sample contributions count toward progress but the goal shows the SAMPLE badge (§208).
- Acceptance: (1) Hiding samples drops sample-funded progress from every goal bar. (2) Showing samples restores it with the badge visible. (3) Overview goal tiles match the Goals screen exactly.

### 358. Add an audit-log entry whenever sample data is deleted or converted.
- Components: `audit_log` table viewer (§743) — no new UI; the entries appear under action types `samples_deleted`, `sample_converted`, `samples_seeded`.
- Data: Every seed (§344), bulk delete (§347), guided replace (§350), and single convert (§353) calls `qryLogAudit` with actor ("user"), table, record id or counts, and timestamp.
- Logic notes: Entries are immutable (append-only per §744) and survive the deletion itself — the log records that rows were deleted, not the row contents. Filterable by action type in the audit viewer.
- Acceptance: (1) Each delete/convert/seed produces exactly one audit entry. (2) Entries persist after the sample rows are gone. (3) Entries show counts (bulk) or record id (single).

### 359. Include the banner text in print-friendly statement views.
- Components: Print stylesheet for the monthly statement view (§100, §498).
- Data: The statement header block renders "Manual tracking — no bank connection yet." as the first line, plus the sample-variant line when applicable (§352).
- Logic notes: Print CSS (`@media print`) forces the banner visible even if the user dismissed it on screen (§342) — printed output must never leave the app without the honesty label. Banner prints in bold, 12pt, top of page 1.
- Acceptance: (1) Print preview shows the banner on page 1. (2) Banner prints even when dismissed in-session. (3) Exported/printed PDFs carry the same text (§485).

### 360. Test that no screen can render financial figures without the banner or a sample indicator present.
- Components: QA checklist screen `scrBannerAudit` (Settings → Diagnostics), runnable on demand: a scripted checklist, not a one-off manual test.
- Data: Checklist runner iterates the 11 main screens in a headless pass: for each screen asserts (a) `modBanner` is present in the DOM and visible, or (b) every financial figure on screen carries a sample indicator (SAMPLE badge / `is_sample` flag rendered). Results table: screen, check, pass/fail, timestamp; failures list the offending component.
- Logic notes: Concrete checks per screen — Overview: banner visible AND net-worth tile shows banner or badge; Transactions: slim banner strip above table (§91) AND sample pills on sample rows (§85); Budgets: header banner (§160) AND actuals respect sample filter (§349); Accounts: header banner (§59) AND SAMPLE badges (§46); Goals: header banner (§220) AND goal badges (§208); Bills, Trends, Reports, Recurring, Cash-flow, Settings: banner present, no figures without it. Export/print paths checked separately (§351, §352, §359). The runner fails closed: any screen where a figure renders with neither banner nor indicator is a FAIL with the component name.
- Acceptance: (1) Runner executes all 11 screens and reports pass/fail per screen. (2) Deliberately removing the banner from a test screen produces a FAIL naming that screen. (3) Results are timestamped and viewable in Diagnostics; a failing run blocks the release checklist.
