# Section 50 — Settings & app administration (items 981–1000)

## Section title
Settings & app administration — a single settings area where the user controls preferences (profile, formatting, theme, notifications), data lifecycle (export, backup, restore, sample-data purge), access scaffolding (app lock, household roles), and transparency (manual-tracking status, diagnostics, about).

## Tables touched
- `accounts`, `transactions`, `budgets`, `goals` (existing — read for storage counts, export payloads, and sample-purge targets; all rows carry `is_sample`)
- **NEW** `app_settings`: key (text, pk), value_json (text), updated_at (timestamp). One row per preference key (e.g. `display_name`, `home_currency`, `date_format`, `number_format`, `first_day_of_week`, `budget_cycle_default`, `theme`, `notification_prefs`, `sample_toggle`, `app_lock_pin_set`, `app_lock_timeout`, `font_scale`, `reduce_motion`, `simplify_interface`, `language`, `bank_opt_in`).
- **NEW** `household_members`: id (text, pk), name (text), role (text: Viewer / Editor), created_at (timestamp). Scaffolding for future multi-user; Retool permission grants themselves stay admin-hands work.

## Cross-cutting rules (apply to every subsection)
- The persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen, including settings.
- Every row with `is_sample=true` renders a SAMPLE badge; the sample-data toggle (`app_settings.sample_toggle`) excludes sample rows from totals, charts, and exports when off.
- Nothing in the app ever presents as a live bank feed. Any bank-related copy is future-tense and clearly labeled.
- Settings are single-user preferences. Nothing irreversible executes without typed confirmation (item 1000).

---

### 981. Build a settings home with sections: profile, data, notifications, privacy, about.
**Components:** New `SettingsHome` screen with a header text "Settings" and five Navigation cards (Container components) labeled Profile, Data, Notifications, Privacy, About; each card has an icon and chevron, click opens the matching screen. A top-level "Back" button returns to Overview.
**Data:** `getSettings` query — `SELECT key, value_json FROM app_settings`; transformer `settingsMap` reduces rows to `{key: JSON.parse(value_json)}` for binding throughout.
**Logic notes:** Cards use conditional text showing the current value summary (e.g. home currency under Profile). All settings screens share the header banner and back-navigation pattern.
**Acceptance:** (1) All five section cards render with correct labels and navigate to the right screens. (2) Current-value summaries on cards match stored settings. (3) Banner "Manual tracking — no bank connection yet." is visible on SettingsHome.

### 982. Allow editing display name and home currency.
**Components:** `ProfileScreen` with Text Input `inputDisplayName` and Select `selectHomeCurrency` (options: USD, EUR, GBP, CAD, AUD, JPY, plus an "Other" free-text fallback); Save button.
**Data:** `getSetting` queries by key (`display_name`, `home_currency`); `upsertSetting` — `INSERT INTO app_settings (key, value_json, updated_at) VALUES ({{key}}, {{JSON.stringify(value)}}, NOW()) ON CONFLICT (key) DO UPDATE SET value_json=EXCLUDED.value_json, updated_at=NOW()`.
**Logic notes:** Display name defaults to "My FinanceOS" when unset. Home currency drives all currency formatting app-wide via a shared `fmtMoney` transformer; changing it refreshes all amounts without touching stored numbers.
**Acceptance:** (1) Edits persist across reloads. (2) Changing home currency updates amount formatting on Overview. (3) Empty display name falls back to the default label.

### 983. Add date-format and number-format preferences.
**Components:** `ProfileScreen` additions: Select `selectDateFormat` (MM/DD/YYYY, DD/MM/YYYY, YYYY-MM-DD), Select `selectNumberFormat` (`1,234.56`, `1.234,56`, `1 234,56`).
**Data:** Same `upsertSetting` pattern with keys `date_format`, `number_format`; global JS helpers `fmtDate(d)` and `fmtNum(n)` read `settingsMap` and apply the chosen format with `toLocaleString`-style logic in plain JS.
**Logic notes:** Stored dates stay ISO; only display formatting changes. Number format applies to decimal/thousands separators, never to stored numeric values or CSV exports (exports stay machine-readable).
**Acceptance:** (1) A sample date renders in each chosen format. (2) Amounts switch separators per preference. (3) CSV export columns remain unformatted raw values.

