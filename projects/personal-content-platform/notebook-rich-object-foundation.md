# Notebook rich-object capture foundation

Status: approved implementation architecture

## Boundary

Notebook remains a private, low-friction capture layer. `UserContentItem` is
the stable identity for independently addressable text, photo, and link source
blocks. Rich metadata extends that record; it does not introduce a second
Note/Object parent. A future Diary is a separate structured artifact that may
retain provenance links to one or more source block IDs.

This milestone does not add Diary entities, place association, route tracking,
web scraping, automatic photo-location assignment, or publication semantics.
Semantic association with a TripIdeas place is distinct from an exact saved
coordinate and must be designed later with Sanity identity and missing-place
behaviour.

## Block contract

Existing `TEXT` and `PHOTO` blocks remain valid. `LINK` adds a required
HTTP/HTTPS URL, optional user title and note, and an idempotent
`clientRequestId`. Capture creates the link from its URL first; annotation is
optional and follows capture.

All block types may optionally carry:

- event metadata with explicit `DATE` or `DATETIME` precision;
- an importance signal named `isImportant`; and
- one exact user-authoritative coordinate with provenance and optional
  accuracy.

Date-only values are calendar dates and are not manufactured midnight
timestamps. Date-time values are instants accompanied by the relevant IANA
time zone. Creation time remains separate.

Initial authoritative location sources are `PIN_NOW` and `MAP_SELECTED`.
`PHOTO_METADATA`, `PLACE_ASSOCIATION`, and `USER_CONFIRMED` are reserved for
future evidence/confirmation workflows and are not accepted current client
write paths. Coordinates and source are present or absent together. Accuracy
is optional and non-negative.

Photo metadata never silently populates block event or location metadata.
Multiple photo locations are not collapsed into one object location.

## Location privacy

> Device location is ephemeral by default. TripIdeas does not transmit,
> persist, associate with an account, or retain a user's device location unless
> a feature has a clear purpose and the user explicitly chooses to save or
> share that location.

Ambient map position is foreground-only, held in memory, omitted from
analytics and API traffic, and discarded with the active UI state. Permission
is requested contextually, never on launch. There is no passive watcher,
background permission, location history, or account association.

`Pin now` is the explicit instruction that converts one foreground fix into
private Notebook content. Manual map selection is also explicit and works
without device-location permission.

Future route tracking must be a separately initiated recording session with
its own LineString/track geometry, timestamps, distance, and duration. It must
not be inferred from Notebook locations or ambient map position.

## Sharing projection

Notebook capability, revocation, viewer-session, and photo-authorisation
semantics remain unchanged. Shared projections include Link URL/title/note and
approved event date/date-time metadata. They exclude exact coordinates,
accuracy, location provenance, and `isImportant` by default.

## Compatibility and rollout

The canonical migration is additive: new metadata is nullable,
`isImportant` defaults to false, and existing Text/Photo rows and cached
snapshots require no user migration. Existing Text/Photo mutation routes remain
compatibility aliases while generic block mutation is introduced.

Follow the shared-schema expand-first sequence:

1. record the shared-environment dependency and migration gate;
2. deploy the additive schema expansion without activating new writers;
3. deploy API readers/writers that tolerate old and new rows and advertise
   explicit rich-block capabilities;
4. activate compatible mobile clients only after capability verification; and
5. retain older DTO fields/routes for the agreed client support window.

An older client must not be handed an unsupported Link block through a legacy
projection. The content projection and capability contract are the rich-block
boundary; legacy Notebook reads remain limited to compatible Text content.
