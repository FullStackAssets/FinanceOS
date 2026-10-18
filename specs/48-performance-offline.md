# Section 48 — Performance & offline (items 941–960)

**Scope:** Speed budgets, large-history handling, and offline-first behavior for FinanceOS. App remains manual-entry: performance work never implies live feeds.

## Tables touched

- `accounts`, `transactions`, `budgets`, `goals` (base tables; `is_sample` flags as defined in App 11).
- NEW `offline_queue`: id (pk, text — client-generated UUID), entity_type (text: transaction / budget / goal / balance_update), entity_key (text — the target row id or client id), op (text: create / update / delete), payload (text — JSON), base_version (timestamp — last known server state before the edit), status (text: pending / syncing / synced / conflict / failed), attempt_count (number), created_offline_at (timestamp), error (text).
- NEW `monthly_rollups`: id (pk), source_table (text), period (text: YYYY-MM), category (text), total (number), row_count (number), is_sample (bool), computed_at (timestamp).
- NEW `perf_metrics`: id (pk), screen (text), load_ms (number), rows_loaded (number), measured_at (timestamp), note (text).
- NEW `archived_periods`: id (pk), year (text), source_table (text), row_count (number), export_ref (text), archived_at (timestamp).

## Shared cross-cutting rules

- Every screen shows the persistent banner "Manual tracking — no bank connection yet."
- Sample rows carry `is_sample=true` + a SAMPLE badge and are excluded from totals when the sample toggle hides them.
- Nothing ever presents as a live bank feed — offline queueing is for the user's own manual entries only.
- Offline semantics: when connectivity drops, writes land in `offline_queue` (and localStorage mirror) with status `pending`; the app shows an offline banner and queues a sync pass on the `online` event. Conflict policy: last-write-wins for balance updates; field-merge for transaction edits using `base_version` diffing; the user picks a winner for deletes-vs-edits. All queued entries keep the manual-tracking labels.

---

### 941. Load the overview screen in under 2 seconds with 10,000 transactions.

**Components:** Overview screen (existing hero), Timer component `perfTimer` (hidden), Statistic components for net worth / month spend / goal progress, `perfBanner` (hidden, shows only in diagnostics mode).
**Data:** Query `overviewFast`: `SELECT total FROM monthly_rollups WHERE period = {{ currentMonth }} AND category='ALL'` plus one `SUM(balance)` over accounts — never a full transactions scan on load. `perfTimer` records `performance.now()` delta into `perf_metrics` on screen load.
**Logic notes:** Rollups (see 951) are the load path; the raw 10k-row table is paginated elsewhere (942). Budget: net-worth SUM + rollup reads must total < 2s; if a load exceeds budget, log it and degrade chart rendering first.
**Acceptance:** (1) With 10,000 synthetic transactions (is_sample=true), overview renders in ≤ 2s on a throttled 3G profile. (2) No query on the load path does `SELECT *` on transactions. (3) Every load writes one `perf_metrics` row.

### 942. Paginate or virtualize the transactions table for large histories.

**Components:** Transactions Table (replace full-load Table with server-side pagination: page size 100), Pagination controls, `rowCountText` showing "Showing X–Y of Z", `loadMoreButton` (alternative infinite scroll on mobile).
**Data:** Query `transactionsPage`: `SELECT ... FROM transactions WHERE ... ORDER BY date DESC LIMIT 100 OFFSET {{ page*100 }}`, plus `transactionsCount` for totals. Both re-run on filter/search change with page reset to 0.
**Logic notes:** Never render more than ~150 rows in the DOM at once; on iPhone prefer infinite-scroll (limit 50/page) to keep scroll smooth. Sorting state resets OFFSET. SAMPLE badge column stays visible in every row.
**Acceptance:** (1) 10k-row history opens the screen with no frame drops > 100ms. (2) Changing page never re-downloads the entire history. (3) Filters/search reset to page 1 and update the count text.

### 943. Cache reference data (accounts, budgets) for instant tab switches.

**Components:** App-level Temporary State `refCache` (accounts, budgets, goals, updatedAt), used by every tab header/dropdown instead of fresh queries on switch.
**Data:** On app load and after any successful write: `loadRefData` populates `refCache`; subsequent tab switches read from state with zero queries. A 60-second stale-while-revalidate refresh runs in the background (does not block render).
**Logic notes:** After offline sync (945) or any mutation, invalidate only the affected slice. Cache is in-memory only — never persisted across sessions, so SAMPLE toggles always reflect the current toggle state on reload.
**Acceptance:** (1) Switching between Overview / Transactions / Budgets tabs issues no new queries. (2) A budget edit is visible in Budgets within 2s via targeted invalidation. (3) Cold load populates the cache before the first render completes.

