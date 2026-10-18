# FinanceOS Builder Spec — Section 08: Budgets setup (items 141–160)

Covers the budgets screen: creating, editing, templating, reviewing, and managing per-category monthly limits from manual data only.

## Tables touched

- `budgets` (existing, App 11): id (pk), category (text: Housing / Food / Transport / Income / Fun / Bills / Other), month (text, `YYYY-MM`), limit_amount (number), is_sample (bool — add this column; the original App 11 schema lacks it, and budget sample rows must be flaggable), notes (text, nullable), created_at (timestamp), updated_at (timestamp)
- `budget_templates` (NEW): id (pk), template_name (text, e.g. "Lean month", "Typical month"), category (text), limit_amount (number), is_sample (bool), created_at (timestamp)
- `audit_log` (NEW, shared later by section 38): id (pk), actor (text), action_at (timestamp), table_name (text), record_id (text), before_json (text), after_json (text), entry_method (text: manual/import/sample/quick-add), note (text)

## Cross-cutting rules

- Every screen shows the persistent banner "Manual tracking — no bank connection yet." (item 160: pinned in the budgets screen header).
- Sample rows carry `is_sample=true` + a visible "SAMPLE" badge, and are excluded from budget totals/actuals when the sample toggle hides them.
- Nothing ever presents as a live bank feed; all figures derive from manually entered transactions.

### 141. Build a budgets screen with one row per category showing month and limit_amount.

- Components: Container `budgetsScreen`; month Selector `budgetMonth` (options = last 12 months + current, default current `YYYY-MM`); Table `budgetTable` with columns category, month, limit_amount, actual (transformer-computed), status badge (Set / Unset); banner Text pinned at top (item 160).
- Data: Query `getBudgets` → `SELECT * FROM budgets WHERE month = {{budgetMonth.value}} ORDER BY category`; transformer `budgetRowsWithActual` joins the fixed 7-category taxonomy to the query result so every category renders a row even when no budget row exists (missing = Unset).
- Logic notes: The 7 categories are fixed (Housing, Food, Transport, Income, Fun, Bills, Other); Income renders in its own section below spending rows. Rows with no matching budget get `limit_amount = null` and the "unset" badge (item 145).
- Acceptance: (1) All 7 categories render one row each for the selected month. (2) Changing the month selector re-queries and updates rows. (3) Rows without a budget show the Unset badge, not $0. (4) The manual-entry banner is visible above the table.

### 142. Add a set-budget form: category picker, month selector, limit amount.

- Components: Modal `setBudgetModal`; Select `budgetCategory` (7 fixed categories); Select `budgetFormMonth` (month options, default current); Number input `budgetLimit` (min 0, step 1, prefix $); Text area `budgetNote` (item 149); Button `saveBudget` (disabled until category + positive limit); Button `cancelBudget`.
- Data: Query `upsertBudget` → insert or update `budgets` on (category, month): `INSERT ... ON CONFLICT (category, month) DO UPDATE SET limit_amount=..., notes=..., updated_at=NOW()` (add a unique index on (category, month)); on success, refresh `getBudgets` and write an audit row.
- Logic notes: Save performs an upsert so re-saving a category/month edits rather than duplicates. Validation per item 147 runs on submit.
- Acceptance: (1) Form opens from the "Set budget" button and from tapping an Unset row. (2) Missing category or non-positive limit blocks save with an inline error. (3) Saving writes exactly one row per (category, month) and refreshes the table. (4) Cancel discards all input.

### 143. Default the budget month to the current month on form open.

- Components: Reuse `setBudgetModal` / `budgetFormMonth`.
- Data: On modal open, set `budgetFormMonth.setValue(dayjs().format('YYYY-MM'))` via the modal's open event handler.
- Logic notes: Month options are generated as the last 12 completed months plus the current month and next 2 months (for planning ahead). The default always tracks the device's current month, not a stale cached value.
- Acceptance: (1) Opening the form in any session preselects the current month. (2) The user can still pick another month without the default fighting them. (3) Preselected month displays in a readable "October 2026" label format.

### 144. Allow copying last month's budgets to the current month in one action.

- Components: Button `copyLastMonth` ("Copy last month's budgets") in the budgets screen toolbar; confirmation Modal `copyConfirm` listing the categories/limits that will be copied; Button `confirmCopy`.
- Data: Query `copyBudgets` → `INSERT INTO budgets (category, month, limit_amount, notes, is_sample) SELECT category, {{targetMonth}}, limit_amount, notes, is_sample FROM budgets WHERE month = {{sourceMonth}} ON CONFLICT (category, month) DO NOTHING`; then refresh `getBudgets`; audit-log one bulk entry with affected-row count.
- Logic notes: Existing current-month budgets are never overwritten (DO NOTHING on conflict). Copy preserves the is_sample flag so sample rows stay sample. Source month = the month immediately before the selected month.
- Acceptance: (1) One click copies all source-month budgets that don't already exist in the target month. (2) Confirmation lists exactly what will be copied. (3) Pre-existing target budgets are untouched. (4) The audit log records the bulk copy with a row count.

