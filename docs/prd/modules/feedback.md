# Module spec — `feedback`

Version 0.1 · July 2026 · Status: Draft · Phase: P0a
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.4 (§3.4, D5, §7, §8) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` (§10, §14) · Depends on: `core.md`

## 1. Purpose and pillar

`feedback` is the instrumentation backbone and the learning loop — the module that turns usage into intelligence. The strategy's defensibility claim (household preference model, kitchen-confidence model, matching quality, decision outcomes) only becomes real if the events that describe behaviour are captured faithfully from day one and actually flow back into the modules that should get smarter. `feedback` serves all three pillars and owns two things: the **semantics** of the event system (envelope, payload rules, the event catalog) and the **dogfood measurement loop** (dashboards in `admin` that tell us whether the P0 gates pass). `core` owns the sink infrastructure; this spec owns what goes into it and how it is read.

## 2. Users and jobs

| User | Jobs |
|---|---|
| Module developers | Know exactly which events to emit, with what payloads; never ship an unmeasurable feature (D5: if it can't be measured, it isn't done) |
| Team (via `admin`) | See dogfood health at a glance; check P0 gate metrics against their targets; inspect raw events when debugging |
| `taxonomy` | Receive correction semantics so write-back rules are consistent (household-scoped aliases, per `taxonomy.md`) |
| `meal plans` (P1) | Consume preference signals (saves, swaps, skips, pins, cooked) to improve proposals |

## 3. Scope: Now / Next / Later

- **Now (P0a):** event envelope semantics and payload policy; the event catalog (registered types + payload contracts); synchronous in-module learning loops (correction write-back); dogfood dashboards in `admin` computed on read; gate-metric definitions with formulas; event-append failure metric.
- **Next (P1):** preference-signal aggregates feed the planner; kitchen-confidence model inputs; aggregate tables/compute jobs for metrics; DynamoDB Streams consumers where async processing earns its complexity.
- **Later (P2+):** aggregated, privacy-thresholded insight products — commercially separated from personal data with k-anonymity minimums, never identifiable (PRD §7; strategy §10 "aggregated unfulfilled-intent insights").

**P0 architecture rule:** events are *records*, not transport. Learning loops run synchronously inside the owning module; no event-driven consumers before P1. This keeps the dogfood system simple while the log accumulates the evidence that justifies later investment.

## 4. Event semantics

### 4.1 Envelope (owned by `core`, semantics defined here)

| Field | Rule |
|---|---|
| event_id | ULID; idempotent append (retries safe) |
| type | `<module>.<past_tense>` (convention owned by `core`, binding) |
| household_id / user_id | household_id required when the action is household-scoped; on account deletion the identity tree is deleted and `user_id` on historical events is left intact (orphan ULID — no joinable profile; surfaces resolve missing users as “a former member”). Events stay append-only |
| payload | per-type contract in the catalog (§4.2); `schema_version` per type |
| occurred_at | client time where meaningful (e.g. scan duration), server time always |

### 4.2 Payload policy

Fields are classified and reviewed at registration:

- **Identifiers** — IDs, enums, status classes. Always allowed.
- **Measures** — counts, durations, scores, confidence values. Always allowed.
- **Content** — free text or media references (e.g. the missed string in `taxonomy.resolution_missed`). Allowed only where the learning loop requires it; flagged `content` in the catalog; never names, emails or household composition.

Additive-only schema changes per type; a breaking change means a new event type. Streams are enabled on the table from day one (RFC §10) so future consumers attach without migration.

### 4.3 Append contract

`EventSink.Append` is synchronous and cheap (one write, millisecond budget), with an idempotency key. **Measurement must never fail a feature:** on append failure the caller logs, increments the `event_append_failed` metric, and continues the user action. Accepted trade-off: a rare lost event over a broken user flow — compensated by the failure metric itself being monitored.

### 4.4 Event catalog (P0 registry)

| Event | Emitted by | Key payload fields | Class notes |
|---|---|---|---|
| `core.user_signed_up` / `core.user_signed_in` | core | provider | identifiers |
| `core.household_created` / `core.invite_created` / `core.invite_accepted` / `core.member_removed` | core | invite_id?, member_count | identifiers |
| `core.preferences_updated` | core | fields_changed[] (names only) | identifiers |
| `core.account_deleted` / `core.household_deleted` | core | grace_period? | identifiers |
| `capture.draft_created` / `capture.draft_discarded` | capture | kind, source_type, photo_count? | identifiers |
| `capture.draft_processed` | capture | kind, duration_ms, item_count, material_count, model_used, est_cost_cents | measures |
| `capture.draft_failed` | capture | kind, reason_class, model_used | identifiers |
| `capture.draft_confirmed` | capture | kind, fields_presented, fields_edited, items_rejected, time_to_confirm_ms | measures — the confirmation-burden feed |
| `capture.scan_session_completed` | capture | photo_count, total_ms | measures — the P0a speed gate |
| `taxonomy.resolution_recorded` | taxonomy | outcome, confidence_bucket, context, resolver_revision, concept_id? | identifiers — every Resolve attempt; no ingredient text |
| `taxonomy.resolution_missed` | taxonomy | missed_string, context, resolver_revision | **content** — feeds review queue |
| `taxonomy.concept_proposed` | taxonomy | concept_id, source, queue_size | identifiers |
| `taxonomy.concept_promoted` | taxonomy | concept_id, source, queue_size | identifiers |
| `taxonomy.concept_merged` | taxonomy | concept_id, into_concept_id, queue_size | identifiers |
| `taxonomy.concept_rejected` | taxonomy | concept_id, queue_size | identifiers |
| `taxonomy.alias_added` | taxonomy | concept_id, scope (global/household), source | identifiers |
| `taxonomy.correction_applied` | taxonomy | from_concept_id?, to_concept_id, corrected_string, scope | **content** — feeds write-back |
| `inventory.scan_applied` | inventory | draft_id, created, updated, rejected | measures |
| `inventory.item_added` | inventory | item_id, source | identifiers |
| `inventory.item_state_changed` | inventory | item_id, from_state, to_state, trigger | identifiers |
| `inventory.item_removed` | inventory | item_id | identifiers |
| `inventory.refresh_prompted` | inventory | stale_item_count | measures |
| `inventory.refresh_completed` | inventory | photo_count, duration_ms | measures — pairs with capture scan timing |
| `recipes.recipe_imported` | recipes | recipe_id, draft_id, source_type, ingredient_count, concept_resolved_count | measures — the P0b import gate |
| `recipes.recipe_added` | recipes | recipe_id, source | identifiers |
| `recipes.recipe_updated` | recipes | recipe_id, fields_changed[] | identifiers |
| `recipes.recipe_visibility_changed` | recipes | recipe_id, from, to | identifiers |
| `recipes.recipe_removed` | recipes | recipe_id | identifiers |
| `meal_plans.plan_generated` | meal plans | plan_id, week_start, slot_count, recipes_considered, inventory_fit_slots, algorithm_revision, duration_ms | measures |
| `meal_plans.plan_regenerated` | meal plans | plan_id, week_start, previous_revision, new_revision, slot_count | measures |
| `meal_plans.meal_swapped` | meal plans | plan_id, week_start, slot_date, from_recipe_id, to_recipe_id? | identifiers — P1 preference signal |
| `meal_plans.plan_accepted` | meal plans | plan_id, week_start, revision, slot_count, swaps_before_accept, inventory_fit_slots, recipe_ids[] | measures — P0c gate; recipe IDs are event-derived |
| `shopping.list_generated` | shopping | list_id, plan_id?, week_start?, still_needed_count, check_at_home_count, carried_manual_count | measures |
| `shopping.list_reconciled` | shopping | list_id, plan_id, plan_revision, trigger, added, removed, moved, preserved_statuses | measures — plan-backed lists only |
| `shopping.list_opened` | shopping | list_id, week_start?, online | identifiers — at most once per user/list/day |
| `shopping.item_added` | shopping | list_id, week_start?, item_id, source | identifiers |
| `shopping.item_status_changed` | shopping | list_id, week_start?, item_id, from, to, section | identifiers |
| `shopping.item_removed` | shopping | list_id, week_start?, item_id, source | identifiers |

Modules added later register their events here as part of their spec's §7; a PR that adds a user action without registering its events fails review (D5).

## 5. Flows

### 5.1 The dogfood measurement loop (P0)

1. Modules append events via `EventSink` as part of normal operation (synchronous, §4.3).
2. `admin` dashboards compute metrics **on read** directly from the event table — dogfood volume (hundreds of events/week) makes aggregation jobs unnecessary at P0.
3. Weekly gate review: the P0a/P0b/P0c targets (PRD §8) are checked against the dashboards; failures route to the owning module's metrics section for diagnosis.
4. Raw-event inspection (`GET /v1/admin/events`) supports debugging and golden-set analysis for the vision model.

### 5.2 Correction write-back (synchronous learning, P0)

Capture confirm detects an edited match → applies the correction **synchronously inside `taxonomy`** (household-scoped alias up, misleading alias weight down — per `taxonomy.md` §4.4) → emits `taxonomy.correction_applied` as the record. The event is not the mechanism; it is the evidence that lets us measure and later automate the mechanism.

### 5.3 Preference signals (P1, designed now)

Planner-facing signals are registered when their user actions enter scope, with payload shapes a preference model can consume without re-processing. P0c planning-lite emits accept and swap signals even though nothing consumes them yet. Skip, pin and cooked-completion events register with the P1/P2 actions rather than fabricating interactions P0c does not offer.

## 6. API boundary

**Consumer surface:** none.

**Admin surface** (Entra JWT, allowlisted, logged): `GET /v1/admin/metrics/overview` — the P0 gate dashboard (per-metric current value, target, trend) · `GET /v1/admin/events?type=&household=&since=` — raw event inspection with content fields visible only to the allowlist.

**Internal:** `EventSink.Append(EventEnvelope)` contract (§4.3, implemented in `core`); metric derivation formulas (§8) binding on dashboard implementations.

## 7. Dependencies

- **Consumes:** `core` (`EventSink`, `MeterSink`, event envelope storage).
- **Consumed by:** `admin` (dashboards), `taxonomy` (correction semantics), `meal plans` (P1 preference signals), `commerce` (meter/cost data for pricing decisions).
- **Events emitted:** none at P0 (the measurement module measures); P1 aggregate jobs will emit `feedback.metric_computed`.

## 8. Metrics

The P0 gate metrics this module computes, with formulas binding on the dashboards (targets in PRD §8):

| Metric | Formula (from catalog events) |
|---|---|
| Scan+confirm vs manual | median `capture.scan_session_completed.total_ms` vs timed manual baseline |
| Auto-resolution rate | (resolver exact + fuzzy_accept) / `Resolve` calls (from `taxonomy.*` outcomes) |
| Correction rate | `taxonomy.correction_applied` / resolutions in period |
| Confirmation burden | mean `fields_edited + items_rejected` per `capture.draft_confirmed`; trend |
| Review queue trend | pending queue size from `taxonomy.concept_*` net flow |
| Import completion | `capture.draft_confirmed{kind=recipe_import}` / `draft_created{kind=recipe_import}` |
| Weeks planned and shopped | consecutive Monday `week_start` values where the same household has `meal_plans.plan_accepted` and `shopping.list_opened` or a shopping item interaction carrying that same `week_start` |
| Cost per active household | MeterCounters / active households in period |
| **Event-append failure rate** | `event_append_failed` / appends — the module's own health metric; target ≈ 0, alert on any sustained non-zero |

## 9. Risks

| Risk | Mitigation |
|---|---|
| Events silently lost → the moat never accumulates | Append-failure metric with alerting; idempotent appends; coverage check in review (D5) |
| Payload creep turns events into surveillance | Three-class payload policy (§4.2); content fields justified at registration; privacy review for any new content class |
| Schema churn breaks dashboards | Additive-only versioning; breaking change = new event type |
| Premature event-driven architecture adds dogfood-phase complexity | P0 rule (§3): events are records; learning is synchronous; streams consumers deferred to P1 |
| Pseudonymisation on account deletion doesn't actually work | Deletion job covers event tombstoning and is tested; verified via admin inspection (PRD §14 "deletion success") |
| Later commercial insight erodes trust | Hard separation (PRD §7): k-anonymity thresholds, no identifiable data, explicit as a P2+ gate rather than a slide |

## 10. Voice and labels

- Internal-facing only; no consumer surface. Dashboard labels stay plain and outcome-named: *Scan-to-done time* · *Corrections per 100 matches* · *Confirmation burden* · *Weeks planned in a row* — the same words the PRD gates use, so conversation about numbers needs no translation.
- Event names are engineer-facing and follow `<module>.<past_tense>` exactly; no cleverness, no synonyms.
