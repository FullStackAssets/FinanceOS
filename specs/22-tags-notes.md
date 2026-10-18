# FinanceOS Builder Spec — §22 Tags & Notes (Items 421–440)

## Section overview
Freeform tagging on transactions plus notes across budgets, accounts, goals, and a daily money-journal, all searchable from global search and exportable. Tags are a many-to-many layer over the fixed category taxonomy (Housing / Food / Transport / Income / Fun / Bills / Other); notes are plain text fields and journal-day records. Tag math and note content always respect the sample-data filter.

## Tables touched
- `transactions` (existing): add `notes` (text, nullable) — freeform memo per transaction.
- `budgets` (existing): add `notes` (text, nullable) — reasoning per category/month.
- `accounts` (existing): add `notes` (text, nullable) — e.g. "don't touch — house fund".
- `goals` (existing): add `notes` (text, nullable) — per-goal notes separate from contribution notes.
- NEW `tags`: id (pk), tag_name (text, unique, lowercase-normalized), color (text, hex), created_at (timestamp), is_sample (bool, default false).
- NEW `transaction_tags`: id (pk), transaction_id (fk → transactions.id), tag_id (fk → tags.id), created_at (timestamp). Unique constraint on (transaction_id, tag_id).
- NEW `journal_notes`: id (pk), note_date (date, unique), body (text), created_at (timestamp), updated_at (timestamp), is_sample (bool, default false). One journal note per calendar day.
- NEW `tag_alerts`: id (pk), tag_id (fk → tags.id), threshold_amount (number), period (text: monthly), enabled (bool, default true), last_fired_month (text, nullable), created_at (timestamp).

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen in this section (tags view, notes views, journal, global search results).
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data filter hides samples, they are excluded from tag totals, clouds, alerts, searches, and exports.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.

---

### 421. Add freeform tags on transactions beyond the fixed category (e.g., "vacation", "work-trip").
- Components: Tags multi-select input `tagTransactionTags` on the add/edit transaction form and in the transaction drawer; displays chosen tags as removable chips.
- Data: On save, upsert each tag into `tags` (lowercase-normalized) and write rows to `transaction_tags`. Query `qryTransactionTags` joins tags for display.
- Logic notes: Tags are independent of the fixed category; a transaction keeps both. Tag names normalized to lowercase, trimmed, max 40 chars. Duplicates on one transaction rejected by the unique constraint.
- Acceptance: (1) Adding "vacation" to a transaction persists and re-renders on reload. (2) Same tag on two transactions creates one `tags` row, two `transaction_tags` rows. (3) Category remains unchanged when tags are added.

### 422. Support multiple tags per transaction with comma-separated entry.
- Components: The tags input accepts comma-separated typing; each comma commits the current token as a chip. Helper text: "Separate tags with commas."
- Data: On commit, split on commas, trim, drop empties, lowercase; bulk-insert into `transaction_tags` in one transaction after upserting new `tags` rows.
- Logic notes: Pasting "dining, work-trip, nyc" yields three tags. Empty tokens ignored. Max 10 tags per transaction (inline warning at the cap).
- Acceptance: (1) Typing "a, b,c" creates exactly three tags. (2) Leading/trailing commas produce no empty tags. (3) 11th tag attempt shows the cap warning and is not saved.

### 423. Build a tag cloud showing spend per tag for the current month.
- Components: New "Tags" tab on the Transactions screen; cloud container `tagCloud` where each tag renders as a pill sized by spend; month selector defaults to current month.
- Data: Transformer `trTagSpend` aggregates `SUM(ABS(amount))` per tag for spend transactions in the selected month (sample filter applied). Query joins `transaction_tags → transactions`.
- Logic notes: Font size scales linearly between min and max spend; ties share size. Income-category transactions excluded from spend cloud (toggle to include). Clicking a tag applies it as a filter (see 424).
- Acceptance: (1) Cloud reflects only the selected month. (2) Hiding samples removes sample spend from sizing. (3) Clicking a pill filters the table to that tag.

