# FinanceOS Builder Spec — 04. Add Transaction Form (Items 61–80)

## Section title
Add transaction form — the single manual-entry path for all FinanceOS transactions, usable one-handed on iPhone and fast on desktop.

## Tables touched
- **transactions** (existing): `id (pk), account_id (fk), date (date), description (text), category (text), amount (number, negative=spend), is_sample (bool)` — **add column `notes (text)`** for item 71.
- **audit_log** (new): `id (pk, auto), actor (text), action (text), table_name (text), record_id (number), before (jsonb, nullable), after (jsonb, nullable), entry_method (text: manual/import/sample/quick-add), created_at (timestamp, default now)`. Append-only; no UI edit/delete.
- **budgets**, **accounts** (existing) are read for budget warnings (76) and prefill (70).

## Cross-cutting rules
- Every screen, modal, and drawer in this flow renders the persistent banner: **"Manual tracking — no bank connection yet."** (item 18 spec places the exact component; this file references `banner_manualTracking` by name).
- Every row the user creates carries `is_sample=false` unless created from a sample-data template, which sets `is_sample=true` plus a visible SAMPLE pill everywhere the row renders.
- The global "hide sample data" toggle filters `is_sample=true` rows out of all totals, charts, and budgets. Nothing in this flow may ever present data as a live bank feed.

---

### 61. Build an add-transaction form with account picker, date, description, category, and signed amount.
- **Components:** Modal `modal_addTx` (also opened as full-screen bottom sheet on mobile per item 74). Inside: Select `tx_account` (populated from `query_accounts` — label `name (institution)`, value `id`), Date picker `tx_date`, Text input `tx_description` (maxLength 200), Select `tx_category`, Number input `tx_amount`, Toggle/Segmented control `tx_sign` (Spend/Income, item 63), primary Button `btn_saveTx`, secondary Button `btn_saveAddAnother` (item 67).
- **Data:** Query `query_accounts` (Retool DB: `SELECT id, name, institution, type FROM accounts WHERE is_sample={{!tgl_hideSamples.value}} OR true ORDER BY name`) — note the sample toggle simply re-runs it. Mutation `mut_insertTx` (Retool DB insert into `transactions` with fields from form + `is_sample` per item 69).
- **Logic notes:** Amount is signed on save: `tx_sign.value === 'income' ? +amt : -amt`. Form disabled until validation passes (items 65, 68).
- **Acceptance:** (1) All five fields render and the save writes one row with correct signed amount. (2) Modal opens from overview quick-add, transactions screen, and palette. (3) Banner text visible above the modal content.

### 62. Default new transactions to today's date with a one-tap date picker.
- **Components:** Date picker `tx_date` with `defaultValue={{ moment().format('YYYY-MM-DD') }}`, `enableClear=false`, `showTodayButton=true`.
- **Data:** No query; `tx_date.formattedValue` feeds `mut_insertTx.date`.
- **Logic notes:** "Today" is computed in the user's browser timezone at modal open; reopening the modal re-stamps it. One tap on the "Today" chip resets to today after the user picked another date. Future dates are allowed (item 72).
- **Acceptance:** (1) Fresh-opened form shows today's date. (2) Tapping the date field opens a calendar in one tap, no nested dialogs. (3) Selecting another date then tapping Today returns to today.

### 63. Provide a spend/income toggle that sets the amount sign automatically.
- **Components:** SegmentedControl `tx_sign` with options Spend / Income (default Spend), placed directly above `tx_amount`.
- **Data:** `mut_insertTx` payload amount = `Math.abs(tx_amount.value) * (tx_sign.value==='income' ? 1 : -1)`.
- **Logic notes:** Switching the toggle does NOT change the digits the user typed — only the sign. The live preview (item 66) updates instantly. Category default flips to Income when the toggle is Income and the current category is a spend category (user can override).
- **Acceptance:** (1) Enter 42.50 with Spend → saved as -42.50; with Income → 42.50. (2) Preview row and amount display flip sign color (red/green) immediately on toggle.

