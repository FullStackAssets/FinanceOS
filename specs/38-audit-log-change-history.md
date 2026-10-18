# Section 38 — Audit log & change history (items 741–760)

## Section title
Audit log & change history — a complete, append-only record of every data change in FinanceOS, viewable and exportable by the user for their own review.

## Tables touched
Existing: `accounts`, `transactions`, `budgets`, `goals`, `bills` (all with `is_sample` flags, schemas per specs.md App 11 and section 15).

New table — `audit_log`:
- `id` (pk, text/UUID)
- `occurred_at` (timestamp, UTC)
- `actor` (text: user name / household member label)
- `action` (text: Create / Edit / Delete / Import / Sample-clear / Convert-sample / Login / Undo)
- `table_name` (text: accounts / transactions / budgets / goals / bills / audit_log)
- `record_id` (text: id of the affected row, nullable for session events)
- `before_json` (text: JSON snapshot of the row before the change, nullable)
- `after_json` (text: JSON snapshot of the row after the change, nullable)
- `entry_method` (text: manual / import / sample / ai-suggested)
- `note` (text: free-text note, nullable)
- `is_bulk` (bool), `bulk_count` (number, nullable)
- `device_label` (text, nullable), `ip_label` (text, nullable)
- `is_sample` (bool — mirrors whether the underlying change touched sample rows)
- `day_hash` (text: SHA-256 chain hash for the day)

## Cross-cutting rules (apply to every screen in this section)
- Every screen shows the persistent banner: "Manual tracking — no bank connection yet."
- Any log entry about sample data carries is_sample=true and renders with a SAMPLE badge; hiding sample data hides those entries.
- Totals and counts derived from the log never present anything as a live bank feed; figures are manual entries.
- The audit log is append-only: no screen, query, or workflow may edit or delete `audit_log` rows. Nothing in this feature deletes data.

---

### 741. Record every create, edit, and delete across accounts, transactions, budgets, goals, bills.
**Components:** no UI of its own — this is infrastructure. Every existing mutation screen (transaction form, account CRUD, budget add/edit, goal contribute/edit, bill add/edit from section 15) gets a success-handler event `logAuditEntry` fired after the write query succeeds.
**Data:** JS query `qLogAudit` inserts one row into `audit_log` with action=Create/Edit/Delete, table_name, record_id, before_json (row fetched before write), after_json (row refetched after write), entry_method from context.
**Logic notes:** before-image fetched in the same event chain before the mutation; if refetch fails, after_json=null and the entry is still written (a partial entry beats a missing one). Delete entries keep the full row in before_json and null after_json.
**Acceptance:** 1) Creating, editing, and deleting a transaction, account, budget, goal, and bill each produce exactly one audit row. 2) before_json/after_json match the row state. 3) A failed mutation writes no entry.

### 742. Store actor, timestamp, table, record id, and before/after values per entry.
**Components:** none new — schema contract enforced by `qLogAudit`. A schema checklist is rendered in the viewer footer (item 743) showing which fields are required.
**Data:** `audit_log` columns: `actor` (from `current_user.fullName`, fallback "Me"), `occurred_at` (server timestamp `NOW()`), `table_name`, `record_id`, `before_json`, `after_json`. before/after stored as canonical JSON via `JSON.stringify(_.omit(row, transientKeys))`.
**Logic notes:** timestamps are UTC at write time; display converts to the user's locale. before/after JSON excludes UI-only fields (e.g. form dirty flags). Entries missing any of the five core fields are rejected by a Retool Database non-null constraint where applicable.
**Acceptance:** 1) Every row has non-null actor, occurred_at, table_name, record_id. 2) An edit shows differing before/after JSON; a create has null before_json. 3) Timestamps sort correctly across timezone changes.