### 984. Provide first-day-of-week and budget-cycle defaults.
**Components:** Select `selectFirstDayOfWeek` (Sunday, Monday) and Select `selectBudgetCycle` (Calendar month, 4-week periods, Custom start day with Number Input `inputBudgetStartDay` shown when custom).
**Data:** Keys `first_day_of_week`, `budget_cycle_default` via `upsertSetting`.
**Logic notes:** First-day-of-week feeds the cash-flow calendar header ordering and weekly aggregation queries. Budget-cycle default pre-fills new-budget `month` labels; custom start day rolls the "current period" window for budget-vs-actual computations.
**Acceptance:** (1) Calendar week headers reorder per preference. (2) New budgets default to the configured cycle label. (3) Custom start day shifts period boundaries in budget actuals.

### 985. Add theme settings: light, dark, system, high-contrast.
**Components:** Radio Group `radioTheme` (Light, Dark, System, High-contrast) with a live preview panel showing sample cards in the chosen theme.
**Data:** Key `theme` via `upsertSetting`; Retool app theme API or CSS variable overrides set on change.
**Logic notes:** "System" follows `prefers-color-scheme`. High-contrast forces a dark background with maximum-contrast text and 3px focus outlines, overriding component-level color choices. Theme applies instantly without reload.
**Acceptance:** (1) Each option visibly changes the app chrome. (2) High-contrast passes a 7:1 text contrast check on primary text. (3) Choice persists across sessions.

### 986. Manage notification preferences per alert type in one screen.
**Components:** `NotificationsScreen` with a Table or repeated Switch rows for alert types (budget exceeded, bill due soon, goal milestone, low balance), each row: name, description, Switch `switchAlert_<type>`; a master "All notifications" toggle at top.
**Data:** Key `notification_prefs` storing `{budget_exceeded: bool, bill_due: bool, goal_milestone: bool, low_balance: bool, master: bool}`.
**Logic notes:** Master toggle off disables all per-type switches (greyed). Alert-sending logic elsewhere ANDs the stored preference before queuing any notification, so disabling here guarantees silence.
**Acceptance:** (1) Toggling a type persists and is read by the alert engine. (2) Master off disables and greys all child switches. (3) Screen lists every alert type the app can raise.

### 987. Add a data-management screen: export all, import, purge samples, delete data.
**Components:** `DataScreen` with four sections: Buttons `btnExportAll`, `btnImportCsv` (File Input), `btnPurgeSamples`, `btnDeleteData` (danger styling). Each section has a one-line description of exactly what it does.
**Data:** `exportAll` — zips per-table CSVs built from the four base tables; import reuses the CSV-import flow (section 24); `purgeSamples` — `DELETE FROM transactions WHERE is_sample=true` (and accounts/budgets/goals sample rows), guarded by typed confirmation; `deleteData` — deletes only the user's non-sample rows, also typed-confirmation gated.
**Logic notes:** Neither destructive action touches the `app_settings` table or app schema. Purge samples is reversible via "reset demo" (989); delete data is not, hence the typed confirmation (1000).
**Acceptance:** (1) Export downloads a file containing all four tables. (2) Purge samples removes only `is_sample=true` rows. (3) Both destructive buttons require typed confirmation before executing.

### 988. Show storage usage with per-table row counts.
**Components:** `DataScreen` section "Storage" with a Stat components row: total rows across tables, plus a small Table listing each table name, row count, and sample vs real split.
**Data:** Queries `countAccounts`, `countTransactions`, `countBudgets`, `countGoals` — each `SELECT COUNT(*) AS total, SUM(CASE WHEN is_sample THEN 1 ELSE 0 END) AS samples FROM <table>`; transformer sums to a grand total.
**Logic notes:** Counts reflect the sample toggle? No — storage counts always show true totals regardless of the display toggle, and the UI labels this explicitly ("storage, not display").
**Acceptance:** (1) Per-table counts match actual row counts. (2) Sample/real split adds up to the total per table. (3) Counts update after purge/import without reload.

### 989. Add a "reset demo" action restoring labeled sample data for evaluation.
**Components:** Button `btnResetDemo` on DataScreen with description "Restores the original labeled sample dataset. Your real entries are kept."
**Data:** `resetDemo` JS query: (1) `DELETE FROM <table> WHERE is_sample=true` for all four tables, (2) re-insert the canonical sample dataset (the same 3 accounts / 15 transactions / 4 budgets / 2 goals from the app spec) with `is_sample=true`.
**Logic notes:** Non-sample rows are never touched. If sample rows already exist they are replaced, so running reset twice yields an identical dataset. Requires typed confirmation because it overwrites current sample edits.
**Acceptance:** (1) After reset, exactly the canonical sample dataset exists with SAMPLE badges. (2) Real (non-sample) rows are untouched. (3) Running reset twice is idempotent.

