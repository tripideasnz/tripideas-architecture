# Stage 1 Notebook RFC

## Status

Approved

Approved: 2026-07-25

## Summary

Stage 1 adds private, cloud-synchronised travel notebooks to the TripIdeas mobile app. A signed-in traveller can create multiple notebooks, add ordered text and processed photos, caption photos, optionally link an item to an existing editorial place, and create or revoke a read-only capability link that anyone holding it can view without an account.

This RFC deliberately stops at the Phase 1 product boundary. It does not add user-created places, Trip Idea integration, collaboration, public profiles or feeds, publishing products, licensing, itinerary features, behavioural personalisation, location tracking, or AI assistance.

The preferred authenticated API host is `tripideas-api`, using its existing user-data Postgres where the API audit confirms ownership and operational suitability. Exact retention periods remain unresolved pending that audit. Object-storage provider selection remains deferred until the photo milestone.

No user notebook content will be stored in Sanity.

## Source-of-truth and repository boundaries

- `tripideas-architecture` owns this RFC and subsequent cross-repository decisions.
- `tripideas-mobile` owns the authenticated notebook experience, local cache, offline queue, photo selection, upload state, and native share sheet.
- `tripideas-api` is the preferred owner of authenticated Notebook endpoints and user-data database access, subject to the focused audit required before implementation.
- `tripideas-web` owns the public shared-notebook page.
- `tripideas-cms` remains a read-only source of editorial place identifiers and presentation projections.
- The existing WorkOS-authenticated API at `api.tripideas.nz` is currently consumed by mobile and web, but its implementation source is not present in the four repositories.

## Audit findings

### Authentication and stable user identity

- Mobile uses WorkOS OAuth with PKCE through `/auth/mobile/authorize`, `/auth/mobile/exchange`, and `/auth/mobile/refresh`.
- The mobile access token is held in memory and sent as `Authorization: Bearer <token>`. Refresh tokens and the cached user are stored in Expo SecureStore.
- Both mobile and web consume `/auth/identity`, whose response contains `id`, `email`, and `name`.
- Mobile treats the returned WorkOS user `id` as `AuthUser.id` and `session.userId`. This is the stable owner identifier available to notebooks.
- Server ownership must always be derived from the verified authenticated identity. An `ownerUserId` supplied in a request body must be ignored or rejected.
- The WorkOS token-verification and account-deletion implementations are not in the audited repositories. Mobile sign-out is currently local-only because a mobile logout endpoint is not implemented.

### Mobile API conventions

- `lib/api-client.ts` centralises the API base URL, bearer token, JSON headers, cookies, non-success handling, and empty response handling.
- Feature-specific modules, such as `saved/api.ts`, expose typed functions over `apiFetch`.
- Existing errors are coarse `Error` instances keyed by status and path. Notebook conflict, validation, quota, and upload errors will require a structured error envelope while preserving the shared client entry point.
- The notebook API should use the same authenticated client rather than adding a second authentication stack.

### Existing Trip Ideas storage and sharing

- The mobile repository contains local Trip Ideas stored as JSON in AsyncStorage, using client-generated IDs, ordered place arrays, and immediate local updates.
- The broader current Trip Idea sharing system supports a public read-only recipient view without login. Actions beyond basic viewing may prompt the recipient to create or use a TripIdeas account.
- After login, a recipient can access the Trip Idea through a shared Trip Ideas area. Authenticated participants may add messages and make changes.
- Shared Trip Idea and collaboration data are currently backed by Sanity.
- This is a collaboration-oriented lifecycle, not merely an unauthenticated static page. It should remain unchanged during Notebook Phase 1.
- It should not be copied as the Notebook storage or permission model because it is designed around Trip Idea collaboration, ties user-created collaboration data to Sanity, and does not provide the preferred ownership, quota, deletion, or reusable photo-asset lifecycle for frequently edited personal notebooks.
- Notebook Phase 1 instead requires owner-controlled, revocable read-only capability links and deliberately excludes authenticated recipient collaboration.
- Reusable presentation patterns include the public-page layout, recipient journey and login prompts where appropriate, native mobile share sheet, place-card rendering, responsive read-only presentation, and editorial place links.

### Postgres and migration conventions

- `tripideas-web` has the `postgres` package and a `DATABASE_URL`-backed singleton for the Nearby Places access graph.
- The only visible SQL migrations are numbered raw SQL files for that access graph. They are script-oriented and not evidence of the production user-data API migration process.
- Web itinerary models use client-side SignalDB/IndexedDB and synchronise to the external authenticated API. They do not define the server database schema.
- The generated API contract refers to user-owned itineraries, favourites, collections, and account deletion, confirming that an operational user-data backend exists elsewhere.
- The production migration runner, transaction conventions, backup policy, and user-data database ownership cannot be verified without the missing API source.

