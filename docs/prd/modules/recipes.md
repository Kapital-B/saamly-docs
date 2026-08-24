# Module spec — `recipes`

Version 0.1 · August 2026 · Status: Draft · Phase: P0b
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.4 (§3.6, D2, D4, D6) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` · Depends on: `core.md`, `capture.md`, `taxonomy.md`, `feedback.md`

## 1. Purpose and pillar

`recipes` is the household recipe box — the place imported food becomes something you can plan and shop. It serves pillar 2 (frictionless personal recipe intelligence) directly and pillar 1 by giving `meal plans` a library to rank. Capture feels like sending content to an inbox, not completing a form: `capture` structures the draft; this module owns the lasting recipe, its provenance, and the personal + household library.

The product test still applies: a household that never publishes, never shops through Saamly, and never sees another user's recipe still gets value from *their* box.

## 2. Users and jobs

| User | Jobs |
|---|---|
| Household member | Paste a WhatsApp recipe or photograph a cookbook page; confirm the draft; find it again in the box; fix a title or an ingredient without ceremony; keep a trial recipe to themselves until they are ready to share |
| `capture` | Hand a confirmed `recipe_import` draft to this module to apply |
| `taxonomy` | Receive ingredient corrections when a recipe is edited |
| `meal plans` (P0c) | Read visible recipes and their resolved ingredient concepts; never invent a recipe that is not in the box |
| Team | Measure import completion and confirmation burden on recipe drafts |

## 3. Scope: Now / Next / Later

- **Now (P0b):** apply confirmed `recipe_import` drafts (photo and pasted text already on `capture`); personal + household library; provenance; list / get / edit / remove; one-tap share-with-household; manual add of a typed recipe; ingredient `raw_text` always preserved with optional taxonomy identity; offline-tolerant list (delta + tombstones + idempotency).
- **Next (P0b–P1):** URL import through `capture` (`source_type=url`); iOS share extension / Android send-to; serving adaptation; substitution suggestions powered by taxonomy relationships.
- **Later (P2+):** licensed collection tooling; controlled dietary adaptation; public publishing (`sharing` / `community` — stage-gated).

**Out of scope:** planning, shopping, cooking timers, nutrition, unit conversion math, retailer packs, seed-library curation (`seeded-content.md`, P1). `capture` still owns parse, drafts and confirm scaffolding. This module does not call the LLM.

## 4. The recipe object

| Field | Notes |
|---|---|
| id, household_id, created_by, created_at, updated_at | standard `core` context |
| visibility | `personal` · `household` |
| owner_id | the importer; required when `personal` |
| title | may be empty; clients say *Give this a name when you have one.* |
| servings | optional integer |
| times | optional `{ prep?, cook? }` free strings — never computed |
| ingredients[] | see §4.1 |
| steps[] | ordered strings; empty is allowed |
| source_note | optional free text ("Gogo's book", a WhatsApp sender) |
| source_draft_id | set when applied from capture; unique per household — apply is idempotent |
| source_type | `photo` · `paste` · `url` · `manual` |
| provenance | `core` ProvenanceRecord (D6); append-only |
| deleted_at? | tombstone for delta sync |

**Privacy boundary (binding):** D6 ("private by default") means recipes are never public. The household is the P0 privacy boundary — the same as inventory. Default visibility on import is **`household`**, so both members see the box they will plan from. `personal` is opt-in (*Just for me*) for experiments that should not enter a shared plan. Promoting personal → household is a one-tap share; the reverse is allowed only for the owner.

### 4.1 Ingredient line

| Field | Rule |
|---|---|
| raw_text | **mandatory passthrough** — the words the household used (D4). Never discarded, never overwritten by a concept name |
| quantity, unit, preparation | optional strings from the parse; labels only — no conversion factors |
| concept_id, concept_name | optional; set only on exact / fuzzy-accept from `taxonomy`. `concept_name` is denormalised; taxonomy renames do not rewrite the recipe |
| candidate_key | stable row key from the worker (RFC 006); used for correction matching |
| resolver_revision | recorded when a concept is attached |

Unmatched or advisory candidates never become `concept_id`. Free text is infinitely better than a confident wrong match.

### 4.2 Lifecycle rules

- **Idempotent apply.** Confirming the same draft twice returns the same recipe (`source_draft_id`). A second paste of similar text is a *new* recipe — we do not silently merge two boboties. Users can delete the duplicate.
- **Idempotent mutations.** Manual writes accept `Idempotency-Key` (24h), matching inventory.
- **Last-writer-wins per field-group.** Groups: `header` (title, servings, times, source_note, visibility), `ingredients`, `steps`. Two people editing different groups do not clobber each other.
- **Tombstones, not deletes.** Remove sets `deleted_at`. Household purge hard-deletes.
- **No silent title invention.** If the parse had no title, store empty and say so. Do not ask a model to name it on apply.

## 5. Flows

### 5.1 Import via capture (the core loop)

1. Member pastes text or photographs a page → existing `capture` draft (`kind=recipe_import`).
2. Review: title, ingredients, steps; material uncertainties first. Voice: *Here's what we found — check the things that matter.*
3. Confirm → taxonomy corrections (already in `ConfirmDraft`) → `recipes.ApplyDraft` → library entry with provenance copied from the draft.
4. Result lands in the box. Empty title is shown plainly; empty steps are a gap, not a failure.

**Edge cases:** apply fails → draft stays confirmable and the client retries (same as inventory); household purged mid-confirm → worker/API no-ops on missing context; same draft confirmed twice → same recipe id.

### 5.2 Browse and edit

- **List:** household recipes + the caller's personal recipes. Default sort `updated_at` desc. Search is title/ingredient prefix on the household partition (P0 scale: tens, not thousands).
- **Get:** 404 if the recipe is personal and the caller is not the owner (do not leak existence).
- **Edit:** title, servings, times, steps, ingredient lines. Ingredient identity or text changes write a household taxonomy correction (RFC 006), then persist.
- **Share / keep to myself:** visibility toggle; owner-only for personal recipes.
- **Remove:** *Not this anymore* — tombstone.

### 5.3 Manual add

Type a title and ingredient lines (one per row, treated as `raw_text`). No capture draft. Source `manual`. Taxonomy resolves lines best-effort; unmatched stay text. Three seconds, no form — the box works for a household that never photographs a cookbook (product test).

### 5.4 Empty and uncertain states

First run: *Add a recipe. Paste from WhatsApp or snap a page — we'll turn it into something you can plan.* Manual add offered equally. A recipe with no steps is still in the box. A recipe with no resolved concepts is still plannable as free text.

## 6. API boundary

Consumer surface (`/v1`, household context from `core`):

| Endpoint | Purpose |
|---|---|
| `GET /recipes?q=&visibility=&updated_since=&cursor=&limit=` | List visible recipes (delta when `updated_since` set; tombstones included on delta) |
| `POST /recipes` | Manual add |
| `GET /recipes/{id}` | Full recipe |
| `PUT /recipes/{id}` | Edit header / ingredients / steps |
| `POST /recipes/{id}/visibility` | `personal` · `household` |
| `DELETE /recipes/{id}` | Tombstone |

**Internal:** `ApplyDraft(draft)` — invoked by `capture` confirm for `recipe_import`; `RecipeLibrary` read port for `meal plans` (list visible + get). No consumer taxonomy autocomplete.

**Capture contract (unchanged):** `ingredients[].raw_text` is the recipe field. Do not introduce `ingredients[].name_text`. Inventory scans keep `items[].name_text`.

## 7. Dependencies

- **Consumes:** `core` (household context, provenance, `EventSink`, purge registry); `capture` drafts via `Applier`; `taxonomy` resolver on manual add / ingredient edits (optional; D4).
- **Consumed by:** `meal plans` (P0c) via `RecipeLibrary`; `shopping` only through a plan, never by reading recipes directly.
- **Events emitted** (registered in `feedback.md` §4.4):

| Event | Key payload fields | Class |
|---|---|---|
| `recipes.recipe_imported` | recipe_id, draft_id, source_type, ingredient_count, concept_resolved_count | measures |
| `recipes.recipe_added` | recipe_id, source (`manual`) | identifiers |
| `recipes.recipe_updated` | recipe_id, fields_changed[] | identifiers |
| `recipes.recipe_visibility_changed` | recipe_id, from, to | identifiers |
| `recipes.recipe_removed` | recipe_id | identifiers |

## 8. Metrics

| Metric (PRD §8/§14) | Source | P0b target |
|---|---|---|
| Recipes imported and kept | `recipes.recipe_imported` − subsequent `recipe_removed` in 24h | ≥20 by end of P0b |
| Import completion | `capture.draft_confirmed{kind=recipe_import}` / `draft_created{kind=recipe_import}` | both members import unprompted; trending up |
| Confirmation burden | `capture.draft_confirmed` fields_edited + items_rejected on recipe drafts | trending down; "minimal correction" |
| Concept coverage on imports | `concept_resolved_count` / `ingredient_count` | measured; feeds the first-50-import taxonomy report |
| Cost per import | existing capture metering (`CapabilityRecipeParse`) | within D7 budget |

## 9. Risks

| Risk | Mitigation |
|---|---|
| Recipe extraction is unreliable | Shared draft-and-confirm; retain source; gaps shown plainly; P0b gate measures correction burden |
| Silent merge destroys two real recipes | Never auto-dedup by title; only `source_draft_id` is unique |
| Concept attach rewrites the cook's words | `raw_text` is the display line; concept is identity, not a rename |
| Personal recipes leak into shared plans | Visibility enforced on read; planner uses `RecipeLibrary` (visible set only) |
| Apply after confirm leaves an orphan confirmed draft | Apply failure must leave the draft confirmable (RFC 007 §6) |
| URL / share sheet delay blocks the box | Photo + paste ship first; URL is a follow-on capture slice |

## 10. Voice and labels

- Entry: *Add a recipe* — never "import" or "ingest".
- Receipt: *Recipe received. We're turning it into something you can plan and shop.*
- Box empty: *Nothing in the box yet. Paste from WhatsApp or snap a page.*
- Untitled: *Give this a name when you have one.*
- Visibility: *Share with household* · *Just for me*.
- Remove: *Not this anymore.*
- No AI-announcement language: Saamly "found", "couldn't read" — never "the model extracted".