### 145. Show which categories have no budget set with an "unset" badge.

- Components: In `budgetTable`, a status column rendering a Badge component: grey "Unset" when `limit_amount IS NULL`, green "Set" otherwise.
- Data: `budgetRowsWithActual` transformer outputs `status: 'set' | 'unset'` per row.
- Logic notes: "Unset" is distinct from a $0 budget — $0 is a deliberate zero-spend budget and renders "Set" with $0. Tapping an Unset row opens `setBudgetModal` prefilled with that category.
- Acceptance: (1) Categories without a budget row show the Unset badge. (2) $0 budgets show Set, not Unset. (3) Tapping an Unset row opens the form with the right category preselected. (4) Badge colors meet contrast minimums on mobile.

### 146. Add budget templates (e.g., "Lean month", "Typical month") that fill all categories.

- Components: Select `templatePicker` ("Apply template…") on the budgets screen; Modal `templateManager` listing templates with Edit/Delete; Button `newTemplate` ("Save current month as template").
- Data: `budget_templates` table; query `applyTemplate` inserts one `budgets` row per template row for the selected month (`ON CONFLICT DO NOTHING`, preserving existing budgets); seed rows: "Lean month" and "Typical month" with all 7 categories, `is_sample=true`; user-created templates save from the current month's budgets via `saveTemplateAs`.
- Logic notes: Applying a template never overwrites existing budgets for the month. Template amounts are editable before saving a template. Sample templates are labeled and deletable without touching real budgets.
- Acceptance: (1) "Lean month" and "Typical month" ship as sample templates. (2) Applying a template fills all missing category budgets for the selected month. (3) Existing budgets are not overwritten by a template. (4) Users can save, rename, and delete their own templates.

### 147. Validate that budget limits are positive numbers.

- Components: On `budgetLimit` Number input: `min=0.01`, required; inline Text `budgetLimitError` ("Enter an amount greater than $0").
- Data: Validation runs client-side on input/change and again in the `upsertBudget` guard; server-side, a CHECK constraint `limit_amount > 0` on `budgets`.
- Logic notes: Zero is rejected — a zero-spend intention is expressed via the separate "pause budget" concept (section 10, item 199), keeping setup validation simple and unambiguous. Non-numeric input is blocked by the number input type.
- Acceptance: (1) Empty, zero, or negative limits block save with a clear message. (2) The database CHECK rejects bad writes even if client validation is bypassed. (3) The error message names the field. (4) Valid positive decimals (e.g. 149.99) save correctly.

### 148. Support editing a budget limit mid-month with the change logged.

- Components: In `budgetTable`, an edit IconButton per Set row opening `setBudgetModal` prefilled (title "Edit budget"); the modal's save button relabels to "Save changes".
- Data: `upsertBudget` performs the update; before the write, capture the old row and insert an `audit_log` entry (table_name='budgets', before/after JSON of limit_amount + notes, entry_method='manual', note='mid-month edit').
- Logic notes: Mid-month edits recompute budget-vs-actual bars immediately (section 09, item 180). The audit entry preserves the old limit so the user can see what changed and when.
- Acceptance: (1) Editing a limit updates the row in place without creating a duplicate. (2) Every mid-month edit writes an audit entry with old and new values. (3) Budget-vs-actual visuals refresh immediately after save. (4) Editing notes alone also logs an audit entry.

### 149. Add per-budget notes explaining the reasoning ("cut Fun to $150 for trip savings").

- Components: Text area `budgetNote` in `setBudgetModal` (maxlength 280, placeholder with the trip-savings example); in `budgetTable`, a notes icon with a Tooltip showing the note on hover/tap.
- Data: `budgets.notes` column; included in `getBudgets` select and in the `upsertBudget` write; surfaced in the audit log after-image.
- Logic notes: Notes are optional and never affect calculations. They render truncated to one line in the table with full text in the tooltip/drawer.
- Acceptance: (1) Notes save and reload with the budget. (2) Rows with notes show a visible notes indicator. (3) The full note is readable on iPhone without layout breakage. (4) Notes are included in the month CSV export (section 25, item 482).

### 150. Display budgets in a card layout optimized for iPhone with big progress bars.

