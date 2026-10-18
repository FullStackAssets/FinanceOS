# FinanceOS — Section 03: Account management (items 41–60)

Builder spec for the Accounts screen and account lifecycle in the FinanceOS Retool app.

## Tables touched

**accounts** (existing; extended): id (pk), name (text), type (text: Checking / Savings / Credit / Investment), institution (text), balance (number), updated_at (timestamp), is_sample (bool), plus new columns —
- `display_order` (number, default 0): user-controlled list order.
- `is_archived` (bool, default false): closed accounts hidden from net worth but kept for history.
- `include_in_net_worth` (bool, default true): per-account net-worth toggle.
- `notes` (text, nullable): freeform account notes.

**account_balance_history** (new): id (pk), account_id (fk → accounts), old_balance (number), new_balance (number), changed_at (timestamp), changed_via (text: manual / stepper / import), is_sample (bool).

## Shared cross-cutting rules

- Every screen carries the persistent banner "Manual tracking — no bank connection yet." The Accounts screen header includes it (item 59).
- Every seeded row has is_sample=true and shows a visible "SAMPLE" badge; sample rows are excluded from totals whenever the sample toggle hides them (default: hidden).
- Nothing ever presents balances as a live bank feed. All figures are labeled manual entries or sample data.

---

### 41. Build an accounts screen listing every account with name, type, institution, balance, and updated_at.

- **Components:** `accountsTable` (Retool Table component) with columns name, type, institution, balance (currency format), updated_at (relative time, e.g. "2h ago"), plus a `searchInput` text field above the table and a `bannerHeader` text component with the manual-entry banner. Row actions: Edit, Update balance, Archive.
- **Data:** Query `getAccounts`: `SELECT * FROM accounts WHERE is_archived = false ORDER BY display_order ASC, name ASC`, filtered client-side by the search term (item 52). A `getArchivedAccounts` query powers an "Archived" tab.
- **Logic notes:** Balance renders with two decimals and the account currency symbol; negative credit balances show a leading minus in red. updated_at uses a transformer `fmtRelative` converting timestamps to "xh ago / xd ago".
- **Acceptance:** (1) All non-archived accounts render with the five required columns. (2) Banner text is visible in the screen header. (3) Search filters the list without a query round-trip delay beyond 300ms.

### 42. Add a create-account form with fields for name, type (Checking/Savings/Credit/Investment), institution, and starting balance.

- **Components:** `createAccountModal` (Retool Modal) with `acctName` (text input, required), `acctType` (select: Checking / Savings / Credit / Investment), `acctInstitution` (text input), `acctStartingBalance` (number input, default 0), and a Save button. Helper text under the type picker explains each type in plain language.
- **Data:** Query `createAccount`: INSERT into accounts with is_sample=false, updated_at=NOW(), display_order = max+1. On success, trigger `getAccounts` refresh and an audit-log insert (see §38 spec).
- **Logic notes:** Name is required and unique (case-insensitive check before insert; show "An account with this name already exists"). Starting balance may be negative only for Credit type; other types get a confirm dialog for negative values (ties to item 55).
- **Acceptance:** (1) Submitting with valid inputs creates the row and closes the modal. (2) Duplicate name is rejected with a clear message. (3) updated_at is stamped at creation time.

### 43. Build an edit-account form to rename accounts and change institution or type.

- **Components:** `editAccountModal` prefilled from the selected row: `editName`, `editType` (select), `editInstitution`, and a read-only balance display with a link to "update balance" (item 45). Save and Cancel buttons.
- **Data:** Query `updateAccount`: UPDATE name/type/institution WHERE id; never touches balance here. Before/after values are written to the audit log with actor and timestamp.
- **Logic notes:** Renaming to an existing account name is blocked (same uniqueness check as create). Changing type from Credit to another type with a negative balance triggers a warning confirm. Type change does not alter stored transactions.
- **Acceptance:** (1) Edits persist and the table reflects them immediately. (2) Audit log records old vs new values. (3) Duplicate-name rename is rejected.

### 44. Add a delete-account flow that requires typing the account name and warns about orphaned transactions.

- **Components:** `deleteAccountModal` showing the account name, a `confirmNameInput` text field, a warning callout listing the count of linked transactions (from `countTxnsByAccount`), and a Delete button disabled until the typed name matches exactly.
- **Data:** Query `deleteAccount`: hard-delete is blocked if transactions exist — instead offer two paths: (a) archive the account (sets is_archived=true, keeps history — preferred), or (b) reassign its transactions to another account via `reassignPicker` then DELETE the account. The flow defaults to archiving.
- **Logic notes:** Exact-match (case-sensitive) typed confirmation. Orphan warning shows "This account has N transactions. Deleting will orphan them unless you reassign." Sample accounts can be deleted freely (logged).
- **Acceptance:** (1) Delete button stays disabled until the name matches exactly. (2) With transactions present, the user must choose archive or reassign. (3) Every deletion or archival is audit-logged.

