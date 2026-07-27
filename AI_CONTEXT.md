# AI_CONTEXT.md

Version 1.2

# TripIdeas – AI Project Context

Last updated: 2026-07-28

Status: Active

# Purpose

A concise operational briefing for AI coding assistants. This document
complements, but does not replace, the architecture repository. It is updated
only at major project milestones.

---

# Project

TripIdeas is evolving into a User Content Platform built around reusable
content rather than individual application features.

The long-term hierarchy is:

User
→ Documents (Notebooks, Trip Ideas, etc.)
→ Pages
→ Content Blocks
→ Assets (optional)

Applications (Notebook, publishing, Trip Ideas, public content, etc.) are built
on this common platform.

The architecture repository remains the authoritative design source.

---

# Current Status

## Nearby Places

Status: COMPLETE

- deployed
- deterministic architecture
- production operational

---

## User Content Platform

### Phase 1 — Text Notebook

Status: COMPLETE

Completed:

- authenticated user-owned notebooks
- versioned documents
- persisted Pages
- generic Content Block framework
- Text Block implementation
- autosave
- optimistic concurrency
- conflict handling
- offline cache
- staging migration
- mobile acceptance

Notebook UI is intentionally lightweight and will continue to evolve.

---

### Phase 2 — Asset Platform

Current phase.

Completed:

- provider-neutral PhotoAsset schema
- provider-neutral storage adapter
- Railway Bucket integration
- staging migration
- staging private bucket
- signed PUT / HEAD / GET / DELETE verification
- provider-neutral storage abstraction
- private location model
- operations documentation

Not yet implemented:

- upload API
- processing worker
- thumbnails
- Photo Blocks
- mobile photo picker
- image processing
- EXIF extraction
- cleanup worker

---

# Architecture Principles

Always preserve:

- deterministic core architecture
- AI only where it adds value
- provider-neutral abstractions
- staging before production
- additive migrations
- small reviewable commits
- architecture-first implementation

Never bypass:

- migrations
- reviews
- staging verification
- backups

---

# Current Milestone

Photo Upload API

Goal:

Allow an authenticated user to securely upload a private image and create a
managed PhotoAsset.

This milestone finishes when:

- upload intent endpoint exists
- signed upload succeeds
- completion endpoint validates upload
- PhotoAsset reaches UPLOADED state

No rendering yet.

---

# Next Milestones

1. Photo Upload API
2. Processing worker
3. Photo Block
4. Mobile photo import
5. Private location controls
6. Cleanup lifecycle
7. Additional Content Blocks

---

# Storage

Approved direction:

- Railway Buckets
- provider-neutral S3 abstraction
- private buckets
- staging and production separated
- opaque storage keys
- signed URLs only

Storage provider must remain replaceable.

---

# Privacy

User content is private by default.

Location metadata:

- private
- asset-level
- never implied by sharing
- removable by user
- not used by TripIdeas unless explicitly contributed

---

# Working Style

Default expectations:

- concise responses
- minimal prompts
- no repetition of established architecture
- small focused commits
- deterministic implementation
- preserve clean working trees

Do not:

- redesign completed architecture
- introduce parallel documents
- broaden milestone scope without approval

---

# Repository Status

Architecture:

- authoritative design documentation
- PCP is the primary architecture document

API:

- User Content Platform foundation complete
- PhotoAsset persistence complete
- Storage adapter complete

Mobile:

- Text Notebook complete
- Generic Content Blocks complete

## Milestone History

v1.0

- Initial AI project context.

v1.1

- User Content Platform.
- Text Notebook complete.
- Pages and Content Blocks introduced.

v1.2

- Provider-neutral PhotoAsset model.
- Provider-neutral storage adapter.
- Railway Bucket foundation.
- Current milestone: Photo Upload API.

# Read First

Before beginning work:

1. Read AI_CONTEXT.md.
2. Assume architecture documents are authoritative.
3. Continue from the current milestone.
4. Do not revisit completed design decisions unless new information requires
   it.
