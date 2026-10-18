# FinanceOS Builder Spec — §21 Navigation & Information Architecture (Items 401–420)

## Section overview
Defines the app's navigation skeleton: five primary sections (Overview, Transactions, Budgets, Accounts, Goals), a secondary More menu, global search, breadcrumbs, deep links, recently-viewed, keyboard navigation, empty states, notification badges, and per-section filter persistence. Everything is built from Retool's navigation components plus a small amount of state in `localStorage` and the URL query string. No new backend tables are required.

## Tables touched
- `accounts`, `transactions`, `budgets`, `goals` (existing, read-only for badges/totals): id, name/type/balance/updated_at, date/description/category/amount/account_id, category/month/limit_amount, goal_name/target/saved/deadline, plus is_sample flags.
- No NEW tables. Recently-viewed and tour-completion state live in `localStorage` (keys `financeos.recentlyViewed`, `financeos.tourDone`, `financeos.compactMode`) so they survive reloads without schema changes.

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." renders at the top of every screen and remains visible across all navigation states, tab switches, drawers, and deep links.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample toggle hides samples they are excluded from tables, counts, totals, badges, and exports.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.

---

### 401. Define five primary sections: Overview, Transactions, Budgets, Accounts, Goals with consistent icons.
- Components: Retool multipage app with five pages `overview`, `transactions`, `budgets`, `accounts`, `goals`. Bottom Tab Bar `tabMain` (mobile, 390px) and a top/desktop `navigation` component sharing one icon set: home, receipt/list, pie-chart, bank, flag. Icons set once via a shared `navIcons` JS object so both navigations stay consistent.
- Data: Page-to-tab mapping object `{ overview: 0, transactions: 1, ... }`; on page load set `tabMain.value` from `retoolContext.pageName`.
- Logic notes: The five pages are the ONLY top-level destinations; everything else nests under More. Adding a sixth later requires updating the mapping object, not component rewiring.
- Acceptance: (1) Both navigations show the same five labels in the same order with matching icons. (2) Tapping/clicking any tab lands on the matching page. (3) Icon set is defined once and referenced by both components.

### 402. Add a secondary "More" menu for Bills, Trends, Reports, Recurring, Settings.
- Components: Drawer `drwMore` (mobile: bottom sheet; desktop: right slide-over) opened from a "More" tab (icon: ellipsis). List View `lstMore` with rows Bills (calendar-clock icon), Trends (trending-up), Reports (file-text), Recurring (repeat), Settings (gear), each row navigable.
- Data: Static option array in a JS query `qryMoreOptions`; row click handler `utils.openPage(option.page)`.
- Logic notes: More-menu pages are standard pages (`bills`, `trends`, `reports`, `recurring`, `settings`) so deep links and back/forward work. Bills-due count badge feeds item 417.
- Acceptance: (1) "More" opens a menu listing exactly those five entries. (2) Each entry opens the correct page. (3) Drawer closes on selection and respects safe-area insets on iPhone.

### 403. Implement breadcrumbs on desktop for deep screens like goal detail and category detail.
- Components: Breadcrumb component `bcDeep` rendered only when `retoolContext.viewport.width >= 768`. Screens affected: Goal detail, Category detail, Transaction detail drawer (desktop), Report view.
- Data: `navStack` array in temporary state `stNavTrail` (e.g. `[{label:'Goals',page:'goals'},{label:'Emergency fund',page:'goalDetail'}]`); breadcrumb click pops the stack and opens the page.
- Logic notes: Trail is set explicitly when opening a deep screen (push) and trimmed on back navigation. Mobile hides breadcrumbs (bottom tab bar is the wayfinding).
- Acceptance: (1) Goal detail shows "Goals / Emergency fund" on desktop. (2) Category detail shows "Budgets / Food" on desktop. (3) Clicking a crumb navigates to that screen. (4) Hidden below 768px width.

