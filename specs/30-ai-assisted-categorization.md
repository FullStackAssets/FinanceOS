# FinanceOS Builder Spec — §30 AI-assisted categorization (Items 581–600)

## Section overview
In-app suggestion engine that proposes categories, tags, account assignments, merchant-name cleanups, duplicate merges, and transaction splits. The engine is pure Retool: a JS transformer that matches transaction descriptions against learned rules stored in Retool Database. There is no external AI service — nothing leaves the app. Every suggestion is advisory: the user always confirms, corrects, or dismisses, and every interaction feeds precision stats back into the rule store.

## Tables touched
- `transactions` (existing): id, account_id (fk), date, description, category (Housing / Food / Transport / Income / Fun / Bills / Other), amount (negative=spend), is_sample.
- NEW `categorization_rules`: id (pk), match_field (text: description / merchant), keyword (text, lowercased), category (text), suggested_tags (text, comma-separated, nullable), hits (number: times rule fired), accepts (number), corrects (number), enabled (bool, default true), source (text: learned / manual), created_at (timestamp), is_sample (bool, default false). Unique index on (match_field, keyword).
- NEW `category_suggestions`: id (pk), transaction_id (fk), suggested_category (text), confidence (number 0–1), reason (text), status (text: pending / accepted / corrected / dismissed), created_at (timestamp), is_sample (bool, default false).
- NEW `merchant_aliases`: id (pk), raw_merchant (text), display_name (text), created_at (timestamp), is_sample (bool, default false). Seed sample: "AMZN MKTP" → "Amazon".

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen, including the suggestions UI.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample-data filter hides samples, they are excluded from tables, counts, totals, and suggestion training.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.
- AI suggestions are assistive only: never auto-applied; every suggestion is labeled as a suggestion, never a verified fact; all learning is local to the app — descriptions are never sent to third parties.

---

