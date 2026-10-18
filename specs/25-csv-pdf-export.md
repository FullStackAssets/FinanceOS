# FinanceOS Builder Spec — §25 CSV/PDF Export (Items 481–500)

## Section overview
Exports leave the app in two forms: CSV for data portability and PDF for statements. A single shared export engine drives both — one parameterized pipeline per dataset (transactions, budgets, accounts, goals, snapshots, bills) that honors current filters, sample-data visibility, household data scoping, and the manual-tracking stamp. Every export is stamped with "Manual tracking — no bank connection" plus the export date; sample rows carry a SAMPLE column value and watermark.

## Tables touched
- Existing: `accounts`, `transactions`, `budgets`, `goals` (all with `is_sample`).
- Assumed from neighboring sections: `net_worth_snapshots`, `bills`, `bill_payments`, `goal_contributions`, `income_sources`.
- NEW `export_history`: id (pk), export_type (text: transactions_csv / budgets_csv / accounts_csv / goals_csv / statements_pdf / snapshots_csv / bills_csv / tax_packet / yearend_pdf), filename (text), filters_json (text), row_count (number), includes_samples (bool), created_at (timestamp), created_by (text).
- NEW `transaction_sources` (or a `source` text column on `transactions` if absent): values `manual`, `imported`, `sample`, `ai_suggested` (item 499).

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen, including the exports screen.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data toggle hides samples, they are excluded from all exports.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.
- Every export is stamped with "Manual tracking — no bank connection" and the export date.

---

### 481. Export transactions to CSV honoring current filters with is_sample column included.
- Components: Button `btnExportTxCSV` on the transactions screen + export drawer `drExport` with format select (CSV/PDF). Reuses the active filter state from §06 (`qryTransactionsFiltered`).
- Data: Transformer `trExportTx` maps filtered rows to CSV columns: date, description, category, account_name, amount, notes, source, is_sample (true/false). JS: `utils.downloadFile(csvString, 'transactions-YYYYMMDD.csv', 'text/csv')`. BOM `\uFEFF` prepended for Excel compatibility.
- Logic notes: Export reflects EXACTLY what the filtered table shows — same filter set, same sample visibility. `is_sample` always included as the last column. Date formatted ISO (YYYY-MM-DD).
- Acceptance: (1) Exported row count equals filtered table count. (2) `is_sample` column present with true/false values. (3) Hiding samples removes those rows from the file.

### 482. Export budgets per month to CSV with category, limit, and actual columns.
- Components: Button `btnExportBudgets` on the budgets screen; month picker `selExportMonth` (defaults to current month).
- Data: Query `qryBudgetsExport` joins `budgets` to per-category actuals computed from `transactions` for the selected month (reuse §09 actuals logic). Columns: month, category, limit_amount, actual_spend, remaining, pct_used, is_sample (budget rows seeded as samples flagged).
- Logic notes: Actuals honor the sample toggle identically to the budgets screen. Categories with no budget set appear with empty limit and their actual.
- Acceptance: (1) Columns are exactly month, category, limit, actual, remaining, pct_used, is_sample. (2) Actuals match the budgets screen bars for that month. (3) Changing the month changes the rows.

### 483. Export accounts list to CSV with balances and updated_at timestamps.
- Components: Button `btnExportAccounts` on the accounts screen header.
- Data: Query `qryAccountsExport`: id, name, type, institution, balance, updated_at (ISO 8601), include_in_net_worth flag, is_sample. CSV via `utils.downloadFile`.
- Logic notes: `updated_at` always included so the user can see staleness outside the app. Archived accounts included with an `archived=true` column; sample accounts flagged.
- Acceptance: (1) One row per account including archived ones. (2) `updated_at` present in full timestamp form. (3) Balances match the accounts screen exactly.

