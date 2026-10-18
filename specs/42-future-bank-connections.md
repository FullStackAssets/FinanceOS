# FinanceOS Builder Spec — §42 Future Bank Connections (opt-in) (Items 821–840)

## Section overview
DESIGN SPEC ONLY for a future optional feature. FinanceOS today has NO bank connection, and nothing in this spec implies one exists. If the user ever chooses to connect an account, connection is strictly opt-in: the user initiates it, the user enters credentials only inside the provider's own OAuth flow (e.g. Plaid Link), and Retool never sees or stores bank login credentials. This section specifies the connections screen, the consent flow, the read-only scope policy, first-sync import and duplicate detection, disconnect/purge behavior, connection health display, the manual-vs-connected distinction rules, the recurring-template migration helper, audit logging, indefinite hybrid-mode support, the FAQ, annual re-consent, and the manual-only default.

## Tables touched
- New table `bank_connections`: id (pk, text/UUID), account_id (fk → accounts.id, nullable — a connection can be scoped per account), institution_name (text), status (text: not_connected / pending / connected / paused / error), scopes_granted (text, e.g. "transactions:read"), access_token_ref (text — a token-vault reference, never a raw token in Retool state), last_sync_at (timestamp, nullable), sync_error (text, nullable), consent_expires_at (date, nullable), created_at (timestamp), updated_at (timestamp), is_sample (bool — demo rows for builder QA are clearly marked SAMPLE).
- New table `import_batches`: id (pk, text/UUID), bank_connection_id (fk), imported_at (timestamp), rows_imported (number), rows_skipped_duplicates (number), status (text: completed / purged).
- Extensions to existing tables: `accounts.data_source` (text: manual / imported / synced / sample, default manual), `transactions.data_source` (same, default manual), `transactions.import_batch_id` (fk → import_batches.id, nullable).
- Writes to `audit_log` (schema defined in the §38 spec: occurred_at, actor, action, table_name, record_id, before_json, after_json, entry_method, note). New actions used here: Connect, Disconnect, Sync, Purge.

## Shared cross-cutting rules
- With zero connections, every screen keeps the exact persistent banner "Manual tracking — no bank connection yet."
- After connecting, a banner remains visible on every screen (per item 829) noting which data is manual vs synced; nothing ever presents as a live bank feed.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge and is excluded from totals when the sample toggle hides them — this applies to demo bank_connections and import rows too.
- Retool never collects bank credentials; only provider-issued tokens (stored as vault references) are kept. Synced rows are always timestamped; a balance without a last-sync timestamp is never presented as verified.

---

### 821. Design a connections screen that states "No bank connected" until the user opts in.
- Components: New screen "Connections" (nav item under Settings/More). Hero card `connStatusCard` with large status text; with no connections it reads exactly "No bank connected". Button `btnConnectBank` ("Connect a bank — optional"), secondary link "Compare manual vs connected" (→ item 823), and "How it works" (→ item 838 FAQ).
- Data: Query `qryConnections` reads `bank_connections`; status text = "No bank connected" when count = 0, otherwise a per-connection table below (institution, status, last sync).
- Logic notes: The empty state is the default and must never show a provider logo wall, "recommended bank" upsell, or skeleton of a connected account — the absence of a connection is presented as normal, not as a problem to fix.
- Acceptance: (1) Fresh install shows exactly "No bank connected". (2) No credential fields exist on this screen. (3) The connect button opens the opt-in consent flow (item 822), never a login form.

### 822. Require explicit opt-in consent with plain-language scope before any bank linking.
- Components: Modal `modalConsent` opened from `btnConnectBank`. Plain-language checklist: what is read (account names, balances, transaction history), what is NEVER done (no transfers, no payments, no selling data), how to disconnect, data-retention note. Two required checkboxes (scope understood; disconnect-anytime understood) and buttons "I agree — continue" / "Cancel". No pre-checked boxes.
- Data: On agree, JS query inserts a `bank_connections` row (status='pending', scopes_granted='transactions:read') recording consent, then launches the provider OAuth flow (item 824). Cancel inserts nothing.
- Logic notes: Our plain-language consent is separate from and precedes the provider's own OAuth screen — the user sees OUR scope summary first. Canceling at any point must leave no lingering 'pending' row beyond the session.
- Acceptance: (1) Both checkboxes required before Continue enables. (2) Cancel leaves no pending connection row. (3) Scope text names read-only access explicitly and lists what is excluded.

