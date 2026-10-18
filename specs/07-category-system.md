# FinanceOS Build Spec — 07. Category system (items 121–140)

Section of the FinanceOS 1000-item roadmap covering the fixed category taxonomy, its presentation, management tooling, and auditability. FinanceOS is manual-entry only; no bank connection exists.

## Tables touched

**Existing:** `transactions` (read/updated category values), `budgets` (read per-category limits).

**New — `categories`:** id (pk), name (text, immutable canonical value), display_label (text), icon (text, emoji), color (text, hex), description (text), notes (text), sort_order (number), default_monthly_budget (number), tracked_closely (bool), is_sample (bool). Seeded with the 7 canonical rows: Housing 🏠 #4C8CFF, Food 🍽️ #FF9F43, Transport 🚗 #22C55E, Income 💰 #10B981, Fun 🎉 #A855F7, Bills 🧾 #EF4444, Other 📦 #6B7280.

**New — `transaction_splits`:** id (pk), transaction_id (fk → transactions), category (text, canonical), amount (number). Parent amount must equal the sum of its splits.

**New — `audit_log`:** id (pk), actor (text), ts (timestamp), table_name (text), record_id (text), action (text), before_value (text), after_value (text), note (text), is_sample (bool). Append-only; no edit/delete UI.

## Cross-cutting rules

- Every screen shows the persistent banner "Manual tracking — no bank connection yet."
- Sample rows carry is_sample=true plus a SAMPLE badge and are excluded from every total when the sample toggle hides them.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled sample data.

---

### 121. Fix the category taxonomy to Housing, Food, Transport, Income, Fun, Bills, Other across the app.

- **Components:** Rebind every category Select/Dropdown/Multiselect (add-transaction form, Transactions filters, Budgets screen, recurring templates) to a shared `qryCategories` query instead of hardcoded option lists; add a `categoryOptions` JS transformer mapping rows to {label: display_label, value: name}.
- **Data:** Seed the `categories` table with the 7 canonical rows. One-time migration query normalizes legacy strings: case-insensitive match to a canonical name, else map to Other; report the remap counts before confirming.
- **Logic notes:** All writes validate `category ∈ {canonical names}` and reject anything else. Stored transaction values always use the immutable `name`; UI renders `display_label`.
- **Acceptance:** (1) Every category picker in the app lists exactly the 7 categories. (2) Submitting a form with a non-canonical category is rejected. (3) Post-migration, zero transactions hold a non-canonical category value.

### 122. Show category icons and colors consistently on forms, tables, and charts.

- **Components:** Shared JS `categoryStyle(name)` returning {icon, color} from the `categories` table; use it in table column HTML mappings (`<span>{{categoryStyle(c).icon}}</span>`), Plotly chart color arrays, and Select components with custom option rendering.
- **Data:** Read icon/color from `categories` at load into a `categoryStyleMap` temp state; refresh on table edit.
- **Logic notes:** No component hardcodes a color — unknown values fall back to Other's style. Text rendered on colored chips uses a contrast-safe foreground (white/black chosen by luminance).
- **Acceptance:** (1) The same category renders an identical icon and color on forms, tables, and charts. (2) Editing a category's color updates every screen on refresh. (3) Chip text meets WCAG AA contrast against its background.

### 123. Add a category management screen showing each category's month-to-date total.

- **Components:** New "Categories" screen under the More menu: Table `tblCategories` (icon, display_label, color chip, MTD total, budget, notes indicator), "Edit" drawer button per row, banner in header.
- **Data:** `qryCategoryMTD` aggregates `transactions` by category for the current month: spend categories sum ABS(amount), Income sums positive amounts. Honors the sample toggle and the "as of" manual-update semantics.
- **Logic notes:** MTD window = first of month through today; future-dated "scheduled" transactions excluded from MTD but shown in a separate count.
- **Acceptance:** (1) All 7 categories appear with MTD totals matching the Transactions table filtered to this month. (2) Hiding sample data removes sample rows from totals. (3) The manual-entry banner renders in the screen header.

### 124. Allow renaming the display label of a category without changing stored values.

- **Components:** Edit drawer on the Categories screen with TextInput `displayLabelInput` and Save; validation message for duplicates.
- **Data:** Update only `categories.display_label`; `transactions.category` keeps the immutable canonical `name`. All UI joins to `categories` for display.
- **Logic notes:** Two categories may not share a display label. Exports use display_label with a footnote mapping to canonical names. Audit log records old → new label.
- **Acceptance:** (1) Renaming "Fun" to "Entertainment" changes every screen while stored values remain "Fun". (2) Duplicate display labels are rejected with a clear error. (3) The rename appears in the audit log with before/after values.

### 125. Add per-category monthly default budgets that prefill the budgets screen.

- **Components:** Number inputs `defaultBudgetInput` per row on the Categories screen; Budgets screen button "Apply defaults" plus per-row "Use default" on the add-budget form.
- **Data:** `categories.default_monthly_budget`. Apply-defaults inserts one `budgets` row per category for the current month, skipping categories that already have a row.
- **Logic notes:** Defaults are suggestions only — never auto-created without the user pressing Apply. Income categories default to 0 (not budgeted as spend).
- **Acceptance:** (1) The add-budget form prefills the limit from the category default. (2) Apply-defaults creates exactly one row per missing category/month. (3) Editing a default does not alter already-saved budgets.

