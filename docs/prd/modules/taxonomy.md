# Module spec — `taxonomy`

Version 0.1 · July 2026 · Status: Draft · Phase: P0a
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.4 (§3.3, D4, open Q3) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` (§10, §14) · Depends on: `core.md`

## 1. Purpose and pillar

`taxonomy` is the ingredient ontology and its resolution service — the shared language that lets a recipe's "1 bunch dhania", a fridge photo's "coriander" and a store's "Fresh Coriander Bunch 50g" mean the same thing. It serves all three pillars: recipe parsing needs it to structure ingredients (pillar 2), the planner needs it to spot reuse across the week (pillar 1), and SKU matching later maps retailer products onto it (pillar 3). Its operating principle: **accuracy comes from making errors cheap, not from upfront curation** — deterministic-first resolution, provisional concept states with a promotion gate, free-text fallback so nothing ever blocks, and every user correction written back as knowledge.

## 2. Users and jobs

| User | Jobs |
|---|---|
| `capture` worker (internal) | Resolve ingredient strings to concept candidates during draft processing, in batches, without blocking drafts |
| `inventory` / `recipes` / `meal plans` / `shopping` (internal) | Share concept identity for reuse, deduction, aggregation and (later) matching |
| Team (via `admin`) | Work the review queue: promote, merge or reject provisional concepts; fix aliases and categories |
| Household members (indirect) | Correct a wrong match once and never see that mistake again |

## 3. Scope: Now / Next / Later

- **Now (P0a):** LLM-seeded starter set (~1,000 SA concepts with aliases, shallow 2-level categories, forms, common units); resolution cascade — exact alias → in-process fuzzy → LLM-propose-new (embedding stage deferred, PRD open Q3 / RFC §14); provisional → canonical promotion via the review queue in `admin`; correction write-back as household-scoped aliases; free-text fallback.
- **Next (P1):** unit conversions; perishability classes (powers "use first"); substitute relationships (powers the planner); auto-promotion of provisional concepts after cross-household evidence; global alias promotion from household corrections.
- **Later (P2+):** pack-size norms; retailer taxonomy mapping (map their trees, never adopt them); SKU-matching support; verified allergen data.
- **Never fully automated (hard rule):** allergen tags, safety-relevant substitutions, and unit conversions that feed basket math — verified sources only, never LLM-generated.

## 4. Flows

### 4.1 Seeding (one-time, repeatable)

1. LLM generates the starter ontology from a schema-locked prompt: ~1,000 concepts covering South African kitchens — canonical name, category, aliases across local usage (dhania/coriander, brinjal/aubergine/eggplant, gem squash, butternut, wors/boerewors, mielie meal/maize meal/pap, samp, chakalaka, atchar, amasi/maas, naartjie, rusks), forms (fresh/frozen/dried/canned/jarred), common units.
2. Validate schema and referential integrity mechanically; spot-check ~100 concepts by hand for systematic errors; fix the prompt, regenerate the bad slices.
3. Load as `state=canonical, source=seed`. Once real imports start, harvest actual ingredient strings (frequency-ordered) to see which ~300 concepts cover 90% of usage — that list gets curation priority.

### 4.2 Resolution (runtime, hot path)

`Resolve(text)` — normalise (lowercase, strip diacritics/parentheticals, singularise) → cascade:

1. **Exact alias** (incl. household-scoped aliases from prior corrections) → accept.
2. **In-process fuzzy** over the cached alias index (trigram/Levenshtein, RFC §10): score ≥0.9 → accept with confidence; 0.6–0.9 → return candidate list, no decision; <0.6 → miss.
3. **Embedding stage** — deferred (PRD open Q3); slots in as a new adapter behind the same port.
4. **LLM-propose-new** — *worker context only* (never the user-facing hot path): on a miss, propose a provisional concept (canonical name, category, aliases, evidence string), dedup-checked against existing concepts and pending queue items.

No match is infinitely better than a confident wrong match (D4): a miss returns the original text untouched, and consumers treat it conservatively.

### 4.3 Review queue (admin)

Provisional concepts accumulate evidence (sample strings, occurrence counts, households seen). Reviewer actions: **promote** → canonical; **merge** → alias onto an existing concept (proposer's future strings resolve there); **reject** → discard and suppress re-proposal of that string. The queue is a P0a dashboard in `admin`; its size is a health metric, and neglect shows up in the correction-rate gate.

### 4.4 Correction write-back

User fixes a match during confirm ("that's not parsley, it's coriander") → `taxonomy.correction_applied` event → (a) the corrected string becomes a **household-scoped alias** of the right concept (source=correction, takes precedence for that household immediately); (b) the misleading alias's weight drops for that household; (c) cross-household evidence feeds the review queue for **global** alias changes (P1 auto-promotion). Household-scoped first is deliberate: one household's correction can never corrupt another's matching.

### 4.5 Auto-promotion (P1, designed now)

Provisional concept observed ≥3 times across ≥2 households with zero corrections → canonical automatically. Specified now so the evidence fields exist from the start; the automation ships after the dogfood proves review-queue precision.

## 5. Data owned

| Entity | Key fields | Key pattern (indicative) | Retention / notes |
|---|---|---|---|
| Concept | id, canonical_name, category_id, state (provisional/canonical/merged), forms[], units[], perishability? (P1), substitutes? (P1), allergen_tags? (verified only), source (seed/llm_proposal/correction/retailer), created_by, created_at, merged_into? | `PK=CONCEPT#<id> SK=META` | Permanent — core asset; merges redirect, never delete |
| Alias | normalised_text, display_text, concept_id, locale (en-ZA first), scope (global/household), household_id?, source (seed/correction/observation/retailer), weight | `PK=CONCEPT#<id> SK=ALIAS#<norm>`; `GSI1PK=ALIAS#<norm> GSI1SK=CONCEPT#<id>` | Permanent; weights decay only via correction events |
| Category | id, name, parent_id? | cached whole (2 levels, <100 rows) | Permanent |
| ReviewQueueItem | concept_id, proposed_payload, evidence {sample_strings[], occurrences, household_count}, status (pending/promoted/merged/rejected), first_seen, last_seen | `PK=QUEUE#<status> SK=<concept_id>` | Evidence samples expire 90d; decision record permanent |

