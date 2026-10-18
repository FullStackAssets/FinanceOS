# FinanceOS Builder Spec — §41 Tax Prep Helpers (Items 801–820)

## Section overview
Tax-organization tools for the user's own manual records: tagging transactions as tax-relevant, a tax-year view grouping income and deductions, charitable/medical totals, a mileage log, 1099-style income tracking, estimated quarterly payment tracking, receipt prompts, a downloadable tax packet, a read-only CPA-share link with expiry, and a "filed" lock per tax year. Everything here is informational and organizational only — FinanceOS never generates tax advice, computes a tax liability, or files anything. All source data is manual entry or labeled samples, same as the rest of the app.

## Tables touched
New tables introduced in this section (full schema):
- `tax_documents`: id (pk), tax_year (number), doc_name (text), doc_type (text: income_summary / deduction_list / receipt_index / packet_pdf), file_link (text), created_at (timestamp), is_sample (bool)
- `mileage_log`: id (pk), date (date), miles (number), purpose (text), transaction_id (fk, nullable), is_sample (bool)
- `tax_years`: id (pk), year (number), status (text: open / filed), filed_at (timestamp, nullable), packet_snapshot (text JSON, nullable), is_sample (bool)
- `tax_shares`: id (pk), tax_year (number), share_token (text), expires_at (timestamp), created_at (timestamp)
- `estimated_tax_payments`: id (pk), quarter (text: Q1 / Q2 / Q3 / Q4), tax_year (number), due_date (date), amount (number), paid (bool), is_sample (bool)
- `tax_splits`: id (pk), transaction_id (fk), deductible_amount (number), non_deductible_amount (number), note (text), is_sample (bool)
- Existing tables extended: `transactions` gains `tax_tag` (text, nullable: deductible / medical / charitable / home_office / business / other), `tax_note` (text, nullable), `payer` (text, nullable, for 1099-style income), and `tax_category` (text, nullable) via migration; `goals`-style audit integration reuses the audit-log pattern from §38.

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears on every tax screen.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data filter hides samples, they are excluded from totals, exports, and packet PDFs.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.
- "Not tax advice — review with a professional" disclaimer on every tax screen (item 811).

---

### 801. Tag transactions as tax-relevant with categories like deductible, medical, charitable.
- Components: On the transaction detail drawer, a "Tax" section with a Select `selTaxTag` (options: None, Deductible, Medical, Charitable, Home office, Business, Other) and a Text Area `txtTaxNote` for a short note. The transactions table gains a small tag chip column showing the tag value.
- Data: Update query `qryUpdateTaxTag` writes `tax_tag` and `tax_note` to `transactions` for the selected row; refresh of the transactions query re-renders chips. A transformer `tfrmTaxTagged` filters `qryTransactions.data` to rows with non-null `tax_tag`.
- Logic notes: Tagging is per transaction and optional; clearing the select back to "None" nulls the tag. Tags survive category recategorization. Sample transactions keep their SAMPLE badge regardless of tag.
- Acceptance: (1) Tag and note persist and re-render on the detail drawer. (2) Tag chip visible in the transactions table. (3) Clearing the tag removes the chip. (4) Untagged totals elsewhere are unaffected.

### 802. Build a tax-year view grouping Income and tagged deductions.
- Components: New screen `TaxYearView` with a Number/Select year picker `selTaxYear` (2025, 2026, + next year auto), two grouped tables or cards: `tblTaxIncome` (transactions with category = Income in that year) and `tblTaxDeductions` (transactions with a non-null `tax_tag` in that year), plus header stat cards.
- Data: Queries `qryTaxIncome` and `qryTaxDeductions` filter `transactions` by `date` year and `tax_tag`/`category`; a transformer `tfrmTaxYearRange` computes Jan 1–Dec 31 bounds for the selected year, honoring the year-boundary note (item 806).
- Logic notes: Income group shows gross total; deductions group sub-totals by tag value. Rows respect the sample toggle — hiding samples removes them from both groups and the totals. Split rows (item 815) contribute their deductible portion only.
- Acceptance: (1) Switching year re-queries both groups. (2) Income total equals sum of Income-category rows for that year. (3) Deduction groups match tagged rows exactly. (4) Sample-hidden totals exclude samples.

