# RFC 007 — Recipes Module Implementation

Status: Draft · August 2026
Relates to: `docs/prd/modules/recipes.md` v0.1 · `docs/prd/modules/capture.md` v0.1 (§5.2, §8) · `docs/prd/modules/taxonomy.md` v0.1 · `docs/prd/modules/feedback.md` v0.1 (§4.4) · `docs/prd/modules/consumer-web.md` v0.1 · `docs/rfc/RFC001_Initial_Technical_Structure.md` · `docs/rfc/RFC003_Capture_Module_Implementation.md` (§8 apply routing) · `docs/rfc/RFC004_Inventory_Module_Implementation.md` · `docs/rfc/RFC006_Taxonomy_Lite_Implementation.md` (§10.1 `raw_text`) · scaffold in `saamly-service` (`domain/recipe` placeholder; `KindRecipeImport` already parsed; confirm still uses `NoopApplier`)

This RFC specifies *how* `recipes` is implemented. The module spec owns product behaviour (the box, visibility, ingredient passthrough); this RFC owns the Go package layout, DynamoDB keys, use-case boundaries, the `Applier` that `capture` already calls, delivery slices, and the decisions that must be locked before handlers. It does not restate the flows in `recipes.md`.

## 1. Goals and non-goals

**Goals**

- Ship the P0b recipe box end-to-end: apply confirmed `recipe_import` drafts; personal + household library; provenance; list / get / edit / visibility / remove; manual add; delta read with tombstones; idempotent writes.
- Make `capture.ConfirmDraft` real for recipes: replace `NoopApplier` with `recipes.ApplyDraft` so a confirmed paste or photo becomes a library entry. This is the P0b loop the PRD gates on (≥20 kept recipes; both members import unprompted).
- Reuse `core` infrastructure verbatim: `EventSink`, `provenance.Record`, household-context middleware, `AuthFrom(ctx)`, the `PurgeHook` registry, `Idempotency-Key` (24h).
- Keep the domain free of DynamoDB / HTTP / AWS SDK types. No SDK structs above `adapters/`.
- Leave capture and inventory green. Recipes supplies a second `Applier` and registers it for `KindRecipeImport` in `cmd/api` and `cmd/worker`. Capture tests stay green.
- Preserve RFC 006's recipe field: `ingredients[].raw_text`. Do not introduce `name_text` on recipe lines.

**Non-goals**

- URL import, share sheet, and voice — capture-owned intake. This RFC defines the apply side so those sources land in the same box when capture adds `source_type=url` / `share`; it does not implement fetch-from-URL.
- Serving adaptation, substitutions, nutrition, unit conversion factors (recipes.md §3 Next / Later; taxonomy lite forbids conversion fields).
- Planner / shopping / cooking behaviour beyond a `RecipeLibrary` read port.
- Seeded content library (`seeded-content.md`, P1).
- Public publishing (`sharing` / `community`).
- The on-device recipe cache — lives in `saamly-mobile`; this RFC defines the server contract (delta + tombstones + idempotency).
- Taxonomy resolution logic — `taxonomy` owns the resolver. Recipes calls it best-effort on manual add and ingredient edits.
- Event *semantics* — owned by `feedback.md`; recipes only emits the `recipes.*` events listed there.

## 2. Package layout

`internal/domain/recipe/doc.go` is the placeholder. New code lands here:

```
saamly-service/
  internal/
    domain/
      recipe/                 # NEW — Recipe, Ingredient, Visibility, Source, sentinels
    application/
      recipes/                # NEW — ApplyDraft, AddRecipe, UpdateRecipe,
                              #       SetVisibility, RemoveRecipe, ListRecipes, GetRecipe,
                              #       RecipeLibrary, PurgeHook
    ports/
      repository.go           # EXTEND — RecipeRepository
      recipes.go              # NEW — RecipeLibrary read port (meal plans later)
    adapters/
      dynamodb/
        keys.go               # EXTEND — recipe key builders
        recipe.go             # NEW — RecipeRepository
      memory/
        recipe.go             # NEW
        store.go              # EXTEND
      httpapi/
        recipes.go            # NEW
        server.go             # EXTEND — /v1/recipes/*
  cmd/
    api/main.go               # EXTEND — wire recipes; register ApplyDraft for KindRecipeImport
    worker/main.go            # EXTEND — same applier + PurgeHook
  api/
    consumer.yaml             # EXTEND — recipe paths before handlers
  scripts/
    seed-recipes.sh           # NEW — paste → confirm → GET /recipes
saamly-web/
  src/routes/                 # EXTEND — library + add-recipe (reuse capture inbox)
```