### 64. Restrict the category dropdown to Housing, Food, Transport, Income, Fun, Bills, and Other.
- **Components:** Select `tx_category` with `values=["Housing","Food","Transport","Income","Fun","Bills","Other"]`, `defaultValue="Food"` (or item-70 prefill).
- **Data:** Static option list duplicated from the shared constant `CONST_CATEGORIES` (defined in the app's preloaded JS) so every dropdown and chart in the app stays in sync.
- **Logic notes:** No free-text entry; no "Uncategorized" option — truly unknown spend goes in Other (nudged per item 126 spec). Category icons/colors come from the shared map in item 122's spec.
- **Acceptance:** (1) Dropdown shows exactly the 7 categories, in that order. (2) No way to save a transaction with a category outside the list (DB check constraint recommended).

### 65. Require an account selection before the form can be submitted.
- **Components:** `tx_account` with `required=true`, placeholder "Choose account…"; `btn_saveTx` and `btn_saveAddAnother` with `disabled={{ !tx_account.value }}`.
- **Data:** None — client-side gate.
- **Logic notes:** On attempted submit without an account, show inline error "Pick an account first" and focus the picker. This is the app's only required-field hard gate besides amount validation.
- **Acceptance:** (1) Save buttons stay disabled until an account is picked. (2) Clearing the account re-disables save. (3) Error message appears on premature submit attempt.

### 66. Show a live preview row of how the transaction will appear in the table.
- **Components:** Container `preview_txRow` at the top of the modal, styled exactly like a row of the transactions table: date chip, description, category pill, account chip, signed amount (red/green), SAMPLE pill when applicable.
- **Data:** Transformer `tf_txPreview` returning `{date, description, category, account: tx_account.selectedLabel, amount: signed amount, is_sample}` from form values; re-evaluates on every keystroke.
- **Logic notes:** Preview mirrors the table row component (item 81–84) markup so users learn what they will see. Shows "—" for empty description.
- **Acceptance:** (1) Typing updates the preview within one keystroke. (2) Preview amount sign/color matches the spend/income toggle. (3) Preview matches the actual table row after save.

### 67. Add a "save and add another" button for rapid manual entry of receipts.
- **Components:** Secondary Button `btn_saveAddAnother` ("Save & add another") beside the primary save.
- **Data:** Runs `mut_insertTx`, then on success: keeps modal open, clears description/amount/notes, keeps account + category + date (item 70 behavior), focuses `tx_description`.
- **Logic notes:** Debounce double-clicks; show the confirmation toast (item 73) per saved row. Counter "3 saved this session" shown in modal header for receipt-batch feedback.
- **Acceptance:** (1) One click saves and resets for the next entry without closing. (2) Account/category persist across saves in the session. (3) Double-click does not create duplicates (button disabled while mutation runs).

### 68. Validate that amounts are non-zero numbers before saving.
- **Components:** `tx_amount` (Number input, `allowNegative=false` — sign comes from the toggle), inline error text `err_amount`.
- **Data:** Validation rule `tx_amount.value !== null && !isNaN(tx_amount.value) && Math.abs(tx_amount.value) > 0`; save buttons disabled otherwise.
- **Logic notes:** Reject 0, empty, and non-numeric input with message "Enter an amount greater than $0." Reject absurd values (> $10,000,000) with a confirm step. Negative typed digits are auto-converted to absolute (sign is toggle-owned).
- **Acceptance:** (1) 0, blank, or letters block save with a clear message. (2) Valid amounts save. (3) Extremely large amounts trigger the confirm step.

### 69. Flag the new transaction with is_sample when it is created from a sample-data template.
- **Components:** In the "Use sample template" picker (item 344's first-run flow and item 881 templates), each template carries `sample_template=true`; the add form opened from one shows an amber SAMPLE ribbon.
- **Data:** `mut_insertTx` sets `is_sample = {{ sampleTemplateActive.value ? true : false }}`; SAMPLE pill renders on the preview row (item 66).
- **Logic notes:** Normal manual entry always writes `is_sample=false`. The sample ribbon must be unmistakable so real money is never accidentally labeled sample.
- **Acceptance:** (1) Creating from a sample template writes `is_sample=true` and shows the pill. (2) Ordinary manual entry writes `is_sample=false`. (3) Global sample toggle hides/shows these rows in all totals.

### 70. Prefill the form from the last transaction's account and category to speed repeat entries.
- **Components:** `tx_account` / `tx_category` `defaultValue` bound to transformer `tf_lastTxPrefill`.
- **Data:** Query `query_lastTx` (`SELECT account_id, category FROM transactions WHERE is_sample=false ORDER BY created_at DESC LIMIT 1`); `tf_lastTxPrefill` maps to the pickers. Runs on modal open.
- **Logic notes:** Only real (non-sample) transactions seed the prefill. If the user has zero transactions, fall back to first account / "Food". Explicit user changes are never overwritten by a re-run.
- **Acceptance:** (1) After saving a Food/Chase transaction, the next open prefills those. (2) First-ever use falls back cleanly. (3) Changing the prefilled values sticks.

### 71. Add a notes field on the transaction form for memos like "birthday dinner."
- **Components:** Text area `tx_notes` (maxLength 500, 2 rows, placeholder "Memo (optional)"), below description.
- **Data:** New column `transactions.notes (text)`; `mut_insertTx` includes `notes: tx_notes.value || null`.
- **Logic notes:** Notes are searchable (item 117) and included in CSV exports (item 435). Row shows a notes indicator when non-empty (item 94).
- **Acceptance:** (1) Memo saves with the row and re-opens on edit (item 82). (2) Empty memo saves as null, no error. (3) Table shows the notes indicator (item 94) when a memo exists.

### 72. Support future-dated transactions for planned spending, marked with a "scheduled" tag.
- **Components:** When `tx_date.value > today`, the preview row (item 66) and saved table row show a "scheduled" tag chip.
- **Data:** No schema change; tag is computed: `moment(tx_date.value).isAfter(moment(), 'day')`. Query `query_upcomingScheduled` lists future-dated rows for the cash-flow calendar (item 301).
- **Logic notes:** Future-dated rows are EXCLUDED from month-to-date actuals (item 161) and budgets until their date arrives; they are INCLUDED in projections (items 301–311). Editing the date to today/past removes the tag automatically.
- **Acceptance:** (1) Tomorrow-dated entry saves with the scheduled tag. (2) It does not inflate this month's budget actuals. (3) It appears in the cash-flow calendar's planned items.

### 73. Show a confirmation toast with an undo button after each manual save.
- **Components:** Toast via `utils.showNotification` (or a custom `toast_confirm` frame) with message "Saved — $42.50 Food · Chase" and an Undo button, 8-second duration.
- **Data:** Undo runs `mut_deleteTx` on the just-inserted id; the id is captured from `mut_insertTx.data.id`. Audit log records the delete as a paired entry (item 80/741).
- **Logic notes:** Undo is only available from the toast; after it expires the normal delete flow (item 83) applies. "Save & add another" shows a compact toast per row so rapid entry isn't blocked.
- **Acceptance:** (1) Toast appears after every save with correct summary. (2) Undo within the window removes the row and logs it. (3) Toast does not steal focus from the next entry.

### 74. Make the form fully usable one-handed on iPhone with large touch targets.
- **Components:** On viewport ≤ 430px, `modal_addTx` renders as a full-height bottom sheet; all inputs/buttons `min-height: 48px`; primary save is a sticky bottom bar; fields are single-column, labels above inputs.
- **Data:** None — CSS/media-query config in the app's custom CSS: `@media (max-width:430px){ .addTx-sheet{...} }`.
- **Logic notes:** Thumb-reachable: save button within the bottom third; date picker opens the native iOS picker; amount opens the numeric keyboard (`inputMode="decimal"`). No horizontal scrolling at 390px (item 381).
- **Acceptance:** (1) At 390px width the form is single-column with no horizontal scroll. (2) Every touch target ≥ 44px. (3) Save reachable without scrolling on a standard entry.

### 75. Add haptic-friendly amount keypad layout with quick amounts ($5, $10, $20, $50).
- **Components:** On mobile, `tx_amount` gets an accessory row of quick chips `$5 $10 $20 $50` plus a Clear chip; on desktop the same chips render beside the amount input.
- **Data:** Tapping a chip sets `tx_amount.setValue(preset)`; tapping again adds (5 → 10) — single tap sets, documented in the field hint.
- **Logic notes:** Chips append rather than replace on repeat taps within 2 seconds (for $5+$5=$10), otherwise replace. Visual press feedback (scale 0.96) substitutes for native haptics (item 392).
- **Acceptance:** (1) Tapping $20 fills 20.00. (2) Amount keypad opens the numeric keyboard on iPhone. (3) Chips work in both Spend and Income modes.

### 76. Warn when a transaction would push a category over its monthly budget.
- **Components:** Inline warning banner `warn_overBudget` above the save buttons: "This puts Food $18 over its $300 budget — save anyway?" Save stays enabled.
- **Data:** Query `query_catMonthActual` (`SELECT COALESCE(SUM(ABS(amount)),0) FROM transactions WHERE category={{tx_category.value}} AND date_trunc('month',date)=date_trunc('month',{{tx_date.value}}::date) AND amount<0 AND <sample filter>`); transformer compares `actual + abs(newAmount)` vs `query_budgetLimit` for that category/month.
- **Logic notes:** Warning only — never blocks saving (manual tracking must not fight the user). Only fires when a budget limit exists for the category/month. Skipped for Income category and future-dated rows.
- **Acceptance:** (1) Entry that crosses the limit shows the warning with the exact overage. (2) Entry under the limit shows nothing. (3) Saving with the warning visible succeeds.

### 77. Allow duplicating an existing transaction into a prefilled form.
- **Components:** "Duplicate" row action on the transactions table opens `modal_addTx` with all fields copied from the source row (new id, today's date, is_sample copied from source).
- **Data:** Transformer `tf_duplicateTx` maps the selected row's fields into the form state; `mut_insertTx` then saves it as a new row.
- **Logic notes:** Amount sign → sets the spend/income toggle accordingly. Notes and tags copy over. Audit log notes `duplicated_from` record id (item 740/80).
- **Acceptance:** (1) Duplicate opens the form fully prefilled, nothing saved yet. (2) Saving creates a distinct new row. (3) is_sample flag matches the source row.

### 78. Add keyboard shortcuts for desktop entry (N for new, S to save).
- **Components:** App-level event handlers: `keydown` listener registered on the Transactions screen (and globally on desktop widths) — `N` opens `modal_addTx`, `S` (or Cmd/Ctrl+Enter) triggers save when the modal is open, `Esc` closes.
- **Data:** None.
- **Logic notes:** Shortcuts are suppressed while typing in text inputs (except Esc and Cmd/Ctrl+Enter). A cheatsheet lives in Help (item 909/974). Never hijack browser-reserved keys.
- **Acceptance:** (1) Pressing N with the modal closed opens it (focus not in a text field). (2) Cmd/Ctrl+Enter saves from the modal. (3) Shortcuts do nothing while typing a description.

### 79. Show the running month total for the chosen category while filling the form.
- **Components:** Small line under `tx_category`: "Food · $212.40 of $300 budget (71%)" or "Food · $212.40 this month" when no budget.
- **Data:** Reuses `query_catMonthActual` (item 76) + `query_budgetLimit`; transformer `tf_catMonthLine` formats the string.
- **Logic notes:** Updates as category or date changes. Respects the sample toggle. For Income category shows total income this month. "This month" follows the transaction's date, not always today (so backdated entries show the right month).
- **Acceptance:** (1) Line updates on category/date change. (2) Numbers match the budgets screen for the same category/month. (3) Hides gracefully when no transactions exist yet.

### 80. Log every manual transaction creation in the audit log with timestamp.
- **Components:** None visible; `mut_insertTx` onSuccess triggers `mut_auditLog`.
- **Data:** New table `audit_log` (schema in header). Mutation `mut_auditLog` inserts `{actor: current_user.email, action:'create', table_name:'transactions', record_id: new id, before:null, after:<full row JSON>, entry_method:'manual'|'quick-add'|'sample', created_at: now()}`. Duplicate-flow sets `entry_method` accordingly (item 918 tags palette-driven writes 'quick-add').
- **Logic notes:** Audit writes are fire-and-forget after the transaction commits — an audit failure must never roll back the user's save (log to console + retry once). Viewer spec is item 743.
- **Acceptance:** (1) Every created transaction has a matching audit row with timestamp. (2) The audit row's `after` JSON equals the saved row. (3) Audit failure does not block or lose the transaction.
