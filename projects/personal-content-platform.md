# TripIdeas Personal Content Platform

## Status

Vision Document (Living)

This document describes the long-term vision and guiding principles for the TripIdeas Personal Content Platform. It intentionally focuses on product direction rather than implementation detail. Technical designs, RFCs and implementation plans will be developed separately as individual phases begin.

## Purpose

TripIdeas helps travellers discover places through high-quality editorial content.

The Personal Content Platform extends this by giving every traveller a personal workspace where they can capture, organise, enrich and publish their own travel experiences.

The goal is to create lasting value beyond a single trip. A traveller’s content should become a growing personal travel library that supports planning future journeys, remembering past experiences, sharing with others, and contributing back to the wider TripIdeas community.

## Vision

Create the leading platform for capturing, organising and publishing travel experiences.

Users should be able to begin with the simplest possible travel notebook and progressively enrich their content without needing to recreate or reorganise it as their needs evolve.

The same underlying content should support multiple future uses including private journals, collections, Trip Ideas, itineraries, blogs, books, social sharing and editorial contributions.

## Guiding Principles

### Traveller ownership

The traveller owns their content.

TripIdeas provides tools to organise, improve, publish and share that content, but ownership remains with the creator.

### One source, many outputs

The user’s notebook becomes the canonical source.

From that source, content may later be published as:

- shared notebook
- Trip Idea
- itinerary
- travel story
- blog
- printed book
- PDF
- social media content
- TripIdeas editorial submission

Users should never need to recreate the same content for different outputs.

### Progressive structure

Users should be able to start with completely free-form notes.

As they choose, those notes can gradually become more structured through the addition of:

- places
- photos
- collections
- dates
- maps
- headings
- tags
- other structured objects

Structure should assist users rather than constrain them.

### AI assists, it does not replace

Artificial intelligence should reduce effort rather than replace creativity.

Examples include:

- improving prose
- organising notes
- suggesting headings
- generating captions
- grouping photos
- linking places
- preparing publication layouts

The traveller remains the author.

### Editorial independence

TripIdeas editorial content remains separate from user-created content.

Editorial recommendations, Nearby Places and other curated content continue to follow editorial processes.

User-created content may later be submitted for editorial consideration under an explicit licensing process.

### Explicit publishing

Publishing is always an intentional user action.

Users decide when content becomes:

- private
- shared
- public
- submitted to TripIdeas
- licensed for editorial use

### Reusable content

Every object should be reusable.

A photo, note or place reference should not belong exclusively to a single notebook or trip.

Instead, content should be linkable into multiple contexts while maintaining a single underlying source.

## Implementation refinement — Content Block Architecture

Notebook Phase 1 confirmed that Pages remain the primary organisational unit presented to users. Pages represent natural chapters or sections within a Notebook, preserving a document-oriented experience rather than asking travellers to work through a collection of forms.

Internally, each Page should contain one or more typed Content Blocks. The initial implementation provides only the Text block. Future block types are expected to include:

- Photo
- Place
- Route
- Map
- Checklist
- Web Link
- AI-generated content

Pages are explicitly persisted beneath their owning Notebook and have their own
order, title and lifecycle. Each Page contains a separately ordered collection
of typed Content Blocks. During the compatibility rollout,
`UserContentItem` remains the physical block table, while existing flat Text
endpoints and responses continue to adapt the persisted Page/block structure
for the completed mobile Text MVP. Photo is the next planned block type, but is
not part of this persistence migration.

This is an implementation refinement rather than a change of direction. Content Blocks are an internal abstraction that allows richer content to be introduced without redesigning the Page or Notebook model.

> Users think in Pages. The platform thinks in typed Content Blocks.

This architecture provides:

- independent evolution of new content types
- modular rendering
- reusable user-owned assets, especially photographs
- support for future publishing formats
- support for future AI processing at block level
- cleaner separation between content structure and presentation

The same underlying block-based content can support future outputs including:

- private Notebooks
- Trip Ideas
- travel journals
- TripIdeas editorial submissions
- PDFs
- books
- other future publication formats

This refinement does not change the existing ownership or privacy model. All user content remains user-owned. Typed Content Blocks exist to improve flexibility and future reuse; no additional sharing or publication occurs without explicit user action.

## Implementation refinement — Private Photo Assets

Photographs are reusable, user-owned assets and remain separate from Notebook
Pages and Content Blocks. A future Photo block will reference a PhotoAsset
rather than owning an uploaded file, allowing the same photograph to be reused
across Notebooks and future publication formats without duplicating its binary
or metadata.