### Object storage and image upload

- No object-storage client, signed-upload flow, upload endpoint, image-processing dependency, or asset cleanup worker exists in the audited repositories.
- Existing images are editorial Sanity assets or remote URLs. The browser `Blob` usage is only a local JSON download.
- Stage 1 therefore requires a new provider-backed storage adapter and processing path. Provider selection is intentionally unresolved in this draft.

### Mobile photo dependencies

- `expo-image` is installed for display.
- No direct photo-picker, media-library, image-manipulation, or durable file-system dependency is installed.
- `expo-file-system` appears only transitively and must not be imported without becoming an explicit dependency.
- The implementation will need an SDK-54-compatible picker and app-managed local file storage for pending uploads. A selected photo must be copied away from the temporary device-library URI immediately.

### Offline and local persistence

- Mobile uses AsyncStorage for Trip Ideas and per-user favourites. Favourites are scoped by stable user ID and preserve a durable per-user cache across sign-out.
- SecureStore is correctly limited to authentication secrets and cached identity.
- Favourites implement optimistic local changes and best-effort reconciliation, but only additive reconciliation is queued; there is no general mutation log or conflict protocol.
- Web has SignalDB with IndexedDB persistence and a debounced sync manager for itineraries. It is useful precedent for local-first interaction, but it is browser-specific and not shared with React Native.
- Stage 1 should reuse the mobile per-user AsyncStorage convention for notebook snapshots and a durable mutation queue, while storing pending photo files in app-managed file storage.

### Place identifiers and projections

- Editorial places are Sanity `page` documents. Their `_id` is already used as `placeId` in favourites and Trip Ideas.
- Mobile has debounced place search and a shared `PlaceCardData` projection containing `_id`, title, subtitle, image, summary, coordinates, and slug.
- Notebook items should persist only the stable editorial `placeId`. The server/public presentation layer should resolve an allowlisted summary from published Sanity data and tolerate a missing or unpublished place.
- User notebook data must not modify editorial place documents.

### Web routes and API conventions

- The web application uses Next.js 15 App Router.
- Local server routes live under `src/app/api/**/route.ts`, validate input, return `NextResponse.json`, and keep secrets server-side.
- Public shared pages use route groups, dynamic parameters, `force-dynamic`, and no-store reads.
- The typed web API client talks to `SERVER_API_URL` or `NEXT_PUBLIC_API_URL`; most authenticated APIs are external rather than Next.js route handlers.
- The shared notebook route should follow the current dynamic/no-store pattern and add explicit search-engine exclusion.

### Error reporting, analytics, and logging

- Both apps use `console` logging with feature prefixes. Web supports limited configurable fetch/performance logging.
- No Sentry, product analytics, structured telemetry service, or privacy-aware event pipeline is installed.
- Stage 1 should introduce structured server logs with request IDs and non-sensitive event names. Photo bytes, text content, captions, tokens, emails, storage keys, and bearer credentials must never be logged.
- Product analytics is optional for initial rollout; operational counts and failures are required before enabling photos broadly.

### Deletion and account data

- Web exposes `DELETE /auth/delete` through the external API and warns that account data is permanently removed, but its cascade and retention behaviour cannot be audited.
- Mobile clears authentication secrets at sign-out but intentionally retains per-user favourites as a cache. Trip Ideas are not user-scoped.
- Existing Trip Idea sharing has different collaboration, access, and Sanity-backed lifecycle semantics. Its precise revocation and deletion guarantees are not defined in the audited repositories.
- Notebook deletion must revoke sharing immediately, hide the notebook from normal reads, clear or invalidate mobile cache, and schedule unreferenced photo assets for delayed deletion.
- Account deletion must include notebooks, shares, items, and assets. Backup retention and completion guarantees require confirmation from the backend owner.

### Genuine blockers and decisions

The API-host preference, database preference, and schema-level deletion architecture are approved. Milestone 1 remains blocked only until the `tripideas-api` audit confirms repository ownership, WorkOS verification, user-data Postgres provider and environments, migration conventions, backup/restore arrangements, account-deletion behaviour, and retention constraints. Object storage and processing remain deliberately deferred, and mobile sign-out cache handling remains open before Milestone 2.

## Scope

### Included

- Multiple notebooks per signed-in traveller.
- Required title and optional description.
- Ordered text and photo items.
- Optional photo captions.
- Optional link from an item to an existing editorial place.
- Create, read, update, reorder, and delete.
- Cross-device cloud synchronisation.
- Practical offline viewing and editing.
- Private-by-default ownership.
- Explicit read-only link creation and revocation.
- Unauthenticated responsive shared web view.
- Storage, request, upload, and abuse safeguards.

