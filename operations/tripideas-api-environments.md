# tripideas-api environment decisions and preflight

## Status

Operational discovery and implementation planning only. Unresolved values must be completed from the authoritative GitHub, Railway, database-provider and WorkOS configuration. Do not infer production facts from repository filenames.

## Confirmed repository facts

| Field | Confirmed value |
| --- | --- |
| Repository | `github.com/tripideasnz/tripideas-api` |
| Default branch | `main` at audit commit `5fed204` |
| Production baseline | `main` |
| Development/integration environment | `staging` at audit commit `29c8e0e` |
| Notebook development target | Staging WorkOS and staging API infrastructure |
| Database interface | PostgreSQL via `DATABASE_URL` |
| Schema and migrations | Prisma schema and checked-in timestamped migrations |
| Runtime queries | Kysely with `pg` |
| Repository pre-deploy command | `bunx prisma migrate deploy` in both Railway JSON files |
| Current container health check | `bunx prisma db push --skip-generate` (schema-mutating; replacement required) |

The product boundary confirms the role of `main` and `staging`. Repository configuration still does not prove the live Railway service IDs, deployed commit SHAs, domains, database resources or whether a public web route can deploy independently.

## Operational decision checklist

Fill every unresolved field from the named authoritative control plane. Record links or immutable identifiers where possible; do not copy credentials into this document.

| Field | Status/value | Manual verification required |
| --- | --- | --- |
| Production Railway service | Unresolved | Railway project → production environment → service name and service ID |
| Staging Railway service | Unresolved | Railway project → staging environment → service name and service ID |
| Production deployment branch and SHA | Expected `main`; live value unresolved | Production service → source/deploy settings; verify deployed commit SHA |
| Staging deployment branch and SHA | Expected `staging`; live value unresolved | Staging service → source/deploy settings; verify deployed commit SHA |
| Production database provider and owner | Unresolved | Railway variables/reference graph and database-provider console; identify legal/operational account owner |
| Staging database provider and owner | Unresolved | Railway variables/reference graph and database-provider console; identify legal/operational account owner |
| Separate credentials and databases | Unresolved | Compare secret identities, database resource IDs, hosts and database names without recording secret values |
| Migration approval and deployment owner | Unresolved | GitHub protections/review rules, Railway deploy permissions and team runbook |
| Backup schedule | Unresolved | Database-provider backup/PITR settings |
| Backup retention | Unresolved | Database-provider backup/PITR settings and contractual plan |
| Backup encryption | Unresolved | Provider encryption documentation and resource settings, including key ownership |
| Restore procedure | Unresolved | Operations runbook; document target environment, validation and rollback steps |
| Restore owner | Unresolved | Named role/team with provider and Railway access |
| Latest restore test | Unresolved | Record date, source backup, isolated restore target, validation result and incident/reference link |
| Account-deletion recovery period | Unresolved | Product/legal/operations decision supported by restore capabilities |
| Permanent-deletion timing | Unresolved | Product/legal/operations decision and deletion-worker operating contract |
| Backup-expiry treatment | Unresolved | Document whether deleted data is left to expire, how restores reapply tombstones, and any legal holds |
| Production web Railway service | Unresolved | Railway production environment → web service name, service ID, source branch and deployed SHA |
| Staging web Railway service | Unresolved | Railway staging environment → web service name, service ID, source branch and deployed SHA |
| Web domain/routing ownership | Unresolved | Railway domains, proxy/routing rules and DNS for production and staging |
| Independent route deployment | Not demonstrated by repository configuration | Determine whether `/notebook/*` can target a distinct service without promoting the full staging web build |

## Manual inspection sequence

1. In GitHub, inspect branch protections, environment rules, deploy integrations and the commit SHA currently reported by each deployment. Preserve `main` as the production baseline and `staging` as the intentional development/integration environment.
2. In Railway, open the actual production and staging environments. Record project, environment and service IDs; source repository and branch; deployed commit SHA; pre-deploy command; health check; domain; and variable references.
3. Follow each environment's `DATABASE_URL` reference without exposing its value. Record the database resource/provider, owner, resource ID, region and whether production and staging are distinct resources with distinct credentials.
4. In the database-provider console, inspect automated backups, point-in-time recovery, retention, encryption/key ownership and restore controls.
5. With the deployment/database owner, trace one checked-in Prisma migration from approval through non-production validation to production deployment.
6. Locate the restore runbook and evidence of the latest isolated restore test. If either is absent, record that absence rather than assuming provider backups are sufficient.
7. Obtain product/legal/operations decisions for recovery, permanent deletion, backup expiry and legal holds. Keep periods configurable until approved.

