# Geographic platform: identity, coordinate QA, relationships, access and Nearby Places

## Status

Accepted — 2026-10-04. This ADR is the architectural authority for geographic
platform implementation. It governs all future geographic processing work and
supersedes the transitional Milestone A.1 / ORS Matrix tooling as the target
design. It does not retroactively invalidate data that tooling produced.

History:

- Proposed 2026-09-29.
- Revised and accepted 2026-10-04 after the Place Processing Stage 1 design and
  the road-access audits. The revision added:
  - road-based Nearby proximity (no crow-flight fallback);
  - road-access semantics and road-access points;
  - explicit `supercluster_member`;
  - the Stage 1 / Stage 2 boundary;
  - the separation of provider observation, policy assessment and human
    decision for road evidence;
  - retirement of manual and estimated road distances.

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
| Road-access semantics | How should road routing treat this Place? | Geographic platform |
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
  or Accommodation, and for later stale or incomplete geographic work. Stage 1
  establishes geographic semantics; Stage 2 calculates road evidence and
  prepares Nearby Places.

**Governing principle — explicit geographic semantics.** Stable, human-known
geographic facts are stored explicitly as structured metadata whenever they can
be represented deterministically. Neither routing code nor a future AI should
have to reconstruct them from article prose, ORS behaviour, snap distances,
Nearby ranking or proximity to a representative pin. Article text is evidence
and a source of suggestions, never the authority.

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
- road-access semantics and road-access points;
- the structural relationship graph;
- the distance/access graph;
- processing and review state;
- Nearby proposal and approval state.

A GeoEntity *references* an editorial Sanity document where one exists. It
never takes ownership of that document.

The platform becomes the processing authority for confirmed classification,
topology and road-access semantics at an explicit cutover, not before.

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
- **Provenance** records how the current coordinate arrived:
  `sanity_editorial` (any coordinate ingested from Sanity, including Studio
  entry and Bookit-synchronised accommodation), `pin_review`,
  `metadata_import`, or `manual_admin_entry`.
- **Single coordinate writer.** For a published entity, the Sanity coordinate
  field is the only mutable coordinate. The platform records, fingerprints and
  verifies it, and never edits a coordinate independently of it.
  - *Studio entry or edit:* the editor writes Sanity; ingestion detects the new
    fingerprint and records it as unverified with provenance
    `sanity_editorial`.
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

## Three separate geographic concepts

Every Place and Accommodation is described by three independent facts. They
are never collapsed into one another.

| Concept | Meaning | Examples | Owner |
|---|---|---|---|
| **A. Canonical location** | Where the actual Place is | the beach, waterfall, track feature, lookout or accommodation itself | Pin Review (coordinate QA) |
| **B. Road-access metadata** | How road routing treats the Place: `direct`, a separate road-access point, or `none` | a beach served by its own car park; a long accommodation driveway | Place Processing Stage 1 |
| **C. Structural topology** | How actual Places relate to one another | track connections; hub, cluster and supercluster membership; road, river and network relationships; departures | Place Processing Stage 1 |

- A canonical pin is never moved to satisfy road routing (B), and never to
  express topology (C).
- A road-access point (B) is routing metadata, not a Place and not topology.
- Topology (C) never produces a road distance.

## Structural relationship model

The structural graph stores **genuine structural relationships only**, using
the existing TripIdeas vocabulary. No parallel vocabulary (for example
`accessed_via`) is introduced.

**Classification (per entity):** the existing `placeType` enum (`supercluster`,
`cluster`, `hub`, `road_hub`, `river_hub`, `track_network`, `place`). No
parallel taxonomy. For processing purposes an unset value means *awaiting
classification*, not `place`. Existing values are imported; missing values are
proposed and then reviewed or bulk-confirmed in Stage 1.

**Relationship types:** the existing structural vocabulary, plus one
deliberate addition:

- `supercluster_member` *(new)* — supercluster → member Place. Supercluster
  membership is a genuine semantic relationship and is no longer represented
  indirectly as `article_reference`, so "which Places belong to this
  supercluster?" can be answered without knowing a Nearby implementation
  convention.
- `cluster_member`, `hub_member`, `hub_connection`, `road_hub`,
  `connected_track`, `connected_place`, `access_hub`, `route_network_member`,
  `departure_point`.