### 484. Export goals and contributions to CSV.
- Components: Button `btnExportGoals` on the goals screen; scope select: "Summary only" vs "Summary + contributions".
- Data: Two CSVs (or two sheets-worth of sections in one file): goals summary (goal_name, target, saved, pct_complete, deadline, status, is_sample) and contributions detail (goal_name, date, amount, source_account, note, is_sample) from `goal_contributions`.
- Logic notes: Contributions export joins goal name for readability. Withdrawals appear as negative amounts. Summary-only mode skips the detail section.
- Acceptance: (1) Summary totals match the goals screen tiles. (2) Detail rows sum to each goal's `saved` value. (3) Sample goals flagged.

### 485. Generate a monthly PDF statement: summary, category table, transaction list.
- Components: Button `btnMonthlyPDF`; month picker; preview modal `mdPDFPreview` with an HTML print view `htmlStatement`.
- Data: Query bundle: month summary (income, spend, savings rate), per-category actuals, full transaction list for the month. Render into a styled HTML template (header with banner text, three sections), then print-to-PDF via the browser print dialog (`utils.print` / window.print on the statement frame) or a PDF library.
- Logic notes: Statement shows the month label prominently, all figures from manual data, sample rows marked SAMPLE inline. Follows the print stylesheet from item 498.
- Acceptance: (1) PDF contains summary, category table, and transaction list sections in order. (2) Banner text appears in the header. (3) Figures match the month-review screen.

### 486. Stamp every export with "Manual tracking — no bank connection" and export date.
- Components: Shared export header builder — a JS function `buildExportStamp()` used by ALL CSV and PDF exporters; no per-export opt-out.
- Data: Stamp text: `Manual tracking — no bank connection | Exported YYYY-MM-DD HH:MM (local) | FinanceOS manual-entry data`. In CSVs it goes in the first two rows (row 1: stamp, row 2: blank) or as a `#` comment header; in PDFs as a fixed header/footer on every page.
- Logic notes: The stamp is applied by the shared engine, so new export types inherit it automatically. Date uses the user's locale/timezone from settings.
- Acceptance: (1) Every export type shows the exact stamp text and a date. (2) No export path exists that skips the stamp. (3) PDFs repeat it on every page footer.

### 487. Add a sample-data watermark on exports that include sample rows.
- Components: Watermark flag computed in `trExportTx`/shared engine: `includesSamples = rows.some(r => r.is_sample)`.
- Data: If true, CSVs get an extra leading row `Contains sample data — not real finances.`; PDFs get a diagonal "SAMPLE DATA" watermark on every page plus the banner variant text.
- Logic notes: Watermark is data-driven — exporting with samples hidden never triggers it. The per-row `is_sample` column (item 481) is the machine-readable counterpart.
- Acceptance: (1) Export with samples visible shows the watermark. (2) Export with samples hidden shows no watermark. (3) PDF watermark is visible on print, not just screen.

### 488. Support exporting a single account's statement for a date range.
- Components: On the accounts screen, per-row overflow action "Export statement"; dialog with date-range picker `dtrStmt` (default: this month) and format select.
- Data: Query `qryAccountStatement`: transactions for that account in range + opening balance (reconstructed: current balance minus in-range net) and closing balance. PDF layout mirrors item 485 but scoped to one account; CSV option included.
- Logic notes: Opening balance is labeled "reconstructed from manual entries". Running-balance column included when a single account is exported (ties to item 88).
- Acceptance: (1) Only the chosen account's transactions appear. (2) Date range filters exactly. (3) Opening/closing balances reconcile with the row math.

### 489. Export net-worth snapshot history to CSV.
- Components: Button `btnExportSnapshots` on the net-worth history screen.
- Data: Query `qrySnapshotsExport` from `net_worth_snapshots`: snapshot_date, total_value, per-account breakdown columns (one column per account, labeled), note, is_sample. Wide-format CSV: one row per snapshot, accounts as columns.
- Logic notes: Column headers use account names at export time; a `accounts_included` header row lists which accounts fed each column. Sample snapshots flagged per row.
- Acceptance: (1) One row per snapshot in date order. (2) Per-account columns sum to total_value per row. (3) Notes preserved verbatim.