PhotoAsset persistence records ownership, idempotent creation, upload and
processing lifecycle, derivative dimensions and file sizes, private location
metadata, and delayed-deletion eligibility. Binary source, processed and
thumbnail files live in private object storage rather than PostgreSQL. The
database stores only opaque internal storage keys; keys, provider configuration
and credentials are not exposed through normal owner-facing content contracts.

Storage access is isolated behind a provider-neutral adapter. Railway Buckets
are the approved initial provider direction, pending a separate provisioning
and storage-adapter milestone, but the asset model contains no Railway-specific
bucket, URL or credential fields.

Location remains asset-level private metadata. Removing location physically
clears precise coordinates and all location-derived fields. Sharing or
publishing a photograph will not share precise location unless a future,
explicitly approved product action says otherwise. Processed image files will
strip GPS EXIF before any later sharing or publication workflow.

PhotoAsset deletion is staged: an active asset may be soft-deleted with a
future purge time, after which storage cleanup can make it purge eligible.
Processed and thumbnail bytes form the durable quota-accounting basis.
Asset versions begin at one and advance for owner-visible metadata mutations,
including location changes and deletion requests. Physical user deletion is
restricted while owned assets remain. PhotoBlock persistence and asset
references belong to the next milestone and are intentionally not introduced
by the asset-foundation migration; those references must later prevent physical
asset deletion while an active block still uses the asset.

## Scope

The Personal Content Platform focuses on traveller-created content.

It does not replace:

- editorial content
- Nearby Places
- itinerary management
- recommendation engines

Those systems will integrate with the platform over time.

## Roadmap

### Phase 1 — Shareable Notebook

Deliver a simple travel notebook supporting:

- text notes
- photos
- links to TripIdeas places
- basic organisation
- sharing with friends

The notebook should feel lightweight and easy to use.

### Phase 2 — Structured Content

Introduce structured travel objects including:

- place cards
- collections
- travel days
- activities
- accommodation
- restaurants
- viewpoints

Users should still feel they are editing a document rather than filling in forms.

Collections and structured place cards become reusable building blocks for future Trip Ideas.

The approved architecture for the first structured object is recorded in
[Stage 2 — Personal Place Cards](personal-content-platform/stage-2-personal-place-cards.md).
It defines a canonical user-owned Place Card, reusable PhotoAsset relationships,
readiness rules, and discriminated integration with Trip Idea entries.

### Phase 3 — TripIdeas Contribution

Allow users to offer selected content to TripIdeas.

Potential contributions include:

- photographs
- written content
- structured place information

TripIdeas may review, accept, decline or license contributions through an explicit workflow.

### Diary — deliberate journey composition

Diary is a separate structured artifact built from selected user-owned and
TripIdeas source material. It organises ordered Days, freely named Topics and
presentation items while leaving Notebook capture, Trip planning and Personal
Place authority unchanged. Manual composition is complete without AI; future AI
acts through reviewable proposals.

The approved domain, provenance, ordering, map, sharing and rights boundaries
are recorded in [Diary foundation](personal-content-platform/diary-foundation.md).
Local product work may proceed while database persistence remains deferred.

### Phase 4 — Publishing

Support multiple publication formats generated from the same underlying content.

Potential outputs include:

- blogs
- PDFs
- printed books
- social media
- public travel stories
- commercial publishing partnerships

Implementation should favour existing technologies and specialist partners wherever appropriate rather than building every publishing capability internally.

### Phase 5 — AI Assistance

Expand AI assistance to improve content creation and organisation.

Potential capabilities include:

- rewriting notes
- summarising travel days
- creating captions
- organising photos
- identifying linked places
- suggesting missing information
- preparing publication drafts

AI should always operate as an assistant rather than replacing the traveller’s own experiences.

## Relationship to Other Projects

### Nearby Places

Nearby Places remains an editorial recommendation system.

The Personal Content Platform may reference Nearby Places but does not alter editorial recommendations.

### Trip Ideas

Trip Ideas will consume structured user content created within the Personal Content Platform.

The platform itself remains independent of itinerary planning.

### Itinerary Platform

The future itinerary system will build upon structured notebooks and collections rather than replacing them.

### Knowledge Graph

The TripIdeas editorial knowledge graph and the traveller’s personal content graph remain distinct.

They intersect through shared references to places, regions and other travel entities.

## Success Measures

Success is not measured by the number of AI features.

Instead, success is measured by whether travellers continue using TripIdeas throughout the entire travel lifecycle:

- planning
- travelling
- remembering
- sharing
- publishing
- returning for future journeys

The Personal Content Platform should become the traveller’s long-term home for travel experiences rather than a feature used only during a single trip.

## Living Document

This document is expected to evolve.

As each roadmap phase begins, a dedicated RFC will be created describing detailed architecture, implementation plans and acceptance criteria while remaining aligned with the principles described here.