### 126. Show uncategorized (Other) totals prominently to nudge better categorization.

- **Components:** Overview alert card `otherNudgeCard`: "N transactions in Other totaling $X — tap to review"; quick-filter chip "Uncategorized" on the Transactions screen.
- **Data:** Reuses the MTD aggregation; counts transactions with category = Other this month.
- **Logic notes:** Card appears only when Other MTD exceeds max($50, 10% of MTD spend). Tap-through opens Transactions pre-filtered to category=Other, this month. Dismissible per session.
- **Acceptance:** (1) Card appears exactly when the threshold is crossed and hides otherwise. (2) Tap-through lands on the correctly filtered transaction list. (3) With samples hidden, sample Other rows don't trigger the card.

### 127. Add category descriptions explaining what belongs in each (e.g., Fun = entertainment).

- **Components:** Description column on the Categories screen; helper text under the category picker on the add-transaction form; legend with descriptions on the Overview donut.
- **Data:** `categories.description`, seeded: Housing = rent/mortgage, property tax, home insurance; Food = groceries, dining out, coffee; Transport = fuel, transit, rideshare, car upkeep; Income = paychecks, freelance, gifts received; Fun = movies, games, concerts, entertainment subscriptions; Bills = phone, internet, utilities, insurance premiums; Other = anything not fitting above.
- **Logic notes:** Descriptions are editable and shown anywhere a category is chosen or explained; they never affect stored values.
- **Acceptance:** (1) Every category has a non-empty description visible on the management screen. (2) The add-transaction form shows the description beneath the picker. (3) Edited descriptions persist and render everywhere.

### 128. Support splitting one transaction across two categories with amounts per split.

- **Components:** Add-transaction form "Split" toggle revealing two rows (category Select + amount Number input each); Transactions table shows a split badge (⿻) opening a splits popover.
- **Data:** `transaction_splits` rows linked to the parent transaction; splits inherit the parent's is_sample flag. Budget actuals aggregate splits by category, and the parent transaction is excluded from category totals when splits exist.
- **Logic notes:** Save blocked unless amount A + amount B equals the transaction total within $0.01 and the two categories differ. Deleting the parent cascades to its splits. Exactly two splits per the item spec.
- **Acceptance:** (1) A $100 split as $60 Food / $40 Housing shows correctly in both categories' budget actuals. (2) Unequal split totals block saving with a clear error. (3) Deleting the parent transaction removes its split rows.

### 129. Show a category detail view: all transactions, budget, trend for that category.

- **Components:** "Category detail" screen reached via deep link `?category=Food`: header (icon, display_label, description, notes), stat cards (MTD spend, budget, remaining, 6-month average), 6-month trend line chart, transactions Table filtered to the category.
- **Data:** Parameterized queries `qryCategoryDetail`, `qryCategoryTrend` (monthly totals, last 6 months, manual transactions only).
- **Logic notes:** Trend chart carries the "manual data only" caption. Budget card shows "no budget set" state when missing. Sample toggle respected throughout.
- **Acceptance:** (1) Tapping any category bar/row opens its detail with the right category preselected. (2) Detail shows current-month budget vs actual. (3) Trend renders six monthly points or an honest empty state.

### 130. Add a "recategorize all matching" tool that reassigns by description keyword.

- **Components:** Modal launched from the Transactions or Categories screen: TextInput keyword, target-category Select, "include sample rows" checkbox, preview list (count + total), Confirm button.
- **Data:** Preview query matches `description ILIKE %keyword%`; confirm runs an UPDATE over the matched ids. Writes one audit_log row noting keyword, target, and affected count.
- **Logic notes:** Sample rows skipped unless explicitly included. Preview must render before Confirm is enabled; confirm requires typing the count or a second click to avoid accidents.
- **Acceptance:** (1) Preview shows the exact affected count and total before applying. (2) Only rows matching the keyword are reassigned. (3) The audit log records the bulk operation with keyword and count.

### 131. Prevent deleting a category that has transactions; require reassignment first.

- **Components:** Categories screen: no delete affordance on the 7 canonical categories (system-protected, shown with a lock tooltip explaining the fixed taxonomy). Reassignment tool (item 130) doubles as the migration path.
- **Data:** A guard query counts transactions per category before any delete path executes.
- **Logic notes:** Because the taxonomy is fixed (item 121), canonical categories cannot be deleted at all. If custom categories are ever introduced, deletion is blocked while the transaction count > 0 and the UI offers "reassign all to…" first; only at count 0 may deletion proceed.
- **Acceptance:** (1) No UI path deletes a canonical category. (2) Any future custom category with transactions cannot be deleted until reassigned. (3) Reassignment moves every affected transaction before delete is allowed.

### 132. Display category totals as horizontal bars ranked from highest to lowest spend.

