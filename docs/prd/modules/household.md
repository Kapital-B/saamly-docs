# Module spec — `household`

Version 0.1 · August 2026 · Status: Draft · Phase: P0c
Parent: `docs/prd/Saamly_PRD_Parent.md` v0.4 (§3.9, D1, D6, D8) · Technical: `docs/rfc/RFC001_Initial_Technical_Structure.md` · Core implementation: `docs/rfc/RFC002_Core_Module_Implementation.md` · P0c implementation: `docs/rfc/RFC008_The_Week_Implementation.md` · Depends on: `core.md`; integrates `inventory.md`, `recipes.md`, `meal-plans.md`, `shopping.md`, `feedback.md`

## 1. Purpose and pillar

`household` is the collaboration layer that makes separately owned features feel like one shared home. `core` already owns the household entity, membership, invites, auth context and deletion; this module owns the multi-member experience on top: a shared week, a shared list, clear personal boundaries and predictable reconciliation between two devices. It serves pillars 1 and 3 by making planning and shopping genuinely household activities without introducing chat, voting or a second household backend.

P0c succeeds when two members can see and change the same plan and list for two consecutive weeks without coordinating in a group chat. A household of one must remain fully useful.

## 2. Users and jobs

| User | Jobs |
|---|---|
| Household member planning | See who shares the household; propose and accept one week; trust that the other member sees it |
| Household member shopping | Use the same list, including offline; see completed changes without wondering whether they synced |
| Household of one | Plan and shop without being blocked by an invite or empty collaborator state |
| Feature modules | Receive household identity and member context from `core`; apply consistent visibility, attribution and conflict rules |
| Team | Distinguish one-person feature use from real household collaboration and measure the P0c gate |

## 3. Scope: Now / Next / Later