- `article_reference` — retained only for genuine article-reference
  semantics, not membership.

**`hub_connection` and `access_hub` need semantic review before use.** They
are not removed merely because current usage is sparse. Before Stage 1
topology schema or migration work relies on either one, determine whether it
represents:
- a genuine, distinct structural relationship that should remain; or
- historical terminology now cleanly covered by another type.

If it is redundant, deprecate it deliberately. There is no incidental enum
cleanup as part of Stage 1A. This review does not block the rest of this ADR.

Each relationship records its type, the evidence signal, and its status
(proposed / confirmed / rejected) with the deciding human. A rejected
relationship is kept so the same false positive is not proposed again.

**Not topology:**

- `manual` Nearby choices are Nearby ranking decisions. They belong to Stage 2
  as manual Nearby positions, not to the structural graph.
- `distance` is never topology (see below).

**Direction:**

- Container/member relationships are directional: container → member.
  - `supercluster_member`, `cluster_member`, `hub_member`, `road_hub` (road page
    → Place on it), `route_network_member`, `departure_point`.
- Connections are symmetric: a confirmed connection can be traversed from
  either Place.
  - `connected_track`, `connected_place`, `access_hub`.
  - The domain does not require separate A→B and B→A facts. The storage
    representation is an implementation choice. A per-source label (for
    example `connected_track` when seen from a beach, `connected_place` when
    seen from the track) is a derived projection.

**Supercluster membership workflow.** A supercluster pin is only a
representative point for an extended geographic concept, so membership is
never inferred from distance to it (this failed previously). Stage 1 follows
the general pattern *evidence → candidate set → map → human selection →
durable relationship*:

1. Gather evidence:
   - Places explicitly linked from or named in the supercluster article;
   - existing approved structural relationships (including the historical
     supercluster Nearby entries);
   - other strong editorial signals already available.
2. Present the candidate list with each candidate's evidence.
3. Show all candidates together on the map. Proximity may be shown as
   secondary context only.
4. Douglas selects the true members and explicitly excludes false positives.
5. Selections become confirmed `supercluster_member` relationships;
   exclusions are recorded as rejected.

Candidate generation never becomes authority by itself.

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
(`island-config.ts`) into confirmed relationships and access records. Being on
an island does not by itself make a Place's road access `none` (see below).

**Mercer Bay — the required topology test.** The journey is conceptually:
road/car park → Ahu Ahu Track or Mercer Bay Loop Track → Comans Track →
informal Mercer Bay track → Mercer Bay. Mercer Bay is represented by:

- its real canonical pin at the bay (A);
- `roadAccess = none`, reason `track_network` (B);
- confirmed connections through the named track network (C), for example
  Comans Track ↔ Mercer Bay.

It does **not** acquire a road-access point because its article mentions a car
park earlier in the journey. That car park belongs to the track network's
access context (the tracks that begin there), not to Mercer Bay. Reachability
from a road is derived by traversing the topology, not stored, and never
produces a road distance.

## Road-access semantics

Each Place and Accommodation carries a durable road-access fact, decided in
Stage 1:

| `roadAccess` | Meaning | Road-routing endpoint |
|---|---|---|
| `direct` | Road distance to the canonical location is meaningful. The normal case. | Canonical location |
| `access_point` | The canonical location is correct, but legitimate vehicle access is represented by a separate road-access point that directly serves the Place | The road-access point |
| `none` | There is no meaningful road-routing endpoint for this Place itself | None — never routed |
| *(unset)* | Not yet assessed | — |

- **`none` does not mean inaccessible.** The Place may be easily reached on
  foot through connected tracks, by boat or ferry, through another Place, or
  through other topology. It means only that road distance to the Place itself
  is not meaningful. This distinction is stated explicitly because AI will
  consume the metadata.
- **There is no permanent `unknown` value.** Unset means not yet assessed.
  Undecided or ambiguous cases are workflow state, not geography.

**Reasons for `none`** (a small, stable, semantic vocabulary):

| Reason | Meaning |
|---|---|
| `regional_extent` | The pin represents an area or corridor construct (supercluster, river corridor), not a vehicle destination |
| `track_network` | Reached on foot through named tracks or Places; confirmed connections must exist |
| `walk_in` | Reached only on foot, with no named intermediate track or Place |
| `water` | Reached by boat, ferry or air; no vehicle access at the destination |
| `other` | Requires a human note |