### 944. Add an offline mode: view and add transactions without connectivity, syncing later.

**Components:** Offline Banner (amber strip under the persistent manual-tracking banner): "You're offline — changes are saved locally and will sync."; read-only cached views of accounts/budgets; Add-Transaction form stays enabled.
**Data:** On load, snapshot accounts + budgets + last 500 transactions into `localStorage.financeos_cache`. Form submissions while `navigator.onLine === false` insert into `offline_queue` (status `pending`) instead of the DB, and optimistically render with a "queued" badge.
**Logic notes:** Reads always work offline from cache; only writes queue. `is_sample` is captured at queue time so syncing preserves SAMPLE labels. Banner visibility derives from a `window.addEventListener('offline'/'online')` handler writing to `appState.offline`.
**Acceptance:** (1) With Wi-Fi off, last-synced data views and the add form works, producing `pending` queue rows. (2) Reconnecting triggers a sync pass with no user action. (3) The offline banner appears/disappears within 1s of connectivity change.

### 945. Queue offline writes with conflict resolution on reconnect.

**Components:** Sync Status indicator (queued count pill in header), `conflictModal` listing each conflict with "Mine / Theirs / Merge" options, `syncLog` table (from `offline_queue` rows with status synced/failed).
**Data:** On `online`: process `offline_queue` rows FIFO, `attempt_count+1` each. Compare `base_version` against the current row `updated_at`: match → apply op; mismatch on a delete-vs-edit → mark `conflict` and open `conflictModal`. Field-level merge for transaction edits (per-field, non-overlapping fields auto-merge).
**Logic notes:** Balance updates are last-write-wins (user confirmed in 944 spec); category/amount edits merge per field. Failed rows (e.g. row deleted server-side) go to `failed` with an error note, never silently dropped. Exponential backoff: 3 attempts max, then manual retry button.
**Acceptance:** (1) Two devices editing the same transaction produce exactly one conflict row and a modal. (2) Non-overlapping field edits auto-merge with no user prompt. (3) No queued write is lost: pending rows survive app reload (localStorage mirror).

### 946. Show a connectivity indicator with "offline — changes saved locally" state.

**Components:** Header pill `connPill`: green dot "Online" / amber dot "Offline — N changes saved locally" (N = pending queue count), Tooltip with last-sync timestamp.
**Data:** `connPill` reads `appState.offline` (boolean) and `offline_queue` count where status='pending'. Updates via the same online/offline event handlers as 944; the count refreshes after every queue insert/sync.
**Logic notes:** The pill is informational only — it never blocks actions. Text wording is fixed ("offline — changes saved locally", lowercase per design) to match the offline banner. During syncing, the pill shows a spinner state "Syncing…".
**Acceptance:** (1) Pill flips to the offline state within 1s of losing connectivity. (2) The pending count matches `SELECT COUNT(*) FROM offline_queue WHERE status='pending'`. (3) After a clean sync, the pill returns to "Online" with the last-sync tooltip updated.

### 947. Compress chart datasets for fast rendering on iPhone.

**Components:** Chart components on Overview and Trends (Plotly/Chart), transformer `downsampleSeries` applied before every chart render.
**Data:** `downsampleSeries` buckets daily points to weekly when the series exceeds 400 points (largest-remainder bucketing preserves totals); monthly buckets beyond 1,200 points. Tooltip queries the underlying data on tap, not the compressed series.
**Logic notes:** Compression only affects rendering — exported CSVs (section 25) always use full resolution. SAMPLE-excluded totals recompute before downsampling. Charts must paint within 500ms on an iPhone 12-class device.
**Acceptance:** (1) A 5-year daily series renders as ≤ 400 points. (2) Tapping a chart point still shows the exact underlying value. (3) Totals displayed above charts match the uncompressed aggregation.

### 948. Debounce search input to keep filtering responsive.

**Components:** Search Text Input on Transactions, debounce wrapper (300ms) before the filter query fires, `searchSpinner` (tiny, inline) while the query runs.
**Data:** Query `transactionsPage` (942) is triggered by the debounced value, not raw keystrokes. The 300ms timer resets on each keystroke; Enter fires immediately.
**Logic notes:** Typing "groceries" must issue at most 2 queries (debounced mid-type + Enter), not 9. Debounce value is a Temporary State so the visible input stays instant while queries lag behind.
**Acceptance:** (1) Fast typing produces ≤ 1 query per 300ms window. (2) The input never loses characters or focus during filtering. (3) Clearing the search resets results within 300ms.