Rules binding (carried from RFC 002–004):

1. **Use cases own transactions.** `ApplyDraft` writes the recipe + the `source_draft_id` uniqueness row in one `TransactWriteItems`. Handlers never orchestrate multi-item writes.
2. **AuthContext is set once.** Use cases read `application.AuthFrom(ctx)`.
3. **Events never fail features.** `EventSink.Append` is best-effort (log + continue).
4. **No AWS types above adapters.**
5. **`capture` owns the draft; `recipes` owns the recipe.** The `Applier` port is the only seam. Recipes never reads draft tables.

## 3. Domain model

### 3.1 `recipe` package (new)

| Type | Fields | Notes |
|---|---|---|
| `Recipe` | `ID`, `HouseholdID`, `OwnerID`, `Visibility`, `Title`, `Servings`, `Times` (`Prep`, `Cook` strings), `Ingredients[]`, `Steps[]`, `SourceNote`, `SourceDraftID`, `Source`, `Provenance`, `CreatedAt`, `UpdatedAt`, `DeletedAt?`, `HeaderUpdatedAt`, `IngredientsUpdatedAt`, `StepsUpdatedAt` | The box entry (recipes.md §4) |
| `Ingredient` | `CandidateKey`, `RawText`, `Quantity`, `Unit`, `Preparation`, `ConceptID`, `ConceptName`, `ResolverRevision` | `RawText` mandatory; concept fields optional |
| `Visibility` | `personal` · `household` | Default on import: `household` |
| `Source` | `photo` · `paste` · `url` · `manual` | Copied from the draft `source_type`, or `manual` |
| `Times` | `Prep`, `Cook` | Free strings; never parsed into durations |

