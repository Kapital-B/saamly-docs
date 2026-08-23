# Module spec — `inventory`

Version 0.1 · July 2026 · Status: Draft · Phase: P0a
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.4 (§3.5, D4, D8) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` (§4.2, §10) · Depends on: `core.md`, `capture.md`, `taxonomy.md`, `feedback.md`

## 1. Purpose and pillar

`inventory` is kitchen awareness — knowing, roughly and honestly, what the household already has. It serves pillar 1 (household-aware weekly optimisation) directly: the planner's first input is "use what you have," and the shopping list's first rule is "don't buy what's already there." It is explicitly **not a warehouse ledger** (strategy §9.2): quantities are hints, containers are opaque, and the system's job is to stay useful anyway — confidence states instead of false precision, lightweight confirmation instead of bookkeeping. Its success is measured by whether two real people keep using it in week two, not by stock accuracy.

## 2. Users and jobs

| User | Jobs |
|---|---|
| Household member | Scan the fridge/freezer/cupboards/spice rack and quickly confirm what's there; add and fix items manually without ceremony; mark something finished, opened, or "mine"; trust the app enough to not buy a second bag of spinach |
| `meal plans` (internal) | Deduct confirmed items from requirements; prefer "probably" items; prioritise "use first" early in the week; never consume personal items without permission |
| `shopping` (internal) | Turn "out" and shortfalls into list entries; route uncertain quantities to "check at home" |
| `capture` | Hand scan drafts to this module for dedup and application |

## 3. Scope: Now / Next / Later

- **Now (P0a):** apply confirmed scan drafts (dedup against existing items); the five confidence states; manual add/edit/remove with inline concept resolution; locations (fridge / freezer / cupboard / spice rack); qualitative quantity hints; personal items; staleness display and refresh prompts (no silent decay); offline-first local storage with outbox sync (D8).
- **Next (P1):** "use first" surfacing in plans; mark-cooked deduction (with `cooking`); "used this" shortcuts; re-detection suppression learning (kitchen-confidence model inputs).
- **Later (P2+):** receipt/barcode updates; retailer-order import; expiry estimation; continuous reconciliation.

**Out of scope:** precise quantities, expiry tracking, nutrition data. If a feature requires knowing *exactly how much* of something exists, it waits for the ledger we have decided not to build (strategy §12).

## 4. The state model (semantics binding on `meal plans` and `shopping`)

| State | Meaning | Planner / list behaviour |
|---|---|---|
| **Have** | Confirmed present in sufficient quantity | Deduct from recipe requirements |
| **Probably have** | Detected or remembered, quantity uncertain | Prefer recipes using it; shortfall goes to "check at home" |
| **Use first** | Open, ripe or likely to expire | Boost priority in early-week meals |
| **Personal** | Owned by one member | Never consumed in shared plans without permission |
| **Out** | Confirmed unavailable | Required quantity goes to the shopping list |

**Quantities are qualitative:** `enough / low / unknown`, plus an optional free hint ("half a bag"). The UI never shows counts it cannot stand behind (brand: "never fabricate exact quantities").

**Staleness is shown, not enforced:** every item carries `last_confirmed_at`; ageing items surface a gentle "Still there?" prompt, and a refresh scan is the renewal mechanism. Nothing silently changes state at P0 — silent decay erodes exactly the trust the states exist to build.

## 5. Flows

### 5.1 Apply a scan draft (the core loop)

1. `capture` routes a confirmed `inventory_scan` draft (edited payload) to this module.
2. **Dedup:** each accepted candidate matches existing items by concept_id, then normalised text (household aliases included). Match → update `last_confirmed_at` and merge notes (never a duplicate milk). No match → create item: state **Have** if confirmed with quantity sufficient, **Probably have** where the quantity flag was material and unchecked; `source=scan`, provenance attached.
3. Rejected candidates produce no items; rejection counts feed `capture`'s confirmation-burden metrics and (P1) re-detection suppression.
4. Result lands in "what you have", grouped by location.

**Edge cases:** item rejected this week but detected next week → re-proposed (P0 accepts occasional re-asks over the risk of permanently hiding a real item); same item in two locations → allowed (freezer peas + cupboard peas are both real); household purged → items cascade-delete via `core`.

### 5.2 Manual management

- **Add:** type name → inline resolution against `taxonomy` (alias match suggests the concept; free text always works, D4) → choose location and state (default Have). Three seconds, no form.
- **Quick actions** on any item: *Used up* (→ Out), *Still there* (re-confirm → refreshes timestamp, Probably→Have), *Use first*, *Mine* (→ Personal, owner = actor), *Not this anymore* (remove).
- **Edit:** location, quantity hint, display name; name edits re-resolve and feed the taxonomy correction loop.

### 5.3 Offline (D8)

"What you have" is one of the two local-first surfaces. Reads and mutations run against the on-device store (drift); mutations queue in the outbox with idempotency keys and sync when connectivity returns. Conflicts resolve last-writer-wins per field-group (two people, one kitchen — collisions are rare at P0); the outbox guarantees no action is lost, and the UI never blocks on connectivity.

### 5.4 Empty and uncertain states

First run: *Show Saamly what you have. A few quick photos are enough to get started.* — manual add offered equally, no forced scanning. A household that never scans still works (product test). Scan-only users get the same editing tools; "probably" items always say why ("Spotted in Tuesday's scan — check the amount").

## 6. API boundary

Consumer surface (`/v1`, household context from `core`):

| Endpoint | Purpose |
|---|---|
| `GET /inventory?updated_since=` | Full or delta read (delta powers offline sync) |
| `POST /inventory/items` | Manual add (name_text and/or concept_id, state, location?, quantity_hint?, idempotency key) |
| `PUT /inventory/items/:id` | Edit state/location/quantity/personal/display name |
| `POST /inventory/items/:id/state` | Quick transitions: used_up · still_there · use_first · personal · shared |
| `DELETE /inventory/items/:id` | Remove |

**Internal:** `ApplyScanDraft(draft_id, edited_payload)` — invoked by `capture`'s confirm routing (by kind); `InventoryAvailability(household, concept_ids)` — the read port `meal plans` and `shopping` will use (returns states + quantity hints, never fabricated precision).

**Offline contract:** all mutations accept idempotency keys; outbox replays hit the same endpoints; `GET ?updated_since=` returns tombstones for deleted items so devices can reconcile.

## 7. Dependencies

- **Consumes:** `core` (household context, provenance, events, personal ownership); `capture` (scan drafts, confirm routing); `taxonomy` (resolution on add/edit, correction write-back); `feedback` (catalog registration).
- **Consumed by:** `meal plans` (availability + state semantics, §4), `shopping` (Out/shortfall → list), `cooking` (P1 deduction).
- **Events emitted (registered in `feedback` §4.4 catalog):** `inventory.scan_applied` (created/updated/rejected counts, draft_id) · `inventory.item_added` (source: scan/manual) · `inventory.item_state_changed` (from_state, to_state, trigger) · `inventory.item_removed` · `inventory.refresh_prompted` (stale item count) · `inventory.refresh_completed` (photo_count, duration_ms — pairs with `capture.scan_session_completed` for the P0a speed gate).

## 8. Metrics

| Metric (PRD §8) | Source | P0 target |
|---|---|---|
| **Scan+confirm faster than manual (P0a gate)** | `inventory.refresh_completed` duration vs timed manual baseline | Faster — and both members agree |
| Manual adds per week | `inventory.item_added{source=manual}` | Measured; high rate signals scans are missing things |
| Post-scan early-outs | `have→out` transitions within 3 days of a scan | Low; spikes mean hallucinated items got through confirm |
| Staleness health | distribution of `last_confirmed_at`; `refresh_prompted` response rate | Refresh scans happening at least weekly during dogfood |
| Use-first engagement | `use_first` marks + items consumed from that state | Measured (feeds P1 planner prioritisation) |
| Inventory kept current in week 2+ | active items touched per member per week | **The retention read on the P0a gate — is it a habit?** |

## 9. Risks

| Risk | Mitigation |
|---|---|
| Maintenance becomes the chore the strategy warns about (§12 top risk) | Photo-first refresh; no quantities to count; staleness prompts instead of ledgers; the P0a gate measures burden directly, and failure here stops the line |
| Hallucinated items erode trust in everything else | Materiality flags at confirm (capture §4.2); Probably state for uncertain quantities; early-out metric watches for leaks |
| Dedup creates duplicates or wrong merges | Concept-then-text matching with household aliases; duplicates mergeable by hand; wrong merges editable without ceremony |
| Stale data misleads plans | Age is visible; refresh prompts; planner treats old `last_confirmed_at` as weaker confidence (semantics in §4) |
| Personal items leak into shared meals | State semantics binding on planner (§4); personal items visually distinct |
| Offline conflicts corrupt the list | LWW per field-group documented; outbox idempotency; tombstone sync |

## 10. Voice and labels

- The surface is *What you have* — never "inventory". States labelled exactly: *Have* · *Probably have* · *Use first* · *Personal* · *Out*.
- Quick actions: *Used up* · *Still there?* · *Use first* · *Mine* · *Already have* · *Bought* (list-side).
- Uncertainty is named, with the reason attached: *Spotted in Tuesday's scan — check the amount*, never a bare guess.
- Corrections: *Not in the fridge anymore? We'll add it back to the list.* (brand §13).
- Empty state: *Show Saamly what you have. A few quick photos are enough to get started.* — with *Add it myself* offered with equal weight.
- Locations in household words: *Fridge* · *Freezer* · *Cupboard* · *Spice rack*.
