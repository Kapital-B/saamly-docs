# Module spec — `shopping`

Version 0.1 · August 2026 · Status: Draft · Phase: P0c
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.4 (§3.8, D1, D4, D5, D8) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` (§4.2) · Implementation: `docs/rfc/RFC008_The_Week_Implementation.md` · Depends on: `core.md`, `meal-plans.md`, `inventory.md`, `feedback.md`

## 1. Purpose and pillar

`shopping` turns an accepted week into one honest, shared answer to *What do we still need?* It is the P0c foundation of pillar 3: aggregate the plan's ingredient requirements, account conservatively for what the household has, separate uncertainty into *Check at home* and stay usable in a supermarket with poor signal. It is a list, not a retailer basket — no prices, packs, stores or false quantity arithmetic.

The boundary is binding: `shopping` consumes accepted plan requirements and inventory availability. It **never reads recipes directly**. The accepted plan is the record of what the household chose to eat.

## 2. Users and jobs

| User | Jobs |
|---|---|
| Household member at home | Review what the week needs, resolve *Check at home*, add something the plan missed and see the other member's changes |
| Household member in a shop | Open the list offline, mark things bought and trust that queued changes will sync |
| `meal plans` (internal) | Reconcile the shared list synchronously when a plan revision is accepted |
| Team | Measure whether the household actually uses the list and where plan coverage or inventory confidence fails |

## 3. Scope: Now / Next / Later

- **Now (P0c):** shared *Still needed* list from the accepted plan; *Check at home* section; manual additions; *Bought* / *Already have* states; offline-first local storage with idempotent outbox sync.
- **Next (P1):** best single-store basket through `retailer catalogs`, pack-aware quantities and price freshness.
- **Later (P2+):** split-shop comparison including fees, minimums and effort; preferred-store basket; direct cart handoff; receipts and barcodes.

**Out of scope:** reading recipes, retailer SKUs or prices, numeric unit conversion, stock accounting, shopping assignments, rotas, cost splitting and automatic inventory creation from a bought item. P0c list generation is deterministic application logic and does not call an LLM.

## 4. Flows

### 4.1 Accepted plan to shared list

1. `meal plans` accepts a revision and synchronously calls `ShoppingListReconciler.ReconcileFromPlan` with its immutable requirement snapshot.
2. Shopping groups requirements:
   - by `concept_id` where identity is attached;
   - otherwise by conservative normalised `raw_text` equality: Unicode case-fold, trim, collapse whitespace and remove surrounding punctuation only — no stemming, singularisation or fuzzy matching;
   - never by fuzzy matching and never by title alone.
3. Grouped lines retain every contributing `raw_text` and display-only quantity label. The first line in slot-date / candidate-key order supplies `display_text`, making retries deterministic. P0c does not add, convert or claim a total. The UI may say *Needed for 3 meals*.
4. The service asks `Availability` for every resolved concept and chooses a section using §4.2.
5. Existing manual rows remain. Existing plan rows with the same stable aggregation key preserve their user state. Removed requirements become tombstones; new requirements become open rows.
6. Both members see the same household list.

Events are evidence, not transport: list reconciliation is a direct in-process call at P0.

### 4.2 Inventory deduction and uncertainty

Inventory remains qualitative. Shopping maps the aggregate returned for all **shared** items matching a concept:

| Availability coverage | List result |
|---|---|
| `covered` | Omit from active sections |
| `check_at_home` | **Check at home** |
| `needed` | **Still needed** |
| requirement has no `concept_id` | **Still needed** — inability to deduct is not evidence that the item is at home |

Multiple locations are considered together. One sufficiently confirmed shared item covers the concept; personal items never cover a shared requirement. No state is silently changed by this module.

Every *Check at home* row carries a reason, for example *We probably saw yoghurt* or *It hasn't been checked recently*. A bare uncertainty badge is not enough.

These rules use inventory's binding aggregation contract. P0c first extends `Availability` to aggregate all shared rows per concept across locations and exclude personal rows; wiring today's single-item projection directly into shopping is invalid. Staleness is already reflected in `coverage` using `SAAMLY_INVENTORY_STALE_DAYS` (default 14), so shopping does not calculate a second threshold.

### 4.3 Resolve the list

- **Bought:** set the row to `bought`. This is a set-state operation, not a toggle, so retries and two members tapping it are safe.
- **Already have:** set the row to `already_have`. This records the list decision only; it does not create or mutate inventory at P0c.
- **Need it:** return a check-at-home row to `open` in *Still needed*.
- A bought or already-have row stays visible in a completed section until the household starts a new week, so both members can see what changed.

Buying food does not prove that it reached the kitchen or remains there. Inventory is refreshed through its own quick actions and scans.

### 4.4 Manual additions

A member may choose *Add something* before or after a plan exists. The item stores their text as `raw_text`, source `manual`, section `still_needed`, state `open` and no taxonomy identity at P0c. Manual rows use `manual:<item_id>` keys and never auto-merge with plan-derived rows, even when the words match. Editing changes the display text only. Manual rows:

- survive plan regeneration and re-acceptance;
- carry forward to a new week while still open;
- archive with the old week when completed;
- can be edited or removed by either member.

A household with no accepted plan still has a useful manual shared list.

### 4.5 Week rollover and plan changes

There is one active list per household. Accepting the first plan attaches its `plan_id` and `week_start`; accepting a revised plan reconciles plan-derived rows while preserving matching statuses and all manual rows.

When a later `week_start` is accepted:

1. archive the previous list snapshot;
2. create the new active list;
3. carry forward open manual rows;
4. derive fresh plan rows;
5. do not carry forward completed rows.

An accepted plan remains the only source of **requirement** changes. Draft swaps and regeneration do not add or remove plan-derived rows from a list somebody may be using in a shop. Inventory evidence may still reclassify an existing requirement under §4.6.

### 4.6 Inventory changes after acceptance

An inventory state change, scan apply or refresh can change whether an existing requirement belongs in *Still needed*, *Check at home* or is covered. `shopping` exposes `RefreshAvailability(household, concept_ids)`; inventory calls it directly after successful mutations that affect an active list. Failure does not roll back the inventory action: the next list open, reconnect or explicit refresh performs a full availability reconcile and self-heals.

This path may reclassify or cover existing plan rows, but it cannot add new recipe requirements, delete manual rows or change a user's `bought` / `already_have` status. An open row that becomes covered is retained server-side as a `covered` tombstone so offline clients remove it; if it later becomes needed/check-at-home, the same row is revived and returned by delta sync. It fulfils inventory's promise that *Used up* can add a required item back to the active list without using events as transport.

When the active list has no plan-derived rows, `RefreshAvailability` is a no-op and emits no reconciliation event.

### 4.7 Offline, empty and error states

*Your shared list* is local-first under D8. Reads come from the device store; writes enqueue with idempotency keys and update the UI immediately. The outbox drains on reconnect, then a delta pull reconciles server changes and tombstones.

- **Offline:** *Offline — changes will sync when you're back.* The list remains fully usable.
- **No plan and no manual rows:** *Plan your week, or add something you already know you need.*
- **Nothing still needed:** *You're set for this week.* Show unresolved *Check at home* separately.
- **Reconcile failure on plan accept:** plan acceptance fails atomically and remains retryable; never report an accepted plan with an unreconciled list.
- **Member removed mid-sync:** next 404 clears the household list from the local store and returns to household selection/onboarding.

## 5. Data owned

| Entity / field | Rule |
|---|---|
| ShoppingList | `id`, `household_id`, `plan_id?`, `week_start?`, `status` (`active` · `archived`), `created_by`, `created_at`, `updated_at`, `archived_at?` |
| ShoppingItem | `id`, `list_id`, `household_id`, `aggregation_key`, `raw_texts[]`, `display_text`, `concept_id?`, `concept_name?`, `quantity_labels[]`, `meal_count?`, `source` (`plan` · `manual`), `section`, `check_reason?`, `state`, `created_by`, `updated_by`, `created_at`, `updated_at`, `deleted_at?`, `omission_reason?` (`covered` · `requirement_removed` · `user_removed`) |
| RequirementRef | `plan_id`, `plan_revision`, `slot_date`, `recipe_id`, `candidate_key` — identity only, not a recipe read path |

`section` is `still_needed` or `check_at_home`. `state` is `open`, `bought` or `already_have`. Completed rows are states, not deleted records.

The stable plan aggregation key is `concept:<id>` when a concept exists, otherwise `text:<conservatively normalised raw_text>` using §4.1. Manual rows use `manual:<item_id>`. The display line remains household wording; `concept_name` is a hint, not a replacement. Plan-derived rows may contain several `raw_texts[]` and quantity labels, but no computed quantity.

Field-group conflict boundaries are `status` (state), `content` (manual display text), `placement` (section/reason) and `derivation` (plan references). Reconciliation may replace only `derivation` and `placement`; it cannot overwrite a manual edit or a newer user status. Deletes are tombstones for delta sync. Household purge hard-deletes lists and items.

## 6. API boundary

Consumer surface (`/v1`, household context from `core`):

| Endpoint | Purpose |
|---|---|
| `GET /shopping-list?updated_since=` | Return the current list, changed items and tombstones; before creation return 200 with `list: null, items: []` |
| `POST /shopping-list/opened` | Record list use with original `occurred_at` and `online`; idempotent and replayable from the mobile outbox |
| `POST /shopping-list/items` | Add a manual row |
| `PUT /shopping-list/items/{id}` | Edit a manual row or move an open row after resolving uncertainty |
| `POST /shopping-list/items/{id}/status` | Set `open` · `bought` · `already_have` |
| `DELETE /shopping-list/items/{id}` | Tombstone a row |

All mutations require `Idempotency-Key`. Updates carry expected field-group timestamps. Cross-household or missing rows return 404. A stale content edit returns `problem+json` conflict; idempotent status sets merge by timestamp and may reconcile silently.

**Internal:**

- `ShoppingListReconciler.ReconcileFromPlan(household, accepted_plan)` is implemented by this module and called synchronously by `meal plans`.
- `RefreshAvailability(household, concept_ids)` is implemented by this module and called directly by inventory after relevant mutations.
- extended `Availability` is consumed from `inventory`; it aggregates all shared rows per concept across locations and excludes personal rows.
- `RecipeLibrary` is deliberately absent. Adding it is an architecture violation.

There is no public regenerate endpoint: accepting a plan is the authoritative trigger. Manual-only lists are created lazily on the first item add. `shopping.list_opened` fires only through the explicit opened endpoint for a persisted list ID; opening the pre-creation empty shell emits nothing. Mobile queues this record when the list is opened offline so the gate measures real use rather than network availability.

## 7. Dependencies

- **Consumes:** `core` (household context, events, clock, purge registry); `meal plans` (accepted requirement snapshot); `inventory` (extended `Availability`); `feedback` (event catalog). Its purge hook must be registered anywhere household purge is composed.
- **Consumed by:** `household` shared-surface UX; `retailer catalogs` and `commerce` in P1+.
- **Does not consume:** `recipes`, `capture`, retailer data or a model.
- **Events emitted** (registered in `feedback`):

| Event | Key payload fields | Class |
|---|---|---|
| `shopping.list_generated` | list_id, plan_id?, week_start?, still_needed_count, check_at_home_count, carried_manual_count | measures |
| `shopping.list_reconciled` | list_id, plan_id, plan_revision, trigger (`plan_reaccepted` · `inventory_changed` · `list_opened`), added, removed, moved, preserved_statuses | measures — only for a plan-backed list |
| `shopping.list_opened` | list_id, week_start?, online | identifiers — emitted at most once per user/list/day |
| `shopping.item_added` | list_id, week_start?, item_id, source (`manual`) | identifiers |
| `shopping.item_status_changed` | list_id, week_start?, item_id, from, to, section | identifiers |
| `shopping.item_removed` | list_id, week_start?, item_id, source | identifiers |

No item text or recipe content is copied into events.

`list_generated` fires only when the first list or a new week list is created. Same-week plan re-acceptance and availability refreshes fire `list_reconciled`; one operation never emits both.

## 8. Metrics

| Metric (PRD §8/§14) | Source | P0c target |
|---|---|---|
| Consecutive weeks planned and shopped | same `week_start` has `meal_plans.plan_accepted` and `shopping.list_opened` or item interaction | 2 consecutive weeks |
| List completion | plan-derived rows moved to `bought` or `already_have` before week end | measured; not a target that encourages false completion |
| Check-at-home resolution | check rows resolved to need/open or already-have | trending up during dogfood |
| Manual-add share | manual rows / total active rows | measured; high share identifies plan coverage gaps |
| Reconciliation churn | plan rows added/removed on re-acceptance | measured; large churn indicates unstable planning |
| Offline reliability | outbox failures, unresolved conflicts and time to sync | no lost list actions during dogfood |
| Shared use | distinct active members opening or mutating a list | both members use the surface during P0c |

`feedback` computes the phase gate as the intersection of accepted-plan weeks and used-list weeks per household. A generated but never opened list does not count as shopping from Saamly.

## 9. Risks

| Risk | Mitigation |
|---|---|
| False quantity precision causes under-buying | Qualitative deduction only; preserve labels; never sum or convert at P0c |
| Unmatched recipe lines disappear | Unresolved requirements go to *Still needed* with `raw_text` intact |
| Personal inventory is exposed or consumed | Personal rows are excluded from shared availability |
| Plan changes erase real-world shopping progress | Lists update only on accepted revisions; stable keys preserve user status and manual rows |
| Two members double-buy or lose a check-off | Offline local source, idempotent set-state mutations, per-field LWW and delta sync |
| Bought items pollute inventory | No automatic inventory write-back at P0c |
| Shopping couples itself to recipe internals | Accepted plan snapshots are the only ingredient source; no `RecipeLibrary` dependency |
| P0c expands into retailer infrastructure | List-only boundary; prices, packs and stores wait for P1 |

## 10. Voice and labels

- Surface: *Your shared list*. Sections: *Still needed* · *Check at home*.
- Actions: *Add something* · *Bought* · *Already have* · *Need it*.
- Empty: *Plan your week, or add something you already know you need.*
- Complete: *You're set for this week.*
- Uncertainty always has a reason: *We probably saw yoghurt — check the amount* · *It hasn't been checked recently*.
- Offline: *Offline — changes will sync when you're back.*
- Never say “deducted inventory”, “optimised basket” or “AI-generated list” in P0c.