### 404. Add deep-linkable URLs per screen (e.g., /transactions?category=Food) for sharing views.
- Components: None new — uses Retool multipage URL query parameters. Helper JS `fnDeepLink(page, params)` builds the URL via `retoolContext.pageUrl` + serialized params and copies it to clipboard with a toast.
- Data: On page load, a `qryInitFromUrl` JS query reads `retoolContext.queryParams` (e.g. `category=Food`, `account=<id>`, `month=2026-10`) and applies them to the screen's filter state (items 101–120).
- Logic notes: Supported params per page: transactions (category, account, search), budgets (month), goals (goal), overview (none). Unknown params are ignored, never error. Shared links reproduce the exact filtered view.
- Acceptance: (1) `/transactions?category=Food` opens Transactions filtered to Food. (2) Copy-link button on each screen produces a working URL. (3) Invalid params degrade to the unfiltered screen. (4) Banner and sample toggle state still apply on deep-linked loads.

### 405. Show a global search in the header that jumps to transactions, accounts, or goals.
- Components: Search input `txtGlobalSearch` in the app header (all pages), results dropdown `lstSearchResults` grouped by Transactions / Accounts / Goals with icons and one-line context (amount, balance, progress %).
- Data: Debounced JS query `qryGlobalSearch` (200ms) running three scoped searches: transactions by description (limit 8), accounts by name/institution (limit 5), goals by goal_name (limit 5); all respect the sample toggle.
- Logic notes: Enter opens the top result; clicking a result deep-links to the record (transaction drawer, accounts screen filtered, goal detail). Empty query hides the dropdown. Keyboard: "/" focuses search.
- Acceptance: (1) Typing "rent" shows matching transactions, accounts, and goals grouped. (2) Selecting a result navigates to it. (3) Results exclude hidden samples when the toggle is off. (4) Search is reachable from every primary screen.

### 406. Add a "recently viewed" strip for quick return to edited transactions or goals.
- Components: Horizontal scroll container `cntRecent` on the Overview below the hero, cards showing icon, label, and context line (e.g. "Groceries · Food · -$42.10"); hidden when empty.
- Data: `localStorage.financeos.recentlyViewed` — array of `{kind, id, label, detail, page, params}` capped at 10, most-recent first. Updated by a shared `fnTrackRecent()` called on transaction edit, goal edit, and goal detail open.
- Logic notes: Duplicates collapse to the newest entry. Entries referencing deleted records are pruned on read. Sample-flagged entries render with the SAMPLE badge.
- Acceptance: (1) Editing a transaction adds it to the strip. (2) Tapping a card returns to that record. (3) Strip caps at 10 with newest first. (4) Clearing app data or deleting the record removes its card.

### 407. Keep the manual-entry banner visible across all navigation states.
- Components: The banner is a shared module `modBanner` (text: "Manual tracking — no bank connection yet.") imported into the header of every page, drawer, and deep-linked view.
- Data: None — static text with one setting: `settings.bannerPlacement` (top bar vs slim footer strip, item 343).
- Logic notes: Because it is a module, there is exactly one banner definition; pages cannot accidentally drop it. Verification procedure (item 360) scans every page for the module instance.
- Acceptance: (1) Banner renders on all five primary pages and all More pages. (2) It survives tab switches, drawer opens, and deep-link loads. (3) Moving it to the footer strip (setting) keeps it on every screen. (4) No page can render financial figures without the module present.

### 408. Add keyboard navigation support for tab order through all main screens.
- Components: Native tab order across all inputs, buttons, tabs, and table rows; visible focus ring via a global CSS rule (`:focus-visible { outline: 2px solid ...; outline-offset: 2px }`).
- Data: None.
- Logic notes: Tab order follows visual top-to-bottom, left-to-right. Skip-link "Skip to content" is the first tab stop on desktop. Modals trap focus until dismissed (Esc closes). Arrow keys move within tab bars and table rows; Enter activates.
- Acceptance: (1) Every interactive element is reachable by Tab alone. (2) Focus is always visible. (3) Esc closes drawers/modals and returns focus to the opener. (4) Screen-reader labels exist on all icon-only buttons (also item 923).

