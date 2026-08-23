# RFC 003 — Capture Module Implementation

Status: Draft · July 2026
Relates to: `docs/prd/modules/capture.md` v0.1 · `docs/prd/modules/core.md` v0.1 (§5, §6) · `docs/prd/modules/feedback.md` v0.1 (§4.4) · `docs/rfc/RFC001_Initial_Technical_Structure.md` (§7, §9, §10) · `docs/rfc/RFC002_Core_Module_Implementation.md` · scaffold in `saamly-service` (walking skeleton + core)

This RFC specifies *how* `capture` is implemented in `saamly-service`. The module spec owns product behaviour (the draft object, materiality, flows, voice); this RFC owns the Go package layout, DynamoDB key design, use-case boundaries, the worker pipeline, the **dual LLM provider design** (Bedrock for cloud, OpenAI-compatible for local dogfood against LM Studio), prompt/schema ownership, delivery slices, and the decisions that must be locked before writing handlers. It deliberately does not restate the flows in `capture.md` — it references them.

## 1. Goals and non-goals

**Goals**

- Ship the P0a `capture` surface end-to-end: photo upload (single + multi-photo scan session) via presigned S3 PUT, pasted-text intake, async worker pipeline (SQS → vision/text parse → draft), draft inbox with poll-based status, confirm/discard/retry, materiality flags, provenance on everything, idempotent processing, per-household metering of every LLM call.
- Make the LLM provider swappable behind `ports.VisionParser` / `ports.TextParser` with **two first-class adapters**: `bedrock` (Nova Lite/Pro/Micro, cloud) and `openai` (OpenAI-compatible HTTP, local dogfood against LM Studio). The worker never knows which provider is active.
- Keep the domain free of DynamoDB / HTTP / AWS SDK / Bedrock / OpenAI types (RFC 001 §3.2).
- Reuse `core` infrastructure verbatim: `EventSink`, `MeterSink`, `ProvenanceRecord`, household-context middleware, `AuthFrom(ctx)`.
- Leave the walking skeleton intact: health endpoints keep working; capture stubs in `httpapi.NewServer` are replaced slice-by-slice.

**Non-goals**

- URL import, share sheet, voice, receipt/barcode, retailer-order import, bulk import (capture.md §3 Next/Later).
- Push notification when a draft is ready (poll-based at P0; the inbox is the source of truth).
- What drafts *mean* — `inventory` owns candidate-item semantics and apply; `recipes` owns recipe semantics and apply. `capture` owns the pipeline, the draft object, and the confirm pattern (capture.md §3).
- Taxonomy resolution — `taxonomy` owns the `ConceptResolver` port and the ontology. `capture` treats the resolver as an optional collaborator and functions fully without it (free-text fallback, D4).
- Event *semantics* and the catalog — owned by `feedback.md`; `capture` only emits the `capture.*` events listed there.
- Push-based "draft ready" delivery (websockets / SSE) — poll at P0; revisit when the mobile inbox lands.
- Per-household daily cost cap enforcement in the worker — metered from day one (D7), but the cap with "try again tomorrow" copy lands with `commerce`. The metering plumbing is built now; the gate is not.

## 2. Package layout

`capture` spreads across the hexagonal layers already scaffolded. New code lands here:

```
saamly-service/
  internal/
    domain/
      capture/              # NEW — Draft, DraftItem, Status, Kind, SourceType, sentinel errors
      provenance/           # EXISTING (RFC 002 §3.3) — embed Record on drafts
    application/
      capture/             # NEW — CreateUpload, CompleteUpload, IngestText,
                            #       ProcessJob (worker), GetDrafts, GetDraft,
                            #       ConfirmDraft, DiscardDraft, RetryDraft
      capture/prompts/     # NEW — kind-specific prompt builders + JSON schemas
                            #       (shared by both LLM adapters; never duplicated)
    ports/
      vision.go            # EXISTING — VisionParser / TextParser / ModelTier
      blob.go              # EXISTING — BlobStore (extend: PresignGet)
      queue.go             # EXISTING — Queue
      repository.go       # EXTEND — DraftRepository
      resolver.go          # EXISTING (taxonomy-owned) — ConceptResolver (optional)
      meter.go            # EXISTING — MeterSink
      events.go           # EXISTING — EventSink
    adapters/
      dynamodb/
        keys.go           # EXTEND — draft key builders
        draft.go          # NEW — DraftRepository
      s3/
        client.go         # EXISTING — extend with PresignGet + lifecycle hook
      sqs/
        client.go         # EXISTING
      llm/
        bedrock.go        # EXISTING (stub) — implement Nova Lite/Pro/Micro
        openai.go         # NEW — OpenAI-compatible HTTP (LM Studio local)
        factory.go        # NEW — selects adapter by SAAMLY_LLM_PROVIDER
        doc.go            # package doc explaining the dual-provider contract
      httpapi/
        capture.go        # NEW — handlers for /capture/* and /drafts/*
        server.go         # EXTEND — replace capture stubs with real handlers
  cmd/
    api/main.go           # EXTEND — wire capture use cases + handlers
    worker/main.go       # EXTEND — wire capture worker + LLM factory
  api/
    consumer.yaml         # EXTEND — capture + drafts paths (before handlers)
```

