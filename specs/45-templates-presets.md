# Section 45 — Templates & presets (items 881–900)

**Section:** Templates & presets. Builder-ready Retool specs for the FinanceOS app (App 11).

## Tables touched

**New table — templates:**
- `templates`: id (pk), template_name (text), kind (text: Transaction / Budget preset / Goal / Bill / Recurring / Report / Checklist / Savings rule / Category note / Month setup), payload (text, JSON with the template's field values), variables (text, JSON list of {name, prompt} used in payload values), is_builtin (bool, true for shipped presets), usage_count (number, default 0), source (text: Built-in / Mine / Household), is_sample (bool), created_at (timestamp)

**Existing tables:** transactions, budgets, goals, accounts (receive applied template rows); audit_log (item 900 logs application rows; see Section 38 spec).

## Shared cross-cutting rules

- Every screen in this section shows the persistent banner **"Manual tracking — no bank connection yet."**
- Any sample template (builtin sample starters) carries `is_sample=true` plus a SAMPLE badge, and is excluded from gallery counts/totals when the sample toggle hides samples.
- Nothing ever presents as a live bank feed. Templates are user-created or user-applied starter kits; applying a template inserts **draft rows the user reviews and confirms** before they become real records.
- Editing a template never retroactively changes records already created from it.

---
### 881. Provide transaction templates: "Weekly groceries" prefills account/category/amount.

**Components:** On the Transactions screen, a "Use template" button beside "Add transaction"; a ModalFrame `templatePickerModal` with a ListView of templates filtered to kind=Transaction; selecting one opens the standard add-transaction Form prefilled. A "Save as template" icon button in the transaction edit form.

**Data:** `getTransactionTemplates` — `SELECT * FROM templates WHERE kind='Transaction' AND (is_sample=false OR {{ showSamples.value }})`. On apply: `tempState.setValue({ draft: applyVariables(template.payload, collectVariables(template)) })`; form fields bind to the draft values. `usage_count` increments on confirmed save.

**Logic notes:** Prefill is never a silent insert — the form always opens with fields editable. Template amount is editable; zero-amount template shows a warning chip. Account referenced by id must still exist; if deleted, default to the first account and show an inline notice.

**Acceptance:** (1) Choosing "Weekly groceries" opens the form with account/category/amount prefilled. (2) User can change every field before saving. (3) Saving increments the template's usage_count by 1. (4) Templates hidden when the sample toggle hides samples carry the SAMPLE badge.

### 882. Add budget presets per life situation: student, family, freelancer.

**Components:** On the Budgets screen, a "Start from preset" SplitButton; ModalFrame `budgetPresetModal` with three preset cards (Student / Family / Freelancer), each listing included categories and suggested limits; "Apply preset" confirms via DialogFrame.

**Data:** `getBudgetPresets` — `SELECT * FROM templates WHERE kind='Budget preset'`. Payload JSON: `{ items: [{category, limit_amount}] }`. Apply: loop items → `addBudget.trigger({additionalScope:{row}})` in draft mode; conflicts with an existing (category, month) budget open a conflict drawer (Skip / Overwrite / Add both).

**Logic notes:** Presets are suggestions, not mandates — limit amounts are editable post-apply. Month defaults to current; user can pick a future month before applying. Duplicate categories within one preset are deduped (last wins). Freelancer preset includes an "Income" category line item as a savings buffer note.

**Acceptance:** (1) Each preset card shows its category list and suggested totals. (2) Applying with conflicts present offers Skip/Overwrite per category. (3) Applied budgets arrive as drafts for review, not final. (4) Preset rows in templates carry is_sample=true and SAMPLE badge.

### 883. Provide goal templates: emergency fund ($1000 starter), vacation, car down payment.

**Components:** Goals screen "New from template" button; ModalFrame `goalTemplateModal` with three cards (Emergency fund — $1,000 starter; Vacation; Car down payment) showing target, suggested deadline horizon, and a one-line description. Confirming opens the goal form prefilled.

**Data:** `getGoalTemplates` — `SELECT * FROM templates WHERE kind='Goal'`. Payload: `{goal_name, target, deadline_months_offset, note}`. Apply maps deadline_months_offset → deadline date (today + offset months). Emergency fund template is marked is_sample=true.

**Logic notes:** Target values are editable; the emergency fund $1,000 is a labeled starter suggestion, not a recommendation. Deadline renders as a date picker prefilled from the offset; clearing it makes the goal open-ended. Variables like {payday} may appear in the goal note — resolved at apply time.

**Acceptance:** (1) All three goal templates appear with target, horizon, and description. (2) Apply prefills goal name, target, and deadline offset date. (3) Emergency fund template shows SAMPLE badge. (4) Editing the applied goal never changes the stored template.

### 884. Add bill templates for common obligations with typical due days.

**Components:** On the Bills screen, "Bill template" dropdown button next to "Add bill"; ModalFrame `billTemplateModal` listing common obligations (Rent/Mortgage, Electric, Phone, Internet, Insurance) with typical due-day chips (e.g., 1st, 15th). Selecting opens the bill form prefilled with payee, typical day, and amount placeholder.

**Data:** `getBillTemplates` — `SELECT * FROM templates WHERE kind='Bill'`. Payload: `{payee, typical_due_day, amount, category}`. Apply prefills form; amount left blank if template amount is null (user fills it). Template gallery search filters by payee name.

**Logic notes:** Typical due days are starting points, not facts about the user's bills. If the due day exceeds the current month's length (e.g., 31 in February), clamp to the last day and note it. Category defaults to "Bills"; user can reassign.

**Acceptance:** (1) At least five common-obligation templates exist with typical due days shown. (2) Apply prefills payee, due day, and category. (3) Due-day clamping works for short months with a visible note. (4) Template list is searchable.

### 885. Support custom user-created templates saved from any existing record.

**Components:** On transaction/budget/goal/bill detail views, a "Save as template" button; ModalFrame `saveTemplateModal` with template_name text input, kind auto-detected (readonly), and a "make available to household" toggle (default off). Success toast: "Template saved."

**Data:** `saveTemplate` — `INSERT INTO templates (template_name, kind, payload, source, created_at)` where payload is the record's fields serialized minus id/created_at. `source='Mine'`, `is_builtin=false`. Duplicate names for the same kind prompt a rename inline.

**Logic notes:** Sensitive-adjacent fields (account balance) are excluded from transaction templates — only form inputs are captured. Variables may be added later in the template editor (item 891). User templates appear in galleries immediately. Deleting a user template never deletes records created from it.

**Acceptance:** (1) Any transaction, budget, goal, or bill can be saved as a template in two clicks. (2) Duplicate name + kind triggers inline rename. (3) Saved template appears in the gallery instantly. (4) Balance fields are never captured into transaction templates.

### 886. Show a template gallery with search during relevant creation flows.

**Components:** A shared reusable container `templateGallery` (used inside all creation modals): search input, kind-filter chips, and a ListView of template cards showing name, kind pill, usage_count ("used 12×"), SAMPLE badge where applicable, and an "Apply" button. Pagination at 25 cards.

**Data:** `searchTemplates` — `SELECT * FROM templates WHERE template_name ILIKE {{ '%' + gallerySearch.value + '%' }} AND (kind = {{ kindFilter.value }} OR {{ kindFilter.value }}='All') AND (is_sample=false OR {{ showSamples.value }}) ORDER BY usage_count DESC`. Debounced 250ms.

**Logic notes:** Gallery sorts most-used first; builtins and user templates intermix (source tag distinguishes). Empty search result shows "No templates match — start from scratch?" link that closes the gallery. Gallery state resets (search cleared) each time it opens.

**Acceptance:** (1) Gallery appears inside transaction, budget, goal, bill, and recurring creation flows. (2) Search filters as you type with results under 500ms. (3) Each card shows name, kind, usage count, and badges. (4) Most-used templates sort first.

### 887. Allow sharing templates with household members.

**Components:** In the template editor, a "Share" section: toggle `shared_with_household`; a read-only note explaining that sharing exposes the template's field values to other app users in this workspace. A Household tab in the gallery showing templates with source='Household' and the sharer's label.

**Data:** Sharing flips `source` to 'Household' on a user template (row-level: Retool org visibility governs who sees it; no separate user table needed). Un-sharing reverts to 'Mine'. Template payload is the only data exposed — never live records.

**Logic notes:** This is template-metadata sharing only, not live account access; state that in the UI copy. Builtin templates are share-neutral. If the org has one user, the Household tab shows "No household members yet" instead of being empty and confusing.

**Acceptance:** (1) Sharing toggle updates the template's visibility in the Household tab. (2) UI copy states only template values are shared, not live records. (3) Un-sharing removes it from the Household tab. (4) Single-user orgs see an explanatory empty state.

### 888. Add month-start checklist template: update balances, set budgets, review bills.

**Components:** A "Month-start checklist" card on the Overview screen on the 1st–3rd of each month; ModalFrame `checklistModal` with checkbox rows (Update all account balances / Set this month's budgets / Review upcoming bills), progress ring, and "Mark month started" completion button. Dismissible for the month.

**Data:** Checklist is a `templates` row with kind='Checklist', payload `{items:[{label, done_key}]}`. User progress stored in `localStorage` keyed by month (`financeos_checklist_YYYY-MM`). Completion logged to audit_log ("Month-start checklist completed").

**Logic notes:** Checklist items deep-link to the relevant screen (balances → Accounts; budgets → Budgets; bills → Bills). Dismissal persists for the month only. The banner rule still applies inside the modal. Sample flag n/a (checklists are config, not data) — template row still carries is_sample=false.

**Acceptance:** (1) Checklist surfaces automatically on the 1st–3rd of each month. (2) Each item links to its screen. (3) Progress persists across sessions within the month. (4) Completion writes an audit_log row.

### 889. Provide report templates: landlord letter, monthly summary, tax packet.

**Components:** On the Reports screen, a "Report templates" section with three cards (Landlord letter, Monthly summary, Tax packet); each opens a preview ModalFrame with the report layout, date-range picker, and "Generate" producing a printable HTML view plus a CSV download. Generated reports include the manual-tracking disclaimer in the footer.

**Data:** `reportTemplates` — `SELECT * FROM templates WHERE kind='Report'`. Payload defines sections and which queries feed them (e.g., tax packet → transactions for the year filtered category, budgets). Generation runs existing report queries with the chosen range; no new data writes except the export log.

**Logic notes:** Reports only include real (non-sample) rows unless the user opts into samples — a "include sample rows" checkbox defaults off, and sample rows are labeled SAMPLE in the output. Landlord letter prefills name/address from a user-profile form, editable inline.

**Acceptance:** (1) Three report templates exist with previews. (2) Generated reports exclude sample rows by default and label them when included. (3) Footer carries the manual-tracking disclaimer. (4) Print layout fits standard letter size without clipping.

### 890. Add recurring-transaction starter pack (rent, phone, streaming) with confirm flow.

**Components:** First-visit prompt on the Recurring screen: "Start with a starter pack?" ModalFrame `starterPackModal` listing Rent / Phone / Streaming rows, each with editable amount, day-of-month, and account dropdowns; "Review & confirm" step shows the draft list; Confirm inserts recurring rows with `template` tag.

**Data:** Starter pack is a `templates` row kind='Recurring', payload `{items:[{description, amount, day, category}]}`. Confirm loops items → `addRecurring.trigger` in draft; rows carry `template_name='Starter pack'` on the transaction record (item 892).

**Logic notes:** This is the only template flow that inserts more than one row; the confirm screen shows the total monthly impact before the user commits. Amounts left blank are skipped with a note. Starter rows are is_sample=false; they are real user data once confirmed.

**Acceptance:** (1) Starter pack offered once on first Recurring screen visit. (2) Confirm screen shows per-row details and total monthly impact. (3) Blank-amount rows are skipped, not inserted as zero. (4) Inserted rows carry the `template` tag.

### 891. Support template variables like {month} and {payday} in names and notes.

**Components:** In the template editor, a Variables tab listing available variables ({month}, {payday}, {year}, {account_name}) with an "Insert" button that appends at cursor in the name/notes field. On apply, a small `variablePromptModal` asks for any variable values not auto-resolvable ({payday} prompts; {month} auto-resolves).

**Data:** Templates store `variables` JSON: `[{name, prompt, auto}]`. `resolveVariables(payload, values)` does string replacement across name/notes fields before prefill. Unresolved variables left in text are highlighted red in the draft form as a warning.

**Logic notes:** Auto variables resolve without prompting: {month}=current month name, {year}=year. User variables always prompt at apply time. Variables only apply to text fields — numeric fields reject variable syntax with an inline error at save time in the editor.

**Acceptance:** (1) Insert button adds variable syntax at the cursor. (2) {month} and {year} resolve automatically. (3) {payday} prompts the user at apply time. (4) Numeric fields reject variable syntax in the editor.

### 892. Show which records were created from templates via a "template" tag.

**Components:** A `template` pill (outlined, indigo) on transaction rows, budget bars, goal cards, and bill rows created from templates; tooltip shows the template name and applied date. A table filter "Created from template" on the Transactions and Bills screens.

**Data:** Applied records store `template_name` (text) — added as a nullable column to transactions, budgets, goals, and the bills/recurring tables. Filter query: `... AND template_name IS NOT NULL`. Tooltip data: `template_name || ' · ' || created_at`.

**Logic notes:** The tag is informational only — records remain fully editable and independent of the template (item 893). Bulk-applying a month template tags every created row with the same template name. Tag survives edits; it is cleared only if the user explicitly removes it via row actions.

**Acceptance:** (1) Records created from any template show the `template` pill with name tooltip. (2) Filter "Created from template" narrows lists correctly. (3) Editing the record keeps the tag. (4) Records not from templates show no pill.

### 893. Allow editing templates without affecting records already created.

**Components:** Template editor ModalFrame `templateEditorModal` (name, payload fields, variables tab, share section) with Save and Delete; a caption under the save button: "Changes apply to future uses only — records already created are untouched."

**Data:** `updateTemplate` — `UPDATE templates SET template_name, payload, variables WHERE id=...`. No cascade to child records (there is no foreign key from records back to templates — only the informational `template_name` tag). `usage_count` and history preserved across edits.

**Logic notes:** Because records carry only the tag text and not a template id, edits are inherently non-retroactive — document this as the design guarantee. Deleting a template leaves existing records' tags intact. Builtin templates are editable too; edits flip `is_builtin` to false and `source` to 'Mine' (a "modified" badge shows).

**Acceptance:** (1) Editing a template updates only future applications. (2) Previously created records keep their original values. (3) Deleting a template preserves record tags. (4) Modified builtins show a "modified" badge.

### 894. Add a "reset to sample templates" option that only touches sample-flagged rows.

**Components:** In Settings → Templates, a danger-zone button "Reset sample templates" opening a confirm DialogFrame listing exactly which builtin templates will be restored/overwritten; requires typing RESET to confirm. Result toast: "Sample templates restored (N)."

**Data:** Reset runs a guarded upsert: for each builtin template definition, `INSERT ... ON CONFLICT (id) DO UPDATE` but **only where is_sample=true**; user templates (is_sample=false) are never touched. A pre-reset snapshot of affected rows is written to audit_log for traceability.

**Logic notes:** This is the one sanctioned bulk overwrite — the WHERE clause on is_sample=true is the safety rail; the query refuses to run without it (add an assertion in the JS). Records created from templates are never modified by reset. Modified builtins (source='Mine') are offered a "restore original" per-row action instead of being force-reset.

**Acceptance:** (1) Reset only modifies rows with is_sample=true — verified by a row-count diff before/after. (2) User templates are never altered. (3) Records created from templates are untouched. (4) Confirmation requires typing RESET.

### 895. Provide category-note templates explaining each category for new users.

**Components:** On first visit to the Categories section (or via a "What is this?" link on category pickers), an info drawer showing the selected category with its explainer note template text, an example transaction line, and "what counts / what doesn't" bullets. Editable by the user per category.

**Data:** `templates` rows kind='Category note', one per category (Housing, Food, Transport, Income, Fun, Bills, Other). Payload: `{heading, explainer, examples:[], exclusions:[]}`. User edits update the template row (flips to source='Mine' per item 893 semantics).

**Logic notes:** Notes are guidance, not rules — the drawer copy says "a starting point; categorize however makes sense to you." Sample flag: builtin notes are is_sample=true but always visible (they're help text, exempt from the sample-hide toggle). Notes appear in the transaction form's category picker as a hover tooltip.

**Acceptance:** (1) Every category has an explainer note with examples and exclusions. (2) Notes appear as tooltips in the category picker. (3) User edits persist and flip the template to 'Mine'. (4) Copy frames notes as guidance, not rules.

### 896. Add savings-rule starter templates (10% rule, round-ups).

**Components:** On the Savings rules screen (see Section 34), a "Starter rules" button; ModalFrame listing "10% of income" and "Round up purchases" templates with plain-English descriptions and editable parameters (percentage, round-up target). "Enable rule" walks the confirm flow from the savings-rule spec.

**Data:** `templates` rows kind='Savings rule', payload `{rule_type, params:{...}, description}`. Applying hands off to the savings-rule creation flow (Section 34) with fields prefilled; the rule engine itself is unchanged.

**Logic notes:** Descriptions must be honest: "10% rule saves 10% of logged income entries — manual entry only, nothing moves money automatically." Round-ups are computed from logged transactions, never from a bank. Templates don't auto-enable — the user confirms in the standard rule flow.

**Acceptance:** (1) Both starter templates exist with clear descriptions. (2) Apply prefills the savings-rule flow without auto-enabling. (3) Copy states nothing moves money automatically. (4) Rule parameters are editable before enabling.

### 897. Support duplicating a full month's setup (budgets + bills + rules) as a template.

**Components:** On the Budgets screen, "Save month as template" button; ModalFrame `monthTemplateModal` with month picker, a summary of what will be captured (N budgets, M bills, K savings rules), template name input, and Save. Gallery shows it as a "Month setup" card with an "Apply to month…" action.

**Data:** `captureMonthSetup` transformer builds payload `{budgets:[...], bills:[...], savings_rules:[...]}` from the selected month's rows. Apply replays each list into draft-creation flows (budgets → draft budgets, etc.); conflicts reuse the item 882 conflict drawer.

**Logic notes:** Amounts are captured as-is; the user adjusts after applying. Applying to a month that already has budgets triggers per-category conflict resolution. This is a snapshot, not a link — later changes to the source month don't propagate (consistent with item 893).

**Acceptance:** (1) Captured template summary counts match the source month. (2) Apply replays all three lists as drafts. (3) Conflicts resolve per category. (4) Source-month edits after capture don't affect the template.

### 898. Show template usage counts to surface the most helpful ones.

**Components:** Usage count chip ("used 12×") on every template card in galleries; a "Most used" sort option (default); a small "Template insights" strip on the Templates settings page showing top-3 most applied templates this month with a bar mini-chart.

**Data:** `usage_count` increments on every confirmed apply (`UPDATE templates SET usage_count = usage_count + 1 WHERE id=...`). Insights query: `SELECT template_name, usage_count FROM templates ORDER BY usage_count DESC LIMIT 3`. Monthly counts derived from audit_log application rows (item 900) filtered by month.

**Logic notes:** Counts increment on confirm, not on gallery open or draft cancel — only completed applications. Sample templates' counts are included in sorting but excluded from insights when the sample toggle hides samples. Reset (item 894) preserves usage counts for untouched rows.

**Acceptance:** (1) Every template card shows an accurate usage count. (2) Gallery defaults to most-used sorting. (3) Insights strip shows top-3 with counts. (4) Cancelled drafts don't increment counts.

### 899. Add onboarding templates applied during first-run setup.

**Components:** During first-run onboarding (see Section 19), a "Start from a setup template" step offering "Simple starter" (basic categories + one budget preset + starter checklist) and "Blank slate"; preview lists what each applies. Choosing starter applies templates silently as drafts, then lands the user on the review screen.

**Data:** Onboarding templates are `templates` rows kind='Month setup' / 'Budget preset' / 'Checklist' flagged `is_builtin=true`. Onboarding flow calls the standard apply functions with a `silent:true` option (no per-template modals), then routes to a single consolidated review screen.

**Logic notes:** "Blank slate" must remain a first-class choice — never default the user into templates. Applied onboarding rows are real user data (is_sample=false). The review screen is mandatory: the user confirms or discards each applied draft. Onboarding template application is logged to audit_log like any other.

**Acceptance:** (1) Onboarding offers starter vs blank slate with previews. (2) Blank slate applies nothing. (3) All applied drafts land on a consolidated review screen. (4) Onboarding applications are logged to audit_log.

### 900. Log template application in the audit log.

**Components:** No new UI — every template apply writes an audit_log row: actor, timestamp, action="template.applied", detail JSON `{template_name, kind, target_month?, rows_created}`. The audit log viewer (Section 38) renders these rows with a template icon and a "view template" link.

**Data:** `logTemplateApply` — `INSERT INTO audit_log (action, detail, created_at) VALUES ('template.applied', {{ JSON.stringify(detail) }}, NOW())`. Fired from the shared `applyTemplate()` JS function so every flow (gallery, onboarding, starter pack, month setup) logs uniformly.

**Logic notes:** Logging happens on confirm, not on gallery open. Detail includes rows_created count for bulk applies (starter pack, month setup). If the audit_log insert fails, the apply still completes — log the failure to console and surface a non-blocking toast ("Applied, but audit logging failed").

**Acceptance:** (1) Every confirmed template application writes an audit_log row. (2) Detail JSON includes template name, kind, and rows created. (3) Audit viewer shows template icon + view link. (4) Apply succeeds even if logging fails, with a warning toast.
