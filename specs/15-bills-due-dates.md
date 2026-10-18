# FinanceOS Builder Spec — Section 15: Bills & due dates (items 281–300)

## Section overview

Adds a full Bills module to FinanceOS: a bills screen with create/edit/pause flows, paid-vs-unpaid state per month, reminders, variable amounts, housemate splits, history, attachments, a calendar view, amount-change tracking, late flags, and CSV/print export. Bills are recurring obligations entered manually; "mark paid" writes a real manual transaction into `transactions` (category Bills/Housing/Other, negative amount). Paid state is derived per month from matching transactions — never auto-posted, never synced from a bank.

## Tables touched

- **bills** (NEW): id (pk, serial), name (text), category (text: Bills / Housing / Other), amount (number, estimated default for variable bills), is_variable (bool), due_day (int, 1–31), account_id (fk → accounts.id), status (text: Active / Paused), autopay_tag (text: Manual / Autopay-reported, default Manual), paused_at (timestamp, nullable), is_sample (bool), created_at (timestamp)
- **bill_payments** (NEW): id (pk, serial), bill_id (fk → bills.id), month (text, YYYY-MM), transaction_id (fk → transactions.id, nullable), paid_late (bool, default false), late_fee_transaction_id (fk → transactions.id, nullable), recorded_at (timestamp), is_sample (bool)
- **bill_amount_history** (NEW): id (pk, serial), bill_id (fk → bills.id), old_amount (number), new_amount (number), changed_at (timestamp), is_sample (bool)
- **bill_splits** (NEW): id (pk, serial), bill_id (fk → bills.id), housemate_label (text), my_share (number), is_sample (bool)
- **bill_attachments** (NEW): id (pk, serial), bill_id (fk → bills.id), file_name (text), file_url (text, Retool Storage link), uploaded_at (timestamp), is_sample (bool)
- **transactions** (existing): mark-paid and late-fee writes insert rows here with category from the bill and `is_sample` inherited from the bill.
- **accounts** (existing): read for balance-check warnings.

## Cross-cutting rules

1. The persistent banner "Manual tracking — no bank connection yet." appears at the top of the Bills screen, the calendar view, and any print/export output.
2. Every seeded bill/payment/attachment row carries `is_sample=true` and a visible SAMPLE badge; the global sample toggle excludes them from all totals and tiles.
3. Nothing is auto-posted: reminders and due states only prompt the user; every transaction is created by an explicit user action and logged.
4. Autopay is a user-reported tag only — the app performs no automatic payments and must never imply otherwise.

---

### 281. Build a bills screen listing recurring obligations with name, amount, and due day.

- **Components:** New "Bills" screen reachable from the More menu and bottom tab overflow. Layout: header (banner + "Add bill" button + 30-day cash requirement tile), `tblBills` Table (columns: name, category pill, amount, due day "15th", account chip, paid-this-month badge, status), `inpBillSearch` TextInput filter.
- **Data:** `qryBills` — `SELECT b.*, a.name AS account_name FROM bills b LEFT JOIN accounts a ON a.id = b.account_id WHERE b.status='Active' ORDER BY b.due_day`. Sample rows seeded: Rent $1,450 due 1st, Electric (variable) $120 est. due 12th, Phone $65 due 20th, Internet $80 due 5th — all `is_sample=true`.
- **Logic notes:** Due day stored as 1–31; months shorter than the due day clamp to month-end (bill with due_day 31 is due Feb 28/29). Paid badge computed by `qryPaidThisMonth` joining `bill_payments` on current `YYYY-MM`.
- **Acceptance:** (1) Screen lists all Active bills with name, amount, due day visible without scrolling on iPhone. (2) Banner renders pinned at top. (3) SAMPLE badge shows on seeded rows. (4) Search filters by name instantly.

### 282. Add a create-bill form: name, category (Bills/Housing/Other), amount, due day, account.

- **Components:** `modBillForm` Modal (bottom sheet on iPhone): `txtBillName` TextInput (required), `selBillCategory` Select (Bills / Housing / Other, default Bills), `numBillAmount` NumberInput (required, > 0), `chkVariable` Checkbox ("Variable amount — this is an estimate"), `numDueDay` NumberInput (1–31, required), `selBillAccount` Select from `qryAccounts` (required), `selAutopayTag` Select (Manual / Autopay-reported), Save/Cancel buttons (44px targets).
- **Data:** `qryCreateBill` — `INSERT INTO bills (name, category, amount, is_variable, due_day, account_id, autopay_tag, status, is_sample) VALUES (...)` with `is_sample=false` for user rows. On success: close modal, refresh `qryBills`, toast "Bill added".
- **Logic notes:** Validation: name non-empty, amount > 0 numeric, due_day integer 1–31, account selected. Category restricted to the three values; amount stored as the estimate when `is_variable` is true.
- **Acceptance:** (1) Form rejects empty name, zero/negative amount, due day 0 or 32 with inline errors. (2) New bill appears in the list immediately. (3) `is_sample=false` on user-created rows. (4) Modal usable one-handed on 390px width.

