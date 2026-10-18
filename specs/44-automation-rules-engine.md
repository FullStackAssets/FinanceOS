# FinanceOS — Section 44: Automation rules engine (items 861–880)

Detailed builder spec for the FinanceOS Retool app (manual-entry personal finance dashboard; no bank connection).

## Tables touched

Existing tables (per specs.md App 11): `accounts`, `transactions`, `budgets`, `goals` — all carry `is_sample`.

New tables:

- `automation_rules`: id (pk), name (text), folder_id (fk → rule_folders.id, nullable), trigger (text: `transaction_added` / `budget_threshold_crossed` / `bill_due` / `goal_milestone`), conditions (text, JSON array of {field, operator, value}), actions (text, JSON array of {kind, params}), enabled (bool), priority (number, lower runs first), is_ai_assisted (bool), version (number), hit_count (number), created_at (timestamp), updated_at (timestamp)
- `rule_folders`: id (pk), name (text: e.g. Spending / Bills / Goals), sort_order (number)
- `rule_runs`: id (pk), rule_id (fk), ran_at (timestamp), matched (bool), event_summary (text), action_taken (text: `none` / `draft_created` / `reminder_created` / `notification_queued` / `approval_requested`), status (text: `ok` / `error`)
- `rule_drafts`: id (pk), rule_id (fk), draft_type (text: `reminder` / `category_suggestion` / `roundup_transfer` / `transaction`), payload (text, JSON), status (text: `pending` / `approved` / `rejected`), created_at (timestamp), resolved_at (timestamp, nullable)
- `rule_versions`: id (pk), rule_id (fk), version (number), snapshot (text, JSON of the rule), archived_at (timestamp)
- `app_settings` (single-row): id (pk), rules_master_enabled (bool) — row id = 1

## Cross-cutting rules (apply to every screen/feature in this section)

1. Persistent banner "Manual tracking — no bank connection yet." renders at the top of every screen.
2. Sample rows carry `is_sample=true` + a SAMPLE badge, and are excluded from totals when the sample toggle hides them.
3. Nothing ever presents as a live bank feed. Rules act only on the user's own manual data (auto-categorize, auto-tag, budget nudges). Rules are user-created and user-paused; no money moves and no external sends — every output stays in-app.
4. Drafts over auto-writes: any rule action that would write data creates a `rule_drafts` row awaiting user approval first.

---

### 861. Build a rules list screen: trigger, condition, action, enabled state.

**Components:** New screen "Automation" with a `Table` (rulesTable: columns name, trigger label, condition summary, action summary, enabled Toggle, hit count) and a `Button` "New rule" opening a rule-builder `Modal`. **Data:** Query `getRules`: `SELECT r.*, f.name AS folder FROM automation_rules r LEFT JOIN rule_folders f ON f.id=r.folder_id ORDER BY r.priority`; `conditionSummary`/`actionSummary` transformers render JSON as readable text (e.g. "amount > $100 → create reminder"). **Logic notes:** Toggling enabled fires `UPDATE automation_rules SET enabled` then refreshes; deleting requires confirm and is user-initiated only. **Acceptance:** (1) Table lists every rule with trigger, condition, action, enabled state visible without opening a row. (2) Toggle flips enabled and persists after reload. (3) Empty state shows "No rules yet" with a CTA to the template gallery.

### 862. Support triggers: transaction added, budget threshold crossed, bill due, goal milestone.

**Components:** Rule builder form `Select` (triggerSelect) with the four trigger options; per-trigger config panel that appears contextually. **Data:** `transactions` table `onRowAdded` success handler evaluates `transaction_added` rules; a `evaluateBudgetTriggers` JS query runs after any transaction save (percent of `budgets.limit_amount` per category); bill-due and goal-milestone evaluated on app load and via a "Check now" button. **Logic notes:** Thresholds stored in rule conditions JSON (`{field:"budget_pct", operator:">=", value:90}`); bill-due uses bills due-dates data; goal-milestone uses `goals.saved/goals.target`. Debounce: one evaluation per trigger event per rule. **Acceptance:** (1) Each trigger type fires on its documented event. (2) A budget rule at 90% fires when the 91% transaction saves. (3) No trigger fires for sample-hidden data when samples are toggled off.