### 823. Show a comparison table: manual tracking vs connected tracking trade-offs.
- Components: Table (two-column cards on mobile) `tblCompare` with rows Effort, Freshness, Accuracy, Privacy, Control; columns "Manual (current)" and "Connected (optional)". Linked from the Connections screen and onboarding.
- Data: Static content transformer `tfCompareRows`; no queries.
- Logic notes: Honest framing — manual wins on privacy/control/zero-setup; connected wins on effort/freshness. No row may present manual as inferior overall; the table's purpose is informed choice, not conversion. Footnote: "You can stay on manual forever — nothing degrades if you never connect" (ties to item 836).
- Acceptance: (1) All five rows present with neutral wording. (2) Table links to the consent flow and back. (3) No upsell language ("upgrade", "unlock") appears anywhere on the table.

### 824. Support read-only connection scopes only; never request transfer or payment scopes.
- Components: Builder-config constant `ALLOWED_SCOPES = ['transactions:read','accounts:read','balances:read']`; the OAuth launch function asserts requested scopes ⊆ ALLOWED_SCOPES before opening the provider flow. Settings shows "Scopes: read-only" as immutable text.
- Data: `scopes_granted` stored on `bank_connections` at connect time; query `qryScopeAudit` lists all connections with their scopes for the health screen.
- Logic notes: This is a policy-level guard, not just UI copy — any code path requesting a scope outside the allow-list throws and writes an audit_log row (action=Sync, note='scope rejected'). Re-auth flows (item 839) re-assert the same allow-list.
- Acceptance: (1) OAuth launch fails closed if a non-read scope is configured. (2) `scopes_granted` on every connected row contains only read scopes. (3) The scope audit query returns all connections with matching scopes.

### 825. On connect, import history as clearly-marked imported rows, never overwriting manual entries.
- Components: First-sync wizard step `wizardImport` with summary "N transactions found · M already in FinanceOS as manual entries", an Import button, and a post-import report. Imported rows render with an "Imported" badge plus the source institution in the transaction list.
- Data: `qryImportBatch` creates an `import_batches` row, then bulk-inserts transactions with data_source='imported', import_batch_id set, is_sample=false; touched account balances get data_source='synced'. Manual rows are never updated or deleted by import.
- Logic notes: Import is additive-only. If a manual account and a synced account represent the same real account, both rows persist and the user merges or keeps both explicitly — no automatic overwrite, no silent balance replacement (ties to items 826 and 830).
- Acceptance: (1) After import, all pre-existing manual rows are unchanged (verifiable via audit_log before/after). (2) Every imported row carries data_source='imported' and an Imported badge. (3) Sync-updated balances show data_source='synced'.

### 826. Add a duplicate-detection pass when first syncing to avoid double-counting manual rows.
- Components: Dedupe review table `tblDedupe` in the import wizard: imported row, matched manual row, match score, action (Link / Keep both / Skip import). "Link" keeps the manual row (data_source stays 'manual') and skips the import row, recording it as linked.
- Data: JS transformer `tfDedupe` matches on account + date ±2 days + equal |amount| + description similarity ≥ 0.7. Skipped counts roll into `import_batches.rows_skipped_duplicates`.
- Logic notes: Default suggestion is Link for high-confidence matches (score ≥ 0.9), Keep both below that — the user confirms; nothing auto-merges without confirmation. Totals show a "review duplicates before totals are final" warning during review.
- Acceptance: (1) Known manual duplicates are proposed as Link with score ≥ 0.9. (2) No row merges without user confirmation. (3) `rows_skipped_duplicates` equals the number of linked/skipped rows.