Errors (sentinel): `ErrRecipeNotFound`, `ErrRecipeAlreadyRemoved`, `ErrInvalidVisibility`, `ErrPersonalOwnerRequired`, `ErrNotOwner` (visibility / delete of someone else's personal recipe), `ErrEmptyRecipe` (manual add with neither title nor any `raw_text` line).

A confirmed draft with empty title and empty steps is **not** `ErrEmptyRecipe` — capture already accepted it. Manual add requires at least a title or one ingredient line.

### 3.2 Visibility (binding)

| Visibility | Who can list/get | Who can edit/remove | Planner (P0c) |
|---|---|---|---|
| `household` | any household member | any household member | included |
| `personal` | owner only | owner only | included only for that member's plan |

Default on `ApplyDraft` and `AddRecipe`: `household`. Request body may set `personal` (requires `OwnerID = actor`).

Cross-household access and other members' personal recipes return `404`, not `403`.

### 3.3 Provenance

Embeds `provenance.Record`. On apply, copy from the draft. On manual add: `SourceType = manual`, `ImportedBy = actor`, `ImportedAt = now`. Edits do not rewrite import provenance.

### 3.4 Lifecycle rules (binding)

- **Idempotent apply** on `source_draft_id` (conditional put of `HOUSE#… / DRAFTSRC#<draft_id>` → recipe id). Retry after a failed confirm cannot create a second recipe.
- **No title dedup.** Two recipes named "Bobotie" coexist.
- **LWW per field-group** (`header` / `ingredients` / `steps`), same shape as inventory.
- **Tombstones.** Soft delete + `SAAMLY_RECIPE_TOMBSTONE_DAYS` (default 30) TTL. Household purge hard-deletes recipes and draft-source pointers.
- **Passthrough.** Apply copies `raw_text` from the draft payload; it does not re-resolve. Re-resolution happens only on manual add and on ingredient edits.

## 4. DynamoDB key design

All items live in `saamly-{env}`. Builders only in `adapters/dynamodb/keys.go`.

| Item | PK | SK | GSI1PK / GSI1SK | Notes |
|---|---|---|---|---|
| Recipe | `HOUSE#<hid>` | `RECIPE#<id>` | `RECIPE#<hid>` / `<updated_at>#<id>` | Payload + field-group timestamps |
| Draft source (unique) | `HOUSE#<hid>` | `DRAFTSRC#<draft_id>` | — | Conditional create; points at recipe id |
| Idempotency | `IDEM#<key>` | `META` | — | 24h TTL; existing pattern |

List/delta: Query `PK=HOUSE#<hid>`, `SK begins_with RECIPE#`. At P0 (tens of recipes) filter `updated_since` and visibility in process. Add a visibility GSI only if dogfood list latency requires it.

Never infer deletion from seed absence — recipes are user data, not the taxonomy seed.

## 5. Mapping from a capture draft

`ApplyDraft` accepts a `capture.Draft` with `Kind == recipe_import` and status `confirmed` (or the pre-status payload — see §6). Payload shape is already in RFC 003 §4 / `consumer.yaml`:

```
title?, servings?, times?, ingredients[] { candidate_key, raw_text, quantity?, unit?,
  preparation?, concept_candidate?, candidates?, resolver_revision? }, steps[], source_note?
```

Mapping rules:

| Draft field | Recipe field |
|---|---|
| `title` | `Title` (trim; may be empty) |
| `servings` | `Servings` if a positive int, else 0 |
| `times.prep` / `times.cook` | `Times` |
| `ingredients[].raw_text` | `Ingredient.RawText` (skip lines with empty raw_text) |
| `ingredients[].quantity/unit/preparation` | same |
| `ingredients[].candidate_key` | `CandidateKey` (generate a ULID if the worker omitted one) |
| `ingredients[].concept_candidate` | attach `ConceptID` / `ConceptName` / `ResolverRevision` **only** when `outcome` is `exact` or `fuzzy_accept` |
| `ingredients[].candidates` | **discard** — advisory only; never persist a list of guesses on the recipe |
| `steps[]` | `Steps` (trim empties) |
| `source_note` | `SourceNote` |
| draft `source_type` | `Source` (`photo` / `paste` / `url`) |
| draft id | `SourceDraftID` |
| draft provenance | `Provenance` |
| actor | `OwnerID` |

Do not call `ConceptResolver` inside `ApplyDraft`. Capture already resolved; `ConfirmDraft` already wrote household corrections.

## 6. Apply and confirm ordering (binding)

RFC 003 allowed confirm to succeed with a missing applier. Inventory and the current `ConfirmDraft` call the applier **after** flipping status to `confirmed`. If `ApplyDraft` fails, the draft is stuck confirmed and the recipe was never written.

**Lock:** `ConfirmDraft` must call `ApplyDraft` **before** the status transition (or revert to `needs_review` on apply error). Taxonomy corrections already run before the transition (RFC 006). Recipes apply sits with them:

1. Load draft; reject if not confirmable.
2. Apply taxonomy corrections against the edited payload.
3. `recipes.ApplyDraft` (idempotent on `source_draft_id`) for `recipe_import`.
4. Transition `needs_review` → `confirmed` and persist the edited payload.
5. Emit `capture.draft_confirmed` (best-effort).

If step 3 fails, return the error; the client retries confirm. A successful apply + failed status write is recovered by the uniqueness row: retry finds the existing recipe and then finishes the status transition.

This fix is in `application/capture/draft.go` and must keep inventory's apply on the same path (scan apply also moves before the status flip). Capture unit tests update to match.

## 7. Use cases

| Use case | Input | Behaviour | Events |
|---|---|---|---|
| `ApplyDraft` | `capture.Draft` | Map §5; conditional create by `source_draft_id`; default visibility household | `recipes.recipe_imported` |
| `AddRecipe` | title?, ingredients[], steps?, visibility?, idempotency_key | Resolve free-text lines best-effort; `ErrEmptyRecipe` if nothing to store | `recipes.recipe_added` |
| `UpdateRecipe` | id, field-group edits, expected timestamps | LWW; ingredient concept/text changes → `CorrectionWriter` | `recipes.recipe_updated` |
| `SetVisibility` | id, visibility | Owner-only when current or target is `personal` | `recipes.recipe_visibility_changed` |
| `RemoveRecipe` | id, idempotency_key | Soft delete | `recipes.recipe_removed` |
| `ListRecipes` | q?, visibility?, updated_since?, cursor | Visible set only; tombstones when `updated_since` set | — |
| `GetRecipe` | id | 404 if not visible | — |
| `RecipeLibrary` | household + user | Thin read port over list/get for P0c | — |

`AddRecipe` / ingredient edits: `Resolve` each `raw_text`; attach concept only on exact / fuzzy-accept; miss stays text. Do not create provisional concepts from the API hot path (RFC 006).

## 8. HTTP surface

Contract changes land in `api/consumer.yaml` *before* handlers. Spectral stays at zero errors.

| Method | Path | Handler |
|---|---|---|
| GET | `/v1/recipes?q=&visibility=&updated_since=&cursor=&limit=` | `ListRecipes` |
| POST | `/v1/recipes` | `AddRecipe` — `Idempotency-Key` |
| GET | `/v1/recipes/{id}` | `GetRecipe` |
| PUT | `/v1/recipes/{id}` | `UpdateRecipe` — field-group body + expected timestamps |
| POST | `/v1/recipes/{id}/visibility` | `SetVisibility` — `{ "visibility": "personal\|household" }` |
| DELETE | `/v1/recipes/{id}` | `RemoveRecipe` — `Idempotency-Key` |

All under existing `Authn`. Cross-household / non-visible → `404`.

**Problem+json:**

- `404` → `not_found` — *We couldn't find that recipe — it may have been removed.*
- `409` LWW → `conflict` — name the field-group and the server `UpdatedAt`.
- `409` already removed → `conflict` — *That recipe is already gone.*
- `422` empty manual add / invalid visibility → `invalid_request`.

**OpenAPI slice A** adds the six routes, `Recipe` / `Ingredient` / `Visibility` / `Source` schemas, and problem responses. Reuse `ConceptCandidate` already on the draft payload; the stored recipe inlines `concept_id` + `concept_name`, not a candidate object.

## 9. Middleware and composition

`cmd/api` and `cmd/worker`:

1. Build `RecipeRepository` (memory or DynamoDB) beside items/drafts.
2. Build the recipes service with optional `ConceptResolver` + `CorrectionWriter` (the taxonomy service already wired).
3. Register `recipes.ApplyDraft` as `Appliers[KindRecipeImport]`, replacing `NoopApplier`.
4. Register `recipes.PurgeHook` next to capture, inventory, and taxonomy hooks.
5. Pass handlers into `httpapi.Deps`.

No new LLM wiring. Recipe parse spend stays on capture's `CapabilityRecipeParse`.

## 10. Configuration

| Env | Purpose | Default |
|---|---|---|
| `SAAMLY_RECIPE_TOMBSTONE_DAYS` | TTL on soft-deleted recipes | `30` |
| `SAAMLY_RECIPE_LIST_LIMIT` | Default page size | `50` |

Validate: tombstone days ≥ 1; list limit 1–200.

## 11. Delivery slices

Independently green PRs. Capture + inventory + taxonomy lite are already merged.

| Slice | Delivers | Demo |
|---|---|---|
| **A — Domain + ports + OpenAPI** | `domain/recipe`; `RecipeRepository` + `RecipeLibrary` ports; consumer.yaml paths; key builders; `make spec-lint`; events in `feedback.md` | `GET /v1/recipes` stub 200 `{ recipes: [] }` — not 501 |
| **B — Apply + CRUD (memory)** | All use cases; memory repo; `ApplyDraft` registered; confirm-ordering fix in `ConfirmDraft`; unit tests | `POST /capture/text` kind `recipe_import` → confirm → `GET /recipes` shows the recipe; retry confirm returns the same id |
| **C — DynamoDB** | Conditional create, LWW, tombstones, draft-source uniqueness, delta query; floci `//go:build integration` | `make dev-floci`; restart API; recipe survives |
| **D — HTTP + composition** | Handlers + problem+json; `cmd/api` + `cmd/worker` wiring; `make seed-recipes` | Web/curl: paste bobotie → confirm → box; personal visibility 404 for the other member |
| **E — Web library** | `saamly-web`: Add a recipe (reuse capture text + optional photos), inbox confirm, `/recipes` list + detail + edit + share/remove; MSW + Vitest | Browser dogfood of the P0b loop |
| **F — URL intake (optional same week)** | Capture `source_type=url` + fetch/extract; still applies through this module unchanged | Paste a URL → same confirm → same box |

**Exit criteria for "recipes done" (P0b software):** all six endpoints real; a confirmed `recipe_import` produces a `GET /recipes` row; manual add/edit/visibility/remove work; apply is idempotent; household purge cascades; `raw_text` never dropped; `make test` + Spectral green; import-completion events exist so the ≥20-recipe gate is measurable.

URL (slice F) is not an exit gate for the box.

## 12. Testing strategy

| Layer | What | How |
|---|---|---|
| Domain | Visibility rules; empty-recipe; ingredient passthrough | Table-driven |
| Apply | Mapping; advisory candidates ignored; draft-source idempotency; empty title allowed | Fixture drafts |
| Confirm order | Apply failure leaves draft confirmable; scan apply still runs | `application/capture` tests with a failing recipe applier **and** a succeeding inventory applier |
| Use cases | CRUD, LWW, corrections on ingredient edit, 404 personal | Fake repo + fake resolver |
| Adapters | Keys, conditional unique draft-source, tombstones | floci integration |
| HTTP | Mapping + problem+json + household scoping | `httptest` |
| Web | Paste → confirm → box; stale LWW banner | MSW/Vitest |

## 13. Clients

**`saamly-web` (slice E)** is in scope because it is the current dogfood client. Reuse the existing capture inbox; add a library home. `consumer-web.md` Next moves from P1 to P0b for this slice only (planner stays P0c).

**`saamly-mobile`** vendors the new `consumer.yaml` and lists recipes with the same contract. A dedicated mobile RFC is not required to start service work; treat Flutter UI as a follow-on PR against this contract (same pattern as RFC 004 vs RFC 005).

Tokens stay Bearer session JWTs. No new auth.

## 14. Risks and open questions

| Risk / question | Proposal |
|---|---|
| Confirm-then-apply orphans recipes | §6: apply before status; uniqueness row makes retry safe |
| Title collision merges two dinners | Never dedup by title |
| Concept name overwrites the cook's words | Persist `raw_text`; UI renders raw_text; concept is identity |
| Household default violates a reading of D6 | D6 is "not public"; household is the P0 boundary (recipes.md §4). Personal is opt-in |
| URL work expands into a second parser | Capture owns URL; this module only applies |
| TransactWrite on large step lists | One recipe row; well under 400 KB at P0. No per-step items |
| Search quality | In-process prefix on the household partition; no OpenSearch |
| `RecipeLibrary` vs repository | Separate narrow port, same as inventory `Availability` |

**Open questions locked by this RFC unless amended:**

1. Default visibility = household (not personal).
2. No title-based dedup.
3. Confirm apply-before-status for **both** appliers.
4. URL import is slice F, not a P0b software gate.

## 15. Out of scope reminders

Do not sneak into `recipes` PRs:

- URL fetching, share extensions, voice (capture).
- Serving math, substitutions, nutrition, conversions.
- Planner ranking, shopping lists, cook-tonight.
- Seed recipe corpus.
- Vector search.
- Public / community publishing.
- Admin review of recipes (not a queue — user data).

## 16. Implementation checklist

When this RFC is accepted:

1. Expand `api/consumer.yaml` (§8); `make spec-lint`.
2. Slices A → E; each PR references this RFC and `recipes.md`. Slice F only if URL is needed the same week.
3. Keep `recipes.md` at Draft v0.1 unless product behaviour changes; technical drift belongs here.
4. Mark `Saamly_PRD_Parent.md` §12 `recipes.md` as Draft v0.1 (done with this RFC pair).
5. Before P0b exit: count `recipes.recipe_imported` minus same-day removes; record confirmation-burden on `kind=recipe_import`; do not turn on taxonomy proposals to "help" imports.
