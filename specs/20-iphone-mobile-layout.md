# FinanceOS Builder Spec — §20 iPhone Mobile Layout (Items 381–400)

## Section overview
Mobile-first layout pass for the FinanceOS Retool app: every screen is usable one-handed on a 390px-wide iPhone in portrait and landscape. This spec defines the mobile app shell (bottom tab bar, safe-area handling, pull-to-refresh, floating action button), responsive transformations (tables → cards, single-column forms, tappable full-width budget rows), compact presentation rules (number format toggle, chart label handling, banner compaction), and offline-tolerant rendering. Item 399 (physical iPhone text-size test) is intentionally skipped — it requires a physical device and cannot be done in this pass.

## Tables touched
- No new tables in this section. Uses existing `accounts`, `transactions`, `budgets`, `goals` (per App 11 spec). Item 398's offline cache is client-side only (Retool localStorage), not a DB table.

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears on every screen, compacted but always visible on small screens (item 388).
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data filter hides samples, they are excluded from tables, cards, counts, totals, and exports.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.

---

### 381. Make every screen render cleanly at 390px width with no horizontal scrolling.
- Components: All screens use a root Container `appRoot` set to `max-width: 100%`, `overflow-x: hidden`. Table components are replaced or wrapped per item 385; set chart components to `width: 100%` with responsive resize enabled.
- Data: None — pure layout. Add a QA query `qryLayoutAudit` (JS) that scans rendered components for `scrollWidth > clientWidth` and reports offenders during builder review.
- Logic notes: 390px is the iPhone 14/15/16 logical portrait width; design for 360px minimum to cover smaller devices with margin. Long institution names and descriptions truncate with ellipsis + tooltip, never wrap into horizontal overflow.
- Acceptance: (1) At 390px viewport, no screen shows a horizontal scrollbar. (2) Layout audit query reports zero overflow offenders. (3) Text truncation never clips the SAMPLE badge or banner.

### 382. Add a bottom tab bar with Overview, Transactions, Budgets, Accounts, Goals.
- Components: Navigation component `navBottomTabs` pinned to the viewport bottom, five tabs with icons (home, list, chart-bar, bank, flag) and labels Overview / Transactions / Budgets / Accounts / Goals. Active tab shows filled icon + tinted label. Respects safe-area inset (item 387) via `padding-bottom: env(safe-area-inset-bottom)`.
- Data: Each tab sets `appState.currentTab`; main content container switches views with `hidden` bindings. Tab state persists per session so back-navigation returns to the last tab.
- Logic notes: Secondary screens (Bills, Trends, Reports, Recurring, Settings per item 402) live under a "More" overflow on the tab bar or a header menu — never as a sixth tab. Tab badge for bills due count hooks to item 417.
- Acceptance: (1) All five tabs reachable with the thumb; tap switches the main view. (2) Active tab is visually distinct. (3) Tab bar never overlaps content (content has matching bottom padding).

### 383. Ensure all forms are single-column with labels above inputs on iPhone.
- Components: Every form (add-transaction, create-account, set-budget, create-goal, create-bill) uses a single-column Container; each field is a Label component above its input (Text Input, Select, Date Picker, Number Input). No side-by-side field pairs below 480px.
- Data: None — layout only. Form validation messages render beneath their field in red, with `aria-describedby` linkage for screen readers.
- Logic notes: Labels above inputs (not floating/placeholder-only) keeps fields identifiable when the iOS keyboard covers the lower half. Required-field asterisks on the label, not the input.
- Acceptance: (1) No form shows two inputs side by side at 390px. (2) Every input has a visible persistent label. (3) Validation errors appear under the failing field.

### 384. Size touch targets at minimum 44px for buttons and list rows.
- Components: Global style rule — Buttons, Icon Buttons, list-row containers, tab items, and toggle chips get `min-height: 44px`. Dense rows (transaction rows, account rows) use 44px minimum row height; padding absorbs the difference on text-only rows.
- Data: None — styling only.
- Logic notes: 44×44pt is the iOS Human Interface Guideline minimum. Small inline actions (e.g., per-row delete ×) get an invisible 44px hit area even if the visible glyph is smaller. Spacing between adjacent targets ≥ 8px to prevent mis-taps.
- Acceptance: (1) Measured hit area of every button/row ≥ 44px in each dimension at 390px. (2) No two tappable targets are closer than 8px edge-to-edge. (3) Inline icon actions still trigger reliably with thumb taps.