### 803. Show a year-to-date deductible total from manually tagged rows.
- Components: Stat card `statYtdDeductible` on the tax-year view and a smaller variant on the Overview screen (optional placement), showing the YTD deductible total with a "manually tagged" caption.
- Data: Transformer `tfrmYtdDeductible` sums `amount` (absolute value of spend rows) where `tax_tag` is non-null and `date` falls in the current calendar year, excluding samples when the sample toggle is off. For split rows, sums the `deductible_amount` from `tax_splits` instead of the full amount.
- Logic notes: YTD = Jan 1 through today, not the selected tax year — label the card explicitly "Year to date (Jan 1 – today)". Income rows never count toward deductions even if mis-tagged. Zero state shows $0 with guidance to tag transactions.
- Acceptance: (1) Total matches the sum of tagged spend rows YTD. (2) Sample-hidden state excludes samples. (3) Split-row transactions contribute only the deductible part. (4) Card caption names the date range.

### 804. Add receipt attachment prompts for transactions tagged tax-relevant.
- Components: When `selTaxTag` is set to a non-None value on a transaction that has no receipt attached, show an inline callout `calloutTaxReceipt` ("Add a receipt for this tax-relevant transaction") with an "Attach receipt" button opening the existing receipt upload flow from §23.
- Data: Reuses the receipt-attachment storage/query from §23; the tax-year view query includes a `has_receipt` boolean per row (join on receipt attachments by transaction_id). A transformer `tfrmTaxMissingReceipts` lists tax-tagged rows with no attachment.
- Logic notes: Prompt is a nudge, never a blocker — the tag saves without a receipt. The tax packet's receipt index (item 805) flags missing receipts explicitly rather than omitting those rows.
- Acceptance: (1) Prompt appears only for tagged rows lacking receipts. (2) Attaching a receipt from the prompt dismisses it. (3) Missing-receipt list matches tagged rows without attachments. (4) Tagging without receipt is allowed.

### 805. Export a tax packet: income summary, deduction list, receipt index.
- Components: Button `btnExportTaxPacket` on the tax-year view; on click, a modal `modalTaxPacket` lets the user confirm the tax year and choose contents (income summary, deduction list, receipt index all checked by default), then generates the packet.
- Data: Queries assemble three datasets: income summary (income rows by month + total), deduction list (tagged rows with tag, amount, note), receipt index (tagged rows with attachment links, flagged missing). A JS query `qryBuildTaxPacket` compiles them into a printable HTML/PDF via Retool's PDF export, and inserts a row into `tax_documents` with doc_type `packet_pdf`.
- Logic notes: Packet generation is a snapshot — it embeds the generation timestamp and tax year. Sample rows are excluded when the sample toggle hides them, and the packet states which data mode was used.
- Acceptance: (1) PDF contains all three sections for the chosen year. (2) Totals match the on-screen tax-year view. (3) A `tax_documents` row records the export. (4) Missing receipts are flagged, not silently dropped.

### 806. Support tax-year selection (2025, 2026) with a year-boundary note.
- Components: `selTaxYear` select on the tax-year view with options 2025 and 2026 (plus the next year appearing automatically each January); a small info banner `bannerYearBoundary` under it reading "Tax year runs Jan 1 – Dec 31 based on transaction date."
- Data: The year options come from a transformer `tfrmTaxYearOptions` (2025, 2026, current year if later). All tax-year queries bind to the selected year via `tfrmTaxYearRange` computing `date >= Jan 1 AND date < Jan 1 of next year`.
- Logic notes: A transaction on Dec 31 belongs to the outgoing year; Jan 1 to the incoming year — the boundary note prevents confusion. The default selection is the current calendar year. Changing years never mutates data, only the view filters.
- Acceptance: (1) 2025 and 2026 are selectable. (2) Boundary note is always visible next to the picker. (3) Dec 31 vs Jan 1 transactions land in the correct years. (4) Default selection is the current year.