### 409. Display section-level empty states with guided next actions ("Add your first account").
- Components: Empty-state panel `pnlEmpty` per section (illustration + headline + one primary CTA): Overview ("Add your first account" → accounts), Transactions ("Add your first transaction"), Budgets ("Set your first budget"), Goals ("Create your first goal").
- Data: Visibility condition per section (e.g. `{{ qryAccounts.data.length === 0 }}` counting non-sample rows; sample-only data still shows the real empty state).
- Logic notes: CTA buttons deep-link to the relevant creation form. Empty states never show when only sample data exists — samples are labeled demo content, not the user's data.
- Acceptance: (1) Fresh account with no real accounts sees the guided empty state on each section. (2) Each CTA opens the correct form. (3) Empty state disappears once one real record exists. (4) Sample-only data does not suppress the empty state.

### 410. Add a sitemap-style settings index so every option is findable.
- Components: Settings home page with a searchable index list `lstSettingsIndex`: rows for Profile, Currency & formats, Notifications, Privacy & sharing, Data management, Appearance, Accessibility, Integrations, About — each with icon, one-line description, and chevron.
- Data: Static index array; the settings search box filters rows by label/description keywords.
- Logic notes: Every settings control must be reachable within two taps from the index. New settings (bank opt-in, integrations) register their row here — the index is the completeness checklist for settings discoverability.
- Acceptance: (1) Every settings screen is listed. (2) Search filters the index live. (3) Tapping a row opens the right screen. (4) No settings screen exists that is not in the index.

### 411. Support browser back/forward correctly across filters and drawers.
- Components: None new — relies on Retool multipage history plus `history.pushState` for in-page filter changes via a JS helper `fnPushFilterState()`.
- Data: Filter state serialized into the URL query string (item 404); drawer open/close pushes a history entry so Back closes the drawer instead of leaving the page.
- Logic notes: Back from a filtered transactions view returns to the previous filter, not the previous page. Drawer Back handling: intercept popstate, close topmost drawer first. Deep-link loads seed history correctly.
- Acceptance: (1) Changing a filter then pressing Back restores the prior filter. (2) Back with an open drawer closes the drawer, staying on the page. (3) Forward re-applies the undone state. (4) No "blank page" history entries.

### 412. Add a collapsible sidebar on desktop with section totals.
- Components: Sidebar `sbrDesktop` visible ≥ 1024px: nav list (five sections + More) each with live total — net worth (Overview), month spend (Transactions), budgets hit/missed count (Budgets), account count + total (Accounts), goals on-track count (Goals). Collapse toggle to icon rail.
- Data: One `qrySidebarTotals` JS query aggregating from existing queries; recomputes on data change; respects sample toggle.
- Logic notes: Totals are read-only summaries — clicking a row navigates to the section. Collapsed state persists in `localStorage.financeos.sidebarCollapsed`.
- Acceptance: (1) Sidebar shows the five sections with correct live totals. (2) Collapse/expand works and persists. (3) Hidden below 1024px. (4) Totals update instantly when a transaction is added.

### 413. Highlight the active tab and section in the navigation at all times.
- Components: `tabMain` selected-index styling (accent underline + filled icon), sidebar `sbrDesktop` active-row highlight, breadcrumb current-page emphasis.
- Data: Active page derived from `retoolContext.pageName`; a `stActiveSection` temp state mirrors it for components that need it.
- Logic notes: The highlight must stay correct across deep-link loads, More-menu pages (More tab highlighted), and back/forward navigation. Exactly one primary section is active at a time.
- Acceptance: (1) Current tab is visually distinct on mobile and desktop. (2) More-menu pages highlight the More entry. (3) Highlight survives deep links and browser back/forward. (4) Never zero or two active tabs.

### 414. Provide a "start of month" quick-jump on date pickers.
- Components: Date picker `dtpAny` wrapper: small "1st" button beside every month-relevant date input (add-transaction date, budget month, report ranges).
- Data: Sets the bound date value to the first day of its current month via `moment(value).startOf('month')`.
- Logic notes: Applies to the budgets month selector, transaction date (when picking within month-end flows), and report date ranges. Does not alter the year or time components.
- Acceptance: (1) Tapping "1st" sets the field to the first of the shown month. (2) Works on all month-relevant pickers. (3) Manual typing still allowed after the jump. (4) Keyboard accessible.

