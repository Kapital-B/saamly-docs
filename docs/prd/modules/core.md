# Module spec — `core`

Version 0.1 · July 2026 · Status: Draft · Phase: P0a
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.3 (§3.1, D1, D5, D6, D7) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` (§8, §10) · Implementation: `docs/rfc/RFC002_Core_Module_Implementation.md`

## 1. Purpose and pillar

`core` is the foundation every other module stands on: identity and auth, the household-first data model (D1), member profiles and preferences, privacy controls (retention, deletion, household removal), provenance on imported content (D6), the event-sink infrastructure behind `feedback` (D5), and per-household AI/vision cost metering (D7). It serves all three pillars by making "household" a first-class, shared, trusted context. `core` owns no user-facing feature of its own beyond account and household management — its job is to make the other modules correct by construction.

## 2. Users and jobs

| User | Jobs |
|---|---|
| Household member (P0: David, his wife) | Sign in without a password; create or join a household; invite the other person; record allergies, dietary needs and preferences; leave; delete their account and data |
| Team (admin app) | Read-only, logged inspection of households for debugging (D9) |
| Other modules (internal client) | Resolve "who is asking and for which household"; attach provenance; append events; increment metering counters |

## 3. Scope: Now / Next / Later

- **Now (P0a):** OAuth (Google, Microsoft) plus magic-link auth; session tokens (issue, refresh, revoke); create household; invite link; join; 2-member household; member preferences (allergies, dietary needs); leave/remove member; account deletion; household deletion with grace period; provenance envelope; event sink; metering counters; admin read-only inspection.
- **Next (P1):** multi-household membership UX (schema already supports it); member roles and permissions (owner/member); account recovery and email-change flows; richer household profile (name, photo, defaults).
- **Later (P2+):** household admin console; data-export tooling (POPIA/GDPR-style export); granular permission delegation; merging households.

**Out of scope:** roles/ACLs beyond "member" (RFC §8.1), cost splitting, social features, SSO for consumers, anything requiring a paywall decision (that is `commerce`, built on `core`'s metering data).

## 4. Flows

### 4.1 First run: create account and household

1. User opens app → sign-in screen → "Continue with Google" / "Continue with Microsoft" / "Email me a sign-in link".
2. OAuth: app obtains provider ID token → `POST /v1/auth/exchange` → server verifies → **find-or-create by verified email** (provider identities attach to one account; see §5) → session issued.
3. New user → create-household screen (household name optional, e.g. "Our home") → `POST /v1/households` → creator becomes first member → household becomes active.
4. Prompt to invite: "Invite your people to the plan." → `POST /v1/households/:id/invites` → shareable link via OS share sheet.
5. Preferences: conversational capture of allergies, dietary needs, anything to avoid → `PUT /v1/me/preferences`.
6. Lands on the P0a home (inventory scan — `inventory` module).

**Edge cases:** provider email already exists via another provider → link to the same account, never a duplicate; user closes app before creating a household → next sign-in resumes at create-household; invite skipped → household of one works fully (product test, PRD §1).

### 4.2 Magic link

1. `POST /v1/auth/magic-link {email}` → enumeration-safe 202 response regardless of account existence.
2. Email via SES with a link and a 6-digit code (code covers the "opened mail on desktop, app on phone" case). TTL 10 minutes, single-use, max 5 verify attempts, rate-limited per email and per IP.
3. `POST /v1/auth/verify {email, code}` → session issued (find-or-create).

**Edge cases:** SES failure → surfaced honestly ("We couldn't send the email — try again or use Google/Microsoft"), event logged; code expired → request a new one (old codes invalidated).

### 4.3 Accept invite

1. Invitee taps link → app (or web fallback page later) → sign-in if needed → `POST /v1/invites/:token/accept` → membership created → household becomes their active household.
2. **Edge cases:** token expired (7 days) or already used → plain explanation + "ask for a fresh invite"; already a member → no-op to household home; P0 single-household UX: if the invitee belongs to another household, accept switches their active household (membership in both is valid — D1 seam).

### 4.4 Sessions

Access JWT (1h, HS256) + opaque refresh token (30d, stored hashed, rotated on every use). Reuse of a rotated refresh token revokes the whole chain (theft detection) and forces sign-in. `POST /v1/auth/logout` revokes the current chain. Multiple devices per user supported (two phones, per §5).

### 4.5 Leave, remove, delete

- **Leave household:** `DELETE /v1/households/:id/members/me`. The member's *personal* items are deleted with them; shared items and their recipe imports remain (provenance attribution becomes "a former member"). Last member leaving = household deletion (below).
- **Remove member:** any member may remove another at P0 (no permissions, RFC §8.1) — same effects as leaving. Documented simplification; P1 introduces owner-only removal.
- **Delete account:** `DELETE /v1/me` with typed confirmation. Deletes PII immediately: user entity, provider identities, sessions, magic codes, preferences, memberships. Household data created by them survives only while the household does, attributed to "a former member" at read time (missing user row). Event `user_id`s are left intact — orphan ULIDs with no joinable profile (see RFC 002 §4).
- **Delete household:** `DELETE /v1/households/:id` with typed confirmation → status `deleting`, hidden from all members, **72-hour grace**, then purged: inventory, recipes, plans, lists, events, meters, invites, provenance, S3 uploads. Any member can cancel during grace. Predictable and reversible-then-final (PRD §7).

### 4.6 Empty, error and uncertain states

Sign-in is the only screen with no household context. Every household-scoped screen must render honestly for: household of one ("It's just you so far — invite your people to the plan."), no preferences set (never block planning; treat as no constraints), removed member mid-session (next request → signed-out state with explanation), and degraded SES/OIDC (offer the other path). No flow may silently lose an in-progress action — local state survives auth refresh (D8).

## 5. Data owned

All entities live in the single DynamoDB table (RFC §10); patterns are indicative, owned by the `dynamodb` adapter.

| Entity | Key fields | Key pattern (indicative) | Retention / notes |
|---|---|---|---|
| User | id, email (verified, unique), display_name, created_at | `PK=USER#<id> SK=META`; `GSI1PK=EMAIL#<normalised>` | Until account deletion |
| ProviderIdentity | provider (google/microsoft/magic), subject, email, linked_at | `PK=USER#<id> SK=PROV#<provider>`; GSI on provider+subject | Deleted with account |
| Household | id, name?, created_by, created_at, status (active/deleting), purge_after? | `PK=HOUSE#<id> SK=META` | 72h grace then purge |
| Membership | household_id, user_id, role (`member` — only value at P0), invited_by, joined_at | `PK=HOUSE#<id> SK=MEMBER#<userId>`; `PK=USER#<id> SK=HOUSE#<id>` | Schema supports multi-household and future roles from day one (D1) |
| Invite | token, household_id, created_by, expires_at, accepted_at? | `PK=INVITE#<token>` + TTL (7d) | Single-use, expiring |
| Preferences | user_id, dietary[], allergies[], avoid_text?, locale (en-ZA), units (metric), currency (ZAR), updated_at | `PK=USER#<id> SK=PREFS` | Deleted with account |
| RefreshSession | id (hashed), user_id, device_label?, created_at, expires_at, rotated_from?, revoked_at? | `PK=USER#<id> SK=SESSION#<id>` + TTL | 30d sliding, rotated |
| MagicLinkCode | email, code_hash, expires_at, attempts, used | `PK=MAGIC#<email>` + TTL (10m) | Single-use |
| ProvenanceRecord | entity_ref, source_type (photo/url/paste/share/voice/manual/seed), source_ref?, imported_by, imported_at, rights_state (personal_use default) | written alongside each imported entity by owning module | Lives with the entity; purged with it (D6) |
| EventEnvelope | event_id, type (`<module>.<past_tense>`), household_id?, user_id?, payload, occurred_at | `PK=HOUSE#<id> SK=EVENT#<ts>#<id>`; streams on | No TTL at P0; retention reviewed at P1; on account deletion `user_id` is left as an orphan (identity tree deleted; read-time “former member”) |
| MeterCounter | household_id, capability (vision_scan/recipe_parse/plan_generate/sku_match/llm_other), period (yyyy-mm), count, est_cost_cents | `PK=HOUSE#<id> SK=METER#<period>#<capability>` | Aggregated monthly; feeds `commerce` later (D7) |

