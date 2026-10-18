# FinanceOS — Section 28: Alerts & notifications (items 541–560)

Builder spec for the FinanceOS alerting system. All alerts evaluate the user's own manually-entered data only — there are no external triggers, no bank feeds, and no bank connections of any kind. Alerts surface in an in-app notification inbox; push notifications are an explicit user opt-in (off by default). Evaluation runs on a Retool Workflow schedule and on data-change events.

## Tables touched

Existing (per specs.md App 11): `accounts` (id, name, type, institution, balance, updated_at, is_sample), `transactions` (id, account_id, date, description, category, amount, is_sample), `budgets` (id, category, month, limit_amount), `goals` (id, goal_name, target, saved, deadline).

Also referenced from earlier sections: `bills` (section 15 — id, name, amount, due_date, paid, account_id, is_sample), `recurring_templates` (section 17 — id, description, category, amount, frequency, next_date), `net_worth_snapshots` (section 2 — id, snapshot_date, net_worth, is_sample).

New tables:

- **alerts**: id (pk), alert_type (text: budget_80 / budget_100 / bill_due / bill_overdue / balance_stale / goal_milestone / large_txn / spend_over_income / recurring_due / goal_deadline / networth_drop / sample_stale), title (text), body (text), created_at (timestamp), read_at (timestamp, nullable), dismissed (bool, default false), trigger_value (text — JSON snapshot of the values that fired the alert, e.g. `{"spent":412,"limit":500}`), source_label (text, always "FinanceOS manual tracking"), fired_key (text, unique per rule+period, prevents duplicate firing)
- **alert_preferences**: id (pk), alert_type (text, unique), enabled (bool, default true), threshold_value (number, nullable — e.g. days-before-bill, stale-days, large-txn multiplier, net-worth drop %), updated_at (timestamp)
- **app_settings**: id (pk), setting_key (text, unique), setting_value (text) — keys: quiet_hours_start, quiet_hours_end, digest_enabled, digest_time, push_opt_in

## Shared cross-cutting rules

- Every screen shows the persistent banner "Manual tracking — no bank connection yet."
- Sample rows carry `is_sample=true` + SAMPLE badge and are excluded from totals when the sample toggle hides them. Alert evaluations run against the same filtered totals the user currently sees, so hiding samples also suppresses sample-driven alerts.
- Nothing ever presents as a live bank feed. Every alert carries `source_label = "FinanceOS manual tracking"` and every alert body references the user's entered data.

---

### 541. Send an in-app alert when a category crosses 80% of its monthly budget.

**Components:** Budget vs Actual screen gains a Retool Workflow `eval_budget_alerts` (scheduled hourly); no new visual components — alerts appear in the notification center (item 549).

**Data:** Query `monthly_spend_by_category` sums `transactions.amount` (abs, spend only) for the current month per category, joined against `budgets` for the same `month`. Transformer computes `pct = spent / limit_amount`.

**Logic notes:** Fire when `pct >= 0.8` and no `alerts` row exists with `fired_key = 'budget_80:' + category + ':' + month`. The `fired_key` guarantees one 80% alert per category per month even if spend fluctuates. Sample toggle: evaluation uses the visible (possibly sample-excluded) totals. Zero-limit budgets are skipped, not alerted.

**Acceptance:** (1) Entering a transaction that pushes a category to ≥80% of its budget produces exactly one unread alert of type `budget_80` within one evaluation cycle. (2) Adding further transactions in the same month produces no second alert. (3) Alert `trigger_value` records both spent and limit. (4) No alert fires when samples are hidden and the 80% crossing was driven only by sample rows.

### 542. Send an in-app alert when a category exceeds 100% of budget.

**Components:** Same `eval_budget_alerts` Workflow, extended with a second threshold check; notification center renders `budget_100` alerts with a red severity badge.

**Data:** Reuses `monthly_spend_by_category` transformer from 541. Second pass fires on `pct > 1.0`.

**Logic notes:** Separate `fired_key` namespace (`budget_100:...`) so a category can fire both the 80% and the 100% alert in one month. Overspend alert body includes the overage amount (`spent − limit`) formatted as currency. Edge case: budget limit edited downward mid-month re-arms evaluation; if the new limit is already exceeded, the 100% alert fires on the next cycle (new `fired_key` includes limit value to allow re-fire on limit change).

**Acceptance:** (1) Crossing 100% produces exactly one `budget_100` alert distinct from any earlier `budget_80` alert for the same category/month. (2) Body shows the overage dollar amount. (3) Lowering a budget limit below current spend triggers the alert within one cycle. (4) Dismissing the alert does not re-fire it the same month.

