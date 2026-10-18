# FinanceOS — Section 29: Reminders & scheduled nudges (items 561–580)

Builder-ready spec for the FinanceOS Retool app. Each item lists exact Components, Data, Logic notes, and Acceptance criteria.

## Tables touched

- **accounts**, **transactions**, **budgets**, **goals** — as defined in specs.md App 11 (all carry `is_sample`).
- **reminders** (NEW): id (pk), title (text), body (text), kind (text: weekly_balance / month_review / recurring_digest / inactivity / goal_deadline / payday_checklist / bill_week / networth_snapshot / missing_budget / backup_quarterly / custom), schedule_type (text: weekly / monthly / monthly_date / custom_date / event), schedule_day (number, nullable: 0–6 for weekly weekday, 1–31 for monthly), schedule_time (text, nullable, HH:MM), scheduled_date (date, nullable), due_date (date), status (text: scheduled / snoozed / done / dismissed), snooze_until (date, nullable), tone (text: neutral / encouraging / direct, nullable), is_sample (bool)
- **reminder_events** (NEW): id (pk), reminder_id (fk), occurred_at (timestamp), action (text: shown / snoozed / completed / dismissed), note (text, nullable)

## Shared cross-cutting rules

1. Every screen shows the persistent banner "Manual tracking — no bank connection yet."
2. Sample rows carry `is_sample=true` plus a SAMPLE badge; when the sample-data toggle hides samples, they are excluded from every total and nudge evaluation.
3. Nothing ever presents as a live bank feed; reminder copy never implies synced or automatic data.
4. Reminders are user-created and user-dismissed only. They prompt action; they never auto-post transactions, change balances, or send email/SMS/push. No automated outreach of any kind — a reminder is rendered inside the app on the Overview checklist and the reminders settings screen.

---

### 561. Add a weekly "update your balances" reminder, user-scheduled day and time.

**Components:** reminders settings screen (Form `fRemindWeekly` with Select `dayOfWeek` Mon–Sun, TimeInput `timeOfDay`, Button `saveWeeklyReminder`); Overview checklist section `weeklyBalanceCheck`.

