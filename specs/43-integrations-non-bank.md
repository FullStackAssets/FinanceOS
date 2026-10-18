# Section 43 — Integrations (non-bank) — Builder Specs

**FinanceOS builder-ready specs for roadmap items 841–860.** These are DESIGN SPECS ONLY for optional future integrations. All credential/account linking is performed by the user themselves in their own accounts; the app never handles, stores, or transmits credentials. Nothing here implies or enables live bank connections.

## Tables touched

Base tables (from App 11): `accounts`, `transactions`, `budgets`, `goals` (all with `is_sample`).

New tables introduced in this section:

- `integrations`: id (pk), provider (text: Calendar / EmailInbox / Spreadsheet / TaskApp / WebhookIn / WebhookOut / CloudStorage / OCR / iOSShortcut / ReadOnlyAPI / AccountingExport / Bookmarklet), status (text: Connected / Disconnected / Revoked), scopes_granted (text), connected_at (timestamp), last_sync_at (timestamp, nullable), created_by (text: 'user'), note (text)
- `webhook_endpoints`: id (pk), direction (text: Inbound / Outbound), label (text), secret_hint (text — last 4 chars only, never full), event_filter (text: transaction.create / goal.contribute / bill.due), is_enabled (bool), created_at (timestamp)
- `integration_activity`: id (pk), integration_id (fk → integrations.id, nullable), kind (text: Sync / Push / Webhook / Export / Consent), detail (text), rows_affected (number), ran_at (timestamp), actor (text: 'user' / 'automation')
- `draft_inbox`: id (pk), source (text: EmailForward / WebhookIn / Shortcut / Bookmarklet), raw_payload (text), suggested_description (text), suggested_amount (number, nullable), suggested_category (text, nullable), received_at (timestamp), status (text: Draft / Promoted / Discarded)

## Shared cross-cutting rules

- Every screen shows the persistent banner: **"Manual tracking — no bank connection yet."**
- Sample rows carry `is_sample=true` + a SAMPLE badge, and are excluded from totals when the sample toggle hides them.
- No screen, feed, export, or sync may present data as a live bank feed. All integration surfaces are labeled "optional, user-connected."
- No integration may be enabled without explicit user action, and enabling one always shows its data scope in plain language first.
- Secrets (API keys, webhook secrets) are never stored in full; only a last-4 hint is kept, per item 851/857 rules.

---

### 841. Integrate with calendar apps to surface bill due dates (read-only, user-authorized).