### 385. Make tables collapse into card lists on narrow screens.
- Components: For the Transactions, Accounts, Bills, and Recurring tables: a Table component (desktop) paired with a List View `lstTransactionsCards` (mobile) rendering one card per row — description + signed amount prominent, category chip, account chip, date, SAMPLE pill, notes paperclip indicator. A viewport transformer `trIsNarrow` (`window.innerWidth < 640`) toggles which is visible.
- Data: Both components bind to the same `qryTransactionsFiltered` data source so sort/filter state is shared; selection state syncs via a shared `selectedTransactionId` temp state.
- Logic notes: Cards preserve every column's information — nothing is dropped in the collapse, only re-laid-out. Bulk-select checkboxes become long-press or an explicit "Select" mode toggle on mobile.
- Acceptance: (1) Below 640px, tables render as cards with no horizontal scroll. (2) Card shows all row data incl. SAMPLE badge and attachment indicator. (3) Filter/sort/selection state carries over between table and card views.

### 386. Add pull-to-refresh on the overview and transactions screens.
- Components: On the Overview and Transactions views, wrap content in a container with pull-to-refresh enabled (Retool mobile "pull to refresh" interaction on the view). Trigger refires `qryOverviewAggregates` and `qryTransactionsFiltered` plus balance/totals queries.
- Data: Refresh handler runs the screen's queries with `additionalScope` unchanged; a `isRefreshing` temp state shows the spinner and blocks duplicate pulls. On completion, toast "Updated just now" with the refresh timestamp.
- Logic notes: Pull-to-refresh re-reads the Retool DB — it does not "sync" with any bank (none exists). If offline (item 398), the gesture shows cached data with the "last synced" note instead of failing.
- Acceptance: (1) Pull gesture on Overview and Transactions triggers a visible refresh. (2) Totals and lists update after refresh. (3) Offline pull shows cached data + last-synced note, no error crash.

### 387. Support iPhone safe-area insets for notched devices.
- Components: Root layout and bottom tab bar use `padding-top: env(safe-area-inset-top)` on headers and `padding-bottom: env(safe-area-inset-bottom)` on the tab bar and floating action button. Fullscreen chart view (item 537) and bottom sheets (item 391) also respect insets.
- Data: None — CSS environment variables only.
- Logic notes: Test against notched (iPhone 14/15/16) and Dynamic Island layouts; the tab bar must never sit under the home indicator. Landscape mode additionally respects left/right insets.
- Acceptance: (1) No interactive element is obscured by the notch, Dynamic Island, or home indicator in portrait or landscape. (2) Banner remains fully visible below the notch. (3) Bottom sheets clear the home indicator.

### 388. Make the manual-entry banner compact but always visible on small screens.
- Components: Banner component `bannerManual` renders full text "Manual tracking — no bank connection yet." on desktop; below 640px it collapses to a slim 28px strip with a "MANUAL" chip + info icon; tapping expands the full text in a popover. It is pinned (sticky) at the top of every screen and every navigation state.
- Data: Dismissal is per-session only (item 342): `appState.bannerDismissed` resets on app load; even dismissed, the slim strip remains.
- Logic notes: "Compact but always visible" — the banner may shrink, never disappear. The slim variant keeps ≥ 4.5:1 contrast and a 44px tap target on the info icon.
- Acceptance: (1) Banner strip is visible at the top of every screen at 390px without scrolling. (2) Full text is one tap away. (3) Banner survives tab switches, drawer opens, and filter changes.

### 389. Add a floating action button for quick-add transaction on mobile.
- Components: Floating Action Button `fabQuickAdd` (plus icon, 56px circle) fixed bottom-right above the tab bar, visible only below 640px. Tap opens the add-transaction form as a bottom sheet (item 391) prefilled with today's date (item 7/62).
- Data: Opens `formAddTransaction` in a modal; on save, the new row appears in the transactions list and overview totals recompute (item 180). Quick-add writes are tagged `source: "quick-add"` in the audit log (item 918).
- Logic notes: FAB hides while the keyboard is open or a bottom sheet is active to avoid overlap. It must clear the safe-area home indicator (item 387).
- Acceptance: (1) FAB visible on all five tabs at 390px, above the tab bar. (2) Tap opens the transaction form bottom sheet with today's date. (3) Saved transaction appears in list and totals immediately.

