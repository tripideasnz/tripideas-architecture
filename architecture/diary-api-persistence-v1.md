# Diary API persistence v1

Status: approved for local implementation. Staging and production rollout require separate approval.

## Domain boundary

Diary is a dedicated owner-scoped aggregate and does not reuse Notebook's
`UserContentDocument`, `UserContentPage`, or `UserContentItem` tables. Its
canonical hierarchy is `Diary -> DiaryDay -> DiaryTopic -> DiaryObject`, with
ordered `DiaryCoverMedia` attached directly to the Diary.

Days are identified and ordered by calendar date. A configured date range does
not pre-create Day rows; Days are instantiated lazily. Topics have an explicit
user-controlled order independent of optional start time. Every Topic object
participates in one explicit mixed order.

## Lifecycle and ownership

Diary uses the smallest complete lifecycle: `ACTIVE` and `DELETED`, with
`deletedAt` and `purgeAfter`. All owner routes authenticate through the mobile
bearer identity and scope the root Diary query to the canonical API user. A
nested Day, Topic, object, or cover-media identifier is valid only when it
belongs to that locked owner Diary.

The Diary row is the optimistic-concurrency aggregate. Every mutation locks it,
checks `expectedVersion`, changes the aggregate transactionally, and increments
the Diary version exactly once. Calendar dates are not reorderable. Topic,
object, and cover-media positions are dense, non-negative sequences.

## Idempotency

Every mutation carries a non-empty `clientRequestId`. `DiaryMutationReceipt`
is unique by Diary and request ID and stores the operation, canonical payload
hash, resulting version, and optional resulting entity ID. Receipt lookup occurs
before expected-version validation.

An exact retry returns the current authoritative Diary without applying the
mutation or incrementing its version again. Reusing a request ID for a
materially different operation or payload is rejected. Create-Diary requests
also use an owner-scoped unique create request ID so retries can resolve before
an aggregate-specific receipt exists.

## Objects

The v1 vocabulary is `NARRATIVE`, `PHOTO`, `LINK`, `EDITORIAL_PLACE`,
`PERSONAL_PLACE`, and `PIN`.

- Narrative owns optional title and text.
- Photo references one completed, same-owner `PhotoAsset` and may have a caption.
- Link owns an HTTP(S) URL and optional title and note; creation is atomic.
- Editorial Place stores an external identifier, title snapshot, and optional
  location snapshot. It has no relational source FK.
- Personal Place stores a same-owner `PersonalPlaceCard` reference, title
  snapshot, and optional location snapshot. It deliberately stores no body
  snapshot in v1.
- Pin owns optional label, coordinates, source, accuracy, and map-inclusion state.

Duplicate Place references are permitted. Personal Place and PhotoAsset foreign
keys are restrictive. When either Place source is unavailable, its object stays
in sequence and exposes only the last-known title; body and coordinates are
withheld. Removing Photo objects or cover media detaches the asset without
deleting it, and v1 introduces no automatic orphan cleanup.

## Cover and date-range behavior

A Diary may contain at most four cover-media rows. Attachment validates that
the PhotoAsset belongs to the Diary owner and is uploaded/usable. Cover order is
independent of Topic object order.

Reducing a Diary date range reports instantiated Days outside the proposed
range. They are deleted only when the same mutation explicitly supplies
`removeOutsideDays: true`; deletion of those Days and the range update are one
transaction and one version increment.

## Deletion and account deletion

Owner deletion soft-deletes the Diary aggregate. Contained rows remain until
purge so mutation receipts and exact retry behavior remain available. Final
account deletion hard-deletes Diary aggregates first; contained rows cascade,
releasing restrictive Personal Place and PhotoAsset references before those
owner records are removed.

Personal Place soft deletion leaves Diary objects representable as unavailable.
Hard purge is prohibited while a Diary object still references the card. This
does not weaken the stricter existing Trip attachment rules.

## Contract and capability

Owner mutations return a refreshed authoritative Diary DTO. Stale versions use
HTTP 409 with `diary_version_conflict` and the current version. Other-owner
identifiers are projected as not found.

The deployable API advertises `diaries-v1`. Clients distinguish a successful
identity response without the capability from a temporarily unreachable API;
they must never silently fall back to local Diary authority. Prototype-v1
mobile Diaries are disposable and are not imported.

## Future sharing

Sharing is excluded from v1. Stable aggregate/object identities, explicit
ownership, snapshots, private PhotoAsset references, unavailable states,
versioning, and soft deletion permit a later Diary-specific share aggregate
without changing the owner persistence hierarchy.

## Rollout gates

1. Validate additive migrations on clean PostgreSQL 16 and a disposable fully
   migrated baseline.
2. Validate ownership, locking, idempotency, ordering, references, account
   deletion, and existing API regressions locally.
3. Commit and push architecture and API work separately.
4. Before staging, record exact source/service/database identities, all shared
   readers and writers, the migration ledger, and a verified backup/restore
   point.
5. Apply migrations with `prisma migrate deploy`, verify constraints/indexes,
   then deploy the exact API SHA.
6. Advertise and accept `diaries-v1` only when schema, routes, readiness, and
   authenticated acceptance all pass.

Rollback is application-first: disable Diary writers/capability, retain the
additive schema, and forward-fix. Production requires separate explicit
authorization.