### 424. Add tag-based filtering combined with category and date filters.
- Components: Tag multi-select `selTagFilter` in the transactions filter bar; active filter chips include tags with individual clear buttons.
- Data: `qryTransactionsFiltered` adds `AND EXISTS (SELECT 1 FROM transaction_tags tt JOIN tags t ON t.id=tt.tag_id WHERE tt.transaction_id = transactions.id AND t.tag_name IN (...))`. Combines with category/date via AND.
- Logic notes: Multiple selected tags use OR within the tag clause, AND against other filters. Empty tag selection = no tag constraint.
- Acceptance: (1) Selecting "dining" + category Food shows only rows matching both. (2) Two tags show rows with either. (3) Clearing the tag chip restores prior results.

### 425. Show tag suggestions based on previously used tags as the user types.
- Components: Autocomplete dropdown under the tags input; shows up to 6 matches with usage counts ("dining — 14 uses").
- Data: Query `qryTagSuggestions` = `SELECT tag_name, COUNT(*) uses FROM tags JOIN transaction_tags ... GROUP BY tag_name WHERE tag_name LIKE prefix% ORDER BY uses DESC LIMIT 6`.
- Logic notes: Matches on prefix of the current token only (after last comma). Keyboard navigable (↑/↓/Enter). Selecting inserts the tag as a chip.
- Acceptance: (1) Typing "di" suggests "dining" before never-used tags. (2) Enter accepts the highlighted suggestion. (3) No duplicate chip if the tag is already selected.

### 426. Add a notes field on budgets per category/month.
- Components: Textarea `txtBudgetNotes` on the add/edit budget form and in the budget row drawer; placeholder "Why this limit? (e.g., cut Fun to $150 for trip savings)".
- Data: Writes to new `budgets.notes` column via `qrySaveBudget`. Shown read-only under the budget bar on the budgets screen.
- Logic notes: Notes are per category+month row, so history is preserved when limits change month to month. Max 500 chars.
- Acceptance: (1) Note saved with a new budget renders under its bar. (2) Editing the limit does not erase the note. (3) Note persists across month copies only if the user opts in.

### 427. Support notes on accounts (e.g., "don't touch — house fund").
- Components: Textarea `txtAccountNotes` on the add/edit account form; read-only note line under the account name in the accounts list.
- Data: Writes to new `accounts.notes` column via `qrySaveAccount`.
- Logic notes: Notes are informational only — they never affect balances, net worth, or totals. Max 500 chars.
- Acceptance: (1) Note saved on an account shows in the list row. (2) Note survives balance updates. (3) Sample accounts can carry notes with the SAMPLE badge retained.

### 428. Add a daily money-journal note attached to calendar days.
- Components: "Journal" tab with a month calendar; clicking a day opens a note editor (textarea + save); days with notes show a dot indicator.
- Data: Upsert into `journal_notes` keyed by `note_date` (one row per day). `qryJournalMonth` fetches the visible month.
- Logic notes: Empty body on save deletes the day's row (no empty notes stored). `updated_at` stamps edits. Journal is personal reflection, not a transaction — never affects totals.
- Acceptance: (1) Saving text on a day shows the dot; reopening shows the text. (2) Clearing the text removes the dot. (3) Only one row per date exists after repeated saves.

### 429. Display journal notes inline in the cash-flow calendar day popover.
- Components: Cash-flow calendar day popover gains a "Journal" section at the bottom showing the day's note (or "No note — add one" link that jumps to the Journal tab).
- Data: Popover query left-joins `journal_notes` on the calendar date.
- Logic notes: Read-only in the popover; editing happens in the Journal tab to keep one edit path. Sample-filter does not hide the user's own journal notes.
- Acceptance: (1) A day with a journal note shows its first 140 chars in the popover. (2) Days without notes show the add link. (3) Popover still lists planned/recorded items above the journal section.

