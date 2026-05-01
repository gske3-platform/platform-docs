# GSKE³ Platform — L1 Shared Spine (Specification)

**Status:** committed · 2026-05-01
**Owner:** Amin
**Implements:** Layer 1 of the build plan in
[`PLATFORM-2026-04-30-umbrella-architecture.md`](./PLATFORM-2026-04-30-umbrella-architecture.md)
**Supersedes:** the "single FastAPI app" assumption from the original
umbrella doc. Keeps everything else.

---

## 1. Why a spine

The platform federates five tools across two runtimes:

```
Hetzner (Caddy + Postgres)         Cloudflare (Pages + D1 + Vectorize)
   │                                       │
   ├── SGP            (FastAPI)            ├── KGSK Brain   (Pages Functions)
   ├── HeatStress     (FastAPI, soon)      └── Förder-Check (Pages Functions)
   └── AI Visualizer  (FastAPI, soon)
```

Without a shared spine, every cross-tool feature requires custom
point-to-point integration: SGP webhooks Brain, Brain webhooks SGP,
HeatStress webhooks both, the Visualizer webhooks SGP… N tools generate
N² integration paths. Each one re-implements auth, geocoding, identity
resolution, retry policy.

The spine collapses N² to N: every tool talks to **one** spine, the
spine fans events / cache reads / id lookups across the federation.

## 2. The four shared services

```
                       ┌──── ENTRA SSO ────┐
                       │  (M365 OIDC)      │
                       └─────────┬─────────┘
                                 │  bearer token
                                 ▼
                  ┌──────── SPINE (FastAPI) ────────┐
                  │                                  │
                  │  /api/bus/events       (1)       │  events
                  │  /api/bus/geocode      (2)       │  geocode cache
                  │  /api/bus/buildings    (3)       │  id mapping
                  │  /api/bus/identity     (4)       │  user resolution
                  │                                  │
                  └──────────┬───────────────────────┘
                             │
              Postgres (sgp.bus.* schema) + Redis (caches)
                             │
       ┌────────────┬────────┴────────┬────────────┐
       ▼            ▼                 ▼            ▼
      SGP         BRAIN          HEATSTRESS    AI-VIZ
   (host app)  (CF Pages)        (Hetzner)     (in SGP)
```

**Decision:** the spine lives **inside the SGP FastAPI app** for L1.
- New routes under `/api/bus/*`, namespaced cleanly.
- New Postgres schema `bus.*`, separate from SGP's existing tables.
- No new deployment unit. SGP becomes the host.

**When to extract:** when a third Hetzner-side service mounts (Phase
3.3, HeatStress arrival), reassess. If the bus sees > 100 RPS, extract
to its own FastAPI app on `bus.kgsk.de`. Until then, co-located.

This decision trades the architectural cleanliness of a separate
service for **two weeks of saved deployment / DNS / TLS / monitoring
work**. The trade is worth it because L1's routes are stateless reads
+ append-only writes; nothing about the bus is coupled to SGP's
business logic.

### 2.1 Service 1 — Events

A simple append-only event log. Every state change in any tool gets
posted here. Other tools subscribe by polling (L1) or by webhook
(L4+).

**Phase 3.2 amendment — reactive side-effects:** the bus is also a
fan-out point for selected event types. As of 2026-05-01:

| Event | Trigger | Side-effect |
|---|---|---|
| `project.created` from `actor_tool != 'sgp'` | Brain creates a project | Bus inserts a matching row in SGP's `public.project` table (same `id`, name from payload, status=`draft`). Idempotent. Emits a follow-up `project.mirrored` event for timeline visibility. |

This stays within the L1 spec ("append-only event log") because the
side-effect is a *DB write inside the same transaction as the event
write*, not a separate webhook delivery. No retry policy, no DLQ, no
processed-flag tracking needed. If the request fails, both the event
and the mirror are rolled back together.

**Storage:** Postgres `bus.event` table, append-only.

**Schema:**