### 743. Build an audit-log viewer with filters by table, date, and action type.
**Components:** new app page "Audit log": `tblAudit` (Table, columns: occurred_at, actor, action, table_name, record_id, entry_method, note), `selTable` (Select: All/accounts/transactions/budgets/goals/bills), `selAction` (Select: All/Create/Edit/Delete/Import/…), `dateFrom`/`dateTo` (Date pickers), `btnClearFilters` (Button).
**Data:** `qAuditList` — `SELECT * FROM audit_log WHERE ({{selTable.value}}='All' OR table_name={{selTable.value}}) AND ({{selAction.value}}='All' OR action={{selAction.value}}) AND occurred_at BETWEEN {{dateFrom.value}} AND {{dateTo.value}} ORDER BY occurred_at DESC LIMIT 500`.
**Logic notes:** filter changes re-trigger the query via event handlers; default date range = last 30 days. Row click opens a detail drawer (item 745 reuse). SAMPLE badge on is_sample rows.
**Acceptance:** 1) Filtering by table+action+date returns exactly matching rows. 2) 500-row cap with "refine filters" notice when hit. 3) Filters reset to defaults on Clear.

### 744. Make the audit log append-only; no edit or delete of log entries.
**Components:** `tblAudit` row actions: none — the table's Edit/Delete row actions are disabled and hidden. A lock icon + tooltip "Audit entries are permanent" in the viewer header.
**Data:** no write queries target `audit_log` except the single `qLogAudit` INSERT. Retool Database table permissions: the app role has INSERT and SELECT only on `audit_log`, no UPDATE/DELETE grants.
**Logic notes:** enforce at two layers: UI (no affordances) and data (DB grants). If a future workflow needs correction, it appends a compensating entry (item 746 undo), never mutates. Document this invariant in the viewer header note.
**Acceptance:** 1) No button, row action, or query in the app updates/deletes audit rows. 2) Attempting a direct UPDATE via the query editor fails on permissions. 3) The only mutation path is the INSERT query.

### 745. Show a per-record history timeline (e.g., all changes to one transaction).
**Components:** `drawerRecordHistory` (Drawer) with `listTimeline` (List View) of entries: date, action pill, actor, and a before→after diff rendered as key: old → new lines. Opened from any record's "History" button or from an audit row.
**Data:** `qRecordHistory` — `SELECT * FROM audit_log WHERE table_name={{ctxTable}} AND record_id={{ctxRecordId}} ORDER BY occurred_at ASC`. Transformer `xformDiff` computes changed keys between consecutive before/after JSON pairs.
**Logic notes:** first entry (Create) shows full after-image. Diff ignores unchanged keys and formatting-only differences. Timeline labels entries with entry_method icons.
**Acceptance:** 1) Opening history for an edited transaction shows Create then Edit entries in order. 2) The diff lists exactly the changed fields. 3) History works for deleted records (entries persist via before_json).

### 746. Add an undo action directly from an audit entry where safe.
**Components:** in `drawerRecordHistory` and `tblAudit` row detail: `btnUndo` (Button, visible only when `entry.action` is Create or Edit and `entry.is_bulk` is false). Confirm modal `modalUndo` explains what will be restored.
**Data:** `qUndo` — for Edit: `UPDATE <table> SET <before_json fields> WHERE id=<record_id>`; for Create: `DELETE FROM <table> WHERE id=<record_id>`. After success, `qLogAudit` writes an Undo entry referencing the original entry id in `note`.
**Logic notes:** "safe" = non-bulk, target row still exists (for Edit), no later dependent edits after this entry (checked via `qLaterEdits` count — if >0, button is disabled with explanation). Undo of a Delete is not offered (would resurrect data the user chose to remove).
**Acceptance:** 1) Undo of an Edit restores the exact before-values. 2) Undo of a Create deletes the created row. 3) Button is disabled when later edits exist. 4) The undo itself is logged as an Undo entry.