### 581. Suggest a category for new transactions based on description keyword matching.
- Components: Add-transaction form gets an inline suggestion chip `txtSuggestedCategory` under the description field ("Suggested: Food — tap to apply"), plus an Apply and Dismiss button.
- Data: Transformer `trsSuggestCategory` takes `{{ txtDescription.value }}`, lowercases it, tokenizes on non-alphanumerics, and looks up each token in `qryRules` (enabled rules). The rule with the highest (accepts / hits) weight wins. Query: `SELECT keyword, category, suggested_tags FROM categorization_rules WHERE enabled = true`.
- Logic notes: Match on whole tokens only ("amzn" won't match "amazonian"). If no rule matches, show "No suggestion yet". Suggestion recomputes on description blur, not keystroke, to avoid flicker.
- Acceptance: (1) Typing a description containing a learned keyword shows the chip with the learned category. (2) Dismissing clears the chip without touching the category field. (3) Unknown descriptions show the no-suggestion state.

### 582. Learn from the user's past manual categorizations to improve suggestions.
- Components: Background event handler `evtLearnOnSave` on transaction save; no visible UI besides a "learning on" indicator in Settings.
- Data: On every manual save, run `qryUpsertRule`: split description into tokens, and for each token increment `hits` for the (token → category) rule, creating the row if missing with source='learned'. Amount sign is also stored as a tiebreaker field (`match_field='amount_sign'` with keyword 'neg'/'pos').
- Logic notes: Minimum 2 occurrences before a rule becomes eligible to fire (prevents one-off noise). Rules learn only from transactions the user categorized manually — accepted suggestions also count as manual confirmations; corrected suggestions count toward the corrected category.
- Acceptance: (1) After the user manually categorizes "Whole Foods" as Food twice, the third entry suggests Food. (2) A one-off categorization does not create a firing rule. (3) Rules table shows source='learned' rows with incrementing hits.

### 583. Show suggestion confidence and always require user confirmation before applying.
- Components: Suggestion chip shows confidence as a percent ("Suggested: Food — 92%"), styled green ≥80%, amber 50–79%, gray <50%.
- Data: Confidence = (rule.accepts / NULLIF(rule.hits,0)) blended with prior base rate: `0.5 + 0.5 * (accepts/hits)`, then +0.1 if the same merchant string matched before, capped at 0.97. Computed in `trsSuggestCategory`; logged to `category_suggestions.confidence` on every display.
- Logic notes: Apply writes the category into the form but the transaction is only saved when the user taps Save — the suggestion never writes to `transactions` itself. Dismiss logs status='dismissed'.
- Acceptance: (1) Every suggestion displays a percent confidence. (2) Applying a suggestion still requires tapping Save on the form. (3) A dismissed suggestion is never applied and is recorded in `category_suggestions`.

### 584. Add a "suggest categories for all Uncategorized" bulk tool.
- Components: Transactions screen button `btnBulkSuggest` ("Suggest for Uncategorized"), opening modal `mdlBulkSuggest` with a table of Uncategorized rows, each with suggested category, confidence, and a checkbox (default checked only if confidence ≥ 70%).
- Data: Query `qryUncategorized`: `SELECT * FROM transactions WHERE category='Other' AND is_sample = false` (treat 'Other' as the app's Uncategorized bucket per taxonomy) — or rows with category NULL if the schema allows. Run `trsSuggestCategory` per row and bulk-insert into `category_suggestions` with status='pending'.
- Logic notes: Bulk apply writes only checked rows, and each write is a separate `transactions` UPDATE so a single failure can't corrupt the batch. Unchecked rows stay untouched. Progress toast counts applied/total.
- Acceptance: (1) Button lists only Uncategorized rows with per-row suggestions. (2) Applying checked rows updates categories and marks suggestions 'accepted'. (3) A row with no matching rule shows "no suggestion" and stays unchecked.

### 585. Display why a category was suggested ("you filed 'Whole Foods' as Food 12 times").
- Components: "Why?" link button on each suggestion chip, opening popover `popWhy` with plain-English explanation text.
- Data: Explanation is built in `trsSuggestCategory` from the winning rule row: `"You filed descriptions containing '" + keyword + "' as " + category + " " + hits + " times (" + accepts + " accepted)."`. Stored in `category_suggestions.reason` at suggestion time.
- Logic notes: Wording must be factual and derived from counts — never invented. If the rule came from manual source, say "you created a rule for…". Keep to one sentence plus the counts.
- Acceptance: (1) Every suggestion has a working "Why?" control. (2) The text names the keyword, the category, and the hit count. (3) The same reason text appears in the review queue row for that suggestion.

### 586. Allow correcting a suggestion with one tap, feeding the correction back.
- Components: On the suggestion chip, a "Not Food?" dropdown next to Dismiss listing all seven categories; selecting one applies it in one tap and closes the chip.
- Data: Writes `category_suggestions.status='corrected'`, increments `categorization_rules.corrects` for the fired rule, and immediately runs `qryUpsertRule` toward the corrected category for the description tokens — the correction is learned instantly.
- Logic notes: One tap = select the right category; no second confirmation screen. The corrected category becomes the new learning signal, so the same description suggests the corrected category next time (after the 2-occurrence threshold in 582).
- Acceptance: (1) Correcting takes exactly one tap from the chip. (2) The transaction form shows the corrected category. (3) Re-entering the same description afterward suggests the corrected category.

### 587. Never auto-categorize without explicit user approval per transaction.
- Components: Settings row `swAutoCategorize` is permanently absent — there is deliberately no auto-apply switch. The bulk modal's Apply button label reads "Apply checked suggestions (manual)".
- Data: Enforced at the query layer: `category_suggestions` rows are only ever written by suggestion generation; the only code paths that UPDATE `transactions.category` are user-initiated form saves, one-tap corrections, or the bulk modal's checked-apply — each logs status accepted/corrected.
- Logic notes: This is a hard product rule, not just UI copy. Any future automation-rule engine integration (§44) must route through the same confirmation paths; auto-apply is rejected at code review.
- Acceptance: (1) No settings toggle exists that enables auto-categorization. (2) Generating suggestions never changes `transactions.category`. (3) Every category change on an AI-touched row has a matching accepted/corrected `category_suggestions` row.

### 588. Show suggestion accuracy stats (accepted vs corrected) in settings.
- Components: Settings screen card `cardAISuggestions` with stats: acceptance rate %, total suggestions, accepted count, corrected count, dismissed count, trailing 30-day trend sparkline (Plotly mini chart).
- Data: Query `qrySuggestionStats`: `SELECT status, COUNT(*) FROM category_suggestions GROUP BY status`, plus a 30-day window variant. Acceptance rate = accepted / (accepted + corrected). Excludes is_sample rows.
- Logic notes: Dismissed suggestions are shown but excluded from the acceptance-rate denominator (dismissal means "not now", not "wrong"). Stats update on every accept/correct/dismiss event via query refetch.
- Acceptance: (1) Settings shows all five numbers and the rate. (2) Correcting a suggestion changes the rate on next view. (3) Sample transactions never appear in the counts.

### 589. Support merchant-name normalization suggestions ("AMZN MKTP" → "Amazon").
- Components: In the add-transaction form, when the description matches a raw merchant in `merchant_aliases`, show chip "Looks like Amazon — use clean name?" with Apply/Dismiss.
- Data: Query `qryMerchantAliases`: `SELECT raw_merchant, display_name FROM merchant_aliases`. Match on case-insensitive prefix of the description. Aliases can be added manually from Settings or learned when the user edits a description (store raw → cleaned mapping after 2 consistent edits).
- Logic notes: Applying replaces the description text but preserves any trailing reference codes in a `notes` field if present. Normalization never changes the amount, date, or category.
- Acceptance: (1) Typing "AMZN MKTP US" proposes "Amazon". (2) Applying rewrites the description only. (3) A user-added alias persists and fires on the next matching entry.

### 590. Add description cleanup suggestions (trim codes, expand abbreviations).
- Components: Same suggestion chip pattern as 589; label "Clean up description?" showing before → after preview text.
- Data: JS function `fnCleanupDescription` in a shared query library: trims leading/trailing transaction codes (regex `^[A-Z0-9*#]{4,}\s+`), collapses multiple spaces, expands a built-in abbreviation map (e.g. "WLMRT"→"Walmart", "MCD"→"McDonald's") merged with `merchant_aliases`.
- Logic notes: Preview is diff-style (strikethrough removed parts) so the user sees exactly what changes. Cleanup is a suggestion — the original text stays if dismissed. Never auto-runs on import; the CSV-import flow (§24) can surface it per row.
- Acceptance: (1) "  *POS  WLMRT #1234 " previews as "Walmart #1234". (2) The user sees the before/after before applying. (3) Dismissing keeps the original description byte-for-byte.

### 591. Suggest tags based on description patterns with user confirmation.
- Components: Tag chips row in the transaction form (`tagRowSuggested`) showing proposed tags with dashed-outline styling to mark them as suggestions, each with an × to reject and a "add all" button.
- Data: `categorization_rules.suggested_tags` stores comma-separated tags learned from tags the user previously assigned to matching descriptions. Transformer proposes tags alongside the category suggestion in one pass.
- Logic notes: Suggested tags never write to the transaction until confirmed; rejected tags are recorded (a `tag_rejects` count on the rule) and stop being proposed after 3 rejections. Tags depend on the tags feature (§22); if tags aren't enabled, this item's UI is hidden.
- Acceptance: (1) A description with learned tags shows dashed suggestion chips. (2) Confirming writes tags; rejecting one chip removes only it. (3) A tag rejected 3 times is no longer proposed.

### 592. Detect probable duplicates and suggest merging, user-confirmed.
- Components: Transactions screen banner `banDuplicates` ("3 possible duplicates found — review") opening modal `mdlDuplicates` with side-by-side row pairs and Merge / Not duplicates buttons.
- Data: Query `qryDuplicateCandidates`: self-join on `transactions` where ABS(amount) equal, dates within 2 days, and normalized descriptions share ≥ 60% token overlap, excluding already-merged pairs (tracked in `category_suggestions`? no — separate `duplicate_reviews` columns: add `merged_into_id (fk, nullable)` and `is_duplicate_reviewed (bool)` to `transactions` — schema note in header of file §32 references; define here: `transactions.merged_into_id`, `transactions.is_duplicate_reviewed`).
- Logic notes: Merge keeps the older row, copies the better description/notes into it, marks the other row merged_into_id (soft-delete, never hard-delete). "Not duplicates" sets is_duplicate_reviewed=true so the pair never resurfaces.
- Acceptance: (1) Two same-amount, same-week, similar-description rows appear in the modal. (2) Merge preserves one row and links the other; totals unchanged. (3) Dismissed pairs never reappear.

### 593. Suggest splitting transactions when descriptions contain multiple merchants.
- Components: In the transaction form, if the description matches the split pattern, show chip "Split into two transactions?" opening modal `mdlSplit` with two editable rows (description, amount, category) that must sum to the original amount.
- Data: Detection regex in `fnDetectSplit`: separators like `+`, `&`, `/`, `;` between merchant-like tokens, or keywords like "SPLIT". On confirm, INSERT two new transaction rows and mark the original merged_into_id → first new row (audit trail), with is_sample copied.
- Logic notes: Amounts must sum exactly to the original (validate to the cent; block confirm otherwise). Each split row gets its own category suggestion pass. Undo = the audit-log restore flow (§38) can reverse via merged_into_id.
- Acceptance: (1) "Amazon + Whole Foods $120" triggers the split chip. (2) Confirm requires the two amounts to total $120.00 exactly. (3) Original row is preserved as merged, not deleted.

### 594. Run entirely on-device or in-app; never send descriptions to third parties without consent.
- Components: Settings privacy card `cardAIPrivacy` stating "All suggestions are computed inside this app. Your transaction descriptions never leave Retool." plus a data-flow diagram (static text component, no imagery needed).
- Data: Architectural constraint, not a query: the suggestion engine is `trsSuggestCategory` (client-side JS) + Retool DB reads. No REST queries to external AI endpoints exist in this app; a code-search checklist item in the acceptance verifies this.
- Logic notes: If a future model-backed provider is ever added, it must sit behind an explicit opt-in consent modal naming the provider and data sent, default off, revocable — spec'd here as a requirement, not built.
- Acceptance: (1) Settings shows the in-app-only privacy statement. (2) No query in the app targets an external AI endpoint (verifiable by searching the query list). (3) The spec records the opt-in requirement for any future provider.

### 595. Add a "train on my history" button that rebuilds suggestions from manual data.
- Components: Settings button `btnRetrain` ("Train on my history") with confirm modal explaining it rebuilds rules from all manually-categorized transactions; progress bar during rebuild.
- Data: `qryRetrain` runs in three steps: (1) `DELETE FROM categorization_rules WHERE source='learned'`; (2) aggregate tokens from all non-sample transactions with manual categories: `SELECT LOWER(description) ... GROUP BY` processed client-side; (3) bulk INSERT rebuilt rules with recomputed hits/accepts. `category_suggestions` history is preserved for stats.
- Logic notes: Rebuild is idempotent — running twice yields the same rule set. Manual (user-created) rules are never deleted. Large histories run in batches of 500 transactions to avoid query timeouts; progress bar reflects batch count.
- Acceptance: (1) After deleting learned rules, Train restores them from history. (2) Manual rules survive the rebuild untouched. (3) Suggestion behavior after retrain matches pre-retrain behavior on a spot-checked set.

### 596. Show per-category suggestion precision to build trust.
- Components: Settings table `tblCategoryPrecision` with rows per category: suggestions fired, accepted, corrected, precision %.
- Data: Query `qryCategoryPrecision`: `SELECT suggested_category, status, COUNT(*) FROM category_suggestions WHERE is_sample=false GROUP BY 1,2`. Precision = accepted / (accepted + corrected) per category, NULL when no data (show "—").
- Logic notes: Categories with precision below 50% get an amber "needs more examples" note and their rules require 3 occurrences instead of 2 before firing (adaptive threshold). This table is the trust-building counterpart to the global stats in 588.
- Acceptance: (1) Each of the seven categories shows a precision row. (2) A category with zero history shows "—", not 0%. (3) A sub-50% category's rules stop firing until more examples exist.

### 597. Allow disabling all AI suggestions in one settings toggle.
- Components: Settings switch `swAISuggestions` (default ON), labeled "AI suggestions", with helper text "Turn off to hide every suggestion chip, queue, and bulk tool."
- Data: A single app-level variable `varAIEnabled` (localStorage-backed). Every suggestion component's Hidden property includes `&& {{ varAIEnabled.value }}`; `trsSuggestCategory` short-circuits to null when disabled. No suggestion rows are written while disabled.
- Logic notes: Disabling hides UI but preserves learned rules and stats — re-enabling restores full behavior instantly. The review queue (599) and bulk tool (584) are also hidden, not just the chips.
- Acceptance: (1) One toggle hides every suggestion surface in the app. (2) No `category_suggestions` rows are created while disabled. (3) Re-enabling restores suggestions without retraining.

### 598. Suggest account assignment for imported rows based on patterns.
- Components: CSV-import review table (§24) gains a "Suggested account" column with a per-row Apply button and the standard confidence chip.
- Data: New rule dimension on `categorization_rules`: `match_field='account_pattern'` rows map keyword → account_id learned from manual history (e.g. descriptions containing "chase" → the Chase checking account). Transformer `trsSuggestAccount` runs during import review.
- Logic notes: Account suggestions only fire when exactly one account's pattern matches strongly (confidence ≥ 75%); ambiguous rows show "review needed" instead of guessing. Applying sets the row's account_id before import commit.
- Acceptance: (1) Imported rows matching a learned pattern show a suggested account with confidence. (2) Ambiguous rows show no suggestion rather than a wrong one. (3) Applied suggestions stick through the import commit.

### 599. Flag low-confidence suggestions for manual review in a queue.
- Components: Transactions screen tab "Review queue" (`tabReviewQueue`) listing pending suggestions with confidence < 50%, sorted lowest first, each row with description, suggested category, confidence, and Accept / Correct / Dismiss actions.
- Data: Query `qryReviewQueue`: `SELECT s.*, t.description, t.amount, t.date FROM category_suggestions s JOIN transactions t ON t.id = s.transaction_id WHERE s.status='pending' AND s.confidence < 0.5 AND s.is_sample = false ORDER BY s.confidence ASC`.
- Logic notes: Acting on a queue row updates both the suggestion status and the transaction in one user-confirmed step; the queue count badge on the tab clears as items are resolved. Items 597's toggle hides the tab entirely.
- Acceptance: (1) Sub-50% suggestions appear in the queue, higher-confidence ones do not. (2) Resolving a row removes it from the queue and updates the transaction. (3) The tab badge count matches the pending low-confidence count.

### 600. Label every AI suggestion as a suggestion, never a verified fact.
- Components: Global style rule: every suggestion chip, queue row, and bulk-modal row carries the prefix label "Suggested" (chip component `lblSuggestedTag`, gray italic) and a ⓘ tooltip: "This is a suggestion based on your past entries — not a verified fact."
- Data: No query change; enforced in the component library — a shared chip template `cmpSuggestionChip` used by all suggestion surfaces, so the label can't be forgotten on a new screen.
- Logic notes: Copy audit: the words "AI detected", "identified as", or "is" (stating category as fact) are banned in suggestion UI; approved phrasings are "Suggested:", "Looks like", "Maybe". Acceptance includes a copy checklist for review.
- Acceptance: (1) Every suggestion surface renders the "Suggested" label. (2) The tooltip explains the suggestion is not a verified fact. (3) No suggestion copy states a category as fact (spot-check the seven surfaces).