`created_by` is recorded on every household-scoped entity across all modules so an owner/member split later needs no migration (RFC §8.1).

## 6. API boundary

Consumer surface (all under `/v1`, session JWT unless noted):

| Endpoint | Purpose |
|---|---|
| `POST /auth/exchange` | Provider ID token → session (unauthenticated, rate-limited) |
| `POST /auth/magic-link` · `POST /auth/verify` | Magic-link request/verify (unauthenticated, enumeration-safe, rate-limited) |
| `POST /auth/refresh` · `POST /auth/logout` | Rotate session / revoke chain |
| `GET /me` · `DELETE /me` | Profile; account deletion |
| `GET/PUT /me/preferences` | Allergies, dietary needs, locale/units |
| `POST /households` · `GET /households/current` | Create; active household with members |
| `POST /households/:id/invites` | Create invite link |
| `POST /invites/:token/accept` | Join household |
| `DELETE /households/:id/members/:userId` | Leave (`me`) / remove member |
| `DELETE /households/:id` | Household deletion (72h grace) · `POST /households/:id/cancel-deletion` |

Admin surface (Entra JWT, allowlisted, all reads logged): `GET /v1/admin/households/:id` — read-only household inspection for debugging (D9).

**Household context resolution (for all modules):** auth middleware validates the token, loads memberships, and resolves the active household — P0: the user's household, auto-selected; `X-Household-ID` header reserved and validated for the multi-household future. Every module receives `(user, household, role)` — never re-resolves identity itself.