### 430. Search across all notes and tags from the global search.
- Components: Global search input in the header searches a new `qryGlobalSearchNotesTags` source; results grouped "Tags" and "Notes" below the existing transaction/account/goal groups.
- Data: One query UNIONs: tag names matching the term (→ jumps to tag-filtered transactions), transaction notes, budget notes, account notes, goal notes, and journal bodies matching (→ jumps to the owning record).
- Logic notes: Case-insensitive substring match. Sample-tagged notes excluded when samples hidden. Result rows show context snippet with matched text highlighted (see 115 pattern).
- Acceptance: (1) Searching "trip" finds the "trip savings" budget note and the "vacation" journal entry. (2) Clicking a tag result opens transactions filtered to that tag. (3) Hidden samples do not appear in results.

### 431. Add tag-level monthly totals with budget-style progress bars.
- Components: On the Tags tab, each tag row shows month spend total plus a progress bar against an optional per-tag monthly target (`tag_targets` fields on `tags`: target_amount nullable).
- Data: Reuse `trTagSpend`; bar = spend ÷ target when target set, else show total with no bar fill. Target edited inline per tag.
- Logic notes: Bars use the same green/amber/red thresholds as budget bars (under 80% / 80–100% / over 100%). Targets are planning aids, not enforced limits.
- Acceptance: (1) Tag with $120 spend and $200 target shows a 60% green bar. (2) Tag with no target shows total and "set target" affordance. (3) Overspend renders red with the overage amount.

### 432. Support renaming a tag with all tagged transactions updated.
- Components: Tag row "Rename" action opens a dialog with the new name field; shows count of affected transactions before confirming.
- Data: Single `UPDATE tags SET tag_name = ... WHERE id = ...`; `transaction_tags` rows reference the id, so all transactions update atomically. Lowercase-normalize the new name; block merge into an existing tag name (offer merge explicitly instead).
- Logic notes: Rename is instant across the app (cloud, filters, exports) since everything keys off tag id. Logged per item 440.
- Acceptance: (1) Renaming "dining" → "eating-out" updates all 14 tagged rows at once. (2) Attempting to rename to an existing tag name is blocked with an explanation. (3) Tag color and targets survive the rename.

### 433. Show untagged transaction counts to nudge complete tagging.
- Components: Banner card on the Tags tab: "23 of 140 transactions this month have no tags" with a "Review untagged" button.
- Data: `qryUntaggedCount` counts transactions in the selected month with no `transaction_tags` rows (sample filter applied). Button applies an "untagged only" filter to the table.
- Logic notes: Counts spend transactions only (income tagging optional); the nudge is dismissible per session. Framed as a suggestion, never blocking.
- Acceptance: (1) Count matches a manual spot check. (2) "Review untagged" filters the table to exactly those rows. (3) Tagging a row decrements the count on refresh.

### 434. Add tag colors for visual scanning in the table.
- Components: Tag chips in the transactions table render in the tag's color (`tags.color`); color picker on the tag row and in the tag editor; default palette assigned to new tags round-robin.
- Data: `color` column stores hex (e.g. "#3b82f6"). Query returns color with tag names for the table's chip renderer.
- Logic notes: Colors are visual only. Auto-picked colors guarantee ≥ 4.5:1 contrast against the chip background; custom picks warn but are allowed. Sample tags keep their color under the SAMPLE badge.
- Acceptance: (1) Changing a tag's color updates every chip app-wide. (2) New tags get a readable default color. (3) Table remains scannable with 5+ tags on a row (chips wrap).

