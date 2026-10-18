# FinanceOS — Section 47: Accessibility (items 921–940)

Builder spec for FinanceOS (Retool manual-entry personal finance dashboard; NO bank connection). Each item below is a builder-ready spec: exact Retool components, data wiring, logic, and acceptance criteria.

## Tables touched

- Existing (App 11): `accounts`, `transactions`, `budgets`, `goals` (all with `is_sample` bool).
- **NEW — `a11y_settings`**: id (pk), setting_key (text: high_contrast / reduced_motion / dyslexia_font / simplify_ui / text_scale / chart_summaries), setting_value (text), updated_at (timestamp). One row per key; single-user app so no user_id. Drives every preference in this section.
- **NEW — `a11y_feedback`**: id (pk), created_at (timestamp), topic (text: contrast / keyboard / screen_reader / motion / other), message (text), app_screen (text), status (text: New / Reviewed / Fixed).

## Shared cross-cutting rules

1. Every screen shows the persistent banner "Manual tracking — no bank connection yet." at the top of the viewport, above all content, on every item implementation.
2. Sample rows carry `is_sample=true` plus a SAMPLE badge and are excluded from totals when the sample toggle hides them.
3. Nothing ever presents as a live bank feed; no component label or text may imply automatic syncing.

---

### 921. Meet WCAG 2.1 AA contrast ratios on all text and chart elements.

**Components:** All screens; chart components (`plotlyJsonChart`, `chart`); custom CSS block in App Settings > Custom CSS. Add a `txtContrastAudit` JS query (retired after audit) that scans theme tokens.
**Data:** Define semantic color tokens in a `themeTokens` transformer: `textPrimary #1a1a1a`, `textSecondary #4a4a4a`, `chartPalette` (8 colors, all ≥ 4.5:1 against card background `#ffffff`). Feed chart colors from `themeTokens.data.chartPalette`, not Retool defaults.
**Logic notes:** Body text ≥ 4.5:1; large text (≥18pt/14pt bold) ≥ 3:1; non-text UI (focus rings, chart axes, bar edges) ≥ 3:1. Placeholder text in search inputs must also pass — use `#595959` minimum on white.
**Acceptance:** (1) A color-contrast scan of every screen's computed styles shows zero failures against WCAG 2.1 AA. (2) All chart series colors come from the audited palette token. (3) No hardcoded gray-on-gray text remains in any component.

### 922. Provide full keyboard operability for every screen and dialog.

**Components:** Navigation (use `navigation` component or button group with roving tab order), all modals (`modalFrame` with focus trap), transaction table (enable row actions via keyboard), add-transaction form fields in logical tab order.
**Data:** `keymapShortcuts` transformer documents shortcuts (e.g. `n` = new transaction, `?` = shortcut help). A `modalFocusReturn` JS query stores `document.activeElement` on modal open and restores focus on close.
**Logic notes:** Every mouse action has a keyboard equivalent; dropdowns open with Enter/Space and navigate with arrows; Escape closes any dialog and returns focus to the trigger. Tab order follows visual order left-to-right, top-to-bottom. No keyboard traps.
**Acceptance:** (1) The full add-transaction flow completes using keyboard only, no mouse. (2) Focus never lands on a hidden/disabled element. (3) Escape from every dialog returns focus to its opener.

### 923. Add screen-reader labels to all icon-only buttons.

**Components:** Every icon-only `button` (search, close, filter, download, edit-pencil, trash). Set each button's Tooltip = the action name AND its ARIA/accessibility label field to the same text (e.g. "Delete transaction").
**Data:** `iconButtonAudit` JS query (run once, retired after) that lists all buttons where `label` is empty; builders fix each hit.
**Logic notes:** Labels must be action-specific, never generic "button" — "Close dialog", "Export CSV", "Add budget". Dynamic buttons include the target: "Delete transaction 'Whole Foods'". Labels update if the action changes (e.g. "Show sample data" vs "Hide sample data").
**Acceptance:** (1) Zero icon-only buttons without an accessibility label across all screens. (2) Labels read as imperative actions, not nouns. (3) Toggle buttons announce their current state in the label.

### 924. Supply text summaries for every chart ("Food spending rose 12%").

