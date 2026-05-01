# TICKET — Mount GründachHeatStress on the GSKE³ Platform

**Status:** SGP-side ready · Server-side pending operator action
**Phase:** 3.3 of the platform rollout
**Owner:** Amin (server steps), me (code already landed)
**Implements:** [PLATFORM-2026-04-30-umbrella-architecture.md §7](./PLATFORM-2026-04-30-umbrella-architecture.md)
**Related:** [SPINE-2026-05-01-shared-services.md](./SPINE-2026-05-01-shared-services.md)

---

## Summary

GründachHeatStress (FastAPI + Postgres+PostGIS, ~5 M buildings ingested
from ALKIS + LoD2 + LANUK) federates into the platform via simple HTTP.
SGP cross-references its building rows against HeatStress's catalogue
through `alkis_id`, surfaces the GPS tier and heat-stress class in the
right-rail conditions panel, and shows the Solar-Gründach combo flag
in the suitability badge.

This ticket captures **only the server-side steps**. SGP-side wiring is
already merged (PR `gske3-platform/sgp@<this commit>`):

  - ✅ `public.building.alkis_id` column (migration `005_alkis_id`)
  - ✅ `app/clients/heatstress.py` graceful client
  - ✅ `app/api/routes/heatstress.py` — `/status`, `/resolve`, `/{alkis_id}`
  - ✅ Bus alias rows + `project.building_resolved` events emitted on resolve

When the steps below are complete and `SGP_HEATSTRESS_URL` is set on
the SGP backend, the federation goes live with **no further code change**.

---

## Pre-flight

| Item | Status / value |
|---|---|
| Hetzner box | `178.104.36.202` (already runs SGP at `sgp.kgsk.de`) |
| Repo on disk | `C:\Users\amink\Tools Folder\gske3-heatstress` (`gske3-platform/gske3-heatstress` on GitHub) |
| Repo branch | `main` — 10 days of pre-handover work was already committed in `feat: WIP snapshot — Hamburg + GPS v1.2 + heat-corridor + NDVI scaffold` |
| Local data | `data/lod2/`, `data/lod2_hh/`, `data/klimaanalyse/`, `data/phase2/` — gitignored, ~250 GB raw + an active local PostGIS `gske` database with the ingested derivatives |
| DNS | IONOS — needs a new A record for `heatstress.kgsk.de` → `178.104.36.202` |
| Auth | None today; Entra OIDC adds in 3.4 — for now relies on the same Caddy/network perimeter as SGP |

---

## Step 1 — DNS (5 min, IONOS dashboard)

1. Log in to IONOS.
2. Domains → `kgsk.de` → DNS.
3. Add A record:
   - Name: `heatstress`
   - Type: `A`
   - Value: `178.104.36.202`
   - TTL: 3600
4. Wait ~5 min for propagation. Confirm:
   ```
   dig +short heatstress.kgsk.de
   ```
   should return `178.104.36.202`.

## Step 2 — Provision a separate Postgres database (10 min on the box)

HeatStress is **not** sharing SGP's Postgres. Different sizing
(~5 M building rows + raster derivatives), different lifecycle
(annual open-data ingest vs. per-project writes), different backup
cadence. Easiest path: a **second Postgres container** inside the same
Docker network.

```bash
ssh root@178.104.36.202

mkdir -p /opt/heatstress
cd /opt/heatstress

# Pull the repo fresh from GitHub.
git clone https://github.com/gske3-platform/gske3-heatstress.git src
cd src
```

Set the secrets file:

```bash
mkdir -p infra/secrets
openssl rand -hex 32 > infra/secrets/pg_password
chmod 600 infra/secrets/pg_password
```