### 949. Add a data-size dashboard: row counts per table and storage used.

**Components:** Settings → "Data size" panel: per-table row counts (accounts, transactions, budgets, goals, offline_queue, monthly_rollups), a "Storage used" estimate bar, and a "Refresh" button.
**Data:** Query `tableSizes`: `SELECT 'transactions' AS t, COUNT(*) FROM transactions UNION ALL ...` — one lightweight query. Storage estimate = sum of `pg_total_relation_size` equivalents if available, else row_count × avg-row-bytes heuristic labeled "estimate".
**Logic notes:** Read-only panel; counts include sample rows with a "（incl. N samples）" note so the SAMPLE toggle's effect is visible. Refresh re-runs counts, never full scans of payload columns.
**Acceptance:** (1) Panel loads in < 1s regardless of transaction count. (2) Counts match `SELECT COUNT(*)` exactly. (3) The storage figure is labeled as an estimate when computed heuristically.

### 950. Support archiving old years to keep the working set fast.

**Components:** Settings → "Archive" section: year picker, "Export first" checkbox (links to 958), "Archive year" button with confirm modal, `archived_periods` table view of past archives.
**Data:** Archive flow: (1) export year to CSV (958), (2) insert row into `archived_periods`, (3) `DELETE FROM transactions WHERE date < 'YYYY-01-01' AND date >= ...` for that year — only after export_ref is stored. Current year is never archivable.
**Logic notes:** Archived years disappear from all screens and totals; a banner on Trends notes "2023 and earlier archived". Undo is via CSV re-import, documented in the confirm modal. Sample rows archive with everything else (is_sample preserved in the export).
**Acceptance:** (1) Archiving requires a completed export first (button disabled otherwise). (2) Post-archive, totals and charts exclude the archived year. (3) `archived_periods` records year, row count, and export reference.

### 951. Optimize budget-actual computation with pre-aggregated monthly rollups.

**Components:** Budgets screen bars (read from `monthly_rollups`, not live aggregation), background "Recompute rollups" job trigger.
**Data:** `monthly_rollups` rows: one per (period, category, is_sample) with `total` and `row_count`. Recompute query runs after transaction add/edit/delete/import/sync, and nightly: `INSERT ... ON CONFLICT (period, category, is_sample) DO UPDATE`.
**Logic notes:** Budget-vs-actual bars render from rollups in a single indexed read — the 10k-row scan disappears from the render path. Rollup freshness: recompute is debounced 5s after the last write; a "data as of HH:MM" caption shows staleness honestly.
**Acceptance:** (1) Budgets screen issues zero scans over the transactions table on render. (2) A new transaction updates the bar within 10s. (3) Rollup totals exactly equal live `SUM()` on the same filters.

### 952. Lazy-load receipt thumbnails in the gallery.