### 435. Export tags with transactions in CSV exports.
- Components: Transactions CSV export gains a `tags` column (semicolon-separated tag names) and a `notes` column; column-chooser includes both, on by default.
- Data: Export query left-joins aggregated tag names per transaction (`STRING_AGG(t.tag_name, ';')`). Sample rows keep `is_sample=true` and the SAMPLE watermark rules.
- Logic notes: Transactions with no tags export an empty tags cell (not null text). Tag order alphabetical for stable diffs.
- Acceptance: (1) Exported CSV has `tags` and `notes` columns with correct values. (2) Multi-tag rows show "dining;work-trip". (3) UTF-8 encoding opens cleanly in spreadsheet apps.

### 436. Add per-goal notes separate from contribution notes.
- Components: Textarea `txtGoalNotes` on the goal detail view ("Notes about this goal"); contribution notes remain on each contribution row per item 227.
- Data: Writes to new `goals.notes` column; contribution notes stay on `goal_contributions.note`. Both shown in the goal detail, clearly labeled.
- Logic notes: Goal notes describe the goal ("Japan trip — flights ~$900"); contribution notes describe a single deposit. Editing one never touches the other.
- Acceptance: (1) Goal note saves and renders in the detail header. (2) Adding a contribution note does not alter the goal note. (3) Both export in the goals/contributions CSV.

### 437. Support @mention-style links from a note to a specific transaction.
- Components: In transaction/budget/goal note textareas, typing "@" opens a picker of recent transactions; selecting inserts "@txn:<id>" rendered as a chip showing date + description + amount.
- Data: Renderer parses `@txn:(\d+)` tokens and resolves via `qryTransactionById`; broken ids render as plain text. Stored raw in the notes column.
- Logic notes: Clicking the chip jumps to the transaction in the filtered table. Mentions are display links only — no notifications, no data changes. Deleting the transaction leaves the raw token as text.
- Acceptance: (1) Typing "@" and picking a transaction inserts a chip. (2) Saved note re-renders the chip on reload. (3) Chip click navigates to that transaction.

### 438. Show a timeline of journal notes alongside spend for context.
- Components: Journal tab gains a "Timeline" toggle: chronological list interleaving journal notes with that day's total spend (from transactions).
- Data: `qryJournalTimeline` unions journal rows with per-day spend aggregates for the visible range, ordered by date desc.
- Logic notes: Spend side is read-only context; only the note body is editable inline. Days with spend but no note show spend alone; days with neither are omitted.
- Acceptance: (1) Timeline shows note + same-day spend on one row. (2) Sample spend excluded when samples hidden. (3) Editing a note inline saves to `journal_notes`.

### 439. Add tag-based alerts ("notify when 'dining' tag exceeds $200").
- Components: Per-tag "Alert" control on the Tags tab: threshold amount input + enable toggle; alerts surface in the notification center and as an in-app banner.
- Data: `tag_alerts` stores threshold per tag per month; evaluator runs on transaction save and on month rollover: `IF SUM(tag spend this month) > threshold AND last_fired_month != current_month THEN fire`. `last_fired_month` prevents repeat firing.
- Logic notes: Alerts reference manual data ("based on your entered transactions") per item 556. One fire per tag per month; re-arm automatically next month. Disabled toggle stops evaluation.
- Acceptance: (1) Setting $200 on "dining" fires when month spend passes $200. (2) No second alert fires for the same tag that month. (3) Disabling the toggle suppresses the alert.

### 440. Log tag renames and bulk-tag operations in the audit log.
- Components: No new UI; audit-log viewer gains tag operations under the existing table filter ("tags", "transaction_tags", "journal_notes").
- Data: Every rename (432), bulk-tag/untag action, tag delete/merge, and tag-alert create/edit writes an audit entry with actor, timestamp, table, record id, and before/after values. Bulk ops logged once with affected-row counts (per item 749).
- Logic notes: Append-only; no edit/delete of entries (per item 744). Data-entry method recorded as "manual".
- Acceptance: (1) Renaming a tag creates an audit entry with old/new names. (2) Bulk-tagging 20 rows logs one entry with count=20. (3) Entries appear when filtering the log to the tags tables.