(The repo's `docker-compose.yml` references PG vars from `.env.local`.
We override with the platform's secrets convention.)

Add an `.env.local` next to `docker-compose.yml`:

```
POSTGRES_USER=gske
POSTGRES_DB=gske
POSTGRES_PASSWORD=<read from infra/secrets/pg_password>
POSTGRES_PORT=5433     # SGP already uses 5432, second instance on 5433
REDIS_PORT=6380        # likewise, second Redis on 6380
API_PORT=8001          # second FastAPI on 8001 (SGP is on 8000)
```

## Step 3 — Bring up the stack (5 min)

```bash
cd /opt/heatstress/src
docker compose --env-file .env.local up -d postgres redis
docker compose --env-file .env.local logs -f postgres   # wait for "ready to accept connections"
```

Then start the API:

```bash
cd services/api
docker build -t heatstress-api:local .
# (no compose target for the API yet — run via docker run, or extend docker-compose.yml)
docker run -d --name heatstress-api \
    --network gske3-heatstress_default \
    -p 127.0.0.1:8001:8001 \
    -e DATABASE_URL="postgresql+asyncpg://gske:$(cat /opt/heatstress/src/infra/secrets/pg_password)@postgres:5432/gske" \
    -e REDIS_URL="redis://redis:6379/0" \
    heatstress-api:local
```

Smoke-test:

```bash
curl http://127.0.0.1:8001/api/v1/health   # expect 200
```

## Step 4 — Schema bootstrap (1 min)

HeatStress doesn't have Alembic migrations yet — schema is created
via SQLAlchemy `Base.metadata.create_all` on first startup. Trigger it:

```bash
docker exec -it heatstress-api python -c "
from src.db import engine
from src.models import Base
import asyncio
async def go(): 
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
asyncio.run(go())
"
```

After this, `gske` database has empty tables in `base.*` and `derived.*`
schemas, ready for ingest.

## Step 5 — DECISION POINT — data ingest (the long pole)

We have two options for the ~5 M building dataset:

### Option A — re-ingest from open sources (slow, deterministic)

Run the existing ingest scripts on the server against the open data
endpoints. Each script in `scripts/ingest/` pulls from a public source
(NRW WFS, LANUK Klimaanalyse, INSPIRE Denkmal, …):

```bash
docker exec -it heatstress-api python scripts/ingest/01_alkis_buildings.py
docker exec -it heatstress-api python scripts/ingest/02_lod2.py
# … etc., follow the order in scripts/ingest/README.md if present
```

**Pros:** reproducible, no big binary transfer, gets the freshest data.
**Cons:** can take many hours; some endpoints rate-limit; LoD2 raster
ingest needs ~250 GB scratch space transiently.

### Option B — transfer the local PostGIS dump (fast, less reproducible)

Dump the existing local `gske` database, ship it, restore it:

```bash
# On Windows laptop:
pg_dump -h localhost -p 5432 -U gske -F c -f gske.dump gske

# Transfer:
scp gske.dump root@178.104.36.202:/opt/heatstress/

# On server:
docker exec -i $(docker ps -qf name=heatstress-postgres) \
    pg_restore -U gske -d gske --clean --if-exists < /opt/heatstress/gske.dump
```

**Pros:** ~10–30 min total wall-clock, deterministic dataset.
**Cons:** dump is probably 30–80 GB (compressed); needs scratch space
on both sides.

### Recommendation

**Option B** for the v0 deploy — ship the dump that's already producing
correct output on Amin's laptop. Switch to Option A's automated ingest
in v0.1 once we can verify the live pipeline incrementally.

Need from Amin: the `pg_dump` step + the `scp` line above. ~30 min of
his time + bandwidth.

## Step 6 — Caddy site block (5 min)

Edit `/etc/caddy/Caddyfile` on the server, append:

```
heatstress.kgsk.de {
    encode zstd gzip

    # FastAPI backend — strip /api prefix only if HeatStress's routes
    # need it; current router uses /api/v1/* literally so we just
    # forward whole.
    reverse_proxy 127.0.0.1:8001 {
        header_up Host {host}
        header_up X-Real-IP {remote_host}
    }

    # CORS — explicitly allow the SGP origin so the frontend can call
    # HeatStress directly for any read-only requests we add later.
    @options method OPTIONS
    handle @options {
        header {
            Access-Control-Allow-Origin "https://sgp.kgsk.de"
            Access-Control-Allow-Methods "GET, POST, OPTIONS"
            Access-Control-Allow-Headers "Content-Type, Authorization"
        }
        respond 204
    }
}
```

Validate + reload:

```bash
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
```

Smoke-test:

```bash
curl -sI https://heatstress.kgsk.de/api/v1/health
# expect HTTP/2 200
```

## Step 7 — Light up SGP-side federation (1 min)

On the SGP backend, add the env var so it knows where HeatStress lives:

```bash
ssh root@178.104.36.202
cd /opt/sgp/infra/deploy
# Edit docker-compose.yml or .env.local — add:
#   SGP_HEATSTRESS_URL=https://heatstress.kgsk.de
docker compose up -d backend
```

Smoke-test the federation status endpoint:

```bash
curl -s https://sgp.kgsk.de/api/heatstress/status
# expect: {"configured": true, "base_url": "https://heatstress.kgsk.de", ...}
```

End-to-end test:

```bash
# Pick a known-good alkis_id (e.g. one from Amin's local DB).
ALKIS_ID="DENW36AL10009lidBL"

curl -s "https://sgp.kgsk.de/api/heatstress/$ALKIS_ID" | python3 -m json.tool | head -40
# expect: composite BuildingFull payload with heat, gps, suitability, etc.
```

## Step 8 — Resolve a real project's building (1 min)

Pick an existing SGP project with a geocoded address (e.g. the
Roonstr. 20 demo or any post-Phase-3.2 Brain-mirrored project), then:

```bash
# Roonstr. 20 is at lat=53.580, lon=9.961 (Hamburg).
curl -s -X POST 'https://sgp.kgsk.de/api/heatstress/resolve' \
    -H 'Content-Type: application/json' \
    -d '{"sgp_building_id":"<known building id>","lat":53.580,"lon":9.961}'
```

Should return:
- `alkis_id`: the matched federal building ID
- `match_distance_m`: small number (typically < 5 m)
- `building_full`: the full HeatStress payload

After this, `GET /api/projects/<id>` returns a `building.alkis_id`
field, the bus has a `building_alias` row, and the timeline shows a
`project.building_resolved` event with `actor_tool: platform`.

---

## After Step 8 — what works automatically

- The umbrella shell (sgp.kgsk.de) shows GPS tier + heat-stress class
  on any project whose building has been resolved.
- The conditions panel has a new "Heat priority" + "Combo bonus" check.
- The activity feed shows `project.building_resolved` events.

## Decisions still open

- **Hamburg heat dimension** — LANUK NRW-only data means HH buildings
  cap at C-tier. Decide whether to (a) accept that until a Hamburg
  Klimaanalyse equivalent exists, or (b) integrate Sentinel-3 LST as
  fallback. For v0, accept it.
- **Alembic for HeatStress** — no migrations exist. Once the deploy is
  stable, write a `001_initial.py` reflecting the current schema so
  future changes are tracked. Owner: HeatStress maintainer (Amin).
- **Mapbox token** — Amin's personal token is hard-coded into the
  HeatStress launcher's `.env.local`. Replace with a KGSK org token
  before any external user touches `heatstress.kgsk.de` directly.
- **Auth perimeter** — pre-Entra, HeatStress is reachable to anyone
  who can resolve `heatstress.kgsk.de`. If we want to keep the
  catalogue private until Entra lands, gate Caddy with an IP allow-list
  that includes Hetzner's egress IP for the SGP backend.

---

## Roll-back

If anything goes wrong after Step 7:

```bash
# Just unset the env var on SGP — federation goes back to "not configured".
ssh root@178.104.36.202
cd /opt/sgp/infra/deploy
# Remove SGP_HEATSTRESS_URL from .env.local / docker-compose.yml
docker compose up -d backend
```

SGP behaves exactly as it did before this ticket (no GPS data, no
combo flag), nothing breaks. The new column / route / client all stay
in place ready for the next attempt.