### 543. Notify when a bill is due in 3 days and unpaid.

**Components:** Workflow `eval_bill_alerts` (runs daily, morning); notification center entry with a "Mark paid" quick action button.

**Data:** Query `upcoming_bills`: `SELECT * FROM bills WHERE paid = false AND due_date = CURRENT_DATE + make_interval(days => threshold)` where threshold reads `alert_preferences.threshold_value` for `bill_due` (default 3).

**Logic notes:** `fired_key = 'bill_due:' + bill_id` prevents repeat notifications; if the bill is edited (due date moved), the key includes due_date so it re-arms. The quick action sets `paid = true` and marks the alert read. Bills flagged `is_sample` carry the SAMPLE badge in the alert body.

**Acceptance:** (1) An unpaid bill due exactly 3 days out produces one `bill_due` alert after the daily run. (2) Marking the bill paid from the alert clears it (no duplicate remains unread). (3) Changing the bill's due date re-arms the alert for the new date. (4) Disabling the `bill_due` toggle stops all such alerts.

### 544. Notify when a bill becomes overdue.

**Components:** Same `eval_bill_alerts` Workflow; overdue alerts render with red severity and show days-overdue count.

**Data:** Query `overdue_bills`: `SELECT *, CURRENT_DATE - due_date AS days_overdue FROM bills WHERE paid = false AND due_date < CURRENT_DATE`.

**Logic notes:** Fire once per bill per overdue spell (`fired_key = 'bill_overdue:' + bill_id`). If a bill remains unpaid, a weekly re-reminder is allowed: the fired key includes the week number so at most one reminder per week fires. Days-overdue in the body is computed at fire time; the notification-center detail view recomputes it live from `due_date` so it stays current.

**Acceptance:** (1) A bill unpaid past its due date produces one `bill_overdue` alert on the next daily run. (2) No second alert fires the next day for the same bill. (3) A re-reminder fires at most once per calendar week while still unpaid. (4) Paying the bill stops further reminders and marks existing alerts read.

### 545. Alert when an account balance hasn't been manually updated in 30 days.

**Components:** Workflow `eval_staleness` (weekly); notification center entry with "Update balance" button deep-linking to the Accounts screen.

**Data:** Query `stale_accounts`: `SELECT * FROM accounts WHERE updated_at < NOW() - make_interval(days => threshold)` with threshold from `alert_preferences` (`balance_stale`, default 30).

**Logic notes:** Fire once per account per stale spell (`fired_key = 'balance_stale:' + account_id + ':' + date_trunc('week')` — one reminder per week max). Updating the balance resets the spell. Body text reminds the user the balance is manual: "You last updated this balance 34 days ago. Based on your entered data — no bank sync." Archived/closed accounts (if a status column exists) are excluded.

**Acceptance:** (1) An account untouched for 30+ days yields one `balance_stale` alert per weekly run. (2) Updating the balance clears the spell — no alert the following week. (3) The body explicitly states the balance is manually maintained. (4) Threshold is user-editable in notification settings and takes effect on the next run.

### 546. Notify when a goal reaches 50%, 75%, and 100% of target.

**Components:** Trigger on the Goals screen "Contribute" action plus Workflow `eval_goal_milestones` (runs after any contribution); milestone alerts use a celebratory (green) style.

**Data:** Query `goal_progress`: `saved / NULLIF(target, 0) AS pct` per goal. Transformer compares against milestones [0.5, 0.75, 1.0].

**Logic notes:** `fired_key = 'goal_milestone:' + goal_id + ':' + milestone` so each milestone fires once per goal; if contributions are later withdrawn below a milestone and re-cross, a new key with an incrementing `crossing_count` allows re-fire. 100% milestone body suggests setting a new goal. Goals with `target = 0` or null are skipped.

**Acceptance:** (1) A contribution crossing 50% fires exactly one `goal_milestone` alert. (2) Crossing 75% and 100% each fire their own single alert. (3) Withdrawing savings below a milestone and re-crossing fires again (new crossing). (4) Alert body shows saved, target, and percent.

### 547. Alert on unusually large transactions (over 3× the category's daily average).

**Components:** Event handler on the Add Transaction form's submit query (`onSuccess` → `check_large_txn`); alert appears in the inbox immediately, no schedule needed.

**Data:** Query `category_daily_avg`: average of daily absolute spend per category over the trailing 30 days (`SUM(amount)/30` on spend rows). Compare `NEW.amount > 3 * avg * 1.0`.

**Logic notes:** Multiplier (3) is user-editable via `alert_preferences` (`large_txn`). Requires at least 7 days of category history; with less history the check is skipped to avoid false positives. Income-category transactions are excluded. The alert is advisory ("unusually large for this category") and never blocks the save — it fires after insert.

