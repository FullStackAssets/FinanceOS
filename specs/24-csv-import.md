# FinanceOS Builder Spec — §24 CSV Import (Items 461–480)

## Section overview
A guided CSV import pipeline for bringing the user's own exported transaction history (bank/credit-card CSV exports, spreadsheets) into FinanceOS. Flow: upload → column mapping (with auto-detect and saved presets) → validation + duplicate detection → preview → confirm → progress → post-import summary → full batch rollback. Every imported row is user-supplied manual data: it lands with `is_sample=false` and a `source='imported'` tag, and is never presented as a live bank feed.

## Tables touched
- `transactions` (existing): add `source` column — text, values `manual` / `imported` / `sample`, default `manual`. Backfill existing rows: `is_sample=true` → `sample`, else `manual`. Indexed for filtering.
- `accounts` (existing): no schema change; used for default/unmapped account assignment and opening-balance imports.
- NEW `import_batches`: id (pk), filename (text), imported_at (timestamp), status (text: `preview` / `importing` / `complete` / `rolled_back`), row_count (number), imported_count (number), skipped_count (number), error_count (number), default_account_id (fk, nullable), notes (text), is_sample (bool, default false).
- NEW `import_mappings`: id (pk), preset_name (text, unique), mapping_json (text: `{csv_header: field}`), sign_flip (bool), currency_column (text, nullable), created_at (timestamp), is_sample (bool, default false).
- `audit_log` (defined in §38): imports write entries with filename, row counts, timestamp.

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every import screen and wizard step.
- Seeded rows carry `is_sample=true` + SAMPLE badge and are excluded from totals when hidden; imported rows are `is_sample=false`, `source='imported'`, and always count as real user data.
- Imported data is labeled everywhere as user-supplied manual data — in the table, exports, history, and summaries. Nothing ever presents as a live bank feed or sync.

---

### 461. Build a CSV import wizard: upload, map columns, preview, confirm.
- Components: Modal `mdlImportWizard` (large, 4-step stepper header `stpImport`): Step 1 File Upload `uplCsv` (accept `.csv`, 5 MB cap); Step 2 Column Mapping (see §462); Step 3 Preview (see §464); Step 4 Confirm & Import. Buttons Back / Next / Cancel; Next disabled until step requirements pass. Entry point: "Import CSV" button on Transactions screen and in Settings → Data.
- Data: App state `stgImportFile` holds parsed rows (JS transformer `trParseCsv` splits on newlines, handles quoted commas). Steps stored in `stgImportStep` (1–4).
- Logic notes: File is parsed client-side first; nothing is written until Step 4 confirm. Canceling at any step discards state. Re-uploading resets mapping to auto-detect (§463).
- Acceptance: (1) Wizard opens from both entry points. (2) Cannot reach Preview with unmapped required fields (date, amount, description). (3) Nothing is written to `transactions` before Step 4 confirm.

### 462. Support mapping CSV columns to date, description, amount, category, account.
- Components: In wizard Step 2, one Select per CSV header row: `selMapDate`, `selMapDescription`, `selMapAmount`, `selMapCategory`, `selMapAccount`, plus optional `selMapNotes`, `selMapCurrency`. Each select lists target fields plus "Ignore column". Required: date, amount, description; category/account may fall back to defaults (§469).
- Data: Mapping object `stgColumnMap = {csvHeader: targetField}`. Preview transformer `trMappedPreview` applies it to the first 20 rows.
- Logic notes: Duplicate target assignment is blocked (one CSV column per target field). "Ignore column" drops the data. Category values are normalized to the fixed taxonomy; unknown values → "Other" with a warning chip.
- Acceptance: (1) Required-field selects must be set before Next. (2) Two CSV columns cannot map to the same target. (3) Preview reflects the mapping immediately.

### 463. Auto-detect common bank-export formats and suggest column mappings.
- Components: Info callout `txtAutoDetect` in Step 2: "Detected: Chase checking format — mapping suggested. Adjust if wrong." with "Use suggested" / "Start blank" buttons.
- Data: JS transformer `trDetectFormat` scores the header row against known patterns (regexes): `/(transaction|post) date/i` → date; `/description|memo|details/i` → description; `/amount|debit|credit/i` → amount; `/category/i` → category. Scores per known bank layout (Chase: "Details,Posting Date,Description,Amount,Type,Balance"; Amex: "Date,Description,Amount"; Citi; generic Mint/YNAB).
- Logic notes: Best-score match ≥ 3 header hits wins; otherwise no suggestion is shown and the user maps manually. Detection never writes data — it only pre-fills the selects.
- Acceptance: (1) A Chase-format CSV pre-fills date/description/amount correctly. (2) An unrecognized header set shows no suggestion. (3) User can override any suggested mapping.