### 827. Allow disconnecting at any time with an option to keep or purge synced data.
- Components: Per-connection "Disconnect" button opens confirm modal `modalDisconnect` with radio choice: "Keep synced data (marked as imported)" / "Purge all synced data from this connection", plus an explicit "Disconnect" confirm button.
- Data: Disconnect query sets `bank_connections.status`='not_connected' and clears `access_token_ref`. Purge deletes transactions whose import_batch_id belongs to the connection's batches and resets affected accounts' data_source to 'manual' (balance values stay). audit_log records action=Disconnect or Purge with counts.
- Logic notes: Purge never touches data_source='manual' rows — only imported/synced rows from that connection's batches. Keep flips data_source='synced' → 'imported' so provenance stays honest. Disconnect is one tap from the Connections screen, not buried in settings.
- Acceptance: (1) Purge removes only that connection's imported/synced rows; manual rows intact. (2) After disconnect with zero connections remaining, the status card returns to "No bank connected". (3) audit_log records the disconnect with the keep/purge choice.

### 828. Display connection health (last sync time, errors) on the accounts screen.
- Components: On the Accounts screen, each connected account row shows a status dot plus "Synced 2h ago" or "Sync failed — tap to retry". A top banner appears when any connection is paused or in error: "1 connection needs attention".
- Data: Query `qryConnHealth` joins accounts → bank_connections (status, last_sync_at, sync_error); relative freshness text computed in JS transformer `tfFreshness`.
- Logic notes: Freshness reads relative ("12 min ago", "3 days ago") with the exact timestamp in a tooltip. Sync older than 48h shows an amber dot; any sync_error shows red with the provider's message in plain language. Manual accounts show no health UI — no fake status.
- Acceptance: (1) Every connected account shows its last-sync time. (2) Failed sync surfaces the error message and a retry button. (3) Manual accounts show no connection-health elements.

### 829. Keep the manual-entry banner visible even after connecting, noting which data is manual.
- Components: Persistent banner component `bannerTracking` on every screen. Zero connections → exact text "Manual tracking — no bank connection yet." One or more connections → "Hybrid tracking: bank data synced {time}; manual entries marked Manual."
- Data: Banner text computed from `qryConnections` count and max(last_sync_at).
- Logic notes: The banner must never disappear after connecting — the provenance split stays visible at a glance. Per-row badges (Manual / Imported / Synced / SAMPLE) carry the detail; the banner carries the headline. This rule overrides any generic "hide banner when connected" instinct.
- Acceptance: (1) Banner present on all screens with ≥1 connection. (2) Text names the last sync time and distinguishes manual entries. (3) Zero-connection state keeps the original exact banner text.

### 830. Support per-account connection toggles: connect checking but keep savings manual.
- Components: On the Connections screen, each connection expands to per-account toggle rows `toggleAcctSync` (Checking on / Savings off style). Toggling off stops sync for that account while the connection stays live for the others.
- Data: One `bank_connections` row per connected account (account_id set) plus a parent institution row (account_id null); a toggle flips status connected↔not_connected on the account row without touching siblings.
- Logic notes: Toggling an account off keeps its previously synced rows (re-marked imported) and stops future syncs for it; toggling back on resumes with a fresh dedupe pass (item 826). The Accounts screen visually separates manual and connected account groups.
- Acceptance: (1) Disabling savings sync leaves checking syncing. (2) Re-enabling triggers a dedupe review, not a blind re-import. (3) The Accounts screen groups manual and connected accounts separately.

### 831. Show a "connection paused" state when sync fails, with manual-entry fallback prompts.
- Components: "Paused" status pill on the connection row plus a banner on affected accounts: "Sync paused — your manual entries still count." with buttons "Add manual transaction" and "Retry sync".
- Data: Failed syncs set `bank_connections.status`='paused' and sync_error=message; `qryPausedConns` drives the banner.
- Logic notes: Pause triggers automatically after ≥3 consecutive sync failures, and manually via a "Pause syncing" button. While paused, totals use synced rows as-of last_sync_at (stale, timestamped per item 832) and the app prompts manual entry so the user never has an invisible gap. Resume is one tap and re-runs auth if the token expired.
- Acceptance: (1) After 3 failed syncs the connection shows Paused automatically. (2) Paused state shows the manual-entry fallback CTA. (3) Totals never silently treat stale synced data as current.

