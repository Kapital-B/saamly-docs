# RFC 004 — Inventory Module Implementation

Status: Draft · July 2026
Relates to: `docs/prd/modules/inventory.md` v0.1 · `docs/prd/modules/core.md` v0.1 (§5, §6) · `docs/prd/modules/capture.md` v0.1 (§8) · `docs/prd/modules/taxonomy.md` v0.1 · `docs/prd/modules/feedback.md` v0.1 (§4.4) · `docs/rfc/RFC001_Initial_Technical_Structure.md` (§4.2, §9, §10, §15) · `docs/rfc/RFC002_Core_Module_Implementation.md` · `docs/rfc/RFC003_Capture_Module_Implementation.md` · scaffold in `saamly-service` (capture shipped P0a slices A–E)

This RFC specifies *how* `inventory` is implemented in `saamly-service`. The module spec owns product behaviour (the five confidence states, the apply loop, staleness, voice); this RFC owns the Go package layout, DynamoDB key design, use-case boundaries, the `Applier` contract that `capture` already calls, the offline sync contract, delivery slices, and the decisions that must be locked before writing handlers. It deliberately does not restate the flows in `inventory.md` — it references them.

## 1. Goals and non-goals

**Goals**

- Ship the P0a `inventory` surface end-to-end: apply confirmed `inventory_scan` drafts (dedup against existing items); the five confidence states (`have` / `probably_have` / `use_first` / `personal` / `out`); manual add / edit / remove with inline concept resolution; locations (fridge / freezer / cupboard / spice rack); qualitative quantity hints (`enough` / `low` / `unknown` + free hint); personal items with owner; staleness display and refresh prompts (no silent decay); offline-first sync contract (delta read + tombstones + idempotency keys).
- Make `capture.ConfirmDraft` real: replace `NoopApplier` with `inventory.ApplyDraft` so a confirmed scan produces kitchen state. This is the P0a loop the PRD gates on (scan+confirm faster than manual, both members agree).
- Reuse `core` infrastructure verbatim: `EventSink`, `MeterSink`, `provenance.Record`, household-context middleware, `AuthFrom(ctx)`, the `PurgeHook` registry already wired in RFC 003 §4.
- Keep the domain free of DynamoDB / HTTP / AWS SDK types (RFC 001 §3.2). No SDK structs above `adapters/`.
- Leave capture green: the `Applier` interface already exists in `application/capture/service.go`; inventory supplies a real implementation and registers it in `cmd/api`. Capture tests stay green.

**Non-goals**

- Precise quantities, expiry tracking, nutrition data (inventory.md §3 out of scope; strategy §12 — the ledger we have decided not to build).
- "Use first" surfacing in `meal plans`, mark-cooked deduction, "used this" shortcuts, re-detection suppression learning (inventory.md §3 Next, P1).
- Receipt / barcode updates, retailer-order import, expiry estimation, continuous reconciliation (inventory.md §3 Later, P2+).
- The planner / shopping read port `Availability` — defined here as a port so `meal plans` and `shopping` can consume it later, but those modules are out of scope for this RFC.
- Taxonomy resolution logic — `taxonomy` owns the `ConceptResolver`; inventory calls it best-effort and falls back to free text (D4).
- Event *semantics* and the catalog — owned by `feedback.md`; inventory only emits the `inventory.*` events listed there.
- The on-device drift store and outbox — lives in `saamly-mobile`; this RFC defines the *server contract* they sync against (delta read, tombstones, idempotency), not the client.
- Push notification when staleness prompts fire (poll / pull at P0; the inbox is the source of truth).

## 2. Package layout

`inventory` spreads across the hexagonal layers already scaffolded. `internal/domain/inventory/doc.go` exists as a placeholder. New code lands here:

