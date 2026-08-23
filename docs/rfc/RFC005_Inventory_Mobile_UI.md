# RFC 005 — Inventory Mobile UI

Status: Draft · August 2026
Relates to: `docs/prd/modules/inventory.md` v0.1 · `docs/prd/modules/capture.md` v0.1 · `docs/rfc/RFC001_Initial_Technical_Structure.md` (§4.2, §10) · `docs/rfc/RFC003_Capture_Module_Implementation.md` · `docs/rfc/RFC004_Inventory_Module_Implementation.md` · scaffold in `saamly-mobile` (auth + onboarding + home stub)

This RFC specifies *how* the inventory UI is implemented in `saamly-mobile`. The module spec owns product behaviour (the five confidence states, the apply loop, staleness, voice); this RFC owns the screen-by-screen flow, the review/confirm interaction model, the manual-add interaction, the offline-first store, the camera integration, state management, accessibility, empty/error states, delivery slices, and the decisions that must be locked before writing widgets. It deliberately does not restate the flows in `inventory.md` — it references them.

## 1. Goals and non-goals

**Goals**

- Ship the P0a inventory UI end-to-end: photo scan of fridge/freezer/cupboard/spice rack → review/confirm → "what you have" populated; manual add in three seconds; quick actions on existing items (used up, still there, use first, mine, remove); staleness prompts; offline-first with outbox sync.
- Make scan+confirm feel effortless, not tedious. The P0a gate is "scan+confirm faster than manual, both members agree" — the UI is where that gate is won or lost.
- Replace the home stub with the "what you have" surface: items grouped by location, scan FAB, add button, search, staleness prompts.
- Reuse the existing Riverpod + go_router + Dio scaffold verbatim; add drift for the on-device store and the camera plugin.
- Keep the mobile app free of service-side types. The UI talks to the consumer API (`/v1/inventory`, `/v1/capture/*`, `/v1/drafts/*`) and a local drift mirror; it never sees DynamoDB keys, Go structs, or Bedrock.
- Leave auth and onboarding green. The home stub becomes the inventory screen; the auth/onboarding flow is unchanged.

**Non-goals**

- The on-device drift schema and outbox engine internals — owned by a separate offline RFC (D8); this RFC defines the *contract* the store must satisfy (delta read, tombstones, idempotency), not the drift implementation.
- Taxonomy resolution UI beyond inline autocomplete on add/edit — the taxonomy review queue lives in the admin app, not here.
- "Use first" surfacing in meal plans, mark-cooked deduction, "used this" shortcuts (inventory.md §3 Next, P1).
- Receipt/barcode capture, retailer-order import, expiry estimation (inventory.md §3 Later, P2+).
- Push notification when a draft is ready (poll at P0; the inbox is the source of truth).
- The planner / shopping UI — those modules consume `Availability` server-side; their UIs land in their own RFCs.
- Multi-household switching (one household at P0; the active household comes from `core` middleware).
- Web/admin inventory management — this RFC is the consumer mobile app only.

## 2. Design principles (binding)

These principles are the difference between a scan flow that feels like magic and one that feels like a chore. They bind on every screen in this RFC.

1. **Smart defaults, not review-everything.** High-confidence detections are pre-confirmed at review time. The user only touches exceptions. A 30-item scan should require fewer than 5 taps to confirm. The default is "this is right"; the work is "fix the wrong".
2. **Group by location, not by confidence.** Items appear grouped the way they physically sit in the kitchen — Fridge, Freezer, Cupboard, Spice rack. This matches the mental model of "I'm looking at my fridge", not "I'm reviewing a list of detections". Confidence is a per-item attribute, not a section heading.
3. **Materiality is always visible, never a bare guess.** "Probably have" items get a yellow callout with the reason: *Spotted in Tuesday's scan — check the amount*. Never a confident-looking row that's actually uncertain. The user must be able to trust what the app says.
4. **Never force precision.** Quantities are `enough / low / unknown` + an optional free hint. One tap. No number entry, no unit picker, no "how many millilitres". The brand promise is "never fabricate exact quantities".
5. **Three seconds for manual add.** Type → autocomplete suggestions appear → tap one (or keep typing) → save. No form, no required fields, no confirmation modal. Default state Have, default location last-used or Fridge.
6. **Photo-first, not photo-forced.** Scan and manual add are offered with equal weight everywhere. A household that never scans still works (product test). Empty state shows both CTAs at equal size.
7. **Staleness is shown, not enforced.** Ageing items surface a gentle "Still there?" prompt. Nothing silently changes state. A refresh scan is the renewal mechanism; the prompt is a nudge, not a block.
8. **Offline is invisible.** Photos queue in the outbox when offline; items mutate the local store immediately; sync happens when connectivity returns. The UI never blocks on a spinner for a network call the user didn't initiate.
9. **Undo is one tap.** After every destructive or batch action, an in-app snackbar offers undo for 4 seconds. "Added 14 items" / "Removed milk" / "Marked 3 as out". This is the safety net that lets the user act fast.
10. **Honest about uncertainty.** "Probably" never looks like "Have". Hallucinations surface as "probably" with a reason, not as confident facts. The early-out metric (have→out within 3 days) catches leaks; the UI prevents them at the source by never dressing uncertainty as confidence.