### Excluded

- User-created place cards or editorial mutations.
- Adding user-created content to Trip Ideas or itinerary functionality.
- Collaborative editing, comments, reactions, followers, profiles, or feeds.
- Public discovery or searchable notebooks.
- Licensing, editorial submission, or reuse by TripIdeas.
- Blogs, books, PDFs, social-post generation, or commercial publishing.
- AI assistance of any kind.
- Behavioural recommendations, personalisation, or location tracking.
- Full revision history and real-time multi-user conflict resolution.

## Phase 1 completion boundary

Phase 1 is complete when signed-in users can:

- create and manage multiple notebooks;
- add and edit flexible text content;
- add processed photos with optional captions;
- optionally link content to existing TripIdeas places;
- reorder and delete notebook items;
- access notebooks across devices;
- use practical offline caching and queued synchronisation;
- keep notebooks private by default;
- create, copy, and share a read-only public capability link;
- revoke that link; and
- allow friends to view the current notebook without an account.

Phase 1 does not include recipient collaboration, comments or messages, user-created place cards, adding user-created content to Trip Ideas, licensing or offering content to TripIdeas, editorial review, public feeds, publishing products, AI writing or photo assistance, itinerary functionality, or behavioural personalisation.

## Later-phase context

The next phase may create user-authored personal place cards from notebook content, add those cards to a Trip Idea for private use, reuse notebook content in future itinerary functionality, and let users offer selected photos, notes, and structured place information to TripIdeas through an explicit licence workflow.

Subsequent phases may add broader publishing outputs, AI-assisted writing, captions and organisation, partner or commercial publication services, and the other capabilities described in the Personal Content Platform vision.

These are context only. Stage 1 must preserve clean ownership and reuse boundaries without designing speculative workflows for them. Future capabilities should enrich notebooks rather than replace them.

## User journeys

### Create and edit

1. A signed-out traveller opening Notebooks is asked to sign in.
2. A signed-in traveller sees locally cached notebooks immediately while a refresh runs.
3. They create a notebook with a title and optional description.
4. They add text or choose photos, edit content and captions, reorder items, and optionally link an item to an editorial place.
5. Changes appear locally first. Clear state indicates pending, uploading, synced, failed, or conflicted work.

### Offline use

1. Previously loaded notebooks remain readable without a connection.
2. Text and metadata edits are appended to a per-user durable mutation queue.
3. A selected photo is copied into app-managed local storage and appears immediately as pending.
4. Queued changes retry when connectivity and a confirmed session return.
5. The UI never labels a photo as synced until the processed cloud asset is complete.
6. Destructive actions warn when they would discard unsynced-only content.

### Link a place

1. The traveller chooses “Link a place” on a text or photo item.
2. Existing published TripIdeas places are searched using the established editorial projection.
3. The selected Sanity document `_id` is stored as `linkedPlaceId`.
4. If the editorial place later disappears, notebook content remains and the missing place link is omitted.

### Share and revoke

1. The owner opens notebook share controls and explicitly creates a link.
2. The server returns the only copy of a new raw share token and the public URL.
3. Mobile can copy the URL or open the native share sheet.
4. A recipient holding this read-only capability link opens a responsive page without an app or account.
5. The page renders the live current notebook: owner edits appear, and removed content disappears.
6. The recipient cannot edit, comment, message, or collaborate.
7. The owner revokes the link. The same URL stops working immediately or within the documented minimal cache window.
8. Creating another link generates a new token; revoked tokens are never reactivated.

### Delete

1. Item deletion removes the notebook relationship after confirmation where unsynced data is at risk.
2. Notebook deletion requires confirmation, revokes shares in the same transaction, and removes it from active owner reads.
3. Unreferenced assets enter a delayed-deletion grace period.
4. Account deletion invokes the established account-data workflow once its cascade contract is confirmed.

## Screen hierarchy

The existing five-tab structure should remain. Notebooks belong in the authenticated personal-content area rather than creating another top-level tab.

```text
Saved
├── Notebooks
│   ├── Notebook list
│   ├── Create notebook
│   └── Notebook detail/editor
│       ├── Edit title and description
│       ├── Add/edit text
│       ├── Add photo
│       ├── Edit caption
│       ├── Link/unlink editorial place
│       ├── Reorder/delete items
│       └── Share controls
└── Existing My Trips and Favourites

Public web
└── /notebook/[shareToken]
```

The Notebook list shows title, first available thumbnail, updated date, useful item/photo count, and private/shared state. The editor should feel like a document with insert actions, not a schema form.

## Data ownership and systems of record

