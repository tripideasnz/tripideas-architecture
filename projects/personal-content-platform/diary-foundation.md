# Diary foundation

Status: approved local product and domain foundation; persistence deferred

## Product boundary

Diary is a user-owned, deliberately composed record of a journey or travel
period. It is not a structured Notebook and it is not a view of a Trip.

- Notebook is low-friction capture.
- Trip is itinerary planning and ordered place selection.
- Personal Place is reusable canonical user-authored place content.
- Diary is an independently editable narrative and presentation.

The hierarchy is `Diary -> Day -> Topic -> ordered Diary item`. A Topic is a
freely named episode; it does not use a closed activity taxonomy. Initial item
types are Narrative, Photo, Link, Editorial Place, Personal Place and explicit
Location. A future Track attaches as another Topic item, but track persistence
and recording are outside this phase.

## Aggregate and eventual persistence

The eventual additive relational design is:

- `Diary`: owner, idempotency key, lifecycle, title, optional introduction,
  cover PhotoAsset, optional date range and optimistic version.
- `DiaryDay`: Diary parent, calendar date, optional heading/summary, dense
  position and lifecycle.
- `DiaryTopic`: Day parent, free-form title, dense position, creation method,
  manual-edit marker, optional version and lifecycle.
- `DiaryTopicItem`: Topic parent, discriminator, dense position, presentation
  fields or canonical target, explicit map inclusion, creation method and
  lifecycle.
- `DiarySource`: selected Notebook or Trip construction source at Diary scope.
- `DiaryItemSource`: small owner-only many-to-many provenance association.

Diary persistence is deliberately absent while staging database recovery is
unresolved. The API Phase B work contains only database-independent contracts
and pure logic. Mobile storage is temporary prototype authority, namespaced by
authenticated user. Canonical authority moves to the API when persistence is
implemented; local data must then be reconciled through an explicit migration,
not silently merged with server content.

## Sources and durable composition

A Diary may have no source, one source or several Notebook/Trip sources. Source
adapters normalize raw DTOs into candidates with stable identity, label,
timestamp evidence, capture/source order, Trip order, location evidence,
content origin and selected presentation metadata. Diary UI must not depend on
raw Notebook or Trip DTOs.

Composition is independent:

- Diary edits never mutate sources.
- Source edits never silently refresh a Diary.
- Source deletion never deletes composed Diary content.
- New source material is proposed or appended; it never silently reshuffles
  existing content.
- Mutable external presentation data is copied only to the minimum necessary
  snapshot for durable rendering. Entire source records are not duplicated.
- User photos continue to reference canonical `PhotoAsset` records. Eventual
  persistence must prevent purge while an active Diary item references an
  asset, following the Personal Place media precedent.

`DiaryItemSource` supports only the needed source kinds: Notebook Block, Trip,
Itinerary Entry, Personal Place Card, Photo Asset and Editorial Place. It keeps
the source ID, optional source version and evidence role. Provenance is private
owner data and is absent from shared projections.

## Content origin and rights

Rights determination is deterministic, never delegated to AI. Item-level
origin is at least `USER_OWNED` or `TRIPIDEAS_SUPPLIED`; a Diary's `MIXED`
classification is derived from its items.

Import location does not determine ownership. User Notebook text, user Trip
notes and user PhotoAssets are user-owned. TripIdeas editorial text/photos are
TripIdeas-supplied. Personal Place components use their recorded provenance and
rights basis.

Three product cases remain distinct:

1. A user-only Diary needs no special TripIdeas content licence beyond ordinary
   service terms. The user retains ownership and independent use rights.
2. Private use and ordinary revocable read-only sharing may include TripIdeas
   material as normal product functionality. No separate acknowledgement is
   required to create, share or view it.
3. Future publication/distribution outside ordinary capability sharing, when
   TripIdeas material is present, requires an explicit personal-use,
   non-commercial acknowledgement for that material. It does not restrict the
   user's own content and it is not implemented in this phase.

Creating a Diary does not require updated Terms acceptance. Creating a share
link is not publication.

## Ordering

Initial import order uses, in sequence:

1. explicit event timestamp;
2. reliable photo/capture timestamp;
3. capture/source order;
4. Trip itinerary order;
5. `createdAt`, source kind and source ID as deterministic fallback.

The result receives dense positions. Persisted positions immediately become
authoritative. Manual reorder validates one exact permutation of active sibling
IDs and writes dense positions atomically. Later material is inserted
deterministically without altering any existing relative order; the default is
append when no explicit insertion point is selected.

## Map projection and privacy

The Diary map is derived only from selected active Diary items. Editorial Place,
Personal Place and Location items may opt in with `includeOnMap`; Narrative,
Photo and Link do not appear merely because their sources contain coordinates.
Future Track items follow the same explicit rule.

Notebook coordinates, photo metadata, Trip source coordinates and ambient
device location never leak into the map without selection into a Diary item.
Ambient device position remains ephemeral. There is no passive tracking or
location history. Track recording will require an explicit start/stop session.

## AI proposal boundary

AI receives only normalized, owner-authorized selected material. It returns a
proposal containing proposed Days, Topics, ordered items, source references,
suggested map inclusion and unsupported/uncertain claims.

