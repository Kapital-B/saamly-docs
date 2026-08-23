# RFC 006 — Taxonomy Lite implementation and evolution

Status: Proposed · August 2026

Owners: `taxonomy`, `capture`, `inventory`, `admin`

Product source: `docs/prd/modules/taxonomy.md` v0.1

Technical context: RFC 001 §10/§14, RFC 003, RFC 004, `feedback.md`

## 1. Summary

Implement **taxonomy lite** as a small, deterministic ingredient identity service inside
`saamly-service`:

- a versioned South African starter set of concepts, aliases and shallow categories;
- household-scoped aliases that always outrank global knowledge;
- exact and conservative in-process fuzzy resolution;
- free-text passthrough on every result and on every failure;
- worker-only, batched LLM proposals for unresolved text;
- a provisional-concept review queue with promote, merge and reject actions;
- synchronous correction write-back during draft confirmation; and
- events and a golden set that make resolution quality measurable.

Taxonomy lite is not a product catalog, nutrition database or general knowledge graph. Its
job is to give common ingredient strings a stable identity without making capture, inventory
or future recipes depend on perfect ontology coverage.

The long-term strategy is to preserve stable concept IDs and add capabilities around them:
evidence-driven aliases and relationships in P1, then retailer product mappings and
packaging in P2. Embeddings, a search index and richer graph infrastructure are optional
projections behind ports, introduced only when measured misses or scale justify them.
DynamoDB remains the P0 system of record.

## 2. Context and current baseline

The service already has the right dependency direction:

```text
capture worker ─┐
                ├── ConceptResolver port ── taxonomy application ── repositories
inventory API ──┘
```

Current code has:

- `internal/ports/resolver.go` with `ConceptResolver.Resolve`, resolution outcomes and
  passthrough text;
- optional resolver fields on capture and inventory services;
- inventory support for `ConceptID` and `concept_candidate`;
- the capture worker seam where parsed item names can be resolved before a draft is saved;
- one DynamoDB table and one GSI, sufficient for the P0 access patterns;
- `EventSink` and `MeterSink`; and
- a `GET /v1/admin/taxonomy/queue` OpenAPI path and HTTP stub.

It does not yet have taxonomy domain entities, repositories, a resolver implementation,
seed data, worker resolution, correction detection, admin mutations or taxonomy metrics.
Inventory currently performs separate text similarity for household dedup. That remains an
inventory concern: taxonomy answers “what ingredient is this?”, while inventory answers
“does this household already have this item?”.

## 3. Decision drivers

1. **Prefer a miss to a wrong match.** Free text must always survive unchanged.
2. **Make corrections useful immediately and locally.** One household's correction must
   never alter another household's result.
3. **Keep P0 cheap and operable.** A few thousand aliases do not require a vector database.
4. **Keep future migration possible.** Concept identity must survive richer categories,
   products, embeddings and storage projections.
5. **Measure before adding intelligence.** The golden set and real miss distribution decide
   when another resolver stage is warranted.
6. **Separate ingredient identity from retailer products.** Retailer trees and SKUs map to
   Saamly concepts; they never define them.
7. **Keep safety claims out of inferred data.** Allergens and safety-relevant substitutions
   require verified sources and are not part of taxonomy lite.

## 4. Scope

### 4.1 P0a — taxonomy lite

- Ingredient concepts with stable opaque IDs.
- `en-ZA` aliases, including South African names and spelling variants.
- Two-level categories used for browsing and review, not behavioural inheritance.
- Forms and common units as descriptive hints only.
- Versioned normalisation.
- Household exact alias → global exact alias → global fuzzy cascade.
- Conservative auto-accept and candidate outcomes.
- A seed artifact targeting roughly 1,000 concepts, loaded idempotently. The first curated
  slice may be smaller; measured coverage, not a round number, is the release criterion.
- Worker-only proposal of provisional concepts for misses, in one LLM call per batch.
- Review queue and admin actions.
- Household-scoped correction write-back.
- Resolution and queue metrics.

### 4.2 Explicitly deferred

- Unit conversion or basket arithmetic.
- Perishability behaviour and “use first” automation.
- Substitute relationships.
- Automatic provisional/global-alias promotion.
- General multilingual localisation.
- Embedding retrieval or a vector store.
- Retailer categories, brands, products, packs, offers and SKUs.
- Nutrition, dietary suitability and allergen claims.
- A consumer taxonomy browse/search API.

