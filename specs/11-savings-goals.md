# FinanceOS — Savings Goals (items 201–220)

## Tables touched

- **goals** (extended from App 11 base): id (pk), goal_name (text), target (number), saved (number), deadline (date, nullable), category (text: emergency fund / vacation / down payment / custom — default custom), is_sample (bool, NEW), paused (bool, NEW, default false), priority (number, NEW, default 0 — lower = higher), notes (text, NEW), linked_budget_category (text, NEW, nullable — one of Housing / Food / Transport / Income / Fun / Bills / Other), completed_at (timestamp, NEW, nullable), created_at (timestamp, NEW), updated_at (timestamp, NEW)
- **goal_contributions** (NEW): id (pk), goal_id (fk → goals.id), amount (number, positive=in / negative=out), contribution_date (date), account_id (fk → accounts.id, nullable), note (text, nullable), is_sample (bool), created_at (timestamp)
- **goal_audit** (NEW): id (pk), goal_id (fk → goals.id), changed_at (timestamp), field_changed (text), old_value (text), new_value (text)

## Cross-cutting rules

- Every screen carries the persistent banner: "Manual tracking — no bank connection yet."
- Rows with is_sample=true show a SAMPLE badge and are excluded from totals when the sample toggle hides samples.
- Nothing in this app may present figures as a live bank feed; all values are manual entries or labeled sample data.
- All money movement is a manual record the user creates and confirms — no automatic transfers.

---

### 201. Build a goals screen listing each goal with goal_name, target, saved, and deadline.

- Components: Goals screen container; goalsListView (List View bound to goals query) with per-row Text components for goal_name, target, saved, deadline; progressBar201 (Progress component, value = saved/target*100); SAMPLE badge (Tag, visible when is_sample); pausedTag (Tag "Paused", visible when paused); bannerHeader (shared banner component).
- Data: `listGoals` query (Retool DB: SELECT * FROM goals ORDER BY priority ASC, created_at ASC; filter paused/is_sample client-side via toggle state). Transformer `goalRows` computes pct = target>0 ? min(saved/target,1)*100 : 0.
- Logic notes: Deadline shown as date + countdown (see §206). Empty state: "No goals yet — create your first goal" + button opening §202 form. Respect sample toggle in all rows.
- Acceptance: (1) Each row shows goal_name, target, saved, deadline. (2) Sample goals carry SAMPLE badge. (3) Paused goals show Paused tag. (4) List orders by priority, then creation date.

### 202. Add a create-goal form with name, target amount, current saved, and optional deadline.

- Components: newGoalModal (Modal); goalNameInput (Text Input, required), targetInput (Number Input, required, min 0.01), savedInput (Number Input, default 0, min 0), deadlinePicker (Date Picker, nullable), categorySelect (Select: emergency fund, vacation, down payment, custom), notesInput (Text Area), saveGoalButton.
- Data: `createGoal` query (INSERT INTO goals ...; saved defaults to current saved input; is_sample=false unless created from sample template per §208 rules).
- Logic notes: Validate target > 0 and saved ≥ 0 before submit; disable submit until valid. On success: close modal, re-run listGoals, toast "Goal created".
- Acceptance: (1) Form requires name and positive target. (2) Deadline optional. (3) New goal appears in list immediately. (4) Audit row written to goal_audit (created).

### 203. Render per-goal progress bars showing saved divided by target as a percentage.

- Components: progressBar203 (Progress) inside each goalsListView row; pctLabel (Text, e.g. "62%"); remainingLabel (Text, e.g. "$380 to go").
- Data: Transformer `goalPct`: pct = target > 0 ? round(saved/target*100) : 0; clamp 0–100 for the bar, show raw value in label (can exceed 100% if oversaved).
- Logic notes: Bar color: default blue; green when pct ≥ 100. Handle target = 0 by showing "—" and prompting to set a target. Oversaved goals show full bar + "128%" label.
- Acceptance: (1) Bar fill matches saved/target percentage. (2) Label shows exact percent. (3) Remaining amount shown. (4) 100%+ renders full bar in green.

### 204. Add a contribute button on each goal that records a manual contribution.