**Components:** "Integrations" screen (section 855) hosts a Calendar connector card: Toggle `cal_enabled`, Text `cal_scope_note` (fixed text "Reads bill due dates only — cannot see your personal events"), Button `btn_open_calendar_auth` (opens the provider's own authorization page in a new tab; the user pastes the issued read-only token into a Password input `cal_token_input` themselves), Button `btn_sync_bills_now`.

**Data:** Query `upsert_integration` writes `integrations` row (provider='Calendar', status='Connected', scopes_granted='read bill reminders'). Query `fetch_due_bills` pulls bills with `due_date >= today` from the user's authorized calendar via the provider API using the user-supplied token (token lives only in a temp state variable, never persisted). Transformer `bills_to_events` maps to {title, date, amount}.

**Logic notes:** Read-only means the sync never writes to the calendar. If the token is invalid, the card shows "Authorization failed — re-connect" and reverts status to Disconnected. The sync reads bill-reminder events only, never the user's full calendar. Sample bills are skipped when the sample toggle hides them.

**Acceptance:** (1) Connecting requires the user to paste their own token from the provider's page — no auto-connect path exists. (2) A sync with a valid token surfaces due bills on the Bills screen within 10 seconds. (3) An invalid/revoked token shows a clear error and marks the integration Disconnected.

### 842. Add email-to-inbox: forward receipts to a personal address to create draft transactions.

**Components:** Integrations screen card for Email Inbox: Text `inbox_address_display` showing a generated inbound address (e.g. `financeos+<random>@inbox.example`), Button `btn_copy_inbox_address`, Table `tbl_draft_inbox` (columns: received_at, suggested_description, suggested_amount, suggested_category, status), Button `btn_promote_draft` and Button `btn_discard_draft` per row.

**Data:** Query `list_drafts` reads `draft_inbox` where status='Draft'. Query `promote_draft` inserts into `transactions` (is_sample=false) and sets draft status='Promoted'. Query `discard_draft` sets status='Discarded'. Inbound parsing is a design hook: the forwarding address is hosted by the user's own email provider; a JS snippet `parse_receipt_subject` extracts amount via regex `/\$?(\d+\.\d{2})/` from subject/body to prefill suggestions.

**Logic notes:** Drafts never become transactions automatically — every row needs an explicit Promote click. The inbound address is unique per user so stray mail can't pollute another ledger. Body text is stored only in `draft_inbox.raw_payload` until promoted or discarded; discards are soft-deleted (status) not purged.

**Acceptance:** (1) A forwarded receipt appears as a Draft row within 5 minutes with amount and description prefilled. (2) Promote creates a real transaction and marks the draft Promoted; Discard hides it from the inbox. (3) No email can create a transaction without a Promote click.

### 843. Support spreadsheet sync: push monthly summaries to a connected sheet on demand.

**Components:** Integrations screen card for Spreadsheet: Button `btn_push_summary` (disabled unless a sheet is linked), Text `sheet_link_note` ("You authorize the sheet in your own Google/Microsoft account — paste the share link"), Text input `sheet_url_input`, Select `summary_month` (defaults to current month), Table `tbl_push_preview` showing the summary rows before pushing.

**Data:** Query `upsert_integration` (provider='Spreadsheet', scopes_granted='write monthly summaries only'). Transformer `monthly_summary` aggregates `transactions` by category for the selected month (respecting the sample toggle) into {category, spent, budget, remaining}. The push itself is a manual two-step design: the app generates the summary table on screen; the user copies it or uses their sheet's own import — no server-side sheet API credential is stored.

**Logic notes:** On demand only — there is no scheduled auto-push. The preview table must match the pushed data exactly. If the month has no transactions, the preview shows an empty-state message instead of a blank push. Exported sheets carry the "Manual tracking" label in the header row.

**Acceptance:** (1) Preview shows one row per category with spent/budget/remaining for the chosen month. (2) Push is only possible after the user pastes their own sheet link. (3) Nothing is pushed automatically on any schedule.

### 844. Integrate with a task app to create bill-pay checklist items.

**Components:** Bills screen: per-bill Button `btn_send_to_tasks` ("Send to task app"). Integrations screen card for Task App: Text `tasks_scope_note` ("Creates checklist items in your task app — cannot read or delete your tasks"), Text input `tasks_webhook_url` (user pastes their task app's incoming webhook or share URL), Toggle `taskapp_enabled`.

**Data:** Query `upsert_integration` (provider='TaskApp', scopes_granted='create items only'). Query `log_integration_activity` records each send in `integration_activity`. The actual delivery uses the user's own task app URL pasted by them; payload is {title: "Pay <bill> — $<amount> due <date>", due: date}.

**Logic notes:** One click = one item; there is no bulk auto-creation and no sync-back of completion state (completion is tracked manually in FinanceOS if desired). If the URL is blank or invalid, the button shows "Configure your task app first" and opens the integrations card. Sending twice creates two items — the button disables for 3 seconds after click to prevent double-taps.

**Acceptance:** (1) Clicking the button on a bill creates exactly one checklist item in the user's task app. (2) With no URL configured, the button directs the user to setup instead of failing silently. (3) Each send is recorded in `integration_activity`.

### 845. Add a webhook receiver for user-built automations (e.g., "log $5 coffee").

**Components:** Integrations screen card for Inbound Webhook: Text `inbound_url_display` (unique per-user URL), Button `btn_regenerate_inbound` (with confirm modal — old URL stops working), Button `btn_copy_inbound_url`, Table `tbl_inbound_log` (from `integration_activity` where kind='Webhook'), Table `tbl_draft_inbox` filtered to source='WebhookIn'.

**Data:** Table `webhook_endpoints` row (direction='Inbound', label user-editable, is_enabled). Inbound posts land in `draft_inbox` with status='Draft' — they NEVER write directly to `transactions`. A JS validator `validate_inbound_payload` requires {description: string, amount: number}; rejects anything else with a 400-style error row logged in `integration_activity`.

**Logic notes:** The inbound URL contains an unguessable token segment; regenerating it invalidates the old one immediately. Payloads are capped at 10 KB to prevent abuse. Every received call is logged even if the payload is invalid, so the user can debug their automation. Only the secret hint (last 4) is shown.

**Acceptance:** (1) A valid POST with description+amount creates a Draft inbox row, never a transaction. (2) Regenerating the URL disables the old one — a POST to the old URL is rejected and logged. (3) Malformed payloads are rejected with a logged error the user can inspect.

### 846. Provide an outgoing webhook on transaction create for personal dashboards.

**Components:** Integrations screen card for Outbound Webhook: Form with Text input `out_url` (user's own endpoint), Multiselect `out_events` (transaction.create, goal.contribute, bill.due), Toggle `out_enabled`, Button `btn_test_outbound` ("Send test ping"), Table `tbl_outbound_log` (from `integration_activity` where kind='Webhook' and direction implied).

**Data:** Table `webhook_endpoints` row (direction='Outbound'). Trigger design: after `insert_transaction` succeeds, a JS query `fire_outbound` POSTs {event, transaction_id, date, description, category, amount, is_sample} to the user's URL, then `log_integration_activity` records success/failure. Test ping sends {event:'test'} only.

**Logic notes:** Delivery is best-effort with one retry after 30 seconds; failures are logged, never block the transaction save. Payloads never include account names or balances beyond the transaction itself — the event filter lets the user restrict to categories. Disabling the toggle stops all sends immediately. Signing uses an HMAC with the user-generated secret (only hint stored).

**Acceptance:** (1) Creating a transaction with the webhook enabled delivers a payload to the user's URL within 30 seconds. (2) A failed delivery is logged and does not block or roll back the transaction. (3) Test ping delivers a test event without creating any transaction.

### 847. Support importing from budgeting-app CSV exports (Mint/YNAB formats).

**Components:** CSV Import screen: Select `import_preset` (options: Generic / Mint / YNAB / "Detect automatically"), File button `import_file`, Table `tbl_import_preview` (mapped columns with per-row validity icon), Button `btn_confirm_import`, Text `preset_mapping_note` showing the detected column mapping.

**Data:** Transformer `detect_format` inspects headers: Mint has `Date,Description,Original Description,Amount,Transaction Type,Category`; YNAB has `Date,Payee,Category,Memo,Outflow,Inflow`. Transformer `normalize_import` maps both to {date, description, amount (negative=spend), category}. Query `bulk_insert_transactions` inserts with is_sample=false; all rows land only after Confirm.

**Logic notes:** YNAB Outflow/Inflow columns are combined: amount = Inflow − Outflow. Mint "Transaction Type" credit/debit flips the sign when Amount is unsigned. Unknown categories fall back to "Other" and are flagged in the preview. Duplicate detection hashes date+description+amount and warns before inserting. The preset never auto-imports on file select — preview first.

**Acceptance:** (1) A Mint CSV and a YNAB CSV both preview with correct amounts and categories before any insert. (2) Duplicates already in `transactions` are flagged in preview, not silently inserted. (3) Nothing is written until Confirm is clicked.

### 848. Add a browser bookmarklet that opens quick-add with a prefilled description.

**Components:** Integrations screen card for Bookmarklet: Text `bookmarklet_code` (a read-only code block containing the `javascript:` snippet), Button `btn_copy_bookmarklet`, Text `bookmarklet_instructions` (numbered: bookmark any page → edit → paste code as URL), quick-add modal gains URL params support (`?desc=...`).

**Data:** No new tables. The bookmarklet JS reads `document.title` and the user's selected text, then opens the FinanceOS quick-add URL with `?desc=<encoded selection or title>`. The quick-add form's default value for description is set from `utils.getUrlParam('desc')`; amount and category remain blank for the user to fill.

**Logic notes:** The bookmarklet is pure client-side JavaScript — it never reads page cookies or sends data anywhere except opening the app URL. Prefill is description-only; prefilling amounts from arbitrary pages would be unreliable and is out of scope. If no text is selected, it falls back to the page title. Works in any desktop or mobile browser that supports bookmarklets.

**Acceptance:** (1) Installing the snippet per the instructions and clicking it on any page opens quick-add with the description prefilled. (2) The snippet contains no network calls — it only opens the app URL. (3) Submitting still requires the user to enter amount and category.

### 849. Integrate with cloud storage for attachment backup (user-authorized, encrypted).

**Components:** Integrations screen card for Cloud Storage: Select `storage_provider` (iCloud Drive / Google Drive / Dropbox / OneDrive), Button `btn_open_storage_auth` (opens the provider's own authorization page), Text `storage_scope_note` ("Reads and writes one FinanceOS backup folder — nothing else"), Button `btn_backup_now`, Text `last_backup_status`.

**Data:** Query `upsert_integration` (provider='CloudStorage', scopes_granted='single backup folder only'). Query `log_integration_activity` per backup. Design: the app zips receipt attachments + a manifest CSV; the user uploads the zip through their provider's own picker (file input), or grants folder access via the provider's OAuth page they complete themselves. The app never stores provider tokens.

**Logic notes:** Backup is manual ("Backup now") — no scheduled uploads. The zip is created client-side before upload; filenames include the transaction date for traceability. If attachments were deleted locally, the manifest notes the gap rather than failing. Restore is manual download + re-attach; no auto-restore.

**Acceptance:** (1) "Backup now" produces a zip containing all attachments plus a manifest, ready for the user's own upload. (2) The app stores no cloud provider tokens — authorization happens on the provider's page. (3) A backup with zero attachments still produces a manifest and reports success.

### 850. Provide an iOS Shortcut action: "Log expense in FinanceOS" via the webhook.

**Components:** Integrations screen card for iOS Shortcut: Text `shortcut_setup_steps` (numbered instructions), Button `btn_copy_shortcut_payload` (copies a sample JSON template), Text `shortcut_inbound_url` (reuses the item-845 inbound webhook URL), Table `tbl_draft_inbox` filtered to source='Shortcut'.

**Data:** No new tables — posts arrive through the existing `webhook_endpoints` Inbound row and land in `draft_inbox` with source='Shortcut', status='Draft'. The sample payload template is {description, amount, category?} with a Shortcut "Ask for Input" / "Dictate Text" front end the user builds in Apple's Shortcuts app.

**Logic notes:** Everything on the iOS side is built by the user in Shortcuts — the app only documents the expected JSON shape. Dictated amounts must parse to a number or the payload is rejected and logged. Shortcut posts follow the same validation and never-write-directly rules as item 845. Siri invocation is the user's own Shortcut naming.

**Acceptance:** (1) Following the setup steps, running the Shortcut creates a Draft inbox row. (2) A dictated non-numeric amount is rejected with a logged error. (3) Shortcut submissions can never create transactions directly.

### 851. Add a read-only API key for personal scripts with scoped permissions.

**Components:** Integrations screen card for Read-only API: Button `btn_generate_key` (shows the key ONCE in a modal with a copy button, then never again), Multiselect `key_scopes` (transactions:read, budgets:read, goals:read, accounts:read), Table `tbl_api_keys` (key hint last-4, scopes, created_at, last_used_at, is_revoked), Button `btn_revoke_key` per row.

**Data:** New logical store reuses `webhook_endpoints` with direction='Outbound', label='API key', plus scopes in `event_filter` — or a dedicated `api_keys` table: id (pk), key_hint (text), scopes (text), created_at (timestamp), last_used_at (timestamp, nullable), is_revoked (bool). Full key material is never persisted.

**Logic notes:** Keys are read-only by construction: the only operations they authorize are SELECT-style reads on the scoped tables. Key generation uses a cryptographically random 32-byte value; the app stores only a hash + last-4 hint. Revocation takes effect immediately. Keys cannot read `draft_inbox.raw_payload` or any integration secrets. Rate-limited to 60 requests/minute per key.

**Acceptance:** (1) A generated key can read scoped tables but any write attempt is rejected. (2) The full key is shown exactly once and never retrievable afterward. (3) Revoking a key immediately blocks its next request.

### 852. Support exporting to accounting software formats (QIF/OFX) for CPAs.

**Components:** Export screen: Select `export_format` (CSV / PDF / QIF / OFX), Date range pickers `export_from`, `export_to`, Checkbox `export_include_sample` (default off), Button `btn_generate_export`, Text `export_preview_info` (row count, date range, format notes).

**Data:** Transformer `to_qif` emits `!Type:Bank` header with `D`, `T`, `P` lines per transaction. Transformer `to_ofx` emits OFX 2.x XML with `<STMTTRN>` blocks (FITID = transaction id). Both respect the date range and the sample toggle. Query `log_integration_activity` records the export.

**Logic notes:** Amounts keep their sign convention (negative=spend) in both formats; OFX `<TRNAMT>` uses the raw number. Categories map to OFX `<MEMO>` alongside the description. Sample transactions are excluded unless the checkbox is explicitly ticked. The export filename includes the date range and the word "manual" (e.g. `financeos-manual-2026-10.qif`) so the CPA knows the provenance.

**Acceptance:** (1) QIF and OFX exports open correctly in a reference parser with all transactions in range. (2) Sample rows are excluded by default. (3) The filename marks the export as manual data.

### 853. Integrate with a receipt OCR service only with explicit per-receipt consent.

**Components:** Receipt attachment viewer: per-receipt Button `btn_ocr_this_receipt` ("Extract with OCR — sends this image to the OCR service"), confirm Modal `ocr_consent_modal` ("This sends one receipt image to <provider>. Continue?"), read-only Text fields `ocr_merchant`, `ocr_total`, `ocr_date` filled after extraction, Button `btn_apply_ocr` (copies values into the draft/transaction form).

**Data:** No new tables; OCR results are transient until the user applies them. Query `log_integration_activity` records each consented OCR call (kind='Consent'). The image is sent only after the modal's explicit Confirm; the provider endpoint is configured by the user in the integrations card.

**Logic notes:** Consent is per receipt, per tap — there is no "always allow OCR" setting. The service receives only the single image the user chose, never the whole attachment library. If extraction confidence is low, fields show "unsure" styling and the user must verify before applying. The original image is unchanged; OCR never overwrites user-entered values automatically.

**Acceptance:** (1) No image leaves the device without the per-receipt consent modal being confirmed. (2) There is no global auto-OCR toggle anywhere. (3) Every OCR call is logged in `integration_activity`.

### 854. Add calendar subscription feed (ICS) of bill due dates.

**Components:** Integrations screen card for Calendar Feed: Text `ics_url_display` (unique per-user `.ics` URL), Button `btn_copy_ics_url`, Button `btn_regenerate_ics` (confirm modal), Text `ics_instructions` ("Paste into Apple Calendar / Google Calendar → Add by URL"), preview List `ics_upcoming` (next 5 bill events).

**Data:** Query `upcoming_bills` (bills with due_date >= today, ordered, respecting sample toggle). Transformer `to_ics` emits `BEGIN:VCALENDAR` with one `VEVENT` per bill: UID = bill id + domain, DTSTART = due_date, SUMMARY = "Pay <name> — $<amount> (manual entry)". The feed regenerates on each request — no caching of personal data server-side beyond the bills themselves.

**Logic notes:** The ICS URL contains an unguessable token; regenerating invalidates the old feed. Events are read-only reminders — completing a bill in FinanceOS does not update the subscriber's calendar (by ICS design). Sample bills are excluded when the sample toggle hides them. The feed includes a `DESCRIPTION` noting "Manual tracking — no bank connection."

**Acceptance:** (1) Subscribing to the URL in a calendar app shows upcoming bill events with amounts. (2) Regenerating the URL kills the old feed. (3) Each event is labeled as manual tracking.

### 855. Show an integrations settings screen with connect/disconnect and scope notes.

**Components:** New screen "Integrations": persistent banner at top; Repeating container `rep_integrations` of connector cards (Calendar, Email Inbox, Spreadsheet, Task App, Inbound/Outbound Webhook, Cloud Storage, Read-only API, Bookmarklet, iOS Shortcut, OCR, Calendar Feed); each card has Status pill `card_status`, Text `card_scope_note` (plain-language data access), Button `btn_connect` / Button `btn_disconnect` (confirm modal on disconnect), Text `card_last_sync`.

**Data:** Query `list_integrations` reads `integrations` ordered by provider. Query `disconnect_integration` sets status='Disconnected', clears `last_sync_at`, and logs to `integration_activity`. Cards for URL/token-based integrations (webhook, ICS, API key) show "Configured" instead of "Connected" to avoid implying an account link.

**Logic notes:** Disconnect is one click + confirm, and immediately stops all related activity (webhooks disabled, feeds invalidated only on regenerate — disconnect stops future syncs). The screen lists every integration in one place so nothing is connected invisibly. Cards never show full secrets — only hints.

**Acceptance:** (1) Every integration from items 841–854 appears as a card with a plain-language scope note. (2) Disconnect stops the integration's activity immediately and is logged. (3) No full secret or token is visible on this screen.

### 856. Log all integration activity in the audit log.

**Components:** Integrations screen tab "Activity": Table `tbl_integration_activity` (columns: ran_at, provider, kind, detail, rows_affected, actor), Date filter `activity_from`/`activity_to`, Select `activity_kind_filter`, Button `btn_export_activity_csv`.

**Data:** Query `list_integration_activity` reads `integration_activity` joined to `integrations` for the provider name, newest first, paginated (100/page). Every integration feature calls `log_integration_activity` on success AND failure. The table also feeds the main audit log (section 38) via a union view so integration events appear in the global change history.

**Logic notes:** Logging is mandatory, not optional — a feature that can't write its activity row must surface an error rather than act silently. `detail` is human-readable ("Pushed October summary: 7 categories") and never contains secrets or full payloads. Rows are append-only; no edit or delete UI exists. Retention: keep 2 years, then archive to CSV.

**Acceptance:** (1) Every connect, disconnect, sync, push, webhook, export, and OCR consent appears as a row within 5 seconds. (2) Rows cannot be edited or deleted from the UI. (3) Integration events also appear in the global audit log.

### 857. Never auto-enable an integration; every one requires explicit user action.

**Components:** App-wide rule enforced in UI: all integration toggles and "Connect" buttons default to off/disconnected on first run; a shared Modal `explicit_consent_modal` used by every connector card with Checkbox `consent_understood` ("I understand what this integration can access") that must be ticked before the confirm button enables.

**Data:** Query `list_integrations` must return zero Connected rows for a fresh install — verified by a seed check. The `integrations` table has no default-connected seed rows. Each connect action writes its row only after the modal's confirm.

**Logic notes:** This is a design invariant, not a per-feature toggle: no onboarding step, sample-data loader, or update may flip an integration to Connected. Even re-installing or clearing app data returns everything to disconnected. The consent checkbox text names the specific provider and scope each time — no blanket consent.

**Acceptance:** (1) A fresh install shows every integration as disconnected/off with no way to have been auto-enabled. (2) Every connect flow passes through the explicit consent modal with the provider named. (3) No background process can create a Connected row.

### 858. Display which data each integration can read in plain language before connecting.

**Components:** Each connector card on the Integrations screen includes a fixed Text block `card_data_access` with 2–4 bullet lines in plain language (e.g. Calendar: "• Reads bill due dates you choose to sync • Cannot see your other calendar events • Cannot create or change events"), shown ABOVE the Connect button. The consent modal repeats the same list.

**Data:** Static copy per provider stored in a `INTEGRATION_COPY` JS object (no table needed); each entry has `reads[]` and `cannot[]` arrays rendered as bullets. Query-free — pure presentation.

**Logic notes:** Copy must be reviewed for accuracy whenever a provider's scope changes — the "cannot" lines are as important as the "reads" lines. The same copy appears on the card and in the consent modal so there's no bait-and-switch. Technical scope strings (e.g. `calendar.readonly`) are shown in a muted footnote for verifiability, never as the primary explanation.

**Acceptance:** (1) Every connector card shows its plain-language read/cannot list before any Connect button. (2) The consent modal repeats the identical list. (3) No card uses jargon alone to describe access.

### 859. Support manual CSV-based "integrations" as the default offline alternative.

**Components:** Integrations screen section "No accounts needed": explanatory Text `offline_alt_note` ("Every integration above has a manual alternative — CSV in, CSV out. Nothing requires an account."), per-feature Buttons linking to the manual path: `btn_manual_bills_csv` (→ CSV import), `btn_manual_export_csv` (→ export), `btn_manual_summary_copy` (copies the monthly summary table to clipboard).

**Data:** Reuses existing CSV import (item 847 presets) and CSV export queries — no new tables. Transformer `summary_to_clipboard` formats the monthly summary as tab-separated text for pasting into any spreadsheet.

**Logic notes:** This is the default posture of the whole app: integrations are conveniences layered over manual flows, never requirements. The offline alternatives must remain fully functional even if every integration is disconnected. Documentation copy explicitly states that no feature is gated behind connecting anything.

**Acceptance:** (1) Every integration card's scope note links to its manual alternative. (2) With all integrations disconnected, all core features (import, export, summaries) still work. (3) The screen states plainly that no account connection is required for any feature.

### 860. Add a "suggest an integration" feedback form.

**Components:** Integrations screen footer: Button `btn_suggest_integration` opening Modal `suggest_modal` with Text inputs `suggest_provider` ("Which app or service?"), Textarea `suggest_usecase` ("What would you use it for?"), Select `suggest_direction` (Import into FinanceOS / Export from FinanceOS / Two-way), Button `btn_submit_suggestion`.

**Data:** New table `integration_suggestions`: id (pk), provider (text), usecase (text), direction (text), submitted_at (timestamp), status (text: New / Reviewed / Planned / Declined). Query `insert_suggestion` writes the row; an admin view (future) lists them. No external ticketing — suggestions stay in the local database.

**Logic notes:** Submitting shows a thank-you state and clears the form; duplicates are allowed (vote-by-duplicate is a feature, not a bug — count duplicates when prioritizing). The form sets expectations: "Suggestions are reviewed monthly; nothing is promised." Suggestions never auto-create integrations (reinforces item 857).

**Acceptance:** (1) Submitting the form creates a row in `integration_suggestions` with status New. (2) The form states the review cadence and makes no promises. (3) Duplicate suggestions are accepted and countable.