### 990. Provide app-lock settings: PIN, biometric, timeout.
**Components:** `PrivacyScreen` with: "Set PIN" flow (two Password inputs + Save, stored as a salted hash in `app_settings.app_lock_pin_hash` — never plaintext), Switch `switchBiometric` ("Use device biometrics when available"), Select `selectLockTimeout` (Immediately, 1 min, 5 min, 15 min, Never).
**Data:** Keys `app_lock_pin_hash`, `app_lock_biometric`, `app_lock_timeout`. Lock enforcement lives in the app's global auth wrapper, not in this spec's settings screen.
**Logic notes:** Biometric toggle only appears if the device reports biometric capability; the setting stores intent, and the lock screen attempts the device prompt. "Never" timeout is allowed but shows a warning about device-level security.
**Acceptance:** (1) PIN is stored hashed, never displayed or exported. (2) Biometric toggle is hidden on incapable devices. (3) Timeout selection persists and is honored by the lock wrapper.

### 991. Manage household members and roles from settings.
**Components:** `PrivacyScreen` (or dedicated `HouseholdScreen`): Table `tblHousehold` (name, role, added date) backed by `household_members`; "Add member" form (name input, role Select: Viewer/Editor); per-row Remove button.
**Data:** Standard CRUD queries on `household_members`. Roles are labels only for now: Viewer = sees dashboards, Editor = can add/edit records.
**Logic notes:** This table is scaffolding — the app is currently single-user, so roles are recorded but not enforced; enforcement requires Retool admin permission grants, which is Nic's-hands work. The UI states this plainly ("role labels recorded; app access control is not yet enforced").
**Acceptance:** (1) Members can be added, role-changed, and removed. (2) Screen states clearly that enforcement is future work. (3) No credential or invite is ever sent from this screen.

### 992. Show connected integrations with scopes and disconnect buttons.
**Components:** `DataScreen` (or `IntegrationsScreen`) section "Connected integrations": list rendering from `app_settings.integrations` JSON array; each row shows name, scopes granted, connected date, and a Disconnect button. Empty state: "No integrations connected."
**Data:** Keys `integrations` (array of `{name, scopes[], connected_at}`); `disconnectIntegration` removes the entry and calls the provider's revoke endpoint when one exists.
**Logic notes:** There are currently no integrations — the spec defines the contract (name, scopes, revoke) so future connections (section 42/43 work) plug in. Disconnect never deletes user financial data, only the integration record and token.
**Acceptance:** (1) Empty state renders correctly today. (2) A mock entry displays name, scopes, and date. (3) Disconnect removes the entry and revokes the token.