- Components: contributeButton per goal row (opens contributeModal); amountInput (Number Input, quick amounts $10/$25/$50/$100 chips), accountSelect (Select from accounts), noteInput (Text Input), confirmContributeButton.
- Data: `addContribution` (INSERT INTO goal_contributions; UPDATE goals SET saved = saved + amount, updated_at=now). Runs inside a Retool workflow/JS query to keep both writes together; on failure show error toast and roll back via re-fetch.
- Logic notes: Amount must be > 0. Contribution date defaults to today, editable. Every contribution is a manual record — no auto-transfer. is_sample copied from the goal.
- Acceptance: (1) Contribution increases goal saved by exact amount. (2) Row written to goal_contributions with date/account. (3) Non-positive amounts rejected. (4) Undo not required here (see §225 for edit/delete).

### 205. Support contributions from a specific account, reducing that account's balance.

- Components: accountSelect in contributeModal (from §204) now mandatory-or-"external"; balanceCheckNote (Text, warning when contribution > account balance).
- Data: `contributeWithAccount` extends addContribution: also UPDATE accounts SET balance = balance − amount, updated_at=now() WHERE id = accountId. JS query sequences: insert contribution → update goal → update account → re-run listGoals + accountsList.
- Logic notes: If "external/cash" chosen (account_id null), skip the account update. Warn (not block) when contribution exceeds account balance — user may be moving money they haven't recorded yet. Show resulting balance in the confirmation.
- Acceptance: (1) Account balance decreases by contribution amount with updated_at stamped. (2) Skipping account leaves balances untouched. (3) Over-balance shows warning but allows confirm. (4) Audit entries cover goal + account changes.

### 206. Show goal deadlines as countdowns ("42 days left") with overdue highlighting.

- Components: deadlineChip (Text/Tag) on each goal row and detail view.
- Data: Transformer `deadlineCountdown`: days = ceil((deadline − today)/86400000). Text: days>1 → "{n} days left"; =1 → "1 day left"; =0 → "Due today"; <0 → "{n} days overdue".
- Logic notes: No deadline → show "No deadline" muted. Overdue (days<0) renders chip in red with warning icon. Countdown recomputes on render — no stored value to go stale.
- Acceptance: (1) Countdown text matches date math. (2) Overdue goals highlighted red. (3) Goals without deadlines show neutral "No deadline". (4) "Due today" shown on the deadline date.

### 207. Add goal categories like emergency fund, vacation, down payment, or custom.

- Components: categorySelect on create/edit forms (Select with fixed options + "custom"); categoryTag on goal rows; categoryFilter (Select "All categories") above the goals list.
- Data: goals.category column; `listGoals` filtered client-side by selected category.
- Logic notes: Fixed set: emergency fund, vacation, down payment, custom. Custom goals show their notes as subtitle if present. Categories are labels only — no separate behavior except filtering and templates (§883).
- Acceptance: (1) Category selectable at create and edit. (2) Tag visible on each row. (3) Filter narrows the list correctly. (4) Existing goals default to custom on migration.

### 208. Display a "SAMPLE" badge on goals seeded with sample data.

- Components: sampleBadge (Tag, amber "SAMPLE") on goal row, detail view, and overview progress bars; sampleToggle chip (global, from §5) filters them.
- Data: goals.is_sample; all queries pass through the global sample filter transformer.
- Logic notes: Seeded sample goals (2 in App 11 sample data) have is_sample=true. Any goal created from a sample template (§894) also gets is_sample=true until the user clears it via "mark all real" (§57) or per-row convert (§353).
- Acceptance: (1) Sample goals always show SAMPLE badge. (2) Hiding samples removes them from list and totals. (3) New user-created goals are never sample by default. (4) Badge visible in detail view too.

### 209. Allow editing goal targets and deadlines with changes logged.

- Components: editGoalModal (prefilled: goalNameInput, targetInput, deadlinePicker, categorySelect, notesInput, linkedBudgetCategory); saveEditButton.
- Data: `updateGoal` (UPDATE goals SET ... WHERE id); `logGoalChange` (INSERT INTO goal_audit per changed field with old/new values).
- Logic notes: Diff old vs new client-side; only changed fields logged. Reducing target below current saved is allowed but shows a confirm ("Saved $800 exceeds new target $500 — goal will show over 100%").
- Acceptance: (1) Edits persist and list refreshes. (2) Each changed field has an audit row with old/new values. (3) Target-below-saved requires confirmation. (4) Deadline can be cleared.

