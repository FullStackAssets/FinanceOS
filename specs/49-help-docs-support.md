# FinanceOS Builder Spec — §49 Help, Docs & Support (Items 961–980)

## Section overview
In-app help center, guides, troubleshooting, support contact, changelog, and feedback for the FinanceOS Retool app. All help content explains manual-entry workflows and the sample-data model; support contact is a user-initiated action only — the app never auto-creates tickets or sends messages on its own.

## Tables touched
- New `help_articles`: id (pk), slug (text, unique per locale), title (text), category (text: Getting started / Manual tracking / Troubleshooting / Shortcuts / Privacy), body (markdown text), locale (text, default 'en'), version (int), is_published (bool), updated_at (timestamp), is_sample (bool)
- New `changelog_entries`: id (pk), version (text), released_at (date), summary (text), details (markdown text), is_sample (bool)
- New `support_tickets`: id (pk), subject (text), message (text), contact_email (text), status (text: Open / In review / Resolved), app_version (text), schema_version (text), created_at (timestamp), responded_at (timestamp nullable), is_sample (bool)
- New `feature_requests`: id (pk), title (text), description (text), votes (number), status (text: New / Under review / Planned / Declined), created_by (text), created_at (timestamp), is_sample (bool)
- New `feature_votes`: id (pk), request_id (fk → feature_requests.id), voter (text), voted_at (timestamp)
- Existing `accounts`, `transactions`, `budgets`, `goals` read by troubleshooting flows (per App 11 spec).

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears on every screen, including the help center.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data filter hides samples, they are excluded from tables, counts, totals, and exports.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.
- Support tickets and feedback are created only by explicit user action; the app never auto-files or auto-closes tickets.

---

### 961. Build an in-app help center with articles per screen.
- Components: Help screen (`helpCenter`) with left nav List `lstHelpNav` (search box + category groups) and right Article Viewer (Markdown component rendering `body`). Each main app screen gets a "?" header button that opens the help center pre-filtered to that screen's article.
- Data: Query `qryHelpArticles` (Retool DB: `SELECT * FROM help_articles WHERE is_published AND locale = {{appState.locale}} ORDER BY category, title`); transformer `trHelpByCategory` groups rows. Article viewer query `qryHelpArticle` fetches by slug.
- Logic notes: Screen-to-article mapping stored in a JS map (`{overview: 'overview-tour', transactions: 'adding-transactions', ...}`); unknown screens fall back to the "getting started" article. Articles are editable in-app only by the admin role (item 983).
- Acceptance: (1) "?" on any screen opens the help center with that screen's article shown. (2) Search filters articles by title/body. (3) Unpublished articles never render for viewers.

### 962. Write a "manual tracking explained" guide covering the banner and sample data.
- Components: Article `manual-tracking-explained` in the help center (item 961) plus a one-time onboarding card linking to it; illustrated callout boxes inside the article via Markdown admonitions.
- Data: Static article content seeded into `help_articles` (locale 'en', category 'Manual tracking'); transformer `trManualGuideTOC` builds an anchor list from headings.
- Logic notes: Article covers: what the banner means, why figures are hand-entered, what SAMPLE badges and `is_sample` mean, how the sample toggle changes totals, and the roadmap note that bank connections are a future opt-in (item 42x family). Screenshots reference the actual banner component and toggle.
- Acceptance: (1) Article exists in both help-center nav and the onboarding flow link. (2) Mentions banner text verbatim, SAMPLE badge, and the sample toggle. (3) Makes no claim of live bank connectivity anywhere.

### 963. Add contextual tooltips on net worth, budgets, and goals screens.
- Components: Tooltip (Retool "Tooltip" on Text/Hoverable components): net-worth hero number, each budget bar, each goal progress bar. Tooltip text pulled from `help_articles` snippets (category 'Getting started', slugs prefixed `tip-`).
- Data: Query `qryHelpTips` (`SELECT slug, title, body FROM help_articles WHERE slug LIKE 'tip-%' AND locale = {{appState.locale}}`); tooltips bind `body` by slug. No write path.
- Logic notes: Net-worth tooltip explains "sum of manual balances, sample rows excluded when toggle hides them". Budget tooltip explains actuals = current-month transactions in that category. Goal tooltip explains contributed vs target. Tooltips never block taps on mobile — tap-to-open pattern.
- Acceptance: (1) Hover/tap on net worth, each budget bar, each goal shows an explanatory tooltip. (2) Tooltip text matches the published tip article. (3) Tooltips dismiss without navigating away.