**Components:** Below each `plotlyJsonChart`/`chart`, a `text` component bound to `chartSummary_<name>.data.summary`. Toggle via `a11y_settings` key `chart_summaries` (default on).
**Data:** `chartSummary_spending` JS query: takes the chart's dataset, computes top mover (largest MoM % change by category), total, and direction; returns a 1–2 sentence string like "Food spending rose 12% to $412; Transport fell 5%."
**Logic notes:** Summary regenerates whenever filters, month, or the sample toggle change. Numbers in summaries respect the sample toggle (excluded when hidden). Never fabricate trends — if data is flat, say "No significant change."
**Acceptance:** (1) Every chart has a matching summary text directly beneath it. (2) Summary updates within the same render as the chart after any filter change. (3) Summaries are hidden when the chart_summaries setting is off.

### 925. Support dynamic type scaling up to 200% without layout breakage.

**Components:** App-wide custom CSS using `rem`-based sizing for text containers; `text_scale` value in `a11y_settings` (100/125/150/200) applied as a class on the app root; table columns set to flexible width, no fixed pixel heights on cards.
**Data:** `applyTextScale` JS query reads `a11y_settings` row `text_scale`, sets `localStorage.financeos_text_scale`, and toggles root class `scale-200` etc.
**Logic notes:** At 200%, no text is clipped or overlapped: cards grow vertically, tables scroll horizontally rather than squashing columns, modals become scrollable. Buttons keep minimum 44×44px touch targets. Test at 320px viewport width.
**Acceptance:** (1) At 200% scale on a 320px viewport, all screens remain fully readable with no overlapping text. (2) No horizontal page scroll on card layouts. (3) The setting persists across sessions.

### 926. Add a high-contrast theme option.

**Components:** Settings screen toggle `tglHighContrast` (switch) bound to `a11y_settings` key `high_contrast`; app-level custom CSS block `.hc-theme` overriding tokens.
**Data:** `applyHighContrast` JS query: on toggle, writes `a11y_settings` and reloads theme by adding/removing `.hc-theme` on root. Colors: pure black text on white, links/buttons `#0033cc`, chart palette swapped to 8 high-contrast distinguishable hues.
**Logic notes:** High-contrast mode must also satisfy item 921 ratios (black/white trivially passes). SAMPLE badges get a black border in HC mode. Banner remains visible and legible. The toggle itself must be keyboard-operable.
**Acceptance:** (1) Toggle switches the entire app to the HC palette with no unstyled remnants. (2) Contrast scan in HC mode shows zero failures. (3) Preference persists across sessions via `a11y_settings`.

### 927. Ensure focus indicators are visible on all interactive elements.

**Components:** Global custom CSS: `:focus-visible { outline: 3px solid #005fcc; outline-offset: 2px; }` applied to buttons, links, inputs, table rows, tabs, chart legends.
**Data:** None beyond CSS; `focusAudit` manual checklist (run once per release, see item 939).
**Logic notes:** Never use `outline: none` without a replacement. Focus ring must be visible against both default and high-contrast themes (test both). In dark-on-light and HC palettes the `#005fcc` ring passes 3:1 on white; HC theme uses a 3px black+white double ring.
**Acceptance:** (1) Tabbing through any screen shows an unmistakable focus ring on every stop. (2) No element relies on color change alone for focus. (3) Ring visible in both default and high-contrast themes.

### 928. Provide reduced-motion mode disabling animations and transitions.

**Components:** Settings toggle bound to `a11y_settings` key `reduced_motion`; global CSS `@media (prefers-reduced-motion: reduce)` plus a `.rm` root class with `* { animation: none !important; transition: none !important; }`.
**Data:** `applyReducedMotion` JS query sets the class and persists the setting. Also auto-enable when the OS reports `prefers-reduced-motion: reduce` on first load.
**Logic notes:** Chart animations (bar growth, line draws) render instantly in final state. Modal open/close is instant, no slide/fade. The manual-entry banner never animates regardless. Loading spinners are replaced with static "Loading…" text.
**Acceptance:** (1) With reduced motion on, no element animates or transitions on any screen. (2) OS-level reduced-motion preference is honored automatically on first visit. (3) Charts still render correct final values instantly.

### 929. Add descriptive alt text to receipt thumbnails.

