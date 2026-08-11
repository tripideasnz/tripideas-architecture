# TripIdeas Architecture

This repository is the shared source of truth for long-lived TripIdeas product vision, architecture, major project documents, RFCs, and architectural decisions across mobile, web, CMS, data, and future services.

It captures durable direction and cross-project context. Detailed implementation work remains in the repository responsible for delivering it.

## Structure

- `vision/` contains product direction, design principles, and the roadmap.
- `projects/` contains major cross-cutting project documents.
- `architecture/` is reserved for architecture documentation and RFCs.
- `decisions/` is reserved for architectural decision records.

Mandatory cross-service policy: [Shared database schema and API compatibility
discipline](architecture/shared-database-schema-discipline.md).