### 490. Provide a year-end summary PDF: 12-month totals by category plus net-worth chart.
- Components: Button `btnYearEndPDF`; year picker `selYear`; preview modal.
- Data: Aggregation query: 12 monthly columns × 7 categories of spend, plus income row and savings-rate row; net-worth snapshots plotted as a line chart (canvas → image embedded in PDF). Reuses §26 annual-report logic.
- Logic notes: Months with no data show `—`, not zero, to distinguish "no entries" from "$0". Sample-data note included when samples are hidden. Chart rendered at 2× for print sharpness.
- Acceptance: (1) All 12 months present in order. (2) Category totals match the monthly reports. (3) Net-worth chart renders in the PDF.

### 491. Allow choosing columns for CSV exports via checkboxes.
- Components: In the export drawer, a checkbox group `chkColumns` listing all available columns for the chosen dataset; "Select all / clear" links; per-dataset column sets persisted in `localStorage` (`financeos_export_cols_<type>`).
- Data: Transformer applies the selection before CSV serialization: `rows.map(r => pick(r, selectedCols))`. The `is_sample` column is locked ON (cannot be unchecked) so sample labeling can never be stripped.
- Logic notes: Column order follows the checkbox order, not table order. Persisted preferences load on drawer open.
- Acceptance: (1) Unchecking a column removes it from the file. (2) `is_sample` cannot be unchecked. (3) Preferences persist across sessions.

### 492. Export bills and their payment history to CSV.
- Components: Button `btnExportBills` on the bills screen.
- Data: Two sections: bills master (name, category, amount, due_day, account, status, autopay_tag, is_sample) and payment history (bill_name, month, paid_amount, paid_date, paid_late, late_fee, is_sample) from `bill_payments`.
- Logic notes: Paused bills included with status `paused`. Variable-amount bills export their estimated default with an `amount_estimated=true` column.
- Acceptance: (1) Every bill has one master row. (2) Payment rows link to bills by name. (3) Late flags preserved.

### 493. Generate a tax-helper export: Income-category transactions plus deductible-tagged rows.
- Components: Button `btnTaxExport`; tax-year picker `selTaxYear`.
- Data: Query: all `Income`-category transactions for the year + all transactions tagged deductible/medical/charitable (§41), with columns: date, description, category, tax_tag, amount, receipt_attached (true/false), is_sample. Excludes sample rows when the sample toggle hides them.
- Logic notes: This is a helper, not advice — the export header includes the "not tax advice — review with a professional" disclaimer (§41 item 811). Deductible splits export as separate rows with `split_of` reference.
- Acceptance: (1) All Income rows for the year present. (2) All deductible-tagged rows present, including splits. (3) Disclaimer stamped on the file.

### 494. Support emailing an export to yourself (human-initiated send only).
- Components: In the export drawer, "Email to me" button `btnEmailExport` with an email input prefilled from settings; explicit confirm dialog ("Send this export to you@example.com?") — the send fires ONLY on this tap.
- Data: On confirm, the generated file is handed to the send path (Retool email resource or `mailto:` fallback with attachment note). The action is logged to `export_history` with `delivery: email` and in the audit log.
- Logic notes: No automatic, scheduled, or rule-triggered sends — the button is the only trigger, and it requires a fresh tap each time. Never sends to any address other than the user's own configured address.
- Acceptance: (1) No export is emailed without the explicit tap + confirm. (2) Only the user's own address is accepted. (3) Each send is logged with timestamp.

### 495. Show export history with filename, filters used, and timestamp.
- Components: Exports screen tab "History": table `tblExportHistory` bound to `export_history`, newest first, with columns filename, type, filters summary, rows, samples included, created_at, delivery.
- Data: Every export writes one `export_history` row via `qryLogExport` before the download starts. Filters stored as compact JSON (`{category, date_range, account, hide_samples}`).
- Logic notes: History is append-only; no delete. "Re-run" action re-executes the same export with the stored filters.
- Acceptance: (1) Every export creates a history row. (2) Filters column describes the filter set readably. (3) Re-run reproduces the same row count.

