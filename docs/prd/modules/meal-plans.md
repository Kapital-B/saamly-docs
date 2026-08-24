# Module spec — `meal plans`

Version 0.1 · August 2026 · Status: Draft · Phase: P0c
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.4 (§3.7, D1, D3, D4, D5) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` · Implementation: `docs/rfc/RFC008_The_Week_Implementation.md` · Depends on: `core.md`, `recipes.md`, `inventory.md`, `taxonomy.md`, `feedback.md`

## 1. Purpose and pillar

`meal plans` turns the household's own recipe box and kitchen awareness into a useful answer to *What are we eating this week?* It is the heart of pillar 1 (household-aware weekly optimisation), but P0c deliberately starts with an explainable planner-lite: seven dinner slots, recipes from the household box only, ranked by inventory fit, with swap and regenerate controls. The product test is whether one household can plan from its own recipes without community content, retailer data, precise stock counts or a model inventing meals.

## 2. Users and jobs

| User | Jobs |
|---|---|
| Household member | Ask Saamly to plan the week; understand why a meal was suggested; swap a poor suggestion; regenerate the proposal; accept one shared week |
| Other household member | Open the same proposal, see changes made by the first member and help settle the week |
| `shopping` (internal) | Read an accepted plan's snapshotted ingredient requirements; never read recipes directly |
| Team | Measure whether plans are accepted, how much swapping is needed and whether the household returns the next week |

## 3. Scope: Now / Next / Later

- **Now (P0c):** planner-lite — suggest dinners from the household recipe library, rank by inventory fit, explain useful reasons, swap one meal, regenerate the proposal and explicitly accept the shared week.
- **Next (P1):** full weekly optimisation across ingredient reuse, participation, time and budget; pinned and fixed meals; manual planning; preference-signal learning; conversational refinement.
- **Later (P2+):** leftover chains, batch-prep scheduling, deeper nutrition goals, multi-week planning and cooking-led inventory deduction.

**P0c is dinner-only.** A week is Monday through Sunday in `Africa/Johannesburg` time (the single P0 market); `week_start` must be a Monday and every slot date must fall in its seven-day range. Seven slots exist, but empty days are valid. Breakfast, lunch, snacks, serving adaptation, retailer prices, voting, rotas, nutrition targets, cooking completion and inventory deduction are out of scope. Household timezone configuration arrives with multi-market support.

**Generation is heuristic at P0c.** It does not call an LLM, so it does not increment `CapabilityPlanGenerate`. Any later model-backed implementation must stay behind the same capability boundary, preserve the rules in §4.1 and increment the existing meter.

## 4. Flows

### 4.1 Generate a proposal

1. Member chooses *Plan my week* for the current or next Monday-to-Sunday week.
2. The service loads **household-visible recipes only**. Personal recipes never enter a shared plan; a member must first use *Share with household* in the recipe box.
3. For each resolved ingredient concept, the service reads aggregated household availability. Scoring is explainable:
   - `has_use_first=true` is the strongest positive signal and favours earlier-week placement.
   - `coverage=covered` is a positive inventory-fit signal.
   - `coverage=check_at_home` is a weak positive signal and never supports a claim that enough exists.
   - `coverage=needed` gives no inventory-fit credit.
   - personal inventory is already excluded by the port.
4. The proposal uses each recipe at most once. If fewer than seven eligible recipes exist, remaining slots stay empty; Saamly never invents or silently repeats a meal.
5. Each filled slot receives zero or more structured reason tags and one short explanation derived only from evidence, for example *Uses the spinach first* or *Uses more of what you have*. A recipe with no positive evidence may be suggested for variety, but its explanation must not claim inventory fit.

The exact score weights and tie-breaks belong in the implementation RFC. The product rules above are binding: household recipes only, no personal leakage, no false quantity claims and no opaque reason text.

P0c extends inventory's internal availability contract before either planner or shopping implementation: it aggregates **all shared items per concept across locations** into `coverage`, `has_use_first`, `last_confirmed_at` and `evidence_count`. A single-item concept pointer is not sufficient for planning or deduction. Staleness is computed inside the port from `SAAMLY_INVENTORY_STALE_DAYS`, not reinterpreted by each consumer.

### 4.2 Household preferences

The planner reads every current membership's `core` preferences; pending invites are not members and a missing preferences row is an empty set. Any active member's constraint applies to the shared proposal.

- Allergy values and comma/newline-separated `avoid_text` phrases are compared with recipe `raw_text` and concept names using taxonomy `normaliser-v1` plus whole-token / whole-phrase equality — never arbitrary substring matching.
- A verified taxonomy allergen conflict excludes the recipe.
- A dietary value excludes only when verified taxonomy metadata explicitly conflicts. P0c does not infer “vegetarian”, “halal” or similar labels from ingredient text.
- Unresolved ingredients or dietary constraints without verified metadata set `preference_unknown`. The warning is visible during review but does not block acceptance; the product never labels the recipe safe.

Missing allergen metadata means **unknown**, never safe.

P0c does not ask *Who's eating?*; participation arrives in P1. Until then, every active member's recorded constraints apply.

### 4.3 Review, swap and regenerate

- **Swap meal:** replace one slot with the next recipe in the same ranked candidate order used by generation, excluding recipes already in the proposal. When no alternative remains, return `no_alternative` and leave the slot unchanged.
- **Regenerate:** replace the draft proposal with a different eligible combination where alternatives exist. When no different combination exists, return `no_alternative` and leave the draft unchanged. Regeneration does not create new recipes and does not preserve slots; *Keep this meal* / pins arrive in P1.
- Both actions autosave the shared draft. Changes from another member appear on foreground, pull-to-refresh and ordinary client polling; P0c does not require sockets or presence.
- Concurrent edits to different dates do not clobber. A stale write to the same slot returns a conflict with the server slot and timestamp; the client asks the member to review the changed week.

Calling generate again for a week that already has a current revision returns that revision unchanged; clients use regenerate for a different proposal. Swap and regenerate always operate on a draft. If the current revision is accepted, the mutation first forks it into a new draft revision, leaves the accepted revision immutable, then applies the requested change.

Eligible recipes are non-deleted, household-visible recipes that do not have an explicit preference conflict. Empty titles, ingredients or steps do not make a recipe ineligible. Ranking and swap use the same stable order; generation number rotates only equal-score candidates so regenerate can produce a different combination without randomness.

### 4.4 Accept the week

1. A member chooses *Use this plan*.
2. The service verifies that every slotted recipe still exists and has the `updated_at` captured on the slot. A changed or removed recipe returns `stale_plan` with affected dates; the draft remains editable.
3. The service snapshots every ingredient line from every planned recipe into the plan revision. Each requirement retains `raw_text`, optional identity and the source slot/recipe; it is not numerically aggregated.
4. The service calls `ShoppingListReconciler.ReconcileFromPlan` with the accepted snapshot. Plan acceptance and list reconciliation are one commit boundary: if reconciliation fails, the plan stays a retryable draft and the previous accepted plan/list remain visible. Events are records, not transport at P0.
5. The accepted revision is the week counted by the P0c gate.

Editing an accepted plan creates a new draft revision. The existing shopping list remains stable until the revised plan is accepted, avoiding a surprise list change while somebody is in the shop. Re-accepting replaces the accepted revision for that week; metrics count the week once.

Accepting an already accepted revision is an idempotent read of that plan/list result. If another revision won concurrently, return a stale-plan conflict. A failed accept persists no requirement snapshot; retry repeats validation and snapshots the then-current, still-matching recipe versions.

### 4.5 Empty, error and uncertain states

- **No household recipes:** *Add a few recipes first — Saamly plans from your box.* Link to *Add a recipe*.
- **No inventory:** planning still works from the box; inventory-fit reasons are omitted and `shopping` treats requirements conservatively.
- **No resolved concepts:** the recipe remains plannable from `raw_text`; no match is better than a confident wrong one.
- **Generation failure:** keep the last saved draft and offer retry. Never clear a plan before its replacement is persisted.
- **Household of one:** the full flow works; inviting another member is optional.
- **Recipe edited or removed after acceptance:** the accepted requirement snapshot remains unchanged. A future regeneration reads current recipes.

## 5. Data owned

| Entity / field | Rule |
|---|---|
| Plan | `id`, `household_id`, `week_start`, `status` (`draft` · `accepted` · `archived`), `revision`, `created_by`, `updated_by`, `created_at`, `updated_at`, `accepted_by?`, `accepted_at?` |
| PlanSlot | `date`, `recipe_id`, `recipe_updated_at`, `recipe_title_snapshot`, `reason_tags[]`, `explanation?`, `review_flags[]` (`preference_unknown` only at P0c), `updated_by`, `updated_at` |
| RequirementSnapshot | `candidate_key`, `slot_date`, `recipe_id`, `raw_text`, `quantity?`, `unit?`, `preparation?`, `concept_id?`, `concept_name?` |
| Generation context | algorithm revision and generation number; enough to reproduce or explain the proposal, never a hidden prompt transcript |

There is one current plan per household and `week_start`. Revisions are retained through P0 so acceptance and swap metrics remain auditable; a superseded revision becomes `archived`. Plans and snapshots are household-private, never public, and hard-delete through the `core` household purge registry.

`raw_text` is mandatory in every requirement snapshot. Concept identity is optional and never replaces the household's words. Numeric quantities remain display strings; this module performs no unit conversion or quantity arithmetic.

Field-group conflict boundaries are `status`, each individual `slot:<date>` and `requirements`. Accept snapshots slots and requirements atomically. All mutations are idempotent.

## 6. API boundary

Consumer surface (`/v1`, household context from `core`):

| Endpoint | Purpose |
|---|---|
| `GET /meal-plans/current?week_start=YYYY-MM-DD` | Return the household's current draft or accepted plan; before first generation return 200 with `plan: null` |
| `POST /meal-plans/generate` | Create a first draft or return the current revision unchanged; accepts `week_start` and `Idempotency-Key` |
| `POST /meal-plans/{id}/regenerate` | Replace a draft combination; auto-fork an accepted revision first |
| `POST /meal-plans/{id}/slots/{date}/swap` | Replace one dinner slot; auto-fork an accepted revision first |
| `POST /meal-plans/{id}/accept` | Atomically snapshot requirements, accept the revision and rebuild the shared list |

Writes carry the expected plan or slot timestamp and an `Idempotency-Key`. Missing or cross-household plans return 404, not 403. Conflicts use `problem+json` with the field group and current server timestamp.

P0c generation accepts only the current or immediately following Monday in `Africa/Johannesburg`; malformed, non-Monday and past/far-future dates return validation problems. Reads use an explicit valid `week_start`, so clients do not depend on the server's idea of “current” after the date has been selected.

**Internal ports:**

- `RecipeLibrary` from `recipes`, followed by a mandatory household-visibility filter for shared planning.
- extended `Availability` from `inventory`, returning aggregated `coverage`, `has_use_first`, `last_confirmed_at` and `evidence_count`.
- `HouseholdPreferences` from `core`, returning current members' per-user preferences; missing rows are empty.
- `ShoppingListReconciler`, implemented by `shopping`: `ReconcileFromPlan(household, accepted_plan)` synchronously reconciles a list from the accepted requirement snapshot.

## 7. Dependencies

- **Consumes:** `core` (household context, member preferences, events, clock, purge registry); `recipes` (`RecipeLibrary`); `inventory` (`Availability` and state semantics); `taxonomy` (existing concept identity and verified metadata only); `feedback` (event catalog).
- **Consumed by:** `shopping` through the accepted snapshot passed to `ShoppingListReconciler`; `household` shared-surface UX.
- **Does not consume:** retailer catalogs, community recipes, shopping state or a model.
- **Events emitted** (registered in `feedback`):

| Event | Key payload fields | Class |
|---|---|---|
| `meal_plans.plan_generated` | plan_id, week_start, slot_count, recipes_considered, inventory_fit_slots, algorithm_revision, duration_ms | measures |
| `meal_plans.plan_regenerated` | plan_id, week_start, previous_revision, new_revision, slot_count | measures |
| `meal_plans.meal_swapped` | plan_id, week_start, slot_date, from_recipe_id, to_recipe_id? | identifiers — P1 preference signal |
| `meal_plans.plan_accepted` | plan_id, week_start, revision, slot_count, swaps_before_accept, inventory_fit_slots | measures — P0c gate |

No recipe title, ingredient text, allergy or preference content is copied into events.

`swaps_before_accept` is the draft revision's successful swap count. `inventory_fit_slots` counts filled slots with at least one `covered` concept or non-stale `has_use_first` signal; a slot counts once.

## 8. Metrics

| Metric (PRD §8/§14) | Source | P0c target |
|---|---|---|
| Consecutive weeks planned | distinct `week_start` with `meal_plans.plan_accepted` | 2 consecutive weeks |
| Recipes imported and used | distinct recipe IDs in accepted slots | validates the P0b-to-P0c handoff |
| Swap burden | `meal_swapped` / accepted plan; swaps before acceptance | measured; sustained high burden blocks planner expansion |
| Proposal acceptance | accepted plans / generated plans | trending up during dogfood |
| Inventory-fit coverage | accepted slots with evidence-backed inventory reason / filled slots | measured, never optimised by inventing confidence |
| Time to first useful plan | household creation → first accepted plan | measured for P1 |
| Generation cost | `CapabilityPlanGenerate` meter when a future model is used | zero for the P0c heuristic |

The phase gate combines this module's accepted-week signal with `shopping`'s weekly list-use signal; `feedback` computes the intersection by household and `week_start`.

## 9. Risks

| Risk | Mitigation |
|---|---|
| Planner feels restrictive or repetitive | Swap and regenerate are first-class; empty slots beat invented or forced repetition |
| Explanations overstate what is in the kitchen | Reasons are generated from structured evidence only; qualitative states never become counts |
| Personal recipes or items leak into a shared plan | Shared-recipe filter plus personal-inventory exclusion are binding at the service boundary |
| Allergy metadata creates a false safety claim | Explicit conflicts exclude; unknown remains unknown and is flagged for review |
| Recipe edits silently change an accepted shop | Accepted requirement snapshot is immutable; changes require regeneration and re-acceptance |
| P0c becomes an optimisation project | Heuristic ranking only; budget, participation, reuse optimisation and learning wait for P1 |
| Two members overwrite the same day | Per-slot LWW, expected timestamps and a visible stale-plan conflict |

## 10. Voice and labels

- Surface: *This week*. Primary action: *Plan my week*. Acceptance: *Use this plan*.
- Slot action: *Swap meal*. Whole-plan action: *Plan again*.
- Evidence: *Uses the spinach first* · *Uses more of what you have*.
- Uncertainty: *Check this recipe against your household's needs.* Never *allergy-safe* or *fits everyone* without verified evidence.
- Empty box: *Add a few recipes first — Saamly plans from your box.*
- No AI-announcement language. Saamly *suggests*, *uses* and *notices*; it does not describe an algorithm or model to the household.
