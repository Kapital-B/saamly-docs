# RFC 002 — Core Module Implementation

Status: Implemented (P0a slices A–E) · July 2026
Relates to: `docs/prd/modules/core.md` v0.1 · `docs/prd/modules/feedback.md` v0.1 (§4) · `docs/rfc/RFC001_Initial_Technical_Structure.md` (§3, §8, §10) · scaffold in `saamly-service` (walking skeleton)
Amended July 2026: account deletion anonymises by deleting the identity tree only; event `user_id`s are left intact and resolved as “former member” at read time (§4, §6.3). Local default store is in-memory (`SAAMLY_STORE=memory`); DynamoDB EventSink/MeterSink + key builders land for cloud; full DynamoDB entity repos remain a follow-up hardening task.

This RFC specifies *how* `core` is implemented in `saamly-service`. The module spec owns product behaviour; this RFC owns the Go package layout, DynamoDB key design, use-case boundaries, auth/session mechanics, sink implementations, delivery slices, and the decisions that must be locked before writing handlers. It deliberately does not restate the flows in `core.md` — it references them.

## 1. Goals and non-goals

**Goals**

- Ship the P0a `core` surface end-to-end: OAuth + magic-link auth, sessions, households (create / invite / join / leave / delete with 72h grace), preferences, account deletion, event sink, metering counters, admin household inspection.
- Make household context resolution correct by construction for every later module (PRD D1, core.md §6).
- Keep the domain free of DynamoDB / HTTP / AWS SDK types (RFC 001 §3.2).
- Leave the walking skeleton intact: health endpoints keep working; unauthenticated routes that are not yet implemented continue returning `501` until their slice lands.

**Non-goals**

- Multi-household UX, owner/member roles, email-change, account recovery (core.md §3 Next).
- Event *semantics* and the catalog — owned by `feedback.md`; `core` only implements the sink and emits the `core.*` events listed there.
- Entra admin *login* wiring beyond the allowlist check already scaffolded — admin inspection lands in slice E, but the Entra verifier activates only when tenant credentials exist.
- Rate limiting at the application layer (API Gateway / WAF later); P0 uses coarse per-IP / per-email checks in-process for magic-link only.
- Provenance as a standalone store — provenance is an envelope written *alongside* owning-module entities (core.md §5); `core` exports the type and a helper, not a repository.

## 2. Package layout

`core` spreads across the hexagonal layers already scaffolded. New code lands here:

```
saamly-service/
  internal/
    domain/
      household/           # EXISTING — extend with Invite, Preferences, PreferenceRules
      identity/            # NEW — User, ProviderIdentity, Session, MagicLink (auth domain)
    application/
      auth/                # NEW — Exchange, RequestMagicLink, VerifyMagicLink, Refresh, Logout
      household/           # NEW — Create, Invite, AcceptInvite, Leave, Remove, Delete, CancelDeletion
      profile/             # NEW — GetMe, UpdatePreferences, DeleteAccount
    ports/
      repository.go        # EXTEND — UserRepository, SessionRepository, InviteRepository, …
      events.go            # EXISTING EventSink
      meter.go             # EXISTING MeterSink
      idgen.go             # EXISTING — implement ULID adapter
    adapters/
      dynamodb/
        keys.go            # NEW — PK/SK builders (single place)
        user.go            # NEW
        household.go       # NEW
        session.go         # NEW
        invite.go          # NEW
        magic.go           # NEW
        events.go          # NEW — EventSink
        meter.go           # NEW — MeterSink
        membership.go      # NEW — MembershipLoader for Authn middleware
      ulid/                # NEW — ports.IDGen
      httpapi/
        auth.go            # NEW — handlers for /auth/*
        household.go       # NEW
        profile.go         # NEW
        admin_household.go # NEW
        # server.go wires the real handlers; stubs replaced slice-by-slice
```

Rules binding on this work:

1. **Use cases own transactions.** A use case that writes multiple items (e.g. create household + membership) receives a `Transactor` port or a repository method that performs one DynamoDB `TransactWriteItems`. Handlers never orchestrate multi-item writes.
2. **AuthContext is set once.** Middleware resolves `(user, household, role)`; handlers and use cases read `application.AuthFrom(ctx)` — they never re-query identity (core.md §6).
3. **Events never fail features.** Every use case that emits wraps `EventSink.Append` in a best-effort call: log + continue on error (feedback.md §4.3).
4. **No AWS types above adapters.** Domain entities are plain structs; repositories accept/return them.

## 3. Domain model

### 3.1 `identity` package (new)

| Type | Fields | Notes |
|---|---|---|
| `User` | `ID`, `Email` (normalised lowercase), `DisplayName`, `CreatedAt` | One account per verified email |
| `ProviderIdentity` | `UserID`, `Provider` (`google`/`microsoft`/`magic`), `Subject`, `Email`, `LinkedAt` | Find-or-create attaches providers; never duplicates users |
| `RefreshSession` | `ID` (opaque, stored hashed), `UserID`, `DeviceLabel`, `CreatedAt`, `ExpiresAt`, `RotatedFrom`, `RevokedAt` | 30d TTL; reuse of a rotated token revokes the chain |
| `MagicLinkCode` | `Email`, `CodeHash`, `Token` (link token, hashed), `ExpiresAt`, `Attempts`, `Used` | 10 min TTL, max 5 verify attempts |

Errors (sentinel): `ErrUserNotFound`, `ErrInvalidCredentials`, `ErrSessionRevoked`, `ErrMagicLinkExpired`, `ErrMagicLinkUsed`, `ErrMagicLinkLocked` (attempts exhausted).

### 3.2 `household` package (extend existing)

Already present: `User` (move to `identity` — see §3.4), `Household`, `Membership`, `Status`, `Role`, sentinel errors.

Add:

| Type | Fields | Notes |
|---|---|---|
| `Invite` | `Token`, `HouseholdID`, `CreatedBy`, `ExpiresAt`, `AcceptedAt?`, `AcceptedBy?` | Single-use, 7d TTL |
| `Preferences` | `UserID`, `Dietary[]`, `Allergies[]`, `AvoidText?`, `Locale` (`en-ZA`), `Units` (`metric`), `Currency` (`ZAR`), `UpdatedAt` | Defaults applied on first read if missing |
| `ActiveHousehold` | resolved `(Membership, Household)` | Convenience for middleware |

Keep existing sentinel errors; add `ErrInviteInvalid` (covers forged/unknown tokens without leaking existence of a household).

### 3.3 Provenance envelope (shared type)

```go
// internal/domain/provenance/provenance.go
type Record struct {
    SourceType  SourceType // photo|url|paste|share|voice|manual|seed
    SourceRef   string
    ImportedBy  string
    ImportedAt  time.Time
    RightsState RightsState // personal_use (default)
}
```

No repository. Owning modules embed or write this alongside their entities. `core` only defines the type so the shape cannot drift.

### 3.4 Migration of `household.User`

The scaffold put `User` in `household`. Move it to `identity` as part of slice A. Update the one import in `ports/repository.go`. No behaviour change — pure relocation so auth and household domains stay cleanly separated.

## 4. DynamoDB key design

All items live in the single table (`saamly-{env}`). Keys are built only inside `adapters/dynamodb/keys.go`. Attribute names are PascalCase Go fields mapped to DynamoDB via `attributevalue` tags (`PK`, `SK`, `GSI1PK`, `GSI1SK`, `ttl`, entity-specific attrs).