**Components:** Receipt gallery `imageGrid`/`listView` thumbnails; each image's alt/tooltip bound to `receiptAlt.data[i]`.
**Data:** `receiptAlt` transformer maps the receipts table: `"Receipt from {merchant}, {date}, ${amount}"` using transaction.description, date, amount; falls back to "Receipt image, details unavailable" if fields are missing.
**Logic notes:** Alt text never includes raw filenames. Decorative icons (not receipt images) get empty alt. When a receipt is deleted, its alt entry disappears with it. Sample receipts get "(sample)" appended when the sample toggle is on.
**Acceptance:** (1) Every receipt thumbnail exposes a descriptive alt string. (2) Alt updates when the linked transaction's description changes. (3) No alt text leaks internal filenames or storage paths.

### 930. Support VoiceOver rotor navigation through transactions by date.

**Components:** Transactions table grouped by a date header row per day (`listView` with date section headers) instead of one flat grid; each header is a real heading element (`h3`) so the rotor's Headings menu lists dates.
**Data:** `txByDate` transformer groups `transactions` query rows by `date` desc; sample-toggle filter applied first so rotor never announces hidden sample rows.
**Logic notes:** Rotor actions: swipe through headings = jump day to day. Within a day, each transaction row exposes description, category, amount as its accessible name. Avoid nesting interactive elements inside the row action (edit/delete become rotor Actions on the row itself).
**Acceptance:** (1) VoiceOver rotor Headings menu shows one entry per transaction date. (2) Each row's accessible name includes description + amount. (3) No nested-focusable traps inside rows.

### 931. Make all form errors announced to screen readers with field-level links.

**Components:** Add-transaction and budget forms: an `alert` region (`aria-live="assertive"`) `errSummary` at top of form listing each error as a link; each invalid input gets `aria-describedby` pointing at its error text.
**Data:** `validateTxForm` JS query returns `[{field, message}]`; `errSummary` renders links that call `focusField(field)` on click.
**Logic notes:** Error summary appears on submit attempt and moves focus to it. Links jump focus to the offending field. Errors clear per-field as the user fixes them; the summary updates live. Messages are specific ("Amount must be a number, e.g. 42.50"), never "Invalid input".
**Acceptance:** (1) Submitting an invalid form moves focus to the error summary. (2) Every summary link focuses its field. (3) Screen reader announces the summary via the assertive live region.

### 932. Add a dyslexia-friendly font option.

**Components:** Settings toggle bound to `a11y_settings` key `dyslexia_font`; global CSS `.dyslexia { font-family: "OpenDyslexic", "Comic Sans MS", sans-serif; letter-spacing: 0.02em; line-height: 1.6; }`.
**Data:** `applyDyslexiaFont` JS query toggles the root class and persists the setting. Font files served from the app's hosted assets (no external CDN dependency for offline use).
**Logic notes:** Italics are avoided in dyslexia mode (use bold for emphasis). Line length capped at ~70ch in prose areas (help articles, summaries). The numeric figures in tables keep tabular figures for alignment. Combines cleanly with high-contrast and text-scaling.
**Acceptance:** (1) Toggle swaps the app typeface everywhere including tables and charts labels. (2) No italics render while the mode is on. (3) Works together with high-contrast mode and 200% text scaling.

### 933. Ensure color is never the only indicator (bars also show text percents).

**Components:** Budget-vs-actual bars: each bar shows overlaid text `%` (e.g. "72% of $500"); net-worth trend line adds up/down arrow + text delta; status dots pair with text labels ("Over budget").
**Data:** `barLabel` transformer: `round(actual/limit*100) + "% of " + currency(limit)`; guard divide-by-zero (limit 0 → "No limit set").
**Logic notes:** Text labels must meet contrast ratios (item 921) on top of bar colors — use dark text on light fill or place the label beside the bar. Chart legends include pattern/shape differences as a secondary cue where feasible. The SAMPLE badge is text, not color-only, by design.
**Acceptance:** (1) Every color-coded element has an adjacent text equivalent. (2) Bar percent labels remain legible in high-contrast mode. (3) Zero-division budgets render "No limit set", not NaN%.

### 934. Provide a text-only statements view for assistive technology.

**Components:** New "Statements (text)" screen: a `table` per account listing date, description, category, amount, running balance — no charts, no color coding; a "Text statements" link in main navigation.
**Data:** `textStatement` SQL/transformer per selected account + month: transactions ordered by date with running balance computed in JS (`running += amount`). Sample-toggle filter applied.
**Logic notes:** Running balance starts from the account's opening balance for the period (document the assumption in help). Tables use proper `<th scope>` headers. Print stylesheet renders the same view cleanly for paper statements.
**Acceptance:** (1) The view contains zero charts and zero color-only meaning. (2) Running balance matches the account balance at period end. (3) Screen reader table navigation (row/column headers) works throughout.