```
saamly-service/
  internal/
    domain/
      inventory/              # NEW — Item, State, Location, QuantityHint, Source, sentinel errors
    application/
      inventory/            # NEW — ApplyDraft, AddItem, UpdateItem, TransitionState,
                             #       RemoveItem, ListItems (delta), GetItem,
                             #       Availability (read port impl), PurgeHook
    ports/
      repository.go         # EXTEND — ItemRepository
      availability.go       # NEW — Availability read port (consumed by meal plans / shopping later)
      resolver.go            # EXISTING (taxonomy-owned) — ConceptResolver (optional)
      events.go / meter.go / idgen.go / clock.go   # EXISTING
    adapters/
      dynamodb/
        keys.go            # EXTEND — item key builders
        item.go            # NEW — ItemRepository
      memory/
        item.go            # NEW — in-memory ItemRepository (mirrors memory/draft.go)
        store.go           # EXTEND — add items map
      httpapi/
        inventory.go       # NEW — handlers for /inventory/*
        server.go          # EXTEND — replace inventory stub with real handlers
  cmd/
    api/main.go            # EXTEND — wire inventory use cases + handlers; register
                             #          inventory.ApplyDraft as capture.Applier for KindInventoryScan
    worker/main.go         # EXTEND — register inventory.PurgeHook in the household purge registry
  api/
    consumer.yaml          # EXTEND — inventory paths (before handlers)
```

Rules binding on this work (carried from RFC 002 / RFC 003):

1. **Use cases own transactions.** `ApplyDraft` writes multiple items (creates + updates + a `scan_applied` event) and receives a `Transactor` port or a repository method that performs one DynamoDB `TransactWriteItems`. Handlers never orchestrate multi-item writes.
2. **AuthContext is set once.** Middleware resolves `(user, household, role)`; handlers and use cases read `application.AuthFrom(ctx)` — they never re-query identity (core.md §6). The actor is the `ImportedBy` / `ChangedBy` on provenance.
3. **Events never fail features.** Every use case that emits wraps `EventSink.Append` best-effort: log + continue on error (feedback.md §4.3).
4. **No AWS types above adapters.** Domain entities are plain structs; repositories accept/return them.
5. **`capture` owns the draft; `inventory` owns the item.** The `Applier` port is the only seam. Inventory never reads draft tables; it receives an edited payload and provenance from capture and writes items.

## 3. Domain model

### 3.1 `inventory` package (new)

| Type | Fields | Notes |
|---|---|---|
| `Item` | `ID`, `HouseholdID`, `ConceptID?`, `NameText`, `DisplayName`, `State`, `Location`, `QuantityHint`, `QuantityFree?`, `OwnerID?`, `Source` (`scan` / `manual`), `Provenance` (embeds `provenance.Record`), `LastConfirmedAt`, `CreatedAt`, `UpdatedAt`, `DeletedAt?`, `Notes[]` | The central object (inventory.md §4) |
| `State` | `have` · `probably_have` · `use_first` · `personal` · `out` | The five confidence states; semantics binding on `meal plans` and `shopping` (inventory.md §4) |
| `Location` | `fridge` · `freezer` · `cupboard` · `spice_rack` | Household words (inventory.md §10) |
| `QuantityHint` | `enough` · `low` · `unknown` | Qualitative only; never a count (inventory.md §4) |
| `Source` | `scan` · `manual` | Drives `inventory.item_added{source}` and dedup behaviour |
| `Note` | `Text`, `Source` (`scan` / `edit` / `system`), `At`, `By?` | Merged on dedup; never a duplicate milk (inventory.md §5.1) |
| `ConceptCandidate` | `ID`, `Name`, `Score` | From `taxonomy.ConceptResolver`; optional (D4 free-text fallback) |

Errors (sentinel): `ErrItemNotFound`, `ErrItemAlreadyRemoved` (idempotency guard on delete), `ErrInvalidState`, `ErrInvalidLocation`, `ErrInvalidQuantityHint`, `ErrPersonalOwnerRequired` (transition to `personal` without an owner), `ErrConceptUnresolved` (only when a caller demands a concept and none matches — manual add does not).