### 832. Never present synced balances as verified without a last-sync timestamp.
- Components: Every synced balance renders as "$4,210.33 · synced 2h ago" (small muted timestamp under the figure) via shared formatter `fmtSyncedBalance(balance, lastSync)` used in all balance displays; manual balances show no timestamp.
- Data: `last_sync_at` from `bank_connections`; the formatter applies a "stale — last synced {date}" style when older than 48h.
- Logic notes: Rendering rule with no exceptions — net worth, account cards, and the overview hero all use the formatter for synced figures. The word "verified" or a checkmark badge may never appear next to a synced balance — only the timestamp. SAMPLE balances keep the SAMPLE badge instead.
- Acceptance: (1) No synced balance renders without its last-sync timestamp. (2) Balances older than 48h show the stale style. (3) The word "verified" appears nowhere near synced figures.

### 833. Add a migration tool converting manual recurring templates to sync-matched rules.
- Components: Tool screen "Migrate recurring" (from Connections or the Recurring section): table pairing each manual recurring template with candidate synced transactions (match preview), buttons "Convert to sync-matched" / "Keep manual". Converted rules show a "Synced" origin badge.
- Data: Matcher JS `tfRecurringMatch` compares template (amount, cadence, description keywords) against synced transactions from the last 90 days; conversion updates the recurring template row (origin='synced', linked account) without deleting history.
- Logic notes: Conversion is per-template and reversible ("Revert to manual") — the original manual schedule is kept in the template's history note. Only templates on connected accounts are eligible; manual-only accounts' templates are hidden from this tool with an explanatory note.
- Acceptance: (1) Only connected accounts' templates are eligible. (2) Each conversion is reversible with history preserved. (3) Converted rules are badged as sync-matched, not manual.

### 834. Log every connect, disconnect, and sync event in the audit log.
- Components: No new UI — event hooks on connect, disconnect, purge, sync, pause, resume, and re-consent actions each fire `qLogAudit` after success.
- Data: `audit_log` rows with action in (Connect, Disconnect, Sync, Purge), table_name='bank_connections', record_id=connection id, entry_method='import' for syncs, note carrying counts ("42 imported, 3 duplicates linked") or error text.
- Logic notes: Sync events log at most one summary row per run — per-row provenance lives in `import_batches`. Failed syncs also log (action=Sync, note=error) so silent failures are impossible. audit_log stays append-only per the §38 rules.
- Acceptance: (1) Connect/disconnect each produce exactly one audit row. (2) Each sync run logs a summary row including error text on failure. (3) No audit rows are editable or deletable.