### 283. Show upcoming bills sorted by due date with days-remaining chips.

- **Components:** `lstUpcomingBills` ListView (or grouped section above `tblBills`) driven by `qryUpcomingBills`, each row: name, amount, chip (`chipDaysLeft` Container styled as pill).
- **Data:** Transformer `tUpcomingBills`: for each Active bill, compute next due date = this month's due_day if today ≤ due date else next month's (clamped to month-end); days_remaining = ceil((next_due − today)/86400000). Sort ascending by next due date.
- **Logic notes:** A bill due today shows "Due today"; due tomorrow "1 day left". Overdue-but-unpaid bills (past due date, no payment this month) appear in the overdue section (§289), not here. Clamp rule: due_day 31 → Feb shows 28/29.
- **Acceptance:** (1) List sorted strictly by next due date ascending. (2) Chip text matches computed days remaining for at least 3 sample bills. (3) Changing device date (test) re-sorts correctly. (4) Chips use amber for ≤3 days, neutral otherwise.

### 284. Add a "mark paid" action that creates the matching manual transaction.

- **Components:** Per-row `btnMarkPaid` Button in `tblBills` (and in the upcoming list). Opens `modConfirmPaid`: shows bill name, amount (editable `numPaidAmount` prefilled from bill amount for variable bills), account (prefilled from bill), date (default today), optional "paid late" checkbox → Confirm writes.
- **Data:** `qryMarkPaid`: (1) `INSERT INTO transactions (account_id, date, description, category, amount, is_sample) VALUES (account, today, 'Bill: ' || name, bill.category, -amount, bill.is_sample)`; (2) `INSERT INTO bill_payments (bill_id, month, transaction_id, paid_late, is_sample)` with month = `YYYY-MM` of payment date; (3) update the account's balance and `updated_at`. All three in one Retool multi-statement query; on failure, roll back with error toast and log to audit.
- **Logic notes:** Amount stored negative (spend). Description prefixed "Bill: " for traceability. One payment per bill per month is the norm — if a payment already exists for the month, confirm dialog warns "Already marked paid this month" and requires explicit override.
- **Acceptance:** (1) Mark-paid creates exactly one transaction and one bill_payments row. (2) Account balance decreases by the paid amount. (3) Bill shows paid badge for the month immediately. (4) Duplicate-month attempt triggers the override warning.

### 285. Display paid vs unpaid state per bill for the current month.

- **Components:** `colPaidStatus` custom column in `tblBills`: green "Paid ✓" pill vs gray "Unpaid" pill; overview-style summary strip `txtBillsSummary`: "4 of 6 bills paid · $1,730 of $2,015".
- **Data:** `qryPaidThisMonth` — `SELECT bill_id FROM bill_payments WHERE month = to_char(CURRENT_DATE,'YYYY-MM')`. Transformer `tBillStatus` left-joins bills to this set.
- **Logic notes:** Paid state is strictly per calendar month; a bill paid on the 31st counts for that month. Paused bills (§292) are excluded from the unpaid count. State recomputes on every `qryBills`/`qryPaidThisMonth` refresh (after mark-paid, undo, or month change).
- **Acceptance:** (1) After mark-paid, pill flips to Paid without reload. (2) On the 1st of a new month all pills reset to Unpaid (test by overriding date in preview). (3) Summary strip numbers equal the sum of paid vs total active bills. (4) Sample bills excluded when sample toggle is off.

### 286. Support bills with variable amounts (e.g., utilities) with an estimated default.

- **Components:** `chkVariable` Checkbox on the bill form (§282); list shows "~$120" prefix tilde for variable bills; `modConfirmPaid` pre-fills `numPaidAmount` with the estimate but leaves it editable.
- **Data:** `bills.is_variable` bool; `bills.amount` holds the estimate. `qryUpdateEstimate` lets the user update the estimate from the bill detail drawer; writes a `bill_amount_history` row (old vs new) per §298 when the estimate changes.
- **Logic notes:** Estimates feed the 30-day cash requirement (§297) and upcoming list; actual paid amounts (from mark-paid) feed history (§291) and monthly totals (§287). Never silently replace an estimate with an actual — both are stored.
- **Acceptance:** (1) Variable bills display with "~" prefix everywhere. (2) Mark-paid allows entering an actual different from the estimate. (3) Changing the estimate logs an amount-history row. (4) Cash-requirement tile uses estimates for variable bills.