The alias index is small (a few thousand rows at P0); the resolver caches it in-process with a short TTL refresh (RFC §10). Allergen tags carry `verified_by` + `source_ref` — without them the field stays empty (hard rule, §3).

## 6. API boundary

**Internal (the `ConceptResolver` port consumed by `capture` and others):**

```
Resolve(text, household_context?) → {
  outcome: exact | fuzzy_accept | candidates | miss,
  concept_id?, candidates[]?, confidence,
  passthrough_text           # always present (D4)
}
ResolveBatch(texts[], worker_context) → per-string results + at most one
  batched LLM-proposal call   # cost control: proposals never run per-string
```

Resolution never blocks and never mutates on the hot path; provisional creation happens only in worker context with `MeterSink` accounting (D7).

**Admin surface** (Entra JWT, allowlisted, mutations logged): `GET /v1/admin/taxonomy/queue` · `POST /v1/admin/taxonomy/concepts/:id/promote` · `/merge` · `/reject` · `PUT /v1/admin/taxonomy/concepts/:id` · `POST /v1/admin/taxonomy/concepts/:id/aliases`.

**Consumer surface:** none at P0 — modules denormalise display names onto their own entities at write time, so taxonomy reads stay off user requests.

## 7. Dependencies

- **Consumes:** `core` (household context, provenance, `EventSink`, `MeterSink`); the `llm` port (seeding + propose-new only — resolution itself is deterministic).
- **Consumed by:** `capture` (`ConceptResolver`, optional collaborator per `capture.md`), `inventory`, `recipes`, `meal plans`, `shopping`; later `retailer catalogs` (mapping target).
- **Events emitted:** `taxonomy.concept_proposed`, `taxonomy.concept_promoted`, `taxonomy.concept_merged`, `taxonomy.alias_added`, `taxonomy.resolution_missed` (with string + context, feeds queue and metrics), `taxonomy.correction_applied`.

## 8. Metrics

| Metric (PRD §8) | Source | P0 target |
|---|---|---|
| Auto-resolution rate | `Resolve` outcomes (exact + fuzzy_accept) / calls | **≥85% — P0a gate** |
| Correction rate | `taxonomy.correction_applied` / resolutions | **<10% — P0a gate** |
| Review queue trend | pending queue size over time | Shrinking toward zero by end of P0b |
| Seed coverage | seed hits / ingredient strings in first 50 imports | Measured; drives curation priority (~300 concepts ≈ 90% usage) |
| Merge rate | merges / promotions | Low; high rate signals proposal-time dedup is weak |
| LLM proposal cost | MeterSink | Within D7 budget; batched proposals only |

## 9. Risks

| Risk | Mitigation |
|---|---|
| LLM seed has systematic errors (wrong SA mappings, hallucinated synonyms) | Mechanical schema validation + ~100-concept human spot-check; regenerate bad slices; corrections repair the tail |
| Fuzzy false positives auto-accept (parsley ≠ coriander at 0.91) | Conservative auto-accept threshold; 0.6–0.9 returns candidates instead of deciding; correction write-back lowers misleading alias weights |
| Review queue becomes a neglected chore | Queue lives on the `admin` dashboard with a size metric; dogfood reviewers are the founders; P1 auto-promotion caps manual load |
| Concept explosion / near-duplicate concepts | Proposal-time dedup against concepts and pending queue; merge tooling with redirect (never delete) |
| Allergen data wrong → safety issue | Hard rule: verified sources only, `verified_by` required; field stays empty otherwise; planner treats absent tags as unknown, never as safe |
| One household's correction corrupts others' matching | Corrections are household-scoped until cross-household evidence promotes them (P1) |
| Language diversity (11 official languages) | en-ZA first per brand; local names absorbed organically as aliases via corrections and proposals — no decorative localisation (brand guardrail) |

## 10. Voice and labels

- Mostly internal-facing. Admin labels are plain: *Review queue* · *Promote* · *Merge* · *Reject* · *Aliases* · *Category*.
- Consumer-visible traces: concept names appear in "what you have" and lists — use household language ("coriander" if the household corrected to it), never internal IDs or latin/binomial names.
- Correction prompt per brand honesty: *Not this? Tell us what it was.* — one tap, no forms.
- Vocabulary per brand §7: *ingredient* or *product*, never "inventory item" or "SKU concept" in consumer copy.