### 964. Provide a glossary: net worth, actuals, rollover, is_sample.
- Components: Help-center article `glossary` (Markdown with definition list) plus an inline glossary popover: tapping any underlined term in articles opens a Modal with the term's definition.
- Data: Terms seeded as `help_articles` rows (category 'Getting started', slug `glossary-<term>`) OR a single glossary article with a transformer `trGlossaryIndex` parsing `**term** — definition` lines into an index. Inline terms detected by wrapping in `<u data-term="...">` via a JS renderer.
- Logic notes: Required terms: net worth, actuals, rollover, is_sample, SAMPLE, category, budget limit, contribution, recurrence. `is_sample` definition states plainly: "marks seeded demo rows; excluded from totals when hidden."
- Acceptance: (1) Glossary article lists all required terms with accurate definitions. (2) Tapping an underlined term in any article opens its definition. (3) Definitions never imply bank data.

### 965. Add video-free step-by-step guides with screenshots for key flows.
- Components: Help-center articles with numbered steps; screenshots stored as image attachments (Retool Storage) and embedded in Markdown via URL; each step has a "Mark done" checkbox for progress tracking (client state `helpProgress`).
- Data: Static articles (`guide-add-transaction`, `guide-set-budget`, `guide-create-goal`, `guide-weekly-review`) with `body` containing image URLs. Progress persisted to Retool localStorage keyed by article slug.
- Logic notes: Key flows: add a transaction, set a monthly budget, create and fund a goal, run the weekly balance update. Screenshots are static captures of the actual app screens at 1440px; step text mirrors real button labels. "Mark done" state survives reloads.
- Acceptance: (1) Each of the four guides has numbered steps with at least one screenshot per guide. (2) Step labels match live UI text. (3) Marked-done steps persist across reloads.

### 966. Include a "how to update balances weekly" routine guide.
- Components: Article `guide-weekly-balances` + an in-app checklist widget on the Accounts screen ("Weekly update" card) with per-account checkbox rows and a "Mark week complete" button.
- Data: Checklist reads `accounts` (name, balance, updated_at); query `qryStaleAccounts` flags accounts not updated in >7 days. Completion stamps a row in a new `weekly_checkins` table (id, week_start date, completed_at timestamp, is_sample).
- Logic notes: Guide prescribes the routine: open each real account, type the current balance, tap update, confirm totals. Stale-account nudge shows a gentle inline warning — never a push notification without the reminders opt-in (item 29x). Sample accounts are included in the checklist but marked SAMPLE.
- Acceptance: (1) Guide describes the full weekly routine end to end. (2) Accounts untouched for 7+ days show a stale indicator. (3) Completing the checklist records a dated check-in.

### 967. Add troubleshooting articles: duplicates after import, wrong budget actuals.
- Components: Two articles (`troubleshoot-duplicate-import`, `troubleshoot-budget-actuals`) with symptom → cause → fix structure; each fix links to the relevant screen via a "Open screen" button.
- Data: Static content; the duplicates article references the CSV-import dedupe settings (item 24x), the budget article references the actuals computation (transactions of this month in category, sample rows excluded).
- Logic notes: Duplicates: caused by re-importing the same CSV or overlapping date ranges; fix = delete the newer batch from the import-review screen, enable the dedupe check. Wrong actuals: usually a mis-categorized transaction or sample toggle state; fix = re-categorize, check toggle, re-run month filter.
- Acceptance: (1) Each article names the symptom, the likely causes, and concrete fix steps. (2) "Open screen" buttons navigate to the referenced screen. (3) No step requires support contact as the only option.

### 968. Provide a contact-support form (human-reviewed, no bots auto-closing).
- Components: Help-center "Contact support" Modal: Subject (Text Input), Message (Text Area), Contact email (Text Input, prefilled from app settings), app/schema version auto-attached (item 969), Submit button. Confirmation state shows ticket ID.
- Data: Insert into `support_tickets` (`subject`, `message`, `contact_email`, `app_version`, `schema_version`, status 'Open', is_sample=false). Query `qryMyTickets` lists the user's tickets with status. No auto-responder, no email sending from the app.
- Logic notes: Ticket creation is strictly user-initiated (button tap). Status changes happen only by the human reviewer; the app never auto-closes or auto-replies. Duplicate-submit guard: disable button while the insert is in flight; identical subject+message within 60s shows "already sent".
- Acceptance: (1) Submitting creates one `support_tickets` row with status Open and returns its ID. (2) No automated reply or status change occurs without human action. (3) Double-tap on Submit does not create two tickets.

### 969. Show app version and data-schema version in help for support tickets.
- Components: Help-center footer + contact-support form (item 968) auto-display `v{{appState.appVersion}} · schema {{appState.schemaVersion}}`; a "Copy diagnostics" button copies a small JSON blob.
- Data: Versions stored in app state constants; schema version = max migration number from the Retool DB schema log (or a static constant bumped per release). Diagnostics blob includes app version, schema version, locale, sample-toggle state, row counts per table.
- Logic notes: Schema version must be bumped whenever a table gains/drops a column. Diagnostics include no personal or financial values — only counts and versions. The copy button uses the Clipboard API with a fallback textarea.
- Acceptance: (1) Version strings render in help footer and on the support form. (2) "Copy diagnostics" copies valid JSON with the documented fields. (3) Blob contains no balances, descriptions, or emails.

