# Polymorphic Trip Idea entries

## Status

Accepted

## Context

The existing persisted Trip Idea is an `Itinerary` containing ordered
`ItineraryEntry` records. Each entry currently carries an opaque `placeId`
representing an editorial TripIdeas place. The API does not own an editorial
Place table.

Stage 2 must allow the same ordered Trip Idea to contain both editorial places
and canonical Personal Place Cards without duplicating card content or creating
two competing order systems.

## Decision

Evolve `ItineraryEntry` into an explicitly discriminated relationship that
references exactly one of:

- an editorial TripIdeas place through an external editorial ID; or
- a canonical `PersonalPlaceCard` through a relational reference.

The two targets are mutually exclusive and must be protected by database and
API validation.

The entry owns:

- the containing Trip Idea;
- its participation in Trip Idea ordering;
- future itinerary-specific metadata; and
- future per-Trip-Idea annotations.

The Personal Place Card owns its shared title, body, media and location.

Removing a Personal Place Card entry from one Trip Idea removes only that
entry. It does not delete the card or affect references in other Trip Ideas.
Canonical-card deletion is rejected while one or more active entries reference
the card. The API returns the active attachment count. The user must explicitly
remove the card from every Trip Idea before deleting the canonical record;
deletion never performs an implicit detach-all operation.

Trip Idea API responses expose a discriminated union. They retain a common
presentation opportunity without presenting editorial and personal targets as
identical domain records.

Existing entries are backfilled deterministically as editorial entries.
Existing `entryOrder` remains client-authoritative initially, with API
ownership and input validation. A broader ordering redesign is deferred until
implementation evidence demonstrates a need.

## Authorization boundary

The authenticated, authorized route or trusted persisted record is the only
source of the parent Trip Idea identity. A conflicting `itineraryId` supplied
in a request body must not redirect a mutation.

Personal-card attachment must transactionally verify:

- ownership and active state of the Trip Idea;
- ownership and active state of the card;
- current Trip-Idea readiness;
- absence of a duplicate attachment;
- ordering validity; and
- idempotency for retried client requests.

Entries belonging to soft-deleted Trip Ideas must not appear in active reads.

## Consequences

- Editorial and personal items share one ordered entry stream.
- Existing editorial IDs remain intact and do not require synthetic local
  editorial records.
- Canonical Place Card edits are reflected through every entry.
- Canonical card deletion is predictable and cannot silently remove content
  from several Trip Ideas.
- Clients must switch on the explicit source type.
- Migrations require an additive discriminator, nullable card reference,
  deterministic editorial backfill and exactly-one-target validation.
- Legacy editorial DTO compatibility must be preserved during rollout.
- Current ownership-boundary and deleted-parent filtering defects must be fixed
  before or alongside the extension.

## Alternatives rejected

### Separate personal-card join table

Rejected because it creates two item collections that must be merged, a new
cross-table ordering system, and duplicate authorization and lifecycle logic.

### Encode a Personal Place Card ID in the existing `placeId`

Rejected because prefix-based application logic cannot provide adequate
referential integrity, ownership validation or deletion behaviour.

### Copy the card into each Trip Idea entry

Rejected because content would diverge, edits would not propagate, and media
and rights metadata would be duplicated.

### Create synthetic editorial Place rows

Rejected because editorial content remains externally owned and resolved.
Forcing it into a local personal-content table would blur system-of-record and
ownership boundaries.

## References

- [Stage 2 — Personal Place Cards](../projects/personal-content-platform/stage-2-personal-place-cards.md)
- [Personal Place Card domain model](personal-place-card-domain-model.md)
- Source audit: `tripideas-api`, branch
  `feature/notebook-user-content-foundation`, SHA
  `f6737de192c855a89081a83d6d7cba36456d9227`