### 464. Show a preview of the first 20 rows with mapped fields before import.
- Components: Table `tblImportPreview` in Step 3 with columns Date, Description, Category, Account, Amount (signed), plus status icons per row (valid ✓ / duplicate ⚠ / error ✗). Caption: "Showing first 20 of N rows."
- Data: Transformer `trMappedPreview` applies column map + defaults + sign-flip + validation to the first 20 parsed rows, returning `{rows, validCount, dupCount, errCount}`.
- Logic notes: Preview is computed live from current mapping — changing a mapping in Step 2 and returning updates it. Rows with errors show the error message inline; duplicates show the matched existing transaction date.
- Acceptance: (1) Exactly 20 rows render (or fewer if file is shorter). (2) Status icons match validation results. (3) Editing the mapping updates the preview without re-upload.

### 465. Flag imported rows with is_sample=false but an "imported" source tag.
- Components: Chip `Imported` (blue outline) on transaction rows where `source='imported'`, shown in the transactions table and row detail drawer. Tooltip: "Added by you via CSV import on <date>."
- Data: `INSERT INTO transactions (...) VALUES (..., is_sample=false, source='imported')` for every imported row. Migration adds `source` column default `'manual'`; backfill sets `sample` where `is_sample=true`.
- Logic notes: `source` is immutable after insert (no UI control changes it) to keep provenance honest. Filters can include/exclude `source='imported'` alongside the sample filter.
- Acceptance: (1) Every imported row has `is_sample=false`, `source='imported'`. (2) Imported rows appear in totals even when samples are hidden. (3) Chip renders on all imported rows, none on manual rows.

### 466. Detect likely duplicates against existing transactions by date+amount+description.
- Components: Warning banner in Step 3: "N of M rows look like duplicates of transactions you already have." Per-row ⚠ icon in preview linking to the matched existing row (opens read-only drawer).
- Data: Transformer `trFindDuplicates` — for each candidate row, query `transactions` for `date = ? AND amount = ? AND LOWER(description) = LOWER(?)` (exact match after trim). Returns matched row id per candidate.
- Logic notes: Match is exact on the triple after normalization (trim, lowercase description, amount rounded to cents). Near-matches (same date+amount, different description) are NOT flagged — only exact triples, to avoid false positives.
- Acceptance: (1) Re-importing the same file flags every row as duplicate. (2) A row differing only in description case still flags. (3) A row with a different amount does not flag.

### 467. Allow skipping duplicates or importing them with a "possible duplicate" note.
- Components: Radio group in Step 3 `rdoDupHandling`: "Skip duplicates (recommended)" / "Import anyway and mark as possible duplicate". Default: skip.
- Data: If skip: duplicate rows are excluded from the insert set and counted in `skipped_count`. If import-anyway: rows insert with `notes = '[Possible duplicate] ' + notes` appended, and `skipped_count=0` for those rows.
- Logic notes: The choice applies to the whole batch, not per row — keeps the wizard fast. The "[Possible duplicate]" prefix is searchable text so users can find and clean them later.
- Acceptance: (1) Skip mode inserts zero duplicate rows and reports the skipped count. (2) Import-anyway mode inserts them with the note prefix. (3) Post-import summary (§478) reflects which mode was used.

### 468. Validate dates and amounts row by row, listing errors before confirm.
- Components: Error panel `lstImportErrors` in Step 3 listing each bad row: "Row 14: 'Amount' is not a number ('abc')". Error rows are excluded from the import set; the Confirm button shows "Import N valid rows" with the error count.
- Data: Transformer `trValidateRows` checks per row: date parses to a real calendar date (rejects 2026-02-30, empty, or unparseable strings — tries MM/DD/YYYY, YYYY-MM-DD, DD-Mon-YY); amount is a finite number (strips `$`, commas, parentheses-as-negative); description non-empty after trim.
- Logic notes: Validation runs on mapped values, so a wrong column mapping surfaces as errors — which is the point. All errors are shown, not just the first.
- Acceptance: (1) A row with "not a number" amount is listed with its row number. (2) Impossible dates are rejected. (3) Confirm imports only valid rows; error count is reported.

### 469. Support assigning a default account and category for unmapped rows.
- Components: In Step 2, `selDefaultAccount` (populated from `accounts`) and `selDefaultCategory` (fixed taxonomy) labeled "Use for rows without an account/category". Shown when the account or category mapping is unset or a row's category value is blank.
- Data: Stored on the `import_batches` row (`default_account_id`) and applied in `trMappedPreview`/`trFinalRows`: blank mapped account → default account; blank/unknown category → default category.
- Logic notes: Defaults are required before leaving Step 2 (category default "Other" pre-selected; account must be chosen — no orphan transactions allowed). Defaults apply per row, so partially-mapped files still import cleanly.
- Acceptance: (1) Rows with blank category land in the chosen default category. (2) Wizard blocks progress without a default account. (3) Batch record stores the default account used.

