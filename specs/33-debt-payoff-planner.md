# FinanceOS — Section 33: Debt payoff planner (641–660)

Section title: Debt payoff planner.

## Tables touched

- `accounts` (existing): debts can link to accounts of type Credit via `account_id`; manual balance updates stay the source of truth.
- `transactions` (existing): manual debt payments recorded as spend transactions linked to the payoff plan (note field carries debt name).
- `debts` (NEW): `id` (pk), `account_id` (fk, nullable), `name` (text), `balance` (number), `original_balance` (number), `apr` (number, user-entered), `min_payment` (number), `promo_apr` (number, nullable), `promo_expiry` (date, nullable), `status` (text: Active / Paid off), `paid_off_date` (date, nullable), `strategy_used` (text, nullable), `months_taken` (number, nullable), `is_sample` (bool), `created_at` (timestamp)
- `payoff_plans` (NEW): `id` (pk), `name` (text), `strategy` (text: Avalanche / Snowball / Custom), `order_json` (text, JSON array of debt ids), `extra_payment` (number), `created_at` (timestamp), `is_sample` (bool)

## Cross-cutting rules (apply to every item below)

1. Every screen shows the persistent banner: "Manual tracking — no bank connection yet."
2. All debts, plans, and balances are user-entered; sample rows carry `is_sample=true`, show a SAMPLE badge, and are excluded from totals when the sample toggle hides them.
3. Nothing ever presents as a live bank feed. Payoff math is illustrative, not financial advice; APR fields are labeled as user-provided inputs, not bank-verified terms.
4. Estimates (interest accrual, payoff dates) are always labeled "estimate".

---

### 641. Mark accounts of type Credit as debts with balance, APR, and minimum payment fields.

**Components:** Debt list (List View) on a "Debts" tab with "Add debt" Button. Form: Text Input (name), Number Input (balance), Number Input (APR %), Number Input (minimum payment), Select (link to Credit account, optional), linked Account select from `accounts` where type=Credit. **Data:** Retool Database queries `getDebts`, `addDebt` (INSERT into `debts`), `updateDebt`. **Logic notes:** Balance/APR/min-payment are manual fields, not synced from accounts; linking is informational. Debt records require balance ≥ 0, APR ≥ 0, min payment > 0. **Acceptance:** (1) New debt saves with all fields and appears in list. (2) Invalid values (negative balance) are rejected with inline error. (3) Linked account pre-fills name and balance. (4) Sample debts show SAMPLE badge and vanish from totals when the toggle hides samples.

### 642. Build a debt list sorted by balance, APR, or payoff date.

**Components:** Debt list (Table component) with Sort control (Select: Balance / APR / Payoff date) and direction toggle. Row shows name, balance, APR, min payment, computed payoff date. **Data:** `getDebts` query; JS transformer `sortDebts(debts, key, dir)`; payoff date computed per-debt by the amortization transformer shared with item 645. **Logic notes:** Payoff-date sort uses the active plan's extra-payment assumption; debts with no plan fall back to minimums-only payoff. Nulls sort last. **Acceptance:** (1) Changing the sort key reorders the table immediately. (2) Payoff-date sort puts the soonest-free debt first. (3) Sort persists per session via temporary state. (4) Sample toggle does not change sort correctness for real debts.

### 643. Implement avalanche simulation: extra payments to highest APR first.

**Components:** Strategy Select (Avalanche / Snowball / Custom) on the plan screen; results panel showing per-debt payoff order and totals. **Data:** JS transformer `simulateDebts(debts, extra, order)` — allocate minimums to all active debts, route all extra to the highest-APR unpaid debt each month; iterate month-by-month until all balances are zero. **Logic notes:** Promotional APR periods (item 653) use `promo_apr` before `promo_expiry`. Edge cases: zero extra payment → minimums-only; APR ties → break by smaller balance. **Acceptance:** (1) Extra payment always flows to the highest-APR active debt first. (2) Totals (months, interest) match a manual spot-check on 3 debts. (3) Changing extra payment re-runs the simulation. (4) Results are labeled estimates, never advice.