RFC 005's assumed `GET /taxonomy/concepts?q=` autocomplete endpoint is superseded for P0:
manual entry remains free text and server-side resolution is best-effort. Reconsider a
consumer suggestion endpoint only when candidate-selection UX is intentionally designed;
the admin concept search in §11 is not a consumer contract.

## 5. Domain model and invariants

Place pure types and validation in `internal/domain/taxonomy`.

### 5.1 Concept

```go
type Concept struct {
    ID             string
    CanonicalName  string
    CategoryID     string
    State          ConceptState // provisional | canonical | merged
    Source         Source       // seed | llm_proposal | correction | retailer
    Forms          []string
    CommonUnits    []string     // labels only; not conversion claims
    MergedInto     string
    SeedVersion    string
    Version        int
    CreatedBy      string
    CreatedAt      time.Time
    UpdatedAt      time.Time
}
```

Binding invariants:

- IDs are opaque ULIDs and are never reused.
- Canonical concepts have a non-empty canonical name and valid category.
- A merged concept has exactly one `MergedInto` target.
- Merge targets must resolve to a canonical concept; merge chains are flattened on write.
- Concepts are never hard-deleted after use. A merge preserves the old ID as a redirect.
- `Version` is incremented by every admin mutation and used for optimistic concurrency.
- Forms and common units are hints attached to the ingredient concept in lite. They do not
  create variant identity and must not drive quantity or safety behaviour.

### 5.2 Alias

```go
type Alias struct {
    NormalisedText  string
    DisplayText     string
    ConceptID       string
    Locale          string // en-ZA in P0
    Scope           AliasScope // global | household
    HouseholdID     string
    Source           Source
    Weight           float64
    ReplacesConceptID string
    NormaliserVersion string
    CreatedAt        time.Time
    UpdatedAt        time.Time
}
```

Binding invariants:

- A household alias requires `HouseholdID`; a global alias must not contain one.
- Household aliases are only read in that household and always precede global aliases.
- A household correction never lowers or rewrites a global alias. It records a preferred
  target and, where known, the concept it replaces for that household.
- An alias points to a canonical concept or follows a merged redirect before returning.
- Global exact aliases should have one effective target. Conflicts are review items, not
  nondeterministic results.
- Weight affects fuzzy ranking. It cannot make a result cross the acceptance threshold if
  the lexical score is below the threshold.

### 5.3 Category

P0 categories are a controlled two-level tree:

```go
type Category struct {
    ID        string
    Name      string
    ParentID  string
    SortOrder int
    Version   int
}
```

Categories organise review and later planning rules. They do not imply allergens,
substitutability, storage life or unit conversions.

### 5.4 Review item and evidence

```go
type ReviewItem struct {
    ID              string
    ConceptID       string
    Status          ReviewStatus // pending | promoted | merged | rejected
    ProposalKey     string
    SampleStrings   []string
    Occurrences     int
    HouseholdCount  int
    FirstSeenAt     time.Time
    LastSeenAt      time.Time
    DecidedAt       *time.Time
    DecidedBy       string
    DecisionReason  string
    Version         int
}
```

`ProposalKey` is a deterministic fingerprint of normalised proposed name and category.
It prevents duplicate provisional concepts under concurrent worker batches. Sample strings
are capped, deduplicated and removed after 90 days; counts and the decision remain.

Rejected proposal keys are retained as suppression records. A repeated miss increments
evidence but does not repeatedly spend an LLM call or recreate the same concept.

## 6. Normalisation

Normalisation is deterministic, locale-aware and versioned. `normaliser-v1`:

1. trim and lowercase;
2. Unicode-decompose and remove diacritics;
3. replace punctuation with spaces while preserving letters and digits;
4. remove balanced parenthetical text from a matching variant only when it is an allowlisted
   preparation or unit note; identity-bearing or unknown text remains, while the full
   original is always retained as `PassthroughText`;
5. collapse whitespace; and
6. generate a conservative singular matching variant.

Singularisation never rewrites stored user text. It only creates a second lookup key, and an
exact result is possible only if that key exists as an alias. Irregular and dangerous forms
belong in the seed aliases; taxonomy lite does not ship a general English stemmer.

Every alias stores the normaliser version that produced its key. A future normaliser is
rolled out by dual-writing old and new keys, rebuilding the alias projection, evaluating
both against the golden set, then switching reads. Existing records are not silently
reinterpreted.

## 7. Resolution

### 7.1 Port evolution

Keep the existing single-item port for API use and introduce an optional batch port for the
worker:

```go
type ResolutionRequest struct {
    Text        string
    HouseholdID string
    Context     string // inventory_manual | inventory_scan | recipe_import
}

type ConceptResolver interface {
    Resolve(ctx context.Context, text, householdID string) (Resolution, error)
}

type BatchConceptResolver interface {
    ResolveBatch(
        ctx context.Context,
        requests []ResolutionRequest,
    ) ([]Resolution, error)
}
```

The split avoids forcing worker-only proposal behaviour into user-facing API calls. Capture
depends on both interfaces; inventory depends only on `ConceptResolver`.

Use `taxonomy.ConceptCandidate` as the canonical internal candidate type. Remove the
duplicate inventory-domain candidate shape and have the resolver port refer to the taxonomy
domain type; HTTP and draft payloads map it to `id`, `name` and `score` at their boundaries.
This prevents three structurally similar candidate types from drifting.

Add these fields to `Resolution`:

- `CanonicalName` for denormalised display;
- `ResolverRevision` (for example `normaliser-v1+lexical-v1+seed-2026-08`);
- `MatchedAlias` for internal diagnostics only; and
- candidate names and scores when the outcome is `candidates`.

`PassthroughText` is always the trimmed original text, including on repository or resolver
failure.

### 7.2 Deterministic cascade

For one request:

1. **Household exact alias.** Return immediately. This is the correction fast path.
2. **Global exact alias.** Follow merge redirects and return `exact`.
3. **Global fuzzy candidates.** Score the immutable in-process global alias snapshot.
4. **Miss.** Return the original text and no concept.

Fuzzy scoring uses a documented combination of rune-aware Levenshtein ratio and trigram
Dice similarity. Token equality receives a small boost; alias weight only breaks close
ties. Defaults:

- top score `>= 0.90` **and** margin to runner-up `>= 0.08` → `fuzzy_accept`;
- top score `>= 0.60` but below either acceptance condition → up to five `candidates`;
- otherwise → `miss`.

The runner-up margin is binding. It prevents two plausible neighbours from becoming an
automatic decision merely because both score highly. Thresholds are configuration, but a
change is a resolver revision and must pass the golden set before deployment.

Household aliases are exact-only in P0. A typo in a household-specific name falls through
to the global resolver instead of fuzzy-searching private aliases across households.

Repository errors are distinguishable from semantic misses in logs, but callers receive a
conservative miss result plus the error. Inventory and capture continue with free text.

### 7.3 Batch behaviour and proposals

`ResolveBatch`:

1. deduplicates requests by `(household, normalised_text)`;
2. runs the deterministic cascade for all unique strings;
3. groups `miss` strings into one proposal request per capture job; ambiguous candidate
   outcomes are not proposed as new concepts;
4. checks proposed concepts against canonical aliases, merged concepts, pending proposal
   keys and rejection suppressions;
5. creates or updates provisional concept/review records idempotently; and
6. returns the deterministic results for the current draft.

A newly proposed concept is **not** silently attached to the current consumer draft.
Until promoted or explicitly selected during review, the outcome remains a miss and the
original text is shown. This protects P0 from converting an LLM suggestion into a product
fact.

The proposal adapter is a narrow `ConceptProposer` port implemented using the configured
LLM adapter and schema-locked prompts. It returns proposals plus token usage. The taxonomy
application service owns deduplication, persistence, events and `MeterSink` accounting.
P0 records proposal spend under the existing `CapabilityLLMOther`; add a dedicated meter
capability only if reporting needs to separate taxonomy from other LLM work. There is never
one LLM call per ingredient.

If the proposer is unavailable, deterministic results still return and processing
continues. Proposal failure cannot fail a capture draft.

## 8. Persistence and cache

Add `ConceptRepository`, `AliasRepository` and `ReviewRepository` ports in
`internal/ports/repository.go`. Ports express domain access patterns, never DynamoDB keys.

Indicative single-table records:

```text
PK=CONCEPT#<id>             SK=META
PK=CONCEPT#<id>             SK=REVIEW

PK=TAXONOMY#GLOBAL          SK=CATEGORY#<id>
PK=TAXONOMY#GLOBAL          SK=CONCEPT#<normalisedName>#<conceptId>
PK=TAXONOMY#GLOBAL          SK=ALIAS#<locale>#<norm>#<conceptId>

PK=HOUSE#<householdId>      SK=TAXALIAS#<locale>#<norm>

PK=TAXONOMY#QUEUE#pending   SK=<firstSeen>#<reviewId>
PK=TAXONOMY#DECISION        SK=<decidedAt>#<reviewId>
PK=TAXONOMY#SUPPRESS        SK=<proposalKey>
```