- Postgres is the system of record for notebook metadata, items, ordering, ownership, asset metadata, synchronisation versions, and shares.
- Object storage is the system of record for processed notebook images and thumbnails.
- The mobile cache is a user-scoped working copy and offline queue, never the cross-device authority.
- Sanity remains the system of record for editorial places only.
- The public page reads an allowlisted server projection; it never reads a client-supplied public snapshot.
- The traveller owns their content. Phase 1 grants TripIdeas only the operational permission needed to store, process, synchronise, and explicitly share it.
- Capability-link possession grants read-only access to the live presentation projection; it does not create account membership, ownership, or collaboration rights.

## Database schema

The following is the logical schema. Exact SQL types and migration syntax must be adapted to the confirmed production API repository and migration tooling.

### `notebooks`

| Column | Constraint/purpose |
| --- | --- |
| `id` | UUID primary key |
| `owner_user_id` | Stable WorkOS user ID, indexed |
| `title` | Required, trimmed, maximum 200 characters |
| `description` | Optional, maximum 10,000 characters |
| `visibility` | `private` or `shared`; default `private` |
| `version` | Monotonic integer for optimistic concurrency |
| `created_at` | Server timestamp |
| `updated_at` | Server timestamp |
| `deleted_at` | Nullable soft-deletion timestamp |

Owner queries index `(owner_user_id, updated_at desc)` and exclude deleted rows.

### `notebook_items`

| Column | Constraint/purpose |
| --- | --- |
| `id` | UUID primary key; client generation allowed for offline creation |
| `notebook_id` | Foreign key to notebook with deletion cascade |
| `type` | `text` or `photo` |
| `position` | Non-negative integer; unique within active notebook ordering |
| `linked_place_id` | Nullable stable Sanity `page` `_id` |
| `text` | Required only for text items; maximum 100,000 characters |
| `photo_asset_id` | Required only for photo items |
| `caption` | Nullable photo caption; maximum 10,000 characters |
| `created_at` | Server timestamp |
| `updated_at` | Server timestamp |
| `deleted_at` | Nullable soft-deletion timestamp |

Database checks enforce valid field combinations for each type. Reorder operations submit the complete active item-ID order and update positions atomically. Creation time is never used as display order.

### `photo_assets`

| Column | Constraint/purpose |
| --- | --- |
| `id` | UUID primary key and stable reusable asset ID |
| `owner_user_id` | Stable WorkOS user ID, indexed |
| `storage_key` | Private processed-image object key |
| `thumbnail_storage_key` | Private thumbnail object key |
| `mime_type` | Allowlisted processed output type |
| `width`, `height` | Positive processed dimensions |
| `file_size_bytes` | Processed image size |
| `upload_status` | `pending`, `uploaded`, `processing`, `ready`, `failed`, or `deleting` |
| `created_at` | Server timestamp |
| `updated_at` | Server timestamp |
| `deleted_at` | Nullable soft-deletion timestamp |
| `delete_after` | Nullable delayed-cleanup timestamp |

Objects are never addressed publicly by storage key. A delivery endpoint or time-limited URL is generated from an authorised projection.

### `notebook_shares`

| Column | Constraint/purpose |
| --- | --- |
| `id` | UUID primary key |
| `notebook_id` | Foreign key, indexed |
| `token_hash` | Unique SHA-256 hash of a 32-byte random token |
| `created_at` | Server timestamp |
| `revoked_at` | Nullable; active only while null |
| `last_accessed_at` | Optional coarse operational timestamp |

Only one active share is needed per notebook in Stage 1. Rotation revokes the active row and creates a new row in one transaction. Raw tokens are never persisted or logged.

### Referential and concurrency rules

- Every item and asset mutation validates notebook ownership in the same transaction.
- A photo item may reference only an asset owned by the same user and in a usable state.
- Deleting a photo item schedules its asset only when no active item references it.
- Notebook writes use `expectedVersion`; success increments `version`, and mismatch returns a conflict without overwriting server content.
- Share revocation and notebook deletion are transactionally coupled to visibility.

## Photo storage and processing

The following storage policy is approved for Phase 1:

- TripIdeas stores an optimised copy of every photo added to a notebook.
- The device photo-library URI is temporary input and never the durable source.
- Full camera originals are not retained by default.
- Each upload receives a stable permanent internal asset ID.
- Store one high-quality notebook image and one thumbnail.
- Keep photo assets independent of notebook items so the same asset can be referenced by more than one future context.
- Original-resolution upload belongs to a later explicit licensing or publication workflow.

### Upload flow