- Components: Below 640px width, `budgetTable` hides and a ListView `budgetCards` renders instead: each card shows category icon + name, limit, spent-to-date, remaining, and a Progress bar (height 12px, min touch target 44px for the whole card). Cards are full-width tappable rows (opens edit modal).
- Data: Same `budgetRowsWithActual` transformer feeds both the table and the card list; spent-to-date comes from the shared actuals computation (section 09, item 161).
- Logic notes: The switch is CSS/media-query based so rotating the phone re-renders instantly. Card progress bars reuse the green/amber/red thresholds from section 09 (item 163) for consistency.
- Acceptance: (1) At 390px width the budgets screen shows cards, never the desktop table. (2) Every card element is ≥44px touch target. (3) Progress bars match the desktop table's percentages exactly. (4) Cards scroll vertically with no horizontal overflow.

### 151. Add a total monthly budget sum across all categories.

- Components: Stat `totalBudgetStat` ("Total budgeted") pinned above `budgetTable`/`budgetCards`, formatted as currency.
- Data: Transformer `totalBudgeted = SUM(limit_amount) WHERE category != 'Income' AND limit_amount IS NOT NULL` for the selected month; Income-category budgets (if any) are shown as a separate "Planned income" line, not folded into the spend total.
- Logic notes: Unset categories contribute $0 to the sum. Sample budgets are included only when the sample toggle is on; a small "(excludes samples)" or "(includes samples)" caption reflects the toggle state.
- Acceptance: (1) The total equals the visible rows' limits summed. (2) Income rows are excluded from the spend total. (3) The total updates when the month selector changes. (4) A caption states whether sample budgets are included.

### 152. Allow deleting a budget for a category/month with confirmation.

- Components: Delete IconButton per Set row → confirmation Modal `deleteBudgetConfirm` ("Delete the Food budget for October 2026? Actuals already recorded are kept."); Buttons Delete/Cancel.
- Data: Query `deleteBudget` → `DELETE FROM budgets WHERE id = {{selectedRow.id}}`; audit-log the deletion with the before-image; refresh `getBudgets`.
- Logic notes: Deleting a budget never deletes transactions — actuals simply lose their comparison baseline and the row returns to Unset. The confirmation names the exact category and month to prevent mistakes.
- Acceptance: (1) Deletion requires explicit confirmation; no one-tap deletes. (2) After deletion the row shows the Unset badge. (3) Recorded transactions are untouched. (4) The audit log records who deleted what and when.

### 153. Show the previous three months' limits per category when setting a new one.

- Components: In `setBudgetModal`, a "Recent limits" strip below the amount input listing the last 3 months' limits for the chosen category as tappable chips ("Sep: $400").
- Data: Query `recentBudgetLimits` → `SELECT month, limit_amount FROM budgets WHERE category = {{budgetCategory.value}} ORDER BY month DESC LIMIT 3`, re-run when the category picker changes.
- Logic notes: Tapping a chip fills `budgetLimit` with that amount. Months with no budget show "—" (not $0) to avoid confusion. This is the manual, in-form version of the "suggested budget" hint (item 157).
- Acceptance: (1) The strip shows up to 3 prior months for the selected category. (2) Switching category refreshes the strip. (3) Tapping a chip prefills the amount field. (4) Empty months render as "—", never as $0.

### 154. Add a budget calendar view showing limits by month across the year.

- Components: Toggle `budgetViewToggle` (List / Calendar) on the budgets screen; `budgetCalendar` grid: 12 month columns × 7 category rows, cells show the limit or "—"; year Selector `budgetYear`.
- Data: Query `getYearBudgets` → `SELECT category, month, limit_amount FROM budgets WHERE month LIKE {{budgetYear}} || '-%'`; transformer pivots to a category × month matrix.
- Logic notes: Cells are tappable to open `setBudgetModal` for that category/month. The current month's column is highlighted. Totals row per month at the bottom mirrors item 151's sum.
- Acceptance: (1) All 12 months × 7 categories render for the selected year. (2) Tapping a cell opens the set/edit form for that exact category and month. (3) The current month column is visually highlighted. (4) Monthly totals row matches the list view's total for each month.

### 155. Warn when the sum of budgets exceeds average monthly income from manual data.

- Components: Warning Callout `overBudgetWarning` above the budgets table, shown conditionally; text: "Your budgets ($X) exceed your average monthly recorded income ($Y) — based on your manual entries."
- Data: Transformer `avgMonthlyIncome` = total of Income-category transactions over the last 6 full months ÷ 6 (excluding samples when the toggle is off); compare against `totalBudgeted` from item 151.
- Logic notes: The warning is informational, not blocking — saving is still allowed. It only appears when there are ≥2 months of income history (otherwise "not enough data" state). All figures are labeled as from manual entries.
- Acceptance: (1) The warning appears exactly when total budgets > 6-month average income. (2) It names both dollar figures. (3) It never blocks saving a budget. (4) With <2 months of income data it shows a neutral "not enough history" note instead.