### 470. Show import progress with counts of imported, skipped, and errored rows.
- Components: Step 4 becomes a progress view on confirm: Progress bar `prgImport`, live counters "Imported 42 · Skipped 3 · Errors 2", and a scrolling log list of the last few actions. On completion, auto-advances to the summary (§478).
- Data: Insert runs in chunks of 50 via a JS query loop `qryInsertChunk`, updating `stgImportProgress {done, imported, skipped, errors}` after each chunk; `import_batches` row updates to `status='importing'` then `'complete'`.
- Logic notes: Chunking keeps the UI responsive on thousand-row files. If a chunk fails, the batch is marked failed, already-inserted rows stay tagged with the batch id for rollback (§471), and the error is shown.
- Acceptance: (1) Progress bar advances during a 500-row import. (2) Counters match the final batch record. (3) A failed chunk surfaces the error instead of hanging.

### 471. Roll back an entire import batch with one undo action.
- Components: On the post-import summary and the import history screen (§480): red "Undo this import" button with a confirm dialog ("This will delete all N imported transactions. This cannot be undone."). Also an "Undo" toast for 30 seconds after import completes.
- Data: All rows inserted carry the batch id (via `import_batches.id` joined on `imported_at` window — simplest reliable key: store `batch_id` on a hidden staging column or match `source='imported' AND imported_at` range; spec uses a `batch_id` integer column on `transactions` set at insert). Rollback = `DELETE FROM transactions WHERE batch_id = ?`, then `import_batches.status='rolled_back'`, audit-logged.
- Logic notes: Rollback is all-or-nothing per batch. If the user edited an imported row after import, rollback still deletes it (warned in the confirm dialog). Rolled-back batches stay in history with status shown.
- Acceptance: (1) Undo deletes exactly the batch's rows, nothing else. (2) Batch status flips to `rolled_back`. (3) Audit log records the rollback with row count.

### 472. Log imports in the audit log with filename, row counts, and timestamp.
- Components: No new UI — entries appear in the audit-log viewer (§38) filterable by action type "import".
- Data: On import completion, insert into `audit_log`: `{actor:'user', action:'import', table:'transactions', filename, row_count, imported_count, skipped_count, error_count, batch_id, created_at}`. Rollbacks log `action:'import_rollback'` with the deleted count.
- Logic notes: One log entry per batch, not per row — row-level detail lives on the batch record. The entry links to the batch (click → import history detail).
- Acceptance: (1) Each completed import creates exactly one audit entry. (2) Entry shows filename and all three counts. (3) Rollback creates a matching `import_rollback` entry.

### 473. Support importing account opening balances from a simple name/balance CSV.
- Components: Wizard mode toggle at Step 1: "Transactions" / "Account balances". Balances mode shows a simplified 2-step flow (upload → map name/balance → confirm) and a note: "This sets balances, it does not create transactions."
- Data: CSV columns map to `account name` + `balance`. Matching is by exact account name (case-insensitive); unmatched names create new accounts of type "Checking" with `is_sample=false` after a confirm list ("2 new accounts will be created"). Each update stamps `updated_at` and writes to the balance-history log (§50).
- Logic notes: Balances mode never touches `transactions`. Negative balances on non-credit accounts trigger the same confirmation as manual entry (§55).
- Acceptance: (1) Matching names update balances with fresh timestamps. (2) Unknown names are listed for approval before creation. (3) No transaction rows are created in this mode.

### 474. Provide a downloadable CSV template with example rows.
- Components: "Download template" link button on Step 1 and in Settings → Data. Downloads `financeos-import-template.csv`.
- Data: Static file (stored as an app asset): headers `Date,Description,Amount,Category,Account,Notes` with 3 example rows (e.g., `2026-10-01,Whole Foods,-84.32,Food,Checking,groceries`), clearly real-format examples marked as examples in the Notes column.
- Logic notes: Template headers are chosen to auto-detect cleanly (§463) so a filled template imports with zero mapping effort.
- Acceptance: (1) Download produces a valid CSV that opens in spreadsheet apps (UTF-8). (2) Filling it and uploading auto-maps without manual fixes. (3) Example rows are obviously examples, not real finances.