The workflow is `Generate -> Review -> Accept selected proposal -> persist
ordinary Diary content`. Generation never mutates a Diary directly. User edits
and positions remain authoritative; regeneration is explicit and scoped to a
chosen Diary, Day or Topic. Unsupported factual events must be surfaced rather
than invented.

## Sharing and publication

Diary will have a separate share authority and shared projection while reusing
the proven capability credential, revocation, rotation and anonymous-session
security mechanisms. A shared projection includes only selected presentation
content and explicitly included map features. It excludes provenance, source
IDs, private evidence, accuracy, editing metadata and unselected content.

The lifecycle is `Private -> read-only capability sharing -> optional future
publication/distribution`. Publication is a distinct future rights and review
workflow, not an advanced share setting.

## Mobile foundation

Diary belongs in the Saved estate and follows the established hub, feature
index, detail/editor, header, card, action, spacing and accessibility canon.
Phase B supports authenticated local drafts, Diary library/create/delete,
overview, Days, freely named Topics, ordered local items and pure map/source
contracts. AI affordances are omitted until functional.

Photo, editorial-place and Personal Place item contracts are present, but a UI
integration may be deferred where it would invade unfinished Notebook or
provider work.

### Day-page refinement

The owner Diary is presented through four distinct top-level views rather than
one combined settings page. Cover presents Diary metadata and cover imagery as
an artifact, with a separate transient editor for title, introduction, date
range and cover selection. Index is only the sparse list of instantiated Days.
Map is the derived spatial presentation. Day displays one date and its Topics.
Diary-level navigation exposes Cover, Index and Map without embedding one
view's editor or content inside another.

The owner experience is one vertically scrollable Day page at a time. Horizontal
page movement changes dates; vertical movement reads the current Day; dragging
reorders Topics or Topic items; tapping opens or captures an object. Reorder
controls must not resemble Day navigation. Drag handles retain accessible
increment/decrement actions for people who cannot perform the gesture.

Diary date input follows New Zealand `d/m/y` order and is normalized to a
calendar-date string without inventing a meaningful midnight instant. Owner
presentation uses compact forms such as `1 Jan 27`.

A date range defines potential Day pages but does not eagerly create Day
records. Entering or adding content to a date may instantiate its Day lazily.
The lightweight Diary index lists only instantiated Days; it never expands the
complete range into index entries. Owner Diaries permit stored empty Days and
manual deletion; deletion removes the stored Day and its index entry, not its
date from the range. A future shared/public projection may omit empty Days.

Each Topic ends with a subordinate object toolbar for Narrative, Photo, Link,
Place and Pin. Link uses a transient finder boundary: a search or URL may be
opened to select a page, but only the confirmed HTTP/HTTPS URL and editable
title persist. It is not a general browser. Pin offers explicit one-fix `Pin
now` and a dedicated `Find on map` coordinate picker; cancellation persists
nothing and ambient location remains ephemeral.

Topic is the editing boundary. Existing Topics normally render as clean,
completed reading content; a newly created Topic opens directly into editing.
Only one Topic editor is normally active on a Day. Done, date navigation and
leaving the screen end that transient presentation state and flush pending
local field saves. Editing/completed is not a persisted Topic lifecycle or
domain status.

Completed Topics omit editor labels, object toolbars, reorder affordances and
removal controls. Their Links present title, domain and optional note; selected
Places and Locations use compact presentation labels without exposing raw
coordinates. In Topic editing, only ordered Topic objects expose an internal
drag affordance. Day and Topic are structural containers and use the canonical
destructive control; removal of an object within a Topic uses a subordinate X.

Topic objects share one editor contract: canonical Narrative, Photo, Link,
Place and Pin labels; one internal reorder grip; one contained-object removal
control; debounced text autosave; immediate selection, removal and reorder
persistence; and no object-level completion action. Narrative supports an
optional title and a 10,000-character body. Place is one presentation type over
either an editorial Place or a Personal Place source. Future multi-photo
selection creates one ordered Photo item per selected asset while presenting
adjacent photos with the established Notebook grid treatment.

The Link Finder remains a capture interface with a provider boundary;
availability of compact web results is not a Diary domain requirement and
browser history is never Diary state.

Diary map presentation is a read-only projection of the Diary's current,
explicitly included spatial items. Each projected feature retains Diary, Day,
Topic and item identity, so deletion of an item or structural ancestor removes
it naturally from the next projection. This overview is separate from the Pin
location-selection map: the picker chooses one coordinate, while the Diary map
shows selected Diary content and never requests ambient device location.
Opening the same Diary Map from a Day fits the initial camera to that Day's
features and visually emphasizes them while retaining the wider Diary features
as context. Opening it from the Cover/menu fits all eligible Diary features.

## Future persistence and rollout

After staging recovery, persistence requires a separate shared-schema gate and
an additive expand-first migration:

1. inventory every shared-database consumer and migration ledger;
2. approve and deploy additive tables without enabling writers;
3. deploy compatible API readers/writers and identity capability;
4. verify authenticated contracts and ownership isolation;
5. migrate/reconcile authenticated prototype data explicitly;
6. enable mobile only after compatibility verification.

No Diary Prisma model, migration, active route, capability advertisement,
deployment or publication workflow belongs to Phase B.