Rules binding on this work (carried from RFC 002):

1. **Use cases own transactions.** A use case that writes multiple items (e.g. confirm draft → apply to inventory) receives a `Transactor` port or a repository method that performs one DynamoDB `TransactWriteItems`. Handlers never orchestrate multi-item writes.
2. **AuthContext is set once.** Middleware resolves `(user, household, role)`; handlers and use cases read `application.AuthFrom(ctx)` — they never re-query identity (core.md §6).
3. **Events never fail features.** Every use case that emits wraps `EventSink.Append` in a best-effort call: log + continue on error (feedback.md §4.3).
4. **No AWS / Bedrock / OpenAI types above adapters.** Domain entities are plain structs; the worker receives `ports.VisionParser` and calls `Parse(ctx, VisionRequest{...})`.
5. **Prompts live in `application/capture/prompts`, not in adapters.** Both adapters receive the same prompt text and JSON schema; they differ only in wire format. This is the single biggest guard against provider lock-in and the thing that makes A/B testing providers cheap (capture.md §9).

## 3. Domain model

### 3.1 `capture` package (new)

| Type | Fields | Notes |
|---|---|---|
| `Draft` | `ID`, `HouseholdID`, `CreatedBy`, `CreatedAt`, `Kind`, `SourceType`, `SourceRef`, `Provenance`, `Status`, `Images[]`, `Payload`, `Uncertainty`, `InterpretationRaw`, `ModelUsed`, `Metering`, `UpdatedAt` | The central object (capture.md §4.1) |
| `Kind` | `inventory_scan` · `recipe_import` (P0); extensible: `receipt`, `barcode`, `order_import` | Drives prompt selection and apply routing |
| `SourceType` | `photo` · `paste` (P0); `url` / `share` / `voice` / `manual` later | |
| `Status` | `queued → processing → needs_review → confirmed` · `failed` · `discarded` | Guarded transitions (§3.4) |
| `DraftImage` | `Key`, `ContentType`, `Order`, `BBox?` | S3 object keys; `BBox` optional "we saw it here" evidence |
| `Payload` | `map[string]any` (kind-specific shape, §3.2) | Always carries `name_text` / `raw_text` (D4 free-text fallback) |
| `Uncertainty` | `Items[]` / `Fields[]` with `Confidence` (0–1) + `Materiality` (`material` / `cosmetic`) | Drives confirm UX (capture.md §4.2) |
| `Metering` | `Capability`, `ModelUsed`, `InputTokens`, `OutputTokens`, `EstCostCents`, `DurationMs` | One entry per LLM call; rolled into MeterSink |
| `Provenance` | embeds `provenance.Record` | Imported-by, imported-at, source-type, rights-state |

Errors (sentinel): `ErrDraftNotFound`, `ErrDraftAlreadyProcessed` (idempotency guard), `ErrDraftNotConfirmable` (wrong status), `ErrDraftNotRetryable`, `ErrInvalidKind`, `ErrInvalidSourceType`, `ErrUploadNotComplete`, `ErrMaterialUnresolved` (confirm blocked on material flags).

### 3.2 Payloads (P0)

The payload is a `map[string]any` at the port boundary so the worker can store whatever the model returns without a schema migration per prompt change, but **the application layer owns typed shapes** in `application/capture/prompts` and validates before persisting. P0 shapes:

- **`inventory_scan`**: `{ "items": [ { "name_text": string, "quantity_hint"?: string, "concept_candidate"?: { "id": string, "name": string, "score": float }, "bbox"?: [x,y,w,h], "confidence": float, "materiality": "material"|"cosmetic" } ] }`. Multi-photo sessions merge detections into one candidate list, deduplicated by normalised text/concept similarity; near-duplicates collapse, conflicts surface as one item with two notes (capture.md §4.3).
- **`recipe_import`**: `{ "title"?: string, "servings"?: int, "times"?: { "prep"?: string, "cook"?: string }, "ingredients": [ { "raw_text": string, "quantity"?: string, "unit"?: string, "preparation"?: string, "concept_candidate"?: {...}, "confidence": float, "materiality": "material"|"cosmetic" } ], "steps": [string], "source_note"?: string }`. Missing title or steps is not a failure — the draft is still useful and the gap is shown plainly (capture.md §5.2).

`concept_candidate` comes from `taxonomy.ConceptResolver` when available. Capture treats it as optional and functions fully without it (free-text fallback, D4).

### 3.3 Provenance

Embeds `provenance.Record` (RFC 002 §3.3). `SourceType` on the draft *is* `provenance.SourceType`; `SourceRef` is the S3 key prefix for photos or empty for paste; `ImportedBy` is the user; `ImportedAt` is the worker's clock time at processing (not upload time — upload time is `CreatedAt`). Rights-state defaults to `personal_use`.

### 3.4 Lifecycle rules (binding)