### 210. Add a delete-goal flow that asks where to return the saved funds.

- Components: deleteGoalButton (in detail view / row menu) → deleteConfirmModal with returnAccountSelect (accounts list + "Do not move funds"), typeToConfirm (Text Input — must type goal name), confirmDeleteButton.
- Data: `deleteGoalFlow` JS query: if return account chosen and saved>0, record a withdrawal contribution (negative amount, note "Goal deleted — funds returned") and credit the account balance; then DELETE FROM goal_contributions and goals; audit rows for all writes.
- Logic notes: Funds never vanish silently — user picks a destination or explicitly leaves them. Deleting a goal with saved = 0 skips the return step. Type-to-confirm required.
- Acceptance: (1) Funds move to chosen account with balance updated. (2) "Do not move" leaves account balances untouched. (3) Goal and its contributions removed. (4) Audit log records the deletion and fund movement.

### 211. Show a projected completion date based on average manual contribution rate.

- Components: projectedDateLabel (Text) on goal detail and row subtitle, e.g. "On pace: Mar 2027".
- Data: Transformer `projectPace`: rate = total contributions (last 90 days, is_sample respected) / 3 per month; remaining = target − saved; months = rate>0 ? remaining/rate : null; projected = today + months.
- Logic notes: If no contributions in 90 days or remaining ≤ 0, show "—" (or "Complete" when saved ≥ target). Label as estimate from manual entries. Rate uses net contributions (in minus out).
- Acceptance: (1) Projection = today + (target−saved)/monthly rate. (2) No history → "—" not a bogus date. (3) Completed goals show "Complete". (4) Caption notes it is an estimate.

### 212. Render a goal detail view with contribution history and a progress timeline.

- Components: goalDetailDrawer (Drawer, opens on row click): header (name, category tag, SAMPLE badge), big progress bar, stats row (saved/target/remaining/projected date), contributionsTable (date, amount, account, note), timelineList (contributions + audit events chronologically).
- Data: `goalDetail` (SELECT goal), `goalContributions` (SELECT ... WHERE goal_id ORDER BY contribution_date DESC), `goalAudit` (SELECT ... WHERE goal_id ORDER BY changed_at DESC).
- Logic notes: Timeline merges contributions (money events) and audit (edits) sorted by timestamp, visually distinguished. Empty contributions → "No contributions yet — add the first one."
- Acceptance: (1) Drawer shows all header stats. (2) Contribution history lists every entry with account + note. (3) Timeline interleaves money and edit events. (4) Banner and sample rules apply inside the drawer.

### 213. Add milestone markers at 25%, 50%, 75% of target with check-off states.

- Components: milestoneRow (three milestone chips under the progress bar): each shows "25%" + check icon state (reached = filled, current = highlighted, upcoming = muted).
- Data: Transformer `milestones`: for each of [25,50,75], reached = saved/target*100 ≥ mark. Reached date = earliest contribution date when cumulative crossed the mark (computed from contribution history).
- Logic notes: Purely presentational — no auto-actions. 100% is the celebrate state (§219), not a milestone chip. Milestones recompute live from saved; no stored state to desync.
- Acceptance: (1) Chips reflect reached/current/upcoming correctly. (2) Reached date shown when determinable from history. (3) Works when saved exceeds target. (4) No milestone state persists stale after edits.

### 214. Support pausing a goal so it is excluded from overview progress bars.

- Components: pauseToggle (Switch/Button "Pause"/"Resume") in detail drawer and row menu; pausedTag on row.
- Data: `togglePauseGoal` (UPDATE goals SET paused = NOT paused); overview query excludes paused goals from progress-bar aggregation.
- Logic notes: Paused goals stay in the goals list (with Paused tag) but drop out of: overview progress bars (§3), totals tile (§215), and deadline countdown urgency. Contributions still allowed while paused (user may still add money). Resume restores all.
- Acceptance: (1) Paused goals excluded from overview bars and totals. (2) Still listed with Paused tag. (3) Contributions work while paused. (4) Resume restores inclusion.

### 215. Display total saved across all goals vs total targets as a summary tile.