### 45. Implement an update-balance action that stamps updated_at with the current time on every edit.

- **Components:** `updateBalanceModal` with `newBalance` number input, current balance shown for reference, delta preview ("+$120.00"), and Save. Triggered from the table row action and the quick-balance stepper (item 54).
- **Data:** Query `updateBalance`: in a single Retool DB transaction — INSERT into account_balance_history (old_balance, new_balance, changed_at=NOW(), changed_via) then UPDATE accounts SET balance, updated_at=NOW() WHERE id. Refresh `getAccounts` and any net-worth computed values.
- **Logic notes:** updated_at is server-stamped (NOW()), never user-editable. The history row is written before the update so old/new are captured even if the update query retries.
- **Acceptance:** (1) After save, updated_at is within 5 seconds of now. (2) A history row exists with correct old/new values. (3) Net-worth tiles recompute without page reload.

### 46. Show a prominent "SAMPLE" badge on any account row where is_sample is true.

- **Components:** In `accountsTable`, a `sampleBadge` mapped column rendering a high-contrast amber pill with text "SAMPLE" when `is_sample` is true; empty otherwise. Also show the badge inside `editAccountModal` and `updateBalanceModal` headers.
- **Data:** No new query; uses the is_sample column already returned by `getAccounts`.
- **Logic notes:** Badge styling: amber background (#92400e on #fef3c7), bold, uppercase, minimum 44px touch height on mobile. The badge must also appear on archived-sample rows and in the per-type totals (item 53) via a "includes sample" footnote when samples are visible.
- **Acceptance:** (1) Every is_sample=true row shows the badge; real rows show none. (2) Badge is legible at 390px width and at 135% iOS text size. (3) Toggling the global sample filter hides badged rows and their totals.

### 47. Add account type icons for Checking, Savings, Credit, and Investment in the list.

- **Components:** `typeIcon` mapped column in `accountsTable` rendering an icon + type label: Checking = banknote/card icon, Savings = piggy-bank, Credit = credit-card, Investment = trending-up chart. Use Retool's built-in icon set; each icon has an aria-label with the type name.
- **Data:** Pure presentation over the existing `type` column; a `typeIconMap` JS object maps type → icon name and color.
- **Logic notes:** Unknown or legacy type values fall back to a generic wallet icon and are flagged in a data-quality note. Colors: Checking blue, Savings green, Credit purple, Investment amber — consistent with the category color system (§07 spec) so icons match across screens.
- **Acceptance:** (1) All four types render the correct icon and label. (2) Icons have screen-reader labels (ties to §47 accessibility). (3) Unknown types show the fallback, never a blank cell.

### 48. Allow reordering accounts via drag handles so the overview order is user-controlled.

- **Components:** Drag handles in `accountsTable` using Retool's draggable row ordering (or up/down arrow buttons as the accessible fallback on iPhone). A "Reset to default order" link restores alphabetical ordering.
- **Data:** On drop, query `saveAccountOrder`: UPDATE accounts SET display_order = index for each row in the new order. `getAccounts` and the overview screen both ORDER BY display_order. The overview account cards (§01) read the same order.
- **Logic notes:** Archived accounts keep their display_order but are excluded from the visible list. Order changes are audit-logged as a single bulk event. Keyboard users get the up/down buttons with the same handler.
- **Acceptance:** (1) Dragging an account persists the new order after reload. (2) The overview screen reflects the same order. (3) Keyboard-only reordering works via the arrow buttons.

### 49. Show per-account transaction counts linking directly to a filtered transactions view.

- **Components:** `txnCount` mapped column in `accountsTable` rendering the count as a link-styled button (e.g. "24").
- **Data:** Query `getTxnCounts`: `SELECT account_id, COUNT(*) FROM transactions GROUP BY account_id` joined into the table via a transformer. Clicking sets the global `selectedAccountFilter` and navigates to the Transactions screen with the account filter applied (deep link `/transactions?account=<id>` per §21).
- **Logic notes:** Counts respect the current sample-data filter (hidden samples excluded from the count when the toggle hides them). Zero-transaction accounts show "0" in muted text, still clickable.
- **Acceptance:** (1) Counts match the transactions table for each account. (2) Clicking lands on Transactions pre-filtered to that account. (3) Counts update after the sample toggle changes.

### 50. Add a balance history log per account recording each manual update with old and new values.

- **Components:** `balanceHistoryDrawer` (Retool Drawer) opened from the account row: a timeline list showing changed_at, old → new balance, delta, and changed_via (manual / stepper / import).
- **Data:** Query `getBalanceHistory`: `SELECT * FROM account_balance_history WHERE account_id = {{selectedAccount}} ORDER BY changed_at DESC`. Populated by `updateBalance` (item 45) and backfilled once for existing accounts with a single "baseline" row.
- **Logic notes:** History is append-only; there is no edit or delete UI for history rows. Deltas render green for increases, red for decreases. Export includes history when exporting the account (item 60 covers the accounts CSV; history export rides the audit export in §38).
- **Acceptance:** (1) Every balance update creates exactly one history row. (2) Old/new values are correct to the cent. (3) The drawer lists history newest-first.

### 51. Support archiving closed accounts so they leave net worth but keep their history.

- **Components:** `archiveAccount` row action (with confirm modal) and an "Archived" tab on the accounts screen listing archived accounts with an "Unarchive" action.
- **Data:** Query `archiveAccount`: UPDATE accounts SET is_archived=true WHERE id. `getAccounts` excludes archived; net-worth queries exclude archived automatically. Transactions and balance history remain untouched and queryable.
- **Logic notes:** Archiving a sample account is allowed and logged. An archived account with a non-zero balance shows a warning ("balance $X will leave net worth") before confirming. Unarchiving restores the account to net worth with its last balance.
- **Acceptance:** (1) Archived accounts disappear from net worth immediately. (2) Their transactions and history remain visible via the Archived tab. (3) Unarchive restores net-worth inclusion.

### 52. Add a search box on the accounts screen filtering by name or institution.

- **Components:** `accountSearch` text input with a search icon and a clear (×) button, positioned above `accountsTable`.
- **Data:** Client-side filter in the table's data transformer: `row.name.toLowerCase().includes(q) || row.institution.toLowerCase().includes(q)`. Debounced at 200ms for responsiveness on iPhone.
- **Logic notes:** Search is case-insensitive and matches partial strings. When the Archived tab is active, search covers archived accounts too. Empty query restores the full list.
- **Acceptance:** (1) Typing filters by name and institution. (2) Clear button resets the list. (3) No server round-trip per keystroke.

### 53. Display total balances grouped by account type at the top of the accounts screen.

- **Components:** `typeTotalsStrip`: four summary cards (Checking / Savings / Credit / Investment) each showing the summed balance and account count, styled with the type icons from item 47.
- **Data:** Transformer `typeTotals` over `getAccounts` data: group by type, sum balances (credit balances sum as negative per §02 rules), count accounts. Respects the sample toggle and excludes archived accounts.
- **Logic notes:** When samples are visible, each card shows a small "incl. sample" footnote. Totals use the same currency formatting as the table. Cards are tappable to filter the table to that type.
- **Acceptance:** (1) Four cards show correct sums and counts. (2) Tapping a card filters the table to that type. (3) Sums match the table's visible rows.

### 54. Add a quick-balance stepper (+/- buttons) for fast manual adjustments on iPhone.

- **Components:** In each `accountsTable` row (and in the account detail drawer), a compact stepper: [−] [balance] [+] buttons with 44px touch targets, plus a long-press to open the full update-balance modal (item 45).
- **Data:** Each tap calls `updateBalance` with newBalance = current ± step. Default step $10; a small step-picker ($1 / $10 / $100) sits in the table header. changed_via is recorded as "stepper" in account_balance_history.
- **Logic notes:** Optimistic UI: the displayed balance updates immediately, then the query confirms; on failure the value rolls back with an error toast. Negative-balance confirmation (item 55) applies to stepper taps too.
- **Acceptance:** (1) Each tap adjusts the balance by the selected step. (2) History rows are written with changed_via="stepper". (3) Failed writes roll back the displayed value.

### 55. Validate that balance entries are numeric and require confirmation for negative checking balances.

- **Components:** Validation rules on `newBalance` (item 45) and `acctStartingBalance` (item 42): numeric-only input, inline error "Enter a number" on invalid input, Save disabled until valid.
- **Data:** Client-side validation in the form's validation handler; server-side the balance column is numeric so non-numeric values are rejected at the DB layer as a backstop.
- **Logic notes:** If type is Checking or Savings and the entered balance is negative, a confirm dialog appears: "Negative balance on a Checking account — is this an overdraft? [Confirm] [Cancel]". Credit accounts accept negatives without confirmation. Empty input is treated as invalid, not zero.
- **Acceptance:** (1) Non-numeric input blocks saving with a clear message. (2) Negative Checking/Savings balances require explicit confirmation. (3) Credit accounts accept negative balances directly.

### 56. Show which accounts feed the net-worth total with an included/excluded indicator.

- **Components:** `nwIncluded` mapped column in `accountsTable`: a toggle switch bound to `include_in_net_worth`, showing "Included"/"Excluded" text alongside it for non-color indication.
- **Data:** Query `toggleNwInclude`: UPDATE accounts SET include_in_net_worth = NOT include_in_net_worth WHERE id; refresh `getAccounts` and net-worth computed values instantly. Archived accounts are always excluded regardless of the toggle (shown as "Excluded — archived").
- **Logic notes:** Toggling is audit-logged. The net-worth hero (§01/§02) recomputes from the same query so the change is visible immediately. Sample accounts excluded by the sample toggle show "Excluded — sample hidden".
- **Acceptance:** (1) Toggling updates net worth without reload. (2) Archived accounts always show excluded. (3) The indicator has a text label, not color alone.

### 57. Add a bulk "mark all real" action to clear is_sample flags after the user replaces sample data.

- **Components:** `markAllRealButton` in the accounts screen header (visible only when at least one sample account exists), opening a confirm modal listing how many sample accounts, transactions, and goals will be converted.
- **Data:** Query `markAllReal`: UPDATE accounts SET is_sample=false WHERE is_sample=true; the modal offers checkboxes to also clear transactions, budgets, and goals sample flags in the same action. Every conversion is written to the audit log as a bulk event with row counts (ties to item 358 in §18).
- **Logic notes:** Destructive-adjacent: requires typed confirmation "REAL". After conversion, the sample-data inventory (§18) shows zero rows and the "delete all sample data" action hides. This is the companion to the guided "replace samples with my data" flow (item 350).
- **Acceptance:** (1) All sample flags clear in one action after typed confirmation. (2) Audit log records the bulk conversion with counts. (3) Totals recompute to real-only values.

### 58. Include an account notes field for details like "joint with partner" or "emergency fund."

- **Components:** `acctNotes` textarea in both `createAccountModal` and `editAccountModal` (placeholder: "e.g. joint with partner, emergency fund"), plus a notes indicator (sticky-note icon) on table rows that have notes, with the full text in a tooltip/popover.
- **Data:** New `accounts.notes` text column; included in `getAccounts` and in the accounts CSV export (item 60). Notes are searchable via the global search (§21).
- **Logic notes:** 500-character limit with a live counter. Notes never render on shared/exported views marked "mask details" (privacy toggle, §39). Empty notes store NULL, not empty string.
- **Acceptance:** (1) Notes save and display on the account. (2) Rows with notes show the indicator. (3) Notes appear in the CSV export.

### 59. Display the manual-entry banner inside the accounts screen header as well.

- **Components:** `bannerHeader` text component pinned at the top of the Accounts screen: "Manual tracking — no bank connection yet." with the same styling as the overview banner (compact variant on iPhone per item 388 in §20).
- **Data:** Static text; follows the global banner settings (dismissible per session, position toggle top/footer from §18 items 342–343).
- **Logic notes:** The banner component is shared across screens — one Retool module `manualBanner` reused on every main screen so wording and behavior stay identical. It never scrolls out of view on the Accounts screen.
- **Acceptance:** (1) Banner text is visible in the Accounts header on load. (2) Dismissing it on one screen dismisses it session-wide. (3) It restores on next app load.

### 60. Add an export-accounts CSV button including the is_sample flag.

- **Components:** `exportAccountsBtn` in the accounts screen header; on click generates and downloads `financeos-accounts-YYYY-MM-DD.csv`.
- **Data:** Query `exportAccounts`: SELECT id, name, type, institution, balance, updated_at, is_sample, is_archived, include_in_net_worth, notes FROM accounts (respecting current tab: active or archived, and the sample toggle). A JS transformer builds the CSV with UTF-8 BOM for spreadsheet compatibility.
- **Logic notes:** Every export is stamped per §25 rules: filename includes the date; the CSV's first comment row (or a companion line in the download toast) states "Manual tracking — no bank connection". If sample rows are included, an on-screen notice warns "Contains sample data". The export is logged in export history (§25 item 495).
- **Acceptance:** (1) CSV downloads with all listed columns including is_sample. (2) The file opens correctly in Excel/Sheets with UTF-8 encoding. (3) Export is recorded in export history.
