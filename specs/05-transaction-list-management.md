# FinanceOS Builder Spec — Section 05: Transaction list & management (items 81–100)

## Tables touched

Existing (from specs.md App 11):
- `accounts`: id (pk), name, type (Checking/Savings/Credit/Investment), institution, balance, updated_at, is_sample
- `transactions`: id (pk), account_id (fk), date, description, category (Housing/Food/Transport/Income/Fun/Bills/Other), amount (negative=spend), is_sample — plus planned column additions below
- `audit_log`: id (pk), actor, action, table_name, record_id, before_json, after_json, created_at, entry_method (manual/import/sample/ai-suggested)

Planned column additions to `transactions` (add via Retool Database "Add column"):
- `pinned` (bool, default false) — item 98
- `entry_status` (text: posted/draft/voided, default "posted") — items 96, 99
- `void_reason` (text, nullable) — item 99
- `notes` (text, nullable) — item 94 (may already exist from section 04)

No brand-new tables are required in this section; drafts and voided rows reuse `transactions` with `entry_status`.

## Cross-cutting rules (apply to every item below)

- Every screen shows the persistent banner: **"Manual tracking — no bank connection yet."**
- All seeded rows carry `is_sample=true` plus a visible SAMPLE badge; when the sample toggle hides samples, sample rows and their amounts are excluded from every total, subtotal, and chart.
- Nothing ever renders as a live bank feed. All figures are manual entry or labeled sample data.
- Any delete/edit of a transaction writes an `audit_log` row with before/after JSON.

---

### 81. Build a transactions table showing date, description, category, account, and signed amount.

- **Components:** Retool `Table` named `tblTransactions` on the Transactions screen. Columns: `date` (date, format MM/DD/YYYY), `description` (text), `category` (Tag column, mapped to category colors from section 07), `account_id` → display account name via joined query (text), `amount` (number, format currency with explicit sign). Enable server-side sort on date/amount.
- **Data:** Query `qryTransactions`: `SELECT t.*, a.name AS account_name FROM transactions t JOIN accounts a ON a.id = t.account_id ORDER BY t.date DESC, t.id DESC`. Filter out samples when `{{ state.sampleHidden }}` is true.
- **Logic notes:** Signed amount rule: negative = spend, positive = income. Sort default newest-first; ties broken by id. Draft rows (`entry_status='draft'`) render with a "DRAFT" tag; voided rows show the voided state (item 99).
- **Acceptance:** (1) Table renders all five columns for every posted transaction. (2) Amounts display with explicit −/+ signs. (3) Hiding samples removes sample rows and their amounts from the table.

### 82. Add an edit-transaction drawer that preserves the original values in the audit log.

- **Components:** `Drawer Frame` named `drwEditTransaction`, opened by row action "Edit" or double-click on `tblTransactions`. Form fields mirror the add-transaction form (account select, date picker, description, category select, signed amount, notes). Buttons: Save, Cancel.
- **Data:** On open, store `{{ tblTransactions.selectedRow }}` into temp state `stEditOriginal` (deep copy). On Save, query `qryUpdateTransaction` (UPDATE by id), then query `qryAuditInsert` writing before=`stEditOriginal`, after=new values, action="update".
- **Logic notes:** Capture the original BEFORE any field changes — the drawer form binds to a separate temp state object, not the row directly. Concurrent-edit guard: if `updated_at`-style check fails, warn and reload.
- **Acceptance:** (1) Opening the drawer prefills all current values. (2) Saving updates the row and totals recompute (item 180). (3) `audit_log` holds the pre-edit values in `before_json`.

### 83. Implement transaction deletion with a confirm step and an undo window.