### 496. Compress large exports with a progress indicator.
- Components: Progress modal `mdExportProgress` with determinate bar and cancel button, shown when estimated rows > 5,000.
- Data: Chunked generation: build CSV in 1,000-row chunks via `setTimeout` slices, updating progress; then compress with a client-side zip (JSZip) when the user chose "ZIP" format or rows > 20,000.
- Logic notes: Cancel aborts cleanly with no partial download. Progress shows "Preparing rows… 45%". Small exports skip the modal entirely.
- Acceptance: (1) Exports over 5,000 rows show progress. (2) Cancel stops generation. (3) ZIP output opens with the CSV inside, UTF-8 intact.

### 497. Ensure exports never include other users' data in shared-household mode.
- Components: No UI — enforced in the data layer: every export query includes `AND household_member_id = {{ currentUser.id }}` (or the shared-household visibility rule from §39).
- Data: A shared transformer `trScopeRows(rows)` filters to the viewer's permitted scope; export engine calls it last, after filters but before serialization. Violation attempts (tampered filter JSON) are logged to the audit log.
- Logic notes: Defense in depth: UI scoping + query predicate + final transformer check. The rule applies to CSV, PDF, and emailed exports identically.
- Acceptance: (1) A viewer-role member's export contains only their permitted rows. (2) Tampering the filter state cannot widen the scope. (3) Scope violations are audit-logged.

### 498. Add a print stylesheet for all statement views.
- Components: Global `@media print` stylesheet `cssPrint` in the app theme; statement HTML uses semantic classes (`stmt-header`, `stmt-table`, `stmt-footer`).
- Data: Pure CSS: hide nav/banner chrome duplicates (keep the stamp banner), force black-on-white, repeat table headers (`thead { display: table-header-group }`), page-break rules (no orphaned totals), A4/Letter sizing.
- Logic notes: One stylesheet serves monthly statements, account statements, year-end PDFs, and audit summaries — statement templates share the same classes. Screen styles never leak into print.
- Acceptance: (1) Printed statements show no app chrome. (2) Multi-page tables repeat headers. (3) Colors print legibly in grayscale.

### 499. Include the data-entry method ("manual"/"imported"/"sample") per row in exports.
- Components: `source` column added to every CSV export's column set (locked visible like `is_sample`).
- Data: Values: `manual` (hand-entered), `imported` (CSV import), `sample` (seeded/sample template), `ai_suggested` (accepted AI suggestion). Derived: `is_sample=true → sample`; else the row's `source` field (backfilled to `manual` where null).
- Logic notes: The column answers "where did this row come from" for every row, complementing `is_sample`. Backfill migration sets legacy nulls to `manual` on first export.
- Acceptance: (1) Every row has a non-empty source value. (2) Sample rows always show `sample`. (3) Column cannot be deselected in the column picker.

### 500. Validate exported CSVs open correctly in spreadsheet apps with UTF-8 encoding.
- Components: Export QA checklist (part of the build notes, item 960): a scripted check run before release — generate each CSV type with adversarial data (commas, quotes, newlines, emoji, non-Latin scripts in descriptions) and open in Excel, Google Sheets, and Numbers.
- Data: CSV writer rules: RFC 4180 quoting (quote fields containing `,` `"` `\n`; double internal quotes), `\r\n` line endings, UTF-8 BOM prefix. Newlines inside fields preserved via quoting, never stripped.
- Logic notes: The BOM is what makes Excel detect UTF-8; Google Sheets handles it fine. This is a build-time QA gate, re-run whenever the CSV writer changes.
- Acceptance: (1) Adversarial CSV opens in Excel/Sheets/Numbers with columns aligned. (2) Non-Latin characters render correctly. (3) No data loss on round-trip re-import.
