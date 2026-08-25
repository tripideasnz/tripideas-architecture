# Notebook object contract v2

Status: approved for local architecture and API/schema implementation. Staging and production rollout require separate approval.

## Canonical model

Notebook remains a `UserContentDocument`; Page remains `UserContentPage`; every Page object remains a `UserContentItem`. The canonical object vocabulary is `TEXT`, `PHOTO`, `LINK`, `PLACE`, and `PIN`. Every active object has one page-scoped `position`, including the historical page-body text object.

Historical page-body text retains `isPageBody = true`. That flag is identity and legacy-projection metadata, not a positional constraint. At most one active page-body object exists per Page. Additional TEXT objects are ordinary mixed-sequence content.

PLACE uses exactly one reference variant:

- `EDITORIAL_PLACE` plus `editorialPlaceId`; or
- `PERSONAL_PLACE` plus `personalPlaceCardId`.

The persisted title is a last-known title snapshot. Stored coordinates are a location snapshot, not the source of truth. Duplicate PLACE references within a Page are allowed.

PIN owns its coordinate pair and has no Place reference. Existing PHOTO and LINK persistence remains unchanged.

## Availability and privacy

A Personal Place soft deletion does not remove its Notebook objects. Those objects become unavailable and retain their title snapshot. Unavailable PLACE output must omit its stored location snapshot. The restrictive Personal Place foreign key prevents hard purge while any Notebook reference exists.

Shared Notebook projections redact Personal Places and standalone Pins by default. Redaction omits identifiers, titles, and coordinates. Editorial PLACE objects may expose their last-known title snapshot but never depend on a stored location snapshot for source availability.

## Contracts and compatibility

`X-TripIdeas-Notebook-Contract: objects-v2` opts into the full object vocabulary and mixed ordering. The API advertises `notebook-object-blocks-v2` before a client enables this contract.

`rich-v1` receives only the historical page-body TEXT plus supported PHOTO and LINK objects. The older default projection receives the historical page-body TEXT plus PHOTO. Neither projection receives additional TEXT, PLACE, or PIN objects. The legacy Notebook `items` projection selects `isPageBody = true`, rather than every TEXT item.

All object mutations retain Notebook ownership checks, optimistic version checks, transaction boundaries, private-photo rules, share revocation, and cache behavior. Notebook does not acquire Diary dates/chronology or Topic start-time semantics.

## Migration sequence

1. Apply `20260825120000_add_notebook_object_types` alone. It only adds PostgreSQL enum types/values and must commit before later SQL references the new values.
2. Apply `20260825121000_expand_notebook_object_contract`. It adds object columns, marks the canonical historical TEXT per Page as `isPageBody`, installs the restrictive Personal Place foreign key, adds partial/indexed invariants, and replaces the type-field check constraint.
3. Deploy the capability-gated API only after schema verification.
4. Enable an objects-v2 client only after its version gate observes `notebook-object-blocks-v2`.

Rollback is application-first: disable the capability and v2 writers, retain additive columns and enum values, and continue serving legacy projections. PostgreSQL enum values are not removed during rollback.
