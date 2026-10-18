# Section 46 — Command palette & quick actions (items 901–920)

**Tables touched:** accounts, transactions, budgets, goals (existing App 11 tables).
New table:

- **quick_actions_log**: id (pk), action_name (text), payload (text, JSON of the resulting write), source (text: palette / widget / voice / repeat), created_at (timestamp), undone (bool, default false), undo_deadline (timestamp, nullable), is_sample (bool)

**Shared cross-cutting rules:** every screen shows the persistent banner "Manual tracking — no bank connection yet."; sample rows carry `is_sample=true` + SAMPLE badge and are excluded from totals when the sample toggle hides them; nothing ever presents as a live bank feed. Quick actions are shortcuts into manual-entry screens or pre-filled drafts — nothing auto-executes against real accounts. Every palette-driven write lands in `quick_actions_log` with its source tag.

---

### 901. Add a command palette (Cmd/Ctrl+K) with actions: add transaction, update balance, go to screen.

**Components:** a global Modal (transparent overlay, input field at top, ListView of results below) rendered on every screen; global keyboard listener via an app-level Event Handler (keydown: Meta+K or Ctrl+K toggles; Esc closes). Result rows show icon + label + hint.

**Data:** a `commandRegistry` JS array (label, keywords, type: action/navigation, handler) stored in app state; filter function matches typed text against label + keywords.

**Logic notes:** palette is always mounted but hidden (state `paletteOpen=false`); opening focuses the input immediately; handlers navigate (`utils.openApp`/screen switch) or open the target form modal. No Retool-native palette exists, so this is a custom modal.

**Acceptance:** (1) Cmd+K (Mac) and Ctrl+K (Windows) open the palette on every screen; (2) typing "add transaction" surfaces the action and Enter opens the add-transaction form; (3) Esc closes and returns focus to the prior field; (4) palette is usable with the keyboard only.

---

### 902. Support natural-language quick-add: "coffee 4.50 food" parses to a draft transaction.

**Components:** palette input doubles as the parser box; a preview card under the input showing parsed amount, description, category before confirmation.

**Data:** `parseQuickAdd(input)` JS: regex extracts a decimal amount, then matches the trailing/leading word against the known category list (Housing/Food/Transport/Income/Fun/Bills/Other); remainder becomes the description.

**Logic notes:** ambiguous parses (no amount found, unknown category word) show a "couldn't parse — adjust" message with the raw text editable; Enter inserts a DRAFT transaction row flagged for review rather than saving directly; amount sign rule: spend = negative unless prefixed with +/income.

**Acceptance:** (1) "coffee 4.50 food" parses to description "coffee", amount -4.50, category Food; (2) "salary +2000 income" parses to +2000 Income; (3) unparseable input shows the raw text with an inline error, not a silent wrong row; (4) parsed drafts open the transaction form pre-filled for review.

---

### 903. Show recent commands at the top of the palette for repeat actions.

**Components:** palette results ListView with a "Recent" section header pinned above "All commands".

**Data:** `getQuickActionHistory` query: `SELECT action_name, MAX(created_at) FROM quick_actions_log GROUP BY action_name ORDER BY 2 DESC LIMIT 5`; mapped to command labels.

**Logic notes:** recency list refreshes on each palette open (not cached); undone actions (undone=true) still count as used commands — the *command* was recent even if the write was reversed; empty state shows "No recent commands yet" hint text.

**Acceptance:** (1) after running "add transaction", reopening the palette shows it in Recent; (2) the 6th-most-recent command drops off the list; (3) clicking a recent command re-runs it immediately; (4) recent section never duplicates entries also listed under All commands.

---

### 904. Add quick actions on iPhone via long-press on the app icon (new transaction, update balance).

**Components:** this is an OS-level configuration, not a Retool component: document the iOS Shortcut + "Add to Home Screen" wrapper setup in the builder spec, including the deep-link URL scheme (`financeos://add-transaction`, `financeos://update-balance`) handled by a URL-parameter reader on the Retool mobile layout.

**Data:** app-level JS on startup parses `window.location` hash/query for `?action=add-transaction`; when present, opens the corresponding form modal.

