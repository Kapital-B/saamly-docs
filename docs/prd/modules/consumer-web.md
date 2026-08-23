# Module spec — `consumer-web`

Version 0.1 · August 2026 · Status: Draft · Phase: P0a
Parent: `docs/prd/Saamly_PRD_Parent.md` · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md`
Depends on: `core.md`, `inventory.md`, `capture.md` · Sister client: `saamly-mobile`

## 1. Purpose and pillar

`consumer-web` is the browser client of the consumer APIs — the same
headless surface the Flutter app uses (D3). It exists so dogfood and
development can exercise sign-in, household, inventory and scan without
a phone. It is **not** `admin`: no taxonomy queue, no gate dashboard, no
household inspection beyond the signed-in household's own data.

It serves the same three pillars as the mobile app, through the same
API. A consumer-web release never waits on `admin`, and `admin` being
down affects nobody's dinner.

## 2. Users and jobs

| User | Jobs |
|---|---|
| Household member (David during dogfood) | Walk the P0 kitchen flows in a browser; confirm a scan; add/edit items |
| Future household | Same as mobile, once hosted |

## 3. Scope: Now / Next / Later

- **Now (P0a):** Magic-link sign-in + local `dev-token` bypass; household create/join/invite; preferences; inventory list with manual add, edit, quick actions and undo; photo scan via file picker → presigned PUT → draft poll → review (cosmetic / material) → confirm / discard. Repo: `saamly-web`. Stack copied from `cloudmanager-web` (Vite, React, TanStack Query, Tailwind tokens, MSW).
- **Next (P1):** Week planner, recipe import and shared list once those land on mobile; S3 + CloudFront hosting.
- **Later (P2+):** Offline, camera capture, OAuth exchange UI.

**Out of scope:** anything on `/v1/admin/*`; Entra allowlist; write paths into other households.

## 4. Flows

Parity with `saamly-mobile` (RFC 002 / 004 / 005):

1. Sign in → verify code (or Continue as local user) → `GET /v1/me`.
2. No `household_id` → create or join → optional invite → preferences.
3. Home is **What you have**, grouped Fridge / Freezer / Cupboard / Spice rack.
4. Scan: pick locations → photos → upload → poll draft → confirm accepted items.

## 5. API boundary

Consumes `consumer.yaml` only. Client: hand-rolled fetch wrapper + types
in `saamly-web/src/api`, against a vendored spec (`vendor/api/`). Tokens:
Bearer session JWT in `localStorage`, refresh on 401. No cookies, no CSRF.

## 6. Voice and labels

Reuse the mobile / design copy: *Plan saam. Eat better.* · *What you have*
· *Have* · *Probably* · *Use first* · *Personal*. AI is never named.
Chips always pair colour with a word.