### 3.2 State semantics (binding)

The state machine is intentionally permissive — most transitions are legal because the user is always right and the UI offers quick actions for all of them. The binding part is what each state *means* to other modules, not which transitions are allowed.

| From → To | Allowed | Notes |
|---|---|---|
| any → `have` | yes | "Still there" / "Bought" — refreshes `LastConfirmedAt` |
| any → `probably_have` | yes | Scan-detected, quantity unchecked |
| any → `use_first` | yes | Open / ripe / likely to expire — boosts early-week priority (P1) |
| any → `personal` | yes | Requires `OwnerID`; never consumed in shared plans without permission |
| any → `out` | yes | "Used up" — required quantity goes to the shopping list |
| `out` / `personal` → `have` | yes | "Already have" / "Bought" re-confirms |
| any → deleted | yes | "Not this anymore" — sets `DeletedAt`, emits tombstone |

**Staleness is shown, not enforced** (inventory.md §4): `LastConfirmedAt` ages; the read path computes a staleness band (`fresh` / `aging` / `stale`) but never mutates state. A refresh scan is the only renewal mechanism; nothing silently decays at P0.

### 3.3 Provenance

Embeds `provenance.Record` (RFC 002 §3.3). On scan-apply, provenance is copied from the draft (so the item remembers which scan produced it, who confirmed it, when). On manual add, `SourceType = manual`, `ImportedBy = actor`, `ImportedAt = now`. On edit, provenance is *appended* as a `Note` with `Source = edit` — the original import provenance stays on the item; we do not rewrite history (RFC 002 §3.3 — provenance is append-only).

### 3.4 Lifecycle rules (binding)

- **Idempotent mutations.** Every write endpoint accepts an `Idempotency-Key` (header). A second request with the same key returns the original result. The idem item lives at `IDEM#<key>` with a 24h TTL, exactly mirroring RFC 003 §4. This is what makes offline outbox replay safe (inventory.md §5.3).
- **Last-writer-wins per field-group.** Two members editing the same item concurrently: the later `UpdatedAt` wins, scoped to the field-group that changed (`state` / `location` / `quantity` / `display_name` / `ownership`). The repository stores `UpdatedAt` per field-group, not a single row timestamp, so a state change doesn't clobber a simultaneous location edit. Collisions are rare at P0 (two people, one kitchen) but the contract is documented (inventory.md §5.3).
- **Tombstones, not deletes.** `DELETE /inventory/items/:id` sets `DeletedAt` and writes a tombstone event. The item is excluded from the default read but retained for delta sync. Household purge hard-deletes; individual removal soft-deletes.
- **Retention.** Soft-deleted items are hard-deleted after `SAAMLY_INVENTORY_TOMBSTONE_DAYS` (default 30) via a TTL attribute on the row. Household purge (`core`) cascades to inventory via the `PurgeHook` registry (RFC 002 §4) — `inventory` registers its hook in slice E.
- **Offline (D8).** The server contract is: `GET /inventory?updated_since=` returns items with `UpdatedAt > since` *including tombstones*; mutations accept idempotency keys; the outbox replays hit the same endpoints. The client owns conflict resolution; the server is the source of truth.

## 4. DynamoDB key design

All items live in the single table (`saamly-{env}`). Keys are built only inside `adapters/dynamodb/keys.go`. New patterns:

| Entity | PK | SK | GSI1PK / GSI1SK | TTL attr |
|---|---|---|---|---|
| Item | `HOUSE#<householdId>` | `ITEM#<id>` | `ITEM#<householdId>#<location>` / `<updatedAt>#<id>` (items by location, freshest first) | optional `ttl` on tombstoned items |
| Item concept lookup | `HOUSE#<householdId>` | `CONCEPT#<conceptId>` | — | — |
| Idempotency key | `IDEM#<key>` | `META` | — | `ttl` (= 24h) |

