# Personal Place Card domain model

## Status

Accepted

## Context

Stage 2 needs a constrained, reusable user-created place object that can appear
in several Trip Ideas while remaining owned and controlled by its creator.
Stage 1 Notebook is intentionally flexible and document-oriented, while current
Trip Idea entries contain only opaque editorial place IDs.

The platform already has reusable, independently owned `PhotoAsset` records and
established patterns for owner scoping, optimistic concurrency, idempotency and
soft deletion.

## Decision

Introduce one canonical, user-owned and versioned `PersonalPlaceCard`.

The card:

- is independent of Notebook and Trip Ideas;
- may be referenced by multiple Trip Ideas;
- is edited once, with every reference resolving the current canonical record;
- supports incomplete drafts;
- must pass explicit Trip-Idea-readiness validation before attachment;
- begins with a title and plain-text body;
- uses one main photo and up to ten ordered body photos;
- requires confirmed card-level coordinates before attachment; and
- preserves provenance, rights basis and publication assessment separately
  from private ownership.

Media is associated through an ordered, role-based
`PersonalPlaceCardMedia` relationship to existing `PhotoAsset` records. The
relationship owns `MAIN` or `BODY` role and body ordering. It does not duplicate
the stored file.

Trip-Idea-valid content requires:

- title of 1–80 trimmed characters;
- non-empty plain-text body up to 8,000 characters;
- exactly one eligible main photo;
- no more than ten eligible body photos;
- confirmed valid coordinates; and
- an active, owner-consistent aggregate.

An attached card may not be edited into an invalid state.

Component provenance, rights status and publication eligibility remain distinct:
control of a private Place Card does not prove ownership of every component or
grant TripIdeas publication rights.

## Consequences

- One edit updates every Trip Idea use of the same card.
- Notebook material must be explicitly copied or referenced into the card; the
  card never depends on live Notebook rendering.
- PhotoAssets remain reusable across Notebooks, Place Cards and later contexts.
- Readiness must be evaluated from the current aggregate, not only a stored
  state label.
- Owner-scoped mutations need optimistic concurrency because the canonical card
  may be edited from several contexts.
- Asset deletion must be blocked while an active card relationship requires it.
- Canonical card deletion must be rejected while active Trip Idea attachments
  exist. The API returns the active attachment count, and the user removes each
  Trip Idea entry explicitly before deleting the card.
- Account deletion must coordinate cards, relationships and stored assets.
- Publication, licensing and moderation require later explicit workflows.

## Alternatives rejected

### Make Notebook the Place Card parent

Rejected because Notebook is a flexible capture document and a Place Card is a
constrained canonical object. Live dependence would couple final structured
content to Notebook layout and lifecycle.

### Copy a complete Place Card into every Trip Idea

Rejected because copies diverge, duplicate media and provenance, and violate
canonical edit-once semantics.

### Reuse arbitrary Notebook Content Blocks as the card schema

Rejected for the initial version because the product requires a predictable
title, body, photo group and confirmed location rather than arbitrary document
layout.

### Require a postal address

Rejected because remote, rural and unnamed locations must remain valid.
Confirmed coordinates are authoritative; address lookup is supplementary.

## References

- [Stage 2 — Personal Place Cards](../projects/personal-content-platform/stage-2-personal-place-cards.md)
- [Polymorphic Trip Idea entries](polymorphic-trip-idea-entries.md)
- Source audit: `tripideas-api`, branch
  `feature/notebook-user-content-foundation`, SHA
  `f6737de192c855a89081a83d6d7cba36456d9227`
