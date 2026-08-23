# Saamly — Product Requirements (Parent)

Version 0.4 · July 2026 · Status: Draft

**Plan saam. Eat better.**

This is the parent PRD for Saamly. It defines the module decomposition, the phased roadmap (starting with a private dogfood phase), the cross-cutting architecture decisions, and the success gates that govern progression between phases. Detailed requirements live in per-module specs (see §12); this document owns only what must be true across all modules.

**Sources:** `docs/strategy/Saamly_Product_Strategy_v3.docx` (strategy), `docs/strategy/Saamly_Brand_Foundations_and_Messaging.docx` (brand). Where this document conflicts with the strategy, the strategy wins until this PRD is updated.

---

## 1. Vision and product test

Saamly is a South African household meal-planning platform that helps people plan an entire week around the food they already have, the people who will be eating, their schedules and preferences, and prices across multiple grocery retailers. It converts recipes from almost any source into structured plans and turns the missing ingredients into a consolidated, comparable shopping basket.

> **PRODUCT TEST** — Would Saamly create meaningful value for one household if no public community existed and no other user had published a recipe? The required answer is yes.

The product stands on three independent pillars:

1. **Household-aware weekly optimisation** — plan across the week, not one meal at a time, using existing inventory, shared preferences, ingredient reuse, leftovers, time and budget.
2. **Frictionless personal recipe intelligence** — capture recipes from photos, screenshots, links, text, voice and mobile sharing; structure them automatically; ask only for material corrections.
3. **Neutral cross-retailer commerce** — compare the missing basket across South African retailers and optimise for the household, not for a single store.

## 2. Scope of this document

The parent PRD owns:

- The module map (§3) and each module's Now/Next/Later boundary.
- The phased roadmap and phase exit gates (§4).
- Cross-cutting architecture and data decisions (§6) that per-module specs must respect.
- Privacy, trust and brand constraints (§7, §9).
- Success metrics and open questions (§8, §11).

Per-module specs own detailed user stories, flows, data models, API surfaces and acceptance criteria. The spec template is in Appendix A.

## 3. Module map

Modules are grouped into five layers plus one operational workstream. Build order within a layer follows the dependency arrows; layers build roughly bottom-up.

| Layer | Module | Purpose | Pillar |
|---|---|---|---|
| Foundation | `core` | Identity, household-first data model, preferences, privacy/provenance, instrumentation, cost metering | All |
| Foundation | `capture` | One shared intake pipeline: photo, screenshot, URL, share sheet, text, voice → structured draft → confirmation | 2 (also serves 1, 3) |
| Foundation | `taxonomy` | Ingredient ontology: concepts, aliases, categories, forms, units, substitutes; automated resolution with a promotion gate | 1, 2, 3 |
| Foundation | `feedback` | Event instrumentation and the learning loop: saves, swaps, skips, cooked, corrections | All |
| Experience | `inventory` | Kitchen awareness: photo scan → confidence states → lightweight confirmation | 1 |
| Experience | `recipes` | Recipe capture, parsing, personal + household library with provenance | 2 |
| Experience | `meal plans` | Household-aware weekly planner with explanations | 1 |
| Experience | `shopping` | Aggregated "still needed" list; later, basket construction and comparison | 3 |
| Experience | `household` | Invitations, shared surfaces, participation, permissions | 1 |
| Experience | `cooking` | Cook-tonight view, mark-cooked, leftovers (placeholder, post-dogfood) | 1 |
| Commerce | `retailer catalogs` | SKU/pack/price/availability ingestion; ingredient-to-SKU matching | 3 |
| Commerce | `commerce` | Subscription tiers, usage metering, sponsorship (later) | — |
| Growth | `sharing` | Share links, web previews, invite loops (the MVP slice of "social") | — |
| Growth | `community` | Public publishing, follows, comments, creator features — **stage-gated** | — |
| Internal | `admin` | React web app for dashboards, the taxonomy review queue, data analysis and ops tooling | Internal |

**Workstream (not software):** `seeded content` — sourcing, testing and rights-clearing the original South African seed recipe library.

### 3.1 `core`