### 644. Implement snowball simulation: extra payments to smallest balance first.

**Components:** Same plan screen; Strategy Select set to Snowball; results panel identical in shape to avalanche output for direct comparison. **Data:** Reuse `simulateDebts(debts, extra, order)` with `order` sorted by ascending balance. **Logic notes:** Extra goes to the smallest-balance unpaid debt each month; minimums to all others. Balance ties → break by higher APR. Paid-off debts drop out of the order. **Acceptance:** (1) Extra payment always flows to the smallest-balance active debt first. (2) Payoff order differs from avalanche on the sample set (sample includes a low-balance/high-APR case). (3) Totals update when extra payment changes. (4) Output labeled estimate.

### 645. Show payoff date and total interest for each strategy from user-entered numbers.

**Components:** Results cards: Payoff date (Text), Total interest paid (Text, "estimate" caption), Months (Text), per-debt breakdown Table (debt, payoff month, interest share). **Data:** `simulateDebts` returns `{ payoffDate, totalInterest, months, perDebt: [...] }` from the user's entered balances/APRs/minimums. **Logic notes:** Monthly rate = APR/12; interest accrues before payment each simulated month; final partial month prorated. If minimums don't cover interest, flag "will never pay off" instead of looping forever (cap at 1,200 months). **Acceptance:** (1) Each strategy shows a payoff date and total interest. (2) Numbers derive only from user-entered fields. (3) Insufficient-minimum case shows a warning, not a hang. (4) All figures carry the "estimate" label.

### 646. Allow a custom payoff order via drag-and-drop.

**Components:** Draggable list (Retool List View with drag handle or a custom HTML drag-drop component) of active debts; "Save as custom plan" Button. **Data:** `saveCustomPlan` writes `payoff_plans` row with strategy=Custom and `order_json` array of debt ids; `simulateDebts` consumes that order. **Logic notes:** Drag reorders a local state array; unsaved changes show a "not saved" pill. Removing a paid-off debt keeps the saved order valid (missing ids skipped). **Acceptance:** (1) Dragging reorders and the simulation follows the new order. (2) Saved custom order persists across reloads. (3) Missing/paid debts in the order don't break the simulation. (4) Custom plans are labeled with their name and date.

### 647. Record manual debt payments as transactions linked to the payoff plan.

**Components:** "Log payment" Button on each debt row opening a Form: amount, date, optional note; linked Transaction created in `transactions`. **Data:** INSERT into `transactions` with `account_id` = linked credit account (or manual debt pseudo-account), `amount` negative, `category` = Bills, description = "Debt payment — {debt name}"; debt balance decremented by the same amount in one action block. **Logic notes:** Wrap in a single JS query run so balance and transaction stay in sync; if either fails, roll back with an error toast. Payments below the minimum show a soft warning, not a block. **Acceptance:** (1) Logged payment reduces debt balance and appears in the transaction list. (2) Payment note links back to the debt (searchable). (3) Failed insert leaves balance unchanged. (4) Sample toggle hides sample payments and their balance effects.

### 648. Display a payoff progress bar per debt with remaining balance.

**Components:** Progress bar component per debt row: percent paid = (original − current) / original; Text showing remaining balance and "$X of $Y paid". **Data:** Derived in `getDebts` transformer; `original_balance` set at creation, editable via "adjust starting balance" with a confirm. **Logic notes:** Progress clamps 0–100%; overpayments (balance < 0) clamp at 100% and flag for review. Paid-off debts render a full green bar with "Paid off" label. **Acceptance:** (1) Bar percent matches (original − balance) / original on spot check. (2) Remaining balance text matches the debt balance. (3) Overpayment flags rather than showing >100%. (4) Bar updates immediately after logging a payment.

### 649. Simulate "what if I pay $X extra" with updated payoff dates.