### 807. Add estimated quarterly tax payment tracking as bills.
- Components: On the tax screen, a card list `listQuarterlyTax` showing Q1–Q4 rows for the selected tax year with due date, amount input, and a Paid checkbox; an "Add to Bills" toggle creates corresponding entries in the bills flow (§15).
- Data: CRUD on `estimated_tax_payments` via `qryEstimatedTax` (select), `qryUpsertEstimatedTax` (insert/update), `qryToggleEstimatedPaid` (update). A transformer `tfrmQuarterDefaults` seeds Q1–Q4 due dates (Apr 15, Jun 15, Sep 15, Jan 15 of next year) with blank amounts.
- Logic notes: Amounts are user-entered estimates — the app never computes a required payment. Marking Paid records the payment date. These are planning reminders, not IRS forms; the disclaimer (item 811) applies here too.
- Acceptance: (1) Four quarter rows render for the selected year with correct default due dates. (2) Amounts persist per quarter. (3) Paid toggle records state. (4) Nothing computes or suggests a payment amount.

### 808. Display charitable giving totals from tagged transactions.
- Components: Stat card `statCharitable` on the tax-year view showing total charitable giving for the year, with a drill-down table `tblCharitable` listing each tagged transaction (date, description, amount, note).
- Data: Transformer `tfrmCharitable` filters tax-year transactions to `tax_tag = 'charitable'`, summing absolute spend amounts (respecting the sample toggle). The drill-down reuses the same filtered set sorted by date.
- Logic notes: Only spend rows (negative amounts) count; any income row accidentally tagged charitable is listed but excluded from the total with a warning flag. Totals display as positive dollars ("gave $X").
- Acceptance: (1) Total equals sum of charitable-tagged spend for the year. (2) Drill-down lists exactly those rows. (3) Sample-hidden excludes samples. (4) Income rows tagged charitable are flagged, not counted.

### 809. Show medical expense totals vs the AGI-threshold concept as an informational note.
- Components: Stat card `statMedical` with the medical-tagged total, beneath it an info callout `calloutAgiNote`: "Tax rules may only allow deducting medical expenses above a percentage of AGI — check current rules with a professional."
- Data: Transformer `tfrmMedical` sums `tax_tag = 'medical'` spend rows for the year. No AGI input is required; the card does not compute any threshold — it only shows the user's total plus the informational note.
- Logic notes: Deliberately no percentage math and no AGI field — computing thresholds would cross into tax advice. The note is static text, dated ("rules change; verify for the current tax year"). Keeps item 811's disclaimer visible.
- Acceptance: (1) Medical total matches tagged rows. (2) Callout text is present and static. (3) No threshold calculation or AGI input exists anywhere on the card. (4) Disclaimer visible on the screen.

### 810. Add a CPA-share view: read-only tax packet link with expiry.
- Components: Button `btnShareTaxPacket` opens `modalCpaShare` with an expiry select (7 / 14 / 30 days) and a "Generate link" button; the generated URL + expiry date display in a read-only field with a copy button. A list `listTaxShares` shows active links with a Revoke button each.
- Data: `qryCreateTaxShare` inserts into `tax_shares` (tax_year, random `share_token`, expires_at); the shared view is a public Retool page rendering the tax packet read-only (no edit controls, no sample-mode toggle — packet content fixed at generation). `qryRevokeTaxShare` deletes the row.
- Logic notes: The link exposes only the packet snapshot, never the full app or edit access. Expired tokens return a "link expired" page. Revocation is immediate. Share tokens are unguessable random strings, never sequential ids.
- Acceptance: (1) Link works read-only with no edit controls. (2) Expired link shows an expired message. (3) Revoke kills access immediately. (4) Active shares list shows expiry dates.

### 811. Clearly disclaim "not tax advice — review with a professional" on tax screens.
- Components: A shared Text component `txtTaxDisclaimer` ("Not tax advice — review with a professional.") placed at the top of the tax-year view, the packet modal, the CPA-share modal, and the quarterly-tax card; styled as a persistent info banner, not dismissible.
- Data: None — static text. A builder checklist query is unnecessary; this is enforced by component placement on each tax screen.
- Logic notes: The disclaimer must survive screen redesigns — document it as a required element in the tax section's builder notes. Wording is fixed and identical everywhere to avoid implying different levels of advice on different screens.
- Acceptance: (1) Disclaimer visible on all four tax surfaces without scrolling. (2) Identical wording on each. (3) Not dismissible or hidden behind a toggle. (4) Present in the exported packet PDF footer.