- **Now (P0c):** collaboration across the existing shared inventory and recipe box plus the new weekly plan and shopping list; member-aware presentation; refresh and conflict behaviour; offline-first shared list.
- **Next (P1):** meal participation (*Who's eating?*), voting, shopping assignments, cooking rotas, member roles and richer household profile.
- **Later (P2+):** cost splitting, multi-household workflows, granular permissions and delegated household administration.

**Already owned by `core`, not reimplemented here:** household creation, active-household resolution, invite lifecycle, membership records, preferences storage, leave/remove, account deletion and household purge.

**Out of scope at P0c:** chat, comments, presence indicators, push notifications, real-time cursors, voting, assignments, roles beyond `member`, owner-only permissions, a two-member database cap and public sharing.

The P0c exit gate is exercised on the two Flutter phones because shopping offline is a mobile requirement. `saamly-web` may add planner/list parity for development and dogfood convenience, but web parity does not block the phase gate.

## 4. Flows

### 4.1 Enter the shared workspace

After auth, `core` resolves the active household and returns its member roster. The client presents the product as one household workspace with these top-level destinations:

- *This week* — owned by `meal plans`;
- *What you have* — owned by `inventory`;
- *Your shared list* — owned by `shopping`;
- *Recipes* — owned by `recipes` and available from the planner and navigation.

The exact mobile navigation control may change, but the four surfaces share one household context. The client never asks each feature to select or infer a household independently.

For a household of one, show *It's just you so far — invite your people to the plan* without blocking any destination.

### 4.2 Plan together

1. Either member may generate, swap, regenerate or accept the shared plan under the rules in `meal-plans.md`.
2. The plan header may show household member display names or derived initials from `GET /households/current`; this is context, not live presence and requires no avatar field.
3. After one member mutates the plan, their client updates optimistically after the server accepts the write and other clients discover it on foreground, pull-to-refresh or polling.
4. Different dinner slots merge independently. A stale edit to the same slot is not silently overwritten; the client refreshes and says *This week changed while you were editing. Review it before saving.*
5. `updated_by` resolves to a current display name or *a former member*. Missing identity never breaks the plan.

P0c has one flat `member` role. Either member may change the shared plan. Participation, pins, votes and fixed-meal ownership arrive later.

### 4.3 Shop together

1. Both members read the same server list and keep a local copy on device.
2. Item actions are idempotent set states (`open`, `bought`, `already_have`), not blind toggles.
3. Offline actions update the local list and queue in the outbox. On reconnect, the client sends mutations in order, then performs a delta pull.
4. Two members setting the same state is a no-op after the first write. Different status writes use the latest server-accepted timestamp; content edits are a separate field group and cannot erase a check-off.
5. Completed rows remain visible for the active week. The server always persists `updated_by`; clients may show *Updated by …* where attribution helps the second shopper.

The shopping list is the required offline P0c surface. The plan may be online-first. Inventory and shopping are the two D8 local-first surfaces; P0c cannot exit until both have an on-device store and outbox. Recipe browsing may remain offline-tolerant rather than fully local-first.

### 4.4 Refresh and connectivity

P0c uses HTTP reconciliation, not sockets:

- invalidate and refetch after a local mutation;
- delta pull when a shared surface opens or the app returns to foreground;
- pull-to-refresh on plan and list;
- ordinary low-frequency polling while a shared plan/list is visible;
- outbox drain followed by delta pull after reconnect.

Push notifications, WebSockets and SSE wait until observed dogfood latency justifies them. The UI may say when data last synced; it must not display a green “live” indicator.

### 4.5 Personal and shared boundaries

- Household inventory is shared except items explicitly in state `personal`; those remain owner-visible and never satisfy a shared plan/list requirement.
- Recipes default to household visibility. A personal recipe is owner-only and cannot enter a shared plan until its owner chooses *Share with household*.
- Plans and shopping lists are always household-shared at P0c; there is no personal plan or private list.
- Member preferences remain per-user in `core`. P0c adds an internal `HouseholdPreferences` read port so `meal plans` can read the active members' union without copying preference content onto the household.
- Cross-household and hidden-personal reads return 404, not 403.

### 4.6 Membership changes and deletion

- A removed member loses the household context on the next request and enters the signed-out state defined by `core`. Before showing auth/onboarding, clients clear **all** data and queued mutations for that household: inventory, recipe caches, plans, shopping lists and outbox entries.
- Personal inventory is deleted under `core` rules. Shared plans, lists, recipes and inventory remain and resolve old attribution as *a former member*.
- Household deletion follows the existing 72-hour grace period. After grace, registered purge hooks hard-delete plans, shopping lists and their local-sync tombstones alongside existing household data.
- An invite is helpful but optional. P0c does not enforce exactly two members in storage; “two people” is the dogfood test context.

### 4.7 Empty and error states

- **One member:** invite prompt, never a blocker.
- **Other member has not opened the week/list:** do not nag or claim collaboration; the surface still works.
- **Offline list:** *Offline — changes will sync when you're back.*
- **Offline plan mutation:** retain unsaved form state and ask to retry online; never imply the shared plan changed.
- **Conflict:** show the fresh server plan for consequential slot edits; silently reconcile idempotent list states.
- **Removed/deleting household:** explain that the household is unavailable; do not expose cached shared data after auth confirms removal.

## 5. Data owned

`household` owns **no second persisted household aggregate**. `core` remains the sole owner of `Household`, `Membership`, `Invite` and `Preferences`; sibling modules own inventory, recipes, plans and shopping lists.

This module defines collaboration fields that **plan and shopping-list entities** must carry. Existing inventory and recipe records keep their own established field groups and do not require a migration for this module:

| Field | Rule |
|---|---|
| `household_id` | required on every shared entity and mutation |
| `created_by` | actor that created the entity; preserved after membership removal |
| `updated_by` | actor for the current field-group revision |
| field-group `updated_at` | conflict boundary used for LWW / stale-write detection |
| `deleted_at?` | tombstone where a local-first client needs delta reconciliation |

The client may compose a `SharedWorkspace` view from the current household, current plan and current shopping list, but it is not stored or exposed as a new aggregate endpoint. Display names come from the live member roster; initials are derived client-side. Clients render missing users as *a former member*.

No member preference, recipe text, inventory content or list text is copied into membership records.

## 6. API boundary

There is **no new `/household/workspace` API** at P0c. Clients compose the experience from module-owned headless surfaces:

| Owner | Surface used |
|---|---|
| `core` | `GET /households/current`; existing invite/member/deletion endpoints |
| `meal plans` | current plan, generate, regenerate, swap, accept |
| `shopping` | current delta list and item mutations |
| `inventory` | existing full/delta reads and mutations |
| `recipes` | existing visible library and visibility mutations |

Every request uses the active household resolved by `core` middleware. `X-Household-ID` remains reserved for the future multi-household UX.

**Shared write contract:**

- mutations are explicit set operations and accept `Idempotency-Key`;
- expected field-group timestamps protect stale edits;
- 404 hides cross-household existence;
- `problem+json` conflicts include the changed field group and server timestamp;
- delta endpoints include tombstones;
- event append failure never fails the feature action.

No client joins data across households or bypasses module visibility rules to build the shared view.

## 7. Dependencies

- **Consumes:** `core` (active household, roster, preferences, identity rendering, auth context, deletion); module contracts from `inventory`, `recipes`, `meal plans` and `shopping`; `feedback` for collaboration metrics.
- **Consumed by:** consumer clients as a UX and sync contract. It has no service-domain consumers.
- **Events emitted:** none of its own at P0c. `core` records membership/invite actions; `meal_plans` and `shopping` record shared user actions. A generic `household.shared_surface_viewed` event would duplicate stronger domain evidence and is deliberately omitted.

Collaboration is derived by grouping the existing domain events by `household_id`, `week_start` and distinct `user_id`. Event payloads never include member names or household composition.

## 8. Metrics

| Metric (PRD §8/§14) | Source | P0c target |
|---|---|---|
| Consecutive weeks planned and shopped | same `week_start` has `meal_plans.plan_accepted` and `shopping.list_opened` or a shopping item interaction | 2 consecutive weeks |
| Shared planning participation | distinct users emitting plan actions per accepted week | measured, not required to be equal |
| Shared list participation | distinct users opening or mutating the list per week | both dogfood members use it during P0c |
| Group-chat fallback | weekly dogfood interview/check-in | none for plan or shopping coordination in the two exit weeks |
| Invite activation | `core.invite_created` → `core.invite_accepted` | both members joined |
| Sync trust | unresolved list conflicts, lost-action reports, outbox failures | zero lost actions |

The group-chat fallback part of the gate cannot be inferred honestly from app events alone; it remains a declared weekly dogfood check alongside telemetry.

## 9. Risks

| Risk | Mitigation |
|---|---|
| `household` duplicates `core` and creates two sources of truth | No owned aggregate or new workspace endpoint; core remains identity/membership owner |
| Shared UX leaks personal recipes or inventory | Server-side visibility rules; personal data never enters a shared plan/list |
| Polling feels stale | Foreground refresh, mutation invalidation and visible pull-to-refresh; add real-time transport only with evidence |
| Offline reconciliation loses a shopper's action | Local-first list, ordered outbox, idempotent set-state writes, delta + tombstones |
| Flat permissions become unsafe beyond dogfood | Explicit P0 simplification; owner/member roles arrive in P1 before wider rollout |
| Attribution breaks after account deletion | IDs remain on records; missing user renders as *a former member* |
| Collaboration is claimed from views alone | Gate combines domain actions with a weekly qualitative fallback check |
| Household module becomes chat/project management | P0c boundary is shared food surfaces only; no comments, votes, assignments or rotas |

## 10. Voice and labels

- Category: *household* and *people*, never “workspace users” or “collaborators”.
- Shared surfaces: *This week* · *What you have* · *Recipes* · *Your shared list*.
- Empty household: *It's just you so far — invite your people to the plan.*
- Conflict: *This week changed while you were editing. Review it before saving.*
- Attribution: *Updated by David*; missing identity: *Updated by a former member*.
- Connectivity: *Offline — changes will sync when you're back.* Never claim “live” when the client is polling.
- Participation language (*Who's eating?*), assignments and rotas are reserved for P1.