- **Components:** Row action "Delete" → `Modal Frame` `mdlConfirmDelete` ("Delete this transaction? This is logged and reversible for 10 seconds."). After delete, `Notification` toast with Undo button (10s duration).
- **Data:** Query `qryDeleteTransaction` (DELETE by id) + `qryAuditInsert` (action="delete", before=row JSON). Undo: query `qryRestoreTransaction` re-INSERTs the cached row JSON from temp state `stLastDeleted` (including original id).
- **Logic notes:** Undo must restore the exact original id so audit trail stays coherent. Deleting a pinned or sample row follows the same path. After the 10s window expires, clear `stLastDeleted`.
- **Acceptance:** (1) Delete requires explicit confirmation; no one-click deletes. (2) Undo within 10s restores the row with its original id and values. (3) Every delete writes an audit entry with the full before-image.

### 84. Show negative spend amounts in red and income in green with explicit minus/plus signs.

- **Components:** In `tblTransactions`, set the `amount` column to a custom-mapped Tag/HTML cell. Cell background: `#FDE8E8`/text `#B42318` when amount < 0; `#E6F6EC`/text `#067647` when amount > 0. Zero renders neutral gray.
- **Data:** No new query; formatting done in a column mapper: `{{ currentRow.amount < 0 ? '−' + formatCurrency(abs) : '+' + formatCurrency }}` (use ASCII hyphen-minus in code, en dash acceptable in display copy).
- **Logic notes:** Color is never the only indicator (accessibility, item 933): the explicit −/+ sign and the words "spend"/"income" in tooltips carry the meaning. Keep the rule consistent with the overview and budgets screens.
- **Acceptance:** (1) Spend rows show red with an explicit minus sign; income rows show green with a plus sign. (2) A grayscale/color-blind check still distinguishes direction via the sign. (3) Zero-amount rows (if any) render neutral.

### 85. Display a "SAMPLE" pill on rows where is_sample is true.

- **Components:** Add a mapped column `samplePill` on `tblTransactions`: Tag component, text "SAMPLE", amber background `#FEF3C7`, dark amber text, shown only when `{{ currentRow.is_sample }}`.
- **Data:** Uses the existing `is_sample` boolean from `qryTransactions`; no new query.
- **Logic notes:** The pill must survive all filter/sort/group states — it is a row attribute, not a filter artifact. Draft copies of sample rows (item 96) inherit `is_sample=true`. Exported CSVs preserve the `is_sample` column (items 351, 481).
- **Acceptance:** (1) Every sample row shows the SAMPLE pill; no real row shows it. (2) The pill remains visible when samples are shown and disappears with the row when hidden. (3) Bulk "mark as real" (item 90) clears the pill immediately.

### 86. Group transactions by day with date headers and daily subtotals.

- **Components:** On `tblTransactions`, set grouping by a computed `dayKey` column (YYYY-MM-DD). Group header template: `{{ formatDate(dayKey, 'EEEE, MMM d') }}` + right-aligned daily subtotal chip.
- **Data:** Transformer `tfnDailyGroups`: groups `qryTransactions.data` by date, computing per-day spend total, income total, and net. Sorting inside groups by time/id.
- **Logic notes:** Daily subtotal = sum(amount) for the day; display spend and income separately ("−$86.40 spend · +$0 income") so a mixed day is unambiguous. Sample-hidden mode recomputes subtotals without samples. Timezone: group by the user's local date (America/New_York default).
- **Acceptance:** (1) Rows cluster under one header per day, newest day first. (2) Each header shows correct daily spend/income/net. (3) Toggling samples off recomputes all subtotals without sample rows.

### 87. Add infinite scroll or pagination for large manual transaction histories.