### 475. Handle credit-card CSVs where spends are positive via a sign-flip option.
- Components: Toggle `tglSignFlip` in Step 2: "My CSV shows spends as positive numbers" (default off). When on, a hint explains spends become negative on import.
- Data: In `trFinalRows`, if enabled: `amount = -abs(amount)` for rows whose mapped category is not Income; Income-category rows keep their sign. Saved per mapping preset (§477).
- Logic notes: The flip applies at transform time so preview (§464) shows the final signed amounts — no surprises at confirm. Bank CSVs that already use negative spends must leave the toggle off.
- Acceptance: (1) With toggle on, a `45.00` spend imports as `-45.00`. (2) Income rows stay positive. (3) Preview reflects the flip before confirm.

### 476. Support multi-currency CSVs with a currency column mapped to settings.
- Components: Optional mapping target "Currency" in Step 2 (`selMapCurrency`), plus a note linking to Settings → currency (§40 spec). If a currency column is mapped, each row shows its currency code in preview.
- Data: Row's currency stored alongside the amount (transactions gains no new column in v1 — store as `notes` suffix `[USD]`? No: spec adds `currency` text column on `transactions`, defaulting to the home currency). Converted display follows the multi-currency spec (§785–§789); import itself stores native amounts only, never converts.
- Logic notes: Import never fetches rates and never converts — it records what the CSV says. Unknown currency codes fall back to home currency with a warning chip.
- Acceptance: (1) A `currency` column maps and stores per-row codes. (2) No conversion happens at import. (3) Unknown codes warn and fall back cleanly.

### 477. Allow saving a column mapping as a named preset for repeat imports.
- Components: In Step 2: "Save as preset" button prompting for a name; preset picker `selMappingPreset` at the top of Step 2 ("Start from preset…"); manage list in Settings → Data with rename/delete.
- Data: `import_mappings` row: `{preset_name, mapping_json, sign_flip, currency_column}`. Applying a preset fills all mapping selects + toggles; headers are matched by name, missing headers fall back to manual.
- Logic notes: Presets are per-user and survive app restarts. Deleting a preset never affects already-imported data.
- Acceptance: (1) Saving stores name + full mapping + sign-flip state. (2) Applying to a same-format file maps everything automatically. (3) A preset applied to a different-format file leaves unmatched fields blank for manual mapping.

### 478. Show a post-import summary: date range, total rows, total spend/income.
- Components: Summary card after Step 4: "Imported 118 transactions · Skipped 4 duplicates · 2 errors", date range "Sep 1 – Oct 18, 2026", totals "Spend $1,240.55 · Income $3,100.00", buttons: "View transactions" (filtered to the batch), "Undo import" (§471), "Done".
- Data: Computed from the batch's inserted rows: `MIN(date)`, `MAX(date)`, `SUM(amount < 0)`, `SUM(amount > 0)`, stored on `import_batches` for the history screen.
- Logic notes: Totals exclude skipped/error rows. Amounts shown signed, spend shown as positive dollars with a "spend" label to avoid sign confusion.
- Acceptance: (1) Date range matches the imported rows. (2) Spend/income totals reconcile with the transactions table. (3) "View transactions" opens the table filtered to exactly the batch.

### 479. Clearly label imported data as user-supplied manual data, never a live feed.
- Components: Persistent microcopy across the wizard, preview, summary, history, and row chips: "Imported from your CSV — your own data, entered by you." The transactions table's Imported chip tooltip (§465) repeats it. Export files include the `source` column (§499).
- Data: No new tables — this is a presentation rule enforced in every import-surface component's subtitle/help text and in the `source='imported'` provenance chain.
- Logic notes: The rule also covers edge phrasing: nowhere may the app say "synced", "from your bank", or "live". Copy review checklist is part of this item's acceptance.
- Acceptance: (1) Every import screen carries the user-supplied-data label. (2) No import-surface copy implies a bank connection. (3) Exports preserve the `source` column so the label survives outside the app.

### 480. Add an import history screen listing every past import batch.
- Components: Settings → Data → "Import history": table `tblImportHistory` with columns Date, Filename, Rows, Imported, Skipped, Errors, Status (complete / rolled_back / failed), actions: View (opens batch-filtered transactions), Undo (§471), Delete record (removes the history row only, never transactions — with confirm).
- Data: `SELECT * FROM import_batches ORDER BY imported_at DESC`. Detail drawer shows defaults used, preset name if any, and the audit-log entry link.
- Logic notes: History is the audit surface for imports — batches are never hard-deleted silently; "Delete record" only removes the history row and is itself audit-logged.
- Acceptance: (1) Every import and rollback appears with correct counts. (2) "View" filters the transactions table to exactly that batch's rows. (3) Deleting a history row does not delete transactions.