### 747. Export the audit log to CSV for personal record-keeping.
**Components:** `btnExportAuditCSV` (Button) in the viewer header; `selExportRange` (Select: Current filters / Last 30 days / All time) optional — defaults to current filter set.
**Data:** `qAuditExport` fetches rows (no 500 cap, chunked via pagination or a single query with a 10,000-row safety limit). Transformer `xformAuditCSV` flattens JSON fields to columns and triggers `utils.downloadFile`.
**Logic notes:** export honors the active filters; filename `financeos-audit-YYYYMMDD.csv`. Includes a header row stating "Manual tracking — no bank connection" and the export timestamp. before/after JSON serialized as single-quoted cells.
**Acceptance:** 1) CSV contains exactly the filtered rows with all core columns. 2) File downloads with the dated filename. 3) before/after columns are valid JSON per cell.

### 748. Display data-entry method per change: manual, import, sample, AI-suggested.
**Components:** `entry_method` shown as a colored pill in `tblAudit`, the record timeline, and the digest view. Legend in the viewer header: Manual (blue), Import (purple), Sample (gray), AI-suggested (green).
**Data:** `qLogAudit` sets `entry_method` from a global `auditContext.entryMethod` variable: form screens set "manual"; CSV import (section 24) sets "import"; sample-data generation sets "sample"; AI categorization (section 30) sets "ai-suggested". Defaults to "manual".
**Logic notes:** the variable is set per-flow before mutation and reset after, so cross-flow contamination can't occur. Filter `selMethod` added to the viewer (All + four methods).
**Acceptance:** 1) A manual edit shows "manual"; an imported row shows "import". 2) Sample-generated rows show "sample" with SAMPLE badge. 3) Method filter narrows the list correctly.

### 749. Highlight bulk operations in the log with affected-row counts.
**Components:** `tblAudit` renders bulk rows (`is_bulk=true`) with a "BULK ×N" badge and expanded row shows the operation summary. In the timeline, bulk entries are visually grouped with a left border.
**Data:** `qLogAuditBulk` — a single entry with `is_bulk=true`, `bulk_count=N`, `note` describing the operation (e.g. "CSV import: 120 rows"), `after_json=null`. Bulk entries are written once per operation, not per row.
**Logic notes:** bulk = CSV import batches, sample-data generation/clearing, month-end bulk actions. Per-row audit rows are NOT written for bulk ops (avoids log flooding); the bulk entry carries the count and the operation parameters in `note`.
**Acceptance:** 1) A 120-row CSV import writes exactly one bulk entry with bulk_count=120. 2) Bulk rows are visually distinct in the viewer. 3) Undo is disabled for bulk entries.

### 750. Add a "what changed this week" digest view of the audit log.
**Components:** `containerDigest` on the Audit log page (tab "This week"): summary cards (`statCreates`, `statEdits`, `statDeletes`, `statImports`) + `listDigest` grouped by table with top changes and `txtDigestSummary` auto-text (e.g. "12 edits, mostly transactions").
**Data:** `qDigest` aggregates `audit_log` for `occurred_at >= start_of_week`: counts by action and table, top-5 most-edited records. Transformer builds the summary sentences. No email — this is an in-app view only.
**Logic notes:** week starts Monday, user locale. Bulk entries counted once (by operation, not by bulk_count). Sample-data activity grouped separately under a SAMPLE heading.
**Acceptance:** 1) Digest shows correct counts for a week with known activity. 2) Empty weeks render "No changes this week." 3) No email or notification is sent — view only.

### 751. Show login/session events in the log for shared devices.
**Components:** session rows in `tblAudit` with a distinct "SESSION" action pill and a key icon; filterable via `selAction` = "Login".
**Data:** on app load, `qLogSession` checks a `sessionStorage` flag; if absent, inserts an entry (action=Login, table_name=null, record_id=null, device_label from `navigator.userAgent` parsed, note="Session started") and sets the flag. `actor` falls back to "Shared device" when no user name is set.
**Logic notes:** one entry per browser session, not per page load. Device label is a friendly parse (e.g. "iPhone · Safari"), never raw user-agent in the UI (raw kept in `ip_label`/`device_label` fields if needed).
**Acceptance:** 1) Opening the app in a fresh session writes exactly one Login entry. 2) Reloads in the same session write none. 3) Login entries filter correctly under action=Login.