- **Idempotent processing.** SQS is at-least-once. The worker guards every status transition with a conditional write: `queued → processing` only if currently `queued`; `processing → needs_review|failed` only if currently `processing`. A redelivered message that finds the draft already past `processing` no-ops and returns nil (capture.md §4.4).
- **Client upload retries** reuse the same `draft_id` via an idempotency key supplied at `POST /capture/uploads`. A second request with the same key returns the existing draft and its presigned URLs (still valid or reissued).
- **Retry and discard.** `failed` drafts can be retried (same source, new job); `discarded` drafts delete their S3 objects immediately and are excluded from the default inbox but retained for analysis (status stays `discarded`, not deleted).
- **Retention.** Raw uploads are S3 lifecycle-deleted after `SAAMLY_IMAGE_RETENTION_DAYS` (default 30, configurable — PRD §7). Drafts (payload + raw interpretation, no images) are retained at P0 for the correction/feedback loop. Household purge (`core`) cascades to drafts and uploads via the purge registry (RFC 002 §4) — `capture` registers its purge hook in slice E.
- **Offline (D8).** Photos taken offline queue in the app's outbox and upload when connectivity returns; the draft appears in the inbox when processing finishes. This is a mobile concern; the service just sees a late upload.

## 4. DynamoDB key design

All items live in the single table (`saamly-{env}`). Keys are built only inside `adapters/dynamodb/keys.go`. New patterns:

| Entity | PK | SK | GSI1PK / GSI1SK | TTL attr |
|---|---|---|---|---|
| Draft | `HOUSE#<householdId>` | `DRAFT#<id>` | `DRAFT#<status>` / `<householdId>#<createdAt>#<id>` (inbox by status, newest first) | — |
| Draft image | `DRAFT#<draftId>` | `IMG#<order>` | — | optional `ttl` (= image retention) |
| Idempotency key | `IDEM#<key>` | `META` | — | `ttl` (= 24h) |

**Inbox query (binding):** `GSI1PK = DRAFT#<status>`, `GSI1SK = <householdId>#<createdAt>#<id>` — the inbox is "all drafts in this household with this status, newest first" via a single GSI1 query scoped by household prefix on the SK. One index, no scan.

**Conditional writes that matter:**

- Draft create: `attribute_not_exists(PK)` on `HOUSE#…/DRAFT#…`; idempotency key item written in the same `TransactWriteItems` (or a preceding conditional put on `IDEM#…`).
- Status transitions: `UpdateItem` with a condition expression on `status` matching the expected current value. On `ConditionalCheckFailedException` the use case treats it as already-done and returns nil (idempotent).
- Confirm: `needs_review → confirmed` only if currently `needs_review` and no material flags are unresolved (the use case checks materiality before issuing the conditional write; `ErrMaterialUnresolved` is a domain error, not a DB error).

**Household purge cascade:** `capture` registers a `PurgeHook` with `application/household` that, for a given `householdID`, (a) deletes all `HOUSE#<id>/DRAFT#*` items via a Query + batch delete, (b) deletes the corresponding S3 objects via `BlobStore.Delete` for each draft's image keys, (c) drops `IDEM#*` items owned by that household (tracked via a household attribute on the idem item). Full cascade is wired in slice E; the hook interface is defined in slice A so `core` can call it.

## 5. Ports to add or extend

| Port | Owner | Notes |
|---|---|---|
| `DraftRepository` | ports (new) | `Put`, `Get`, `UpdateStatus` (conditional), `ListByStatus(householdID, status, limit, cursor)`, `DeleteImages(draftID)`, `ListImageKeys(draftID)` |
| `BlobStore` | ports (extend) | add `PresignGet(ctx, key, ttl)` (worker fetches images before calling the parser) |
| `Queue` | ports (existing) | `Enqueue(ctx, body)` — body is the job envelope (§7) |
| `VisionParser` | ports (existing) | `Parse(ctx, VisionRequest) (VisionResult, error)` — unchanged contract |
| `TextParser` | ports (existing) | `ParseText(ctx, TextRequest) (TextResult, error)` — unchanged contract |
| `ConceptResolver` | ports (existing, taxonomy-owned) | Optional; `Resolve(ctx, text) (ConceptCandidate, error)` — capture calls it best-effort and ignores `ErrNoMatch` |
| `EventSink` | ports (existing) | DynamoDB adapter |
| `MeterSink` | ports (existing) | DynamoDB adapter — `Increment` per LLM call |
| `IDGen` | ports (existing) | ULID for draft IDs, idempotency keys, event IDs |
| `Clock` | ports (existing) | `SystemClock`; tests inject fixed clock |
| `Transactor` | ports (optional) | Only if repository methods prove insufficient |
| `PurgeHook` | application/household (new interface) | `Purge(ctx, householdID) error` — registered by `capture` in slice E |

## 6. LLM provider design (dual adapter)

This is the heart of the RFC. RFC 001 §7 locked the routing decision (Nova Lite default, Pro escalation, Micro for text) but left the provider swappable. This RFC makes the swap first-class.

### 6.1 Contract

Both adapters implement `ports.VisionParser` and `ports.TextParser`. The worker receives a `*Parser` composite (§6.5) wired in `cmd/worker` and never knows which provider is active.

```go
// internal/ports/vision.go (unchanged)
type VisionParser interface {
    Parse(ctx context.Context, req VisionRequest) (VisionResult, error)
}
type TextParser interface {
    ParseText(ctx context.Context, req TextRequest) (TextResult, error)
}
```

