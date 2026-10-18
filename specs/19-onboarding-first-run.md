# FinanceOS Builder Spec — §19 Onboarding & First-Run Setup (Items 361–380)

## Section overview
A 5-step first-run flow (welcome → sample-data choice → accounts → budgets → goals) that gets a new user from empty app to a working dashboard in about five minutes, plus the completion checklist, restart path, walkthrough slides, and audit logging around it.

## Tables touched
- `accounts`, `transactions`, `budgets`, `goals` (existing): written during onboarding steps; seeded rows carry `is_sample=true`.
- NEW `onboarding_state`: id (pk), current_step (number 1–5), steps_completed (text: JSON array of step names), sample_choice (text: "sample" / "blank" / null), payday_schedule (text: nullable, e.g. "monthly-1st"), currency (text, default "USD"), date_format (text, default "MM/DD/YYYY"), nudge_opt_in (bool, default false), completed_at (timestamp, nullable), is_sample (bool, default false). Single row per install (id=1).

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every onboarding step and every screen afterwards.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data toggle hides samples they are excluded from all totals.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.

---

### 361. Build a 5-step onboarding: welcome, sample data choice, add accounts, set budgets, add goals.
- Components: Full-screen Container `onboardingWizard` shown when `qryOnboardingState.data.completed_at` is null. Stepper `stpOnboarding` (5 steps), per-step Containers `cntStep1..5` with Back/Continue buttons, `btnSkipOnboarding` ("Skip for now" → jumps to Overview with blank data).
- Data: `qryOnboardingState` (SELECT * FROM onboarding_state WHERE id=1); `qrySaveOnboarding` upserts current_step and steps_completed JSON on every Continue.
- Logic notes: Steps are sequential; Continue validates the current step's minimum input (step 3 requires ≥1 account OR explicit "add later"). Back preserves entered values via temporary state `tmpOnboarding`.
- Acceptance: (1) Fresh install lands on step 1, not the dashboard. (2) Refreshing mid-flow resumes at the saved step with values intact. (3) Completing step 5 sets completed_at and routes to Overview.

### 362. Explain manual tracking on the welcome step with the banner text shown upfront.
- Components: Step-1 Container with the standard banner component `bannerManual` (same shared component used app-wide), Heading "Welcome to FinanceOS", Text `txtManualExplainer` (3 short paragraphs), Continue button.
- Data: Static copy; no query needed.
- Logic notes: The explainer states in plain language: no bank connection exists, every number is typed in by the user or labeled SAMPLE, and nothing updates on its own. Copy must match the banner text exactly: "Manual tracking — no bank connection yet."
- Acceptance: (1) Banner renders at the top of step 1 before any other content. (2) Copy mentions manual entry, sample labeling, and no automatic updates. (3) Banner styling matches the app-wide banner.

### 363. Offer "start with sample data" or "start blank" as an explicit first choice.
- Components: Step-2 Radio Group `rgSampleChoice` with two cards: "Start with sample data" (subtext: "Explore with labeled example numbers") and "Start blank" (subtext: "Enter everything yourself"); Continue requires a selection.
- Data: `qrySeedSample` inserts the standard sample rows (3 accounts, 15 transactions, 4 budgets, 2 goals, all is_sample=true) only when "sample" is chosen and no rows exist yet. Sets `sample_choice` in onboarding_state.
- Logic notes: Choice is explicit — no default preselected. Switching from sample to blank later is handled by the "delete all sample data" action (item 347), not by re-running onboarding.
- Acceptance: (1) Continue is disabled until a card is selected. (2) "Sample" seeds exactly the standard set, all flagged is_sample=true. (3) "Blank" creates zero rows. (4) Choice is recorded in onboarding_state.

