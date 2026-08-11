# Shared database schema and API compatibility discipline

Status: mandatory cross-repository policy

## Boundary

TripIdeas API services may have independent deployment lifecycles while sharing
one database in the same non-production environment. Deployment independence
does not create independent schema ownership.

The canonical relational schema and migration ledger live in
`tripideas-api/prisma`. No mobile, web, CMS, worker, or additional API service
may maintain a competing schema or mutate the database outside checked-in
Prisma migrations.

## Mandatory rules

1. Use `prisma migrate deploy` for an approved migration. Never use
   `prisma db push` against a shared environment.
2. Inspect the live migration ledger and every service that shares the target
   database before applying a migration.
3. Make shared-environment migrations additive and compatible with every
   currently deployable reader and writer. Adding a required column, removing
   or renaming a field, changing meaning in place, or tightening a constraint
   requires an explicit expand/migrate/contract sequence.
4. Deploy schema expansion before code that requires it. Deploy compatible
   readers and writers next. Backfill and validate separately. Remove legacy
   schema only after all dependent services are proven off it.
5. A service deployment must not silently apply unrelated or unreviewed schema
   changes. Record the exact source SHA and migration set before deployment.
6. API DTOs are contracts, not direct database mirrors. Preserve compatible
   response fields through the agreed client support window and version
   breaking contracts explicitly.
7. Each independently deployed API exposes a non-secret identity endpoint with
   build SHA, environment, API version, and capabilities. Clients must verify
   compatibility without treating network unavailability as incompatibility.
8. A generic route 404 is an environment/capability failure. A missing record
   uses a domain-specific error code such as `notebook_not_found`; clients must
   not present the former as deleted user content.
9. Production and non-production databases, credentials, buckets, auth
   applications, and release decisions remain separate. Sharing dependencies
   inside staging does not authorise production access or deployment.

## Change gate

Before a shared-schema change, record:

- repository, branch, source SHA, and worktree state;
- target project, environment, service IDs, domains, and deployed SHAs;
- exact database resource and migration-ledger state, without recording
  credentials;
- all known readers, writers, workers, and deployment triggers;
- compatibility sequence, backup/restore position, smoke tests, and rollback
  or forward-fix decision.

Stop if the target or dependency graph is ambiguous. Never infer a live target
from a filename, branch name, or local environment variable alone.

## Release discipline

- Keep infrastructure, schema, API-contract, and client changes in focused,
  reviewable commits.
- Validate route existence separately from authenticated behaviour: a protected
  route without credentials should normally return `401`, not route `404`.
- Preserve offline user-scoped caches when a server is unreachable or
  incompatible; block or clearly label online actions that cannot be trusted.
- Do not repurpose another consumer's API service to gain deployment speed.

## Source control discipline

For completed, validated feature work:

1. commit locally;
2. push the authorised working branch promptly;
3. treat deployment as a separate permission boundary; and
4. do not allow large bodies of validated work to remain local-only without an
   explicit reason.

A `do not deploy` instruction does not imply `do not push`. Use a `do not push`
restriction only when a specific review or safety reason requires it. This
rule does not weaken protected-branch, review, environment, or staging controls.