## API infrastructure preflight plan

This is a proposed file-level plan, not authorisation to implement, merge or deploy.

### Non-mutating health check

- `Dockerfile`
  - Remove `bunx prisma db push --skip-generate` from `HEALTHCHECK`.
  - Point the container health check at a dedicated HTTP readiness endpoint.
- `src/app/index.ts`
  - Register a readiness route such as `GET /health/ready`.
- Proposed `src/app/health/readiness.ts`
  - Execute only `SELECT 1` through the existing database connection.
  - Return a minimal success/failure response without schema introspection, migration, DDL or credentials.
- `src/main.ts`
  - Reuse one application database pool rather than introducing a second health-check connection model.
- Proposed `src/app/health/readiness.test.ts`
  - Prove success and database-unavailable failure.
  - Spy/fake the database adapter and prove no Prisma push/migrate or schema-changing statement is invoked.

### WorkOS bearer verification

- `src/config/config.ts`
  - Add required expected WorkOS issuer and audience configuration, sourced separately per environment.
- `src/lib/middleware/auth.ts`
- `src/lib/middleware/mobile-auth.ts`
- `src/lib/middleware/mobile-or-web-auth.ts`
  - Consolidate duplicated bearer verification behind one verifier.
  - Require exact issuer and audience in addition to JWKS signature and time-claim validation.
  - Accept only a correctly formed Bearer scheme and require a subject before resolving `User.authId`.
- Proposed `src/lib/auth/verify-mobile-access-token.ts`
  - Own JWT parsing and WorkOS verification policy so every authenticated route uses the same checks.
- Proposed `src/lib/auth/verify-mobile-access-token.test.ts`
  - Cover a valid token, wrong issuer, wrong audience, expired token, missing token and malformed authorization scheme.
- Proposed `src/lib/middleware/mobile-or-web-auth.test.ts`
  - Confirm a valid WorkOS subject resolves through `User.authId` to internal `User.id`, while invalid tokens never query protected data.

Expected issuer and audience values must be read from the authoritative WorkOS application/environment configuration and verified against current WorkOS documentation. They must not be guessed from client IDs or URLs.

### Sensitive authentication logging

- `src/lib/middleware/mobile-or-web-auth.ts`
  - Remove temporary `console.log` calls and all logs containing WorkOS IDs, internal user IDs or email addresses.
- `src/lib/middleware/auth.ts`
- `src/app/auth/delete.ts`
- `src/app/user/delete.ts`
  - Review authentication/deletion logs and retain only non-sensitive event names and correlation IDs.
- `src/lib/middleware/logger.ts`
  - Ensure request logging does not record authorization headers, cookies, tokens or sensitive query parameters.

### Production and staging release boundary

1. Treat `main` as the current production baseline and `staging` as the intentional development/integration environment.
2. Record the API and web Railway services' configured branches, deployed commit SHAs, domains and database references.
3. Develop Notebook against staging WorkOS and staging API infrastructure.
4. Inventory Notebook changes separately from the existing web staging release programme.
5. Do not merge, promote, retarget a Railway service or otherwise use Notebook work to pre-empt the current web release process.
6. Validate Notebook infrastructure and migrations only in the confirmed staging environment until a separate production rollout is approved.
7. Treat Notebook production rollout and broader web staging promotion as independent release decisions.

## Public shared-page deployment boundary

### Confirmed from repositories

- `tripideas-web` is a Next.js application built as one standalone Docker image.
- Its Railway JSON files describe whole-service Docker builds and do not define route-level or share-page services.
- The existing public Trip Idea route, `src/app/(share)/trip/[shareId]/page.tsx`, is present on the production `main` baseline. The current `staging` branch deletes it and its supporting `src/lib/public-trip.ts` and `src/app/api/trips/share/route.ts` files relative to `main`.
- A future public Notebook page would require only read-only capability-link rendering. No web Notebook authoring, management, account controls or navigation are allowed in Phase 1.
- Repository evidence does not establish that either public page can be deployed independently from the full web service.