### 752. Retain audit entries for at least 24 months with a retention notice.
**Components:** `txtRetentionNotice` in the viewer header: "Audit entries are kept for 24 months. Entries older than 24 months may be archived." A `statOldest` chip shows the oldest entry date.
**Data:** `qOldestEntry` — `SELECT MIN(occurred_at) FROM audit_log`. No automated deletion is built (append-only invariant, item 744) — the notice is honest: retention is guaranteed for 24 months, and any future archiving would be a separate, user-confirmed export-then-archive flow.
**Logic notes:** deliberately no auto-purge: silent deletion would violate the append-only promise. If the table grows large, the remedy is manual archive (export CSV, then a user-confirmed archive action — not in this section's scope).
**Acceptance:** 1) The retention notice is visible in the viewer. 2) Oldest-entry date is accurate. 3) No query in the app deletes entries older than 24 months.

### 753. Allow filtering the log to a single record's full history.
**Components:** `inpRecordId` (Text Input) + `btnGoToRecord` in the viewer toolbar; also "View full history" links on every record screen pass table+id via URL params. Result renders the same timeline component as item 745.
**Data:** reuses `qRecordHistory` with `ctxTable`/`ctxRecordId` set from the input. Deep-link: `auditLog?table=transactions&id=<uuid>` opens the drawer on page load via a URL-param reader query.
**Logic notes:** input accepts a pasted record id; invalid ids show "No history found for this id" rather than an error. Table defaults to the last-used table when only an id is given (best-effort match across tables).
**Acceptance:** 1) Pasting a transaction id shows its complete history. 2) Invalid id shows the friendly empty state. 3) Deep links open the correct record's timeline.

### 754. Display IP/device labels on entries when available.
**Components:** `device_label` and `ip_label` shown in the audit row detail drawer under "Session info"; a compact device icon in `tblAudit` when a label exists.
**Data:** `qLogAudit` captures `device_label` from a friendly user-agent parse at write time; `ip_label` left null in the Retool context (client IP isn't reliably available — the schema holds it for future native contexts, and the UI says "not captured" rather than fabricating one).
**Logic notes:** honest labeling: the drawer shows "Device: iPhone · Safari" when known and "IP: not captured in this app" otherwise. Never invent an IP. Labels are informational only, never used for access decisions.
**Acceptance:** 1) Entries written from the app carry a device label. 2) The drawer shows "not captured" for IP rather than a fake value. 3) Labels don't affect filtering or totals.

### 755. Add a tamper-evidence note: entries are hash-chained per day.
**Components:** `txtTamperNote` in the viewer header: "Entries are hash-chained per day — any gap or alteration breaks the chain." A `btnVerifyChain` (Button) runs verification and shows `txtChainStatus` (green "Chain intact" / red "Chain broken on <date>").
**Data:** `day_hash` = SHA-256(prev day's last hash + occurred_at + actor + action + table_name + record_id + after_json), computed in `qLogAudit` via a JS transformer using the Web Crypto API (async) before insert. `qVerifyChain` recomputes sequentially for a selected date range and compares.
**Logic notes:** hash uses the serialized entry fields, so any edit of a stored row breaks the chain. Verification is on-demand (not on every load) for performance. First entry of each day chains from the previous day's last hash; genesis uses a fixed seed string.
**Acceptance:** 1) Every entry has a non-null day_hash. 2) Verify reports "intact" on an unmodified log. 3) Manually altering a row's after_json (via query editor) makes Verify report the broken date.

### 756. Show who (which household member) made each change in shared mode.
**Components:** `actor` column in `tblAudit` already carries this; shared mode adds `selHouseholdMember` (Select: profile switcher in the app header, e.g. "Nic", "Partner") persisted to `localStorage`. Timeline and digest group/filter by actor.
**Data:** `qLogAudit` reads `actor` from `localStorage.householdMember || current_user.fullName || "Me"`. Viewer gets an actor filter `selActor` (All + distinct actors from `qDistinctActors`).
**Logic notes:** this is a lightweight label, not authentication — the UI says "labeled as" to stay honest. Switching member mid-session labels subsequent entries only; past entries keep their original actor.
**Acceptance:** 1) Changing the member switcher labels new entries with that name. 2) Actor filter lists all members who have entries. 3) UI copy says "labeled as", never "signed in as".