### 970. Add a "what's new" changelog screen per release.
- Components: Help-center "What's new" screen: List of `changelog_entries` (version header, release date, summary, expandable details Markdown); "New" badge on the Help "?" button when an unseen release exists.
- Data: Query `qryChangelog` (`SELECT * FROM changelog_entries ORDER BY released_at DESC`); client state `seenChangelogVersion` in localStorage drives the badge.
- Logic notes: Entries are written by the release process (item 98x family), not by users. Badge clears when the user opens the screen. Each entry lists user-facing changes only — no internal jargon. Past entries remain browsable.
- Acceptance: (1) Opening the screen clears the "New" badge. (2) Entries render in reverse-chronological order with version and date. (3) Non-admin users cannot add or edit entries.

### 971. Include a feedback form with feature voting.
- Components: "Give feedback" Modal (message Text Area, Submit) + "Feature voting" screen listing `feature_requests` with vote buttons and vote counts, sorted by votes desc.
- Data: Feedback inserts into `support_tickets` with subject prefix `[feedback]` (or a dedicated `feedback` table if preferred — spec uses tickets for simplicity); votes insert into `feature_votes` and increment `feature_requests.votes` in one transaction; query `qryFeatureRequests` joins counts.
- Logic notes: One vote per user per request (unique index on `(request_id, voter)`); tapping again removes the vote. Feedback submissions are user-initiated only and never emailed anywhere. Status field shows the review pipeline honestly (New → Under review → Planned / Declined).
- Acceptance: (1) Voting twice toggles (vote then unvote), never double-counts. (2) List sorts by vote count descending. (3) Feedback submit creates exactly one record with no external send.

### 972. Provide printable quick-start one-pagers for new household members.
- Components: Help-center "Quick-start one-pagers" screen with a Print button; print stylesheet renders a clean one-page layout (no nav, no banner chrome) via a dedicated Print View container.
- Data: Static article `onepager-quickstart` (Markdown) with the essentials: banner meaning, adding transactions, updating balances weekly, SAMPLE badges, where help lives. A second variant `onepager-viewer` for read-only household members.
- Logic notes: One-pagers assume zero prior knowledge; every step references visible UI labels. Print CSS hides the app chrome (nav, tabs, banner) so the page prints cleanly. Includes a QR-free URL line pointing at the app's shareable link (set by admin).
- Acceptance: (1) Print view contains the full one-pager with no app chrome. (2) Two variants exist (full + viewer). (3) All UI references match current labels.

### 973. Add a guided "fix my budget" troubleshooter flow.
- Components: Stepper wizard Modal: Step 1 "Which budget looks wrong?" (Select of budgets); Step 2 shows computed actuals breakdown (list of contributing transactions); Step 3 offers fixes (re-categorize transaction, adjust limit, toggle sample filter); Step 4 summary.
- Data: Query `qryBudgetDiagnosis` (`SELECT * FROM transactions WHERE category = {{sel}} AND date >= month_start`, honoring the sample toggle); transformer `trActualsBreakdown` sums and lists rows. Fix actions reuse existing edit-transaction and edit-budget screens.
- Logic notes: Wizard never changes data on its own — every fix is a user-confirmed action on the real screen. If actuals are correct, Step 4 says so plainly ("your budget math is right; the limit may just be tight"). Sample rows are labeled in the breakdown.
- Acceptance: (1) Wizard shows the exact transactions behind a budget's actuals. (2) Each fix navigates to the right edit screen pre-filled. (3) Wizard performs zero automatic data changes.

### 974. Document keyboard shortcuts in a dedicated article.
- Components: Article `keyboard-shortcuts` (Markdown table: action → keys) plus a "?" keyboard hint footer on desktop-width screens; shortcut help opens with `?` key.
- Data: Static article; a JS keymap object `SHORTCUTS` in app state is the single source of truth — the article table is generated from it by transformer `trShortcutTable` so docs never drift from behavior.
- Logic notes: Shortcuts: `n` new transaction, `/` focus search, `?` shortcuts article, `g then b` budgets, `g then o` overview, `Esc` close modal. Single-key shortcuts disabled while typing in inputs. All shortcuts listed are actually implemented (item 46x command-palette family defines them).
- Acceptance: (1) Article table is generated from the live `SHORTCUTS` map. (2) Pressing `?` outside an input opens the article. (3) No shortcut fires while typing in a text field.

