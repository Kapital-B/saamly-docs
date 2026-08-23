# Module spec — `capture`

Version 0.1 · July 2026 · Status: Draft · Phase: P0a
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.3 (§3.2, D2, D4, D6, D7) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` (§7, §9) · Depends on: `core.md`

## 1. Purpose and pillar

`capture` is the single intake pipeline for everything the household sends to Saamly (D2). Every input route — photo, pasted text, and later URL, share sheet and voice — feeds the same flow: accept the raw material, structure it automatically into a **draft**, attach provenance, expose uncertainty honestly, and ask for confirmation only where it could materially change a meal, a safety outcome or a basket. It serves pillar 2 (frictionless recipe intelligence) directly and pillars 1 and 3 by feeding `inventory`, `recipes` and eventually `retailer catalogs`. The interaction model is the product's soul: *capture naturally, structure automatically, confirm only when necessary.*

## 2. Users and jobs

| User | Jobs |
|---|---|
| Household member | Photograph the fridge/cupboards and get a checkable candidate list; paste a recipe from WhatsApp and get a structured draft; review and correct with minimal taps; trust that Saamly says "probably" when it means "probably" |
| `inventory` / `recipes` modules (internal) | Receive well-formed drafts with uncertainty flags; render confirmation using shared scaffolding; apply confirmed payloads with provenance |
| Team | Measure confirmation burden and parsing quality; replay drafts to improve prompts and parsing |

## 3. Scope: Now / Next / Later

- **Now (P0a/P0b):** photo intake (single or multi-photo scan session) via presigned S3 upload; pasted-text intake; async worker pipeline (SQS → vision/LLM parse → draft); draft inbox with poll-based status; shared confirm scaffolding with per-item accept/edit/reject; materiality flags; provenance on everything; idempotent processing; draft retention for the feedback loop.
- **Next (P0b–P1):** URL import with structured extraction; iOS share extension + Android intent ("Send to Saamly" — technical spike, RFC §4.2); voice capture; allergen cross-flags against member preferences at review time.
- **Later (P2+):** receipt and barcode intake; retailer-order import; bulk import; push notification when a draft is ready.

**Out of scope:** what drafts *mean* — candidate-item semantics belong to `inventory`, recipe semantics to `recipes`. `capture` owns the pipeline, the draft object and the confirm pattern; owning modules own payload schemas and apply logic. No module builds its own parser or confirmation UX (D2).

## 4. The draft object and lifecycle

### 4.1 Draft

| Field | Notes |
|---|---|
| id, household_id, created_by, created_at | standard `core` context |
| kind | `inventory_scan` · `recipe_import` (P0); extensible: receipt, barcode, order_import |
| source_type | photo / paste (P0); url / share / voice / manual later |
| source_ref | S3 key(s) or none for paste; never the raw text duplicated |
| provenance | `core` ProvenanceRecord envelope (D6) |
| status | `queued → processing → needs_review → confirmed` · `failed` · `discarded` |
| images[] | for scan sessions: N photos merged into one draft; optional per-item bbox for "we saw it here" |
| payload | kind-specific structure (§4.3), with `name_text` always populated — free-text fallback is mandatory (D4) |
| uncertainty | per-item/per-field confidence + **materiality** flag (§4.2) |
| interpretation_raw | unedited model output, retained for correction analysis |
| metering | capabilities invoked + cost, written to `core` MeterCounters (D7) |

### 4.2 Materiality — the confirm-only-what-matters rule

Every draft item/field carries confidence; every flag is classified:

- **Material** — could change a meal, a safety outcome or a basket: wrong ingredient identity, quantity that changes a purchase, anything intersecting a member allergy. Material flags block auto-accept; the user must resolve them.
- **Cosmetic** — wording, brand guesses, exact pack size the household will see anyway. Auto-accepted on confirm-all; editable but never demanded.

This classification — not raw confidence — decides what the review screen asks. A confident-but-material guess still gets shown; an uncertain-but-cosmetic one never interrupts. Honesty rule (brand §6): the UI always says *probably* when the system is unsure, and never invents quantities or freshness.

### 4.3 Payloads (P0)

- **`inventory_scan`:** `items[] { name_text, quantity_hint?, concept_candidate?, bbox?, confidence, materiality }`. Multi-photo sessions merge detections into one candidate list, deduplicated by normalised text/concept similarity; near-duplicates collapse, conflicts surface as one item with two notes.
- **`recipe_import`:** `{ title?, servings?, times?, ingredients[] { raw_text, quantity?, unit?, preparation?, concept_candidate?, confidence, materiality }, steps[], source_note? }`. Missing title or steps is not a failure — the draft is still useful and the gap is shown plainly.

`concept_candidate` comes from the `taxonomy` resolver when available (interface owned by `taxonomy.md`); capture treats it as an optional collaborator and functions fully without it (free-text fallback, D4).

### 4.4 Lifecycle rules

- **Idempotent processing:** SQS is at-least-once; the worker guards status transitions on `draft_id` + expected status. Client upload retries reuse the same draft via idempotency key.
- **Retry and discard:** `failed` drafts can be retried (same source); `discarded` drafts delete their S3 objects immediately and are excluded from the inbox but retained for analysis.
- **Retention:** raw uploads are S3 lifecycle-deleted after the retention window (default 30 days, configurable — PRD §7); drafts (payload + raw interpretation, no images) are retained at P0 for the correction/feedback loop; household purge (`core`) cascades to drafts and uploads.
- **Offline (D8):** photos taken offline queue in the app's outbox and upload when connectivity returns; the draft appears in the inbox when processing finishes.

## 5. Flows

### 5.1 Kitchen scan (P0a)

1. "Show Saamly what you have. A few quick photos are enough to get started." → camera flow (fridge, freezer, cupboards — light guidance, no forced structure).
2. Photos upload via presigned PUT → one draft per session → status polling.
3. Review screen: merged candidate list as editable chips. Material items first with reasons ("We probably saw yoghurt. Check the amount before we leave it off your list."). Per item: accept / edit / reject; one-tap "confirm all".
4. Confirm → `inventory` applies → items appear in "what you have" with confidence states.

**Edge cases:** photo unreadable (dark/blurry) → `failed` with human copy ("We couldn't read that photo — try again with the light on?") and source retained for retry; nothing detected → honest empty draft, offer manual add; same item scanned twice across sessions → `inventory` dedups at apply time.

### 5.2 Paste a recipe (P0a/P0b)

1. Paste sheet (from WhatsApp, email, notes) → `POST /capture/text` → draft.
2. "Recipe received. We're turning it into something you can plan and shop." → review when ready: title, ingredient list with quantities, steps; material uncertainties highlighted (e.g. "1 bunch coriander — how much is a bunch to you?").
3. Confirm → `recipes` applies → library entry with provenance.

### 5.3 Error and uncertain states

Worker down / SQS backlog → draft stays `queued`, inbox shows "Still working on it" — never silent; LLM provider error → `failed` + retry; partial parse (title but no steps) → `needs_review` with gaps shown plainly; household purged mid-processing → worker no-ops on missing context.

## 6. API boundary

Consumer surface (`/v1`, household context from `core` middleware):

| Endpoint | Purpose |
|---|---|
| `POST /capture/uploads` | Create upload session → presigned PUT URL(s) + draft_id (kind, content types, idempotency key) |
| `POST /capture/uploads/:id/complete` | Client finished PUT → enqueue processing |
| `POST /capture/text` | Pasted-text intake → draft (kind, text) |
| `GET /drafts?status=` | Draft inbox (default `needs_review`) |
| `GET /drafts/:id` | Full draft: payload, uncertainty, materiality, provenance, status |
| `POST /drafts/:id/confirm` | Confirm with edited payload → routed internally to the owning module's apply (by kind) |
| `POST /drafts/:id/discard` · `POST /drafts/:id/retry` | Discard (deletes uploads) / retry failed |

**Internal ports:** `BlobStore` (presigned PUT/GET, delete), `Queue` (enqueue parse job), `VisionParser`/`TextParser` behind the `llm` port (Nova Lite default, Nova Pro escalation on failed/low-confidence parses, Nova Micro for text — RFC §7), `ConceptResolver` (optional; owned by `taxonomy`). The worker is the only caller of the `llm` port at P0 and records every invocation to `MeterSink` (D7).

**Confidence semantics:** payloads always carry text (`name_text`/`raw_text`); structured enrichments (concept, quantity) are optional and confidence-tagged. Consumers must treat unmatched/unconfident data conservatively (D4) — no match is infinitely better than a confident wrong match.

## 7. Dependencies

- **Consumes:** `core` (household context, provenance envelope, `EventSink`, `MeterSink`); `taxonomy` resolver (optional collaborator); S3, SQS (RFC §9).
- **Consumed by:** `inventory` (scan drafts), `recipes` (import drafts); later `shopping` (receipts), `retailer catalogs` (order imports).
- **Events emitted:** `capture.draft_created`, `capture.draft_processed` (with duration + item count), `capture.draft_failed` (with reason class), `capture.draft_confirmed` (with edit stats: fields presented/edited/rejected), `capture.draft_discarded`, `capture.scan_session_completed` (photo count, total time).

## 8. Metrics

| Metric (PRD §8/§14) | Source | P0 target |
|---|---|---|
| Scan+confirm time vs manual entry | `capture.scan_session_completed` durations vs timed manual entry | Faster — the P0a gate |
| Time to draft ready | upload complete → `needs_review` | p95 < 60s for a 3-photo session |
| Confirmation burden | fields edited + items rejected per draft | Trending down week over week |
| Material-flag precision | material flags later edited vs accepted | Measured; drives prompt/threshold tuning |
| Parse success rate | processed vs failed | >90% excluding user-error photos |
| Import completion (PRD §14) | recipe drafts confirmed / created | P0b gate: minimal correction, unprompted use |
| Cost per draft | MeterSink data | Within D7 budget |

## 9. Risks

| Risk | Mitigation |
|---|---|
| Vision hallucination (invented items/quantities) | Confidence + materiality flags; free-text fallback; "probably" language; bbox evidence where available; never auto-apply material items |
| Nova Lite quality on real SA kitchens is unproven | Automatic Nova Pro escalation on failed/low-confidence parses; golden-set eval gates any model, tier or prompt change; prompts and raw interpretations versioned per draft so providers can be A/B tested |
| Cost runaway | Every call metered (D7); per-household daily cap with honest "try again tomorrow" copy; worker concurrency limits |
| Privacy of kitchen photos | Presigned PUT direct to S3 (never through the API); lifecycle deletion; discard deletes immediately; purge cascades (PRD §7) |
| Confirmation UX becomes the chore it replaced | Materiality classification caps what's asked; bulk-confirm; burden is a gated metric (§8) |
| Duplicate processing (SQS redelivery, client retries) | Idempotency keys + guarded status transitions (§4.4) |
| Multi-photo merge dedups wrongly | Dedup collapses only high-similarity candidates; conflicts surface as one item with notes rather than silent loss |

## 10. Voice and labels

- Capture entry points: *Send to Saamly* (share sheet, P0b) · *Snap your fridge* · *Add a recipe* (never "import" or "ingest").
- Receipt copy: *Recipe received. We're turning it into something you can plan and shop.*
- Review screen: *Here's what we found — check the things that matter.* Material items carry reasons: *We probably saw yoghurt. Check the amount before we leave it off your list.*
- Empty inbox: *Nothing to check. Snap your fridge or send a recipe when you're ready.*
- Failures are honest and actionable: *We couldn't read that photo — try again with the light on?* Never "processing error".
- Bulk action: *Looks right — confirm all*. Corrections are *Fix* / *Not this*, never "reject entity".
- No AI-announcement language (brand guardrail): Saamly "found", "noticed", "couldn't read" — never "the model detected".