- **Components:** Categories screen (and optional Overview widget) horizontal bar list: each row shows icon, display_label, amount, and a bar whose width = value / max value.
- **Data:** Reuses the MTD spend aggregation; Income shown in a separate section or excluded via a spend/income toggle.
- **Logic notes:** Sorted by ABS(spend) descending. Bars use category colors; text labels carry exact amounts so color is never the only indicator.
- **Acceptance:** (1) Bars render strictly highest-to-lowest. (2) Displayed values match the MTD aggregation. (3) Tapping a bar opens the category detail view.

### 133. Add per-category average monthly spend computed from manual history.

- **Components:** "Avg/mo" column on the Categories screen; stat on the category detail view shown as "$X/mo · based on N months".
- **Data:** Query sums spend per category per month over the available history (default: since first transaction, capped at 24 months); average = total ÷ months in range.
- **Logic notes:** Months with zero spend for a category count in the divisor (honest average); the month count is always displayed. Income excluded. Sample toggle respected.
- **Acceptance:** (1) Each category shows an average with its month count. (2) Categories with no history show "—", not $0. (3) Toggling sample data recomputes the averages.

### 134. Let users set a custom sort order for categories in dropdowns.

- **Components:** Drag handles (or up/down arrows on iPhone) on the Categories screen rows; Save order button.
- **Data:** `categories.sort_order`; on save, renumber 1–7. Every category dropdown queries `ORDER BY sort_order`.
- **Logic notes:** Default order: Housing, Food, Transport, Income, Fun, Bills, Other. Order applies to pickers, the management screen, and chart legends.
- **Acceptance:** (1) Reordering updates every category dropdown on refresh. (2) The order persists across sessions. (3) Charts and legends follow the same order.

### 135. Show the category breakdown as a donut chart on the overview.

- **Components:** Overview Plotly donut `donutCategories`: labels = display_label, values = MTD spend, colors from the categories table; "manual data only" caption beneath.
- **Data:** MTD spend aggregation (spending categories; Income excluded, with an optional spend/income toggle).
- **Logic notes:** Slice click sets the Transactions category filter and navigates there. Sample toggle respected; empty state when no MTD spend exists.
- **Acceptance:** (1) Slice values sum to the MTD spend total. (2) Slice colors match the category color scheme. (3) Clicking a slice opens the filtered transaction list.

### 136. Add category-level notes (e.g., "Fun includes streaming subs").

- **Components:** TextArea `categoryNotesInput` in the Categories edit drawer; note icon on category rows and the detail view header showing the note on hover/tap.
- **Data:** `categories.notes` (freeform text, nullable).
- **Logic notes:** Notes are informational only — they never affect totals or stored category values. Included in the categories CSV export.
- **Acceptance:** (1) Notes save and persist per category. (2) Notes render on the category detail view. (3) Notes appear in the categories export.

### 137. Highlight the category with the largest month-over-month increase.

- **Components:** Overview insight tile `biggestMoverTile`: "{icon} {label} up {pct}% vs last month".
- **Data:** Query compares last complete month vs the prior month per spending category: pct = (this − prev) / ABS(prev).
- **Logic notes:** Only categories with prev-month spend > $0 qualify; Income excluded. No tile renders with fewer than two months of history. Ties broken by larger absolute increase.
- **Acceptance:** (1) The tile names the correct category and percentage. (2) It recomputes on month rollover. (3) It stays hidden until two months of data exist.

### 138. Support marking a category as "tracked closely" to pin it to the overview.

- **Components:** Toggle `trackedCloselyToggle` per row on the Categories screen; Overview "Watched" strip pinned near the top showing a compact card per tracked category (MTD vs budget mini-bar).
- **Data:** `categories.tracked_closely` (bool, default false).
- **Logic notes:** Tapping a watched card opens the category detail. Toggling updates the overview immediately without reload.
- **Acceptance:** (1) Toggling a category adds/removes its overview card at once. (2) Watched cards link to the category detail view. (3) The selection persists across sessions.

### 139. Add a yearly category summary table with 12 monthly columns.

- **Components:** Categories screen "Yearly" tab: Table with rows = categories, columns = Jan…Dec + Total; Year select defaulting to the current year.
- **Data:** Pivot query of monthly spend totals per category for the selected year (manual transactions only).
- **Logic notes:** Cells link to the Transactions screen filtered to that category and month. Year options run from the first transaction year to current. Income row shown separately.
- **Acceptance:** (1) Twelve monthly columns render with correct per-cell totals. (2) Row totals equal the sum of the twelve months. (3) The year picker covers every year with data.

### 140. Log category renames and reassignments in the audit log.

- **Components:** No new UI — all writes go to `audit_log` from the rename flow (item 124), the recategorize tool (item 130), and any transaction edit that changes category.
- **Data:** `audit_log` rows: rename → action "category_rename", before/after = display labels; reassignment → action "category_reassign", note carries keyword or source ("manual edit", "bulk tool") and affected count.
- **Logic notes:** The log is append-only: no edit or delete affordance on entries. Bulk operations write one row with the affected-row count, not one row per transaction.
- **Acceptance:** (1) A display-label rename produces a row with before/after labels. (2) The recategorize tool produces a row with keyword and count. (3) No UI path exists to alter or remove audit entries.