Provider and routing observations are **not** reasons. A private-road routing
failure, an ORS failure, a large snap distance, or a shared access point are
observations or derived facts and are recorded elsewhere.

**Deterministic rules from confirmed classification:**

- `supercluster` ⇒ `none` / `regional_extent`. The pin represents the
  geographic construct and is never a road-distance destination.
- `river_hub` ⇒ `none` / `regional_extent`. The pin represents a river or
  corridor, not a vehicle destination. Individual Places along the river may
  be `direct`. The corridor nature is already expressed by the classification
  together with the reason.
- No other classification implies a road-access value. Hubs, clusters, road
  hubs and track networks follow their actual semantics.
- **Islands are not a rule.** An island may have its own usable road network
  (Waiheke's Oneroa and Palm Beach are `direct`). A mainland ↔ island road
  discontinuity is a Stage 2 road-evidence outcome, not a property of every
  Place on the island. An island destination with no vehicle access (for
  example Rangitoto or Tiritiri Matangi) is `none` / `water` by human decision.

**The 100 m snap threshold is evidence, not geographic truth.**

- At or below 100 m it is a useful automatic road-evidence acceptance
  threshold (Stage 2), and above 100 m a useful review trigger.
- It never makes a canonical coordinate wrong. It never requires an access
  point, and never implies `none`. New Zealand geography produces many
  legitimate cases beyond it (beaches, headlands, park features and track
  starts).

**Ambiguity goes to human review.** Road-access decisions are not forced by a
distance rule. Example: Tawhitokino Beach's article describes both the beach
and an approach of about 1 km on foot. Keeping the canonical pin at the beach
and accepting that routing is materially removed from it, and treating the
start of the walk as relevant access metadata, are both defensible. Such a
Place carries a workflow review reason (`destination_vs_access_ambiguous`)
while its durable `roadAccess` stays unset until Douglas decides.

**Decision currency.** A human road-access decision is bound to the
canonical-location fingerprint it was made at. If the canonical coordinate
changes:

- the decision becomes **stale**;
- Stage 1 surfaces the Place for reconsideration;
- **Stage 2 does not route the Place using a stale decision**; the decision
  must be reconfirmed or changed first;
- the road-access state and any road-access point are never changed
  automatically, and no ORS call occurs. A pin move simply creates
  deterministic Stage 1 work, which is practical because canonical pin changes
  are exceptional.

## Road-access points

A road-access point is **the genuine vehicle-access location that directly
serves a Place**. It is not the road location from which a multi-stage walking
journey eventually reaches the Place. That journey is topology (see Mercer
Bay).

- **A dedicated concept, not a GeoEntity** (working name
  `GeoRoadAccessPoint`). It has:
  - a stable UUID;
  - a PostGIS location and coordinate fingerprint;
  - provenance;
  - human verification at a fingerprint;
  - audit timestamps.
  It has no editorial identity. It is not a Place, is not published, never
  appears in Pin Review or Nearby, and takes no part in topology.
- **Shared.** One road-access point may serve several Places or Accommodation
  records (for example one regional-park car park serving a beach, a historic
  house and two walks). This avoids duplicating routing coordinates and is
  expected.
- **Coordinate coincidence does not imply identity.** A road-access point may
  sit at exactly the same coordinate as a GeoEntity, for example a trailhead
  Place that is also the vehicle access for a nearby beach. These are
  different facts and coexist. Nothing automatically merges them, creates
  topology, creates a Place, or infers identity.
- **Accommodation** may use road-access points, for example a long driveway
  whose entrance is the legitimate routing endpoint. The Accommodation's
  canonical coordinate is never moved to make routing easier. Bookit
  coordinate synchronisation is a separate integration concern.
- **Lifecycle:**
  - Created or placed by a human in Stage 1. Its placement is its verification
    at that fingerprint.
  - Moving it changes its own fingerprint, which makes road evidence
    calculated from or to that endpoint stale.
  - Changing a referencing Place's canonical pin makes that Place's road-access
    decision stale (see *Decision currency*), but never moves the access point.
- **Not `accessed_via`.** `accessed_via` (rejected) was a Place → gateway
  relationship used to inherit distance. A road-access point is only the
  endpoint for the Place's own road evidence. No road distance is ever
  manufactured by combining road distance → access point → walking distance
  → Place.

**Distance candidates are not relationships.** Candidate generation for
distance enrichment is derived, transient processing state. It is never stored
as topology.

## Distance/access model

- The access graph stores **actual, defensible calculated evidence**: one row
  per directed pair and travel mode, with:
  - endpoint fingerprints at calculation time;
  - route distance and duration;
  - routing-provider provenance (`ors_matrix`, and the legacy `openroute`
    Directions pass);
  - for road evidence, the ORS snap distance and snapped point of each
    endpoint.
- **Routing endpoint.** Road evidence for a Place uses its road-access
  endpoint: the canonical location (`direct`) or its road-access point
  (`access_point`). Places with `none` are never routed. Each evidence row
  records the endpoint used and that endpoint's fingerprint, so a moved
  access point, or a change between `direct` and `access_point`, makes the
  evidence stale by the usual fingerprint rule.
- **All useful calculated pairs are retained**, regardless of whether they are
  selected into Nearby Places. Nearby is one consumer of the graph.
- **Crow-flight is never road evidence.** Straight-line distance is used
  internally for candidate discovery and diagnostics (for example road/crow
  ratios) only. It is never stored as access evidence, never published as a
  Nearby distance, and never substitutes for missing road evidence.
- **Road outcome for a pair:**
  - a credible measured road connection;
  - no road connection found;
  - human review required.
  Provider observations (route returned, no route, endpoint unsnappable, zero
  distance, snap distances) are evidence. A human conclusion such as "this
  Place legitimately has no road connection" is recorded separately from them.
- **Observation, assessment and decision are separate layers.**
  - **Provider observation:** what an actual routing provider returned, kept
    unchanged. This includes the outcome (route, no route, endpoint
    unsnappable, zero distance), measured distance and duration, endpoint snap
    observations, provider and calculation time.
  - **Policy assessment:** whether the observation is credible under a named
    policy version, with reason codes.
  - **Human decision:** for example, accepting evidence, or confirming that no
    road connection exists, recorded with the deciding human and the endpoint
    fingerprints it applies to.

  Because observations are never rewritten, old evidence can be reassessed
  under an improved policy.
- **Valid provider evidence** is a technically usable observation from an
  actual routing provider: a measured, positive distance with duration,
  endpoint observations and provider provenance. **Valid is not credible.**
- **Credible road evidence** is valid provider evidence that also satisfies the
  current credibility policy. Only credible evidence (or evidence a human has
  explicitly accepted) is used for Nearby and distance reasoning.
  - A historical `needs_review` status never counts as credible automatically.
    It covers very different cases:
    - legitimate routes flagged only by the historical road/crow ratio rule;
    - endpoint snap concerns;
    - same-snap and zero-distance observations;
    - genuinely questionable evidence.
  - Each case is reassessed under the permanent policy. That reassessment is a
    separate, later task.
  - The web-era predicate `route_status IN ('ok','manual','needs_review') AND
    route_km IS NOT NULL` is historical. It describes *valid* rows, not
    credible ones.
- **Manual and estimated distances are retired.** A human may confirm:
  - road-access semantics;
  - topology;
  - a road-access point's location;
  - whether evidence is acceptable.

  A human never enters or estimates kilometres as road evidence. Historical
  `manual` and `estimated` rows (including the two Rangatira estimates) remain
  legacy/archive data during migration. They are never permanent measured
  road evidence, and stay historical until replaced by genuine measured
  evidence.
- **Road-evidence credibility policy**
  (`DEFAULT_ROAD_EVIDENCE_POLICY.maxCredibleSnapM = 100`). A Matrix road result
  is credible only when:
  - the distance is > 0;
  - both endpoints snap within 100 m;
  - the endpoints snap to different road points.

  Otherwise the observation is retained, and the pair is assessed as needing
  road-routing review; it contributes no credible road distance.
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

**Nearby proximity is road-based.** Nearby Places is derived (in Stage 2) from
human selections, structural relationships and road evidence, in this order:

1. **Human and structural selections first.**
   - Manual Nearby positions assigned by Douglas in Stage 2 and
     `departure_point` always rank first.
   - Then the structural buckets, ranked by source page type (the existing
     `bucketRank`: supercluster, route network, hub, track, place).
   - Human structural or manual selections may occupy all six positions and
     are never displaced by distance.
2. **Road evidence fills only the remaining positions.** For a pair A→B, use
   credible (or human-accepted) A→B road evidence; otherwise credible (or
   human-accepted) B→A road evidence. Never
   average directions. When both exist, source → candidate wins even if the
   reverse is shorter. A candidate with no credible road evidence in either
   direction is not distance-ranked: its outcome is "no road connection found"
   or human review. **There is no crow-flight fallback.**

Places whose `roadAccess` is `none` take no part in road-distance ranking,
either as source or as candidate. Their Nearby Places come from structure and
human selection. Structural relationships do not require road-distance
evidence to be included or ranked. If fewer than six sensible Places exist,
positions stay empty rather than being filled by straight-line proximity.

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
     topology or road-access semantics** (connected-track or network
     membership, island access, departure relationships, `roadAccess` of
     `none` or `access_point`).

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

**Stage 1 — geographic semantics.** Stage 1 answers: *what is this Place, how
is it connected to other Places, and how should road routing treat it?* It
owns:

- classification;
- structural topology;
- road-access semantics;
- road-access points;
- human confirmation of all four.

1. Discover entities with gaps (new, unclassified, topology unreviewed,
   road access unassessed or due for reconsideration).
2. Propose classification from deterministic evidence. A human confirms or
   amends it, individually or in bulk.
3. Propose structural relationships using the existing vocabulary and signals:
   - supercluster, cluster and hub membership (superclusters through the
     member workflow above);
   - road/road-hub relationships;
   - track/track-network and river-network relationships;
   - connected Places;
   - island/departure relationships.

   A human confirms or rejects them.
4. Propose road access:
   - deterministic rules from confirmed classification;
   - otherwise evidence (for example current road snap evidence, topology,
     article excerpts).

   A human confirms `direct`, places or reuses a road-access point, or records
   `none` with a reason. Ambiguous cases carry a workflow review reason and
   stay unset.
5. A Place that appears to have a wrong pin is referred to Pin Review. Stage 1
   never edits a canonical coordinate.

Stage 1 performs no ORS or Matrix work. It may display existing road evidence
as context for a proposal, but all routing happens in Stage 2.

**Workspace.** A purpose-built tool, not a reproduction of Studio:

- a queue of Places with gaps;
- a map showing the selected pin, related Places with their typed
  relationships, road-access points (with a line to the Place) and existing
  nearby access points available for reuse;
- a detail panel with classification, relationships, road access and short
  article excerpts shown as evidence;
- a completion action.

Every value is labelled as an existing human-approved fact, a system proposal,
or unresolved.

**Bulk confirmation.** Douglas is not required to click through obvious cases
one by one to create human provenance.
- Deterministic or inferred proposals, with their evidence, can be reviewed
  and confirmed in bulk; exceptions are handled individually.
- *Proposal or inference → human confirmation (individual or bulk) → durable
  fact.*
- Historical ORS success is evidence for a proposal. It is never persisted as
  if it were human geographic knowledge.
- Heuristics are never silently promoted into canonical metadata.

**Stage 2 — road evidence and Nearby Places.** Stage 2 owns:

- spatial (crow-flight) candidate discovery, internal only;
- road-distance evidence and Matrix execution;
- provider observations, measured road connections, no-road-connection
  outcomes and road-evidence review;
- manual Nearby positions;
- the derived Nearby six;
- final review and publication.

1. Identify candidate pairs. Spatial proximity is used internally for
   discovery only. Accommodation uses its own profile (the existing 30 km
   rule).
   - Only Places with a current (non-stale) road-access decision of `direct`
     or `access_point` are routed.
   - `none` Places are never routed.
   - Unset or stale decisions wait for Stage 1.
2. Reuse current road evidence (endpoint fingerprints match).
3. Determine the missing applicable pairs. Structural pairs are not routed
   merely because they exist.
4. Explicitly calculate missing road evidence through a single, quota-gated
   action using the existing Matrix engine, at each Place's road-access
   endpoint.
5. Retain all useful calculated pairs.
6. Review non-credible road evidence.
7. Douglas assigns any manual Nearby positions. Road evidence fills only the
   remaining positions (see Nearby derivation).
8. Produce and review a Nearby proposal.
9. After explicit approval, publish the approved Nearby result to Sanity
   through the existing controlled-write approach.

**Accommodation** uses the same batch view with its own proximity and selection
profile (the existing 30 km candidate rule). Accommodation Nearby selection
rules are a product decision still to be made.

## Stale and recalculation model

- **Fingerprint mismatch is the fundamental stale signal.** Access evidence
  whose stored endpoint fingerprint differs from the current fingerprint is
  stale, row by row.
- A pin change therefore:
  - makes the dependent access evidence stale;
  - makes the Place's road-access decision stale, so Stage 2 does not route it
    until Stage 1 reconfirms the decision;
  - may make dependent Nearby output stale (its recorded inputs no longer
    match).

  Once the decision is current again, the stale pairs become eligible for
  targeted recalculation. Moving a road-access point stales the evidence that
  used it as an endpoint.
- **No ORS call occurs automatically.** Stale work appears as deterministic
  gaps in Place Processing, which supports both batch recalculation and
  targeted recalculation for a single Place.
- Processing state is **derived from gaps**, not from creation dates.
  - Entity states:
    - not yet registered;
    - location missing;
    - awaiting classification;
    - topology unreviewed;
    - road access unassessed or due for reconsideration;
    - Stage 1 complete;
    - distances incomplete or stale;
    - road evidence awaiting review;
    - Nearby proposal awaiting review;
    - current.
  - Coordinate verification (unverified / verified at fingerprint) and Pin
    Review referrals are tracked alongside these states, not as one of them.
  - Workflow review reasons (for example `destination_vs_access_ambiguous`)
    are processing state, never durable geographic facts.
  - Only facts that cannot be derived from data presence are recorded
    explicitly:
    - classification, topology-review and road-access decisions (road access
      with the canonical-location fingerprint it was made at);
    - completed relationship runs (with their inputs hash);
    - Nearby decisions.

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

- **`placeType` publication.** Confirmed classifications reach Sanity through
  an explicit, reviewed batch publication analogous to Pin Review's controlled
  coordinate write, not through uncontrolled dual writers.
- **Road-access metadata stays in the platform.** Road-access semantics and
  road-access points are not projected into Sanity merely because future AI
  will use them. They are projected only if a public or editorial consumer
  needs them.

## Migration and transition strategy

- The web-era `place_access` database may remain temporarily behind a
  repository/service interface. The new tools depend on that interface, not on
  the database's location.
- **Structural relationships** are seeded from:
  - confirmed structural entries in Sanity `nearbyPlaces` (primary source);
  - the structural rows of `place_relationship`;
  - island and departure configuration currently in code.

  Seeding rules:
  - Symmetric connections approved in both directions become one connection.
  - Historical supercluster `article_reference` entries become *proposed*
    members in the supercluster member workflow, not automatic memberships.
  - Historical `manual` entries become Stage 2 manual Nearby positions, not
    topology.
  - Rows found only in `place_relationship`, and contradictory pairs (for
    example two hubs each listed as a member of the other), go to review.
- **Classification** is seeded from existing Sanity `placeType` values.
  Missing values are proposed for review or bulk confirmation.
- **Road access** is not seeded as fact. Classification rules and historical
  road evidence produce proposals for human (bulk) confirmation.
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
- supercluster pool — only for candidate discovery, article-anchor evidence,
  presenting possible members and map context. Its proximity fallback is
  contextual evidence only and never establishes membership. Membership
  follows the supercluster member workflow (evidence → candidate list → map →
  human selection/exclusion → confirmed `supercluster_member`);
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

## Semantic substrate for AI

Classification, topology and road-access metadata serve more than Nearby
Places. They are the structured substrate that future deterministic and AI
reasoning (map reasoning, itinerary reasoning) reads.

**Facts it should read directly:**

- this Place is directly road accessible;
- this Place has a distinct vehicle-access coordinate;
- this beach requires a walk from road access;
- this Place lies inside a named track network, connected to another named
  track;
- this Place belongs to this supercluster;
- this river hub is a corridor, not a road destination;
- these Places share a vehicle-access point;
- this Accommodation has a long driveway and a distinct routing entrance;
- this fact has been human verified.

**Sources it must never infer them from:**

- article prose;
- ORS failures or snap distances;
- Nearby ranking;
- proximity to a representative hub or supercluster pin.

**Authority levels.** Article text and AI may later supply evidence and
proposals. Authority is distinguished using the platform's existing
conventions:
- *human-confirmed fact* — confirmed status, deciding human and time;
- *imported human fact* — a migration actor reference;
- *deterministic system inference* — computed, not stored as a decision;
- *proposal* — proposed status.

## Consequences

- Nearby Places stops being the relationship authority. Its Sanity data becomes
  a projection, which removes the current circularity (relationships stored
  inside the output derived from them).
- Nearby proximity is road-based. Places without credible road evidence are
  not ranked by straight-line distance, so some Nearby lists will have fewer
  distance-filled positions.
- Canonical location, road access and topology are separate facts. Pins are
  no longer distorted to suit routing, and Places such as Mercer Bay are
  described correctly.
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

1. Accept this ADR (done 2026-10-04).
2. Foundation schema in `tripideas-api` (Prisma + PostGIS, shared-schema
   discipline): GeoEntity, coordinate verification, classification, structural
   relationship, access evidence (done).
3. Pin Review (done).
4. **Stage 1A.1 — Schema foundation** (next). Only the approved domain/schema
   foundation for:
   - road-access semantics;
   - road-access-point metadata;
   - explicit `supercluster_member`;
   - topology review/completion state;
   - separation of provider observation, policy assessment and human decision
     where the access-evidence model requires it.

   No UI.
5. **Stage 1A.2 — Seeding dry run.** A deterministic, read-only seeding process
   reporting:
   - imported classifications;
   - structural relationships available from approved Sanity Nearby data;
   - deduplication and normalisation;
   - proposed `supercluster_member` migration;
   - relationship exceptions requiring review;
   - road-access proposals;
   - ambiguous cases;
   - likely road-access-point cases;
   - actual population sizes.

   No canonical writes without a later explicit approval step.
6. **Review checkpoint.** Inspect the real dry-run populations before designing
   the bulk-review workspace. The dry run may change which UI is most
   efficient.
7. **Stage 1A.3 — Bulk review workspace.** Classification confirmation,
   road-access confirmation, bulk confirmation and exception handling.
8. **Stage 1B.** Topology editor, the supercluster candidate/member workflow,
   road-access-point placement and reuse, and richer structural review
   (including the `hub_connection` / `access_hub` semantic review before
   either type is used).
9. Add a repository/service interface over the web-era access graph for the
   transition.
10. Place Processing Stage 2: Matrix engine; road-evidence review; manual
    Nearby positions; Nearby proposal, review and publication.
11. Migrate the access graph (reassessing historical evidence under the
    permanent policy); archive the candidate log.
12. Retire A.1 tooling; deprecate Sanity workflow fields.

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
  and network relationships already describe it. Road-access points are not a
  revival of it: they are routing endpoints for a Place's own road evidence,
  not relationships, and never combine with walking segments.
- **Crow-flight as a Nearby distance or fallback.** Straight-line distance is
  internal discovery and diagnostic evidence only.
- **Representing road-access points as GeoEntities.** They are routing
  metadata, not Places. Coordinate coincidence with a Place does not imply
  identity.
- **A permanent `unknown` road-access value, or an `ambiguous` state.** Unset
  means not yet assessed; ambiguity is workflow review state.
- **Deciding road access from a snap-distance threshold.** The 100 m policy is
  an evidence threshold and review trigger, not geographic truth.
- **Inferring `none` from island location.** Islands may have their own road
  networks.
- **Inferring supercluster membership from distance to the supercluster pin.**
  Membership is selected by a human from editorial evidence.
- **Storing manual Nearby choices as topology.** They are Stage 2 ranking
  decisions.
- **Treating historical `needs_review` (or any valid-but-unassessed) evidence
  as credible.** Credibility is a policy assessment over preserved provider
  observations.
- **Manually entered or estimated road distances.** Humans confirm semantics,
  topology, access-point locations and evidence acceptability, never
  kilometres.
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