`VisionRequest` carries `Kind`, `ImageKeys []string`, `HouseholdHint`, `Tier`. The adapter does **not** fetch images — that's the worker's job (§7.2) so both adapters receive already-resolved bytes and the worker can record fetch latency separately. This is a small change to the existing port: `ImageKeys` stays for record-keeping, but the worker passes image bytes via a context-attached `[]byte` slice or — cleaner — the port gains a `Images []VisionImage` field where `VisionImage { Bytes []byte; ContentType string }`. The adapter ignores `ImageKeys` when `Images` is populated. This keeps the port stable for any future caller that only has keys.

**Decision:** extend `VisionRequest` with `Images []VisionImage` and deprecate `ImageKeys` (keep for now, remove at P1). Adapters read `Images` only.

### 6.2 Tier mapping is provider-specific

`ports.ModelTier` (`TierLite` / `TierPro` / `TierMicro`) is a routing *intent*, not a model ID. Each adapter maps it to its own models:

- **Bedrock** (`adapters/llm/bedrock.go`): `TierLite → amazon.nova-lite-v1:0`, `TierPro → amazon.nova-pro-v1:0`, `TierMicro → amazon.nova-micro-v1:0` (RFC 001 §7).
- **OpenAI-compatible** (`adapters/llm/openai.go`): env-driven, no hardcoded defaults. `SAAMLY_LLM_VISION_MODEL`, `SAAMLY_LLM_VISION_ESCALATE_MODEL`, `SAAMLY_LLM_TEXT_MODEL` point at whatever LM Studio exposes. If the escalate model is unset, Pro tier falls back to the vision model (single-model local dogfood is fine).

Escalation (Lite → Pro on low confidence) stays in the **capture application** layer (§7.3); adapters just honor `req.Tier`. The application decides when to escalate based on the first parse's confidence flags; the adapter re-calls with `TierPro`.

### 6.3 Prompt and schema ownership

`application/capture/prompts` owns:

- `BuildInventoryScanPrompt(householdHint string) (system, user string, schema map[string]any)`
- `BuildRecipeImportPrompt() (system, user string, schema map[string]any)`
- `BuildTextParsePrompt(kind string) (system, user string, schema map[string]any)`
- Version constants (`PromptV1Inventory`, `PromptV1Recipe`) stored on the draft so a prompt change is A/B-testable and replayable (capture.md §9).

Both adapters receive `(system, user, schema)` and translate to their wire format:

- **Bedrock** wraps the prompt in Nova's native content blocks and passes `inferenceConfig` for the JSON schema (or a tool definition, depending on Nova's structured-output support at implementation time).
- **OpenAI-compatible** sends `messages: [{role: system}, {role: user, content: [{type: text}, {type: image_url, image_url: {data: base64}}]}]` and `response_format: { type: "json_schema", json_schema: schema }` when the local model supports it; falls back to a prompt-injected schema and a JSON-only `response_format` when it doesn't. The adapter detects capability via a one-shot probe at startup and logs the mode.

### 6.4 Bedrock adapter (`adapters/llm/bedrock.go`)

Already scaffolded as a stub. Implementation:

- `New(ctx, region, endpointURL)` — existing; `endpointURL` empty in cloud, set to floci only if floci ever emulates Bedrock (it doesn't today; local tests stub the port instead).
- `Parse(ctx, VisionRequest)` — map `req.Tier` to model ID; build the Bedrock request body from the prompt builder + image bytes; call `bedrockruntime.InvokeModel`; parse the JSON response into `ports.VisionResult` (`Payload`, `RawResponse`, `ModelUsed`, token counts from the response metadata).
- `ParseText(ctx, TextRequest)` — same, text-only, normally `TierMicro`.
- Cost estimation: a per-model lookup table (`novaLiteCostPer1KInput`, etc.) converts token counts to `EstCostCents` for the MeterSink. The metering call is made by the worker, not the adapter, so the adapter returns token counts and the worker computes cost — keeps the adapter pure.

### 6.5 OpenAI-compatible adapter (`adapters/llm/openai.go`)

New. Talks to any OpenAI-compatible `/v1/chat/completions` endpoint — LM Studio's local server is the primary target.

```go
type OpenAIClient struct {
    baseURL    string          // http://localhost:1234/v1
    apiKey     string          // "lm-studio" — often ignored, but some clients require a value
    visionModel string          // SAAMLY_LLM_VISION_MODEL
    escalateModel string        // SAAMLY_LLM_VISION_ESCALATE_MODEL (may equal visionModel)
    textModel  string          // SAAMLY_LLM_TEXT_MODEL
    http       *http.Client
}
```

- `New(ctx, cfg)` reads `SAAMLY_LLM_BASE_URL`, `SAAMLY_LLM_API_KEY`, model envs. Probes `/v1/models` at startup and logs what's available; does not fail if the probe fails (LM Studio may be started after the worker).
- `Parse` — builds the multimodal message (text + base64 `image_url`), sets `model` from tier, calls `/v1/chat/completions`, parses the JSON content. Token counts from `usage` in the response.
- `ParseText` — text-only, `TierMicro` → `textModel`.
- Cost estimation: local calls meter with `EstCostCents=0` (still increment count so D7 plumbing is exercised). The worker handles this — the adapter just returns zero token-cost.

### 6.6 Factory (`adapters/llm/factory.go`)

```go
// NewParser returns a VisionParser + TextParser selected by SAAMLY_LLM_PROVIDER.
// Default: "bedrock" in dev/prod, "openai" in local.
func NewParser(ctx context.Context, cfg config.Config) (ports.VisionParser, ports.TextParser, error) {
    switch cfg.LLMProvider {
    case "openai":
        return newOpenAI(ctx, cfg)
    case "bedrock":
        return newBedrock(ctx, cfg)
    default:
        return nil, nil, fmt.Errorf("SAAMLY_LLM_PROVIDER must be openai|bedrock, got %q", cfg.LLMProvider)
    }
}
```

Both adapters are returned as a single composite `*Parser` that implements both interfaces, so the worker receives one value. The factory is the only place that knows both adapters exist; everywhere else takes `ports.VisionParser` / `ports.TextParser`.

### 6.7 Local dogfood config (binding)

`local` env defaults to `openai` so a developer with LM Studio running gets capture working with zero AWS credentials. Suggested `.env`:

```
SAAMLY_LLM_PROVIDER=openai
SAAMLY_LLM_BASE_URL=http://localhost:1234/v1
SAAMLY_LLM_API_KEY=lm-studio
SAAMLY_LLM_VISION_MODEL=qwen2-vl-7b-instruct
SAAMLY_LLM_TEXT_MODEL=qwen2.5-7b-instruct
```

If the worker runs inside Docker, `localhost` won't reach the host; use `http://host.docker.internal:1234/v1`. The Makefile / compose should expose this as a single env override.

## 7. Worker pipeline

The worker (`cmd/worker`) is the only caller of the LLM port at P0 (capture.md §6). It consumes SQS, processes one job, writes the draft, emits events, records metering.

### 7.1 Job envelope

```json
{ "type": "capture.parse", "draft_id": "<ulid>", "household_id": "<id>", "kind": "inventory_scan|recipe_import", "attempt": 1 }
```

`attempt` increments on retry; the worker caps attempts (default 3) and moves to `failed` after that. The envelope is opaque to SQS — it's just bytes.

### 7.2 ProcessJob use case (binding)

1. Load draft by `draft_id`. If not found → no-op (household purged mid-processing; capture.md §5.3).
2. Conditional `queued → processing`. On `ConditionalCheckFailedException` → return nil (already processed by a redelivery).
3. Fetch image bytes via `BlobStore.PresignGet` + HTTP GET (or a future `BlobStore.Get` — see §5). Record fetch latency on the draft for the feedback loop.
4. Build prompt via `application/capture/prompts` for `req.Kind`.
5. Call `VisionParser.Parse(ctx, VisionRequest{Kind, Images, HouseholdHint, Tier: TierLite})` (or `TextParser.ParseText` for paste).
6. Validate the returned payload against the kind's shape (§3.2). On validation failure → escalate to `TierPro` and retry once (§7.3). On second failure → `failed` with `reason_class = "schema_validation"`.
7. Optional: call `ConceptResolver.Resolve` for each item/ingredient; attach `concept_candidate` where it matches; ignore `ErrNoMatch` (free-text fallback, D4).
8. Compute `EstCostCents` from token counts + the adapter's per-model cost table (Bedrock) or zero (local).
9. `MeterSink.Increment(ctx, householdID, capability, estCostCents)` — best-effort, log on error.
10. Conditional `processing → needs_review` (or `failed`). Persist `Payload`, `Uncertainty`, `InterpretationRaw`, `ModelUsed`, `Metering`.
11. Emit `capture.draft_processed` (with `duration_ms`, `item_count`, `model_used`, `tier_used`) or `capture.draft_failed` (with `reason_class`). Best-effort.
12. For `inventory_scan` with multiple photos: emit `capture.scan_session_completed` (with `photo_count`, `total_ms`).

### 7.3 Escalation rule (Lite → Pro)

Binding: escalate when **any** of:

- The adapter returns an error that the application classifies as a parse error (not a network/timeout error — those retry at the same tier).
- The payload fails schema validation (§7.2 step 6).
- The mean per-item confidence is below `SAAMLY_LLM_CONFIDENCE_THRESHOLD` (default 0.6) — configurable so the golden set can tune it.

Escalation re-runs steps 4–9 with `TierPro` and overwrites the draft payload. The `ModelUsed` and `Metering` reflect the Pro call; the Lite call's metering is still recorded (both calls meter, so cost analysis sees the escalation rate — capture.md §8 "Material-flag precision" neighbour). `capture.draft_processed` carries `tier_used: "pro"` and `escalated: true`.

### 7.4 Failure modes (binding)

- **Adapter network/timeout error** → leave draft `processing`, return error from the handler so SQS redelivers (visibility timeout expires). After `max_attempts` SQS receives → DLQ. The draft stays `processing` until a manual `retry` or a requeue.
- **Adapter parse error (non-retryable)** → `failed` with `reason_class = "parse_error"`. Retryable via `POST /drafts/:id/retry`.
- **Schema validation fails twice** → `failed` with `reason_class = "schema_validation"`.
- **Household purged mid-processing** → step 1 no-ops; no event emitted (the household is gone).
- **Image fetch fails** → `failed` with `reason_class = "image_fetch"`; source retained for retry.

### 7.5 Worker wiring (`cmd/worker/main.go`)

After core wiring (RFC 002 §8), add:

1. Build `BlobStore` (S3 or floci), `DraftRepository` (memory or DynamoDB), `Queue` (existing SQS consumer).
2. Build LLM adapters via `llm.NewParser(ctx, cfg)` (§6.6).
3. Build `ConceptResolver` if taxonomy is wired (nil at slice A; taxonomy lands later).
4. Build `ProcessJob` use case with all deps + clock + idgen + sinks.
5. Register `capture.parse` in the job switch (alongside `core.purge`).

## 8. Use cases (API-side)

Each use case is a struct with explicit dependencies, a single exported method, and unit tests with fakes. Handlers translate HTTP ↔ use-case I/O and map domain errors to problem+json.

| Use case | Input | Behaviour (capture.md) | Events |
|---|---|---|---|
| `CreateUpload` | kind, content_types[], idempotency_key | Create draft `queued` + idem item; presign PUT URLs for each content type; return draft_id + URLs | `capture.draft_created` |
| `CompleteUpload` | draft_id | Validate all expected objects exist (HEAD); enqueue `capture.parse` job; status stays `queued` until worker picks it up | — |
| `IngestText` | kind, text | Create draft `queued` with `source_type=paste`; enqueue `capture.parse` job | `capture.draft_created` |
| `GetDrafts` | status, cursor | Paginated inbox query via GSI1 | — |
| `GetDraft` | draft_id | Full draft; 404 if not in caller's household | — |
| `ConfirmDraft` | draft_id, edited_payload | Validate material flags resolved; conditional `needs_review → confirmed`; route to owning module's apply (by kind) | `capture.draft_confirmed` (with `fields_presented`, `fields_edited`, `items_rejected`) |
| `DiscardDraft` | draft_id | Conditional `needs_review|failed → discarded`; delete S3 objects immediately | `capture.draft_discarded` |
| `RetryDraft` | draft_id | Only `failed`; reset to `queued`; enqueue new job | — |

**Apply routing (binding):** `ConfirmDraft` does not write inventory/recipe entities itself. It calls a per-kind `Applier` port (`inventory.ApplyDraft(ctx, draft)` / `recipes.ApplyDraft(ctx, draft)`) that the owning module implements. At P0 slice D, `inventory` is the only applier; `recipes` lands at P0b. Until an applier exists, confirm still marks the draft `confirmed` and emits the event — the applier is a follow-the-draft call, not a blocker. This keeps `capture` independently demoable.

## 9. HTTP surface

Replace the `501` stubs in `httpapi.NewServer` slice-by-slice. Contract changes land in `api/consumer.yaml` *before* handlers (RFC 001 §6). Spectral must stay at zero errors.

**New routes (capture.md §6):**

| Method | Path | Handler |
|---|---|---|
| POST | `/v1/capture/uploads` | `CreateUpload` — returns `{ draft_id, uploads: [{ key, url, headers, expires_at }] }` |
| POST | `/v1/capture/uploads/{id}/complete` | `CompleteUpload` — `204` on success |
| POST | `/v1/capture/text` | `IngestText` — returns `{ draft_id }` |
| GET | `/v1/drafts?status=&cursor=&limit=` | `GetDrafts` — paginated inbox |
| GET | `/v1/drafts/{id}` | `GetDraft` |
| POST | `/v1/drafts/{id}/confirm` | `ConfirmDraft` — body is the edited payload |
| POST | `/v1/drafts/{id}/discard` | `DiscardDraft` |
| POST | `/v1/drafts/{id}/retry` | `RetryDraft` |

All under the existing `Authn` middleware (household context from `core`). Drafts are scoped to the caller's active household — a draft from another household returns `404` (not `403`, to avoid leaking existence).

**Problem+json shapes (binding):**

- `404` draft not found / not in household → title `not_found`, detail `We couldn't find that draft — it may have been removed.`
- `409` draft not confirmable (wrong status) → title `conflict`, detail `This draft can't be confirmed yet — it's still processing.` for `queued`/`processing`, or `This draft is already confirmed.` for `confirmed`.
- `409` material unresolved → title `conflict`, detail `A few items need your attention before we can confirm — check the things marked "probably".` (brand voice, capture.md §10).
- `409` upload not complete → title `conflict`, detail `Finish uploading all photos, then tap "I'm done".`
- `422` invalid kind / source type → title `invalid_request`, detail naming the field.

**OpenAPI gaps to close in slice A** (before handlers):

- All eight routes above with request/response schemas.
- `Draft` schema (§3.1) and the two payload shapes (§3.2) as `InventoryScanPayload` / `RecipeImportPayload`.
- Problem responses for `400` / `404` / `409` / `422` where domain conflicts apply.
- `DraftStatus` enum: `queued | processing | needs_review | confirmed | failed | discarded`.

## 10. Middleware and composition

`cmd/api` wiring after `core` (RFC 002 §8):

1. Build `BlobStore` (S3), `DraftRepository` (memory or DynamoDB), `Queue` (SQS).
2. Build capture use cases (`CreateUpload`, `CompleteUpload`, `IngestText`, `GetDrafts`, `GetDraft`, `ConfirmDraft`, `DiscardDraft`, `RetryDraft`).
3. Pass `CaptureHandlers` into `httpapi.Deps`; replace the four capture stubs in `server.go` with real handlers.
4. Dev bypass unchanged — `dev-user` + seeded household lets dogfood capture without OAuth.

`cmd/worker` wiring after `core`:

1. Build `BlobStore`, `DraftRepository`, LLM adapters via `llm.NewParser(ctx, cfg)` (§6.6).
2. Build `ProcessJob` use case.
3. Register `capture.parse` in the job switch alongside `core.purge`.

## 11. Configuration additions

Extend `platform/config` (and `.env.example`):

| Env | Purpose | Default |
|---|---|---|
| `SAAMLY_LLM_PROVIDER` | `openai` \| `bedrock` | `openai` in `local`, `bedrock` in `dev`/`prod` |
| `SAAMLY_LLM_BASE_URL` | OpenAI-compatible base URL (openai only) | `http://localhost:1234/v1` |
| `SAAMLY_LLM_API_KEY` | API key (openai only; often ignored by LM Studio) | `lm-studio` |
| `SAAMLY_LLM_VISION_MODEL` | Vision model ID (openai only) | empty (must be set when provider=openai) |
| `SAAMLY_LLM_VISION_ESCALATE_MODEL` | Pro-tier vision model (openai only) | falls back to `SAAMLY_LLM_VISION_MODEL` |
| `SAAMLY_LLM_TEXT_MODEL` | Text model ID (openai only) | falls back to `SAAMLY_LLM_VISION_MODEL` |
| `SAAMLY_LLM_CONFIDENCE_THRESHOLD` | Mean per-item confidence below which Lite escalates to Pro | `0.6` |
| `SAAMLY_CAPTURE_MAX_ATTEMPTS` | Worker retry cap before `failed` | `3` |
| `SAAMLY_IMAGE_RETENTION_DAYS` | S3 lifecycle for raw uploads (existing) | `30` |
| `SAAMLY_CAPTURE_INBOX_LIMIT` | Default page size for `GET /drafts` | `50` |

`config.Load` validates: if `LLMProvider=openai`, `LLMVisionModel` must be set; if `bedrock`, `AWSRegion` must be set (already required). The factory (§6.6) is the only consumer of these.

## 12. Delivery slices

Implement and merge as five small PRs. Each slice is independently demoable and keeps `main` green.

| Slice | Delivers | Demo |
|---|---|---|
| **A — Domain + ports + OpenAPI** | `domain/capture` (Draft, Status, Kind, payloads, sentinels); `DraftRepository` port; `PurgeHook` interface; extend `BlobStore.PresignGet`; extend `VisionRequest` with `Images`; all eight OpenAPI paths + schemas; `make spec-lint` green; LLM factory + OpenAI adapter skeleton (no prompts yet) | `curl` against a stubbed `CreateUpload` returns a presigned URL; Spectral stays green |
| **B — LLM adapters + prompts** | Implement `bedrock.go` (Nova Lite/Pro/Micro) and `openai.go` (LM Studio); `application/capture/prompts` builders for inventory + recipe; factory wired in `cmd/worker`; confidence threshold + escalation rule; unit tests with fake parser | Worker unit test: feed a fixture image → fake parser → draft `needs_review` with a known payload; same test against LM Studio with a real model (integration-tagged) |
| **C — Worker pipeline** | `ProcessJob` use case; `capture.parse` job in `cmd/worker`; conditional status transitions; metering + events; failure modes; retry cap; DLQ wiring in Terraform | Local end-to-end: `POST /capture/text` → SQS → worker (LM Studio) → `GET /drafts` shows `needs_review`; `POST /drafts/:id/discard` deletes S3 |
| **D — API handlers + confirm** | `CreateUpload`, `CompleteUpload`, `IngestText`, `GetDrafts`, `GetDraft`, `ConfirmDraft`, `DiscardDraft`, `RetryDraft` handlers; `Applier` port; `inventory.ApplyDraft` (stub or real if inventory lands in parallel); idempotency keys | Mobile: snap fridge → upload → poll → review → confirm → items appear in inventory (or stubbed "applied" status) |
| **E — Purge cascade + retention + Terraform** | `capture.PurgeHook` registered with `application/household`; S3 lifecycle rule in Terraform; SQS + DLQ in Terraform; EventBridge schedule for `capture.parse` retries (optional); seed script for local dogfood | Delete household → drafts + S3 objects gone; Terraform apply to dev green |

**Exit criteria for "capture done" (P0a):** all eight endpoints return real responses (no `501`); worker processes a real photo end-to-end against LM Studio locally and against Bedrock in dev; one integration test against floci covers `POST /capture/text` → worker → `GET /drafts` → `POST /drafts/:id/confirm`; `make test` and Spectral stay green; metering counters increment per call.

## 13. Testing strategy

| Layer | What | How |
|---|---|---|
| Domain | Status transition invariants; materiality rules; payload validation | Table-driven pure tests |
| Use cases | Every use case with fake ports + fixed clock | Assert state changes, emitted events, error mapping; fake `VisionParser` returns canned payloads |
| Prompts | Prompt builders produce stable output; schema validates against a fixture | Snapshot tests on the prompt text; JSON schema validation on a known-good payload |
| LLM adapters | Wire format; tier → model mapping; token-count parsing; error classification | Unit tests with `httptest` (OpenAI) and a stubbed Bedrock client; one integration test against a running LM Studio tagged `//go:build integration` |
| Worker | Idempotent redelivery; escalation; failure modes; metering + events | Fake parser + fake repos; drive `ProcessJob` with a fixture draft |
| Adapters | DynamoDB key helpers; conditional-write paths; GSI1 inbox query | floci integration test tagged `//go:build integration` |
| HTTP | Handler mapping + problem+json shapes; household scoping (404 for other households) | `httptest` with fake use cases |
| End-to-end | Upload → worker → confirm | One floci integration test; one LM Studio integration test (developer machine, not CI) |

Fakes live next to the use case tests (`application/capture/fakes_test.go`). The fake `VisionParser` returns a configurable payload + confidence so escalation and materiality logic can be exercised without a real model.

## 14. Risks and open questions

| Risk / question | Proposal |
|---|---|
| LM Studio model quality is far below Nova Lite | Accept for local dogfood of the *pipeline*; golden-set eval gates Bedrock quality, not local. Local is for plumbing, not for the P0a scan-time gate |
| OpenAI-compatible structured output support varies by local model | Probe at startup; fall back to prompt-injected schema + `response_format: json_object`. The prompt builder emits the schema in the system prompt as a fallback so both modes work |
| Bedrock `InvokeModel` response shape differs from OpenAI's `usage` | Adapters parse their own response; both return the same `VisionResult` shape. Token-count extraction is adapter-specific and tested there |
| Image bytes in `VisionRequest` bloat the struct | Acceptable at P0 photo sizes (a few MB per session); revisit with a streaming or URL-based approach if Lambda memory pressure shows up |
| floci S3 presign parity | Integration tests assert behaviour; deployed `dev` is the safety net (RFC 001 §9) |
| SQS redelivery races the worker | Conditional status transitions make this safe; redelivery no-ops |
| Household purge deletes S3 objects slowly (batch delete limits) | Batch in chunks of 1000 (S3 limit); acceptable at P0 household counts |
| Concept resolver not ready | Capture functions without it; `concept_candidate` is nil; free-text fallback (D4). Wire when taxonomy lands |
| `recipes` applier not ready at P0a | Confirm still marks `confirmed` and emits the event; `recipes.ApplyDraft` is a no-op until P0b. Capture is independently demoable |
| Per-household daily cost cap not enforced | Metered now, gated by `commerce` later. Documented as a non-goal (§1) |
| Prompt versioning drift between adapters | Prompts live in one package; both adapters import the same builder. Version constant stored on the draft |

**Open questions for decision before slice A:**

1. Confirm `VisionRequest.Images` extension vs a separate `VisionParser.ParseImages` method — proposal: extend the struct, deprecate `ImageKeys` at P1.
2. Confirm the confidence threshold default (0.6) — tunable via env, but the golden set should drive the real value before P0a exit.
3. Confirm whether `ConfirmDraft` should call the applier synchronously (current proposal) or emit a `capture.draft_confirmed` event that the owning module consumes — proposal: **synchronous** at P0 (feedback.md P0 rule: events are records, not transport); revisit at P1.
4. Confirm the local default LLM provider is `openai` (LM Studio) vs requiring a developer to set `SAAMLY_LLM_PROVIDER` explicitly — proposal: default `openai` in `local` so dogfood works with zero AWS credentials, but log a warning if `SAAMLY_LLM_VISION_MODEL` is unset.

## 15. Out of scope reminders

Do not sneak into `capture` PRs:

- URL import, share sheet, voice, receipt/barcode, retailer-order import, bulk import (capture.md §3 Next/Later).
- Inventory item writes — `inventory` owns apply.
- Recipe entity writes — `recipes` owns apply.
- Taxonomy resolution logic — `taxonomy` owns the resolver.
- Event consumers (feedback.md P0 rule: events are records, not transport).
- Per-household daily cost cap enforcement (commerce).
- Push notification when a draft is ready.
- The mobile camera UI (lives in `saamly-mobile`; this RFC is service-side).

## 16. Implementation checklist (summary)

When this RFC is accepted:

1. Expand `api/consumer.yaml` for the §9 gaps; `make spec-lint`.
2. Slice A → E as sequenced above; each PR references this RFC and the relevant capture.md section.
3. Update `capture.md` status to `Accepted` / bump to v0.2 only if product behaviour changes; technical drift belongs here.
4. Mark RFC 001 §15 step 3 (`capture`) as in progress in a one-line amendment note when slice A merges.
5. Before P0a exit: run the golden-set eval against Bedrock in dev; record the baseline scan+confirm time and confirmation burden (capture.md §8).