### 835. Provide a data-source column (manual/imported/synced/sample) on all tables post-connect.
- Components: `data_source` column added to the transactions table, accounts table, and list views; renders as a colored pill (Manual grey / Imported blue / Synced green / SAMPLE amber). Filter chips "All / Manual / Imported / Synced" above the transactions table.
- Data: Reads the `data_source` columns added in this section's schema; a one-time migration sets existing rows to 'manual' (or 'sample' where is_sample=true).
- Logic notes: The column appears only when ≥1 connection or any imported rows exist — manual-only installs never see it (avoids clutter, supporting item 840's manual-first default). Exports include the column so provenance survives outside the app. Trends/aggregations can group by source.
- Acceptance: (1) Column hidden on manual-only installs, visible once imported/synced rows exist. (2) Filter chips filter correctly by source. (3) CSV/PDF exports include the data-source value per row.

### 836. Allow running in hybrid mode indefinitely with no pressure to connect.
- Components: No new UI — a negative requirement enforced across the Connections screen, onboarding, and banners: no countdowns, no "X days left on manual", no degraded-manual threats, no persistent "Connect now!" prompts. The empty state stays calm and informational.
- Data: None — policy. A builder checklist step (manual review) scans app text for banned pressure phrases ("upgrade", "unlock", "limited", "don't miss out").
- Logic notes: Hybrid (some accounts connected, some manual) is a first-class permanent state, not a funnel stage. Feature announcements for connection features must include the manual equivalent or state "manual users: nothing changes". This item backstops items 823, 829, and 840.
- Acceptance: (1) No time-limited or scarcity language anywhere in connection flows. (2) Manual-only users see zero repeated connect prompts after dismissing once. (3) Hybrid state has no "finish connecting" progress bar.

### 837. Show which features improve with connection (auto-categorization) vs stay manual.
- Components: "What changes if I connect?" card on the Connections screen with two lists — "Gets better" (auto-categorization of synced transactions, balance freshness, duplicate detection) and "Stays the same" (budgets, goals, manual entry, reports, privacy).
- Data: Static content; each "gets better" row deep-links to the feature screen.
- Logic notes: Honest scoping prevents the top disappointment driver (expecting budgets/goals to automate). The auto-categorization note must say it applies to synced rows only and the user can override any suggestion. No feature may be described as "requires connection" unless it genuinely cannot work manually.
- Acceptance: (1) Both lists present with at least 3 items each. (2) No manual-capable feature is listed as connection-required. (3) Auto-categorization is described as suggestion-only with override.

### 838. Add a connection FAQ addressing security, scope, and how to disconnect.
- Components: FAQ accordion `faqConnections` on the Connections screen covering at least: How do my credentials stay safe? (provider OAuth; Retool never sees them) What exactly can FinanceOS read? (read-only list) Can it move money? (no — transfers never requested) How do I disconnect? (one tap, keep-or-purge) What happens to my data if I disconnect? Does it cost anything or slow the app?
- Data: Static content component; the "How do I disconnect?" answer deep-links to the disconnect button.
- Logic notes: Answers must be verifiable against the actual implementation (scopes from item 824, disconnect from item 827) — the FAQ is a spec contract, not marketing. Any implementation change to scopes or retention requires a matching FAQ update; flag this as a builder checklist item.
- Acceptance: (1) All six questions present with accurate, implementation-matching answers. (2) The disconnect answer links to the real disconnect control. (3) No answer contradicts the read-only scope policy.

### 839. Require re-consent annually for continued bank access.
- Components: 30 days before `consent_expires_at`, a banner plus Connections-screen notice: "Bank access expires on {date} — re-confirm to keep syncing." A re-consent button re-opens the item-822 consent modal (abbreviated: scope-unchanged confirmation) and extends `consent_expires_at` by 12 months. If expired, status flips to 'paused' with the item-831 fallback prompts.
- Data: Query `qryConsentExpiry` lists connections with consent_expires_at < today+30d; a daily Retool workflow sets expired rows to paused and logs audit action=Sync with note='consent expired'.
- Logic notes: Re-consent must not re-request broader scopes — the item-824 allow-list is re-asserted. Expiry never deletes data; it only pauses syncing. The user can also proactively re-consent early from the connection row.
- Acceptance: (1) Notice appears 30 days before expiry. (2) Expired consent pauses syncing without deleting data. (3) Re-consent extends exactly 12 months and re-asserts read-only scopes.

### 840. Default all new installs to manual-only; connection is never the default.
- Components: First-run state — zero `bank_connections` rows, banner shows "Manual tracking — no bank connection yet.", Connections screen shows the item-821 empty state. No onboarding step enables or pre-configures a connection; the provider SDK is not initialized until the user opens the consent flow.
- Data: The seed/first-run script creates no `bank_connections` rows; `qryConnections` returning empty is the expected healthy state, not an error.
- Logic notes: This is the architectural default that makes items 821, 823, and 836 honest — every downstream spec assumes manual-only unless the user acted. Builder QA must test the full app with zero connections and confirm no broken states, no "connect to continue" gates, and no errors from the uninitialized provider SDK.
- Acceptance: (1) Fresh install has zero connection rows and full app functionality. (2) No onboarding screen presents connecting as a required or recommended step. (3) The provider SDK initializes only after explicit consent.