- Components: goalsSummaryTile (top of goals screen): "Saved $X of $Y" + overall Progress bar + count "3 of 5 goals complete".
- Data: Transformer `goalsSummary`: sums saved and target over non-paused, sample-filter-respecting goals.
- Logic notes: Excludes paused goals (§214) and hidden samples. A goal with saved > target contributes its actual saved (no capping) — note in caption. Zero goals → tile hidden, empty state shown.
- Acceptance: (1) Totals match visible goals only. (2) Paused goals excluded. (3) Hidden samples excluded. (4) Overall percent = totalSaved/totalTarget.

### 216. Add goal priority ordering via drag handles on iPhone.

- Components: dragHandle (icon, 44px touch target) on each goal row; list uses a sortable list component (mobile-friendly drag); on drop, new order saved.
- Data: `saveGoalOrder` (UPDATE goals SET priority = index WHERE id for reordered rows).
- Logic notes: Priority is a plain integer; list sorts by priority ASC. Drag works with touch events; also provide up/down arrow buttons as a fallback for accessibility. New goals get priority = max+1 (append at end).
- Acceptance: (1) Drag reorders rows on touch. (2) Order persists after reload. (3) Arrow fallback reorders without drag. (4) Overview bars (§3) respect the same order.

### 217. Show a monthly contribution suggestion: (target − saved) ÷ months until deadline.

- Components: suggestionLabel (Text) on goal detail: "Suggested: $X/mo to hit your deadline".
- Data: Transformer `monthlySuggestion`: monthsLeft = max(ceil(daysUntilDeadline/30.44), 1); suggestion = (target − saved)/monthsLeft; if no deadline → "Set a deadline to see a monthly suggestion."
- Logic notes: If saved ≥ target, show "Goal funded — no monthly amount needed." Negative (overdue) deadlines use monthsLeft = 1 with an "overdue — catch-up" note. Rounded to whole dollars.
- Acceptance: (1) Math matches (target−saved)/months. (2) No deadline → prompt to set one. (3) Funded goals show funded state. (4) Overdue goals show catch-up framing.

### 218. Allow linking a goal to a budget category (e.g., vacation fund vs Fun budget).

- Components: linkedCategorySelect (Select: none + 7 categories) on create/edit forms; linkedCategoryTag on row/detail ("vs Fun budget").
- Data: goals.linked_budget_category; detail view shows side-by-side: goal progress vs that category's month spend/budget bar (reuses budget actuals query).
- Logic notes: Link is informational — it does not move money or change the budget. Changing the budget category does not affect the goal. "None" is the default.
- Acceptance: (1) Link selectable and saved. (2) Tag visible on goal. (3) Detail shows linked category's budget bar alongside. (4) Unlinking works.

### 219. Add a celebrate state when a goal reaches 100% with a manual "mark complete."

- Components: celebrateBanner (confetti-free per §175 convention — use a success card with checkmark) shown when saved ≥ target; markCompleteButton ("Mark complete"); completedTag ("Completed ✓" with date).
- Data: `markGoalComplete` (UPDATE goals SET completed_at = now()); audit row. Goal stays in list with completed tag; excluded from "active" counts but kept in history.
- Logic notes: Reaching 100% shows the celebrate card but the goal is NOT auto-completed — user taps "Mark complete". If saved later drops below target (withdrawal), completed_at clears with audit note. Celebrate copy is modest, no confetti.
- Acceptance: (1) 100%+ shows celebrate card. (2) Completion requires the manual tap. (3) Completed goals tagged with date. (4) Withdrawal below target clears completion with audit.

### 220. Include the manual-entry banner on the goals screen header.

- Components: bannerHeader (shared banner component instance) pinned at top of goals screen, goals detail drawer, and contribute/edit modals.
- Data: None — static text: "Manual tracking — no bank connection yet."
- Logic notes: Uses the same shared banner component as every other screen (§341) so wording and behavior (dismiss-per-session, settings position) stay consistent. Banner never overlaps the summary tile or FAB on small screens.
- Acceptance: (1) Banner visible at top of goals screen on load. (2) Present in detail drawer and modals. (3) Wording identical to other screens. (4) Dismiss/position behavior matches global banner settings.