### 863. Support actions: create reminder, send notification, suggest category, create draft.

**Components:** Rule builder action `Select` with four action kinds + per-action param fields; an in-app `Notification` toast on fire; approvals inbox table (draftsTable from `rule_drafts`). **Data:** Action kinds stored in `actions` JSON. On match, a `dispatchActions` JS query writes `rule_drafts` (reminder/category/transaction) or an in-app notification record; `send notification` = in-app toast + notifications list only — never email/SMS. **Logic notes:** Multiple actions per rule execute in array order; each action logged to `rule_runs.action_taken`. **Acceptance:** (1) Each of the four action kinds executes when its rule matches. (2) "Send notification" never leaves the app (no email/SMS calls anywhere). (3) Every fired action creates a `rule_runs` row.

### 864. Require user approval for any action that writes data; drafts over auto-writes.

**Components:** Approvals inbox table (pendingDrafts) with per-row Approve / Reject buttons and payload preview. **Data:** All write-capable actions insert into `rule_drafts` with `status='pending'`; `approveDraft` query applies the payload (creates reminder / updates transaction category) and sets status `approved`; `rejectDraft` sets `rejected`. **Logic notes:** Only read-only actions (notification) execute immediately; every write path funnels through drafts. Bulk approve/reject supported with per-row confirmation summary. **Acceptance:** (1) No rule writes to any table without a user-approved draft. (2) Approving a category-suggestion draft updates the target transaction. (3) Rejected drafts change nothing and are logged as resolved.

### 865. Add rule templates: "alert me over $100 Fun spend in a day."

**Components:** Template gallery `Grid` of cards (template name, plain-English description, "Use template" button) shown in the rule builder and on first visit to Automation. **Data:** `rule_templates` seeded table: id, name, description, trigger, conditions JSON, actions JSON, is_ai_assisted; `useTemplate` JS clones a template into `automation_rules` (enabled=false by default) and opens it for editing. **Logic notes:** The "Fun over $100/day" template: trigger `transaction_added`, condition category=Fun AND rolling-day sum > 100, action in-app notification. Templates carry no data writes until enabled + approved. **Acceptance:** (1) Gallery lists ≥ 6 templates with plain-English descriptions. (2) Using a template creates a disabled rule prefilled with its trigger/conditions/actions. (3) User can edit every field before enabling.

### 866. Show a rule activity log with what fired and when.

**Components:** "Activity" tab with a `Table` (ruleRunsTable: ran_at, rule name, matched, event summary, action taken, status) and a rule filter dropdown. **Data:** Query `getRuleRuns`: `SELECT rr.*, r.name FROM rule_runs rr JOIN automation_rules r ON r.id=rr.rule_id ORDER BY rr.ran_at DESC LIMIT 200`. **Logic notes:** Every evaluation writes a run row, matched or not; error rows (`status='error'`) surface the error text in `event_summary`. Retention: keep last 500 runs per rule (cleanup on insert). **Acceptance:** (1) Each rule fire appears within seconds with rule name, timestamp, and action taken. (2) Non-matching evaluations also logged (matched=false). (3) Filter by rule returns only that rule's runs.

### 867. Allow testing a rule against past manual data before enabling.

**Components:** Rule builder "Test rule" button + test-results panel showing matched rows and what would have happened. **Data:** `testRule` JS query replays the rule's trigger/conditions against historical `transactions`/`budgets`/`goals` (respecting the sample toggle) and returns matches without writing anything. **Logic notes:** Dry-run flag forces `action_taken='none'` semantics; results show up to 10 sample matches with plain-English "would create reminder: …" lines. **Acceptance:** (1) Test produces match/no-match results on past data with zero writes (no `rule_drafts`, no `rule_runs` marked as live). (2) A rule that would fire daily shows its would-be hit count. (3) Test output clearly labeled "Simulation — nothing was created."

### 868. Support time-window conditions (e.g., only on weekends).

**Components:** Condition builder row: `Select` (time field: day-of-week / time-of-day), operator, `MultiSelect` (days) or time inputs. **Data:** Conditions JSON gains `{field:"dow", operator:"in", value:[0,6]}` and `{field:"hour", operator:"between", value:[9,17]}`; `matchConditions` JS evaluates using the event timestamp. **Logic notes:** Timezone: America/New_York fixed. A rule whose only mismatch is the time window logs `matched=false` with summary "outside time window". **Acceptance:** (1) Weekend-only rule fires Saturday, skips Tuesday, both logged. (2) Time-of-day window blocks fires outside the window. (3) Combined with other conditions via AND.