### 415. Add contextual help icons per screen linking to the relevant help article.
- Components: Icon button `btnHelp` (question-circle) in each screen header: Overview, Transactions, Budgets, Accounts, Goals, Bills, Trends, Reports, Recurring, Settings.
- Data: Mapping `helpArticles = { overview: 'manual-tracking-explained', transactions: 'adding-transactions', ... }`; click opens the in-app help center (items 961+) to the article, or the article in a drawer on mobile.
- Logic notes: Every screen must have a mapped article before release — the mapping is the QA checklist. Articles open in-app, not an external browser tab.
- Acceptance: (1) Each of the ten screens shows a help icon. (2) Each opens the correct article. (3) Works on mobile (drawer) and desktop (side panel). (4) No dead links — mapping coverage is 100%.

### 416. Implement a first-visit coachmark tour of the five primary sections.
- Components: Coachmark overlay `ovlTour`: 5 steps, each highlighting one tab/section with a short caption (Overview: "Your money at a glance — all manual entries"; Transactions; Budgets; Accounts; Goals). Next/Skip controls, progress dots.
- Data: `localStorage.financeos.tourDone` flag; tour auto-starts on first app open after onboarding, skippable anytime, replayable from Settings → About.
- Logic notes: Tour pauses if a modal is open and resumes after. Each step's caption restates manual tracking so the banner message is reinforced. Never auto-starts twice.
- Acceptance: (1) First visit shows the 5-step tour. (2) Skip exits cleanly and sets the done flag. (3) All five sections are covered in tab order. (4) Replay is available in Settings.

### 417. Show notification badges on tabs (e.g., bills due count on the More menu).
- Components: Badge `bdgMore` on the More tab (count of unpaid bills due within 7 days), badge on Budgets tab (categories over 100%), badge on Goals tab (goals with deadlines ≤ 30 days and under 80%).
- Data: `qryNavBadges` JS query computing the three counts from bills, budgets/actuals, and goals; respects sample toggle; refreshes on data change and on app focus.
- Logic notes: Badges show counts only, capped at "9+". Zero = no badge. Tapping the badged tab clears nothing automatically — badges reflect live state, not read/unread.
- Acceptance: (1) Unpaid bill due in 3 days increments the More badge. (2) Over-budget category increments the Budgets badge. (3) Counts update without reload. (4) No badge when count is zero.

### 418. Add a compact mode that reduces card padding for dense desktop views.
- Components: Global CSS class `compact-mode` toggled from Settings → Appearance; reduces card padding (16px→8px), row height, and section gaps on desktop layouts only.
- Data: `localStorage.financeos.compactMode` boolean; applied as a class on the app root at load.
- Logic notes: Compact mode never hides information — only spacing changes. Touch targets stay ≥ 44px on mobile regardless (compact mode is desktop-only; on mobile the toggle is disabled with an explanation).
- Acceptance: (1) Toggling visibly densifies desktop cards. (2) No content is clipped or hidden. (3) Setting persists across sessions. (4) Mobile layouts are unaffected.

### 419. Keep filter state per section so switching tabs doesn't lose work.
- Components: None new — behavior of existing filter controls (transactions filters, budgets month, reports ranges).
- Data: Per-section state objects in temp state (`stTxFilters`, `stBudgetMonth`, `stReportRange`) that survive page switches within the session; optionally persisted to `localStorage` for cross-session restore.
- Logic notes: Switching Overview → Transactions → Overview must not reset the transaction filters. The "clear all filters" button (item 114) resets only its own section. Deep links override stored state for that load.
- Acceptance: (1) Set a Food filter, switch tabs, return — filter intact. (2) Budgets month persists across tab switches. (3) Clear-all affects only its section. (4) State survives a page reload (localStorage).

### 420. Add a "back to overview" home affordance in every header.
- Components: Home icon button `btnHome` at the left of every screen header (all five primary + all More pages), labeled "Overview" for screen readers.
- Data: Click handler `utils.openPage('overview')`.
- Logic notes: Always visible, always in the same position — it is the universal escape hatch. On the Overview itself the button is hidden (it would be a no-op) to avoid confusion.
- Acceptance: (1) Every header except Overview's shows the home button. (2) Tapping it always lands on Overview. (3) Position is consistent across screens. (4) Screen-reader label reads "Back to overview".