### 390. Ensure charts render legibly on iPhone with rotated or abbreviated labels.
- Components: All Chart components get a mobile config: x-axis labels rotate −45° or abbreviate (e.g., "Jan", "$1.2k") when `trIsNarrow` is true; legend moves below the chart; minimum chart height 220px; bar charts switch to horizontal layout when category names are long.
- Data: A transformer `trChartMobileConfig` returns `{ rotateLabels, abbreviate, legendPosition }` from viewport width; each chart's options bind to it.
- Logic notes: Abbreviation must never lose the unit — "$1.2k" not "1.2". Donut charts show the total in the center so small slices stay interpretable. Tooltips remain available on tap.
- Acceptance: (1) No chart label overlaps or clips at 390px. (2) All amounts still show units after abbreviation. (3) Tapping a data point still shows the exact value tooltip.

### 391. Test all drawers and modals as bottom sheets on iPhone.
- Components: Every Drawer/Modal (edit-transaction, account edit, budget edit, goal detail, category detail, bill detail) is configured to present as a bottom sheet on mobile: slides from bottom, 90% max height, drag handle, swipe-down to dismiss (with unsaved-changes guard), backdrop tap dismisses.
- Data: None — presentation only. Unsaved-changes guard: dismiss with dirty form shows "Discard changes?" confirm.
- Logic notes: Bottom sheets keep the underlying screen's scroll position. Focus moves into the sheet on open and returns to the invoking control on close (accessibility). Sheets respect safe-area bottom inset (item 387).
- Acceptance: (1) Every drawer/modal opens as a bottom sheet below 640px. (2) Swipe-down dismiss works; dirty forms prompt before discarding. (3) Sheet content scrolls internally without moving the page behind it.

### 392. Add haptic-style visual feedback on button presses (no native dependency).
- Components: Global button style: `:active` state scales to 0.97 and darkens the background for 120ms; primary actions (Save, Confirm, Mark paid) additionally show a brief checkmark morph on success. Implemented in the app's custom CSS — no native haptics API required.
- Data: None — CSS only.
- Logic notes: Visual feedback substitutes for `navigator.vibrate` (unavailable/unreliable in the Retool mobile webview). Keep the animation under 150ms so it never feels laggy; honor reduced-motion mode (item 928) by disabling the scale animation.
- Acceptance: (1) Every button shows a visible press state on tap. (2) Success actions show the confirmation morph. (3) With reduced-motion enabled, no scale animation plays.

### 393. Make the kanban-free FinanceOS screens scroll vertically with sticky section headers.
- Components: Each screen's content is a single vertical scroll container; section headers (e.g., "This month", "Bills due soon", per-goal headers) use `position: sticky; top: <banner+header offset>` so they pin while their section scrolls. No horizontal carousels or kanban boards anywhere in FinanceOS.
- Data: None — layout only.
- Logic notes: Sticky offsets must account for the compact banner (item 388) + screen header height. Sticky headers get a solid background so scrolled content doesn't show through.
- Acceptance: (1) All screens scroll vertically only at 390px. (2) Section headers stick while their section is in view. (3) No content is hidden behind stuck headers.

### 394. Support landscape orientation without broken layouts.
- Components: At landscape (≈844×390), the bottom tab bar remains; two-column grids (account cards, goal cards) switch to a horizontal scroll row or compact two-up grid; charts expand to available width; the FAB repositions above the tab bar clear of the side insets.
- Data: `trOrientation` (portrait/landscape from viewport aspect) drives layout switches alongside `trIsNarrow`.
- Logic notes: Landscape on iPhone is short vertically — prioritize scrollable content over fixed headers; the banner stays slim (item 388). Left/right safe-area insets apply (item 387).
- Acceptance: (1) Rotating to landscape shows no overlapping or clipped controls. (2) All five tabs remain reachable. (3) Rotating back restores the portrait layout exactly.