### 869. Add rule priority ordering when multiple rules match.

**Components:** Rules list drag handle or Up/Down buttons adjusting priority; matched-rules evaluator sorts by `priority` ascending. **Data:** `reorderRule` query swaps/sets priority values (renumber 10, 20, 30…); dispatcher processes matches in priority order and stops at the first rule marked `exclusive` (optional action param). **Logic notes:** Default: all matching rules fire (each logged); exclusive rules short-circuit lower-priority ones. Ties broken by created_at. **Acceptance:** (1) Reordering persists and the list reflects the new order. (2) Two matching rules both fire, logged in priority order. (3) An exclusive high-priority rule suppresses lower-priority matches.

### 870. Display which rules are AI-assisted vs deterministic.

**Components:** Rules table badge column ("AI-assisted" purple pill vs "Rule" grey pill); rule builder toggle "Use AI assistance for suggestions"; tooltip explaining the difference. **Data:** `is_ai_assisted` bool on `automation_rules` and templates; AI-assisted rules may only produce `category_suggestion` drafts — never direct writes or notifications without review. **Logic notes:** Deterministic = exact condition matching; AI-assisted = suggestion text generated then stored as a draft the user must approve. Banner rule applies: never present AI output as bank-verified fact. **Acceptance:** (1) Every rule shows its AI-assisted/deterministic badge in the list. (2) AI-assisted rules can only create drafts, never auto-write. (3) Disabling AI assistance converts the rule to deterministic conditions.

### 871. Allow pausing all rules with a single master toggle.

**Components:** Automation screen header `Switch` ("All rules paused") bound to `app_settings.rules_master_enabled`. **Data:** `getSettings` / `updateSettings` queries; the dispatcher checks the master flag first and skips all evaluation when paused. **Logic notes:** Master pause is independent of per-rule `enabled`; re-enabling master restores prior per-rule states. Pause state shown as a banner on the Automation screen. **Acceptance:** (1) Flipping master off stops all rule evaluation (verified: no new `rule_runs` rows on trigger events). (2) Individual enabled states are preserved while paused. (3) State persists across reloads.

### 872. Show rule hit counts to identify noisy rules.

**Components:** Rules table "Hits (30d)" column with a `Progress` sparkline; noisy-rule warning icon when hits exceed a threshold; sort by hits. **Data:** `hit_count` incremented on each matched run; `getRuleHits`: `SELECT rule_id, COUNT(*) FROM rule_runs WHERE matched AND ran_at > now()-30d GROUP BY rule_id`. **Logic notes:** Noisy threshold default 20 hits/30 days (user-adjustable in settings); noisy rules suggest adding time windows or raising thresholds via a hint link. **Acceptance:** (1) Hit counts visible per rule and sortable. (2) Rules above the noisy threshold show a warning badge. (3) Counts exclude test/simulation runs.

### 873. Support "round up to savings" rules that create confirmable drafts only.

**Components:** Template "Round up to savings": params target goal `Select`, round-to amount; approvals inbox shows roundup drafts with computed amount. **Data:** On `transaction_added` for spend, compute `roundup = ceil(amount/round_to)*round_to − amount`; insert `rule_drafts` with `draft_type='roundup_transfer'`, payload {goal_id, amount}. Approval transfers `goals.saved += amount` (manual transfer record — no money moves; it logs a contribution entry). **Logic notes:** Skip roundups under $0.25; one draft per transaction; approving twice is idempotent (draft status check). **Acceptance:** (1) A $4.30 coffee creates a $0.70 roundup draft to the chosen goal. (2) Nothing changes `goals.saved` until the draft is approved. (3) Duplicate approval attempts are rejected.

### 874. Add budget-guard rules that warn before a transaction is saved over budget.

