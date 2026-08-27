# Situation References Policy

Which endpoints may populate `data.references.situations`, and why.

## Problem

`references` blocks are a client convenience: they carry objects related to the
response's entries so clients avoid follow-up requests. Populating
`references.situations` from GTFS-RT service alerts on otherwise-static
endpoints couples those responses to real-time data, which causes:

1. **Cacheability** — static endpoints are served with the static feed ETag and
   `Cache-Control: public, max-age=...` (long). A static ETag does not change
   when an alert appears, changes, or clears, so conditional requests return
   `304` and caches serve stale situations.
2. **Client semantics** — clients treat search/lookup responses as static and
   memoize them (autocomplete caches, SQLite stop caches). Alert data embedded
   there is stale by the time anyone reads it.
3. **Endpoint responsibility** — a stop search endpoint returning alert objects
   is no longer just stop search; every alert-serialization change would have to
   be audited across every coupled endpoint.

Commit 4e09c1c added RT situation population blanket-style across all non-trip
handlers. This policy reverses that for static endpoints.

## Policy

An endpoint may populate `references.situations` only if its entries already
depend on real-time data. Static/search endpoints emit `"situations": []`
(always present — the generated JS SDK types it as required — but never
populated). Clients that need alerts fetch them from explicit real-time
endpoints or dedicated alert channels (OBACO / GTFS-RT ServiceAlerts feeds).

## Endpoint inventory

| Endpoint | Responsibility | Situations | Cache tier |
|---|---|---|---|
| `agencies-with-coverage` | static | empty | long + static ETag |
| `search/stop.json` | static (autocomplete) | empty | short (was long+ETag until #1374) |
| `search/route.json` | static (autocomplete) | empty | short (was long+ETag until #1374) |
| `agency/{id}` | static | empty | long + static ETag |
| `routes-for-agency/{id}` | static | empty | long + static ETag |
| `route-ids-for-agency/{id}` | static | empty | long + static ETag |
| `stop-ids-for-agency/{id}` | static | empty | long + static ETag |
| `stops-for-agency/{id}` | static | empty | long + static ETag |
| `trip/{id}` | static | empty | long + static ETag |
| `route/{id}` | static | empty | short (was long+ETag until #1374) |
| `stop/{id}` | static | empty | long + static ETag |
| `shape/{id}` | static | empty | long + static ETag |
| `stops-for-route/{id}` | static | empty | long + static ETag |
| `schedule-for-stop/{id}` | static | empty | long + static ETag |
| `schedule-for-route/{id}` | static | empty | long + static ETag |
| `block/{id}` | static | empty | long + static ETag |
| `stops-for-location` | static (location search) | empty | short |
| `routes-for-location` | static (location search) | empty | short |
| `current-time` | dynamic, no entities | empty | short |
| `vehicles-for-agency/{id}` | real-time | populated | short |
| `trips-for-location` | mixed (active trips) | populated | short |
| `trips-for-route/{id}` | real-time (trip status) | populated | short |
| `trip-details/{id}` | mixed (status opt-in) | populated | short |
| `trip-for-vehicle/{id}` | real-time | populated | short |
| `arrival-and-departure-for-stop/{id}` | real-time | populated | short |
| `arrivals-and-departures-for-stop/{id}` | real-time | populated | short |

Endpoints whose entries carry `situationIds` must keep populating the matching
references, so entry IDs always resolve. Endpoints without `situationIds` have
no resolution requirement, which is what makes dropping references there safe.

## Cacheability

PR #1374 worked around the staleness tactically by dropping
`search/stop.json`, `search/route.json` and `route/{id}` from the static ETag /
long-cache tier to the short cache while they still embedded GTFS-RT alerts.
This PR removes the root cause: the coupling itself. With the coupling gone,
those three are pure functions of static GTFS data again, so restoring them to
the long + static-ETag tier is a valid follow-up. `stops-for-location` and
`routes-for-location` are also purely static now; promoting them to the long
tier is another possible follow-up, not required for correctness.

## Client impact

All four official clients were reviewed (full-source grep of default branches;
Android also checked against the legacy v2.9.3 release):

- **Wayfinder** reads `references.situations` in exactly one place:
  arrivals-and-departures-for-stop (`StopPane.svelte`). Map/system-wide alerts
  come separately from OBACO `alerts.pb`. Search results are cached server-side
  1h (`serverCache.js`) — precisely the kind of cache that would have pinned
  stale alert references.
- **JS SDK** types `References.situations` as a required array but has no
  situation-consuming logic. Emitting `situations: []` keeps the type contract.
- **iOS** decodes tolerantly (`decodeIfPresent ?? []`) and resolves
  `situationIds` only on arrival/trip-status models. Agency alerts come from the
  Obaco API; stop-search results persisted in SQLite contain no situation data.
- **Android** resolves situations only on arrivals paths (both the rewritten
  main branch and legacy v2.9.3). Wide-area alerts come from direct GTFS-RT
  ServiceAlerts feeds.

No client consumes situations from any static/search endpoint, so removing the
population breaks none of them. Legacy Java OBA behaves the same way: its wiki
documents `references.situations` as present-but-empty for these endpoints
(e.g. routes-for-location), which Maglev now matches again.

## Follow-up work

- Code: none further required for correctness. Optional: promote
  stops-for-location / routes-for-location to the long cache tier.
- Spec/wiki: record the policy on each affected endpoint's wiki page
  (Implementation Decisions sections).
- Clients: no migration needed.