### 993. Add a future bank-connection opt-in entry point, default off, clearly labeled.
**Components:** Card on SettingsHome labeled "Bank connections (coming soon)" with body copy: "Not available yet. FinanceOS is manual tracking only." plus a disabled Switch `switchBankOptIn` set to OFF.
**Data:** Key `bank_opt_in` defaults to `false`; the switch is disabled until a connection flow exists, so the value cannot be turned on from this UI.
**Logic notes:** This is a design placeholder only — no actual connection is possible, no credentials are collected, and nothing in the app implies a live feed. When a real connection flow is built (Nic's-hands work), this card becomes the opt-in gate and stays default-off.
**Acceptance:** (1) Card is visibly labeled as future/unavailable. (2) Switch renders OFF and disabled. (3) No part of the screen collects bank credentials or implies connectivity.

### 994. Provide accessibility settings: font scale, reduce motion, simplify interface.
**Components:** `AboutScreen` (or dedicated Accessibility section): Slider `sliderFontScale` (80%–150%), Switch `switchReduceMotion`, Switch `switchSimplifyInterface` ("hide charts, show plain lists").
**Data:** Keys `font_scale`, `reduce_motion`, `simplify_interface`; applied via root font-size scaling, CSS `prefers-reduced-motion` override, and conditional rendering of chart components.
**Logic notes:** Font scale applies to text only, never breaking layouts (containers scroll). Simplify-interface replaces chart components with accessible data tables — the same underlying queries, different presentation. Changes apply instantly.
**Acceptance:** (1) Font scale 150% keeps all screens usable with scrolling. (2) Reduce motion disables chart animations. (3) Simplify interface swaps every chart for a data table.

### 995. Add language selection.
**Components:** Select `selectLanguage` (English, Spanish, French, Portuguese, German, Chinese Simplified) on the Profile screen, with a note "More languages coming; some labels may remain in English."
**Data:** Key `language`; a `strings` transformer maps UI label keys to the selected locale, falling back to English for untranslated keys.
**Logic notes:** Language affects labels and helper copy only — stored data (descriptions, notes) is never translated or altered. Date/number formats remain controlled by item 983 independently of language.
**Acceptance:** (1) Selecting a language switches chrome labels. (2) Untranslated keys fall back to English visibly. (3) User-entered data text is never modified.

### 996. Show the persistent manual-tracking status ("Bank connection: not connected") in settings.
**Components:** A status card pinned near the top of SettingsHome: green/grey indicator dot, headline "Bank connection: not connected", subcopy "Manual tracking — every figure is entered by you or labeled SAMPLE."
**Data:** Static text plus the `bank_opt_in` value to confirm it is off; no query needed beyond `settingsMap`.
**Logic notes:** This card exists so the manual-tracking stance is discoverable inside settings too, not just the banner. If a bank connection ever ships, this card becomes the single source of truth for connection state.
**Acceptance:** (1) Card renders on SettingsHome above all sections. (2) Copy matches the banner's stance exactly. (3) Card updates automatically if connection state ever changes.

### 997. Provide a diagnostics screen: load times, sync status, schema version.
**Components:** `DiagnosticsScreen` (linked from About): stats for last screen-load time, query timings table (query name, ms), "Sync status: local — no cloud sync" line, schema version string, and a "Copy diagnostics" button producing a text report.
**Data:** Timing captured via `performance.now()` deltas in query success handlers stored in temporary state; schema version from a `schema_version` key in `app_settings` (set at build time, e.g. `"2026-10-18"`).
**Logic notes:** Since there is no backend sync, sync status honestly reports "No cloud sync configured." The copied report includes app version, schema version, timings, and settings keys (never PIN hashes or personal data).
**Acceptance:** (1) Timings shown are real measured values. (2) Sync status states no cloud sync. (3) Copy produces a complete text report with no secrets.

### 998. Add an about screen with version, credits, and the manual-tracking philosophy.
**Components:** `AboutScreen`: app name + version, "Built with Retool" credit, and a "Our philosophy" section with the manual-tracking statement: figures are entered by the user or labeled SAMPLE; no bank feed is implied or promised; the user owns their data.
**Data:** Version string from a constant (`APP_VERSION = "1.0.0"` in a JS query) plus `schema_version` from settings; philosophy copy is static text.
**Logic notes:** The philosophy text is the user-facing counterpart to the cross-cutting rules — plain language, no jargon. Version shown here must match the version stamped on exports.
**Acceptance:** (1) Version and schema version render. (2) Philosophy text explicitly disclaims any live bank feed. (3) Export stamp version matches this screen.

### 999. Support backing up settings to a JSON file for restore.
**Components:** Buttons `btnBackupSettings` (downloads JSON) and `btnRestoreSettings` (File Input accepting the JSON) on DataScreen.
**Data:** `backupSettings` — `SELECT key, value_json FROM app_settings` → downloads `{exported_at, app_version, settings: {...}}`. `restoreSettings` — parses the file, validates each key against an allowlist of known setting keys, then upserts rows.
**Logic notes:** Backup excludes `app_lock_pin_hash` and any future secrets — the file is preferences only, and the UI says so. Restore validates key names and JSON types before writing; unknown keys are skipped and reported, never written.
**Acceptance:** (1) Backup file restores an identical preference set on a fresh database. (2) PIN hash is never included in the file. (3) Malformed files are rejected with a clear error and no partial writes.

### 1000. Require typed confirmation for destructive settings actions (purge, delete, reset).
**Components:** A shared confirmation Modal `modalDestructiveConfirm`: shows the action name, what will happen, what will NOT be affected, a Text Input where the user must type the exact word shown (e.g. `PURGE`), and Confirm/Cancel buttons. Confirm is disabled until the typed text matches exactly.
**Data:** The modal is a reusable pattern: each destructive query (`purgeSamples`, `deleteData`, `resetDemo`) is triggered only from the modal's Confirm handler after validation passes.
**Logic notes:** Typed word differs per action (PURGE / DELETE / RESET) and the match is case-sensitive. The modal lists affected tables and row counts fetched live before confirmation, so the user sees the blast radius.
**Acceptance:** (1) Confirm stays disabled until the exact word is typed. (2) Wrong-case or partial input does not enable Confirm. (3) Every destructive action in settings routes through this modal — no direct-delete buttons exist.