1. Mobile asks the authenticated API to create an upload for the incoming file’s size, MIME type, dimensions where available, and client-generated request ID.
2. The server checks ownership context, rate limits, quota, MIME allowlist, and maximum 20 MB incoming size.
3. The server creates a pending `photo_assets` row and returns a short-lived, single-object upload instruction.
4. Mobile uploads from its app-managed pending file and then calls complete.
5. The server verifies object size/type and queues or performs processing.
6. Processing produces one notebook image with longest edge at most approximately 2,400 px and quality approximately 80–85%, plus one 400–600 px thumbnail.
7. Processing safely rejects unsupported, malformed, or corrupt files; strips unnecessary EXIF data, including embedded precise GPS; validates decoded image content; and updates the asset to `ready`.
8. Mobile polls or refreshes until ready, then replaces the pending local presentation with server URLs.

### Storage rules

- Do not retain the original full-resolution camera file by default.
- Originals and temporary uploads are deleted after successful processing or a short failure timeout.
- Processed objects are private by default.
- Delivery URLs must not expose bucket names, raw storage keys, owner identifiers, or permanent write credentials.
- Shared-page image access may be mediated by the active share token or a short-lived read URL whose lifetime defines the documented revocation cache window.
- Reusable assets remain separate from notebook items so later phases can link the same asset without duplicating bytes.
- Unreferenced assets receive a grace period before asynchronous object and database cleanup.
- Capture date or location may be retained only as separate controlled fields where supported and appropriate, not as uncontrolled metadata embedded in a shared image.

## API contract

The paths below describe the version-one contract. Their host is pending the authenticated API ownership decision.

### Response conventions

- JSON uses camelCase.
- Success returns the updated authoritative resource and `version` where relevant.
- Errors use:

```json
{
  "error": {
    "code": "notebook_conflict",
    "message": "The notebook changed on another device.",
    "requestId": "..."
  }
}
```

- Expected statuses include `400` malformed, `401` unauthenticated, `403` not owner, `404` absent, `409` version conflict, `413` request/file too large, `415` unsupported media, `422` validation, `429` rate/quota boundary, and `500` unexpected.
- Owner IDs are never accepted from the client.

### Owner endpoints

| Method and path | Purpose |
| --- | --- |
| `GET /notebooks` | List current owner’s non-deleted notebook summaries |
| `POST /notebooks` | Create a private notebook |
| `GET /notebooks/:notebookId` | Read owner notebook, ordered items, and resolved place summaries |
| `PATCH /notebooks/:notebookId` | Update title/description using `expectedVersion` |
| `DELETE /notebooks/:notebookId` | Delete notebook and revoke active share |
| `POST /notebooks/:notebookId/items/text` | Add text item at a requested position |
| `PATCH /notebooks/:notebookId/items/:itemId` | Update text, caption, or linked place as allowed by type |
| `DELETE /notebooks/:notebookId/items/:itemId` | Delete item and evaluate asset cleanup |
| `PUT /notebooks/:notebookId/order` | Atomically replace active item order |
| `POST /notebooks/:notebookId/photo-uploads` | Validate quota and create pending asset/upload instruction |
| `POST /notebooks/:notebookId/photo-uploads/:assetId/complete` | Verify upload and start processing |
| `POST /notebooks/:notebookId/photo-uploads/:assetId/retry` | Issue a safe retry when allowed |
| `POST /notebooks/:notebookId/share` | Create or rotate a read-only share token |
| `DELETE /notebooks/:notebookId/share` | Revoke active share immediately |

Text-item creation, photo-item attachment, and retries accept an idempotency key so queued mobile requests can be replayed safely.

### Public endpoint

| Method and path | Purpose |
| --- | --- |
| `GET /shared-notebooks/:token` | Return allowlisted read-only presentation data for an active share |

The public response contains notebook title/description, ordered content, display image URLs, captions, and resolved public place summaries. It excludes owner identity, internal IDs not needed for rendering, storage keys, versions, upload metadata, deletion data, and share hashes.

## Authentication and authorisation

- Authenticated routes require the existing WorkOS session or bearer token.
- The API validates the token/session and derives the stable WorkOS user ID before any query.
- Every owner query includes both resource ID and derived `owner_user_id`; existence must not grant access.
- Child resource writes verify the parent notebook owner, not merely the item or asset ID.
- Public token reads are the sole unauthenticated notebook operation.
- Mobile cached identity can select a local cache early, but writes wait for a confirmed session to avoid acting as a stale user.
- Ownership and quota checks occur server-side even if the mobile UI has already checked them.

The current repositories do not contain the WorkOS verifier. No authenticated route should be implemented until that code or an approved server-to-server identity verification boundary is available.

## Sharing and revocation

Notebook sharing is a revocable read-only capability-link model:

- Link-based public viewing means anyone possessing the high-entropy link may view the live notebook presentation without an account.
- Named authenticated membership or collaboration means an identified account receives durable permissions such as editing, commenting, or messaging.
- Only link-based public viewing is included in Phase 1. A capability link never creates membership and never grants collaboration rights.
- The shared page always reflects current server state. Edits appear, removed items disappear, notebook deletion removes access, and no immutable client snapshot becomes a second source of truth.

- Generate 32 cryptographically random bytes and encode them as an unpadded URL-safe token.
- Store only SHA-256 of the raw token.
- Public lookup hashes the supplied token and performs a constant-time comparison where supported.
- The URL contains no notebook ID, user ID, email, or sequential value.
- Share creation is explicit and returns the raw token only in that response.
- Revocation sets `revoked_at` and notebook visibility to `private` transactionally.
- Public reads use no-store database lookup. The page and API set `Cache-Control: private, no-store` and must not use a CDN cache for notebook HTML or JSON in Stage 1.
- All shared pages set `robots: noindex, nofollow, noarchive` and equivalent HTTP headers.
- Image delivery must honour revocation immediately or within an approved, documented short signed-URL lifetime.
- Deleted and revoked notebooks return a generic not-found response without revealing prior existence.

## Relationship to existing Trip Idea sharing

- Current Trip Idea sharing remains unchanged during Notebook Phase 1.
- Its public-page layout, recipient journey, login prompts, mobile share-sheet behaviour, place-card rendering, and other useful presentation patterns may be reused.
- Its Sanity-backed user-content storage and authenticated collaboration model must not be copied into the Notebook implementation.
- Notebook token, permission, and public-view boundaries should remain clean enough to inform a future shared access service for Notebooks, Trip Ideas, and itineraries.
- Future convergence is not a dependency, deliverable, migration, or scope expansion for Phase 1.

## Potential implications for existing Trip Idea sharing

The Notebook design exposes future architectural opportunities for Trip Ideas:

- migrating user-owned Trip Idea data from Sanity into the user-data platform;
- adding explicit capability-link revocation and rotation;
- separating capability-link viewers from named authenticated participants;
- sharing common ownership, membership, and permission concepts;
- applying consistent account-deletion cascades;
- using common reusable photo-asset storage and lifecycle handling; and
- sharing public-page access controls, indexing rules, and privacy projections.

All are later architectural opportunities. This RFC neither redesigns nor migrates existing Trip Idea sharing, and none is required for Notebook Phase 1.

## Local caching and offline behaviour

### Cache layout

- Cache keys are scoped by stable user ID, following saved-place precedent.
- Store a notebook index separately from each notebook snapshot to avoid rewriting all content for one edit.
- Store a durable per-user mutation queue with operation ID, notebook ID, base version, operation type, payload, attempt count, and timestamps.
- Copy pending photos into an app-managed directory using a stable local asset ID. Do not retain a photo-picker URI as the durable reference.
- SecureStore remains for credentials only.

### Synchronisation

- Render the cache immediately, then pull server state after authentication confirmation.
- Apply edits optimistically to the cache and enqueue the matching idempotent operation.
- Debounce text edits and cap request size; flush in notebook order.
- Server state remains authoritative after successful synchronisation.
- A version mismatch pauses further writes for that notebook, preserves the local draft/queue, fetches the current server snapshot, and prompts the owner to review rather than silently dropping either version.
- No field-level merge or real-time collaboration is required.
- Failed uploads retain the app-managed pending file and expose retry.
- “Synced” means the API has accepted content and any referenced photo asset is `ready`.
- Signing out removes active in-memory notebook state. The encrypted-auth boundary and product policy for retaining or clearing per-user notebook cache on a shared device must be confirmed before release; at minimum another account can never read it.

### Deletion while offline

- Offline deletion is queued and the item/notebook is hidden locally.
- If a deletion targets unsynced-only content, show a warning that no cloud copy exists.
- A deleted notebook cannot create a new share while deletion is pending.

## Privacy and deletion

- Notebooks default to private in both database and API behaviour.
- No notebook content is written to Sanity, analytics payloads, logs, push notifications, or search indexes.
- Shared presentation excludes owner email/name and internal metadata.
- Uploaded images are re-encoded without unnecessary EXIF or embedded GPS.
- Notebook deletion immediately revokes public access and soft-deletes relational records for an approved recovery period.
- Item deletion removes the relationship; shared assets survive while referenced.
- After the grace period, a cleanup job deletes unreferenced processed objects, thumbnails, temporary originals, and asset rows or tombstones them according to retention policy.
- Mobile removes deleted server content from its active cache after acknowledgement and clears abandoned pending files.
- Account deletion must revoke all shares and schedule all notebook assets. Its integration waits for the existing backend deletion contract.
- Backup expiry, legal holds, disaster recovery copies, and deletion completion times must be documented before rollout. A shared page must never be restored merely because an internal backup is restored.
- TripIdeas receives no editorial or commercial licence to notebook content in Stage 1.