### 287. Show a monthly bills total and its share of recorded income.

- **Components:** `tileBillsTotal` Statistic tile on the Bills screen header: "Bills this month $2,015" + subtext "38% of recorded income ($5,300)".
- **Data:** Transformer `tBillsTotal`: sum of Active bill amounts (estimates for variable) for the current month. `qryMonthIncome` — `SELECT COALESCE(SUM(amount),0) FROM transactions WHERE category='Income' AND date_trunc('month',date)=date_trunc('month',CURRENT_DATE)` respecting the sample toggle. Share = total / income (guard divide-by-zero → "—").
- **Logic notes:** Uses scheduled amounts (what's owed), not paid amounts — labeled "scheduled". If income is 0, show total with "no recorded income yet" instead of a percent. Recompute on bill or transaction changes.
- **Acceptance:** (1) Total equals sum of active bill amounts for the month. (2) Percent = total ÷ recorded Income-category total, rounded to whole %. (3) Zero-income month shows the fallback text, not NaN/Infinity. (4) Sample toggle changes both numbers consistently.

### 288. Add bill reminders as in-app notifications a configurable number of days before due.

- **Components:** Per-bill `numRemindDays` NumberInput in the bill detail drawer (default 3, 0 = off); `notificationCenter` bell in the app header listing reminders; `lstReminders` settings list of upcoming reminders.
- **Data:** `reminders` table (NEW if not already built by §568: id, kind='bill', ref_id → bills.id, remind_at date, dismissed bool). Transformer `tDueReminders`: for each Active bill with remind_days > 0, remind_at = next_due_date − remind_days; surface when today ≥ remind_at and no payment this month and not dismissed. A "Check reminders" JS query runs on app load and on the Bills screen mount.
- **Logic notes:** Reminders are in-app only (no push unless the user opts in per §557). They prompt action — tapping opens the bill with Mark Paid. Dismissing is per-month (a new reminder regenerates next cycle). Never auto-post transactions.
- **Acceptance:** (1) Setting 3 days on a bill due in 3 days produces a notification on next load. (2) Setting 0 disables reminders for that bill. (3) Dismissed reminder does not reappear until next month. (4) Marking the bill paid clears its pending reminder.

### 289. Display overdue bills in red with days-overdue counts.

- **Components:** `secOverdue` section pinned above the upcoming list, visible only when non-empty: red left-border cards, each with `txtOverdueDays` ("6 days overdue" in red).
- **Data:** Transformer `tOverdueBills`: Active bills where next due date < today AND no `bill_payments` row for the current month. days_overdue = floor((today − due_date)/86400000).
- **Logic notes:** A bill due yesterday and unpaid is 1 day overdue. Once marked paid (§284), it leaves the overdue section immediately. Paused bills never appear here. The overdue section also feeds the overview's upcoming-bills tile (§12) count.
- **Acceptance:** (1) Unpaid bill past its due date appears in red with the correct day count. (2) Mark-paid removes it from the section instantly. (3) Section hidden entirely when nothing is overdue. (4) Days count increments correctly across a date change.

### 290. Support splitting a bill across housemates with a "my share" amount.

- **Components:** Bill detail drawer section "Split": `lstSplits` editable list (housemate label + amount), `numMyShare` NumberInput, `btnAddSplit`; list row shows "My share: $725 of $1,450".
- **Data:** `bill_splits` table (bill_id, housemate_label, my_share). The bill's own `amount` stays the full bill; "my share" is informational. `qrySplits` per bill; `qrySaveSplit` upserts.
- **Logic notes:** Splits do NOT change what mark-paid records (full amount paid from the account, per §284) unless the user edits the paid amount to their share — the confirm dialog (§284) shows a hint "Your share: $X" when splits exist. Validation: sum of shares ≤ bill amount (warn if over).
- **Acceptance:** (1) Splits save and reload per bill. (2) "My share" displays on the bill row as secondary text. (3) Mark-paid dialog surfaces the share hint when splits exist. (4) Over-100% split total triggers a warning, not a silent save.

### 291. Add bill history showing paid amounts per month for the last 12 months.

- **Components:** Bill detail drawer tab "History": `tblBillHistory` Table (month, paid amount, paid date, late flag, linked transaction link) for the trailing 12 months; `chartBillHistory` Bar chart of paid amounts by month.
- **Data:** `qryBillHistory` — `SELECT bp.month, t.amount, t.date, bp.paid_late FROM bill_payments bp JOIN transactions t ON t.id = bp.transaction_id WHERE bp.bill_id = {{selectedBill}} AND bp.month >= to_char(CURRENT_DATE - INTERVAL '11 months','YYYY-MM') ORDER BY bp.month`. Months with no payment show "—" (gap, not zero).
- **Logic notes:** Uses actual paid amounts (variable bills show real variance). Late flags (§299) render as an amber dot. Tapping a row jumps to the linked transaction (§405 global search compatible).
- **Acceptance:** (1) History shows up to 12 rows, newest first. (2) Unpaid months display as gaps, not $0. (3) Bar chart matches table amounts exactly. (4) Late payments are visually flagged.

### 292. Allow pausing a bill (e.g., seasonal service) without deleting it.

- **Components:** Bill detail drawer `btnPauseBill` / `btnResumeBill` toggle; paused bills move to a collapsed "Paused" section at the bottom of the Bills screen with gray styling.
- **Data:** `qryPauseBill` — `UPDATE bills SET status='Paused', paused_at=NOW() WHERE id=...`; resume sets `status='Active'`, clears `paused_at`. Audit-logged per §740 conventions.
- **Logic notes:** Paused bills are excluded from: upcoming list, overdue section, monthly total (§287), 30-day cash requirement (§297), reminders (§288), and the paid/unpaid summary (§285). History (§291) is preserved. Mark-paid is disabled on paused bills (button hidden).
- **Acceptance:** (1) Pausing removes the bill from all active computations immediately. (2) Paused section lists it with a Resume button. (3) History remains viewable while paused. (4) Resume restores it to all lists with the next due date recomputed.

### 293. Show which account each bill is paid from with a balance check warning.

- **Components:** `colAccount` chip column in `tblBills` (account name); in `modConfirmPaid`, an amber warning banner `warnLowBalance`: "Checking ···1234 balance $210 — this $1,450 payment exceeds it" shown when account balance < bill amount.
- **Data:** `qryAccounts` provides current balances. Transformer `tBalanceCheck`: compares `selBillAccount` balance vs `numPaidAmount` at confirm-dialog open and on amount change.
- **Logic notes:** Warning only — the user can still confirm (manual tracking; the balance may be stale). Warning text states the balance is user-entered, not bank-verified. Re-checks live as the amount is edited.
- **Acceptance:** (1) Warning appears exactly when balance < amount, with both figures shown. (2) No warning when balance ≥ amount. (3) User can still confirm through the warning. (4) Banner text disclaims the balance is manual.

### 294. Add bill attachments for statements or confirmation screenshots.

- **Components:** Bill detail drawer "Attachments" section: `fileBillUpload` FileInput (images + PDF), `lstAttachments` with thumbnails, `btnDeleteAttachment` (confirm + audit log).
- **Data:** `bill_attachments` (bill_id, file_name, file_url, uploaded_at). Files stored in Retool Storage; URL saved on the row. `qryUploadAttachment` → storage upload then INSERT. Per-file cap 10 MB; images compressed client-side before upload.
- **Logic notes:** Attachments are in-app only, with the §456-style warning that they are not backed up externally. Deleting requires confirmation and writes an audit entry. Sample bills may carry one labeled sample statement.
- **Acceptance:** (1) Upload attaches and lists with thumbnail/file icon. (2) Oversize file (>10 MB) is rejected with a clear message. (3) Delete requires confirmation and audit-logs. (4) Attachments visible offline once downloaded (§457 convention).

### 295. Display a bills calendar view across the month.

- **Components:** `calBills` view toggle (List | Calendar) on the Bills screen: month grid; each day cell shows bill chips (name + amount); paid bills get a green check, overdue a red dot; `selCalMonth` month navigator.
- **Data:** Transformer `tBillCalendar`: for the displayed month, map each Active bill to its clamped due date (§281 logic); join `bill_payments` for the month to mark paid. Prev/next month buttons shift the view.
- **Logic notes:** Day cells with no bills render empty (no clutter). Tapping a bill chip opens the bill detail drawer. The calendar respects the sample toggle. Journal notes (§428) can overlay here later — keep the day-cell component extensible.
- **Acceptance:** (1) Every active bill appears on its correct due day for the displayed month. (2) Paid bills show the check mark; overdue show the red dot. (3) Month navigation preserves paid states. (4) iPhone: grid legible at 390px, chips tappable.

### 296. Add autopay-labeled bills (user-reported) vs manual-pay bills as a tag only.

- **Components:** `selAutopayTag` on the bill form (§282) and detail drawer: "Manual" (default) / "Autopay (reported)". List shows a small "AUTO" tag next to the name for autopay-reported bills.
- **Data:** `bills.autopay_tag` text column. Pure label — no behavior change.
- **Logic notes:** The tag must never imply the app moves money: help tooltip reads "You reported this bill as autopay. FinanceOS does not move money — mark paid manually when it clears." Reminders (§288) still fire for autopay bills (as a nudge to verify), labeled "verify autopay cleared".
- **Acceptance:** (1) Tag saves and displays on the bill row. (2) Tooltip/help text states the app performs no automatic payments. (3) Reminders still fire for autopay bills with "verify" wording. (4) Mark-paid flow is identical for both tags.

### 297. Show upcoming 30-day bill cash requirement as a single number.

- **Components:** `tileCashRequired` hero tile on the Bills screen header: "Need $2,340 in the next 30 days" with subtext listing the count of bills.
- **Data:** Transformer `tCashRequired`: sum over Active bills of amount (estimate if variable) where next due date (§283 logic) is within today..today+30. Sample toggle respected.
- **Logic notes:** Excludes paused bills and bills already paid this month whose next due falls inside the window (their next occurrence is next month — outside the window by construction). Single headline number; tapping opens a breakdown list.
- **Acceptance:** (1) Number equals the hand-computed sum of bills due in the next 30 days. (2) Paid-this-month bills are not double-counted. (3) Paused bills excluded. (4) Tile updates immediately after any bill add/edit/mark-paid.

### 298. Support bill amount change tracking with old vs new comparison.

- **Components:** Bill detail drawer "Amount history" mini-table: date, old → new, delta (+$15 / −$10 in green/red).
- **Data:** `bill_amount_history` (bill_id, old_amount, new_amount, changed_at). `qryUpdateBill` (edit amount flow) inserts a history row before updating `bills.amount`. Estimate changes on variable bills (§286) also log here.
- **Logic notes:** Only actual changes log (same-value saves are no-ops). Delta computed as new − old. The latest change surfaces as a one-line note on the bill row ("↑ $15 since Mar") for 30 days.
- **Acceptance:** (1) Editing a bill's amount creates exactly one history row with correct old/new. (2) Re-saving the same amount creates no row. (3) Delta signs and colors are correct. (4) History survives bill pause/resume.

### 299. Add a "paid late" flag with optional late-fee transaction creation.

- **Components:** `chkPaidLate` Checkbox in `modConfirmPaid` (§284); when checked, reveals `numLateFee` NumberInput (optional, default empty) → confirming writes the bill payment with `paid_late=true` plus an optional second transaction.
- **Data:** `bill_payments.paid_late` bool; `late_fee_transaction_id` fk. Late-fee transaction: `INSERT INTO transactions (account_id, date, description='Late fee: ' || bill.name, category=bill.category, amount=-fee, is_sample)`.
- **Logic notes:** Late fee is optional — flag can be set with no fee. History (§291) shows the late marker. The flag is per payment (per month), not per bill. Audit-logged with the payment.
- **Acceptance:** (1) Checking "paid late" with a fee creates two transactions and sets the flag. (2) Flag without fee creates one transaction with the flag set. (3) History view marks the month late. (4) Unchecking leaves a normal on-time payment.

### 300. Export the bills list to CSV or print view.

- **Components:** `btnExportBills` SplitButton (CSV / Print) on the Bills screen header.
- **Data:** `qryExportBills` — full bill list (name, category, amount, variable flag, due day, account, autopay tag, status, paid-this-month) honoring the sample toggle and current search filter. CSV generated via `utils.downloadFile`; print view opens a clean `htmlBillsPrint` component with the banner, export date, and a "Contains sample data" watermark line when samples are included (§486/§487 conventions).
- **Logic notes:** Export stamped with "Manual tracking — no bank connection" and export date. `is_sample` column included in CSV. Print stylesheet: no navigation chrome, legible type.
- **Acceptance:** (1) CSV downloads with all columns and opens correctly in a spreadsheet app (UTF-8). (2) Print view shows the banner and export date. (3) Sample toggle off → no sample rows in either output. (4) Print view hides app navigation and buttons.