**Components:** Add-transaction form warning banner: projected category total vs budget with "Save anyway" / "Cancel" buttons when a guard trips. **Data:** `checkBudgetGuard` JS runs on amount/category change: sums month-to-date category spend from `transactions` + the pending amount, compares to `budgets.limit_amount`. **Logic notes:** Guard is advisory, never blocking: the user can always save. Guards are rules with trigger `transaction_added` evaluated pre-save (`pre_save_check` flag). **Acceptance:** (1) Entering a transaction that would exceed the category budget shows the warning with exact numbers. (2) "Save anyway" saves normally; "Cancel" aborts with the form intact. (3) No warning when under budget.

### 875. Never let rules contact anyone externally; all outputs stay in-app.

**Components:** Rule builder validation: any action config containing URLs, email fields, or phone fields is rejected with an error. **Data:** `validateRuleActions` JS scans `actions` JSON for external-sink patterns (`http`, `mailto:`, phone regex) and blocks save. **Logic notes:** "Send notification" maps only to the in-app notification store. Code review gate: dispatcher file contains no fetch/XHR/email/SMS calls; grep-verified. **Acceptance:** (1) Saving a rule with an external action is blocked with a clear message. (2) Grep of automation code shows zero network-send calls. (3) All fired outputs are visible only inside the app (toasts, drafts, reminders).

### 876. Export/import rule sets as JSON for backup.

**Components:** Automation header buttons "Export rules" (downloads JSON) and "Import rules" (file picker + preview diff before applying). **Data:** Export serializes rules + folders + templates-used metadata (no run history); import validates schema, shows add/update/skip preview, applies only on user confirm. **Logic notes:** Imported rules default `enabled=false` for safety; ID remapping avoids collisions; malformed JSON rejected with line-level error. **Acceptance:** (1) Export downloads valid JSON containing all rules and folders. (2) Import preview lists exactly what will be added vs skipped. (3) Nothing is applied until the user confirms the preview.

### 877. Add rule versioning: edits create a new version, old ones archived.

**Components:** Rule detail drawer "Version history" list (version, edited-at, Restore button). **Data:** On every rule edit, the pre-edit snapshot is inserted into `rule_versions` and `automation_rules.version` increments; `restoreVersion` copies a snapshot back (creating another version — history is append-only). **Logic notes:** Versions store full rule JSON; run log references `rule_id` + version at fire time for auditability. **Acceptance:** (1) Editing a rule creates an archived version row with the old config. (2) Restoring an old version produces a new version, never rewrites history. (3) Version list shows every historical config.

### 878. Show a simulation mode: "what would have fired last month."

**Components:** Automation "Simulate last month" button + results panel grouped by rule (fires, would-be actions, draft counts). **Data:** `simulateLastMonth` JS replays all enabled rules against the previous calendar month's `transactions`/`budgets`/`goals` data in dry-run mode. **Logic notes:** Simulation writes nothing; results labeled "Simulation"; uses the same matcher as live dispatch so results are faithful. Large histories paginated (process in chunks of 200). **Acceptance:** (1) Simulation completes on a month of sample data without writes (zero new `rule_drafts`/`rule_runs` live rows). (2) Results show per-rule fire counts and would-be actions. (3) Banner states results are simulated, not real.

### 879. Support rule folders for organization (spending, bills, goals).

**Components:** Left sidebar folder list (ruleFolders) with counts, "New folder" inline add, drag-or-select to move rules between folders. **Data:** `rule_folders` table; `moveRuleToFolder` query sets `folder_id`; folder filter on the rules query. **Logic notes:** Deleting a folder moves its rules to "Uncategorized" (folder_id null) — rules are never deleted with a folder. Default folders seeded: Spending, Bills, Goals. **Acceptance:** (1) Rules can be assigned to folders and filtered by folder. (2) Deleting a folder preserves its rules in Uncategorized. (3) Folder counts update live.

### 880. Log rule creates, edits, enables, and disables in the audit log.

**Components:** Audit log view filtered to `entity='automation_rule'` showing actor, action, timestamp, before/after summary. **Data:** Every rule mutation (create, edit, enable, disable, delete, restore) writes to the shared `audit_log` table: (rule_id, action, actor='user', details JSON, created_at). **Logic notes:** Master-toggle pauses are also logged. Log is append-only; entries survive rule deletion (rule name denormalized into the entry). **Acceptance:** (1) Creating, editing, enabling, and disabling a rule each produce an audit entry. (2) Entries show who/when/what changed. (3) Audit entries persist after the rule is deleted.