**Reads (binding):**

- **Default list ("what you have"):** `GSI1PK = ITEM#<householdId>#<location>` for each location the client shows, merged client-side; or a single Query on `PK = HOUSE#<householdId>` with `SK begins_with ITEM#` for the full household list. The GSI1 path is for "show me the fridge" without scanning; the PK path is for delta sync. Both exclude `DeletedAt IS set`.
- **Delta sync:** Query `PK = HOUSE#<householdId>` with a filter on `UpdatedAt > since` (or a GSI2 if we add one — see §14 open question). Tombstones are returned with `DeletedAt` set so the client can reconcile. Page via `LastEvaluatedKey`.
- **Dedup lookup (apply):** Query `PK = HOUSE#<householdId>`, `SK = CONCEPT#<conceptId>` for concept matches; fall back to a normalised-text scan over the household's items for free-text matches. The concept lookup is O(1); the text scan is bounded by household item count (P0: tens to low hundreds) and acceptable without a secondary index. A `normalised_name` attribute on each item powers the scan filter.

**Conditional writes that matter:**

- Item create: `attribute_not_exists(PK)` on `HOUSE#…/ITEM#…`; idempotency key item written in the same `TransactWriteItems` (or a preceding conditional put on `IDEM#…`).
- State transition / edit: `UpdateItem` with a condition expression on the field-group's `UpdatedAt` being older than the incoming write (last-writer-wins). On `ConditionalCheckFailedException` the use case returns the current item and a `409` with the conflicting field-group so the client can merge.
- Soft delete: conditional on `attribute_not_exists(DeletedAt)`; `ErrItemAlreadyRemoved` on conflict (idempotent delete).
- Tombstone expiry: a `ttl` attribute set to `now + SAAMLY_INVENTORY_TOMBSTONE_DAYS` when `DeletedAt` is set; DynamoDB TTL reaps the row.

**Household purge cascade:** `inventory` registers a `PurgeHook` with `application/household` that, for a given `householdID`, deletes all `HOUSE#<id>/ITEM#*` and `HOUSE#<id>/CONCEPT#*` items via a Query + batch delete in chunks of 1000 (DynamoDB `BatchWriteItem` limit). Hard delete — no tombstone, the household is going. Full cascade is wired in slice E; the hook interface is defined in slice A so `core` can call it. This mirrors the capture `PurgeHook` (RFC 003 §4) exactly.

## 5. Ports to add or extend

| Port | Owner | Notes |
|---|---|---|
| `ItemRepository` | ports (new) | `Put`, `Get`, `GetByConcept`, `ListByHousehold(householdID, updatedSince, limit, cursor)`, `ListByLocation(householdID, location, limit, cursor)`, `UpdateState` (conditional, LWW), `UpdateFields` (conditional, LWW per field-group), `SoftDelete` (conditional), `HardDelete(householdID, itemID)`, `PutIdempotency`, `GetIdempotency` |
| `Availability` | ports (new) | `States(ctx, householdID, conceptIDs) (map[conceptID]StateView, error)` — the read port `meal plans` and `shopping` will use. Returns `{state, quantity_hint, last_confirmed_at}` per concept; never fabricated precision. Implemented in `application/inventory` over `ItemRepository`. |
| `ConceptResolver` | ports (existing, taxonomy-owned) | Optional; `Resolve(ctx, text) (ConceptCandidate, error)` — inventory calls it on add/edit and ignores `ErrNoMatch` (free-text fallback, D4) |
| `EventSink` | ports (existing) | DynamoDB adapter |
| `MeterSink` | ports (existing) | Not used by inventory at P0 (no LLM calls); the port is wired for symmetry and for P1 cooked-deduction metering |
| `IDGen` | ports (existing) | ULID for item IDs, idempotency keys, event IDs |
| `Clock` | ports (existing) | `SystemClock`; tests inject fixed clock |
| `Transactor` | ports (optional) | Only if repository methods prove insufficient for `ApplyDraft`'s multi-item write |
| `PurgeHook` | application/household (existing interface) | `Purge(ctx, householdID) error` — registered by `inventory` in slice E, alongside capture's |

