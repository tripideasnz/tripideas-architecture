# Geographic platform: identity, coordinate QA, relationships, access and Nearby Places

## Status

Proposed — 2026-09-29. Governs all future geographic processing work. Supersedes
the transitional Milestone A.1 / ORS Matrix tooling as the target design; it
does not retroactively invalidate data that tooling produced.

## Context

Nearby Places, coordinate QA, the relationship graph and road-distance
calculation have grown as specialist scripts in `tripideas-web`, operated
per-place and through Sanity Studio fields. A repository and data audit
(2026-09-29, read-only) established:

- **Coordinates and classification live only in Sanity.** `page.coordinates`
  (geopoint) and `page.placeType` (the single canonical classification enum:
  `supercluster`, `cluster`, `hub`, `road_hub`, `river_hub`, `track_network`,
  `place`). 1,117 of 1,584 Place pages have no `placeType`; 9 have no
  coordinates. Accommodation carries `coordinates` and `isBookitManaged`.
- **Confirmed structural relationships currently live inside Sanity
  `nearbyPlaces`.** About 1,519 approved, human-reviewed entries carry
  structural buckets (`cluster_member`, `connected_place`, `article_reference`,
  `road_hub`, `connected_track`, `hub_connection`, `route_network_member`,
  `access_hub`, `hub_member`, `manual`, `departure_point`). The published Nearby
  output is therefore acting as the relationship authority.
- **`place_relationship` (web-era Neon database) is predominantly a
  candidate-processing log.** 51,901 of 52,364 rows are `distance` candidate
  rows from Nearby generation runs (40,732 rejected); roughly 463 describe
  genuine topology. It mixes topology with run state (`proposed_status`,
  `in_pool`, `in_nearby_places`, `source_run_id`).
- **The distance/access graph is `place_access` in the same web-era database**
  (35,084 rows): directional, per travel mode, with endpoint coordinate
  fingerprints (`coordHash`, SHA-256 of `lat,lng` to 6 dp), routing provider
  provenance (`openroute`, `ors_matrix`, `manual`, `estimated`) and status. It
  has no migration ledger and is used only by `tripideas-web` scripts.
- **Topology partly lives in code:** island access and ferry records
  (`island-config.ts`) and departure points. A gateway-inheritance mechanism
  (`access-clusters.ts`, distance source `route_inherited`) exists but no
  published Nearby entry uses it.
- **Road-routing credibility has been treated as coordinate validity** in the
  A.1 coordinate audit (`routing_unsupported`), the Matrix recheck
  classification and the first road-routing review UI.
- **`tripideas-api` has no geographic model**, and its production/staging
  database resources are recorded as unresolved in
  `operations/tripideas-api-environments.md`.

The operator (Douglas) needs two simple, on-demand tools outside Sanity Studio
rather than a collection of per-place scripts.

## Decision

TripIdeas separates three durable geographic concepts and one derived output,
and operates them through two user-initiated tools.

| Concept | Question | Authority |
|---|---|---|
| Coordinate QA | Is this editorial location credible? | Geographic platform |
| Structural relationship graph | How are these Places structurally connected? | Geographic platform |
| Distance/access graph | What calculated distance/access evidence exists between these Places? | Geographic platform |
| Nearby Places (derived) | What should we recommend near this Place? | Proposal and approval in the geographic platform; published projection in Sanity |

The permanent geographic processing platform is implemented in `tripideas-api`
on the shared API Postgres with PostGIS, under the shared-schema migration
discipline. `tripideas-api` owns the geographic *service*; it does **not**
become the editorial authority for Place or Accommodation documents.

The two tools, exposed through the TripIdeas tool library:

- **Pin Review** — *Find and review missing or anomalous map locations.*
  Optional, on-demand coordinate QA.
- **Place Processing** — *Classify Places, build relationships and distances,
  and prepare Nearby Places.* On-demand processing after batches of new Places
  or Accommodation, and for later stale or incomplete geographic work.

## Authority boundaries

**Sanity remains the authority for:**

- editorial content and published Place/Accommodation documents;
- initial editorial coordinate entry;
- the published coordinate projection;
- publication-facing classification (`placeType` is read by public Nearby
  rendering);
- the published Nearby Places projection;
- other publication-facing fields required by web and mobile.

**The geographic platform becomes the authority for:**

- geographic processing identity (GeoEntity);
- coordinate fingerprint and provenance;
- coordinate verification;
- structural classification used by geographic processing;
- the structural relationship graph;
- the distance/access graph;
- processing and review state;
- Nearby proposal and approval state.