| Entity | PK | SK | GSI1PK / GSI1SK | TTL attr |
|---|---|---|---|---|
| User | `USER#<id>` | `META` | `EMAIL#<normalised>` / `USER#<id>` | — |
| ProviderIdentity | `USER#<id>` | `PROV#<provider>` | `PROV#<provider>#<subject>` / `USER#<id>` | — |
| Preferences | `USER#<id>` | `PREFS` | — | — |
| RefreshSession | `USER#<id>` | `SESSION#<id>` | — | `ttl` (= ExpiresAt) |
| MagicLinkCode | `MAGIC#<email>` | `CODE` | — | `ttl` (= ExpiresAt) |
| Household | `HOUSE#<id>` | `META` | — | — |
| Membership (house view) | `HOUSE#<id>` | `MEMBER#<userId>` | — | — |
| Membership (user view) | `USER#<id>` | `HOUSE#<id>` | — | — |
| Invite | `INVITE#<token>` | `META` | `HOUSE#<id>` / `INVITE#<token>` | `ttl` (= ExpiresAt) |
| EventEnvelope | `HOUSE#<id>` | `EVENT#<ts>#<id>` | `EVENT#<type>` / `<ts>#<id>` (admin queries by type) | — |
| MeterCounter | `HOUSE#<id>` | `METER#<yyyy-mm>#<capability>` | — | — |
| Deletion marker | `HOUSE#<id>` | `META` (status=`deleting`, `purge_after`) | — | optional `ttl` at purge time |

**Active-household resolution (P0):** query `PK=USER#<id>`, `SK begins_with HOUSE#`. If exactly one membership → that household. If zero → no household (create-household flow). If more than one (invite-accept seam, D1) → prefer the membership most recently joined; `X-Household-ID` is reserved and validated when present, ignored when absent. Middleware never invents a household.

**Conditional writes that matter:**

- User create: `attribute_not_exists(PK)` on `USER#…/META`; email uniqueness via a conditional put on the GSI1 projection item (or a dedicated `EMAIL#…` item written in the same transaction).
- Invite accept: `attribute_not_exists(accepted_at)` + `expires_at > now`.
- Refresh rotate: delete old session + put new session in one transaction; on conditional failure treating the old session as already rotated → revoke chain.
- Household delete: set `status=deleting` only if currently `active`.