## 6. The `Applier` contract (binding)

`capture` already defines the `Applier` interface in `application/capture/service.go`:

```go
type Applier interface {
    ApplyDraft(ctx context.Context, d capture.Draft) error
}
```

`inventory` provides the real implementation for `KindInventoryScan`:

```go
// internal/application/inventory/apply.go
func (s *Service) ApplyDraft(ctx context.Context, d capture.Draft) error
```

**Binding rules:**

1. `ApplyDraft` receives the *edited* payload (the client may have corrected fields at confirm time). It reads `d.Payload` per the `inventory_scan` shape (RFC 003 §3.2): `{ "items": [{ "name_text", "quantity_hint"?, "concept_candidate"?, "confidence", "materiality" }] }`.
2. It dedups each accepted candidate against existing household items by `concept_id` then normalised text (household aliases included). Match → update `LastConfirmedAt` and merge notes. No match → create item with `Source = scan`, provenance copied from `d.Provenance`.
3. State on create: `have` if the candidate was confirmed with sufficient quantity; `probably_have` where the quantity flag was material and unchecked (inventory.md §5.1).
4. Rejected candidates (the client removed them at confirm) produce no items; the rejection count is emitted in `inventory.scan_applied`.
5. The whole apply is one `TransactWriteItems` (or a small number of chunked transactions if the candidate list exceeds the 100-item TransactWrite limit). Idempotent: a second call with the same `draft_id` no-ops (the draft is already `confirmed`; capture guards this).
6. `ApplyDraft` is **synchronous** from `capture.ConfirmDraft` (RFC 003 open decision 3, locked: synchronous at P0). If it fails, `ConfirmDraft` returns the error and the draft stays `needs_review` — the client retries confirm. Events are records, not transport (feedback.md P0 rule).

`recipes.ApplyDraft` for `KindRecipeImport` lands at P0b and registers the same way. Until then, capture's confirm for recipe drafts still uses `NoopApplier`.

## 7. Use cases (API-side)

Each use case is a struct with explicit dependencies, a single exported method, and unit tests with fakes. Handlers translate HTTP ↔ use-case I/O and map domain errors to problem+json.

| Use case | Input | Behaviour (inventory.md) | Events |
|---|---|---|---|
| `ApplyDraft` | `capture.Draft` (edited payload) | Dedup + create/update items per §6 | `inventory.scan_applied` (created/updated/rejected counts, draft_id) |
| `AddItem` | name_text and/or concept_id, state, location?, quantity_hint?, idempotency_key | Inline `ConceptResolver.Resolve` (best-effort, D4); create item with `Source = manual`, default state `have` | `inventory.item_added` (source: manual) |
| `UpdateItem` | item_id, field-group edits | Conditional LWW update per field-group; name edits re-resolve and feed taxonomy correction | `inventory.item_state_changed` (if state changed) |
| `TransitionState` | item_id, to_state, trigger | Quick-action transitions (§3.2); `personal` requires owner; refreshes `LastConfirmedAt` on `have` | `inventory.item_state_changed` (from_state, to_state, trigger) |
| `RemoveItem` | item_id, idempotency_key | Conditional soft delete; tombstone | `inventory.item_removed` |
| `ListItems` | household_id, updated_since?, location?, cursor | Delta or full read; tombstones included when `updated_since` set | — |
| `GetItem` | item_id | Full item; 404 if not in caller's household or soft-deleted (unless `include_deleted`) | — |
| `Availability` (read port) | household_id, concept_ids[] | Return state views for `meal plans` / `shopping` | — |