**Acceptance:** (1) Saving a transaction above 3× the category's 30-day daily average fires one `large_txn` alert referencing the average. (2) The transaction itself is still saved (alert is non-blocking). (3) With fewer than 7 days of history, no alert fires. (4) Changing the multiplier to 5× changes firing behavior accordingly.

### 548. Notify when monthly spend exceeds monthly recorded income.

**Components:** Workflow `eval_spend_vs_income` (runs after each transaction insert via the form's `onSuccess`, plus nightly); notification center entry with a link to the Spending Trends screen.

**Data:** Query `month_totals`: `SUM(CASE WHEN amount < 0 THEN -amount ELSE 0 END) AS spend, SUM(CASE WHEN amount > 0 THEN amount ELSE 0 END) AS income` for the current month.

**Logic notes:** Fire once per month (`fired_key = 'spend_over_income:' + month`). If spend later drops below income and crosses again, allow one re-fire per month. Body states the figures: "Your entered spending ($2,140) is above your recorded income ($1,900) for October." Uses the word "recorded" deliberately — income is manual entry, never assumed complete.

**Acceptance:** (1) A transaction pushing monthly spend above monthly recorded income fires one alert per month. (2) Body shows both dollar figures with the month name. (3) No alert fires when income exceeds spend. (4) Wording says "recorded income", never "your income" as if complete.

### 549. Add a notification center listing all alerts with timestamps and read states.

**Components:** New screen/modal "Notifications": Retool `Table` (or `ListView`) bound to alerts; columns: severity badge, title, body preview, `created_at` (relative time), read/unread dot, actions (Mark read, Dismiss). Header button with unread-count `Badge` visible on every screen. `Drawer` for full alert detail including trigger values.

**Data:** Query `list_alerts`: `SELECT * FROM alerts WHERE dismissed = false ORDER BY read_at NULLS FIRST, created_at DESC`. Mutations `mark_alert_read` (sets `read_at = NOW()`), `dismiss_alert`, `mark_all_read`.

**Logic notes:** Unread alerts sort above read ones; dismissed alerts are hidden from the default view but visible under a "Dismissed" filter. The header badge shows `COUNT(*) WHERE read_at IS NULL AND dismissed = false`, refreshing on `list_alerts` success and on a 60-second `Timer` while the app is open. Tapping an alert opens the detail drawer and marks it read.

**Acceptance:** (1) All fired alerts appear with correct timestamps and unread state. (2) Tapping an alert marks it read and opens detail. (3) Dismissed alerts leave the default list but remain findable under the Dismissed filter. (4) The header unread badge count always matches the inbox.

### 550. Support quiet hours so no notifications fire overnight.

**Components:** Notification Settings screen: two `TimePicker` components (start/end, default 22:00–07:00) bound to `app_settings.quiet_hours_start/quiet_hours_end`; helper text explaining behavior.

**Data:** Settings queries `get_setting`/`set_setting` on `app_settings`. Every alert-firing Workflow calls a shared JS function `inQuietHours()` comparing current time to the stored window (handles overnight wrap, e.g. 22:00→07:00).

**Logic notes:** During quiet hours, scheduled evaluations are skipped entirely — no alert rows are created and no pushes sent, so "no notifications fire overnight" holds literally. Pending conditions are picked up on the first run after quiet hours end. In-app inbox alerts created before quiet hours remain visible; only *firing* is suppressed.

**Acceptance:** (1) With quiet hours 22:00–07:00, a condition met at 23:00 produces no alert until after 07:00. (2) The delayed alert fires on the first post-quiet-hours run. (3) The overnight window wraps midnight correctly. (4) Disabling quiet hours (empty values) restores immediate firing.

### 551. Allow per-alert-type toggles in notification settings.

**Components:** Notification Settings screen: a `ListView` of alert types, each row with a `Switch` bound to `alert_preferences.enabled`, the friendly name, a one-line description, and the threshold `NumberInput` where applicable (bill days, stale days, multiplier, net-worth %).

**Data:** Query `list_preferences` (all rows in `alert_preferences`); mutation `save_preference` upserts enabled/threshold. A shared JS guard `alertEnabled(type)` is called at the top of every Workflow evaluation.

**Logic notes:** Seed rows for all 12 alert types on first run (migration script). Toggling off takes effect immediately for future firings; existing unread alerts are left in the inbox (not deleted). Threshold edits validate ranges (e.g. multiplier ≥ 1.5, days ≥ 1) before saving.

**Acceptance:** (1) Every alert type has a working on/off switch. (2) Turning off `bill_overdue` stops new overdue alerts while others continue. (3) Threshold edits persist and change firing behavior. (4) Invalid threshold values are rejected with a message.

### 552. Alert when a recurring template instance is due for confirmation.

**Components:** Workflow `eval_recurring_due` (daily); alert includes "Confirm" and "Skip" buttons that post the instance to transactions or snooze the template.

**Data:** Query `due_templates`: `SELECT * FROM recurring_templates WHERE next_date <= CURRENT_DATE + 2` (2-day confirmation window). `fired_key = 'recurring_due:' + template_id + ':' + next_date`.

**Logic notes:** Confirming creates the transaction row (manual entry, flagged normally) and advances `next_date` by the frequency; skipping advances `next_date` without creating a row and marks the alert read. Body shows description, amount, and scheduled date. If the template is paused, no alert fires.

**Acceptance:** (1) A template with `next_date` within 2 days produces one `recurring_due` alert. (2) Confirm posts a transaction and advances the template; the alert is marked read. (3) Skip advances the template with no transaction. (4) Paused templates never alert.

### 553. Notify when a goal deadline is 30 days away and the goal is under 80%.

**Components:** Workflow `eval_goal_deadlines` (weekly); alert links to the Goals screen with the goal pre-selected.

**Data:** Query `goals_at_risk`: `SELECT *, (deadline - CURRENT_DATE) AS days_left, saved/NULLIF(target,0) AS pct FROM goals WHERE deadline BETWEEN CURRENT_DATE AND CURRENT_DATE + 30 AND saved/NULLIF(target,0) < 0.8`.

**Logic notes:** `fired_key = 'goal_deadline:' + goal_id + ':' + week` — one reminder per week per goal while the condition holds. Body includes days remaining, current percent, and the shortfall amount (`target − saved`). Goals with no deadline are excluded. If the goal reaches 80%, the spell ends.

**Acceptance:** (1) A goal 25 days out at 60% fires one `goal_deadline` alert per weekly run. (2) Body shows days left, percent, and dollar shortfall. (3) Reaching 80% stops further alerts. (4) Goals without deadlines never fire this alert.

### 554. Alert when net worth drops more than a user-set percent between snapshots.

**Components:** Workflow `eval_networth_drop` (runs after each manual snapshot save, via the snapshot form's `onSuccess`); threshold `NumberInput` in notification settings (default 10%).

**Data:** Query `latest_snapshots`: the two most recent rows in `net_worth_snapshots` (respecting the sample toggle). Compute `drop_pct = (prev - latest) / prev`.

**Logic notes:** Fire when `drop_pct > threshold` and `prev > 0`. `fired_key = 'networth_drop:' + latest_snapshot_id` — one alert per snapshot. Sample snapshots are only compared with samples visible. Body frames the drop neutrally against manual data: "Your tracked net worth fell 12% between your Oct 1 and Nov 1 entries, based on your entered balances."

**Acceptance:** (1) Saving a snapshot 12% below the previous one (threshold 10%) fires one alert. (2) A 5% drop fires nothing. (3) Changing the threshold to 15% changes behavior. (4) Body cites both snapshot dates and the manual-data basis.

### 555. Show alert history with the triggering values for auditability.

**Components:** Notification center "History" tab: table of all alerts including dismissed and read, with an expandable detail `Drawer` rendering `trigger_value` JSON as labeled key–value rows (spent/limit, days overdue, drop %, etc.).

**Data:** Query `alert_history`: `SELECT * FROM alerts ORDER BY created_at DESC` (no dismissed filter). A transformer parses `trigger_value` JSON into display rows; unparseable values fall back to raw text.

**Logic notes:** History is append-only — alerts are never hard-deleted, only dismissed, so the audit trail is complete. Each row shows alert type, fired timestamp, read timestamp, the values that triggered it, and the source label. A date-range `DatePicker` filters the history.

**Acceptance:** (1) Every fired alert appears in History with its triggering values readable. (2) Dismissed alerts remain in History. (3) Trigger values match the figures at fire time even if the underlying data later changed. (4) No alert can be permanently deleted from the UI.

### 556. Make all alerts reference manual data ("based on your entered transactions").

**Components:** A shared JS utility `alertBody(title, detail, dataRef)` used by every Workflow; a lint-style checklist in this spec's acceptance criteria enforced at build time.

**Data:** No new query. The utility appends a `dataRef` string such as "Based on your entered transactions." or "Based on the balances you entered." depending on the alert type.

**Logic notes:** The reference is part of the stored `body` text, not a UI overlay, so it survives into history, digests, and push payloads. Copy variants: transaction-driven alerts → "based on your entered transactions"; balance/staleness alerts → "based on the balances you entered"; goal alerts → "based on your saved goal entries". No alert may imply external or automatic data.

**Acceptance:** (1) Every alert body in the inbox contains one of the approved manual-data phrases. (2) The phrase is in the stored body, visible in History and digest. (3) No alert body contains the words "bank", "synced", or "automatically detected". (4) A spot check of all 12 alert types passes.

### 557. Add push-notification opt-in for iPhone (user-controlled, off by default).

**Components:** Notification Settings screen: a `Switch` labeled "Push notifications on this iPhone" (default OFF) with explanatory text; Retool Mobile push action in the alert dispatcher Workflow.

**Data:** `app_settings.push_opt_in` ('true'/'false', default 'false'). Dispatcher checks opt-in AND quiet hours AND per-type toggle before sending push; the in-app alert row is always created regardless.

**Logic notes:** Push is a convenience copy of the in-app alert — the inbox is the source of truth. Push payload title/body mirror the alert and include the source label. If the user revokes iOS notification permission, pushes silently fail and the inbox remains complete; a settings hint explains this. Push never fires for alerts created during quiet hours.

**Acceptance:** (1) Default state is OFF and no push is sent until the user enables it. (2) Enabling opt-in delivers pushes for new alerts (device permission granted). (3) Disabling stops pushes immediately; in-app alerts continue. (4) Push body includes the FinanceOS manual-tracking source label.

### 558. Bundle multiple alerts into a daily digest option.

**Components:** Notification Settings: `Switch` for digest mode + `TimePicker` for digest time (default 08:00); notification center shows the digest as a single expandable alert row.

**Data:** `app_settings.digest_enabled`, `app_settings.digest_time`. Workflow `send_digest` runs at the digest time: collects unfired-eligible alerts since the last digest, creates one `alerts` row with `alert_type` of the bundle (stored as individual rows too, marked `digested = true`... simpler: digest row references child ids in `trigger_value` JSON), then marks children read.

**Logic notes:** In digest mode, individual alerts are still written to `alerts` (audit trail intact) but pushes are suppressed; only the digest push fires. High-severity alerts (`bill_overdue`, `budget_100`) can optionally bypass the digest via a per-type "always immediate" checkbox — default off. If zero alerts qualify, no digest is sent.

**Acceptance:** (1) With digest on, three overnight-eligible alerts produce one digest row and one push at the set time. (2) Individual alert rows still exist in History. (3) No digest is sent on alert-free days. (4) Marking a digest read marks its children read.

### 559. Alert when sample data is still present after 30 days, suggesting cleanup.

**Components:** Workflow `eval_sample_stale` (weekly); alert includes a "Review sample data" button opening the sample-data management screen (section 18) and a "Keep samples" snooze action.

**Data:** Query `sample_age`: `SELECT MIN(created_at) ... ` — checks for any rows with `is_sample = true` across accounts/transactions/goals; age measured from the oldest sample row's creation (or app install date fallback).

**Logic notes:** `fired_key = 'sample_stale:' + month` — at most one per month. The "Keep samples" action snoozes for 30 days (writes a `sample_snooze_until` app setting). Body is helpful, not scolding: "Sample data is still in your workspace after 30 days. Your totals currently mix sample and real entries — review or clear samples when ready."

**Acceptance:** (1) Sample rows older than 30 days produce one `sample_stale` alert per monthly run. (2) Clearing all sample data stops the alerts. (3) "Keep samples" snoozes for 30 days. (4) Body explains that totals mix sample and real entries.

### 560. Never present an alert as coming from a bank; label the source as FinanceOS manual tracking.

**Components:** Applies to every alert surface: notification center rows (source caption under each title), alert detail drawer, push payloads, digest rows, and the settings screen explainer text.

**Data:** The `alerts.source_label` column (always "FinanceOS manual tracking") is rendered in all alert templates; a shared `AlertCard` pattern (documented component recipe) includes the caption slot.

**Logic notes:** This is a copy and rendering rule enforced by the shared `alertBody` utility (item 556) plus template review: no alert title, body, push text, or digest may name a bank, use "your bank", or imply institutional origin. Institution names stored on `accounts` (e.g. "Chase") are never surfaced inside alert copy. Acceptance includes a text scan of all alert templates.

**Acceptance:** (1) Every rendered alert shows the "FinanceOS manual tracking" source caption. (2) A scan of all 12 alert-type templates finds no bank names and no phrase "your bank". (3) Account institution names never appear in alert titles or bodies. (4) Push payloads carry the same source label.