- **Components:** `tblTransactions` set to server-side pagination, page size 50. Footer shows "Showing X of Y" with Prev/Next. (Retool tables virtualize; if infinite scroll is preferred, enable "infinite scroll" on the table with the same page loader.)
- **Data:** Query `qryTransactionsPaged`: `... ORDER BY date DESC, id DESC LIMIT {{ tblTransactions.pageSize }} OFFSET {{ tblTransactions.paginationOffset }}`, plus `qryTransactionsCount` for the total with identical filters. All filter criteria from section 06 feed both queries.
- **Logic notes:** Filters, search, and sort must apply server-side before pagination so page boundaries are stable. Grouping by day (item 86) works per page; day headers may repeat across page boundaries — acceptable, documented in the header.
- **Acceptance:** (1) 10,000-row history loads the first page in under 2 seconds (ties to item 941). (2) Changing filters resets to page 1. (3) No duplicate or missing rows across page turns.

### 88. Show a running balance column per account when the list is filtered to one account.

- **Components:** Conditional column `runningBalance` on `tblTransactions`, visible only when the account filter (section 06) selects exactly one account.
- **Data:** Transformer `tfnRunningBalance`: takes the single-account filtered rows sorted oldest→newest, computes cumulative sum of `amount`, then maps back to display order (newest-first). Starting point: account's opening balance if known, else cumulative from the earliest transaction on record — label the column header accordingly ("Running bal. (from first entry)").
- **Logic notes:** Samples excluded per toggle. Draft/voided rows excluded from the running balance. The figure is explicitly labeled as reconstructed from manual entries, not a bank balance.
- **Acceptance:** (1) Column appears only under a single-account filter. (2) Each row's running balance equals the sum of that row and all older rows in the account. (3) A caption notes the balance is reconstructed from manual entries.

### 89. Allow bulk-selecting transactions for bulk delete or bulk recategorization.

- **Components:** Enable row selection (checkboxes) on `tblTransactions`. Bulk action bar appears when `{{ tblTransactions.selectedRows.length > 0 }}` with buttons: "Recategorize", "Delete", "Clear selection".
- **Data:** `qryBulkRecategorize`: UPDATE transactions SET category = new WHERE id IN (...). `qryBulkDelete`: DELETE WHERE id IN (...). Both write ONE `audit_log` row with action="bulk_update"/"bulk_delete" and affected-row count (item 749).
- **Logic notes:** Recategorize opens a small modal with the 7-category select; confirm lists the count. Bulk delete reuses the confirm + undo pattern (item 83) — undo restores all deleted rows. Selection clears after the action.
- **Acceptance:** (1) Selecting N rows enables the bulk bar with an accurate count. (2) Bulk recategorize updates all selected rows and logs one audit entry. (3) Bulk delete requires confirmation and offers undo restoring all rows.

### 90. Add a "mark as sample / mark as real" bulk action for sample-data management.

