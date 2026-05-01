# **G**SK e³ — Platform

> Internal umbrella platform for **Grün.Stadt.Klima e³ GmbH (KGSK)** —
> five federated tools sharing one identity, one project record, one event bus.
>
> *Grün · Stadt · Klima · Energie.*

This repo is the **navigation layer**: architecture decisions, the L1 spine
spec, the build plan. The actual code lives in the sibling repos under
[github.com/gske3-platform](https://github.com/gske3-platform).

---

## The five tools

| Repo | Role | Stack | Live at |
|---|---|---|---|
| [`sgp`](https://github.com/gske3-platform/sgp) | **Host** — feasibility planning + report. Hosts the L1 spine. | FastAPI + React + Postgres+PostGIS | [sgp.kgsk.de](https://sgp.kgsk.de) |
| [`kgsk-brain`](https://github.com/gske3-platform/kgsk-brain) | Knowledge core: RAG, HOAI, Fördercheck, drafts, project memory | Next.js + Cloudflare Pages + D1 + Vectorize | [kgsk-brain.pages.dev](https://kgsk-brain.pages.dev) |
| [`gske3-heatstress`](https://github.com/gske3-platform/gske3-heatstress) | Per-building heat scoring (GPS) for ~5 M buildings, NRW + HH | FastAPI + React + Postgres+PostGIS | (laptop only — Phase 3.3) |
| [`kgsk-foerdercheck`](https://github.com/gske3-platform/kgsk-foerdercheck) | Public lead-gen funding finder | Next.js + Cloudflare Pages | [foerdercheck.kgsk.de](https://foerdercheck.kgsk.de) |
| [`gske3-foerdercheck`](https://github.com/gske3-platform/gske3-foerdercheck) | Extraction bundle — source for the AI Greening Visualizer | Next.js | (not deployed; archived after Phase 3.5) |

---

## Where to start reading

1. **[`decisions/PLATFORM-2026-04-30-umbrella-architecture.md`](./decisions/PLATFORM-2026-04-30-umbrella-architecture.md)**
   — strategic doc. What the platform is, who it's for, how the tools relate,
   the workflow stages, the brand hierarchy, the phased rollout 2026-05 → Q4.

2. **[`decisions/SPINE-2026-05-01-shared-services.md`](./decisions/SPINE-2026-05-01-shared-services.md)**
   — L1 spine spec. The four shared services every tool talks to (events,
   geocode cache, building-alias map, identity), endpoint contracts, schema,
   rollout. **This is what's already live as of 2026-05-01.**

---

## State of play (2026-05-01)

- **Live:** SGP (sgp.kgsk.de) hosting the L1 spine at `/api/bus/*` ·
  KGSK Brain (kgsk-brain.pages.dev) emitting `project.created` events to
  the spine · Förder-Check (foerdercheck.kgsk.de) as a satellite, not yet
  reading canonical data from Brain.
- **Next physical step:** Phase 3.1 — Brain serves canonical funding-program
  JSON · Förder-Check switches to fetch from it. ~1 day.
- **Blocked on Peter:** Microsoft Entra App registration for SSO.
  Pre-Entra, the spine uses a stub `amin@kgsk.de` admin identity.
- **Not yet on a server:** GründachHeatStress (still on Amin's laptop;
  Phase 3.3 deploys it to Hetzner under `heatstress.kgsk.de`).

---

## Branding

- **Platform brand** (umbrella, what staff see): **GSKE³ Platform**
- **Sub-tool inside it** (knowledge / drafts / RAG): **KGSK Brain**

These were briefly merged as "GSKE³ Brain" in early drafts (2026-04-30);
disambiguated 2026-05-01. Don't reintroduce the merged name.

---

## Conventions

- **Two canonical IDs** the platform agrees on:
  - `project_id` — owned by KGSK Brain (and SGP for SGP-born projects).
    32-char Crockford-base32 ULID-ish or 32-char hex; both fit `String(32)`.
  - `alkis_id` — owned by GründachHeatStress as the open-data join key.
    Federal building ID.
- **One identity provider** (post-Entra): all tools verify the same JWKS.
- **One geocode cache** (the spine) so no tool calls Photon redundantly.
- **One event log** (the spine) so the umbrella shell timeline is complete.

---

## Adding a new tool

When a sixth tool joins the platform, its onboarding looks like:

1. Push to `github.com/gske3-platform/<name>`.
2. Add an entry to the **Modules** table in this README.
3. Pick its merge mode — A (port into SGP), B (wrap engine, fresh UI),
   C (re-implement formulas), or D (federate, talk to spine over HTTP).
4. Wire its writes to emit events to `/api/bus/events` with a unique
   `actor_tool` value. The umbrella shell timeline picks them up
   automatically.
5. If the tool produces a per-building output, write a `building_alias`
   row mapping its key to the SGP `building_id`.

That's the protocol. No sixth shared service is needed.

---

## 🤝 Contact

**Grün.Stadt.Klima e³ GmbH** · [kgsk.de](https://kgsk.de)
Maintainer: Minka "Amin" Aduse-Poku — [ma@kgsk.de](mailto:ma@kgsk.de)
Product owner: Peter Küsters

> *Grün · Stadt · Klima · Energie.*