```sql
CREATE TABLE bus.event (
    id              CHAR(32)         PRIMARY KEY,           -- ULID
    occurred_at     TIMESTAMPTZ      NOT NULL DEFAULT now(),
    type            TEXT             NOT NULL,              -- e.g. project.created
    actor_user_id   TEXT             NULL,                  -- Entra OID
    actor_tool      TEXT             NOT NULL,              -- sgp|brain|heatstress|viz
    project_id      CHAR(32)         NULL,                  -- if event scoped to a project
    payload         JSONB            NOT NULL,              -- typed by `type`
    -- denormalised for cheap filtering
    address_hash    TEXT             NULL,
    alkis_id        TEXT             NULL
);
CREATE INDEX ON bus.event (project_id, occurred_at DESC);
CREATE INDEX ON bus.event (type, occurred_at DESC);
CREATE INDEX ON bus.event (occurred_at DESC);
```

**Endpoints:**

```
POST /api/bus/events
     body: { type, actor_tool, project_id?, payload, address_hash?, alkis_id? }
     auth: Entra bearer (any authenticated tool/user)
     returns: { id, occurred_at }

GET  /api/bus/events?project_id=...&type=...&since=...&limit=100
     auth: Entra bearer
     returns: { events: [...], next_cursor: ... }

GET  /api/bus/events/{id}
     auth: Entra bearer
     returns: single event
```

**Canonical event types (L1 surface — extend over time):**

| Type | Emitted by | Payload shape |
|---|---|---|
| `project.created` | brain | `{ project_id, name, address?, project_type?, applicant_type? }` |
| `project.address_geocoded` | sgp / brain | `{ project_id, address, lat, lon, bundesland_code, alkis_id? }` |
| `project.building_resolved` | sgp | `{ project_id, building_id, alkis_id?, height_m?, roof_form? }` |
| `project.bestand_captured` | sgp | `{ project_id, building_id, completeness_pct }` |
| `project.feasibility_evaluated` | sgp | `{ project_id, suitability_tier, citation_rule_ids[] }` |
| `project.fall_protection_chosen` | sgp | `{ project_id, vendor, product_id }` |
| `project.pv_layout_computed` | sgp | `{ project_id, kwp, panel_count, kwh_year_estimate }` |
| `project.report_generated` | sgp | `{ project_id, format, bytes, included_artefacts[] }` |
| `project.funding_matched` | brain / fk | `{ project_id?, programmes[] }` |
| `project.draft_authored` | brain | `{ project_id, draft_type, draft_id }` |
| `project.audit` | * | catch-all, free-form payload, kept for GoBD |

Tools post events optimistically. The bus does not validate payload
shape against an external schema in L1 — by L4 we add Pydantic models
per type and reject malformed payloads.

### 2.2 Service 2 — Geocode cache

The platform geocodes more than any other action. Today every tool
that sees an address calls Photon directly. The cache makes the first
call canonical and shares the answer for free with every later caller.

**Storage:** Postgres `bus.geocode_cache` (durable) + Redis (hot cache,
TTL 24h).

**Schema:**

```sql
CREATE TABLE bus.geocode_cache (
    address_hash      TEXT          PRIMARY KEY,    -- SHA-256 of normalised query
    raw_query         TEXT          NOT NULL,
    formatted         TEXT          NOT NULL,
    country           TEXT          NOT NULL,
    bundesland_code   CHAR(2)       NULL,
    postal_code       TEXT          NULL,
    city              TEXT          NULL,
    street            TEXT          NULL,
    house_number      TEXT          NULL,
    lat               DOUBLE PRECISION NOT NULL,
    lon               DOUBLE PRECISION NOT NULL,
    source            TEXT          NOT NULL,   -- 'photon' | 'manual'
    confidence        DOUBLE PRECISION NULL,
    cached_at         TIMESTAMPTZ   NOT NULL DEFAULT now(),
    last_used_at      TIMESTAMPTZ   NOT NULL DEFAULT now()
);
```

**Endpoints:**

```
GET  /api/bus/geocode?q=Roonstraße+20+Hamburg
     auth: Entra bearer (or public, see auth note below)
     behaviour:
       1. Hash the normalised query.
       2. If Redis hit  → return cached.
       3. If Postgres hit → warm Redis, update last_used_at, return.
       4. Else            → call Photon, persist to Postgres + Redis,
                            emit `geocode.fetched` event, return.
     returns: GeocodeResult (same shape as SGP's existing /geocode endpoint)

POST /api/bus/geocode/manual
     body: { raw_query, formatted, lat, lon, ... }
     auth: Entra bearer (admin role)
     use:  override a Photon miss with a hand-curated answer
```