Identity and auth, the household-first schema, member profiles and preferences (including allergies and dietary needs), privacy controls (retention, deletion, household removal), provenance on imported content, the event log, and AI/vision cost-metering hooks.

- **Now:** OAuth (Google, Microsoft) plus magic-link auth; household invite; 2-member household; preference capture; deletion controls; event log; metering counters.
- **Next:** multi-household membership, member roles and permissions.
- **Later:** household admin console, data-export tooling.

### 3.2 `capture`

One intake pipeline for everything the household sends to Saamly. Every input route feeds the same draft-and-confirm flow: structure automatically, attach provenance, expose uncertainty, and ask for confirmation only where it could materially change a meal, safety outcome or basket.

- **Now:** photo and pasted-text intake for inventory and recipes; draft-and-confirm UX; provenance records.
- **Next:** URL import with structured extraction, mobile share sheet, voice capture.
- **Later:** receipt and barcode intake, retailer-order import, bulk import.

### 3.3 `taxonomy`

The ingredient ontology and its resolution service. Accuracy comes from making errors cheap, not from upfront curation: deterministic-first resolution, provisional concept states with a promotion gate, free-text fallback so unmatched ingredients never block or corrupt a plan, and user corrections written back as aliases.

- **Now:** LLM-seeded starter set (~1,000 SA concepts with aliases, shallow 2-level categories, forms, common units); resolution cascade (alias → fuzzy → embedding → LLM-proposes-new); provisional → canonical promotion via a review queue (operated in `admin`); free-text fallback.
- **Next:** unit conversions, perishability classes (powers "use first"), substitute relationships (powers the planner).
- **Later:** pack-size norms, retailer taxonomy mapping, SKU-matching support, verified allergen data.

**Never fully automated:** allergen tags, safety-relevant substitutions, and unit conversions that feed basket math. These require verified sources.

### 3.4 `feedback`

The instrumentation backbone and learning loop. Logs the events that make every other module smarter: saves, swaps, skips, pins, cooked meals, match corrections, confirmation burden, capture latency.

- **Now:** event schema and log; correction write-back to `taxonomy` aliases; dogfood dashboards surfaced in `admin`.
- **Next:** preference signals feed the planner; kitchen-confidence model inputs.
- **Later:** aggregated, privacy-thresholded insight products (commercially separated from personal data).

### 3.5 `inventory`

Kitchen awareness — not a warehouse ledger. Photo scanning produces candidate items with confidence states; the user confirms only what matters. States: **Have**, **Probably have**, **Use first**, **Personal**, **Out**. Uncertain items go to "check at home" rather than being silently trusted.

- **Now:** photo scan of fridge/freezer/cupboards → candidate list → confirm; manual add/edit/remove; confidence states.
- **Next:** "use first" surfacing in plans, mark-cooked deduction (with `cooking`), "used this" shortcuts.
- **Later:** receipt/barcode updates, retailer-order import, expiry estimation, continuous reconciliation.

### 3.6 `recipes`

The personal and household recipe library. Capture feels like sending content to an inbox, not completing a form. Parsing extracts title, ingredients, quantities, steps, servings, prep time and source; normalises units; resolves ingredients against `taxonomy`; flags material uncertainty. Imported recipes are private by default and retain provenance.

- **Now:** import via photo, pasted text and URL; parsed drafts with confirm step; personal + household library; provenance.
- **Next:** share-sheet and voice capture; serving adaptation; substitution suggestions.
- **Later:** licensed collection tooling, controlled adaptation for dietary constraints.

### 3.7 `meal plans`

The weekly planner — the heart of the product. Selects meals as a portfolio: uses confirmed and use-first ingredients, reuses purchases across meals, respects participation, preferences, allergies, time and budget, and explains its choices ("Wednesday uses the spinach first"). Users stay in control: swap, pin, add fixed meals, or plan manually.

- **Now (dogfood):** planner-lite — suggest meals from the library ranked by inventory fit, with reasons; swap and regenerate.
- **Next:** full weekly optimisation across reuse, participation, time and budget; pinned and fixed meals; conversational refinement.
- **Later:** leftover chains, batch-prep scheduling, deeper nutrition goals, multi-week planning.