**Data:** query `qUpsertReminder`: INSERT/UPDATE reminders SET kind='weekly_balance', schedule_type='weekly', schedule_day, schedule_time, due_date=next occurrence, status='scheduled'. Transformer `tNextWeeklyOccurrence`: given weekday + HH:MM, compute next future datetime (roll to next week if today's time passed).

**Logic notes:** one active weekly_balance reminder per user (upsert replaces prior). due_date recalculated each morning by app load logic; on completion, next due_date = +7 days. is_sample=false always for user-created reminders.

**Acceptance:** (1) saving with Wed/19:30 sets due_date to the next Wednesday 19:30, (2) changing the day replaces the old row — no duplicates, (3) Overview shows "Update your balances" on/after due_date until marked done, (4) banner and SAMPLE rules hold on the settings screen.

### 562. Add a monthly "review last month" nudge on the 1st with a link to the month-review screen.

**Components:** Overview checklist item `reviewLastMonth` (Text + LinkButton `goToMonthReview` → month-review screen); settings row auto-seeded.

**Data:** query `qEnsureMonthlyReview`: on app load, if no month_review reminder exists with due_date = 1st of current month, INSERT one (kind='month_review', schedule_type='monthly_date', schedule_day=1, due_date=1st). The link passes `?month=<YYYY-MM>` of the previous month.

**Logic notes:** due date = 1st; grace until the 7th, then auto-mark done so it doesn't linger forever. Repeats monthly without user config. The nudge text: "Time to review last month's spending."

**Acceptance:** (1) on the 1st of the month an unchecked checklist item appears, (2) the button navigates to the month-review screen for the prior month, (3) marking done clears it until next 1st, (4) if dismissed after the 7th it self-resolves.

### 563. Remind users to confirm due recurring instances each morning (digest form).

**Components:** Overview "Morning digest" card `morningDigest` listing today's due recurring instances (from recurring rules, per §17 spec), each row with Confirm / Skip buttons.

**Data:** transformer `tDueToday`: filters recurring rules whose next occurrence date = today. Query `qLogDigestShown`: INSERT reminder_events action='shown' for the digest. Confirm → inserts the transaction (user action, per rule 4).

**Logic notes:** single consolidated row group, not one reminder per instance — "3 recurring items due today: confirm or skip each." Digest collapses when all are actioned. Never posts without an explicit Confirm tap.

**Acceptance:** (1) with two rules due today, digest shows exactly two rows, (2) Confirm creates the transaction and marks that row complete, (3) Skip removes the row without posting, (4) no reminder_events or transactions exist until the digest is shown or actioned.

### 564. Nudge when no transactions have been entered for 7 days.

**Components:** Overview checklist item `inactivityNudge` ("Haven't logged anything in 7 days — add your recent spending").

**Data:** transformer `tDaysSinceLastTxn`: today − max(transactions.date, is_sample=false). Query `qInactivityReminder`: INSERT reminders kind='inactivity' when gap ≥ 7, once per gap; status reset to scheduled when a new transaction is entered.

**Logic notes:** computed from user rows only — sample data must not mask inactivity. Fire at most once per 7-day gap; entering any transaction clears it (status='done'). Quiet-day dates (see 575) suppress it.

**Acceptance:** (1) with the last user transaction 9 days ago, the nudge appears, (2) with only sample data newer than that, it still appears (samples excluded), (3) adding a transaction clears it immediately, (4) only one active inactivity row exists at a time.

### 565. Remind about upcoming goal deadlines at 30, 14, and 7 days.

**Components:** Overview checklist group `goalDeadlines`; per-goal row showing days remaining and progress bar.

**Data:** transformer `tGoalDeadlines`: goals with deadline ≠ null and status scheduled → rows at thresholds {30, 14, 7}. Query `qGoalDeadlineReminder`: upsert reminders kind='goal_deadline', one per goal per threshold (dedupe key: goal id + threshold).

**Logic notes:** thresholds fire when days remaining first ≤ threshold. Each row links to the Goals screen contribute control. Completion/deadline changes recompute; a goal completed early marks all its deadline rows done.

**Acceptance:** (1) a goal 10 days out shows the 14-day and 7-day rows (30 already passed without a row), (2) no duplicate rows after re-load, (3) reaching 100% saved clears pending rows, (4) contributing from the row works and returns to the checklist.

### 566. Add a payday checklist reminder: update income, fund envelopes, review bills.

**Components:** Overview checklist item `paydayChecklist` expanding to three sub-tasks with checkboxes: "Log income transaction", "Allocate to budgets", "Review upcoming bills".

**Data:** query `qPaydayReminder`: INSERT kind='payday_checklist', schedule_type='monthly_date', schedule_day=user's payday day-of-month, body=JSON of the three sub-tasks. Sub-task state stored in reminder_events (action='completed', note=sub-task key) or a JSON column on the row.

**Logic notes:** payday day configured once in settings; the reminder recurs monthly. The three sub-items are suggestions — none auto-executes. Marking all three done completes the parent reminder.

**Acceptance:** (1) setting payday=15 creates the monthly row due on the 15th, (2) expanding shows the three sub-tasks, (3) checking all three marks the reminder done, (4) it recurs on the next month's 15th.

### 567. Allow snoozing any reminder with 1-day, 1-week, or custom options.

**Components:** on each Overview checklist item, DropdownButton `snoozeBtn` with options "1 day", "1 week", "Custom…" (opens DateInput modal `snoozeCustom`).

**Data:** query `qSnoozeReminder`: UPDATE reminders SET status='snoozed', snooze_until = today+1 / today+7 / custom date; plus INSERT reminder_events action='snoozed'. Transformer `tVisibleReminders`: filters out snoozed rows until snooze_until ≤ today, then flips back to scheduled.

**Logic notes:** snooze applies to any kind. Custom date must be ≥ tomorrow and ≤ 90 days out. Unsnooze control in settings clears it immediately.

**Acceptance:** (1) "1 day" hides the item until tomorrow, (2) custom date 2027-01-10 restores it that morning, (3) reminder_events records the snooze, (4) invalid custom dates are rejected with an inline message.

### 568. Show all scheduled reminders in one settings screen with edit/delete.

**Components:** screen `sReminders` (Table `tblReminders`: title, kind, schedule summary, next due, status pill), Form `fEditReminder` (title, body, schedule_type, day/time/date), Buttons `saveReminder`, `deleteReminder` (confirm modal).

**Data:** query `qReminders`: SELECT * FROM reminders ORDER BY due_date. Queries `qUpdateReminder`, `qDeleteReminder` (hard delete; audit row records it).

**Logic notes:** editing schedule recomputes due_date via the same next-occurrence transformer as 561. System-kind rows (inactivity, month_review) are editable for timing but their kind can't be changed. Delete requires confirm; a reminder_events row logs the deletion.

**Acceptance:** (1) table lists all kinds with next due dates, (2) editing a weekly time recomputes due_date, (3) delete removes the row after confirm and logs to audit, (4) SAMPLE/badge/banner rules hold on this screen.

### 569. Support custom reminders with freeform text and dates.

**Components:** settings screen button `addCustomReminder` → Modal `mCustomReminder` with TextInput `customTitle`, TextArea `customBody`, DateInput `customDate`, Button `saveCustom`.

**Data:** query `qInsertCustom`: INSERT reminders SET kind='custom', schedule_type='custom_date', scheduled_date=customDate, due_date=customDate, status='scheduled'.

**Logic notes:** title required (≤ 120 chars), body optional (≤ 500 chars), date ≥ today. Custom reminders show verbatim on the Overview checklist. They are one-shot; on completion no recurrence is generated.

**Acceptance:** (1) a reminder "Call accountant", 2026-11-01 saves and appears on Overview, (2) empty title is rejected inline, (3) past dates are rejected, (4) completing it does not create a follow-up row.

### 570. Display reminders as a checklist on the overview until dismissed.

**Components:** Overview container `overviewChecklist` (CheckboxGroup or per-row Checkbox + Dismiss icon-button `dismissReminder`).

**Data:** transformer `tOverviewChecklist`: reminders WHERE status IN ('scheduled','snoozed' expired) AND due_date ≤ today ORDER BY due_date, kind. Query `qCompleteReminder` (status='done') / `qDismissReminder` (status='dismissed') + reminder_events rows.

**Logic notes:** done vs dismissed both remove the row from the checklist; dismissal keeps it in history. Each row shows kind-specific icon and deep link (goal → Goals screen, bill → bills list, review → month review).

**Acceptance:** (1) due items render as checkable rows sorted by due date, (2) checking marks done and removes the row, (3) X dismisses without marking complete, (4) the checklist section shows an "all clear" empty state when none are due.

### 571. Add a "bill week" prep reminder 2 days before the 1st of each month.

**Components:** Overview checklist row `billWeekPrep` appearing on the 29th/30th of each month ("Bill week: review what's due next month") with link to bills screen.

**Data:** transformer `tBillWeekDue`: computes "2 days before the 1st" = last two days of current month (handles 28/29/30/31-day months). Query `qBillWeekReminder`: upsert kind='bill_week', schedule_type='monthly_date', schedule_day = computed, due_date = that date.

**Logic notes:** monthly recurrence without user setup. Self-resolves after the 3rd of the new month. Quiet-day suppression applies (see 575).

**Acceptance:** (1) the row appears on Jan 30 for a 31-day January, (2) appears on Feb 27 for a 28-day February, (3) the link opens the bills screen, (4) it clears automatically after the 3rd.

### 572. Remind users to snapshot net worth at each month end.

**Components:** Overview checklist row `networthSnapshot` ("Snapshot your net worth for the month") with button `takeSnapshot` → net-worth snapshot action (see §36 spec: writes a row to net_worth_snapshots).

**Data:** transformer `tMonthEnd`: due on the last day of month (compute from calendar). Query `qSnapshotReminder`: upsert kind='networth_snapshot', schedule_type='monthly_date'. Snapshot itself is the user's button tap.

**Logic notes:** the reminder only prompts; the snapshot happens when the user taps the button, which records balances as of that moment. If a snapshot already exists for the month, the row marks done automatically.

**Acceptance:** (1) row appears on the last day of the month, (2) tapping the button creates the snapshot row and marks the reminder done, (3) an existing snapshot for the month clears the row without prompting, (4) no snapshot is ever created without the user's tap.

### 573. Nudge when a budget has no limit set by the 5th of the month.

**Components:** Overview checklist row `missingBudgetLimit` per affected category ("Set a limit for Groceries — it's the 5th").

**Data:** transformer `tBudgetsMissingLimits`: budgets for the current month where limit_amount IS NULL or = 0, evaluated when today ≥ 5th. Query `qMissingBudgetReminder`: upsert one row per category, kind='missing_budget'.

**Logic notes:** categories with no budget row at all for the month count as missing. Link opens the Budgets screen add/edit form prefilled with the category. Row clears when a limit > 0 is saved; suppressed before the 5th.

**Acceptance:** (1) on the 5th, categories without limits produce rows; on the 4th they don't, (2) saving a limit clears that category's row, (3) one row per category — no duplicates on reload, (4) the deep link prefills the category in the budget form.

### 574. Keep reminders human-in-the-loop: they prompt action, never auto-post transactions.

**Components:** system-wide rule surfaced in settings as info text: "Reminders suggest — you decide. Nothing is ever posted automatically."

**Data:** none — this is a guardrail over all reminder kinds. Query audit: any reminder kind's scheduled action handler contains zero INSERTs into transactions, zero UPDATEs to balances, zero external sends.

**Logic notes:** the only state changes a reminder may make are to its own status/snooze fields and reminder_events. Digest confirms (563), checklist items (570), and snapshot button (572) all require an explicit user tap that is separate from the reminder firing. Document this invariant in the settings screen footer.

**Acceptance:** (1) with 20 overdue reminders, zero transactions and zero balance changes exist, (2) each reminder's completion path names an explicit user control, (3) the settings screen states the invariant in plain language, (4) code review of reminder handlers shows no writes outside reminders/reminder_events.

### 575. Add quiet-day settings so reminders skip vacations.

**Components:** settings screen section `quietDays` with DateRangeInput `quietRange` (start/end), list of saved quiet periods, delete buttons.

**Data:** table `quiet_days` (NEW): id (pk), start_date (date), end_date (date), label (text, nullable). Transformer `tIsQuietDay(date)`: true when date falls in any saved range. All reminder visibility transformers (570, 563, 564) AND-gate on NOT tIsQuietDay.

**Logic notes:** quiet days suppress display and event-generated rows (inactivity nudges don't accumulate during vacation). Due dates do not shift — items simply reappear the day after the quiet period ends. Overlapping ranges are allowed; validation: start ≤ end.

**Acceptance:** (1) a quiet range Dec 24–31 hides the weekly balance reminder on Dec 26, (2) it reappears Jan 1 still due, (3) inactivity nudges don't fire mid-vacation, (4) start > end is rejected inline.

### 576. Show reminder history with completion states.

**Components:** settings screen tab `historyTab` with Table `tblReminderHistory`: title, kind, due date, status pill (done / dismissed / snoozed), completed date, note.

**Data:** query `qReminderHistory`: SELECT r.title, r.kind, r.due_date, r.status, max(e.occurred_at) FROM reminders r LEFT JOIN reminder_events e ON e.reminder_id=r.id WHERE r.status IN ('done','dismissed') GROUP BY … ORDER BY occurred_at DESC.

**Logic notes:** history covers the last 12 months; older rows remain queryable via a "show older" toggle. Snoozed-then-completed rows show both events in a detail expander. Export row set is included in the quarterly backup export (579).

**Acceptance:** (1) completed and dismissed items appear with correct statuses and dates, (2) default view covers the trailing 12 months, (3) expanding a row shows its snooze/completion event trail, (4) user rows only — is_sample reminder seeds are excluded or badged.

### 577. Allow sharing a reminder list with a household member (view only).

**Components:** settings screen button `shareReminders` → Modal `mShareReminders` generating a read-only share token (UUID) and a formatted checklist text block with a Copy button.

**Data:** query `qCreateShare`: INSERT INTO reminder_shares (id pk, token text unique, created_at timestamp, expires_at date, is_sample bool). Shared view renders the current due checklist as static text/HTML — no live data access, no account balances, no transaction details.

**Logic notes:** view-only by construction: the share payload is a snapshot of reminder titles/due dates only — never balances, amounts, or full transaction rows. Token expires in 30 days; revoke button deletes it. No email/SMS sending — the user copies the text or link themselves.

**Acceptance:** (1) generating a share produces a unique token with a 30-day expiry, (2) the shared payload contains reminder titles and due dates only — no amounts or balances, (3) revoke invalidates the token, (4) no automated message is sent; sharing requires the user's explicit copy action.

### 578. Add motivational copy variants for savings reminders, user-selectable tone.

**Components:** settings Select `savingsTone` (Neutral / Encouraging / Direct) applying to goal-related reminder copy; preview text showing the variant.

**Data:** transformer `tSavingsCopy(goal, daysLeft, tone)`: returns one of three copy templates — Neutral: "Goal 'X': $Y to go, N days left."; Encouraging: "You're $Y from 'X' with N days to go — almost there!"; Direct: "'X' needs $Y in N days. Make the transfer." Stored tone on the reminder row (`tone` column).

**Logic notes:** tone is per-user preference, editable per reminder too. Copy variants never change the underlying math or deadlines — only phrasing. The default tone is Neutral.

**Acceptance:** (1) switching to Encouraging rewrites all savings reminder rows, (2) preview shows the exact text before saving, (3) amounts and deadlines are identical across all three tones, (4) the preference persists across sessions.

### 579. Remind to back up/export data quarterly.

**Components:** Overview checklist row `quarterlyBackup` due on quarter ends (Mar 31, Jun 30, Sep 30, Dec 31): "Back up your data — export this quarter."

**Data:** transformer `tQuarterEnd`: computes the next quarter-end date. Query `qBackupReminder`: upsert kind='backup_quarterly'. The row's action button links to the export screen (§25 spec: CSV/PDF export); completion is user-marked after they export.

**Logic notes:** self-resolves 14 days after quarter end if ignored. Completion is honor-system — the export screen's log can auto-clear it when an export is actually run (optional refinement noted for the builder).

**Acceptance:** (1) the row appears on Mar 31 with the export-screen link, (2) tapping through and exporting can clear it, (3) manually marking done clears it until next quarter end, (4) it self-resolves two weeks after quarter end.

### 580. Log reminder creation and completion in the audit log.

**Components:** none new — audit log screen (§38 spec) gains a source filter value "Reminders".

**Data:** on every reminder INSERT/UPDATE/DELETE and every reminder_events INSERT, query `qAuditWrite`: INSERT INTO audit_log (actor='user', action, entity='reminder', entity_id, detail, occurred_at). Detail includes title + before/after status for changes.

**Logic notes:** creation logs the schedule summary; snooze logs the new snooze_until; completion logs the event action; deletions log the title. Bulk system upserts (e.g. monthly seeding) log once per row, not once per check. Audit rows are append-only.

**Acceptance:** (1) creating a custom reminder writes one audit row with its schedule, (2) snoozing writes a row with the new snooze_until, (3) deleting writes a row naming the deleted reminder, (4) audit rows are never updated or deleted by reminder actions.