**Components:** "Extra payment" Number Input (slider + input, $0–$2,000) on the plan screen; live-updating payoff date and interest totals for the selected strategy. **Data:** Debounced onChange re-runs `simulateDebts(debts, extra, order)`; results panel shows delta vs current plan ("3 months sooner, $412 less interest"). **Logic notes:** $0 extra = minimums-only baseline shown for reference. Deltas computed against the same debt set; guard against absurd values (> $100k/month) with a sanity warning. **Acceptance:** (1) Typing $X updates payoff date and totals without saving anything. (2) Delta line shows months saved and interest saved vs baseline. (3) $0 renders the minimums-only baseline. (4) Nothing persists until the user saves the plan.

### 650. Show monthly interest accrual estimates labeled as estimates.

**Components:** Per-debt "Interest this month" Text with "estimate" caption; plan-level "Monthly interest across debts" summary; tooltip explaining the math. **Data:** `balance × APR/12` (or `promo_apr` inside a promo window); transformer `monthlyInterest(debts)`. **Logic notes:** Uses the effective APR (promo vs standard) for the current month; rounds to cents; label "estimate" on every instance including tooltips and exports. Zero-balance or paid-off debts show $0. **Acceptance:** (1) Monthly interest matches balance × APR/12 on spot check. (2) Promo-window debts use the promo rate. (3) Every figure carries an "estimate" label. (4) Total matches the sum of per-debt values.

### 651. Add a debt-free date headline across all debts.

**Components:** Hero stat Text on the Debts screen: "Debt-free by {month year}" with "estimate" caption and the active plan name beneath. **Data:** `simulateDebts` full-portfolio run under the active plan's strategy + extra payment; headline = max payoff date. **Logic notes:** If any debt never pays off (minimums < interest), headline shows "Not reachable with current minimums" instead of a date. With no debts, shows a neutral "No active debts" state. **Acceptance:** (1) Headline date equals the latest per-debt payoff date in the simulation. (2) Unreachable case shows the warning text, not a date. (3) Empty state reads "No active debts". (4) Caption labels it an estimate from manual inputs.

### 652. Compare avalanche vs snowball side by side with interest difference.

**Components:** Two-column comparison panel (Statistic components): payoff date, total interest, months for each strategy; center "difference" strip showing interest saved by the cheaper strategy. **Data:** Run `simulateDebts` twice (APR-desc order, balance-asc order) with the same extra payment; transformer `compareStrategies` returns both + deltas. **Logic notes:** Same inputs guaranteed by cloning the debt array before each run; winner highlighted, loser shown for context. If both are equal, show "Identical for your debts". **Acceptance:** (1) Both columns derive from identical debt inputs. (2) Difference row shows the exact interest delta. (3) Winner is labeled without prescribing a choice (illustrative only). (4) Side-by-side updates when the extra payment changes.

### 653. Support promotional 0% APR periods with expiry dates.

**Components:** Debt form fields: Promo APR Number Input, Promo expiry Date Picker; debt list shows a "Promo 0% until {date}" pill; simulation log shows the rate switch. **Data:** `promo_apr` and `promo_expiry` columns on `debts`; `effectiveApr(debt, month)` returns promo rate while month ≤ expiry, else standard `apr`. **Logic notes:** Expired promos fall back automatically; entering a past expiry warns and treats as expired. A debt with promo_apr=0 pays no interest until expiry. **Acceptance:** (1) Simulation charges 0% interest during the promo window. (2) Post-expiry months use the standard APR. (3) Expired-promo entry shows a warning. (4) Promo pill disappears after expiry.

### 654. Track minimum payments as bills linked to each debt.

**Components:** "Create bill" action on each debt; Bills view shows debt minimums alongside other bills; each bill row links back to the debt. **Data:** Bills stored as `transactions` scheduled entries or a lightweight `bills` concept reusing the existing bills pattern from section 15: fields debt_id, amount = min_payment, due day. Toggling "minimum paid" logs the payment via item 647's flow. **Logic notes:** Bill amount auto-updates when min_payment changes; deleting the debt archives (not deletes) its bills. Paid-off debts drop their bills automatically. **Acceptance:** (1) Each active debt with a minimum shows a matching bill. (2) Editing min_payment updates the bill amount. (3) Paid-off debt removes its bill. (4) Marking a bill paid logs the payment transaction.

### 655. Show total monthly debt payments vs recorded income.