**Account-deletion writes:** delete the `USER#…` identity tree (META, PROV#*, PREFS, SESSION#*, HOUSE#* membership views); for each membership, delete the house-side `MEMBER#` item; revoke all sessions. Household-owned data is *not* deleted here (core.md §4.5).

**Account deletion does *not* rewrite event `user_id`s.** Deleting the identity tree *is* the anonymisation: an orphan ULID on historical events no longer joins to email, providers, or preferences, and a re-sign-up with the same email gets a new ULID. Rewriting the event log would turn an append-only record store into a mutable one (against feedback.md §4 / P0 architecture rule) for no privacy gain once the profile is gone. Admin and product surfaces resolve missing `USER#<id>` rows at read time as *a former member*. `created_by` / `imported_by` on household-owned entities follow the same read-time rule — not rewritten.

**Household purge (after 72h):** a scheduled rule — at P0, a daily Lambda trigger (or a one-shot worker job kicked by admin) scans `status=deleting AND purge_after < now` and deletes the `HOUSE#…` partition. Full cascade across inventory/recipes lands as those modules appear; for `core`-owned items the cascade is: META, MEMBER#*, INVITE#* via GSI1, EVENT#*, METER#*. Document the extension point so later modules register their purge hooks.

## 5. Ports to add or extend

| Port | Owner | Notes |
|---|---|---|
| `UserRepository` | ports | `FindByEmail`, `FindByID`, `FindByProvider`, `CreateWithProvider` (transacted), `Delete` |
| `SessionRepository` | ports | `Put`, `Get`, `Rotate` (transacted), `RevokeChain`, `RevokeAllForUser` |
| `MagicLinkRepository` | ports | `Put`, `Get`, `IncrementAttempts`, `MarkUsed` |
| `HouseholdRepository` | ports (extend existing) | `Create` (house + membership transacted), `Get`, `ListMembers`, `AddMember`, `RemoveMember`, `ScheduleDeletion`, `CancelDeletion`, `ListDueForPurge` |
| `InviteRepository` | ports | `Create`, `Get`, `Accept` (transacted with membership write) |
| `PreferencesRepository` | ports | `Get`, `Put` |
| `EventSink` | ports (existing) | DynamoDB adapter |
| `MeterSink` | ports (existing) | DynamoDB adapter — `UpdateItem` ADD on count + cost |
| `MembershipLoader` | httpapi (existing interface) | Implemented by DynamoDB adapter; used by Authn middleware |
| `IDGen` | ports (existing) | ULID adapter (`adapters/ulid`) |
| `Clock` | ports (existing) | `SystemClock`; tests inject fixed clock |
| `TokenVerifier` | ports (existing) | Already scaffolded in `adapters/oidc` |
| `Mailer` | ports (existing) | Already scaffolded in `adapters/ses` |
| `Transactor` | ports (optional) | Only if repository methods prove insufficient; prefer repository-level transactions first |

## 6. Use cases

Each use case is a struct with explicit dependencies (ports + clock + idgen + sink), a single exported `Execute` / named method, and unit tests with fakes. Handlers translate HTTP ↔ use-case I/O and map domain errors to problem+json.

### 6.1 Auth (`application/auth`)

| Use case | Input | Behaviour (core.md) | Events |
|---|---|---|---|
| `Exchange` | provider, id_token, device_label | Verify via `TokenVerifier` → find-or-create by email → issue session | `core.user_signed_up` or `core.user_signed_in` |
| `RequestMagicLink` | email | Always 202; generate code+token; store hashed; send via `Mailer`; on mailer failure return typed error (honest copy) | none (avoid enumeration via events) |
| `VerifyMagicLink` | email + code **or** token | Check TTL/attempts/used → find-or-create → mark used → issue session | `core.user_signed_up` / `core.user_signed_in` |
| `Refresh` | refresh_token | Rotate; reuse of rotated token → revoke chain → `ErrSessionRevoked` | none |
| `Logout` | session subject + refresh_token? | Revoke current chain | none |

Session issue returns `{session_token, refresh_token, expires_at}` matching `api/consumer.yaml`. Access JWT: HS256, `iss=saamly`, `aud=consumer`, `sub=userID`, 1h TTL (existing `oidc.SessionManager`). Refresh: 32-byte random, hex-encoded to client, SHA-256 hashed at rest.

**Find-or-create algorithm (binding):**

1. Look up `ProviderIdentity` by `(provider, subject)`.
2. If found → load user → done (sign-in).
3. Else look up user by normalised email.
4. If found → attach provider → done (sign-in, linked).
5. Else create user + provider in one transaction → done (sign-up).

Magic-link provider value is `magic`; subject is the normalised email.

### 6.2 Household (`application/household`)

| Use case | Behaviour | Events |
|---|---|---|
| `Create` | Create household + membership (role=`member`); name optional | `core.household_created` |
| `GetCurrent` | Active household + members; 404 if none | — |
| `CreateInvite` | Token = ULID; 7d expiry; return shareable URL path `/invites/{token}` | `core.invite_created` |
| `AcceptInvite` | Validate token; add membership; mark accepted; P0: switches active household by being most-recent join | `core.invite_accepted` |
| `RemoveMember` | Leave (`me`) or remove other; last member → schedule deletion | `core.member_removed` (+ `core.household_deleted` if last) |
| `ScheduleDeletion` | Typed confirmation phrase required at handler; status=`deleting`, `purge_after=now+72h` | `core.household_deleted` (requested; purge is a later system event if we need it) |
| `CancelDeletion` | Only while `deleting` and before `purge_after` | — |
| `PurgeDue` | Worker/cron entry: delete partitions past grace | — |

Typed confirmation phrases (handler-level, brand voice): account delete → exact match `"delete my account"`; household delete → exact match `"delete household"`. Wrong phrase → `400` with plain copy, no side effects.

### 6.3 Profile (`application/profile`)

| Use case | Behaviour | Events |
|---|---|---|
| `GetMe` | User + active household id + preferences (defaults if missing) | — |
| `UpdatePreferences` | Upsert; record `fields_changed` (names only) | `core.preferences_updated` |
| `DeleteAccount` | Typed confirmation; delete identity tree; leave households (or schedule household deletion if last member); revoke all sessions. Events keep their original `user_id` (orphan after delete — see §4) | `core.account_deleted` |

## 7. HTTP surface

Replace the `501` stubs in `httpapi.NewServer` slice-by-slice. Contract changes land in `api/consumer.yaml` / `api/admin.yaml` *before* handlers (RFC 001 §6). Spectral must stay at zero errors.

**Auth errors never enumerate.** `exchange` / `verify` failures → `401` with title `unauthorized` and detail *We couldn't sign you in — try again or use another method.* Magic-link request always `202`. Invite errors use the brand copy from core.md §10 (`That invite has expired — ask for a fresh one`) via problem+json `detail`.

**Admin:** `GET /v1/admin/households/{id}` returns household meta + members + preference summaries (no raw event dump — that is `GET /v1/admin/events`). Every call already audit-logged by `AdminAuth` middleware.

**OpenAPI gaps to close in slice A** (before handlers):

- `DELETE /me`, `GET/PUT /me/preferences`
- `POST /households`, `POST /households/{id}/invites`, `POST /invites/{token}/accept`
- `DELETE /households/{id}/members/{userId}`, `DELETE /households/{id}`, `POST /households/{id}/cancel-deletion`
- `GET /admin/households/{id}`
- Problem responses for `400` / `409` where domain conflicts apply (`ErrAlreadyMember`, `ErrLastMember`, etc.)

## 8. Middleware and composition

`cmd/api` wiring after `core`:

1. Build DynamoDB repositories + sinks + ULID gen + clock.
2. Build `oidc.SessionManager` (existing); build `ProviderVerifier`s only when audiences are configured.
3. Build use cases.
4. Pass `MembershipLoader` (DynamoDB) into `httpapi.Deps` — replacing the scaffold's `nil`.
5. Dev bypass (`Authorization: Bearer dev-token` → `dev-user`) remains local-only. Slice A seeds a `dev-user` + household on first request when `SAAMLY_DEV_AUTH_BYPASS=true`, so dogfooding the household endpoints does not require OAuth credentials.

`cmd/worker` gains a `core.purge` job type (envelope `{"type":"core.purge"}`) that runs `PurgeDue`. Scheduling: EventBridge daily rule → SQS, added in the Terraform `api` module in slice E. Until then, purge is invocable via `make purge-due` against the local process or a manual admin action.

## 9. Delivery slices

Implement and merge as five small PRs. Each slice is independently demoable and keeps `main` green.

| Slice | Delivers | Demo |
|---|---|---|
| **A — Identity + sessions** | Domain move; User/Session/MagicLink repos; `Exchange` (dev + real verifiers when configured); `Refresh` / `Logout`; OpenAPI auth paths live; ULID adapter; EventSink (write path only) | `curl` exchange with `dev-token` path replaced by issuing a real session for a seeded user; refresh round-trip |
| **B — Magic link** | `RequestMagicLink` / `VerifyMagicLink`; SES adapter wired; in-process rate limit (per email, 1/min; per IP, 10/min) | Local floci SES capture; verify with code → session |
| **C — Households** | Create / current / invite / accept / leave / remove; MembershipLoader wired into Authn; preferences get/put | Two seeded users: create → invite → accept → both see same household |
| **D — Deletion** | Account delete; household delete + cancel; purge job; read-time “former member” resolution where admin/product shows actors | Delete household → still visible as `deleting` → cancel; or wait/force purge |
| **E — Metering + admin inspect** | MeterSink; admin `GET /households/{id}`; EventBridge purge schedule in Terraform; seed script for local dogfood | Admin curl with allowlisted email returns household; meter increment unit-tested |

**Exit criteria for "core done" (P0a):** all endpoints in core.md §6 return real responses (no `501`); unit tests cover find-or-create, refresh reuse detection, invite expiry, last-member deletion; one integration test against floci covers create-household → invite → accept; `make test` and Spectral stay green.

## 10. Testing strategy

| Layer | What | How |
|---|---|---|
| Domain | Invariants (e.g. last-member cannot leave without deletion) | Table-driven pure tests |
| Use cases | Every use case with fake ports + fixed clock | Assert state changes, emitted events, error mapping |
| Adapters | DynamoDB key helpers; conditional-write paths | floci integration test tagged `//go:build integration` |
| HTTP | Handler mapping + problem+json shapes | `httptest` with fake use cases (or thin handler tests) |
| Auth security | Refresh reuse; magic-link attempt lockout; enumeration-safe status codes | Dedicated use-case tests; one HTTP assertion per auth endpoint |

Fakes live next to the use case tests (`application/auth/fakes_test.go` pattern) or under `internal/ports/fakerepo` if reuse appears. Prefer per-package fakes until duplication hurts.

## 11. Configuration additions

Extend `platform/config` (and `.env.example`):

| Env | Purpose | Default |
|---|---|---|
| `SAAMLY_MAGIC_LINK_BASE_URL` | Link prefix in emails (`https://…/invites` or app deep-link) | `http://localhost:8080` |
| `SAAMLY_SES_SENDER` | From address | `hello@saamly.co.za` |
| `SAAMLY_REFRESH_TTL` | Already present | `720h` |
| `SAAMLY_HOUSEHOLD_PURGE_GRACE` | Grace before purge | `72h` |
| `SAAMLY_SEED_DEV_USER` | When bypass on, ensure `dev-user` exists | `true` locally |

## 12. Risks and open questions

| Risk / question | Proposal |
|---|---|
| Email uniqueness race under two concurrent first-sign-ins | TransactWrite on user META + email identity item; loser retries as sign-in |
| floci SES / TTL parity gaps | Integration tests assert behaviour; deployed `dev` is the safety net (RFC 001 §9) |
| Household purge across modules not yet built | Purge registry interface in `application/household`; inventory/recipes register later; document in this RFC §4 |
| Orphan `user_id`s on events after account delete | Accepted: IDs are opaque ULIDs with no joinable profile; read-time “former member”. PITR restore of a User row could re-link — treat full-table restore as an exceptional ops event, not a product path |
| Should `core.household_deleted` fire at request or at purge? | Fire at **request** (user intent); add `core.household_purged` as a system event in slice D if admin dashboards need both |
| In-process magic-link rate limits are per-Lambda-instance | Accept at P0 dogfood volume; move to API Gateway usage plans at P1 |
| Google OAuth verification lag | Microsoft + magic link unblock dogfood (core.md §9); Exchange accepts whichever verifiers are configured |

**Open questions for decision before slice A:**

1. Confirm deep-link base URL scheme for magic links / invites (custom scheme vs https app-link) — affects `SAAMLY_MAGIC_LINK_BASE_URL` only; default to https placeholder until mobile exists.
2. Confirm typed confirmation strings (`delete my account` / `delete household`) — brand-aligned proposal above; change here if preferred copy differs.
3. Confirm GSI1 for events-by-type is worth the write cost at P0 — proposal: yes, admin event browser needs it (feedback.md §5); revisit if write amplification shows up.

## 13. Out of scope reminders

Do not sneak into `core` PRs:

- Capture upload / draft endpoints (capture module).
- Taxonomy resolution or review queue.
- Inventory item writes.
- Async event consumers (feedback.md P0 rule: events are records, not transport).
- Rewriting historical event `user_id`s on account deletion (identity-tree delete is sufficient — §4).
- Owner role, multi-household picker UX, email change.

## 14. Implementation checklist (summary)

When this RFC is accepted:

1. Expand `api/consumer.yaml` + `api/admin.yaml` for the §7 gaps; `make spec-lint`.
2. Slice A → E as sequenced above; each PR references this RFC and the relevant core.md section.
3. Update `core.md` status to `Accepted` / bump to v0.2 only if product behaviour changes; technical drift belongs here.
4. Mark RFC 001 §15 step 3 (`core`) as in progress in a one-line amendment note when slice A merges.