### 757. Provide a printable audit summary for a date range.
**Components:** `btnPrintSummary` (Button) in the viewer; `modalPrint` (Modal) with `dateFromP`/`dateToP` pickers and a print-optimized `htmlPrintSummary` (HTML component) that calls `window.print()` via a Print button.
**Data:** `qPrintSummary` aggregates the range: counts by action/table/actor, first/last timestamps, bulk ops list. Transformer renders a clean single-column HTML summary with the manual-tracking banner text and the generation date in the header.
**Logic notes:** print CSS hides app chrome (nav, filters, buttons) via `@media print`. Summary is aggregate-only (no raw before/after JSON) to keep it readable. Includes the 24-month retention notice and the tamper-chain status line.
**Acceptance:** 1) Print preview shows the aggregate summary for the chosen range. 2) Printed output excludes app navigation/buttons. 3) Header carries the manual-tracking notice and generation date.

### 758. Link sample-data deletions and conversions as first-class log events.
**Components:** in the sample-data management UI (section 18): "Clear sample data" and "Convert to real" buttons fire `qLogAuditBulk` with action=Sample-clear or Convert-sample, plus per-table counts in `note`.
**Data:** entries use action=Sample-clear / Convert-sample, `is_sample=true`, `table_name` per affected table, `bulk_count` = rows affected, `note` = e.g. "Cleared 15 sample transactions". A filter chip `selAction` includes these action types.
**Logic notes:** these are bulk entries (no per-row flood). Convert-sample writes one entry per table converted. The entries render with the SAMPLE badge and are hidden when the sample toggle hides sample data.
**Acceptance:** 1) Clearing sample data writes Sample-clear entries with correct counts. 2) Converting writes Convert-sample entries. 3) They appear under the action filter and honor the sample-data toggle.

### 759. Add search across audit log notes and values.
**Components:** `inpAuditSearch` (Text Input, search icon) in the viewer toolbar; searches `note`, `actor`, `record_id`, and the before/after JSON text. Debounced 300ms, minimum 2 characters.
**Data:** `qAuditSearch` — `SELECT * FROM audit_log WHERE note ILIKE {{pattern}} OR actor ILIKE {{pattern}} OR record_id ILIKE {{pattern}} OR before_json ILIKE {{pattern}} OR after_json ILIKE {{pattern}} ORDER BY occurred_at DESC LIMIT 200`, where pattern = `%term%`.
**Logic notes:** search composes with the table/action/date filters (AND). JSON search matches on serialized text (e.g. searching "Whole Foods" finds entries whose before/after JSON contains it). Empty/short input falls back to the standard filtered list.
**Acceptance:** 1) Searching a description finds entries whose values contain it. 2) Search respects active filters. 3) Under 2 characters, the normal filtered list shows.

### 760. Surface the audit log link in settings under "Data & privacy."
**Components:** in the Settings page (section 50 area): `sectionDataPrivacy` containing `linkAuditLog` (Link/Button "View audit log") navigating to the Audit log page, plus `txtPrivacySummary` (entries count, oldest entry, retention notice) and `btnExportAuditCSV` duplicated here.
**Data:** `qPrivacyStats` — `SELECT COUNT(*) AS total, MIN(occurred_at) AS oldest FROM audit_log`. The link is a simple page navigation (`utils.openPage('auditLog')`); no data transfer needed.
**Logic notes:** Data & privacy groups: audit log link, retention notice, export button — one place for "what do you know about my changes". The summary chips reuse the same stats queries as the viewer to stay consistent.
**Acceptance:** 1) Settings shows a "Data & privacy" section with a working audit-log link. 2) Stats (count, oldest) match the viewer. 3) Export from settings produces the same CSV as from the viewer.