Concept metadata is duplicated on concept and alias projections where needed for listing
and resolution (`canonicalName`, `state`, `weight`, `version`). This lets one partition
query build the P0 resolver snapshot and supports prefix-based admin concept search without
scanning the table or adding a GSI. Admin detail reads use the source records.

Review status changes use a transaction: delete the old queue projection, update the
concept/review source records and write the decision projection. Alias additions and merge
redirects update source and projections in the same transaction where DynamoDB limits
allow it. Large alias rewrites are idempotent, resumable batches.

### 8.1 Resolver cache

- Global categories, concepts needed for resolution and aliases load into one immutable
  snapshot at process start.
- Refresh builds a complete replacement and atomically swaps it; readers never see a
  partial index.
- Default refresh TTL: five minutes with jitter.
- A successful admin mutation increments a global revision record. A process compares the
  revision during refresh, avoiding a rebuild when nothing changed.
- Household exact aliases are fetched by their direct key and may use a bounded one-minute
  cache. Cache keys always include household ID.
- A cold or failed cache refresh keeps the last good snapshot. If no snapshot exists, the
  resolver returns misses rather than failing startup.

At P0 scale the global partition is read rarely and writes are administrative. If it
becomes hot, §14 replaces this projection without changing domain ports or IDs.

## 9. Seed data and release process

Store source-controlled seed artifacts under:

```text
data/taxonomy/
  seed-2026-08.json
  categories-2026-08.json
  schema.json
  golden-set.json
  README.md
```

The generator may use an LLM, but generated output is not the production source of truth.
The checked-in artifact is.

`cmd/taxonomy-seed` (or an equivalent maintained CLI subcommand):

1. validates JSON schema, unique IDs/names and referential integrity;
2. normalises every alias and rejects collisions with different targets;
3. validates merge targets and category depth;
4. rejects allergen, conversion or safety fields;
5. prints a dry-run diff by default;
6. upserts by stable ID and `seed_version`; and
7. refuses destructive removal unless an explicit merge/deprecation is present.

Seed loading is repeatable in memory, local DynamoDB and deployed environments. Seed IDs
are generated once in the artifact, never regenerated per environment.

Release policy:

- manually review at least 100 representative concepts, weighted toward local terms;
- run the golden set and report exact, fuzzy accept, candidate, miss and false-accept rates;
- begin with the validated high-confidence slice if the full target is not ready;
- after real imports, rank missed strings by frequency and curate the concepts covering the
  most usage first; and
- never hold P0 for obscure tail coverage.

## 10. Capture, inventory and correction integration

### 10.1 Capture worker

After LLM payload schema validation and before `SaveProcessed`, the capture worker extracts:

- `items[].name_text` for `inventory_scan`; and
- `ingredients[].name_text` for `recipe_import`.

It calls `ResolveBatch`, then writes accepted exact/fuzzy results as:

```json
{
  "concept_candidate": {
    "id": "01...",
    "name": "Coriander",
    "score": 1.0,
    "outcome": "exact",
    "resolver_revision": "normaliser-v1+lexical-v1+seed-2026-08"
  }
}
```

Candidate-only and miss outcomes do not populate `concept_candidate`. Candidate lists may
be retained in the draft for confirm UI suggestions, but they are advisory and never
applied without selection.

### 10.2 Inventory API and apply

Wire the deterministic resolver into both API and worker inventory services.

- Manual add/edit resolves supplied free text when `concept_id` is absent.
- A supplied `concept_id` must exist and resolve through any merge redirect.
- `ApplyDraft` continues concept-first household dedup, then text fallback.
- `DisplayName` remains denormalised at write time. Taxonomy renames do not unexpectedly
  rename a household's list.
- When inventory `UpdateItem` changes `ConceptID`, the DynamoDB repository must
  transactionally delete the old `HOUSE#.../CONCEPT#...` pointer and write the new one.
  The current DynamoDB adapter does not mirror the memory adapter here and can otherwise
  return stale `GetByConcept` results.
- If taxonomy is unavailable, inventory behaves exactly as it does today with free text.

### 10.3 Correction write-back

Before `ConfirmDraft` overwrites the processed payload, compare original and edited rows by
a stable row key. The current array-position comparison is insufficient once users can
remove or reorder rows, so capture payload rows gain an internal `candidate_key` generated
by the worker.

A correction exists when:

- the selected concept changes;
- an accepted concept is removed while the ingredient text remains; or
- edited text deterministically resolves to a different concept.