### 812. Track 1099-style income sources separately with payer notes.
- Components: In the add/edit transaction form, when category = Income, a "Payer" text input `txtPayer` and a toggle `tglIs1099` ("1099-style income") appear. The tax-year view gains a `tblPayerIncome` table grouping income by payer with totals.
- Data: `payer` and a `is_1099` boolean (add to the `transactions` migration alongside `tax_tag`) persist via the transaction form queries. Transformer `tfrmPayerIncome` groups tax-year income rows by payer, summing amounts; rows without a payer group under "Unspecified".
- Logic notes: "1099-style" is the user's own label for tracking — the app does not generate or validate any 1099 form. Payer names are free text; duplicates from typos are the user's to clean (show rows, not merged magic).
- Acceptance: (1) Payer + 1099 toggle save on income transactions. (2) Payer table totals match income rows. (3) Unspecified group exists for blank payers. (4) No 1099 form or filing is generated.

### 813. Add mileage log entries linked to Transport transactions (manual entry).
- Components: "Mileage log" section on the tax screen: an "Add entry" button opens `modalMileage` with Date Picker, Number Input `numMiles`, Text Input `txtMileagePurpose`, and an optional Select `selMileageTxn` of Transport-category transactions to link. Table `tblMileage` lists entries with edit/delete.
- Data: CRUD on `mileage_log` via `qryMileageLog` (select by year), `qryAddMileage`, `qryDeleteMileage`. Linking sets `transaction_id`; a transformer shows the linked transaction description. All manual — no GPS, no auto-tracking.
- Logic notes: Miles are user-entered numbers; the app never multiplies by a mileage rate (that would be tax advice territory). Year filter follows the selected tax year. Deleting a linked transaction nulls the link rather than deleting the log entry.
- Acceptance: (1) Entries persist with date, miles, purpose. (2) Optional transaction link displays its description. (3) Year filter follows the tax-year picker. (4) No rate calculation or deduction computation exists.

### 814. Show home-office expense tagging with square-footage notes.
- Components: When `selTaxTag` = "Home office" on a transaction, `txtTaxNote` is joined by an optional Number Input `numSqft` ("Office sq ft (optional note)") in the tax section of the detail drawer. The tax-year view's deduction list shows the sq-ft note inline for home-office rows.
- Data: Square footage stores inside `tax_note` as a prefixed annotation (e.g. "sqft:120; supplies") or a dedicated nullable column — builder's choice, documented in the migration. No computation reads it.
- Logic notes: The sq-ft figure is a memory aid for the user's own records, never an input to a deduction formula — the app does not compute home-office deductions. Keep it clearly labeled as a note, not a calculation input.
- Acceptance: (1) Sq-ft note saves and displays on home-office rows. (2) No formula consumes the sq-ft value anywhere. (3) Label reads as a note ("optional note"). (4) Deduction list shows the annotation.

### 815. Support splitting a transaction into deductible and non-deductible parts.
- Components: In the transaction detail drawer's tax section, a "Split" button `btnTaxSplit` opens `modalTaxSplit` with two Number Inputs (deductible amount, non-deductible amount, must sum to the transaction's absolute amount) and a note field. The transaction row shows a "split" badge.
- Data: `qryUpsertTaxSplit` writes to `tax_splits` (one row per transaction, upsert by transaction_id); `qryDeleteTaxSplit` removes it. Transformers `tfrmDeductibleTotal` (items 802/803) prefer `tax_splits.deductible_amount` when a split row exists.
- Logic notes: Validation blocks saving unless the two parts sum to the full amount (within a cent). Only one split per transaction; re-splitting overwrites. Sample flag on the split inherits the transaction's sample status.
- Acceptance: (1) Parts must sum to the transaction total or save is blocked. (2) Tax totals use the deductible part. (3) Split badge shows on the row. (4) Removing the split restores full-amount behavior.

### 816. Display prior-year comparison for income and deductions.
- Components: On the tax-year view, a comparison strip `cmpPriorYear` with two columns: selected year vs prior year, each showing income total and tagged-deduction total, plus the absolute and percentage change.
- Data: Transformer `tfrmPriorYearCompare` runs the same income/deduction aggregations as item 802 for `selectedYear - 1`, reusing the sample-toggle logic. Change computed as `(current - prior) / |prior|`, guarded against divide-by-zero (shows "—" when prior is 0).
- Logic notes: Prior year is always exactly one year back — no multi-year picker in this item. If the prior year has no data, show zeros with an "no data" caption rather than hiding the column. Percentage change is informational display only.
- Acceptance: (1) Both columns' totals match the tax-year aggregations. (2) Prior year = selected year − 1. (3) Zero prior data shows zeros, not errors. (4) Sample toggle applies to both columns.