### Live Railway checks required

1. Record production and staging web service IDs, source branches and deployed SHAs.
2. Record all attached domains and any path-based proxy, gateway or edge routing rules.
3. Confirm whether Railway or the current DNS/proxy can route `/notebook/*` to a distinct service while all other production paths remain on the current production web service.
4. Confirm whether the existing public Trip Idea route is currently served by the production web service and which commit supplies it.
5. Confirm whether a service can deploy a restricted viewer artifact independently without changing the production web service branch.

### Options to report, not select

| Option | Deployment characteristic | Principal risks |
| --- | --- | --- |
| Add the route to `tripideas-web` | Safe only if it can ship without promoting unrelated staging work, or after the existing web release reaches production | Current repository build deploys the whole app; a route-only release is not demonstrated |
| Serve HTML from `tripideas-api` | Could follow the independently released API path and keep authoring in mobile | Mixes presentation into the API, requires safe HTML/rendering and static-asset handling, and couples public traffic to the authenticated API service |
| Small separate public viewer | Independent build and release boundary; can receive only `/notebook/*` traffic | New service, DNS/routing, operations, monitoring and security surface |
| Retain the route in staging | Avoids production web interference until the existing release completes | Public sharing cannot launch to production until promotion or another viewer option is approved |

Do not choose or implement an option until the live Railway service and routing layout is inspected. The follow-up report should identify the smallest option that is demonstrably deployment-safe.

## Notebook account-deletion integration

Milestone 1 should integrate Notebook records without redesigning the entire account-deletion system.

- Notebook ownership references internal `User.id`.
- The schema supports configurable states: `active`, `deletion_requested`, recoverable soft-deleted, and permanently deleted.
- Owner reads and writes exclude any state that should have immediate access invalidation.
- An account-deletion request transitions owned notebooks out of active access and records deletion intent; permanent physical deletion remains a separate controlled operation.
- Exact recovery and retention periods remain configuration/operations decisions.

Separate account-level work that must be tracked:

| Concern | Current finding / future requirement |
| --- | --- |
| Itinerary deletion blocker | Account deletion leaves itinerary deletion commented out while restrictive foreign keys may prevent deleting `User`. Resolve independently of the Notebook migration. |
| Mixed cascades/manual deletion | Existing relations combine database cascades with ordered manual deletes. Define and test one explicit account-deletion orchestration contract. |
| Database versus WorkOS ordering | Local deletion currently precedes WorkOS deletion, permitting partial completion. Design an idempotent state/job workflow with retry and immediate access denial. |
| Deletion status/job tracking | Add durable status, attempts, timestamps and non-sensitive failure information in dedicated account-deletion work. |
| Photo assets | Once photos exist, enqueue unreferenced assets for asynchronous cleanup after the approved recovery period. Outside Milestone 1. |
| Public links | Once sharing exists, invalidate capability links immediately when notebook/account deletion is requested. Outside Milestone 1, but required by its sharing design. |
| Backups | Let deleted data expire under the documented policy and ensure a restore reapplies deletion tombstones before restored data can become accessible. |

## Staging database collation maintenance position

- Local Notebook schema and API development may proceed against a disposable local PostgreSQL database.
- The staging collation maintenance gate was completed on 26 July 2026. A
  Notebook migration may be applied to staging only through the separately
  approved migration procedure; this record does not authorize production
  deployment.
- The staging database is confirmed separate from production. Its private database reference is the approved maintenance target.
- The staging API uses the staging environment's private Postgres service
  through `DATABASE_URL` and has no `DATABASE_PUBLIC_URL`. The Postgres service
  exposes a Railway-generated public proxy variable, but it is quarantined from
  application, migration and maintenance use. The 26 July maintenance used the
  already-existing staging TCP proxy selected explicitly by Railway project,
  environment and service, with separately injected Postgres credentials; it
  did not use `DATABASE_PUBLIC_URL`.
- Create a fresh staging backup immediately before collation maintenance. An isolated restore test is preferred; otherwise the accountable owner must explicitly accept the recovery risk.
- The preferred maintenance path is to preflight affected unique values, run `REINDEX DATABASE CONCURRENTLY railway`, verify indexes, constraints and representative application reads and writes, and only then run `ALTER DATABASE railway REFRESH COLLATION VERSION`.
- If the rebuild or verification fails, do not refresh the recorded collation version.