### 156. Support zero-based budgeting mode where every dollar of income is assigned.

- Components: Toggle `zeroBasedMode` in budgets screen settings; when on, a "Assign every dollar" panel shows: Planned income (from the Income budget or income average), Assigned total, Unassigned remainder (must reach $0); unassigned renders as a red "Give every dollar a job: $X left" banner until zero.
- Data: Transformer `zeroBasedMath`: plannedIncome (Income-category budget for the month, falling back to the 3-month income average with a "using average" label), assigned = sum of spending-category limits, unassigned = plannedIncome − assigned.
- Logic notes: This is a planning view only — it never moves money or creates transactions (manual-entry rule). A "Distribute remainder" helper can proportionally top up existing category budgets with one confirmation, each logged to the audit log.
- Acceptance: (1) The panel shows income, assigned, and unassigned with the unassigned figure prominent. (2) Reaching $0 unassigned shows a clear "balanced" state. (3) Nothing auto-posts transactions in this mode. (4) The fallback-to-average path is labeled, never presented as real income.

### 157. Add a "suggested budget" hint based on the 3-month manual average per category.

- Components: In `setBudgetModal`, a hint line under the amount input: "Suggested: $X (your 3-month average for Food)" with a "Use" button that fills the field.
- Data: Query `categoryThreeMonthAvg` → average monthly spend per category over the last 3 full months from `transactions` (negative amounts, absolute value; excluding samples when the toggle is off, excluding Income).
- Logic notes: The suggestion is a hint, never auto-applied — the user must tap "Use". With <3 months of history it uses whatever months exist and labels itself ("based on 2 months"). Rounds to whole dollars.
- Acceptance: (1) The hint shows for the selected category with its basis labeled. (2) "Use" fills the amount without saving. (3) Partial history is labeled honestly. (4) The hint never overwrites a value the user already typed without confirmation.

### 158. Let users duplicate a single category's budget to future months.

- Components: In each budget row, an overflow menu action "Duplicate to future months" → Modal `duplicateBudgetModal` with multi-select of upcoming months (next 6) and a preview list; Button `confirmDuplicate`.
- Data: Query `duplicateBudget` inserts one `budgets` row per selected month with the same category, limit, notes, and is_sample flag (`ON CONFLICT DO NOTHING`); audit-logged as a bulk action with row counts.
- Logic notes: Months that already have a budget for that category are skipped, never overwritten, and the confirmation lists exactly which months will receive the budget vs which were skipped.
- Acceptance: (1) The modal lists the next 6 months with checkboxes. (2) Existing budgets for a month are skipped, not overwritten. (3) The preview shows the per-month result before confirming. (4) The bulk insert is audit-logged with affected counts.

### 159. Show budget history per category as a small inline trend.

- Components: In each `budgetTable` row (and card), a Sparkline `budgetHistorySpark` (last 6 months' limits) rendered as a tiny inline line chart next to the limit amount.
- Data: Query `budgetHistorySpark` → `SELECT month, limit_amount FROM budgets WHERE category = {{row.category}} ORDER BY month DESC LIMIT 6`, reversed chronologically for the sparkline.
- Logic notes: The sparkline shows limits, not actuals (actuals live in section 09). Missing months break the line rather than rendering as zero. Tooltip on tap shows "Sep 2026: $400".
- Acceptance: (1) Each row shows a 6-month limit trend. (2) Gaps render as breaks, never as $0 dips. (3) Tapping/hovering reveals exact month + amount. (4) The sparkline stays legible at mobile card sizes.

### 160. Include the manual-entry banner on the budgets screen header.

- Components: Banner Text pinned at the very top of `budgetsScreen`, above the toolbar: "Manual tracking — no bank connection yet." styled as a slim persistent strip (dismissible per session per section 18, item 342, but restored on next load).
- Data: None — static text with a session-level `bannerDismissed` temp-state flag.
- Logic notes: The banner is part of the budgets screen layout, not an overlay, so it never covers the month selector or the total-budgeted stat. It inherits the global banner component for consistent styling and behavior across screens.
- Acceptance: (1) The exact banner text renders at the top of the budgets screen on load. (2) It does not overlap any interactive control at 390px width. (3) Session dismissal works and restores on reload. (4) It uses the same component/style as the banner on other screens.