**Internal ports (RFC §3):** `TokenVerifier`, `Mailer`, `EventSink.Append(EventEnvelope)`, `MeterSink.Increment(household, capability, cost)`, `Clock`, `IdGen`. `core` defines the sink contracts; `feedback` owns event semantics (§7).

**Error semantics:** typed `problem+json`; auth endpoints never reveal account existence; invite errors distinguish expired/used/invalid without leaking household data.

## 7. Dependencies

- **Consumes:** nothing upstream — first module. Infrastructure: SES (magic links), provider JWKS (Google, Microsoft), SSM (session key).
- **Consumed by:** every module — household context, provenance envelope, `EventSink`, `MeterSink`.
- **Events emitted:** `core.user_signed_up`, `core.user_signed_in`, `core.household_created`, `core.invite_created`, `core.invite_accepted`, `core.member_removed`, `core.preferences_updated`, `core.account_deleted`, `core.household_deleted`. Naming convention (`<module>.<past_tense>`) is owned here and binding on all modules; event *schema* is defined in the `feedback` spec.

## 8. Metrics

| Metric (PRD §8/§14) | Source | P0 target |
|---|---|---|
| Invite acceptance rate | `core.invite_created` → `core.invite_accepted` | Measured (P1 target set later); dogfood: both members joined |
| Magic-link funnel | request → verify success | ≥90% verify success; SES sandbox exit is a P0a task |
| Auth reliability | exchange/verify error rates, refresh-chain revocations | No unexplained forced sign-ins during dogfood |
| Deletion success | deletion jobs completed vs requested | 100%, verifiable in admin (PRD §14 "deletion success") |
| Time to first household | sign-up → `core.household_created` | Measured; feeds activation analysis with `feedback` |
| Cost per active household | MeterCounters | Measured from first vision call (D7); budget set in `commerce` spec |

## 9. Risks

| Risk | Mitigation |
|---|---|
| Household schema mistakes are the hardest to reverse | D1 enforced in review; roles field and multi-household membership exist in schema even though P0 UX uses neither |
| SES deliverability (sandbox, region gaps) | OAuth as equal first-class path; production-access request early in P0a; honest failure copy (§4.2) |
| Account enumeration via auth endpoints | Enumeration-safe responses, rate limits, single-use expiring codes |
| Refresh-token theft | Rotation + reuse detection revokes the chain; tokens stored hashed |
| "Any member can remove/delete anything" is abused | Accepted P0 simplification (two trusted users); destructive actions need typed confirmation; household deletion has 72h grace; P1 introduces owner role |
| Google consent-screen verification delays | Start OAuth app registration early; Microsoft + magic link unblock dogfood if Google lags |
| Event store growth | No TTL at P0 by design; retention decision scheduled for P1; streams already on for future export |

## 10. Voice and labels

- Category words: **household**, never "user group" or "account group"; **people**, not "consumers" (brand §7).
- Exact labels: *Invite your household* · *Who's eating?* (when participation arrives) · *Send invite* · *Join household* · *Leave household* · *Delete account*.
- Sign-in: *Continue with Google* · *Continue with Microsoft* · *Email me a sign-in link* — "No passwords to remember" as the supporting line.
- Preferences prompt, conversational per "Tell us about your week": *Any allergies or foods to avoid?* — never a judgemental or clinical tone; convenience and budget are never shamed (brand guardrail).
- Empty household: *It's just you so far — invite your people to the plan.*
- Deletion copy is plain and final: *This deletes your account and your data. It can't be undone.* No guilt, no dark patterns, cancel always available.
- Errors name the action, not the system: *That invite has expired — ask for a fresh one*, never "invalid token".