### 936. Add captions to all onboarding walkthrough slides.

**Components:** Onboarding carousel slides: each slide's `image` gets a `text` caption component beneath it describing the visual ("Screenshot: the Overview screen showing net worth of $12,400").
**Data:** Captions stored as a static `onboardingSlides` array (title, imageUrl, caption) — no query needed.
**Logic notes:** Captions are real text, not burned into images. They are concise (≤ 140 chars) and describe what the sighted user learns from the image, not just "Image of screen". Carousel dots are keyboard-operable buttons with labels ("Go to slide 2: Budgets").
**Acceptance:** (1) Every onboarding slide has a text caption. (2) Captions are perceivable to screen readers in slide order. (3) No meaningful information exists only inside the slide images.

### 937. Support switch-control navigation on iOS.

**Components:** All interactive elements reachable in a logical sequential order; no hover-only actions; custom swipe gestures on charts get button equivalents ("Next month" / "Previous month" buttons beside the chart).
**Data:** `monthNav` JS query backing the previous/next month buttons (same logic as swipe handlers).
**Logic notes:** Focus order matches visual order (item 922 covers desktop; this extends to iOS switch scanning). Every gesture-based interaction (pull-to-refresh, swipe-to-delete) has a visible button alternative. Scanning does not get trapped in the transaction table — provide "Skip table" link.
**Acceptance:** (1) Every gesture action has a visible button equivalent. (2) Switch scanning can complete add-transaction end to end. (3) No focus traps when scanning through tables.

### 938. Provide a "simplify interface" mode hiding advanced features.

**Components:** Settings toggle `tglSimplify` bound to `a11y_settings` key `simplify_ui`; conditional visibility on advanced components (rollover rules, CSV import/export, saved filters, chart drill-downs, diagnostics).
**Data:** `isSimpleMode` = `{{ a11y_settings.data.simplify_ui === 'on' }}` used in Hidden properties.
**Logic notes:** Simplified mode keeps: Overview, add transaction, transaction list, budgets bars, goals. It hides but never deletes data — toggling back restores everything. The banner, sample toggle, and help link stay visible in both modes. First-run onboarding offers the mode as a choice.
**Acceptance:** (1) Toggling simplify hides all marked advanced components, none remain. (2) No data is lost or altered by the toggle. (3) Mode choice is offered during onboarding and changeable in Settings.

### 939. Audit accessibility quarterly with a published checklist.

**Components:** Help screen section "Accessibility statement" rendering the `a11y_audit_checklist` markdown (stored in app `localStorage` or a static text asset, versioned by quarter); `btnDownloadChecklist` exports it as text.
**Data:** Checklist asset `a11y_checklist_Q3_2026.md` covering items 921–940: each with pass/fail, date, tester, notes. A `lastAuditDate` value in `a11y_settings`.
**Logic notes:** The audit re-runs the acceptance criteria of every item in this section. Failures create rows in `a11y_feedback` with status New. The published statement lists known limitations honestly (e.g. "chart drill-down not fully keyboard-operable — button alternative provided").
**Acceptance:** (1) A dated checklist exists and is viewable in-app. (2) Every item 921–940 has a pass/fail entry. (3) Failures are tracked as feedback rows until fixed.

### 940. Add an accessibility feedback channel in help.

**Components:** Help screen form `frmA11yFeedback`: topic dropdown (contrast / keyboard / screen_reader / motion / other), screen dropdown, message textarea, submit button; confirmation text on submit.
**Data:** Insert into `a11y_feedback` (status New). `a11yFeedbackList` query shows the user's own past submissions with statuses.
**Logic notes:** The form itself meets all items in this section (labels, focus, error announcement). No login or email required — it's an in-app note to the builder. Submissions are reviewed alongside the quarterly audit (item 939). Never auto-close or auto-reply with a bot message.
**Acceptance:** (1) Feedback submits with a visible confirmation. (2) User can see their submission history and statuses. (3) The form passes the same keyboard/screen-reader criteria as the rest of the app.