The staging Notebook migration gate requires:

1. Correct or quarantine the staging `DATABASE_PUBLIC_URL`.
2. Assign a maintenance owner.
3. Create a fresh backup.
4. Verify restore capability or explicitly accept the recovery risk.
5. Rebuild collation-dependent indexes.
6. Verify index validity, constraints and application behaviour.
7. Refresh the database collation version.
8. Confirm the recorded and actual versions match with no warning.
9. Confirm migration approval and deployment ownership.

### Completed staging maintenance — 26 July 2026

| Field | Recorded result |
| --- | --- |
| Environment boundary | Railway `staging`; production is a distinct environment and was not accessed |
| Database target | Staging Postgres database `railway`, PostgreSQL 16.14 |
| Application database reference | Staging API private `DATABASE_URL`; no staging API `DATABASE_PUBLIC_URL` |
| Owner | Douglas Paul, deployment, migration and database-maintenance owner |
| Backup | Fresh manual staging database backup created 26 July 2026 at 02:12 UTC |
| Restore position | Railway restore action confirmed; no isolated restore test performed; owner explicitly accepted the limited staging recovery risk |
| Preflight size | 8,600,599 bytes (`8399 kB`, approximately 8.4 MB) |
| Preflight activity | No transaction older than five minutes; no waiting locks; one active connection (the maintenance session); no idle-in-transaction connection |
| Preflight collation | Recorded `2.36`; operating-system/actual `2.41`; mismatch warning present |
| Affected indexes | 23 collatable-column indexes; all valid and ready before maintenance |
| Constraint preflight | Zero unvalidated constraints |
| Duplicate preflight | Zero duplicate groups for `User_email_key` and all eligible single-column unique text indexes checked |
| Rebuild command | `REINDEX DATABASE CONCURRENTLY railway;` completed successfully in approximately 1.225 seconds |
| Post-rebuild index result | Zero invalid or not-ready indexes; zero invalid unique indexes |
| Post-rebuild constraint result | Zero unvalidated constraints |
| Representative database checks | Reads succeeded for User, Favourite and Itinerary; rollback-only writes affected one User and one Favourite row; no Itinerary row existed; no changes were committed |
| API readiness | `GET /health/ready` returned `200` with `{"status":"ready"}` after maintenance |
| Available API regression checks | `/auth/identity` returned `200`; protected `/favourite/by-ids` returned `401` without a bearer token, confirming the route and authentication gate were active; no test bearer credential was used, so authenticated favourites mutation was not exercised |
| Refresh command | `ALTER DATABASE railway REFRESH COLLATION VERSION;` completed successfully after rebuild verification |
| Final collation | Recorded `2.41`; actual `2.41`; a fresh connection emitted no collation warning |
| Final validity | Zero invalid/not-ready indexes and zero unvalidated constraints |
| Migration procedure | Owner-approved staging procedure is checked-in Prisma migrations applied with `prisma migrate deploy`; never `prisma db push` |
| Notebook staging gate | Collation, backup/risk, ownership and staging migration-control gate satisfied; migration 12 remains unapplied and requires a separate staging deployment task |

No Railway branch, service, credential or production setting was changed during
this maintenance. No application deployment or Prisma migration was performed.

### Completed staging Page migration — 27 July 2026