### 364. Guide account creation with type explanations (Checking vs Savings vs Credit vs Investment).
- Components: Step-3 account form: Text Input `txtAcctName`, Select `selAcctType` (Checking/Savings/Credit/Investment), Text Input `txtInstitution`, Number Input `numStartBalance`, inline help Text under the type select, "Add another" button, list of accounts added so far with remove buttons.
- Data: `qryAddAccount` inserts into accounts; if sample_choice="sample", new rows get is_sample=false (user's real data).
- Logic notes: Each type shows a one-line explainer: Checking = everyday spending money; Savings = money set aside; Credit = money you owe (balance counts negative toward net worth); Investment = holdings tracked manually. Starting balance may be 0.
- Acceptance: (1) Each type option shows its explainer when selected. (2) At least one account can be added; empty name or non-numeric balance blocks with an inline error. (3) Added accounts list updates immediately.

### 365. Suggest starter budgets per category based on common defaults, editable before saving.
- Components: Step-4 table of the 7 fixed categories, each row with suggested limit prefilled in Number Input `numBudget_{category}`, Edit-all inline; "Use suggestions" and "Start at zero" quick buttons.
- Data: Transformer `trSuggestedBudgets` returns defaults {Housing: 1200, Food: 400, Transport: 200, Income: 0, Fun: 150, Bills: 300, Other: 100} for the current month; `qrySaveBudgets` bulk-inserts into budgets.
- Logic notes: Suggestions are editable before saving; user can zero any category. Income category budget defaults to 0 (not a spending limit). Month defaults to current month.
- Acceptance: (1) All 7 categories show a suggested value on step load. (2) Edits persist into the saved budgets. (3) "Start at zero" clears all to 0. (4) Saved budgets appear on the Budgets screen.

### 366. Prompt for one savings goal during onboarding with a skip option.
- Components: Step-5 form: Text Input `txtGoalName`, Number Input `numGoalTarget`, Number Input `numGoalSaved` (default 0), Date Picker `dtGoalDeadline` (optional), prominent "Skip" link.
- Data: `qryAddGoal` inserts into goals (is_sample=false); skip writes steps_completed without a goal row.
- Logic notes: Exactly one goal is prompted; more can be added later on the Goals screen. Deadline optional; if set, countdown chips appear on the overview (item 17). Skip is one tap, no confirmation nag.
- Acceptance: (1) Saving creates one goal visible on the Goals screen. (2) Skip completes onboarding with zero goals and no error. (3) Invalid target (≤0 or non-numeric) blocks with an inline error.

### 367. Show a completion checklist on the overview until onboarding steps are done.
- Components: Card `cardOnboardingChecklist` at the top of the Overview (below the banner), with checkbox rows: "Add an account", "Set budgets", "Add a goal", "Enter a transaction", "Update a balance"; dismisses permanently via `btnDismissChecklist`.
- Data: Transformer `trChecklistState` computes each row from live data (accounts count > 0, budgets exist for current month, goals count > 0, transactions count > 0, any account updated_at within 7 days).
- Logic notes: Checklist shows only if onboarding completed but any row is unchecked, or if onboarding was skipped. Each row deep-links to the relevant screen. Dismiss writes a flag to onboarding_state so it never returns.
- Acceptance: (1) Fresh blank install shows all 5 rows unchecked. (2) Completing an action checks its row on next Overview render. (3) Dismiss hides the card permanently.

### 368. Add a progress bar across onboarding steps.
- Components: Progress Bar `prgOnboarding` pinned at the top of the wizard container, value = current_step / 5, with text "Step X of 5 — <step name>".
- Data: Bound to `qryOnboardingState.data.current_step`; step names array in a transformer.
- Logic notes: Progress advances on Continue, retreats on Back. Skipped steps (e.g., skipped goal) still count as completed for the bar — the bar tracks flow progress, not data completeness (that's item 367's checklist).
- Acceptance: (1) Step 1 shows 20%, step 5 shows 100%. (2) Back button moves the bar backwards. (3) Label always names the current step.

### 369. Provide a 60-second video-style walkthrough using captioned slides (no audio dependency).
- Components: Modal `mdlWalkthrough` launched from a "Take the tour" button on step 1 and from Help; Carousel of 6 captioned slides with Prev/Next, progress dots, and "Got it" close.
- Data: Static slide content (script below); no queries.
- Logic notes: No audio, no autoplay video — captioned slides only, so it works muted and on slow connections. Each slide pairs one caption line with a labeled screenshot placeholder of the real screen. Slide script (6 × ~10 seconds):
  1. "FinanceOS is manual on purpose. No bank connection — every number is typed by you or clearly labeled SAMPLE." (caption over Overview screenshot)
  2. "This banner is everywhere. If you see it, you know no bank is feeding these numbers." (banner close-up)
  3. "Add accounts yourself, then tap Update balances whenever money moves." (Accounts screen)
  4. "Log spending in seconds: pick an account, a category, an amount. Done." (Add-transaction form)
  5. "Budgets compare your limits against what you actually entered — green under 80%, red over 100%." (Budget bars)
  6. "Set a goal, tap Contribute, watch the bar grow. That's the whole app — you're in control." (Goals screen, "Got it" button)
- Acceptance: (1) All 6 slides render with caption text and no audio dependency. (2) Prev/Next and dots navigate; "Got it" closes and records a seen flag. (3) Launchable from both onboarding step 1 and Help.

### 370. Ask for the user's payday schedule to prefill bill and budget cycles.
- Components: Step-2 follow-up Select `selPayday` (options: Weekly, Every 2 weeks, Twice a month, Monthly + day-of-month Number Input shown when Monthly) placed after the sample-data choice.
- Data: Saves `payday_schedule` text (e.g. "monthly-15", "biweekly") to onboarding_state; transformer `trCycleDates` derives the current budget-cycle start/end used by the Budgets screen default month range and bill due-date suggestions.
- Logic notes: Optional — "I'll set this later" link skips. Monthly requires a day 1–28 (avoids invalid Feb 30 dates); biweekly asks for the most recent payday date to anchor the cycle.
- Acceptance: (1) Monthly selection requires a valid day 1–28. (2) Saved schedule prefills the Budgets screen cycle and bill form due-day suggestions. (3) Skipping leaves defaults (calendar month) in place.

### 371. Offer to import a CSV of past transactions during onboarding.
- Components: Step-3 secondary card "Have past transactions in a spreadsheet?" with file picker `fileCsvImport` and "Import later" link; routes to the full import wizard (item 461) on file select.
- Data: Hands the uploaded file to the shared `qryCsvPreview` flow; imported rows are flagged with an "imported" source tag and is_sample=false.
- Logic notes: Import is optional and can be done later from Transactions. The onboarding step does not block on import completion — user can continue the wizard while the preview loads, or finish import after onboarding.
- Acceptance: (1) Choosing a CSV opens the import wizard with the file preloaded. (2) "Import later" continues onboarding with no penalty. (3) Imported rows land with the imported tag, not the sample flag.

### 372. Show a sample dashboard preview before the user commits to sample data.
- Components: Step-2 expandable "Preview what sample data looks like" link opening a read-only preview Container with a static mock: net-worth hero number, 3 budget bars, 2 goal progress bars — each stamped with SAMPLE badges and the manual banner.
- Data: Hardcoded mock values (clearly fake: "Sample Checking … $1,234"); no queries, no writes.
- Logic notes: Preview is illustrative only and must not be mistaken for seeded data — every figure carries a SAMPLE badge and the preview header states "This is just a preview — nothing is saved until you choose."
- Acceptance: (1) Preview opens without writing any rows. (2) Every number in the preview shows a SAMPLE badge. (3) Closing the preview returns to the choice cards with no selection made.

### 373. Add a "remind me to update balances" weekly nudge opt-in during setup.
- Components: Step-5 (or final step) Checkbox `chkBalanceNudge` "Remind me weekly to update my balances" + day/time pickers `selNudgeDay`, `timeNudge` shown when checked.
- Data: Writes `nudge_opt_in`, day, and time into onboarding_state; creates the weekly reminder row consumed by the reminders engine (item 561).
- Logic notes: Opt-in, off by default. The nudge is an in-app reminder prompting manual action — it never auto-posts or auto-updates anything (per item 574). User can change or disable it later in reminder settings.
- Acceptance: (1) Unchecked by default; checking reveals day/time pickers. (2) Saving with opt-in creates exactly one weekly reminder. (3) The reminder fires as a human-in-the-loop prompt, never an auto-write.

### 374. Explain is_sample flags in plain language during the sample-data step.
- Components: Step-2 info callout `calloutSampleExplainer` under the choice cards: "What does SAMPLE mean?" expandable text.
- Data: Static copy; no queries.
- Logic notes: Plain-language wording, no jargon: "Rows marked SAMPLE are examples so you can explore. They don't count in your real totals when hidden, and you can delete them all in one tap once you add your own numbers." Avoids the term "is_sample" in user-facing copy.
- Acceptance: (1) Explainer is visible without scrolling on the sample-data step. (2) Copy covers: examples, exclusion from real totals, one-tap deletion. (3) No database/technical terms in the visible text.

### 375. Let users set their currency and date format in onboarding.
- Components: Step-1 (or final step) "Preferences" section: Select `selCurrency` (USD, EUR, GBP, CAD, AUD, JPY, Other), Select `selDateFormat` (MM/DD/YYYY, DD/MM/YYYY, YYYY-MM-DD).
- Data: Saves `currency` and `date_format` to onboarding_state; a global transformer `trFmt` applies them to every money/date render app-wide.
- Logic notes: Defaults: USD and MM/DD/YYYY. Changing later in Settings re-renders all screens. Currency here is display-only for manual entries (multi-currency conversion is items 781–800); no FX math is implied.
- Acceptance: (1) Selections persist and apply to Overview numbers and transaction dates immediately. (2) Changing in Settings later updates all screens. (3) No live FX rates are fetched or implied.

### 376. Provide a "common bills" quick-add list (rent, electric, phone, internet) during setup.
- Components: Step-4 optional card "Add common bills" with checkbox rows (Rent, Electric, Phone, Internet, + "Other" with name field); each checked row reveals Amount and Due-day inputs; "Add bills" button.
- Data: `qryAddBills` bulk-inserts into the bills table (item 281 schema) with is_sample=false; due days default to common values (Rent=1st, others editable).
- Logic notes: Entirely optional — "Skip" continues. Amounts are user estimates, editable later on the Bills screen. Bills created here appear in the upcoming-bills tile (item 12) and the 30-day cash requirement (item 297).
- Acceptance: (1) Checking Rent + entering $1200/due-day 1 creates one bill row. (2) Skipping creates zero bills. (3) Created bills show on the Bills screen with correct due days.

### 377. Show estimated setup time ("about 5 minutes") to reduce abandonment.
- Components: Step-1 subheading Text "About 5 minutes to set up" under the welcome title; each step header shows its own micro-estimate ("Step 3 of 5 · ~1 min").
- Data: Static copy.
- Logic notes: The estimate must stay honest — steps are designed to be completable in ~1 minute each. If the user idles, no timer shames them; the text is an estimate, not a countdown.
- Acceptance: (1) "About 5 minutes" is visible on step 1 before any input is required. (2) No countdown timer or time pressure UI exists. (3) Wording stays as an estimate, not a promise.

### 378. Add a "restart onboarding" option in settings that preserves existing data.
- Components: Settings → "Onboarding" row with Button `btnRestartOnboarding` ("Restart setup tour"); confirmation modal clarifies "Your accounts, transactions, budgets, and goals will NOT be deleted."
- Data: Resets onboarding_state.current_step=1, steps_completed=[], completed_at=null; touches no other table.
- Logic notes: Restart is a guided re-tour, not a data reset — distinct from "delete all sample data" (item 347) and "delete my data" (item 773). After finishing the tour again, completed_at is re-stamped.
- Acceptance: (1) Restarting keeps all existing accounts/transactions/budgets/goals (verified by row counts before/after). (2) Wizard reopens at step 1. (3) Completing it again returns to the Overview with data intact.

### 379. Celebrate onboarding completion with a summary of what was configured.
- Components: Final "You're all set" screen: summary Card listing counts (X accounts, X budgets set, X goals, sample vs blank choice, payday schedule), primary button "Go to my dashboard".
- Data: Transformer `trOnboardingSummary` aggregates counts from accounts/budgets/goals plus onboarding_state choices.
- Logic notes: Celebration is modest — no confetti animations (consistent with the confetti-free tone of item 175). If the user skipped everything, the summary honestly says "You skipped setup — add your first account to begin" with a deep link.
- Acceptance: (1) Summary counts match actual rows created. (2) "Go to my dashboard" routes to Overview. (3) Skip-everything path shows the honest empty-state variant.

### 380. Log onboarding completion and choices in the audit log.
- Components: None user-facing; system action on wizard completion.
- Data: `qryAuditOnboarding` inserts one audit_log row (item 741 schema): action="onboarding_completed", table="onboarding_state", after-values JSON {sample_choice, accounts_added, budgets_set, goals_added, payday_schedule, currency, date_format, nudge_opt_in, completed_at}.
- Logic notes: Single summary entry, not one row per step — keeps the log readable. If onboarding is restarted and completed again, a second entry is appended (append-only log, item 744).
- Acceptance: (1) Exactly one audit entry per completed onboarding. (2) Entry contains all choice fields above. (3) Restart + re-complete appends a second entry without altering the first.