## 3. Information architecture

```
saamly-mobile/lib/src/features/
  inventory/
    presentation/
      inventory_screen.dart       # "What you have" — home replacement
      item_card.dart              # one row in the list
      item_detail_sheet.dart      # edit an item
      add_item_sheet.dart          # manual add (3-second flow)
    application/
      inventory_controller.dart   # Riverpod: list, quick actions, optimistic updates
    data/
      inventory_repository.dart   # API + drift sync
      inventory_store.dart         # drift schema + DAO
      models.dart                  # Item, State, Location, QuantityHint
  scan/
    presentation/
      scan_entry_screen.dart      # choose target location(s)
      camera_screen.dart          # take 1-N photos
      review_screen.dart          # THE confirm step (heart of the RFC)
      review_item_row.dart        # one row in the review list
      scan_result_screen.dart     # "Added N items" + undo
    application/
      scan_controller.dart         # Riverpod: upload, poll, draft→review state
    data/
      scan_repository.dart        # /capture/uploads, /capture/text, /drafts
      draft_poller.dart            # poll draft status until needs_review
```

The home stub (`features/home/presentation/home_screen.dart`) is replaced by `inventory_screen.dart` at the `/home` route. The router keeps `/home` as the post-onboarding landing; it just builds a different widget.

## 4. Screen-by-screen spec

### 4.1 Home → "What you have" (replaces the stub)

The home screen *is* the inventory list. After onboarding, the user lands on their kitchen.

**Layout:**

- `AppBar`: "Saamly" title, sign-out icon (existing).
- Body: greeting + household line (existing, kept short), then the inventory list.
- FAB (bottom-right): camera icon, tooltip "Scan". Primary green. This is the photo-first nudge.
- Top-right action: `+` icon, tooltip "Add item". Opens the manual add sheet. Equal weight to scan, just less visually dominant.
- List: grouped by location, collapsible sections (Fridge / Freezer / Cupboard / Spice rack). Each section header shows count + a "Still there?" prompt if any item in it is stale. Section body: `ItemCard` rows.
- Search: a filter field in the app bar (optional, appears on scroll-up). Filters by name within the current view.
- Empty state: centered illustration + "Show Saamly what you have. A few quick photos are enough to get started." + two equal-weight buttons: "Scan" (filled) and "Add it myself" (outlined). No list, no FAB yet.

**ItemCard (one row):**

- Leading: a 44x44 tinted icon by state (Have = green check, Probably = amber dot, Use first = orange clock, Personal = person, Out = grey dash).
- Title: display name, bold. If "probably", a small amber "check the amount" chip below the name.
- Trailing: a "..." menu revealing quick actions (used up, still there, use first, mine, edit, remove). Tap a quick action → optimistic update + undo snackbar.
- Stale items: a thin amber bar on the left edge of the card + the "Still there?" prompt appears in the section header, not on every row (avoids visual noise).

**Staleness prompt:**

A section header gains a subtle amber "X items need a check — Scan to refresh" when any item in it is older than `SAAMLY_STALE_DAYS` (default 7, client-side). Tapping it opens the scan flow scoped to that location. The prompt is a nudge, not a block — items stay visible and usable.

### 4.2 Scan flow — entry

**Entry points:** FAB on home, "Scan" button in empty state, "Still there?" prompt in a section header.