## 8. HTTP surface

Replace the `501` inventory stub in `httpapi.NewServer`. Contract changes land in `api/consumer.yaml` *before* handlers (RFC 001 §6). Spectral must stay at zero errors.

**New routes (inventory.md §6):**

| Method | Path | Handler |
|---|---|---|
| GET | `/v1/inventory?updated_since=&location=&cursor=&limit=` | `ListItems` — full or delta read |
| POST | `/v1/inventory/items` | `AddItem` — body: name_text and/or concept_id, state, location?, quantity_hint?; header: `Idempotency-Key` |
| GET | `/v1/inventory/items/{id}` | `GetItem` |
| PUT | `/v1/inventory/items/{id}` | `UpdateItem` — body: field-group edits |
| POST | `/v1/inventory/items/{id}/state` | `TransitionState` — body: `{ "state": "...", "trigger": "used_up" }` |
| DELETE | `/v1/inventory/items/{id}` | `RemoveItem` — header: `Idempotency-Key` |

All under the existing `Authn` middleware (household context from `core`). Items are scoped to the caller's active household — an item from another household returns `404` (not `403`, to avoid leaking existence), matching the capture draft scoping rule.

**Problem+json shapes (binding):**

- `404` item not found / not in household → title `not_found`, detail `We couldn't find that item — it may have been removed.`
- `409` LWW conflict → title `conflict`, detail naming the conflicting field-group and the server's current `UpdatedAt`; client merges and retries.
- `409` item already removed → title `conflict`, detail `That item is already gone.` (idempotent delete).
- `422` invalid state / location / quantity → title `invalid_request`, detail naming the field.
- `422` personal owner required → title `invalid_request`, detail `Marking something as "mine" needs a household member.` (brand voice, inventory.md §10).

**OpenAPI gaps to close in slice A** (before handlers):

- All six routes above with request/response schemas.
- `InventoryItem` schema (§3.1) and `State` / `Location` / `QuantityHint` enums.
- `StateView` schema for the `Availability` read port (internal, but documented).
- Problem responses for `400` / `404` / `409` / `422` where domain conflicts apply.
- `Idempotency-Key` header parameter on the three write endpoints.

## 9. Middleware and composition

`cmd/api` wiring after `core` and `capture` (RFC 002 §8, RFC 003 §10):

1. Build `ItemRepository` (memory or DynamoDB), `ConceptResolver` (nil at slice A; taxonomy lands later).
2. Build inventory use cases (`ApplyDraft`, `AddItem`, `UpdateItem`, `TransitionState`, `RemoveItem`, `ListItems`, `GetItem`, `Availability`).
3. Register `inventory.ApplyDraft` as the `capture.Applier` for `KindInventoryScan` in the `capture.Service.Appliers` map (replacing `NoopApplier`).
4. Pass `InventoryHandlers` into `httpapi.Deps`; replace the inventory stub in `server.go` with real handlers.
5. Dev bypass unchanged — `dev-user` + seeded household lets dogfood inventory without OAuth.

`cmd/worker` wiring:

1. Build `ItemRepository`.
2. Register `inventory.PurgeHook` in the household purge registry (alongside capture's).
3. No LLM wiring — inventory does not call the parser at P0.

## 10. Configuration additions

Extend `platform/config` (and `.env.example`):

| Env | Purpose | Default |
|---|---|---|
| `SAAMLY_INVENTORY_TOMBSTONE_DAYS` | TTL on soft-deleted items before hard reaping | `30` |
| `SAAMLY_INVENTORY_LIST_LIMIT` | Default page size for `GET /inventory` | `100` |
| `SAAMLY_INVENTORY_DEDUP_TEXT_THRESHOLD` | Normalised-text similarity threshold for free-text dedup | `0.9` (Levenshtein ratio) |

`config.Load` validates: `SAAMLY_INVENTORY_TOMBSTONE_DAYS` ≥ 1; `SAAMLY_INVENTORY_LIST_LIMIT` 1–500. No AWS-region or LLM requirements — inventory is pure CRUD at P0.

## 11. Delivery slices

Implement and merge as five small PRs. Each slice is independently demoable and keeps `main` green. Slices assume capture (RFC 003) is already merged.

| Slice | Delivers | Demo |
|---|---|---|
| **A — Domain + ports + OpenAPI** | `domain/inventory` (Item, State, Location, QuantityHint, Source, sentinels); `ItemRepository` port; `Availability` port; all six OpenAPI paths + schemas; `make spec-lint` green; key builders in `keys.go`; memory + DynamoDB repo skeletons (no logic yet) | `curl GET /v1/inventory` returns `[]` (stubbed handler, 200 not 501); Spectral stays green |
| **B — Apply + manual CRUD** | `ApplyDraft` use case (dedup, create/update, provenance copy, one transaction); `AddItem`, `UpdateItem`, `TransitionState`, `RemoveItem`, `GetItem`, `ListItems` (delta); register `inventory.ApplyDraft` as capture's `Applier` for `KindInventoryScan`; memory repo implemented; unit tests with fakes | Local end-to-end: `POST /capture/text` (inventory kind) → confirm → `GET /inventory` shows the items; manual `POST /inventory/items` then `POST /items/:id/state` |
| **C — DynamoDB repo + LWW + tombstones** | Implement `adapters/dynamodb/item.go` fully: conditional creates, LWW per field-group, soft delete with tombstone TTL, delta query, dedup lookup; idempotency keys; floci integration test tagged `//go:build integration` | `make dev-floci` against floci: same demo as slice B but persisted; restart API, items survive |
| **D — HTTP handlers + problem+json** | `httpapi/inventory.go` handlers for all six routes; error mapping per §8; `Idempotency-Key` header handling; replace stub in `server.go`; `cmd/api` wiring | Mobile: snap fridge → upload → poll → confirm → "what you have" shows items; manual add; quick actions (used up, still there, mine) |
| **E — Purge cascade + retention + Terraform** | `inventory.PurgeHook` registered with `application/household`; DynamoDB TTL on tombstoned items confirmed in Terraform (the table TTL attr already exists from RFC 002); seed script extension (`make seed-inventory`); `make test` + `make spec-lint` green | Delete household → inventory items gone; `terraform apply` to dev green |

**Exit criteria for "inventory done" (P0a):** all six endpoints return real responses (no `501`); a confirmed `inventory_scan` draft produces items in `GET /inventory`; manual add/edit/remove/state-transition all work; delta sync returns tombstones; household purge cascades to inventory; `make test` and Spectral stay green; the P0a gate metric (scan+confirm faster than manual, both members agree) is measurable from emitted events.

## 12. Testing strategy

| Layer | What | How |
|---|---|---|
| Domain | State transition invariants; materiality rules; quantity validation; normalisation | Table-driven pure tests |
| Use cases | Every use case with fake ports + fixed clock | Assert state changes, emitted events, error mapping; fake `ItemRepository` and `ConceptResolver` |
| Apply | Dedup by concept then text; create vs update; provenance copy; rejection counts; idempotency | Fake repo with pre-seeded items; drive `ApplyDraft` with a fixture draft |
| Adapters | DynamoDB key helpers; conditional-write paths; GSI1 location query; delta query; LWW condition expressions | floci integration test tagged `//go:build integration` |
| HTTP | Handler mapping + problem+json shapes; household scoping (404 for other households); idempotency-key handling | `httptest` with fake use cases |
| End-to-end | Upload → worker → confirm → inventory list; manual add → state transition | One floci integration test covering the P0a loop |

Fakes live next to the use case tests (`application/inventory/fakes_test.go`), mirroring the capture pattern. The fake `ItemRepository` returns configurable items so dedup and LWW logic can be exercised without DynamoDB.

## 13. Risks and open questions

| Risk / question | Proposal |
|---|---|
| Maintenance becomes the chore the strategy warns about (§12 top risk) | Photo-first refresh; no quantities to count; staleness prompts instead of ledgers; the P0a gate measures burden directly, and failure here stops the line |
| Hallucinated items erode trust in everything else | Materiality flags at confirm (capture §4.2); `probably_have` state for uncertain quantities; early-out metric watches for leaks |
| Dedup creates duplicates or wrong merges | Concept-then-text matching with household aliases; duplicates mergeable by hand; wrong merges editable without ceremony |
| Stale data misleads plans | Age is visible; refresh prompts; planner treats old `last_confirmed_at` as weaker confidence (semantics in §3.2) |
| Personal items leak into shared meals | State semantics binding on planner (§3.2); personal items visually distinct; `OwnerID` required on transition |
| Offline conflicts corrupt the list | LWW per field-group documented; outbox idempotency; tombstone sync; client owns merge |
| `ApplyDraft` transaction exceeds 100-item TransactWrite limit | Chunk into multiple transactions; acceptable at P0 scan sizes (a fridge scan is tens of items, not hundreds). Document the chunk boundary |
| Delta sync without a GSI2 scans the household partition | At P0 household item counts (tens to low hundreds) a PK Query with filter is acceptable; add a GSI2 on `updatedAt` only if profiling shows it |
| Concept resolver not ready | Inventory functions without it; `ConceptID` is nil; free-text fallback (D4). Wire when taxonomy lands |
| `recipes` applier not ready at P0a | Capture's confirm for recipe drafts still uses `NoopApplier`; inventory is independently demoable |

**Open questions for decision before slice A:**

1. Confirm LWW granularity — per field-group (proposal) vs per item (simpler, more clobbering). Proposal: per field-group, since two people editing different aspects of the same item simultaneously should not conflict.
2. Confirm the delta-sync read model — PK Query + filter (proposal) vs a dedicated GSI2 on `updatedAt`. Proposal: PK Query at P0; add GSI2 only if profiling during dogfood shows read latency.
3. Confirm the dedup text threshold default (0.9 Levenshtein ratio) — tunable via env, but the golden set should drive the real value before P0a exit.
4. Confirm whether `Availability` should be a separate read-port interface or a method on `ItemRepository` — proposal: separate port, so `meal plans` and `shopping` depend on a narrow interface, not the full CRUD repository.

## 14. Out of scope reminders

Do not sneak into `inventory` PRs:

- Precise quantities, expiry tracking, nutrition data (inventory.md §3 out of scope).
- "Use first" surfacing in plans, mark-cooked deduction, "used this" shortcuts (P1).
- Receipt / barcode updates, retailer-order import, expiry estimation, continuous reconciliation (P2+).
- Taxonomy resolution logic — `taxonomy` owns the resolver.
- Event consumers (feedback.md P0 rule: events are records, not transport).
- The on-device drift store and outbox — lives in `saamly-mobile`.
- Push notification when staleness prompts fire.
- The mobile inventory UI (lives in `saamly-mobile`; this RFC is service-side).

## 15. Implementation checklist (summary)

When this RFC is accepted:

1. Expand `api/consumer.yaml` for the §8 gaps; `make spec-lint`.
2. Slice A → E as sequenced above; each PR references this RFC and the relevant inventory.md section.
3. Update `inventory.md` status to `Accepted` / bump to v0.2 only if product behaviour changes; technical drift belongs here.
4. Mark RFC 001 §15 step 3 (`inventory`) as in progress in a one-line amendment note when slice A merges.
5. Before P0a exit: run the scan+confirm vs manual timing against the dogfood household; record the baseline from `inventory.scan_applied` and `inventory.refresh_completed` events (inventory.md §8).

