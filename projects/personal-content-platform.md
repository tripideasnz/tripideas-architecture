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

### Phase 3 — TripIdeas Contribution

Allow users to offer selected content to TripIdeas.

Potential contributions include:

- photographs
- written content
- structured place information

TripIdeas may review, accept, decline or license contributions through an explicit workflow.

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