**Scan entry screen:**

- Title: "What are we scanning?"
- Body: four large tappable tiles — Fridge, Freezer, Cupboard, Spice rack — each with an icon and a one-line note ("Take a photo of the shelves"). Multi-select is allowed (tap more than one); selected tiles show a check.
- Primary button: "Start camera" (disabled until at least one location is selected).
- Secondary: "Scan later" (cancel, pop).

The chosen locations become the `location` attribute on the created draft and the default grouping at review time. Scoping the scan to a location up front is what makes the review screen feel like "looking at that shelf" rather than "reviewing a flat list".

### 4.3 Scan flow — camera

**Camera screen:**

- Full-screen camera preview (native `camera` plugin, not a webview).
- Top: a thin progress strip of thumbnails (photos taken this session), horizontally scrollable. Tap a thumbnail to retake.
- Bottom: a large shutter button (centre), "Done" (right), flash toggle (left).
- After each capture: a brief "Added" haptic + the thumbnail appears. The user can keep shooting without leaving the camera.
- "Done" → upload starts in the background; navigates to a processing state on the review screen.

**Multi-photo session:** the user can take as many photos as they want; each becomes an `image_url` in the `POST /capture/uploads` body. The draft is created once at entry (with the chosen locations), and `complete` is called once after "Done".

**Permissions:** the first time the user taps "Scan", request camera + photos permissions. If denied, show a one-screen explanation with "Open settings" and an "Add manually instead" fallback. Never block.

### 4.4 Scan flow — review/confirm (the heart of this RFC)

This screen is where scan+confirm wins or loses against manual. The design goal: a 30-item scan requires fewer than 5 taps to confirm.

**Review screen layout:**

- `AppBar`: "Review" + a close button. No back button — closing discards the draft (with a confirm dialog).
- Body: a grouped list, one section per location chosen at entry. Each section is collapsible.
- **Within a section:**
  - High-confidence items (materiality `cosmetic`, confidence ≥ threshold) are pre-checked, shown in a collapsed "N items ready to add" header. One tap on the header expands it; the user can review if they want, but doesn't have to.
  - "Probably" items (materiality `material`, confidence < threshold) are expanded by default, each with an amber callout: *Spotted in this scan — check the amount*. The user must tap to confirm or reject each. This is the only required work.
  - Rejected-by-default items (the model flagged them) appear dimmed with a "Removed" chip; one tap to restore.
- Bottom bar (sticky): "Confirm N items" (primary, shows the count of items that will be added). Disabled if zero items are confirmed. Secondary: "Add all confident" (expands the collapsed section and checks everything) — a power-user shortcut.

**ReviewItemRow (one row):**