### 975. Add a status page link for service incidents.
- Components: Help-center footer link "Service status" (Link component, opens in new tab) + an incident banner slot on the Overview screen shown only when the admin sets `appState.incidentNotice`.
- Data: Status URL stored in app settings (`status_page_url`); incident notice is a manual admin-set text + timestamp, never auto-fetched (Retool platform status is external; the app does not poll it).
- Logic notes: Covers Retool platform incidents affecting the app. The incident banner is set by a human during a real incident and cleared after; the app never fabricates incident state. Link text includes "opens externally" so users know they're leaving the app.
- Acceptance: (1) Footer link opens the configured status URL in a new tab. (2) Incident banner appears only when an admin sets it, with timestamp. (3) No automated status polling or fake incident states.

### 976. Provide sample-data cleanup walkthrough in help.
- Components: Article `guide-cleanup-samples` with a checklist + an in-app "Remove sample data" tool on the Accounts screen: preview count of `is_sample=true` rows per table, type-to-confirm ("DELETE SAMPLES"), Delete button, undo window note.
- Data: Query `qrySampleCounts` (`SELECT 'accounts' t, COUNT(*) c FROM accounts WHERE is_sample UNION ALL ...` per table). Delete runs per-table `DELETE WHERE is_sample=true` in a single user-confirmed action.
- Logic notes: Deletion is irreversible — the confirmation requires typing the phrase, and the article warns to export first (item 25x). The tool refuses to run when zero sample rows exist. Non-sample rows are never touched; the query hard-codes `is_sample=true`.
- Acceptance: (1) Preview shows exact per-table sample counts before deletion. (2) Delete requires typing the confirmation phrase. (3) After deletion, zero `is_sample=true` rows remain and user rows are untouched.

### 977. Include privacy and data-handling documentation.
- Components: Articles `privacy-overview` and `data-handling` in the help center (category 'Privacy'), linked from onboarding and Settings.
- Data: Static content seeded into `help_articles`; versioned (each edit bumps `version`) so the "last updated" date shown is accurate.
- Logic notes: Documents: what is stored (manual entries, tickets, votes — all in the app's Retool Database), what is NOT collected (no bank credentials, no automatic location, no ad tracking), who can see data (the account owner; household members only what is shared), how to request deletion (contact support, item 968). Plain language, no legalese-only walls.
- Acceptance: (1) Articles state exactly what data is stored and what is never collected. (2) "Last updated" reflects the article version date. (3) Deletion path is described with the concrete support step.

### 978. Add a "request a feature" form feeding the roadmap backlog.
- Components: Help-center "Request a feature" Modal: Title (Text Input), Description (Text Area), Submit → creates `feature_requests` row with status 'New', then appears on the voting screen (item 971).
- Data: Insert into `feature_requests` (`title`, `description`, votes 0, status 'New', created_by from app user, is_sample=false). Duplicate-title guard: query `qrySimilarRequests` (ILIKE match) suggests existing requests before submit.
- Logic notes: Requests are user-initiated only. Status transitions are human-driven (admin review). The form sets expectations honestly: "requests are reviewed, not guaranteed." Spam guard: max 5 open requests per user; identical title+description within 24h is rejected with a link to the existing request.
- Acceptance: (1) Submit creates a 'New' request visible on the voting screen. (2) Duplicate guard surfaces similar existing requests. (3) Rate limit blocks a 6th open request per user.

### 979. Show estimated support response times honestly.
- Components: Contact-support form (item 968) and Help-center header show "Typical response: within N business days" from app settings; if the median actual response exceeds the estimate, show the measured median instead.
- Data: Query `qryResponseStats` (`SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY responded_at - created_at) FROM support_tickets WHERE responded_at IS NOT NULL`); transformer `trHonestSla` picks `GREATEST(setting, measured_median)`.
- Logic notes: The displayed estimate never under-promises relative to measured reality — if actuals slip, the number shown slips too. Computed over the last 90 days; "no data yet" state shows the setting with a "new service" note. Business days counted Mon–Fri.
- Acceptance: (1) Displayed estimate is never lower than the measured 90-day median. (2) "No data yet" state is clearly labeled. (3) Estimate updates as new tickets are resolved.

### 980. Localize help articles for the user's language setting.
- Components: Help center reads `appState.locale` (from Settings, item 99x); article viewer falls back to 'en' with a "Showing English — translation in progress" notice when a locale is missing.
- Data: `help_articles.locale` column; query filters `locale IN ({{appState.locale}}, 'en')` and the viewer prefers the user locale. A `translation_status` view shows per-article coverage for admins.
- Logic notes: English is the source of truth; translations are per-article (no machine-translation on the fly). Glossary terms (item 964) and tooltips (item 963) follow the same locale rule. RTL locales set `dir="rtl"` on the article container.
- Acceptance: (1) Switching language shows translated articles where they exist. (2) Missing translations fall back to English with the notice. (3) RTL locale renders the article container right-to-left.