Capture calls a taxonomy `ApplyCorrections` use case synchronously with an idempotency key
`<draft_id>#<candidate_key>#<correction_revision>`. For a known replacement it:

1. writes a household alias from the original normalised string to the corrected concept;
2. records `ReplacesConceptID` when known;
3. updates review evidence for a possible future global change; and
4. emits `taxonomy.correction_applied`.

If corrected text still has no concept, preserve it as free text and record miss evidence;
do not invent a target alias.

Inventory `UpdateItem` calls the same use case when a name edit changes or removes a prior
concept match. Its idempotency key is `<item_id>#<expected_updated_at>#<normalised_text>`.
Manual add is an observation, not a correction, because there is no prior accepted match.

The write is idempotent. A transient taxonomy write failure fails confirmation before the
draft status changes, so the user can retry without losing the correction. Event append
failure is still non-blocking under the feedback-module rule.

### 10.4 Household deletion

Household-scoped aliases and correction idempotency records are household data. Add a
taxonomy `PurgeHook` to the existing household purge registry in both API/worker
composition. Purge queries `PK=HOUSE#<household_id>` and removes `TAXALIAS#...` records;
global concepts, aggregate counts and decision records remain, but retained evidence must
not identify the deleted household. Any bounded household cache entries are evicted.

## 11. Admin API and review workflow

The current queue-only contract is not enough to operate the queue: merge needs concept
search, and decisions need mutations. Extend `api/admin.yaml` before handlers:

| Method | Path | Purpose |
|---|---|---|
| GET | `/taxonomy/queue?status=&cursor=&limit=` | Paginated review queue, oldest first |
| GET | `/taxonomy/concepts?q=&state=&cursor=&limit=` | Search merge targets and inspect concepts |
| GET | `/taxonomy/concepts/{id}` | Concept, aliases, evidence and redirects |
| POST | `/taxonomy/concepts/{id}/promote` | Provisional → canonical |
| POST | `/taxonomy/concepts/{id}/merge` | Merge into canonical target |
| POST | `/taxonomy/concepts/{id}/reject` | Reject and suppress proposal key |
| PUT | `/taxonomy/concepts/{id}` | Edit name, category, forms and unit hints |
| POST | `/taxonomy/concepts/{id}/aliases` | Add a global alias |

Mutations require:

- Entra JWT plus the existing allowlist;
- `expected_version` for optimistic concurrency;
- actor identity from middleware;
- a plain-text reason for merge/reject;
- idempotent handling of a repeated identical decision; and
- an audit event with identifiers, never raw auth claims.

Response conflicts:

- stale version or already-decided item → `409 conflict`;
- unknown concept/target → `404 not_found`;
- alias collision or invalid merge → `422 invalid_request`.

Promote validates category and aliases. Merge keeps the source concept as a redirect,
repoints global aliases, updates queue status and never rewrites historical inventory.
Reject leaves no resolvable concept but retains the suppression and decision record.

Admin UI can remain functional and plain. A queue row needs proposed name/category,
aliases, sample evidence, occurrence/household counts and actions. Merge target search is
the only extra interaction required for P0.

## 12. Events, metrics and privacy

Register payloads in `feedback.md` before implementation:

| Event | Required payload |
|---|---|
| `taxonomy.resolution_recorded` | outcome, confidence bucket, context, resolver_revision, concept_id? |
| `taxonomy.resolution_missed` | missed_string, context, resolver_revision |
| `taxonomy.concept_proposed` | concept_id, source, queue_size |
| `taxonomy.concept_promoted` | concept_id, source, queue_size |
| `taxonomy.concept_merged` | concept_id, into_concept_id, queue_size |
| `taxonomy.concept_rejected` | concept_id, queue_size |
| `taxonomy.alias_added` | concept_id, scope, source |
| `taxonomy.correction_applied` | from_concept_id?, to_concept_id, corrected_string, scope |

`resolution_recorded` contains no ingredient text. Only misses and corrections carry content,
because the learning loop requires it. Admin event views apply the feedback module's
redaction and access rules. Do not log raw strings separately at info level.

P0 dashboards derive:

- auto-resolution rate = exact + fuzzy accept / resolution attempts;
- false-accept proxy = corrections / accepted resolutions;
- outcome and correction rates by resolver revision and context;
- seed coverage among the first 50 imports;
- queue size and age of oldest pending item;
- proposal merge rate; and
- proposal LLM count, tokens and estimated cost.

Release gates remain `>= 85%` auto-resolution and `< 10%` corrections, but must be read
together. A resolver cannot improve the first metric by making aggressive wrong matches.
The golden-set false-accept count is a hard regression check.

