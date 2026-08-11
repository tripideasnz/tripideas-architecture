# Mobile integration API — developer handoff

Status: integration setup in progress, 11 August 2026

## Why it exists

Mobile development previously used the web developer staging API service. Its
Git-connected deployment lifecycle could replace mobile-capable API code with a
different source revision, making routes such as Notebooks and Personal Places
appear and disappear. Mobile now has a separate deployment boundary while
continuing to use the approved shared staging dependencies.

## Confirmed topology

| Resource | Identity |
| --- | --- |
| Railway project | `Trip Ideas` (`a8521249-69da-494d-a33a-ba84cd8f4ae0`) |
| Environment | `staging` (`b13dc657-d539-4513-845a-2a29c247429d`) |
| Existing web-development API | `api` (`e40b481c-02bf-47d3-bd92-df150853eb78`) |
| Mobile integration API | `tripideas-api-integration` (`5c8c304c-1829-4d88-bb75-f54dd9abeb7b`) |
| Mobile integration URL | `https://tripideas-api-integration-staging.up.railway.app` |
| Shared staging database | Railway Postgres (`6b3be7e3-41ce-4608-b2ce-bb79c2310c9b`), database `railway` |
| Shared private photo storage | Railway bucket `tripideas-photo-assets` |

The integration service is deliberately not connected to a Git branch. Promote
approved local source deliberately with `railway up`; do not attach it to the
web staging deployment trigger. Production remains unchanged.

## API/client contract

`GET /version` returns `apiVersion`, `build`, `environment`, and
`capabilities`. The current mobile contract requires API version `1` and:

- `notebooks`
- `personal-place-cards`
- `photo-assets`
- `mixed-itineraries`
- `trip-api-authority`

Development and preview mobile builds target the integration URL. Production
configuration remains managed by the production EAS environment.

Run the unauthenticated deployment smoke check from `tripideas-api`:

```sh
bun run smoke:mobile-integration -- https://tripideas-api-integration-staging.up.railway.app
```

It verifies readiness and identity, then proves representative protected routes
exist by expecting `401`. Complete signed-in CRUD checks with an approved test
account where practical; never put access tokens in documentation or shell
history.

## Schema and release rule

Follow [Shared database schema and API compatibility discipline](../architecture/shared-database-schema-discipline.md).
The integration service shares the staging migration ledger; it does not own a
second schema. Verify the migration set before deployment, use only checked-in
`prisma migrate deploy`, and use expand/migrate/contract for breaking changes.

## Current operational gate

The blank service and domain exist. Deployment remains paused until explicit
approval grants this persistent service access to the full shared staging
dependency scope (database, WorkOS, private storage, mail, Sanity, sharing, and
Bookit configuration). Use Railway variable references rather than copying
secret values.

If a deployment fails, leave the existing `api`, `web`, database, and production
services untouched. Fix forward or remove only the new blank integration
service after confirming it has no unique data or dependencies.