A GeoEntity *references* an editorial Sanity document where one exists. It
never takes ownership of that document.

## Geographic identity

- Every geographic subject has an internal, stable **GeoEntity UUID**.
- A GeoEntity carries an optional, unique **editorial reference** — today the
  Sanity document `_id` of a Place (`page`) or Accommodation
  (`accommodation-bookit-*`).
- The Sanity `_id` is never the only geographic identity. A GeoEntity may exist
  before publication (for example from a future metadata import) and gains its
  editorial reference on publication, keeping the same UUID throughout:
  metadata/import → coordinate verification → Sanity publication → geographic
  enrichment.
- Existing public, mobile and API editorial identifiers (Sanity `_id` used as
  `placeId` / `editorialPlaceId`) are unchanged by this decision.
- Private user-generated Personal Place Cards are out of scope.

## Coordinate lifecycle and verification

- **Fingerprint.** The coordinate fingerprint is the existing `coordHash`
  (SHA-256 prefix of `lat.toFixed(6),lng.toFixed(6)`). It is the key for
  verification and the staleness signal for all dependent evidence.
- **Two verification states only:** *unverified*, or *human verified at
  fingerprint F*. There is no permanent "rejected coordinate" state.
- **Provenance** records how the current coordinate arrived: Studio entry,
  Pin Review move, or import.
- **Single coordinate writer.** For a published entity, the Sanity coordinate
  field is the only mutable coordinate. The platform records, fingerprints and
  verifies it, and never edits a coordinate independently of it.
  - *Studio entry or edit:* the editor writes Sanity; ingestion detects the new
    fingerprint and records it as unverified with provenance `studio`.
  - *Pin Review move:* the tool performs one controlled coordinate write to
    Sanity (the existing controlled-write approach). Only after that write
    succeeds does the platform record the new coordinate, its fingerprint,
    provenance `pin_review` and a verification by the reviewer. Ingestion then
    sees a matching fingerprint and makes no further change.
  - *Pre-publication entities* hold their coordinate in the platform until
    publication, when it is written to Sanity once.
- **Detecting drift.** Any mismatch between the Sanity coordinate and the
  platform's current fingerprint is resolved by ingesting Sanity's value as a
  new unverified coordinate. Two competing writers therefore cannot silently
  diverge.

## Structural relationship model

The structural graph stores **genuine structural relationships only**, using
the existing TripIdeas vocabulary. No parallel vocabulary (for example
`accessed_via`) is introduced.

**Classification (per entity):** the existing `placeType` enum. For processing
purposes an unset value means *awaiting classification*, not `place`.

**Relationship types:** the existing structural selection buckets —
`cluster_member`, `hub_member`, `hub_connection`, `road_hub`,
`connected_track`, `connected_place`, `access_hub`, `article_reference`
(supercluster membership evidence), `route_network_member`,
`departure_point`, `manual`. Each relationship records its type, the evidence
signal, and its status (proposed / confirmed / rejected) with the deciding
human.

**Connected tracks (existing model, preserved):**

- Evidence comes from article text through `src/lib/nearby/signals.ts`:
  - `linked_track`: an inline link from source to candidate, plus
    connection-implying language, plus confirmed reverse evidence;
  - `connected_track`: a bidirectional text reference with connection
    language on at least one side;
  - `article_match` and plain `distance` are **not** connections. List or
    context mentions ("also nearby…") never qualify.
- Only the strong signals (`linked_track`, `connected_track`) count as real
  connections. Candidates are confirmed by a human (currently via reviewed
  edge-list plans, `write-track-connections.ts`).
- A confirmed connection is **bidirectional**. It is recorded as
  `connected_track` when the referenced page is a track, otherwise
  `connected_place`. `access_hub` records a track's connection to a hub page.
- Track networks and river corridors are expressed through these
  relationships. For example, a track deep inside a park that reaches a road
  only through another named track is fully described by its connected-track
  edges.
- A connected-track relationship is **not** evidence that the editorial pin is
  wrong, and is **not** a reason to move the pin to a road.

**Islands and departures:** island access, ferry crossings and
`departure_point` relationships move from code configuration
(`island-config.ts`) into confirmed relationships and access records.