## 13. Configuration

Add:

| Environment variable | Default | Purpose |
|---|---:|---|
| `SAAMLY_TAXONOMY_ENABLED` | `true` | Allows conservative operational disable |
| `SAAMLY_TAXONOMY_SEED_VERSION` | checked-in current | Expected deployed seed |
| `SAAMLY_TAXONOMY_CACHE_TTL` | `5m` | Global snapshot refresh |
| `SAAMLY_TAXONOMY_HOUSEHOLD_CACHE_TTL` | `1m` | Bounded household exact cache |
| `SAAMLY_TAXONOMY_FUZZY_ACCEPT` | `0.90` | Minimum top score |
| `SAAMLY_TAXONOMY_FUZZY_CANDIDATE` | `0.60` | Minimum candidate score |
| `SAAMLY_TAXONOMY_FUZZY_MARGIN` | `0.08` | Required top-to-runner-up margin |
| `SAAMLY_TAXONOMY_PROPOSALS_ENABLED` | `true` in dev/prod | Worker proposal switch |
| `SAAMLY_TAXONOMY_REVIEW_LIMIT` | `50` | Admin page size |

Validate all scores in `[0,1]`, candidate threshold `<` accept threshold, margin `> 0`, and
TTLs positive. Disabling taxonomy yields pass-through misses; it does not disable capture
or inventory.

## 14. Evolution into the full system

Evolution is additive around stable ingredient concepts. It is not a rewrite of taxonomy
lite.

### 14.1 P1 — evidence-driven ingredient intelligence

Add only after P0 correction data exists:

1. **Alias promotion.** Promote a household correction globally only with cross-household
   evidence, no contradictory corrections and reviewable provenance. The initial policy
   follows the module spec (`>= 3` observations, `>= 2` households, zero corrections) but
   runs in shadow mode before automatic writes.
2. **Provisional auto-promotion.** Apply the same shadow-first policy. Keep every decision
   reversible through merge.
3. **Perishability classes.** Model as sourced concept attributes with explicit confidence
   and update history. They can inform “use first” but never manufacture expiry dates.
4. **Substitute edges.** Introduce typed, directional relationships with context
   (`culinary`, form constraints, strength) and provenance. Safety-relevant substitution is
   excluded unless verified.