### 3.8 `shopping`

Turns the plan into what remains to buy. Aggregates requirements across the week, normalises units, deducts inventory, and separates uncertain items into "check at home".

- **Now:** shared "still needed" list from the plan; check-at-home section; manual additions; bought/have toggles.
- **Next:** best single-store basket via `retailer catalogs`.
- **Later:** split-shop comparison including fees, minimums and effort; preferred-store basket; direct cart handoff.

### 3.9 `household`

The collaborative workspace. The household is a first-class data concept (in `core`); this module owns the multi-member experience on top of it.

- **Now:** invite link; shared inventory, plan and shopping list for two members.
- **Next:** meal participation (who's eating), voting, shopping assignments, simple cooking rotas.
- **Later:** cost splitting, multi-household workflows, richer permissions.

### 3.10 `cooking`

Placeholder for the execution layer: "cook tonight" view, mark-cooked, leftover capture. Exists in the roadmap so `meal plans` and `inventory` data models anticipate deduction and leftovers from the start.

- **Now:** nothing built.
- **Next:** mark-cooked with inventory deduction; leftover records that feed the planner.
- **Later:** prep scheduling, guided cooking mode.

### 3.11 `retailer catalogs`

Retailer data ingestion and matching. Acquires SKU, pack-size, price and availability data for 2–3 South African retailers; resolves ingredient needs against products with confidence scores and substitution options. Carries the legal/compliance risk of the platform — acquisition methods require legal review.

- **Now:** nothing built (P1 entry point).
- **Next:** catalog ingestion for 2–3 retailers; ingredient-to-SKU matching with confidence; price freshness tracking.
- **Later:** formal feeds, attribution, direct cart handoff support.

### 3.12 `commerce`

Monetisation. Sequenced so early survival needs no retailer cooperation: consumer subscription first; sponsorship and affiliate later.

- **Now:** metering hooks only (per-household AI/vision usage counters in `core`).
- **Next:** free/paid split — free capture and basic planning; paid household optimisation, advanced constraints and comparison.
- **Later:** CPG sponsorship tests, affiliate/basket commission, price-comparison monetisation with transparent labelling.

### 3.13 `sharing`

The MVP slice of "social". Supports the growth loops without building a network: household invite links, recipe/plan share links with useful web previews, recipient save.

- **Now:** household invite links.
- **Next:** recipe share links with web preview; recipient save-to-library.
- **Later:** shoppable shared-plan previews.

### 3.14 `community`

Public profiles, publishing, follows, comments, challenges, creator monetisation. **Stage-gated** per strategy §5.3 and §14: no investment until sharing volume, recipient saves and voluntary public publishing demonstrate sustained demand. Nothing in P0 or P1 depends on this module.

### 3.15 `seeded content` (workstream)

Original, rights-cleared South African recipes that make the planner useful before users have imported enough of their own. Focused on local routines, budgets and ingredients. Recipe count is not a vanity metric — a small, high-quality, well-tested set beats volume.

### 3.16 `admin`

Internal web application (React) — the second client of the internal APIs (D3), after the consumer app. Carries the operational surfaces the consumer app should never have: dashboards over `feedback` events, the `taxonomy` review queue (promote, merge or reject provisional concepts; edit aliases and categories), data analysis, and later seed-content and retailer-catalog operations.

- **Now:** team-gated auth; dogfood dashboards (capture latency, confirmation burden, match correction rates); taxonomy review queue; read-only household inspection for debugging.
- **Next:** seed-content management; catalog-ingestion monitoring and match-quality tooling; cost dashboards built on D7 metering data.
- **Later:** team roles; feature flags and experiment tooling; support workflows (deletion requests, price disputes).

`admin` favours function over polish and never gates consumer features — but it is bound by the same privacy rules (§7): access to household data is read-only and logged.

## 4. Phased roadmap

P0 is a private dogfood phase (one household: two members, sideloaded builds) that the strategy document does not cover; it de-risks the two hardest UX assumptions before any MVP spend. P1+ follows strategy §13.

| Phase | Modules in play | Scope | Exit gate |
|---|---|---|---|
| **P0a — Kitchen awareness** | `core` (lite), `capture`, `taxonomy` (lite), `inventory`, `feedback`, `admin` (lite), `sharing` (invite link) | Auth + 2-member household; photo scan → confirm → "what you have"; taxonomy seed + review queue in `admin`; event log + dogfood dashboards | Sideloaded on both phones; 2+ weeks of real use; scan+confirm is genuinely faster than manual entry; auto-resolution ≥85%, correction rate <10% |
| **P0b — Recipe box** | `recipes`, `capture` (URL), `taxonomy` | Import from photo, paste, URL; parsed drafts; household library | 20+ real recipes imported; drafts need minimal correction; both members import unprompted |
| **P0c — The week** | `meal plans` (lite), `shopping` (list only), `household` (shared surfaces) | Planner-lite suggestions with reasons; shared "still needed" + "check at home" list | The household plans and shops from Saamly for two consecutive weeks without falling back to the group chat |
| **P1 — MVP** | all Experience + `retailer catalogs`, `commerce` (metering → paid), `seeded content` | Strategy months 0–6: full weekly planning, household invites, 2–3 retailer catalogs, single-store basket comparison, seed library, privacy foundations | Strategy §14: weekly active planning households; repeat planning at 4 and 8 weeks; matched-ingredient rate; comparison engagement |
| **P2 — Deepen & monetise** | `cooking`, `commerce`, `inventory` (deduction), `shopping` (split-shop) | Strategy months 6–12: subscription launch, use-first/deduction, participation/voting/rotas, receipts/barcodes, split-shop with fees, first CPG tests | Free-to-paid conversion; contribution margin per household |
| **P3 — Platform & growth** | `community` (if gated open), white-label readiness | Strategy months 12–24: formal retailer relationships, cart handoff, narrow white-label pilot, aggregated insight — each behind its own stage gate | Strategy §14 stage gates |

**Explicitly out of P0:** retailer comparison, subscriptions, split-shop, voting/rotas, cost splitting, public publishing, voice capture, receipts/barcodes.

**Never descoped, even in P0:** the household-first schema, event instrumentation, provenance, and confidence/free-text-fallback behaviour. Retrofitting any of these later is disproportionately expensive.

## 5. Users and context

- **P0:** one household — two adults, two phones, one kitchen. Mixed cooking confidence; recipes scattered across screenshots, WhatsApp, cookbooks and memory. They are the review queue for `taxonomy` and the confirmation UX test subjects.
- **P1:** alpha households drawn from the brand audiences: busy couples, families, student digs/flatmates, value-conscious households, food-curious cooks. South African; English-first with intentional multilingual review later.

## 6. Cross-cutting architecture decisions

These bind every module spec.

- **D1 — Household-first schema.** Every relevant entity (inventory item, recipe, plan, list) belongs to a household; items can be personal or shared. A user may join multiple households over time. P0's two-person household uses the same schema as a future five-person digs.
- **D2 — One capture pipeline.** All intake routes produce the same draft object with provenance and uncertainty flags. No module builds its own parser or confirmation UX.
- **D3 — Headless capability boundaries.** Internal APIs mirror strategy §7.1: recipe capture/parsing, ingredient and unit normalisation, inventory interpretation and confidence, plan generation and explanation, ingredient-to-SKU matching, basket construction/pricing/comparison. Tenant and catalog identifiers explicit from the start. A modular monolith is expected; do not build multi-tenant self-service or general platform features before a credible buyer validates them.
- **D4 — Confidence and free-text fallback everywhere.** The system exposes uncertainty instead of inventing precision. No match is infinitely better than a confident wrong match; unmatched data remains usable as text and is treated conservatively by planners and baskets.
- **D5 — Event instrumentation from day one.** Every module emits the events defined in the `feedback` spec. If a feature can't be measured, it isn't done.
- **D6 — Private by default.** Imported recipes, inventory and preferences are private unless intentionally shared. Provenance is recorded for all imported content.
- **D7 — Cost metering hooks.** AI/vision calls are counted per household from P0, before any paywall exists. Subscription packaging will be built on this data.
- **D8 — Flutter mobile app, offline-tolerant.** The mobile app is Dart/Flutter. The shopping list and "what you have" must work in a supermarket with bad signal; local-first storage with background sync, and sync conflicts resolve without data loss. Camera intake and the OS share sheet are first-class requirements that drive plugin and native-extension choices.
- **D9 — React admin web app.** Internal operations (dashboards, taxonomy review, data analysis, content and catalog ops) live in a React web app — the second client of the internal APIs (D3). Internal tooling favours function over polish and never gates consumer features; access to household data is read-only and logged (§7).

## 7. Privacy and trust requirements

- Kitchen images, dietary requirements and household membership are sensitive personal data; retain raw images only as long as the user-visible feature and declared improvement purposes require.
- Provide predictable deletion and household-removal controls from P0.
- Separate personal household data from any future aggregated commercial insight; never sell identifiable recipe, health or pantry behaviour.
- Expose source, price freshness, match confidence and uncertainty wherever they affect a user decision.
- Recipe rights: imported recipes retain provenance; publishing copied content requires separate rights and attribution treatment (P3 concern, but the provenance schema must not preclude it).

## 8. Success metrics

**P0 gates** (dogfood — pass/fail before MVP spend):

| Metric | Target |
|---|---|
| Scan+confirm time vs manual entry for a fridge refresh | Faster, and both members agree it's faster |
| Taxonomy auto-resolution rate | ≥85% of ingredient strings |
| Match correction rate | <10% of resolutions |
| Review queue trend | Shrinking toward zero by end of P0b |
| Recipes imported and used in a plan | ≥20 by end of P0b |
| Consecutive weeks planned and shopped from Saamly | 2 by end of P0c |
| AI/vision cost per active household | Measured and within the budget set in the `commerce` spec |

**P1 metrics** (from strategy §14): weekly active planning households; plans accepted; meals completed; repeat planning after 4 and 8 weeks; time to first useful plan; recipe-import completion; share of plans using inventory; invite acceptance; matched-ingredient rate; basket coverage; comparison engagement; match corrections and price disputes; privacy contacts and deletion success.

## 9. Brand and voice constraints

Per-module specs and all UX copy follow the brand guide. Constraints that shape product behaviour:

- Use the brand vocabulary: "what you have" (not inventory management), "check at home" (not uncertain inventory), "add a recipe" (not recipe ingestion), "single-store shop or split shop" (not multi-retailer allocation), "tell us about your week" (not constraint configuration).
- Recommended labels exist and should be reused: *This week · Who's eating? · What you have · Use first · Check at home · Still needed · Your shared list · Best-value basket · Cook tonight · Swap meal · Keep this meal · Plan my week · Send to Saamly*.
- Name uncertainty plainly; never fabricate exact quantities or freshness ("We probably saw yoghurt. Check the amount before we leave it off your list.").
- Explain recommendations when the reason is useful: budget, expiry, reuse, time or participation.
- Keep AI behind the experience. Saamly plans, notices, suggests and compares; it does not announce that AI acted.
- South African English conventions and rand amounts; no shame around budgets, waste or convenience food.

## 10. Risks and mitigations (filtered to P0–P1)

| Risk | Mitigation |
|---|---|
| Inventory scanning becomes burdensome or inaccurate | Photo-first, confidence states, minimal confirmation; P0a gate measures burden directly |
| Recipe extraction is unreliable | Common draft-and-confirm pipeline; retain source material; review only consequential uncertainty; P0b gate measures correction burden |
| Planner feels restrictive or repetitive | Explain choices; swap/pin/manual control; balance reuse with variety |
| Taxonomy errors propagate into plans | Free-text fallback; conservative treatment of unmatched items; correction write-back |
| Catalog acquisition has legal/scraping constraints | Legal review before `retailer catalogs` starts; prefer formal feeds; compliant interim collection |
| Ingredient-to-SKU matching quality | Treat as core R&D; confidence scores, substitutions, fallback text, correction feedback |
| MVP becomes overbuilt | Phase gates in §4; P0 scope exclusions are explicit; platform generalisation waits for a buyer |
| AI/vision costs scale faster than value | Metering from P0 (D7); test willingness to pay before expensive usage scales |

## 11. Open questions

**Decided (July 2026):**

- **Mobile platform** — Dart/Flutter. Recorded as D8. Note: the iOS "Share → Saamly" target (P0b, `capture`) requires a native share extension bundled with the Flutter app; treat as a technical spike in the `capture` spec.
- **Auth** — OAuth (Google and Microsoft) plus magic link. Recorded in §3.1.
- **Admin tooling** — React web app for dashboards, taxonomy review and data analysis. Recorded as D9 / §3.16.
- **LLM/vision models** — Amazon Nova family on Bedrock, behind the `llm` port: Nova Lite (default vision), Nova Pro (automatic escalation on failed or low-confidence parses), Nova Micro (text-only parsing, P0b). A golden set of labelled kitchen photos gates any model, tier or prompt change. Verify Nova availability in `af-south-1` before P0a (cross-region fallback `eu-west-1`). Cost ceilings remain a `commerce`-spec task, built on D7 metering. Recorded in RFC §7.

**Open:**

1. **First retailer set and acquisition method** — which 2–3 retailers; formal approach vs compliant interim collection; legal review timing.
2. **Seed recipe sourcing** — who develops, tests and rights-clears the seed library, and how many recipes are "enough" for P1.
3. **Embedding/fuzzy-match infrastructure** — pg_trgm vs vector index for the `taxonomy` cascade; keep boring until scale demands otherwise.
4. **Multilingual support timing** — brand requires fluent review before any mixed-language copy; P1 is English-first.

## 12. Per-module spec index

Specs live in `docs/prd/modules/`. Written in dependency order; each spec follows Appendix A.

| # | Spec | Depends on | First phase | Status |
|---|---|---|---|---|
| 1 | `core.md` | — | P0a | Draft v0.1 |
| 2 | `capture.md` | `core` | P0a | Draft v0.1 |
| 3 | `taxonomy.md` | `core` | P0a | Draft v0.1 |
| 4 | `feedback.md` | `core` | P0a | Draft v0.1 |
| 5 | `admin.md` | `core`, `taxonomy`, `feedback` | P0a | Draft v0.1 |
| 6 | `inventory.md` | `capture`, `taxonomy`, `feedback` | P0a | Draft v0.1 |
| 7 | `recipes.md` | `capture`, `taxonomy`, `feedback` | P0b | Not started |
| 8 | `meal-plans.md` | `recipes`, `inventory`, `taxonomy` | P0c | Not started |
| 9 | `shopping.md` | `meal plans`, `inventory` | P0c | Not started |
| 10 | `household.md` | `core` | P0c (shared surfaces) | Not started |
| 11 | `sharing.md` | `core`, `recipes` | P0a (invites) / P1 | Not started |
| 12 | `cooking.md` | `meal plans`, `inventory` | P2 | Not started |
| 13 | `retailer-catalogs.md` | `taxonomy`, `shopping` | P1 | Not started |
| 14 | `commerce.md` | `core` (metering) | P2 | Not started |
| 15 | `community.md` | `sharing` | P3 (stage-gated) | Not started |
| 16 | `seeded-content.md` | `recipes` | P1 | Not started |

---

## Appendix A — Per-module spec template

1. **Purpose and pillar** — one paragraph; which pillar(s) it serves.
2. **Users and jobs** — who uses it and what they're trying to do.
3. **Scope: Now / Next / Later** — copied from the parent PRD, then detailed.
4. **Flows** — the key user journeys, including empty, error and uncertain states.
5. **Data owned** — entities, fields, provenance, retention.
6. **API boundary** — the headless capability surface (per D3); inputs, outputs, confidence semantics.
7. **Dependencies** — modules consumed; events emitted to `feedback`.
8. **Metrics** — how this module's contribution to §8 is measured.
9. **Risks** — module-specific risks and mitigations.
10. **Voice and labels** — copy constraints from brand §7; exact labels used.