**Components:** Comparison card: "Debt payments $X / mo vs income $Y / mo" with a ratio bar and a 36% guideline marker (labeled informational, not advice). **Data:** Debt total = sum of min payments + planned extra from the active plan; income = sum of Income-category transactions in the last 3 months averaged. **Logic notes:** Missing income data shows "add income transactions to see this" instead of a misleading ratio; sample toggle applies to both sides. **Acceptance:** (1) Debt total equals min payments + active plan extra. (2) Income is a 3-month average of recorded Income transactions. (3) No income data shows the empty-state prompt. (4) Guideline marker is labeled informational.

### 656. Celebrate each paid-off debt with a manual "mark paid off" action.

**Components:** "Mark paid off" Button on each debt (with confirm modal); confetti/success state on the debt row; plan recalculates automatically. **Data:** UPDATE `debts` set status='Paid off', paid_off_date=today, strategy_used=active plan strategy, months_taken=months since created_at. **Logic notes:** Manual action only — never auto-triggers from balance alone (user may carry a small residual). Confirm modal prevents accidental taps. Paid-off debt leaves the active list and appears in history. **Acceptance:** (1) Confirm modal appears before the action. (2) Debt moves from active list to history with date and strategy recorded. (3) Plan simulation re-runs excluding it. (4) Accidental-click undo available via "reopen" for 24h.

### 657. Keep paid-off debts in history with their payoff story (strategy used, months taken).

**Components:** "Paid-off history" section/table: debt name, original balance, strategy used, months taken, paid-off date, total interest (estimated). **Data:** `debts` rows with status='Paid off'; history view queries them sorted by paid_off_date desc; "payoff story" auto-text generated from fields. **Logic notes:** History is append-only — no edits to paid-off rows except reopen (item 656). Interest total estimated from the final simulation snapshot stored at payoff time. **Acceptance:** (1) Every marked-paid-off debt appears in history. (2) Story line shows strategy + months taken + date. (3) Paid-off rows are read-only except reopen. (4) Sample toggle hides sample history rows.

### 658. Export the payoff plan to PDF with the manual-data banner.

**Components:** "Export PDF" Button on the plan screen; PDF includes debt table, strategy, payoff date, interest estimate, and the banner. **Data:** Retool PDF export of the plan view (print-optimized layout); header block renders "Manual tracking — no bank connection yet." plus "Figures are user-entered estimates." and export date. **Logic notes:** Export reflects the currently displayed plan/strategy; sample rows watermarked SAMPLE in the PDF. Uses the same export pattern as other FinanceOS PDF exports for consistency. **Acceptance:** (1) PDF contains the banner text and the export date. (2) PDF matches the on-screen plan figures. (3) Sample rows carry the SAMPLE watermark. (4) Export requires no external service.

### 659. Warn when a planned extra payment exceeds available checking balance.

**Components:** Warning Callout on the plan screen when extra payment > checking balance; Text showing available balance and the shortfall. **Data:** Available balance = sum of `accounts` balances where type=Checking and is_sample matches toggle; compares against planned extra + minimums. **Logic notes:** Warning is advisory — the user can still save the plan. Re-checks after account balance edits. Checking accounts with sample data only trigger the warning against the sample total, clearly labeled. **Acceptance:** (1) Warning appears when planned payments exceed checking total. (2) Shows both the planned amount and available balance. (3) Warning clears when balances cover the plan. (4) Saving the plan is still allowed with the warning visible.

### 660. Label all APR math as user-provided inputs, not bank-verified terms.

**Components:** "User-entered · not bank-verified" caption under every APR field, simulation result, and export; info tooltip on the plan screen restating it. **Data:** No data change — a presentation rule enforced by a shared `AprDisclaimer` text block component reused everywhere APR appears. **Logic notes:** The rule covers forms, results, comparisons, history stories, and PDF exports. Any new APR surface must include the component — enforced by including it in the shared plan layout template. **Acceptance:** (1) Every APR input shows the caption. (2) Every simulation output shows the caption. (3) PDF export includes the disclaimer. (4) No screen presents APR terms as verified or sourced from a bank.