- Leading: a checkbox (pre-checked for high-confidence, unchecked for "probably", dimmed for rejected).
- Title: the detected name, editable inline (tap → text field → save/undo). Name edits re-resolve against taxonomy on confirm.
- Below title: a one-line quantity picker — three segmented chips: `Enough` / `Low` / `Unknown` (default from detection, or `Unknown`). No number entry.
- Trailing: a "..." menu — Edit name, Remove from scan, Mark as "probably".
- Amber callout (probably rows only): *Spotted in this scan — check the amount* (or the model's reason if available). Tapping the callout expands it to show the photo region if a bbox was returned.

**Confirm action:**

1. User taps "Confirm N items".
2. Optimistic: the items are added to the local store immediately, the screen navigates to the scan result.
3. The edited payload is sent to `POST /drafts/{id}/confirm` with the accepted/rejected items.
4. On 409 (material unresolved — shouldn't happen, the UI prevents it), show the error and return to review.
5. On network failure, the confirm is queued in the outbox; the UI still navigates (offline-first).

**Scan result screen:**

- "Added N items to your [location]" with a checkmark illustration.
- Below: the new items grouped by location, tappable to edit.
- Snackbar: "Undo" for 4 seconds (reverses the local add + sends a delete via outbox).
- Primary: "Done" → back to home, which now shows the new items.
- Secondary: "Scan another" → back to scan entry (defaults to the same locations).

### 4.5 Manual add flow

**Entry points:** "+" action in the app bar, "Add it myself" in the empty state, "Add manually" in any scan error state.

**Add item sheet (bottom sheet, not a full screen):**

- Title: "Add an item"
- Body: a single `TextField` with inline autocomplete. As the user types, taxonomy suggestions appear below the field (canonical name + "did you mean...?"). Tap a suggestion to fill the field + set the concept id. Free text always works — the user can ignore suggestions and type anything.
- Below the field: a row of three segmented chips for state — `Have` (default) / `Probably` / `Out`. One tap.
- Below state: a row of four location chips — `Fridge` (default, or last-used) / `Freezer` / `Cupboard` / `Spice rack`. One tap.
- Below location: a row of three quantity chips — `Enough` (default) / `Low` / `Unknown`. One tap. Optional: a small "hint" link reveals a one-line text field for a free hint ("half a bag").
- Primary: "Add" (enabled once a name is non-empty). Tap → optimistic add to local store + outbox + close sheet + undo snackbar on the list.
- Secondary: "Cancel".

**Three-second target:** type "milk" → tap the "Milk" suggestion → tap "Add". Three taps, no scrolling, no required decisions beyond the name. State defaults to Have, location to last-used, quantity to Enough. The user only deviates from defaults when they want to.

**Why a sheet, not a screen:** a sheet keeps the user in the context of the list — they see the new item appear where it will live. A full screen feels like a form; a sheet feels like a quick note. This is the difference between "three seconds" and "a form".

### 4.6 Item detail / edit

**Entry:** tap an `ItemCard` (not the "..." menu — that's for quick actions).

**Item detail sheet (bottom sheet):**

- Title: display name (editable).
- State: segmented chips (Have / Probably / Use first / Personal / Out).
- Location: chips (Fridge / Freezer / Cupboard / Spice rack).
- Quantity: chips (Enough / Low / Unknown) + free hint field.
- Owner: shown only if household has >1 member; "This is mine" toggle (sets Personal + owner = actor).
- Notes: multi-line text field. Existing notes appended, not overwritten (LWW per field-group; notes are their own group).
- Provenance: read-only line — "Added by scan on Tuesday" / "Added manually by David". Builds trust.
- Last confirmed: read-only relative time ("confirmed 2 days ago").
- Primary: "Save" (optimistic + undo). Secondary: "Remove" (with confirm dialog).

**Quick actions** (from the card "..." menu, not the detail sheet) hit the same controller methods as the detail sheet's save — they're shortcuts, not separate code paths.

## 5. State management

Riverpod providers, mirroring the existing `authControllerProvider` pattern:

- `inventoryControllerProvider` — `AsyncNotifier<InventoryState>` holding the grouped list, last-sync cursor, and pending optimistic mutations. Methods: `refresh`, `addItem`, `updateItem`, `transitionState`, `removeItem`, `undoLast`. Optimistic updates mutate the local drift store first, then enqueue the API call in the outbox.
- `scanControllerProvider` — `Notifier<ScanSession>` holding the chosen locations, taken photos, current draft id, draft status, and the parsed review model (candidates by location). Methods: `startSession`, `addPhoto`, `retakePhoto`, `completeUpload`, `pollDraft`, `confirm`, `reject`, `editCandidate`.
- `draftInboxProvider` — `StreamProvider<List<DraftSummary>>` polling `/drafts?status=needs_review` every 15s while the app is foregrounded. Surfaces drafts from background scans (e.g. text ingest from the seed script). The home app bar shows a badge when the inbox is non-empty.
- `taxonomySuggestProvider` — `FutureProvider.family<String, List<Suggestion>>` debounced by 200ms, used by the add/edit autocomplete.

**Why Riverpod over Bloc/Provider:** the existing scaffold is Riverpod; consistency is cheaper than a second pattern. `AsyncNotifier` handles loading/error/data cleanly for the list; `Notifier` is fine for the scan session (synchronous state, async side effects via the repository).

**No global app state store.** Each feature owns its providers. Cross-feature reads (e.g. the planner reading availability) happen server-side via the `Availability` port, not via shared mobile state.

## 6. Offline-first contract

The mobile app is offline-first for inventory reads and mutations (D8). This RFC defines the contract the on-device store must satisfy; the drift implementation is a separate offline RFC.

**Contract:**

- **Local store** (drift) mirrors inventory items keyed by `item_id`. Reads always hit the store; the UI never shows a loading spinner for the list after the first sync.
- **Delta sync:** on app foreground and every 60s while open, `GET /inventory?updated_since=<cursor>` returns changed items + tombstones. The store applies them, advances the cursor, and the controller emits. Tombstones hard-delete local rows.
- **Optimistic mutations:** every mutation (add/update/transition/remove) writes to the store immediately, generates an idempotency key, and enqueues an outbox entry. The UI updates instantly.
- **Outbox replay:** a background worker drains the outbox when connectivity returns. Each entry hits the same endpoint with the same idempotency key; the server dedupes. On 409 (LWW conflict), the server's version wins for the conflicting field-group; the local store is corrected and the controller re-emits.
- **Photos offline:** photos taken while offline are stored locally and uploaded when connectivity returns. The draft is created lazily on the first upload (or eagerly with a placeholder if the user completes the session offline — the draft moves to `needs_review` once the first image lands).
- **No partial-failure surprises:** if a mutation is in the outbox and the user undoes it, the undo is also enqueued (a delete for an add, a revert for an update). The outbox is an ordered log, not a set of pending requests.

**What the UI never does:**

- Block on a network call the user didn't initiate. The only spinners are: camera shutter processing (local), upload progress (user-initiated), draft polling (background).
- Show an error toast for a background sync failure. Sync failures retry silently; the user sees their data either way.
- Allow the local store and the server to diverge silently. The cursor + tombstone contract guarantees eventual convergence.

## 7. Camera implementation

- Plugin: `camera` (the official Flutter package). Avoid `image_picker` for the scan flow — we want a live preview and multi-shot sessions; `image_picker` is one-shot.
- Resolution: `ResolutionPreset.medium` — good enough for the LLM, small enough to upload on a slow connection. The service caps at 10MB per image.
- Orientation: lock to portrait for the scan flow (the model is trained on shelf photos, which are naturally portrait). Landscape is fine for manual single shots but adds layout complexity for no gain.
- Compression: JPEG quality 0.7, max dimension 1600px, before upload. Done client-side to keep uploads cheap.
- EXIF: stripped before upload (privacy; the service doesn't need GPS from a fridge photo).
- Multiple photos: kept in memory as `XFile` paths; uploaded as multipart in `POST /capture/uploads`. The draft id is created first (empty), then images appended, then `complete` called.
- Background upload: use `dio`'s `MultipartFile` with a progress callback; the review screen shows "Uploading 2 of 5..." until done, then "Processing..." while polling the draft.
- Permission denial: a dedicated screen with "Open settings" + "Add manually instead". Never a bare permission dialog with no fallback.

## 8. Accessibility

- Tap targets: minimum 48x48dp (Material default), enforced on the segmented chips and the "..." menu items — these are the highest-frequency taps.
- State colours are never the only signal. Each state has an icon (check / dot / clock / person / dash) and a text label. Colour-blind users see the same information.
- "Probably" callouts use the amber accent, but the callout text is the real signal — "check the amount" reads the same in any colour.
- VoiceOver/TalkBack: every `ItemCard` announces "Milk, have, confirmed 2 days ago". Every `ReviewItemRow` announces "Milk, probably, check the amount, checkbox unchecked". The amber callout is read as part of the row.
- Haptics: light haptic on shutter, medium on confirm, success haptic on scan result. Subtle, not celebratory — this is a kitchen tool, not a game.
- Font scaling: the layout works at 1.5x system font size. Segmented chips wrap to two lines if needed; the bottom confirm bar grows.
- Motion: the review screen's section collapse/expand is a 200ms ease. No bouncing, no parallax — kitchen users want fast, not playful.

## 9. Empty, error, and edge states

**Empty inventory (first run):**
- Centered illustration (a simple kitchen shelf line drawing).
- Headline: "Show Saamly what you have."
- Subhead: "A few quick photos are enough to get started."
- Two equal-weight buttons: "Scan" (filled, primary) and "Add it myself" (outlined). Equal size, side by side. This is the photo-first-but-not-photo-forced promise made visible.
- No FAB, no list — the empty state is its own screen, not a list with a banner.

**Empty location section:**
- A section header with count 0 is hidden. We don't show "Fridge (0)" — that's noise. Sections appear when they have items.

**Scan — no detections:**
- "We couldn't find anything in these photos."
- Subhead: "Try again with better lighting, or add items manually."
- Buttons: "Try again" (back to camera) and "Add manually" (open the add sheet).

**Scan — upload failed:**
- "We couldn't upload these photos."
- Subhead: "Check your connection and try again."
- Buttons: "Retry" and "Add manually". The taken photos are kept for retry.

**Scan — processing timeout (draft stuck in `processing` > 60s):**
- "Taking a little longer than usual."
- Subhead: "We'll add the items when they're ready — you can keep using the app."
- Button: "Back to kitchen". The draft stays in the inbox; the user is notified (badge) when it moves to `needs_review`.

**Manual add — taxonomy miss:**
- No suggestions appear; the field stays as free text. The "Add" button is enabled. No error state — a miss is not a failure (D4: free text always works).

**Conflict on save (LWW):**
- Silent. The server's version wins for the conflicting field-group; the local store is corrected; the controller re-emits. The user sees the corrected value on next render. No toast — conflicts are rare at P0 and a toast would erode trust more than the conflict itself.

**Offline (no connectivity):**
- A thin banner at the top of the home screen: "Offline — changes will sync when you're back." Dismissible. Mutations still work; the outbox holds them.
- The scan FAB is disabled with a tooltip "Scanning needs a connection to recognise items." Manual add stays enabled (it queues locally). This is the one place offline is visible — because scanning offline genuinely can't work (the LLM is server-side).

**Household purged (404 on sync):**
- The local store is cleared; the user is routed to onboarding with a message "Your household was reset — let's set up again." Rare and handled by `core`.

## 10. Voice and labels

Inherits the module spec (`inventory.md` §10) verbatim. UI-specific additions:

- The home screen title is "Saamly" (brand), not "Inventory" or "What you have". The list itself is the surface; it doesn't need a heading. The greeting ("Hi, David") carries the personal tone.
- Section headers use household words: *Fridge* · *Freezer* · *Cupboard* · *Spice rack*. Never "Refrigerator" or "Pantry".
- States are labelled exactly: *Have* · *Probably have* · *Use first* · *Personal* · *Out*. The "Probably have" chip is two words on the card, abbreviated "Probably" in segmented pickers where space is tight — but the full label is always available to VoiceOver.
- Quick actions: *Used up* · *Still there?* · *Use first* · *Mine* · *Already have* · *Bought* (list-side, future). The "?" on "Still there?" is intentional — it's a question, not a command.
- The scan entry prompt: "What are we scanning?" — conversational, not "Select location".
- The review confirm button: "Confirm N items" — concrete, not "Submit" or "Done".
- The scan result: "Added N items to your [location]" — specific, not "Success".
- Uncertainty is named with the reason: *Spotted in Tuesday's scan — check the amount*. Never a bare "Probably" without a why.
- Corrections: *Not in the fridge anymore? We'll add it back to the list.* (brand §13) — shown when the user marks a "have" item as "out".
- Empty state: *Show Saamly what you have. A few quick photos are enough to get started.* — verbatim from the module spec.

## 11. API surface used

From the vendored `consumer.yaml` (RFC 003 + RFC 004). The mobile app calls only these:

**Inventory:**
- `GET /inventory?updated_since={cursor}&limit={n}` — delta read for sync.
- `POST /inventory/items` — manual add (body: `name_text`, `concept_id?`, `state`, `location?`, `quantity_hint?`, `quantity_free?`, `idempotency-key` header).
- `PUT /inventory/items/{id}` — edit (state/location/quantity/personal/display name).
- `POST /inventory/items/{id}/state` — quick transition (body: `trigger` ∈ `used_up` · `still_there` · `use_first` · `personal` · `shared`).
- `DELETE /inventory/items/{id}` — remove.
- `GET /inventory/items/{id}` — detail (rare; the list has everything).

**Capture:**
- `POST /capture/uploads` — create a draft with chosen locations + first image.
- `POST /capture/uploads/{draft_id}/images` — append more images (multi-photo session).
- `POST /capture/uploads/{draft_id}/complete` — mark upload done, trigger processing.
- `GET /drafts?status=needs_review` — inbox poll.
- `GET /drafts/{id}` — fetch parsed candidates for review.
- `POST /drafts/{id}/confirm` — submit the edited payload (accepted/rejected items).
- `POST /drafts/{id}/reject` — discard the whole draft (close without confirm).

**Taxonomy (read-only at P0):**
- `GET /taxonomy/concepts?q={prefix}` — autocomplete suggestions for add/edit.

The mobile app does **not** call the internal `ApplyScanDraft` or `InventoryAvailability` ports — those are service-side. The confirm endpoint routes to `ApplyScanDraft` internally (RFC 003 §6).

## 12. Delivery slices

Five slices, each shippable behind the existing home stub. The stub stays until slice D replaces it.

### Slice A — Inventory models, repository, controller (no UI)

- `features/inventory/data/models.dart` — `Item`, `State`, `Location`, `QuantityHint`, `Note`, `Provenance` (mirror of the API schema).
- `features/inventory/data/inventory_repository.dart` — Dio calls for the six inventory endpoints; parses problem+json via the existing `ApiClient`.
- `features/inventory/application/inventory_controller.dart` — `AsyncNotifier` with `refresh`, `addItem`, `updateItem`, `transitionState`, `removeItem`, `undoLast`. No drift yet — in-memory cache, optimistic mutations, direct API calls (no outbox). Good enough for online dogfood.
- Unit tests for the controller with a fake repository.

**Exit:** the controller can list/add/edit/remove items against a running service, in tests. Nothing on screen yet.

### Slice B — "What you have" home screen (replaces the stub)

- `features/inventory/presentation/inventory_screen.dart` — grouped list by location, collapsible sections, `ItemCard` rows, scan FAB, "+" action, search field, empty state.
- `features/inventory/presentation/item_card.dart` — state icon, name, "check the amount" chip, "..." quick-action menu, stale amber bar.
- `features/inventory/presentation/add_item_sheet.dart` — manual add sheet with autocomplete (calls `taxonomySuggestProvider`), state/location/quantity chips, "Add" button.
- `features/inventory/presentation/item_detail_sheet.dart` — edit sheet.
- Wire `inventory_screen.dart` into the router at `/home` (replaces `HomeScreen`).
- Widget tests: empty state, list rendering, add sheet, quick action, undo snackbar.

**Exit:** a user can manually populate and manage their inventory end-to-end, online. The scan FAB is a no-op stub that toasts "Scanning lands next".

### Slice C — Scan flow: entry + camera + upload

- `features/scan/presentation/scan_entry_screen.dart` — location tiles, "Start camera".
- `features/scan/presentation/camera_screen.dart` — `camera` plugin preview, thumbnail strip, shutter, "Done".
- `features/scan/data/scan_repository.dart` — `createUpload`, `appendImage`, `completeUpload`, `getDraft`, `listDrafts`, `confirmDraft`, `rejectDraft`.
- `features/scan/application/scan_controller.dart` — session state, photo list, upload progress, draft id.
- Camera permissions flow + denial fallback screen.
- Widget/integration tests with a fake repository and a mocked camera.

**Exit:** a user can take photos and create a draft; the draft appears in the inbox (verifiable via `GET /drafts`). No review UI yet — the draft just sits in `needs_review`.

### Slice D — Review/confirm + scan result (the heart)

- `features/scan/presentation/review_screen.dart` — grouped list, collapsed "ready to add" sections, expanded "probably" sections, sticky confirm bar, "Add all confident".
- `features/scan/presentation/review_item_row.dart` — checkbox, inline-editable name, quantity chips, "..." menu, amber callout.
- `features/scan/presentation/scan_result_screen.dart` — "Added N items", undo snackbar, "Done"/"Scan another".
- `features/scan/data/draft_poller.dart` — poll `GET /drafts/{id}` until status is `needs_review` (or timeout → processing state).
- `draftInboxProvider` — 15s poll, app-bar badge on home.
- Widget tests: pre-checked high-confidence, expanded probably, confirm count, reject, undo, timeout state.

**Exit:** the full scan+confirm loop works online. This is the P0a gate measurable end-to-end.

### Slice E — Offline-first: drift store + outbox

- `features/inventory/data/inventory_store.dart` — drift schema mirroring the API items table, with `updated_at` cursor and tombstone handling.
- `features/inventory/data/outbox.dart` — ordered log of pending mutations with idempotency keys; background drain worker.
- Wire the controller to read from the store, write optimistically, enqueue outbox entries.
- Photo outbox: offline photos stored locally, uploaded on reconnect.
- Offline banner on home; scan FAB disabled offline with tooltip.
- Conflict handling: on 409, server version wins for the field-group, local store corrected.
- Integration tests: offline add → reconnect → sync; concurrent edits → LWW; tombstone → local delete.

**Exit:** the app is offline-first. The P0a gate can be measured in a real kitchen with a flaky connection.

## 13. Risks

| Risk | Mitigation |
|---|---|
| The review screen still feels tedious despite smart defaults | Slice D ships behind a stopwatch; the P0a gate measures scan+confirm vs manual with two members. If it's not faster, the defaults aren't aggressive enough — lower the confidence threshold for pre-confirm. |
| Camera plugin flakiness on the Android emulator | Slice C tests on a physical device before merge; the emulator camera is a known weak spot. The manual add path is always available as a fallback. |
| Drift schema drift from the API | The store is generated from the vendored `consumer.yaml` schema where possible; a unit test asserts the model round-trips against a fixture. |
| Outbox ordering bugs corrupt state | The outbox is an ordered log; the test suite covers add-then-undo, undo-then-add, and concurrent mutations on the same item. |
| Optimistic UI shows data the server rejects | LWW conflict handling corrects the local store silently; the test suite covers the 409 path. The UI never lies for long. |
| Staleness prompts nag the user into ignoring them | The prompt is per-section, not per-item; the threshold is configurable; a prompt that's repeatedly dismissed could be suppressed (P1 learning, not P0). |
| "Probably" callouts are missed because they look like "Have" | Distinct colour + icon + text label + expanded-by-default. Accessibility §8 ensures the signal isn't colour-only. |
| Multi-photo sessions upload slowly on a bad connection | JPEG quality 0.7, max 1600px, background upload with progress. The user can navigate away; the upload continues. |
| The home screen becomes cluttered with many items | Search field, collapsible sections, and (P1) a "use first" pinned section. P0 caps at ~100 items gracefully; real households are smaller. |

## 14. Open questions

1. **Confidence threshold for pre-confirm.** The review screen pre-checks items above a confidence threshold. The threshold is a server-side constant (`SAAMLY_CAPTURE_CONFIDENCE_THRESHOLD`) but the UI must react to it. Decision needed: does the API return a per-candidate `materiality` flag (current RFC 003 design) and the UI pre-checks `cosmetic` ones, or does the API return a numeric confidence and the UI applies a client-side threshold? **Recommendation:** keep the `materiality` flag from RFC 003 — the server owns the threshold; the UI is dumb. Lock before slice D.

2. **Draft polling vs push.** At P0 we poll `/drafts/{id}` every 2s until `needs_review`. Push (SSE/WebSocket) would be nicer but adds service surface. Decision needed: is polling acceptable for the P0a gate, or do we invest in push now? **Recommendation:** poll at P0; revisit if the gate shows the delay hurts the experience. Lock before slice D.

3. **Last-used location persistence.** The manual add sheet defaults location to "last-used". Is "last-used" per-user (shared across the household) or per-session? **Recommendation:** per-user, stored in `core` preferences. Lock before slice B.

4. **Undo window.** 4 seconds is proposed. Too short and the user misses it; too long and it blocks the next action. **Recommendation:** 4s with a visible snackbar; test in dogfood. Lock at slice B.

5. **Taxonomy autocomplete endpoint.** This RFC assumes `GET /taxonomy/concepts?q=` exists. If it doesn't yet, slice B's autocomplete needs a stub or the taxonomy RFC needs to land first. **Recommendation:** check the service; if missing, ship slice B without autocomplete (free text only, still meets the 3-second target) and add autocomplete when the endpoint lands. Lock before slice B.

## 15. Out of scope (explicit)

- Quantities beyond `enough / low / unknown` + free hint.
- Expiry dates, expiry estimation, "use by" surfacing.
- Barcode/receipt scanning.
- Multi-household switching in the UI.
- The planner and shopping UIs.
- Push notifications.
- The taxonomy admin review queue.
- Continuous reconciliation, "used this" shortcuts, mark-cooked deduction (P1).
- Web/admin inventory management.
- The drift schema design itself (owned by the offline RFC).
- Server-side changes to capture or inventory (this RFC consumes the existing API; if the API needs changes, that's a separate RFC).