5. **Units and forms.** Promote free-form hints into stable `Unit` and `Form` IDs. Conversion
   is a separate verified assertion (`from`, `to`, factor/range, conditions, source,
   verified_by`, version), not a property inferred from labels.
6. **Additional languages.** Add locale-tagged aliases based on actual household demand.
   Concept IDs remain language-neutral; canonical display names may become locale-specific
   projections.

Relationships and assertions use their own IDs and provenance. Do not keep growing the
`Concept` record into an unversioned blob.

### 14.2 Embeddings — a measured resolver stage, not a default architecture

Add an embedding candidate generator only if dogfood evidence shows that:

- deterministic auto-resolution remains below the gate after high-frequency curation;
- a material share of misses are semantic paraphrases rather than missing concepts;
- alias volume or latency makes the in-process lexical index unsuitable; or
- retailer descriptions require semantic candidate generation.

The first embedding use is **candidate retrieval/reranking**, not automatic acceptance:

```text
household exact → global exact → lexical candidates
                → embedding candidates → deterministic reranker → candidate/miss
```

It implements a `CandidateGenerator` port. The embedding index is a rebuildable projection
of concept/alias source records and stores concept IDs plus resolver revision. A model or
index change is evaluated offline on the golden set and shadowed in production before it
can alter outcomes. No embedding-only match is auto-accepted until measured precision
meets or exceeds the lexical stage at the intended threshold.

Possible stores include OpenSearch vector search, Aurora PostgreSQL with `pgvector`, or a
managed vector service. The choice is deferred until corpus size, query volume, regional
availability and cost are known.

### 14.3 P2 — retailer and SKU mapping

Do not turn ingredient concepts into retailer SKUs. Add separate entities:

```text
IngredientConcept  ← ProductMapping → Product
Product            ← PackVariant    → RetailerOffer
RetailerCategory   ← CategoryMap    → SaamlyCategory
```

- `Product` represents a branded or generic purchasable product.
- `PackVariant` represents size, count, form and packaging.
- `RetailerOffer` is retailer-specific, price-bearing and time-bounded.
- `ProductMapping` links a product to one or more ingredient concepts with quantity share,
  confidence, provenance and validity period.
- `CategoryMap` maps retailer trees onto Saamly categories; retailer categories never
  become the internal hierarchy.

SKU matching uses exact identifiers first (GTIN/barcode, retailer product ID), then
deterministic attributes, then semantic candidates for review. Historical mappings are
versioned because retailer descriptions and packs change.

This separation lets “fresh coriander bunch 50g” map to coriander without making brand,
retailer, pack size or current price properties of coriander.

### 14.4 P2+ — verified knowledge and richer graph

Allergens, nutrition and safety assertions live in a verified assertion model:

```text
subject_id, predicate, value, source_ref, source_version,
verified_by, verified_at, valid_from, valid_to
```

Unknown is distinct from false. LLM output may suggest a review candidate but can never
create an active safety assertion.

If richer planning needs more than a two-level category tree, add typed relationships
(`is_a`, `part_of`, `substitutes_for`, `derived_from`) and materialised projections. Keep
the original concept IDs and merge redirects. Graph storage is considered only when graph
queries become a measured bottleneck; DynamoDB source records plus purpose-built read
models are sufficient before then.

### 14.5 Storage evolution

The source-of-truth and read model are deliberately separate:

- DynamoDB source records own identity, review decisions, provenance and redirects.
- P0 in-memory lexical indexes are rebuildable read models.
- Later search/vector/graph stores consume DynamoDB Streams or an explicit rebuild job.
- Resolver ports return the same outcome shape regardless of candidate source.
- A projection outage falls back to exact/source reads or free-text misses.

This avoids a data migration every time the retrieval method changes.

## 15. Delivery slices

Implement as six independently demoable slices:

| Slice | Delivers | Demo |
|---|---|---|
| **A — Domain, ports and seed contract** | Domain types/invariants; repository and batch/proposer ports; normaliser; seed schema/artifact skeleton; golden-set harness; OpenAPI admin expansion | Validate seed and run normaliser/golden fixtures locally |
| **B — Memory resolver** | Memory repositories; exact + fuzzy resolver; immutable cache; merge redirects; metrics events; unit tests | `Resolve("dhania")` and typo/candidate/miss cases with passthrough |
| **C — DynamoDB and seed loader** | Key builders; repositories/projections; conditional versions; idempotent seed CLI; local integration tests | Load seed into local DynamoDB, restart API, resolve aliases |
| **D — Capture and inventory wiring** | Worker `ResolveBatch`; draft candidate metadata; API/manual resolver; concept validation; correction comparison and idempotent household write-back; DynamoDB item-concept pointer fix; taxonomy purge hook | Scan “dhania”, confirm as “coriander”, then resolve the original text for that household |
| **E — Proposals and review API** | Schema-locked batched proposer, dedup/suppression, metering; queue/search/detail/promote/merge/reject/edit/alias handlers; audit events | Miss creates one queue item; promote/merge changes the next resolution after cache refresh |
| **F — Admin UI, dashboards and hardening** | Plain review UI in `saamly-admin`; metrics; seed coverage report; deployed cache/revision checks; retention | Work queue end to end and display P0 gates by resolver revision |

Slice D is valuable without LLM proposals or an admin UI and should not wait for them.
Slice F does not gate basic resolution, but the review mutation API in E is required before
proposals are enabled outside a controlled environment.

### Exit criteria

Taxonomy lite is P0a-complete when:

- seed loading is repeatable and stable across environments;
- exact, fuzzy, candidate and miss outcomes pass the golden set;
- false accepts do not regress from the accepted baseline;
- capture and manual inventory attach concepts while preserving free text;
- a corrected match becomes household-specific knowledge on retry-safe confirmation;
- inventory concept edits update DynamoDB pointer rows and household purge removes scoped
  taxonomy data;
- provisional proposals are batched, metered, deduplicated and reviewable;
- promote, merge and reject change subsequent resolver behaviour;
- queue age, auto-resolution and correction metrics are visible by resolver revision;
- one local DynamoDB integration test covers seed → resolve → correct → merge; and
- `make test` and OpenAPI lint remain green.

## 16. Testing strategy

| Layer | Coverage |
|---|---|
| Domain | State transitions, merge-cycle prevention, alias scope, version checks, evidence caps |
| Normaliser | Diacritics, punctuation, parentheticals, plural variants, SA terms, preservation of passthrough text |
| Resolver | Household precedence, exact conflicts, merged redirects, fuzzy thresholds/margins, deterministic ordering, repository failure fallback |
| Golden set | Accepted matches, ambiguous candidate sets, required misses and adversarial near-neighbours |
| Application | Batch dedup, one proposal call, suppression, metering, idempotent corrections and review decisions |
| DynamoDB | Projection key helpers, conditional versions, transactional queue moves, cache rebuild query |
| Capture/inventory | Worker enrichment, concept-first dedup, edited-row correction detection, taxonomy-disabled fallback |
| HTTP | Entra/allowlist enforcement, pagination, conflict/problem shapes, audit events |
| End to end | Seed → scan/parse → resolve → confirm correction → household precedence → admin merge |

The golden set has three classes:

1. **must match** — known aliases and realistic spelling errors;
2. **may suggest, must not accept** — ambiguous or related terms; and
3. **must miss** — semantic neighbours where a wrong match would be harmful.

Report precision separately for exact and fuzzy acceptance. Aggregate “accuracy” can hide
false accepts and is not sufficient.

## 17. Risks and resolved trade-offs

| Risk / trade-off | Decision |
|---|---|
| Starter set has systematic LLM errors | Checked-in artifact, schema validation, weighted review and golden set; generator output is not live truth |
| Fuzzy stage accepts semantic neighbours | High threshold plus runner-up margin and must-miss fixtures; no candidate is silently applied |
| Global alias partition grows hot | Reads are cached and writes rare in P0; replace the projection behind ports when measured |
| One correction corrupts global matching | Household alias precedence; global evidence only, no direct global mutation |
| Provisional concepts explode | Proposal fingerprint, canonical/pending dedup, rejection suppression and one call per batch |
| Concept rename changes household language | Display names are denormalised; aliases preserve local wording |
| Merge breaks historical references | Redirect forever; flatten chains; consumers resolve old IDs |
| Taxonomy outage blocks capture | Passthrough miss and optional wiring; proposal failures never fail processing |
| Correction write failure loses learning | Correction is idempotent and precedes confirmation status transition |
| Units/forms are mistaken for verified semantics | Lite fields are labels only; conversions and safety live in sourced assertion models |
| Embeddings become an early infrastructure project | Objective adoption triggers, shadow evaluation and a replaceable candidate-generator port |
| Retailer model pollutes ingredient identity | Separate Product, PackVariant, Offer and mapping entities |

## 18. Open questions

These do not block slices A–C unless noted:

1. **Seed activation:** activate every mechanically valid generated concept, or require
   explicit review per concept? Proposal: activate the manually reviewed/high-confidence
   slice first, then add validated batches; do not make 1,000 a gate.
2. **Fuzzy implementation:** custom small implementation or a maintained Go library?
   Proposal: implement the documented metrics in-package; the algorithm is small and the
   golden set, not a library choice, defines behaviour.
3. **Correction failure policy:** retry confirmation on taxonomy persistence failure or
   confirm and record an operational miss? This RFC proposes retry-before-status-change
   because the user explicitly supplied knowledge, while resolver/proposer failures remain
   non-blocking.
4. **Candidate UI in P0:** expose ambiguous candidates during confirm or retain them only
   for later? Proposal: return them in the draft contract but keep UI optional; free-text
   editing already provides a correction path.
5. **Admin repository:** `saamly-admin` does not yet exist. Proposal: ship review APIs and a
   CLI/curl workflow in slice E, then create the minimal admin UI in F; do not delay
   deterministic resolution.

## 19. Out-of-scope reminders

Do not add to taxonomy-lite implementation PRs:

- a vector database, OpenSearch cluster or graph database;
- live web search or third-party food datasets without provenance/licensing review;
- inferred allergens, nutrition or dietary-safety claims;
- unit conversion factors;
- recipe/planner/shopping behaviour beyond storing stable concept identity;
- retailer scraping, prices, availability or basket integration;
- automatic global learning before shadow evidence is reviewed; or
- mandatory taxonomy calls on consumer read paths.

## 20. Implementation checklist

1. Accept this RFC and register the new event payloads in `feedback.md`.
2. Expand `api/admin.yaml`; keep consumer contract additions limited to candidate metadata.
3. Deliver slices A–F in order, allowing D to proceed in parallel with admin work.
4. Wire one resolver instance into API composition and one into worker composition.
5. Check in seed and golden artifacts with versions, review notes and repeatable loader.
6. Record the initial golden-set baseline before changing fuzzy thresholds.
7. Run the first-50-import coverage report and use it to prioritise curation.
8. Revisit embeddings only at the §14.2 triggers, with an explicit follow-up RFC naming the
   measured problem, evaluation result, regional availability and cost.