## Storage quotas and safeguards

Initial safety defaults:

- Maximum incoming photo: 20 MB.
- Maximum processed longest edge: approximately 2,400 px.
- Processed quality target: approximately 80–85%.
- Thumbnail width: 400–600 px.
- Processed-photo allowance: approximately 1 GB per user.
- Warning threshold: approximately 80%.
- Text item: maximum 100,000 characters.
- Notebook title: 200 characters.
- Notebook description and photo caption: 10,000 characters each.
- Generous configurable active-item count per notebook; proposed starting safety ceiling: 5,000.
- No practical product-facing notebook-count limit, but server-side abuse/rate controls still apply.
- Generous notebook-level structured-content and request-size limits.
- Bounded upload concurrency, short-lived upload instructions, idempotency, request-size limits, MIME allowlist, decoded-image validation, and per-user/IP rate limiting.
- Debounced or throttled autosave.
- Server-side validation for all safeguards.
- No unlimited revision history in Phase 1.

These are approved safety and abuse-control boundaries, not a prominent commercial tier. Quota errors never delete local pending files automatically. Approximate limits remain server configuration rather than duplicated mobile constants.

## Implementation order

### Preflight

1. Locate and audit the preferred `tripideas-api` source and WorkOS verification.
2. Confirm the existing user-data Postgres provider, ownership, staging/production separation, migration conventions, and backup/restore arrangements.
3. Confirm how the approved Notebook deletion cascade integrates with current account deletion; leave exact recovery and backup-expiry periods unresolved until the infrastructure audit.
4. Present the focused audit and Milestone 1 plan for review before implementation.

### Milestone 1 — Data and API foundation

- Add notebook, text-item, versioning, ownership, and deletion migrations.
- Implement authenticated notebook CRUD and ordering.
- Add schema, ownership, malformed-request, and concurrency tests.
- Do not include photos.

### Milestone 2 — Mobile text notebook

- Add Notebooks under Saved using existing navigation/design conventions.
- Add list, create, edit, delete, ordered text, per-user cache, durable queue, and conflict state.
- Produce the first end-to-end cross-device text notebook.

### Milestone 3 — Photo assets and upload pipeline

- Select and provision approved storage only after separate approval.
- Add asset schema, upload validation, processing, thumbnails, quota accounting, delayed cleanup, and operational metrics.

### Milestone 4 — Mobile photo experience

- Add SDK-compatible photo picker and explicit app-file dependency.
- Add pending preview, progress/state, retry, captions, and safe unsynced deletion.

### Milestone 5 — Place linking

- Reuse published place search/ID/projection.
- Store only stable editorial place IDs and resolve allowlisted summaries.

### Milestone 6 — Read-only sharing

- Add token creation/rotation/revocation.
- Add public no-store shared page and privacy metadata.
- Add native share sheet and copy controls.

### Milestone 7 — Hardening and rollout

- Complete accessibility, performance, privacy, deletion, telemetry, quota, real-device, and TestFlight checks.
- Document operations, recovery, cleanup, and incident response.

## Logical commit boundaries

1. `docs: add stage 1 notebook RFC`
2. `feat(api): add notebook database model`
3. `feat(api): add notebook CRUD and ownership checks`
4. `feat(mobile): add notebook list and creation`
5. `feat(mobile): add offline text notebook editor`
6. `feat(api): add notebook photo asset pipeline`
7. `feat(mobile): add notebook photo uploads`
8. `feat(mobile): add notebook item reordering`
9. `feat(notebooks): link content to editorial places`
10. `feat(web): add read-only shared notebooks`
11. `feat(mobile): add notebook sharing controls`
12. `test: harden notebook privacy and offline flows`
13. `docs: document notebook operations and rollout`

Each milestone must present exact files, migrations, services, test results, and diff summary before production migrations, cloud provisioning, credentials, deployment, or irreversible writes.

## Test plan

### Backend

- Stable identity-derived ownership; body/query owner spoofing rejected.
- Owner CRUD and cross-user read/write denial.
- Private-by-default creation.
- Text/type/size validation and malformed JSON.
- Stable ordering, invalid reorder lists, duplicates, and transactional rollback.
- Idempotent offline replay.
- Expected-version success and conflict without silent overwrite.
- Soft deletion, share revocation, and account cascade.
- Upload MIME, decoded content, 20 MB limit, quota, retry, and processing failure.
- Same-owner asset linkage and cross-owner asset rejection.
- Reference counting and delayed orphan cleanup.
- High-entropy share creation, hashed storage, rotation, invalid/revoked/deleted tokens.
- Public projection privacy and rate limiting.