| Field | Recorded result |
| --- | --- |
| Environment boundary | Railway `staging`; production remained separate and was not accessed or changed |
| Database path | Staging API private `DATABASE_URL`; no staging API `DATABASE_PUBLIC_URL`; no public database URL was used |
| Approved migration | `20260727120000_add_user_content_pages` |
| Approved/deployed API | `0dd09f95bd3d24011022e908289298b013a8341f` |
| Backup | Fresh manual staging Postgres backup confirmed by the owner at 23:04 NZST on 27 July 2026 (11:04 UTC) |
| Restore position | Railway restore remained available; restore the fresh pre-migration backup before reconnecting the old API if a forward repair could not recover a failed new deployment |
| Maintenance boundary | The old staging API deployment was stopped before migration; its service domain returned `404` while quiesced, so it could not accept writes |
| Preflight | Database `railway`, `8623 kB`; zero transactions older than one minute; zero waiting locks; zero invalid/not-ready indexes; zero unvalidated constraints |
| Preflight data | 19 legacy items: 14 active and 5 soft-deleted |
| Preflight collation | Recorded `2.41`; actual `2.41` |
| Migration command | Checked-in Railway pre-deploy command `bunx prisma migrate deploy`, executed inside Railway against `postgres.railway.internal`; `prisma db push` was not used |
| Migration time/result | Finished 27 July 2026 at 11:19:56 UTC; Prisma recorded the migration as finished with no rollback and reported all migrations successfully applied |
| Page/item mapping | 19 Pages for 19 legacy items; 14 active and 5 soft-deleted on each side; zero missing Pages, wrong deterministic `ucp_` IDs, wrong Page links or wrong document links |
| Ordering | Page positions preserve former document-level item positions; all migrated block positions are `0`; zero duplicate active Page positions and zero duplicate active block positions within a Page |
| Data preservation | Titles, Text content, item IDs, timestamps and deletion state preserved; the pre/post preservation digest matched (`4430fed87ba39ef2328e361b6e6f6f48`) |
| Schema validity | `UserContentPage` and both active-position unique indexes exist; zero invalid/not-ready indexes and zero unvalidated constraints |
| Post-migration size | `8711 kB` |
| Final collation | Recorded `2.41`; actual `2.41`; deployment logs contained no collation warning |
| Deployment | Railway deployment of the clean approved source completed successfully; root and `GET /health/ready` returned `200` |
| Authentication boundary | Unauthenticated Notebook, favourites and itinerary requests returned generic `401`; the existing signed-in mobile session refreshed through `POST /auth/mobile/refresh` with `200` and then read a migrated Notebook detail with `200` |
| Legacy DTO/mobile compatibility | Existing mobile Notebook list and detail rendered migrated Pages and Text content without cache parse errors; the legacy flat `items` DTO is retained by the deployed contract and its route/DTO tests passed |
| Favourites and Trip Ideas | The signed-in Saved screen rendered an existing Trip Idea and existing favourites after deployment; no authenticated add/remove mutation was performed |
| Automated contract regression | Focused Notebook service, DTO, route and mobile-bearer suites: 45 passed, 0 failed; covered create/list/read, metadata update, Text create/update/delete, reorder, stale-version conflict, ownership-safe 404, deletion and legacy contract serialization |
| Interactive mutation limitation | Simulator deep-link navigation and signed-in reads were verified. Automated screen interaction was unavailable because the macOS runner lacked assistive access, so live staging create/edit/delete/reorder/conflict and favourites add/remove were not claimed as executed |
| Rollback position | Migration succeeded and the matching API is healthy. Do not run the old API against the migrated schema. Prefer a forward fix; if that is not viable, stop staging, restore the 23:04 NZST backup, then reconnect the old API |
| Gate status | Page migration and matching API deployment completed successfully. Final product acceptance remains conditional on the short interactive signed-in mutation regression noted above |

## Exit criteria before the Notebook migration

- Actual production and staging API/web service, branch and commit mappings are recorded.
- PostgreSQL provider, ownership and environment isolation are confirmed.
- Migration approval/deployment ownership is recorded.
- Backup, encryption, restore ownership and latest restore-test evidence are recorded, or their absence is accepted as an explicit blocker.
- The staging database collation-version mismatch is assessed against the current server runtime.
- Indexes and other stored objects affected by a changed collation are rebuilt where necessary.
- The recorded database collation version is refreshed only after all affected objects have been rebuilt and verified.
- A usable backup is confirmed before collation maintenance, or the accountable owner explicitly accepts the recovery risk.
- Recovery/permanent-deletion/backup-expiry decisions have accountable owners; exact periods may remain configurable if explicitly approved.
- The non-mutating health-check plan is approved.
- WorkOS issuer/audience values and verification approach are confirmed.
- Sensitive authentication logging removal is approved.
- The production/staging release boundary is recorded and Notebook work is isolated from the current web release programme.
- Public shared-page deployment independence and routing are confirmed, or the unresolved viewer decision is explicitly deferred until Milestone 6.
- Notebook account-deletion integration boundaries are approved.