Normalisation rules (so different tools' calls hash to the same row):
- Lowercase
- Collapse whitespace
- Strip 5-digit postcode if present (Photon handles it without)
- Trim trailing punctuation
- ASCII-fold for hashing only (preserve original `raw_query`)

### 2.3 Service 3 — Building / alkis_id mapping

SGP keys buildings by ULID. HeatStress keys by `alkis_id`. The umbrella
needs a translation so SGP can ask "what's the heat-stress data for
this building?" without the planner ever knowing about alkis_id.

**Storage:** Postgres `bus.building_alias` (one building can have
multiple aliases — one per upstream system).

**Schema:**

```sql
CREATE TABLE bus.building_alias (
    sgp_building_id   CHAR(32)      NOT NULL,
    system            TEXT          NOT NULL,    -- 'alkis_de' | 'lod3_hh' | ...
    external_id       TEXT          NOT NULL,
    confidence        DOUBLE PRECISION NOT NULL,  -- 0..1, e.g. footprint IoU
    matched_at        TIMESTAMPTZ   NOT NULL DEFAULT now(),
    PRIMARY KEY (sgp_building_id, system),
    UNIQUE (system, external_id)
);
```

**Endpoints:**

```
POST /api/bus/buildings/resolve
     body: { sgp_building_id?, footprint_geojson?, system: "alkis_de" }
     auth: Entra bearer
     behaviour:
       1. If `sgp_building_id` + system already in alias table → return.
       2. Else call HeatStress: POST heatstress.kgsk.de/api/v1/buildings/match
          with footprint (centroid + IoU lookup).
       3. Persist alias, return.
     returns: { sgp_building_id, system, external_id, confidence }

GET  /api/bus/buildings/{sgp_building_id}/aliases
     auth: Entra bearer
     returns: { aliases: [{ system, external_id, confidence, matched_at }] }
```

The reverse mapping (alkis_id → sgp_building_id) is just a different
query against the same `UNIQUE (system, external_id)` index — exposed
as `GET /api/bus/buildings/by-alias?system=alkis_de&external_id=...`.

### 2.4 Service 4 — Identity resolution

A thin façade over Entra. Tools never call Microsoft Graph directly;
they ask the bus.

**Storage:** Redis cache (TTL 5 min) of `(entra_oid → user_record)`,
backed by Postgres `bus.user` for durable role assignments.

**Schema:**

```sql
CREATE TABLE bus.user (
    entra_oid       TEXT          PRIMARY KEY,
    email           TEXT          NOT NULL,
    display_name    TEXT          NOT NULL,
    role            TEXT          NOT NULL DEFAULT 'planner',  -- planner|admin|viewer
    is_active       BOOLEAN       NOT NULL DEFAULT TRUE,
    last_seen_at    TIMESTAMPTZ   NOT NULL DEFAULT now()
);
```

**Endpoints:**

```
GET  /api/bus/identity/me
     auth: Entra bearer
     returns: { entra_oid, email, display_name, role, tools_authorised: [...] }

GET  /api/bus/identity/{entra_oid}
     auth: Entra bearer (admin role)
     returns: same as above
```

The bus also exposes a FastAPI dependency `current_user()` that other
SGP routes use to attach the user to writes (audit log, project
ownership). Brain and HeatStress reach the same outcome through their
own Entra middleware verifying the same JWT signing key.

### 2.5 Auth model

All `/api/bus/*` endpoints require a valid Entra-issued bearer token.
Tools obtain tokens via the standard Entra OIDC flow — interactively
for browser flows (umbrella shell, Brain UI), or via client_credentials
for service-to-service (e.g. SGP background jobs calling the bus).

The bus validates tokens against the Entra public JWKS (cached 24h).
It does **not** issue its own tokens.

**Public exception:** the geocode `GET` endpoint MAY be open in L1
because the public Förder-Check at foerdercheck.kgsk.de needs it
without a login. Behind a per-IP rate limit (10 req/min) and CORS
allow-listed to KGSK domains. Reassess in L4.

## 3. Wire-format conventions

All endpoints:
- JSON bodies (`Content-Type: application/json`)
- Errors follow FastAPI default: `{ detail: "..." }` with HTTP status
- Timestamps in ISO-8601 UTC (`2026-05-01T12:34:56.789Z`)
- IDs are 32-char Crockford-base32 ULIDs (matches SGP's existing convention)

CORS allow-list (from L1):

```
https://sgp.kgsk.de
https://kgsk-brain.pages.dev
https://heatstress.kgsk.de
https://foerdercheck.kgsk.de
http://localhost:5173
http://localhost:3000
```

## 4. Database schema (full migration)

Lives in `backend/app/db/migrations/versions/004_bus_schema.py`. Creates
the `bus` schema and four tables above. No FKs to SGP tables — the bus
is intentionally loose so it can be extracted later without rewrites.

## 5. Rollout

| Step | Surface | Owner | Effort |
|---|---|---|---|
| 1 | Alembic migration for `bus.*` schema | me | 1 hr |
| 2 | SQLAlchemy models + Pydantic schemas | me | 2 hr |
| 3 | FastAPI router `/api/bus/*` | me | 4 hr |
| 4 | Geocode cache route shells out to existing `clients/photon.py` | me | 1 hr |
| 5 | Entra OIDC dependency (`app/api/deps_entra.py`) | me | 4 hr |
| 6 | Wire SGP's existing `/geocode` to read-through bus cache | me | 1 hr |
| 7 | Deploy backend; verify `/api/bus/identity/me` returns user | me | 1 hr |
| 8 | Brain consumes bus: replace D1 `users` table reads with `/api/bus/identity/me` | me + Brain side | 1 day |
| 9 | Brain emits `project.created` → bus `events` on every new project | me + Brain side | ½ day |
| 10 | SGP emits `project.feasibility_evaluated` after suitability runs | me | ½ day |
| 11 | Umbrella shell shows event timeline on project workspace | me | 1 day |

**Total to a useful L1:** ~3 working days for the bus itself, plus
~2 days to wire SGP and Brain into it. End of L1, both apps share
identity, share project events, share geocoded addresses. That's
the platform getting denser without adding another deployment unit.

## 6. What L1 deliberately does not do

- **No webhook delivery.** Tools poll `/api/bus/events?since=...` for
  what they need. Webhook fan-out lands in L4 when more than two
  tools subscribe.
- **No event replay / event sourcing.** The event log is a notification
  channel, not a state-of-truth. Each tool keeps its own state.
- **No payload schema enforcement.** L1 trusts tools to write
  well-formed JSON. L4 adds Pydantic-per-type validation.
- **No GraphQL / RPC abstraction.** Plain JSON over HTTP. We're not
  building a service mesh.
- **No multi-tenant.** One KGSK realm. Multi-tenant lands when GSKE³
  Platform serves clients other than KGSK — Phase 5+.

## 7. Decisions still owed (flag for review)

- **Entra App registration**: blocked on Peter (M365 admin). Until
  registered, the bus runs with `SGP_REQUIRE_AUTH=0` and a stub
  identity that pretends every caller is `amin@kgsk.de` (admin). Same
  pattern SGP uses today.
- **Domain for the bus**: extracted version (Phase 3.3+) lives where?
  `bus.kgsk.de` is consistent with `sgp.kgsk.de` / `heatstress.kgsk.de`.
  No decision needed in L1 since the bus is co-located.
- **Public geocode endpoint**: rate-limit threshold + CORS list. The
  spec proposes 10 req/min/IP; tighten if abuse appears.
- **Event payload schemas**: when L4 adds enforcement, who owns the
  registry? Recommend `docs/decisions/EVENTS-VOCAB-2026-XX-XX.md` as
  the living document, edited via PR with consensus.

---

## Appendix A — Why polling over webhooks in L1

Webhooks require: receiver to expose a public HTTPS endpoint, the bus
to manage retries, dead-letter queues, idempotency keys, signature
verification, secret rotation. None of that pays back for two
subscribers (SGP + Brain). Polling at 10s intervals across two
subscribers is 6 RPM total — cheaper than the webhook plumbing's
own logging cost.

## Appendix B — Why a SQL events table over a queue

Cloudflare Queues, Redis Streams, Kafka — all overkill. The platform
posts at most a few events per project-day. A boring Postgres table
with `(occurred_at DESC)` indexes is faster, easier to query, and
durable by default. We can put a real queue in front when event volume
exceeds 100/min — until then, Postgres is the queue.
