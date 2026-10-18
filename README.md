# FinanceOS

Personal finance dashboard — **in development**. Manual entry only: no bank connection, no syncing. Every figure is typed in by hand or clearly labeled sample data. A persistent banner says it plainly: *"Manual tracking — no bank connection yet."*

**Status:** working Retool draft (5 screens) plus this 1,000-item build plan. **997/1,000 items** are spec'd as builder-ready specs (see `specs/`); the remaining 3 (items 399, 935, 959) are physical iPhone tests.

## Contents

- `roadmap/financeos-1000.md` — the full 1,000-item plan, checkboxes flipped with dated evidence
- `specs/` — 50 detailed builder specs, one per 20-item section. Each item has Components (exact Retool components), Data (queries/transformers/JS), Logic notes, and Acceptance criteria.

## Design principles

1. Nothing ever presents as a live bank feed.
2. Sample rows carry `is_sample=true` + a SAMPLE badge, excluded from totals when hidden.
3. Bank-connection items are design specs only — connecting anything real is the user's own hands.
4. Single-user app: sharing/household items are specced as deliberate non-implementation guards.

## Links

- Project page: https://financeos.fullstackassets.com
- Built by [Nicholas Albertson](https://github.com/full-stack-assets)