### 817. Add a tax-deadline reminder set (Jan 31, Apr 15) as custom reminders.
- Components: In the reminders area (§29 pattern), a "Tax deadlines" preset button `btnTaxDeadlinePreset` creates two custom reminders: "Jan 31 — tax documents due" and "Apr 15 — tax filing deadline", each editable like any other reminder.
- Data: Reuses the reminders storage from §29; the preset inserts two rows with annual recurrence and default lead-time (e.g. 7 days before). They appear in the same reminder list and notification flow as user-created reminders.
- Logic notes: Deadlines are common U.S. dates presented as planning aids, not legal advice — the reminder text says "check your situation with a professional" per item 811. Users can edit dates, rename, or delete. No auto-creation without the user tapping the preset.
- Acceptance: (1) Preset creates exactly the two reminders. (2) They behave like normal reminders (edit/delete/notify). (3) Reminder text includes the check-with-a-professional nudge. (4) Nothing is created without user action.

### 818. Export the tax packet with the manual-data banner and sample exclusions.
- Components: The packet export modal (item 805) includes a fixed, non-removable header block on every page: the "Manual tracking — no bank connection yet." banner plus a line stating the export mode ("Sample data excluded" or "Sample data included").
- Data: `qryBuildTaxPacket` injects the banner block and mode line into the PDF header, and stamps the `tax_documents` row with the mode used. The mode follows the current sample toggle — no separate export-time toggle that could silently flip it.
- Logic notes: The banner must appear in the PDF itself, not just on screen — a recipient of the file (e.g. a CPA) sees the manual-data context without opening the app. If samples are included, each sample row keeps a SAMPLE label in the packet tables.
- Acceptance: (1) Every PDF page header shows the manual-tracking banner. (2) Mode line matches the actual data included. (3) `tax_documents` row records the mode. (4) Included samples are labeled SAMPLE in the packet.

### 819. Allow marking the tax year as "filed" to lock the packet snapshot.
- Components: On the tax-year view, a "Mark as filed" button `btnMarkFiled` (with a confirm modal warning the action locks the year's packet) and a `badgeFiled` indicator showing FILED + filed date once set. An "Unfile" option exists behind a second confirmation.
- Data: `qryMarkYearFiled` upserts `tax_years` (year, status='filed', filed_at=now, packet_snapshot=JSON of the current packet datasets). While filed, the packet export regenerates from the snapshot, not live data; `qryUnfileYear` flips status back to open (snapshot retained for reference).
- Logic notes: "Lock" means the packet content is frozen — tagging new transactions for a filed year does not change the filed packet (they appear in the live view with a "filed year" warning). This models the real-world "already filed" state without pretending to file anything.
- Acceptance: (1) Filing freezes the packet snapshot. (2) Post-file tagging doesn't alter the filed packet. (3) Live view shows a "filed year" warning when editing tags. (4) Unfile requires confirmation and restores live packet generation.

### 820. Log tax-tag changes in the audit log.
- Components: No new UI — the audit log screen (§38) gains a filter option for "Tax tags" and renders these entries like other change events (timestamp, transaction description, old → new tag, actor).
- Data: `qryUpdateTaxTag`, `qryUpsertTaxSplit`, and the filed/unfile queries each fire an audit insert via the §38 audit pattern (table from §38: event_type, entity, entity_id, before, after, created_at). Event types: `tax_tag_set`, `tax_tag_cleared`, `tax_split_set`, `tax_year_filed`, `tax_year_unfiled`.
- Logic notes: Log the tag value change, not the full transaction row — keeps the audit readable. Reads (viewing the tax screen, exporting the packet) are not logged, only mutations. Sample and non-sample rows log identically.
- Acceptance: (1) Setting/clearing a tag writes an audit entry with old → new values. (2) Filing/unfiling a year writes entries. (3) Audit filter "Tax tags" isolates these events. (4) Viewing/exporting writes nothing.