**Components:** Receipts gallery (Grid view), per-cell placeholder skeleton, IntersectionObserver-style lazy loader (Retool: render thumbnails only for visible rows via the table's virtualized viewport).
**Data:** Thumbnail URLs load on scroll-into-view; full-resolution image loads only on tap. A `thumbLoaded` per-row flag prevents re-fetching on re-scroll.
**Logic notes:** Gallery with 200 receipts must open without fetching 200 images — only the visible ~12. Failed thumbnails show a file-icon fallback, never a broken-image box. Works identically on 3G (957).
**Acceptance:** (1) Opening the gallery issues ≤ 15 image requests initially. (2) Scrolling loads more thumbnails with no layout shift. (3) Tapping a thumbnail always opens the full image.

### 953. Add a "lite mode" disabling charts for low-end devices.

**Components:** Settings toggle "Lite mode", app-level `liteMode` state; when on, all Chart components are replaced by their summary Statistic/table equivalents (conditional visibility, not just hidden — charts unmount).
**Data:** No new queries: lite views reuse the same rollup Statistic values the charts are built from (951), so toggling is instant.
**Logic notes:** Lite mode persists in localStorage. It also disables receipt-thumbnail preloading (952) and chart animations app-wide. The toggle itself explains: "Hides charts to keep the app fast on older phones."
**Acceptance:** (1) Enabling lite mode removes all chart components from the DOM. (2) All figures remain visible as numbers/tables. (3) Preference survives reload and applies before first paint.

### 954. Measure and display screen load times in a diagnostics panel.

**Components:** Settings → "Diagnostics" panel: table of recent `perf_metrics` rows (screen, load_ms, rows_loaded, measured_at), "Clear" button, per-screen average line.
**Data:** Every screen's on-load handler records `performance.now()` delta into `perf_metrics` (capped: keep last 200 rows, delete older). Panel reads the table sorted by measured_at desc.
**Logic notes:** Instrumentation is always on but the panel is the only surface — no user-facing timing badges elsewhere. Metrics exclude time spent in offline queue replay (945) to keep budgets honest; a `note` column records "offline-sync" runs separately.
**Acceptance:** (1) Each screen visit appends exactly one row. (2) Table retention never exceeds 200 rows. (3) Averages shown are computed over the visible window only, labeled as such.

### 955. Handle 5 years of daily transactions without UI jank.

**Components:** Trends screen with year selector, virtualized transaction list (942), downsampled charts (947), rollup-backed summaries (951) — this item is the acceptance harness tying them together.
**Data:** Test dataset generator (sample only): 1,825 daily transactions is_sample=true. All screens must source aggregates from `monthly_rollups` and lists from paginated queries; no component may hold > 500 rows in memory.
**Logic notes:** Jank budget: scroll and tab switches at 60fps with no long tasks > 50ms on a mid-range iPhone. If a screen violates the budget, the fallback is automatic: charts downsample harder, lists paginate smaller.
**Acceptance:** (1) With 5 years of daily data, every screen is interactive within 2s. (2) No screen holds more than 500 transaction rows client-side. (3) Scrolling the transaction list shows no visible stutter.

### 956. Add background refresh of computed summaries after bulk imports.

**Components:** Import completion toast with "Recalculating…" state, `rollupStatus` indicator on Overview ("Summaries current as of HH:MM").
**Data:** After CSV import (section 24) or offline-sync replay (945) completes, a single `recomputeRollups` query runs asynchronously; the UI stays interactive (no blocking modal). Progress is a lightweight status text, not a progress bar over row counts.
**Logic notes:** Imports insert rows with is_sample flags intact; rollups rebuild per (period, category, is_sample). If the user navigates mid-recompute, the job continues and the status indicator updates wherever visible.
**Acceptance:** (1) Importing 5,000 rows never freezes the UI. (2) Summaries show a "recalculating" state until rollups finish. (3) Final totals match a direct SUM() over imported rows.

### 957. Ensure the app remains usable on 3G connections.

**Components:** App-wide loading skeletons for every data region, per-query 15s timeout with a "Slow connection — retry" inline action, `liteMode` auto-suggest banner when load times exceed budget twice.
**Data:** All queries set to "run on page load" only where essential; everything else is event-driven. Payloads stay small via pagination (942), rollups (951), and lazy thumbnails (952).
**Logic notes:** Usability bar on throttled 3G: Overview interactive ≤ 5s, transaction page turn ≤ 3s, form submit ≤ 5s (or queued offline, 944). Timeouts never lose user input — a timed-out form submit is retried, not discarded.
**Acceptance:** (1) On a 3G throttle profile, all five screens reach interactive state without errors. (2) Every query failure surfaces a retry control, not a blank screen. (3) Two slow loads trigger the lite-mode suggestion.

### 958. Provide an export-before-archive flow so nothing is lost.

**Components:** Archive confirm modal (950) with mandatory "Export year to CSV" step: year picker → "Download CSV" → checkbox "I have saved the export" (auto-checked when the download completes) → "Archive" button enables.
**Data:** Export query pulls the full year's transactions (all columns incl. is_sample) to CSV via the section-25 export utilities; the download event writes the export_ref stored in `archived_periods`.
**Logic notes:** The archive button stays disabled until the export completes — this is a hard gate, not a warning. The modal states plainly: "Archiving deletes these rows from the app. The CSV is your only copy."
**Acceptance:** (1) Archive cannot proceed without a completed export in the same session. (2) Exported CSV contains every row later deleted (count verified). (3) The export includes a header row noting manual tracking and the export date.

### 960. Document performance budgets per screen in the build notes.

**Components:** Build-notes document section "Performance budgets" (maintained alongside this spec): table of screen → budget (Overview ≤2s, Transactions page-turn ≤1s, Budgets ≤1.5s, Trends ≤2s, form submit ≤3s online).
**Data:** Budgets are validated against `perf_metrics` (954): a monthly review query flags screens whose p95 exceeds budget over the last 30 days.
**Logic notes:** Budgets cover the 10k-transaction working set (941) and the 5-year dataset (955); they are re-baselined only after deliberate architecture changes, not to hide regressions. The doc records the measurement method (throttled 3G, mid-range iPhone) so numbers are reproducible.
**Acceptance:** (1) Every screen has a written budget with its measurement setup. (2) A review query exists that flags budget breaches. (3) The doc is updated whenever a budget changes, with the reason noted.