- **Components:** Two buttons in the bulk action bar: "Mark as sample", "Mark as real", each with a confirm modal stating the effect on totals.
- **Data:** `qryBulkFlagSample`: UPDATE transactions SET is_sample = true/false WHERE id IN (...). Writes an `audit_log` row (action="bulk_flag_sample", affected count) — this is the per-table half of item 358.
- **Logic notes:** Marking real→sample immediately hides those rows from totals when the sample toggle is off; totals recompute instantly. Marking sample→real requires no extra review for transactions (the row-level review applies at item 353's guided flow). Never allow marking such that zero rows remain visible without an empty-state message.
- **Acceptance:** (1) Bulk flag updates all selected rows' pills and totals in one action. (2) The action is logged with row count and direction. (3) Overview/budget totals recompute to reflect the new flags.

### 91. Display the manual-entry banner as a slim strip above the transactions table.

- **Components:** `Alert`/`Callout` component `bannerTransactions` pinned directly above `tblTransactions`: single line, text "Manual tracking — no bank connection yet.", slim height (32px), info styling, non-dismissible on this screen.
- **Data:** Static text; optionally reads `{{ state.bannerStyle }}` (top vs slim footer, item 343).
- **Logic notes:** The strip must remain visible while the table scrolls (sticky). It collapses nothing — the table sits below it. This is the per-screen instance of the global banner rule (item 341).
- **Acceptance:** (1) The exact banner text renders above the table on load. (2) It stays visible during vertical scroll. (3) It cannot be permanently dismissed on this screen.

### 92. Show month and year headers when scrolling through older transactions.

- **Components:** With day-grouping active (item 86), add a higher-level grouping: month-year divider rows ("October 2026") inserted above the first day-group of each month, styled as sticky section headers.
- **Data:** Transformer `tfnMonthDividers`: derives `monthKey` (YYYY-MM) from each row's date; the table renders a divider when `monthKey` changes. Works with the paged query — a month may span pages.
- **Logic notes:** Dividers are display-only and never affect selection, sorting, or totals. When a date-range filter is active, only in-range months appear. Sticky headers stack: month divider sticks above the day header.
- **Acceptance:** (1) Scrolling a multi-year history shows a month+year divider at each month boundary. (2) Dividers do not break row selection or bulk actions. (3) Filtered views show only relevant months.

### 93. Add swipe-to-edit and swipe-to-delete gestures on iPhone rows.

- **Components:** On the mobile card layout (item 385: tables collapse to cards under 390px), each transaction card supports touch handlers: swipe-left reveals Edit (blue) and Delete (red) action buttons; tap executes.
- **Data:** Same queries as desktop edit/delete (`qryUpdateTransaction` via `drwEditTransaction` as a bottom sheet per item 391; delete via `mdlConfirmDelete`).
- **Logic notes:** Swipe threshold ~30% of card width; a short swipe snaps back. Buttons are ≥44px tall (item 384). Edit opens the drawer as a bottom sheet; delete keeps the confirm modal + undo toast. Provide long-press as a fallback for users who can't swipe (accessibility).
- **Acceptance:** (1) Swipe-left on a card reveals working Edit/Delete buttons. (2) Both actions hit the same audit-logged queries as desktop. (3) Touch targets meet the 44px minimum.

### 94. Include a per-row notes indicator when a memo is attached.

- **Components:** Small `Icon` (note/paperclip) in the description cell of `tblTransactions`, visible when `{{ currentRow.notes }}` is non-empty. Tooltip on hover/tap shows the first 120 characters of the memo.
- **Data:** `notes` column from `qryTransactions`; no new query.
- **Logic notes:** Indicator is display-only; editing the memo happens in the edit drawer (item 82). Long memos truncate in the tooltip with "…" — full text in the drawer. Notes are searchable (item 117).
- **Acceptance:** (1) Rows with memos show the indicator; rows without do not. (2) Tooltip previews the memo text. (3) Clearing a memo removes the indicator on save.

### 95. Show the account name as a compact chip on each row.

- **Components:** In `tblTransactions`, render `account_name` as a `Tag` chip: small pill, neutral background, truncated at ~14 characters with full name in tooltip.
- **Data:** `account_name` comes from the `qryTransactions` JOIN (item 81); no new query.
- **Logic notes:** Chip color stays neutral — account identity is not category color. When the list is filtered to one account (item 88), the chip may be hidden to save space (config flag `hideAccountChipWhenFiltered`). Tapping the chip jumps to the account's detail/filtered view (item 49 linkage).
- **Acceptance:** (1) Every row shows its account as a compact chip. (2) Long account names truncate gracefully with tooltip. (3) Chip tap navigates to the account-filtered view.

### 96. Add a "duplicate month" action that copies last month's recurring-style entries as drafts.

- **Components:** Button "Duplicate last month" in the Transactions toolbar → modal listing candidate rows (last month's transactions, excluding Income and voided) with checkboxes, "Create N drafts" confirm.
- **Data:** Query `qryLastMonthCandidates`: last calendar month's posted transactions. `qryCreateDrafts`: INSERTs copies with `date` = same day-of-month in the current month (clamped to month length), `entry_status='draft'`, `is_sample` inherited, description suffixed " (draft)".
- **Logic notes:** Drafts are excluded from all totals, budgets, and reports until posted. Each draft needs one-tap Post or Discard from the row. This is a manual helper — it never auto-posts (consistent with item 323's never-auto-post rule).
- **Acceptance:** (1) The modal lists exactly last month's eligible rows. (2) Created drafts appear tagged DRAFT and excluded from totals. (3) Posting a draft converts it to a normal transaction with an audit entry.

### 97. Display totals for the current filter (count, total spend, total income) in a sticky footer.

- **Components:** Sticky footer bar under `tblTransactions`: three stat chips — "N transactions", "Spend −$X", "Income +$Y" — updating live with filters.
- **Data:** Transformer `tfnFilterTotals` over the filtered dataset: count = rows; spend = abs(sum(amount<0)); income = sum(amount>0). Uses the same filtered set the table shows (respecting sample toggle, entry_status posted-only).
- **Logic notes:** Footer sticks to the bottom of the table viewport during scroll. Drafts and voided rows excluded. When filters match zero rows, show "0 transactions" with an empty-state hint (item 409 linkage).
- **Acceptance:** (1) Footer always visible while scrolling the table. (2) Counts and sums match the visible rows exactly. (3) Changing any filter updates the footer instantly.

### 98. Allow pinning important transactions (e.g., rent) to the top of the list.

- **Components:** Row action "Pin"/"Unpin" (pin icon). Pinned rows render in a pinned section above the normal list with a subtle highlight and a pin icon; a "Pinned (N)" divider separates them.
- **Data:** `pinned` boolean column on `transactions` (see header). `qryTogglePin`: UPDATE pinned. Sort rule: pinned first (by date), then normal sort order.
- **Logic notes:** Pinned rows still respect filters — a pinned row outside the current filter is hidden, not shown. Unpin returns the row to its natural position. Pin state persists per user. Samples can be pinned too.
- **Acceptance:** (1) Pinned rows appear above all unpinned rows under a "Pinned" divider. (2) Pin/unpin persists across reloads. (3) Filters still apply to pinned rows.

### 99. Show a voided state for corrected transactions instead of hard-deleting them.

- **Components:** Row action "Void" (in the edit drawer and row menu) → modal asking for a reason → row renders struck-through, gray, with a "VOIDED" tag and the reason in a tooltip. A "Corrections" filter can show/hide voided rows; default hides them from totals.
- **Data:** `entry_status='voided'` + `void_reason` columns. `qryVoidTransaction`: UPDATE status/reason + `qryAuditInsert` (action="void"). Un-void action restores to "posted".
- **Logic notes:** Voided rows are excluded from all totals, budgets, trends, and exports unless the corrections filter explicitly includes them. Voiding is the recommended correction path; hard delete remains for true mistakes (item 83). The original values stay intact for audit.
- **Acceptance:** (1) Voided rows show the struck-through VOIDED state with reason. (2) Voided amounts are excluded from every total by default. (3) Un-void restores the row to posted with an audit entry.

### 100. Add a print-friendly transaction statement view for a selected month.

- **Components:** Button "Print statement" → month picker → `Modal`/dedicated view `viewStatement`: clean layout — header (app name, month, "Manual tracking — no bank connection yet." per item 359), summary (income, spend, net), category table, full transaction list (date, description, category, account, amount), footer note that figures are manual entries/sample data.
- **Data:** Query `qryStatementMonth`: posted transactions for the selected month (sample toggle respected; if samples included, each row carries the SAMPLE marker per item 352's export banner variant).
- **Logic notes:** Uses the print stylesheet (item 498): hides nav/buttons, black-on-white, page breaks between sections. Amounts formatted with explicit signs. Include the `is_sample`/entry-method column (item 499).
- **Acceptance:** (1) Statement shows exactly the selected month's posted transactions. (2) Banner text and manual-entry footer appear on the printed page. (3) Print preview hides all app chrome and paginates cleanly.
