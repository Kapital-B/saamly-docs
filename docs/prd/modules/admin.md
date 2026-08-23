# Module spec — `admin`

Version 0.1 · July 2026 · Status: Draft · Phase: P0a
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.4 (§3.16, D3, D9, §7) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` (§5, §8.3, §9) · Depends on: `core.md`, `taxonomy.md`, `feedback.md`

## 1. Purpose and pillar

`admin` is the internal web application — a client of the internal APIs (D3), after the consumer apps (`saamly-mobile` and `saamly-web`). The consumer-facing browser app is specified in `consumer-web.md` and must never share this surface. It carries the operational surfaces the consumer app should never have: the dogfood dashboards that tell us whether phase gates pass, the `taxonomy` review queue that keeps the ontology healthy, raw-event inspection for debugging, and read-only household inspection. It serves no pillar directly; it serves the learning loop, and through it all three. Two rules shape everything in it: **function over polish** (D9) and **it never gates consumer features** — a consumer release never waits on `admin`, and `admin` being down affects nobody's dinner.

## 2. Users and jobs

| User | Jobs |
|---|---|
| David (dogfood operator) | Work the taxonomy review queue in minutes a week; run the weekly gate-review ritual; debug "why did the scan say yoghurt?"; watch cost counters |
| Future team member | Same, with the audit trail making every action attributable |
| No one else | The allowlist is the product boundary — consumers never touch this surface |

## 3. Scope: Now / Next / Later

- **Now (P0a):** Entra ID sign-in (our tenant, single-tenant registration) + email allowlist; gate dashboard home (the PRD §8 metrics, formulas per `feedback.md` §8); taxonomy review queue (promote / merge / reject; alias and category editing); raw-event inspector; read-only household inspection; server-side audit log of every admin call; deploy to S3 + CloudFront via Terraform in the `saamly-admin` repo.
- **Next (P1):** seed-content management; catalog-ingestion monitoring and match-quality tooling; cost dashboards built on D7 metering; golden-set eval results surfaced (harness stays in `saamly-service`); un-merge operation for taxonomy mistakes.
- **Later (P2+):** team roles beyond the allowlist; feature flags and experiment tooling; support workflows (deletion requests, price disputes); data-export tooling.

**Out of scope:** any write path into household data (read-only inspection, D9); anything consumers depend on; visual polish beyond legibility; the consumer web client (`saamly-web` / `consumer-web.md`).

## 4. Flows

### 4.1 Sign-in and boundary

MSAL (auth code + PKCE) against our Entra tenant → API validates the Entra JWT (signature, `iss` = our tenant, `aud` = app registration) → allowlist check → dashboard home. Valid Entra login but not on the allowlist → plain 403 ("This tool is for the Saamly team."). **Break-glass:** if Entra or the allowlist fails, the review queue can still be worked directly against the database/CLI — `admin` is the preferred surface, never the only path.

### 4.2 Weekly gate review (the ritual)

Open dashboard → every P0 gate metric shown as current value vs target vs trend (scan-to-done time, auto-resolution ≥85%, corrections <10%, queue trend, import completion, cost per household) → drill a failing metric into the raw events behind it → route findings to the owning module's metrics section. This is the meeting where "continue / fix / stop" per phase gate gets decided.

### 4.3 Taxonomy review queue (the core workflow)

1. Pending provisional concepts listed with evidence: sample strings, occurrence counts, households seen, age.
2. Reviewer keyboard-flows the queue: **promote** (looks right), **merge** (search existing concepts → alias onto it), **reject** (suppress re-proposal).
3. Alias/category editing on the concept detail screen; every mutation logged with actor and before/after.
4. Queue size and median age are themselves dashboard metrics — a growing queue is a product smell, not a backlog to grind.

### 4.4 Household inspection and event debugging

Search by household ID → read-only view: members (as pseudonyms where PII isn't needed for the task), inventory items, recipes, drafts (payload + uncertainty, not images beyond the retention window), and the household's raw event stream with content fields (allowlist-only, per `feedback.md` §4.2). Every view call is audit-logged server-side (D9: read-only *and logged*, not one or the other).

## 5. Data owned

`admin` owns almost nothing — deliberately.

| Entity | Key fields | Notes |
|---|---|---|
| AdminAuditLog | actor (Entra oid + email), action (read/mutate + endpoint), target_ref, at, outcome | Written **server-side** by the admin auth middleware on every `/v1/admin/*` call; no TTL at P0; the only admin-owned data |
| Allowlist | team emails | Config in SSM, not a database; change = Terraform/SSM edit |

Queue decisions live in `taxonomy`; metrics live in `feedback`; household data lives in its owning modules. `admin` is a window, not a store.

## 6. API boundary

`admin` consumes the admin surfaces defined by the owning modules — it adds no new back-end capability of its own:

| Endpoint | Owning module | UI feature |
|---|---|---|
| `GET /v1/admin/metrics/overview` | feedback | Gate dashboard home |
| `GET /v1/admin/events` | feedback | Raw-event inspector (content fields for allowlist only) |
| `GET /v1/admin/taxonomy/queue` · `POST .../promote` `/merge` `/reject` · `PUT .../concepts/:id` · `POST .../aliases` | taxonomy | Review queue + concept editing |
| `GET /v1/admin/households/:id` | core | Household inspection |

Client: generated from the vendored `admin.yaml` spec (openapi-typescript + openapi-fetch, RFC §5/§6). Tokens: MSAL-acquired Entra access token, memory-only storage, silent renewal. Admin actions do **not** emit household events to `feedback` — they produce audit-log entries instead (§5).

## 7. Dependencies

- **Consumes:** `core` (admin auth path, household inspection), `taxonomy` (queue + mutations), `feedback` (metrics + events); later `capture` (draft inspection), `commerce` (cost dashboards).
- **Consumed by:** nothing — terminal surface.
- **Events emitted:** none; audit log instead (§5).

## 8. Metrics

| Metric | Source | P0 target |
|---|---|---|
| Review-queue median concept age | `taxonomy` queue data | < 3 days during dogfood (feeds the "queue trending to zero" gate) |
| Queue throughput | promotes + merges + rejects / week | Measured; informs P1 auto-promotion tuning |
| Audit-log completeness | audited calls / total `/v1/admin/*` calls | 100% — enforced in middleware, tested |
| Gate-review cadence | weekly ritual happened (manual note) | Every week of dogfood |

## 9. Risks

| Risk | Mitigation |
|---|---|
| Admin becomes a privacy backdoor | Read-only enforced at the API layer (not just the UI); every call audit-logged server-side; allowlist reviewed at each phase gate; content fields restricted (feedback §4.2) |
| Scope creep — building tooling instead of product | Function-over-polish rule (D9); timeboxed dashboards; the P0a fallback (work the queue via DB/CLI) keeps `admin` honest about what earns UI |
| Entra/tenant misconfiguration locks the team out | Break-glass path (§4.1) documented; allowlist in SSM editable without a deploy |
| Admin downtime masks a failing gate | `admin` is independently deployable and never on the consumer path (D9); gate metrics also derivable from raw events via CLI |
| Sensitive content fields leak via screenshots/logs | Content fields allowlist-only; audit log records who viewed what; no images beyond retention window |

## 10. Voice and labels

- Terse and plain — this is a tool, not a brand surface. Dashboard labels reuse the PRD gate names exactly (per `feedback.md` §10): *Scan-to-done time* · *Corrections per 100 matches* · *Confirmation burden* · *Weeks planned in a row*.
- Queue labels per `taxonomy.md` §10: *Review queue* · *Promote* · *Merge* · *Reject* · *Aliases* · *Category*.
- No marketing copy, no delight, no empty-state poetry — *No pending concepts* is a complete sentence.