**Road connection (future, optional):** a track network may later carry a
simple evidence fact such as *road connection exists* (normally true at a
network's start/end tracks; false where access is boat-only). It is not used
by road-distance calculation or Nearby ranking, and needs no further design
now.

**Distance candidates are not relationships.** Candidate generation for
distance enrichment is derived, transient processing state. It is never stored
as topology.

## Distance/access model

- The access graph stores **actual, defensible calculated evidence**: one row
  per directed pair and travel mode, with endpoint fingerprints at calculation
  time, route distance and duration, provider provenance (`ors_matrix`,
  `openroute`, `manual`, `estimated`), status, and — for road evidence — the
  ORS snap distance and snapped point of each endpoint.
- **All useful calculated pairs are retained**, regardless of whether they are
  selected into Nearby Places. Nearby is one consumer of the graph.
- **Valid calculated evidence:** `route_status IN ('ok','manual','needs_review')
  AND route_km IS NOT NULL`.
- **Road-evidence credibility policy**
  (`DEFAULT_ROAD_EVIDENCE_POLICY.maxCredibleSnapM = 100`). A Matrix road result
  is credible only when the distance is > 0, both endpoints snap within 100 m,
  and the endpoints snap to different road points. Otherwise the pair is
  recorded as road-routing review evidence with no road distance.
  - It **is** a road-routing credibility threshold, and potentially an anomaly
    signal when Pin Review is deliberately run.
  - It **is not** a universal coordinate-validity rule, a requirement that
    every Place be within 100 m of a road, or evidence that a track or network
    Place should have its pin moved. Existing connected-track or network
    topology may fully explain a >100 m road snap.
- **No manufactured distances.** A road distance is never constructed by
  combining a route to a gateway with a structural relationship or any other
  inferred segment. The unused `route_inherited` / gateway mechanism is
  retired. When ORS cannot produce credible road evidence directly — for
  example because an endpoint lies inside a connected-track network — that is
  acceptable: structural relationship ≠ requirement for road distance, and
  absence of road distance ≠ invalid structural relationship.
- **No automatic second routing service.** Non-credible Matrix evidence goes to
  review; it is not automatically re-run through ORS Directions.

## Nearby derivation

Nearby Places is derived from relationships and applicable distance evidence,
in this order:

1. **Structural relationships first.** Human `manual` entries and
   `departure_point` always rank first, then the structural buckets ranked by
   source page type (the existing `bucketRank`: supercluster, route network,
   hub, track, place).
2. **Applicable distance evidence second.** Distance ranks candidates and
   identifies additional Nearby Places that Stage 1 did not find. For a pair
   A→B: use valid calculated A→B evidence; else valid calculated B→A evidence;
   else crow-flight. Never average directions; when both exist, source →
   candidate wins even if the reverse is shorter.
3. **Remaining proximity candidates** within the existing pool rules.

Structural relationships do not require road-distance evidence to be included
or ranked.

The result is stored in the platform as a **Nearby proposal** recording the
inputs it used (relationship and access evidence fingerprints). A human
approval turns it into an approved result, which is then published to Sanity.
Source page type is to derive from confirmed classification rather than the
current title heuristic (`isTrackTitle`).

## Pin Review workflow

Pin Review is **optional and user-initiated**. It never runs automatically in
the background, and nothing forces a new Place through it. Place Processing is
not blocked because Pin Review has not been run.

1. **Scan.** When invoked, it scans the current catalogue.
2. **Surface signals.** These are review signals, not conclusions:
   - missing pin;
   - obviously unreasonable coordinate (malformed, outside New Zealand,
     latitude/longitude swapped, shared exactly with a differently titled
     entity);
   - no usable road-routing evidence where road access would normally be
     expected;
   - excessive road snap (beyond the road-evidence policy);
   - distinct entities snapping to the same road point;
   - other strong inconsistencies.
3. **Suppress.** Two kinds of case are not shown:
   - cases already human-verified at the **same fingerprint**;
   - cases whose apparent anomaly is adequately explained by **confirmed
     topology** (connected-track or network membership, island access,
     departure relationships, confirmed non-road features).

   Re-review occurs only when the fingerprint changes or new, stronger evidence
   appears.
4. **Inspect.** A place-centric list sorted by impact, with a map showing the
   editorial pin, ORS snap point(s), the pin→snap line and counterpart
   evidence.
5. **Decide:**
   - **Confirm** — record that the current fingerprint was human-checked. The
     same unchanged soft anomaly is not surfaced again.
   - **Move** — drag the pin, confirm explicitly, and update the coordinate
     through the controlled coordinate-write path. The new fingerprint makes
     dependent distance evidence stale. No ORS call occurs.

Places and Accommodation are reviewed together. Accommodation is normally
road-accessible, but a correctly placed large property may still be confirmed.

## Place Processing workflow

Place Processing presents every entity with outstanding geographic work
**together in one batch view**, instead of per-document Studio work. It is
deterministic, incremental and resumable. Re-running it processes state gaps,
not completed current work.

**Stage 1 — classification and topology**

1. Discover entities with gaps (new, unclassified, stale or incomplete).
2. Propose classification from deterministic evidence; a human confirms or
   amends it.
3. Propose structural relationships using the existing vocabulary and signals:
   - cluster, hub and supercluster membership;
   - road/road-hub relationships;
   - track/track-network and river-network relationships;
   - connected Places;
   - island/departure relationships.

   A human confirms them.
4. Structural relationships establish the first Nearby candidates and ranking.

**Stage 2 — distance/access enrichment**

1. Identify appropriate candidate pairs (existing pool rules: Places within
   the configured radius; Accommodation within 30 km).
2. Reuse current distance evidence (fingerprints match).
3. Determine the missing applicable pairs. Structural pairs are not routed
   merely because they exist.
4. Explicitly calculate missing road distances through a single, quota-gated
   action using the existing Matrix engine.
5. Retain all useful calculated pairs.
6. Use applicable distances to rank candidates and fill Nearby gaps.
7. Produce a Nearby proposal.
8. Review it.
9. After explicit approval, publish the approved Nearby result to Sanity
   through the existing controlled-write approach.

**Accommodation** uses the same batch view with its own proximity and selection
profile (the existing 30 km candidate rule). Accommodation Nearby selection
rules are a product decision still to be made.

## Stale and recalculation model

- **Fingerprint mismatch is the fundamental stale signal.** Access evidence
  whose stored endpoint fingerprint differs from the current fingerprint is
  stale, row by row.
- A pin change therefore makes the dependent access evidence stale, makes those
  pairs eligible for targeted recalculation, and may make dependent Nearby
  output stale (its recorded inputs no longer match).
- **No ORS call occurs automatically.** Stale work appears as deterministic
  gaps in Place Processing, which supports both batch recalculation and
  targeted recalculation for a single Place.
- Processing state is **derived from gaps**, not from creation dates.
  - Entity states: not yet registered; location missing; awaiting
    classification; relationships incomplete; distances incomplete or stale;
    road-routing evidence awaiting review; Nearby proposal awaiting review;
    current.
  - Coordinate verification (unverified / verified at fingerprint) is tracked
    alongside these states, not as one of them.
  - Only facts that cannot be derived from data presence are recorded
    explicitly: classification decisions, completed relationship runs (with
    their inputs hash) and Nearby decisions.

## Sanity projection model

| Sanity field | Future role |
|---|---|
| `coordinates` | Editorial entry and published projection (single-writer rule) |
| `placeType` | Published projection of confirmed classification (read by public Nearby rendering) |
| `nearbyPlaces[]`: `place`, `distanceKm`, `distanceSource`, `sortOrder`, `status` | Published projection of the approved Nearby result (public rendering filters on `status`) |
| `nearbyPlaces[]`: `source`, `selectionBucket`, `reason`, `flags`, `reviewedByHuman` | Workflow state; authority moves to the platform; deprecated once the tools are authoritative |
| `nearbyPlacesProposedChanges`, `nearbyPlacesGeneratedAt`, `nearbyPlacesReviewedAt` | Workflow state; move to the platform, then deprecate |
| `isBookitManaged` | Unchanged (Bookit synchronisation) |
| `islands` singleton | Editorial content unchanged; access topology moves to the relationship graph |

Nothing is removed from Sanity until the tools are authoritative and every
consumer has been verified.

## Migration and transition strategy

- The web-era `place_access` database may remain temporarily behind a
  repository/service interface. The new tools depend on that interface, not on
  the database's location.
- **Structural relationships** are seeded from:
  - confirmed structural entries in Sanity `nearbyPlaces` (primary source);
  - the structural rows of `place_relationship`;
  - island and departure configuration currently in code.
- **Candidate-log rows** (`distance` type in `place_relationship`) are archived
  as a read-only historical export. They are not migrated into the graph.
- **`place_access`** migrates with its fingerprints unchanged (they are
  portable). Snap evidence is backfilled from the Matrix journal where
  available.
- **Transitional A.1 artifacts** (local journal, recheck store, review
  artifacts, `corrections.json`) are imported as evidence where useful, then
  retired.
- The remaining Matrix workload is **not** run under A.1 tooling. Unprocessed
  and stale pairs wait for Place Processing unless separately authorised.

**Reuse:**

- coordinate fingerprint (`coordHash`) and the road-evidence policy;
- directional resolver;
- Nearby candidate selector and `bucketRank`;
- text signals and connected-track/network logic;
- supercluster pool;
- island/departure logic (as data);
- Matrix planner, response mapping, journal/resume and quota gates;
- the Nearby review screen and controlled Sanity write path;
- MapLibre admin map patterns.

**Transitional, then retired:**

- A.1-specific scripts: the pilot, own-pin recheck, reconciliation and
  missing-pair samplers, pipeline classifier, handoff conversion;
- the A.1 coordinate-audit and road-routing review screens (superseded by Pin
  Review);
- one-off write/fix/seed scripts;
- the ORS Directions routing pass;
- the per-place generation scripts, once Place Processing is authoritative.

## Consequences

- Nearby Places stops being the relationship authority. Its Sanity data becomes
  a projection, which removes the current circularity (relationships stored
  inside the output derived from them).
- Coordinate correctness, topology and road credibility are separate. Track
  and network Places inside parks are no longer pushed toward roads.
- Some structural pairs will legitimately have no road distance; Nearby still
  ranks them structurally.
- A single operator workflow replaces about 50 specialist scripts and
  per-document Studio editing.
- A new platform schema and service must be built and operated in
  `tripideas-api`; the web-era database is carried for a transition period.
- ORS quota (observed 50 requests per rolling window; operated at 45) bounds
  distance throughput for large batches.

## Implementation sequence

1. Resolve the blocking decisions below; accept this ADR.
2. Design the foundation schema in `tripideas-api` (Prisma + PostGIS,
   shared-schema discipline): GeoEntity, coordinate verification,
   classification, structural relationship, access evidence, Nearby
   proposal/decision, processing records.
3. Add a repository/service interface over the web-era access graph for the
   transition.
4. Build Pin Review.
5. Build Place Processing Stage 1 (classification and topology; seed
   relationships from Sanity and the relationship table).
6. Build Place Processing Stage 2 (Matrix engine; Nearby proposal, review and
   publication).
7. Migrate the access graph; archive the candidate log.
8. Retire A.1 tooling; deprecate Sanity workflow fields.

## Rejected alternatives

- **Treating every >100 m ORS snap as a bad coordinate.** Road credibility is
  not coordinate validity; correctly placed track, park, island and water
  locations are legitimately far from the driving graph.
- **Requiring Pin Review for every new Place.** Review is optional QA and must
  not block Place Processing.
- **Running Pin Review automatically in the background.** It is invoked
  deliberately by the operator.
- **A permanent rejected-coordinate state.** A coordinate is unverified or
  verified at a fingerprint; a wrong pin is moved.
- **Creating a parallel `accessed_via` topology.** The existing connected-track
  and network relationships already describe it.
- **Manufacturing road distances through connected-track topology or gateways**
  (including the unused `route_inherited` mechanism). The distance graph holds
  only defensible calculated evidence.
- **Treating `nearbyPlaces` as the relationship or distance authority.** It is
  a derived, published projection.
- **Storing distance candidates as topology.** Candidate generation is derived
  processing state.
- **Using the Sanity `_id` as the only permanent GeoEntity identity.** It
  cannot represent pre-publication or imported entities.
- **Moving editorial Place ownership from Sanity into the API.** The API owns
  the geographic service, not editorial documents.
- **Building the permanent platform in the web-era access database.** It has
  no migration ledger and sits outside the shared-schema discipline.
- **Automatic Directions fallback for non-credible Matrix evidence.** It uses
  the same road graph and resolves nothing; such cases go to review.

## Relationship to existing documents

- `architecture/shared-database-schema-discipline.md`: all platform schema
  follows it.
- `projects/personal-content-platform.md` and its stage documents state that
  Sanity is the editorial system of record and that editorial place content is
  not owned or resolved by the API. This ADR preserves that: the API owns
  geographic processing state that references editorial places, not the places
  themselves.
- `AI_CONTEXT.md` records Nearby Places as complete, deployed and
  deterministic. This ADR changes where Nearby authority lives; the status
  entry should be updated when this ADR is accepted.