**Logic notes:** Retool web apps cannot natively register iOS home-screen quick actions — the asset is a setup guide for wrapping the Retool mobile page in a Shortcut with parameters; builder must test on a physical device (Nic's hands for the device test itself); fallback: in-app floating "+" button.

**Acceptance:** (1) deep-link `?action=add-transaction` opens the add form directly; (2) `?action=update-balance` opens the balance update form; (3) unknown action values are ignored silently; (4) spec documents the Shortcut setup steps for iOS.

---

### 905. Support voice-dictated quick-add with manual review before saving.

**Components:** a mic button in the palette header; a dictation sheet (Modal) showing live transcript and the parsed preview from item 902.

**Data:** browser Web Speech API (`webkitSpeechRecognition`) captures the transcript; the same `parseQuickAdd` parses it; result opens the transaction form pre-filled — never saved directly.

**Logic notes:** voice is input-only: the save button remains a manual tap; if the Speech API is unavailable (unsupported browser), the mic button hides itself; transcript errors ("fifty" vs "15") are the user's to correct in the review form; log with source='voice' after save.

**Acceptance:** (1) tapping mic starts dictation and shows the transcript live; (2) "coffee four fifty food" produces an editable parsed draft; (3) nothing is saved without the user pressing Save; (4) mic button hidden on browsers without Speech API support.

---

### 906. Add a "log last amount again" one-tap repeat for habitual purchases.

**Components:** a "Repeat last" button on the Transactions screen header and as a palette command; confirmation toast showing what was repeated.

**Data:** `getLastTransaction` query: latest non-sample transaction by date desc; insert clones description, category, account_id, amount with today's date.

**Logic notes:** only repeats the single most recent transaction — not a history picker; if the last transaction was is_sample=true, the repeat inherits sample status with the SAMPLE badge; the new row gets source='repeat' in quick_actions_log; undo available per item 919.

**Acceptance:** (1) one tap creates a new transaction duplicating the last row's fields with today's date; (2) toast confirms "Logged $4.50 coffee · Food again"; (3) repeat of a sample row is also flagged sample; (4) the original row is never modified.

---

### 907. Provide quick balance update from the accounts list without opening edit forms.

**Components:** inline number input in each Accounts-list row (pencil icon reveals it); Save/Cancel inline buttons; row flashes green on save.

**Data:** `updateBalance` query: `UPDATE accounts SET balance={{inlineValue}}, updated_at=NOW() WHERE id={{row.id}}`; then refresh the accounts query.

**Logic notes:** inline edit bypasses the full account-edit modal but validation still applies (numeric, non-empty); optimistic row highlight reverts if the query fails; updated_at always refreshes; banner/sample rules unchanged.

**Acceptance:** (1) typing a new balance in the row and pressing Enter saves it; (2) invalid input (letters, empty) shows an inline error and saves nothing; (3) net worth on Overview updates after the refresh; (4) Esc cancels without changes.

---

### 908. Add quick goal contribution from the goals list.

**Components:** "+" button on each goal card/row opening a small popover with an amount field and Confirm; progress bar animates on save.

**Data:** `addContribution` query: `UPDATE goals SET saved = saved + {{amount}} WHERE id={{goal.id}}`; refresh goals query.

**Logic notes:** amount must be positive; contribution cannot exceed a sanity cap warning (e.g. > remaining shown as "exceeds target — still add?"); popover defaults focus to the amount field; log source='palette' when launched from palette.

**Acceptance:** (1) entering 50 on a goal raises its saved total by 50 and animates the bar; (2) zero/negative input is rejected with a message; (3) over-target contributions trigger a confirm dialog; (4) goal deadline and target stay unchanged.

---

### 909. Show keyboard shortcut cheatsheet in help.

**Components:** Help screen section (or "?" button opening a drawer) listing shortcuts: Cmd/Ctrl+K palette, arrows/Enter navigation, Esc close, "?" opens cheatsheet.

**Data:** static content — no queries; cheatsheet entries reference the actual keybindings defined in the app's event handlers so docs can't drift.

**Logic notes:** single source of truth: a `shortcuts` JS object in app state feeds both the event handlers and the cheatsheet render; platform note shows Cmd on Mac, Ctrl elsewhere; cheatsheet itself is keyboard-openable with "?".

**Acceptance:** (1) "?" opens the cheatsheet from any screen; (2) every listed shortcut performs as documented; (3) Mac shows ⌘, others show Ctrl; (4) adding a new shortcut to the object automatically adds it to the cheatsheet.

---

### 910. Support jumping to a category's transactions from the palette ("food transactions").

**Components:** palette navigation results that deep-link: selecting "Food transactions" switches to the Transactions screen with the category filter preset.

**Data:** command registry entries per category generated from the category list; handler sets `transactionFilter.category='Food'` then navigates.

**Logic notes:** filter preset happens before navigation so the table renders filtered on arrival; clearing the filter afterwards shows all again; works for all seven categories including Other; "uncategorized" is not a valid target.

**Acceptance:** (1) typing "food transactions" and pressing Enter lands on Transactions filtered to Food; (2) the filter chip visibly shows "Food"; (3) clearing the chip restores the full list; (4) each category appears exactly once in palette results.

---

### 911. Add a "start month review" palette command.

**Components:** palette action "Start month review" that opens the month-end review screen (see section 37) or its first step modal.

**Data:** handler navigates to the review flow with `reviewMonth` preset to the current month; no data writes on launch.

**Logic notes:** the command is navigation-only — the review flow itself handles its checklist; if a review for the current month already exists, the command opens it in read-only/resume state; label includes the month name ("Start October review").

**Acceptance:** (1) command label shows the current month name; (2) running it opens the month-end review flow for that month; (3) an existing review resumes rather than duplicating; (4) no writes occur until the user acts inside the review.

---

### 912. Provide quick export commands ("export this month CSV").

**Components:** palette actions "Export this month CSV" and "Export this month PDF"; a brief "Preparing…" state then the browser download.

**Data:** reuses the export queries from section 25 (filtered transactions for the current month); generates the file client-side and triggers download.

**Logic notes:** export always includes the manual-tracking banner line and export date; respects the sample-data toggle (hidden samples are excluded); large exports show a progress state; logged to quick_actions_log with source='palette'.

**Acceptance:** (1) "export this month CSV" downloads a CSV of this month's visible transactions; (2) the file header notes manual tracking + export date; (3) hidden samples are excluded when the toggle hides them; (4) the command appears in recent commands after use.

---

### 913. Show command palette results grouped by action vs navigation.

**Components:** palette ListView with two sticky section headers: "Actions" (things that do/write) and "Go to" (navigation targets).

**Data:** command registry entries carry `group: 'action' | 'navigation'`; the filter renders matches under their group headers.

**Logic notes:** groups only render when they have matches; empty group header never shows; within groups, recent commands (item 903) still float to a "Recent" header above both; write-actions show a subtle dot marker distinguishing them from navigation.

**Acceptance:** (1) results always appear under "Actions" and/or "Go to" headers; (2) a query matching only navigation shows no "Actions" header; (3) "Recent" appears above both groups when present; (4) action rows are visually distinct from navigation rows.

---

### 914. Allow customizing which quick actions appear on mobile.

**Components:** Settings screen "Quick actions" section with a toggle list (checkbox per action) and drag/order controls; the mobile layout reads this config.

**Data:** `mobileQuickActions` persisted in localStorage (array of action ids in order); mobile FAB/sheet renders only enabled actions in the saved order.

**Logic notes:** desktop palette is unaffected — this config is mobile-only; at least one action must remain enabled (last toggle is blocked with a message); new actions added later default to enabled; config syncs per device, not per account.

**Acceptance:** (1) disabling "update balance" removes it from the mobile quick-action sheet; (2) reorder changes the sheet order immediately; (3) the last enabled action cannot be turned off; (4) desktop palette still shows all actions.

---

### 915. Add a "mark all bills paid" month-end quick action with per-bill confirmation.

**Components:** palette action opening a confirmation modal listing this month's unpaid bills with individual checkboxes (all checked by default); "Confirm" applies.

**Data:** query unpaid bills (recurring bills due this month, not marked paid); per-bill update marks paid; each bill write logged with source='palette'.

**Logic notes:** per-bill confirmation is mandatory — no silent bulk write; unchecking a bill excludes it; bills already paid this month are excluded from the list; the modal shows the total amount being marked paid.

**Acceptance:** (1) modal lists only this month's unpaid bills; (2) unchecking a bill excludes it from the write; (3) confirming writes only the checked bills; (4) a summary toast reports "5 bills marked paid, 1 skipped".

---

### 916. Support quick-add from the lock-screen widget opening a minimal form.

**Components:** minimal quick-add form (amount, description, category, account — four fields only) opened via deep link `?action=quick-add` (same mechanism as item 904's Shortcut wrapper).

**Data:** on submit, inserts the transaction and logs source='widget'; form defaults account to the most-used account.

**Logic notes:** minimal form is a separate mobile-optimized screen, not the full add-transaction form; all validation still applies; the widget/Shortcut setup is Nic's-hands iOS configuration — the builder asset is the deep-link handler + minimal form; full review remains available via "details" link.

**Acceptance:** (1) `?action=quick-add` opens the four-field minimal form; (2) submitting saves a valid transaction and logs source='widget'; (3) invalid input blocks with inline errors; (4) a "full form" link opens the complete add-transaction screen.

---

### 917. Show a confirmation summary after every palette-driven write.

**Components:** a confirmation modal/toast shown after each palette write: what was created/changed, key fields, and an Undo button (item 919).

**Data:** the write's result (inserted row id + fields) feeds the summary text; no extra query needed.

**Logic notes:** summary is mandatory even for one-field updates (e.g. balance change shows old → new); auto-dismisses after 6 seconds unless hovered; the summary never exposes other users' data — single-user app, but the payload shown is only the affected row.

**Acceptance:** (1) every palette write shows what changed with concrete values; (2) balance updates show old → new; (3) summary includes an Undo button; (4) summaries auto-dismiss without blocking further input.

---

### 918. Log palette-driven writes with a "quick-add" source tag.

**Components:** no UI — an insert into `quick_actions_log` with source='palette' fired by every palette write handler.

**Data:** `logQuickAction` query: `INSERT INTO quick_actions_log (action_name, payload, source, created_at, is_sample) VALUES (...)`; payload is the JSON of the written row.

**Logic notes:** logging is fire-and-forget after the primary write succeeds — a log failure never rolls back the user's data (log it to console); payload excludes nothing the user typed; the source tag distinguishes palette writes from form writes in audits.

**Acceptance:** (1) every palette write creates a quick_actions_log row with source='palette'; (2) payload JSON matches the written row's key fields; (3) a failed log insert does not undo the write; (4) the log is queryable for the audit/history screens.

---

### 919. Add an undo toast for 10 seconds after any quick action.

**Components:** toast with countdown bar and Undo button, shown after every quick action (palette, widget, voice, repeat).

**Data:** on write, store the inverse operation in `quick_actions_log` (row id + prior values); `undoQuickAction` reverses it: delete inserted rows, restore prior field values; sets undone=true, guarded by undo_deadline.

**Logic notes:** undo window is 10 seconds from the write; after expiry the button disables and the log row's undo_deadline passes; undo of a delete restores the full row; undo itself is logged (no infinite undo chain — undo cannot be undone).

**Acceptance:** (1) toast appears after every quick action with a working 10s countdown; (2) Undo reverses the write (insert removed / values restored); (3) after 10s the Undo button is gone; (4) pressing Undo twice does not double-reverse.

---

### 920. Make the palette fully keyboard-navigable for accessibility.

**Components:** palette modal with roving tabindex: arrow keys move, Enter activates, Esc closes, Tab cycles within the modal (focus trap).

**Data:** no queries — pure interaction spec; selection index held in `paletteSelectedIndex` state.

**Logic notes:** focus is trapped inside the palette while open and restored to the invoking element on close; every result row is a real button (screen-reader announced with its group: "action" or "navigation"); visible focus ring on the selected row; respects prefers-reduced-motion for the open animation.

**Acceptance:** (1) arrows + Enter operate the palette with zero mouse use; (2) Tab never escapes the modal while open; (3) closing returns focus to the previously focused element; (4) a screen reader announces each row's label and group.