### 395. Add a compact number format ($1.2k) toggle for small screens.
- Components: Toggle `tglCompactNumbers` in Settings → Display (default ON below 640px, OFF on desktop). A global formatter `fmtMoney(n)` checks the toggle: compact → `$1.2k` / `$3.4M`; full → `$1,234.56`.
- Data: Pure presentation — underlying values and CSV exports always use full precision. The toggle is stored in `appState.userPrefs`.
- Logic notes: Compact format keeps one decimal for k/M, two for values under $1k only if space allows; negative signs and red/green spend/income coloring are preserved. Tooltips and detail views always show the full amount.
- Acceptance: (1) Toggle switches all screen amounts between `$1.2k` and `$1,234`. (2) Exports and audit log unaffected (full precision). (3) Tapping a compact amount reveals the exact figure.

### 396. Ensure the add-transaction form fits above the iPhone keyboard.
- Components: The add-transaction bottom sheet uses a scrollable form body with the Save bar sticky at the bottom of the sheet; when the keyboard opens, the sheet resizes (visual viewport) and auto-scrolls the focused field into view above the keyboard.
- Data: None — layout/keyboard interaction only.
- Logic notes: Amount keypad (item 75) uses `inputmode="decimal"` to get the numeric keyboard, which is shorter than the full keyboard. The spend/income toggle and Save button must remain reachable without dismissing the keyboard — test on the smallest viewport (iPhone SE 375×667).
- Acceptance: (1) With the keyboard open, the focused field and Save button are both visible/reachable. (2) Sheet scrolls, never clips fields behind the keyboard. (3) Numeric keyboard appears for the amount field.

### 397. Make budget bars full-width tappable rows on mobile.
- Components: On the Budgets screen below 640px, each category budget renders as a full-width row card: category icon + name, progress bar (green <80%, amber 80–100%, red >100% per item 163), "$42 left" / overspend text, chevron. The entire row is a 56px-minimum tap target opening the category detail (drill-down per item 166); a long-press or edit icon opens inline budget editing (item 177).
- Data: Rows bind to the same budget-vs-actual query as desktop bars; tap navigates to the filtered transactions view for that category.
- Logic notes: Full-width rows replace the desktop bar list one-for-one — same data, thumb-friendly targets. The edit affordance must not trigger accidentally on scroll: use explicit edit icon, not swipe, for inline editing.
- Acceptance: (1) Each budget row spans the full width at 390px with ≥56px height. (2) Tap opens the category's transactions; edit icon opens inline editing. (3) Bar colors and remaining/overspend text match desktop values.

### 398. Add offline-tolerant rendering: cached data shows with a "last synced" note.
- Components: A connectivity banner `bannerOffline` ("Offline — showing data from <time>") appears below the manual-entry banner when `navigator.onLine === false` or a query fails with a network error. All screens render from the last successful query results held in temp state.
- Data: After every successful query run, a JS query `qryCacheSnapshot` copies key datasets (accounts, transactions, budgets, goals) into `localStorage` with a timestamp. On load or query failure, `qryLoadCache` restores them and sets `appState.lastSyncedAt`. Queued writes (item 944/945) are out of scope for this item — display-only tolerance here.
- Logic notes: "Last synced" = last successful Retool DB read, labeled as such — never imply bank sync (no bank exists). Cached figures keep their SAMPLE badges and the manual-entry banner stays visible.
- Acceptance: (1) With network disabled, all five tabs render cached data with the "last synced" timestamp. (2) Banner clearly states offline state. (3) No screen shows a blank error in place of cached content.

### 400. Provide a "desktop tips" note when advanced features are easier on larger screens.
- Components: An info callout `noteDesktopTips` appears on screens with desktop-advantaged features (bulk-select in transactions per item 89, chart gallery per item 540, CSV column picker per item 491, rules simulation per item 878): "Tip: bulk editing is faster on a larger screen." Dismissible per feature, remembered in `appState.userPrefs`.
- Data: None — static guidance. Dismissal state persists per session (resets on reload, like the banner).
- Logic notes: The note is guidance, not a block — every feature remains fully usable on iPhone, just with more steps. Never phrase it as "not available on mobile."
- Acceptance: (1) The tip appears on bulk-select, chart gallery, CSV column picker, and rules simulation screens at 390px. (2) Dismissing hides it for the session. (3) Wording never claims a feature is mobile-unavailable.