### Mobile

- Signed-out gate and authentication loss during editing.
- Multiple notebook create/open/rename/delete.
- Empty notebook, long text, large item counts, and validation feedback.
- Text/item/caption edit and reorder.
- Per-user cache isolation on account switch.
- Cold-start cached read, offline edit, replay, conflict preservation, and retry.
- Photo selection, immediate pending preview, upload success/failure/retry, oversized/corrupt file, and pending-file cleanup.
- Unsynced deletion warning.
- Place search/link/unlink and missing editorial place.
- Share creation, native share sheet, copy, revoke, and rotate.
- Accessibility labels, dynamic text, keyboard, and screen-reader ordering.

### Shared web

- Responsive phone and desktop rendering.
- Exact item order, text, captions, images, missing-photo fallback, and place links.
- No edit controls or authenticated owner metadata.
- Valid, malformed, revoked, rotated, and deleted tokens.
- `noindex`, `nofollow`, `noarchive`, and no-store headers.
- No storage keys, owner IDs, emails, internal versions, or EXIF leakage.
- Image access after revocation within the approved maximum cache window.

### Operations

- Migration forward/rollback rehearsal in non-production.
- Storage lifecycle and orphan cleanup dry run.
- Quota metrics and alert test.
- Backup restore test that preserves share revocation and deletion semantics.
- Real-device poor-connectivity and interrupted-upload testing.

## Rollout approach

1. Keep notebooks behind authenticated server and mobile feature flags.
2. Ship Milestone 1 to a non-production database and run ownership/security tests.
3. Dogfood text-only notebooks with a small internal cohort before photos.
4. Provision storage only after provider, cost, privacy, retention, and processing review.
5. Enable photos for the internal cohort with quota, processing-failure, and orphan metrics.
6. Enable sharing only after token, revocation, indexing, and metadata tests pass.
7. Expand through TestFlight cohorts while monitoring API errors, conflict rates, queue age, upload failures, processed storage, cleanup backlog, and public-token abuse.
8. Preserve a server-side kill switch for new uploads and new share creation without making existing private notebooks unavailable.
9. Do not migrate existing local Trip Ideas into notebooks during Stage 1.

## Architecture decisions and remaining audit questions

The following decision status is approved. No provider or infrastructure is created or selected by this RFC.

| Decision | Approved direction | Audit question or remaining choice | Required by |
| --- | --- | --- | --- |
| Authenticated API host | Prefer `tripideas-api`; do not create a second authenticated API unless the audit invalidates this assumption. | Confirm repository ownership, deployment path, WorkOS bearer verification, route conventions, and capacity to add Notebook endpoints. | Before Milestone 1 |
| User-data Postgres | Prefer the existing API-owned user-data Postgres. | Confirm provider and owner, staging/production separation, schema and migration tooling, migration deployment, backup/restore arrangements, and whether Notebook tables fit the existing boundary. | Before Milestone 1 |
| Account deletion and retention | Immediately invalidate authenticated access and capability links; support documented soft deletion/recovery; asynchronously clean unreferenced photo assets; allow backups to expire under policy. | Confirm current cascade integration, recovery mechanism, backup restoration behaviour, legal/operational retention, and exact periods. Exact periods remain unresolved. | Schema integration before Milestone 1; asset details before Milestone 3; operational periods before rollout |
| Object storage and image processing | Keep the approved provider-neutral photo policy; defer provider selection. | Select provider, upload, processing, validation, delivery, and lifecycle infrastructure during the photo milestone. | Before Milestone 3 |
| Mobile sign-out behaviour | No decision recorded by this approval. Another account must never read cached notebook data. | Choose clearing versus encrypted per-user retention and define treatment of unsynced text and pending photos. | Before Milestone 2 |
| Existing Trip Idea sharing | Leave its sharing, collaboration, and Sanity storage unchanged during Notebook Phase 1. | Future convergence remains an architectural opportunity only. | Outside Phase 1 |

Additional sharing details may wait until Milestone 6:

- Confirm the canonical public path, proposed as `https://www.tripideas.nz/notebook/{token}`.
- Approve the minimal signed-image URL lifetime after revocation, or require each image read to validate the active capability.
- Confirm one active capability link per notebook in Phase 1; this RFC recommends one, with replacement implemented as revoke then create.

## Approval recorded

The Phase 1 Notebook RFC and the decisions above were approved on 2026-07-25. Approval authorises the focused `tripideas-api` audit and preparation of a Milestone 1 implementation plan only. It does not authorise migrations, schema changes, cloud resource creation, credential changes, deployment, or implementation code.
