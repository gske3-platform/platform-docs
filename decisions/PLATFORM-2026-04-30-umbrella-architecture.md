# GSKE³ Platform — Umbrella Architecture

> **Naming note (2026-05-01):** the umbrella platform is now named
> **GSKE³ Platform**, not "GSKE³ Brain." The earlier name collided with
> the **KGSK Brain** sub-tool (knowledge core, regulation library, draft
> generator) and forced disambiguation in every paragraph. The platform
> is the brand; **KGSK Brain** remains the name of the internal
> knowledge-management sub-tool that lives inside it.

**Status:** strategic proposal · 2026-04-30
**Owner:** Amin (KGSK → GSKE³ transition)
**Audience:** internal review with Peter Küsters (KGSK), GSKE³
investors, future technical hires

This document proposes how SGP and **GründachHeatStress** consolidate,
together with the KGSK knowledge core itself, into a single internal
platform — the **GSKE³ Platform** (formerly "GSKE³ Brain";
during the KGSK → GSKE³ transition) — that mirrors the actual
workflow of an urban-climate consultancy from lead intake through
operation.

**Scope of the internal umbrella — exactly three things:**

1. **Solar-Gründach Planner (SGP)** — the feasibility/engineering tool.
2. **GründachHeatStress** — thermal-benefit modelling.
3. **KGSK Brain** — the consultancy's knowledge / decision-support
   surface itself: project register, cross-tool conditions panel,
   regulation library, internal SOPs and templates. It is *both* the
   shell that hosts (1) and (2) and a tool in its own right.

**Explicitly NOT in this internal umbrella:**

- The **Förder-Check on kgsk.de** — this is a *public-marketing* tool
  on the kgsk.de website (previously fronted by Cloudflare, pointing
  at the website). It stays exactly where it is and is not migrated
  into the internal Brain. See §5.1 for why the public/internal
  separation matters.
- **BauKlimaCheck** — different client context, treated as a
  standalone tool, not part of the GSKE³ launcher.
- **Amin's separate consultancy tools** — different brand, legal
  entity, client base, domain. Not represented here, not in the
  launcher, not in any GSKE³ navigation surface. Shared *technical
  infrastructure* (server, identity provider) is fine; shared product
  surfaces are not. Project records never cross that boundary.

---

## 1. Why an umbrella, not a tool collection

Three reasons individual tools must consolidate:

1. **Project continuity.** A real KGSK engagement at e.g. Roonstr. 20
   uses *both* internal tools across its lifecycle: SGP for the
   Solar-Gründach feasibility and GründachHeatStress for the thermal
   benefit case, then PDF/DOCX compositing for the actual
   deliverable. Today each tool re-asks the planner the same five
   questions (address, building height, roof form, Bundesland,
   Schneelastzone). Under the umbrella they share **one Project
   record** owned by the KGSK Brain.
2. **Trust transfer.** A reviewer who trusts SGP's regulation
   citations should automatically trust the thermal numbers in
   GründachHeatStress because they come from the same shop. Today
   each tool ships its own credibility from scratch. The umbrella is
   a single "Grün.Stadt.Klima e³" identity.
3. **One operating surface for the consultancy.** Today the project
   register, regulation library, and templates live in folders, email
   threads, and Peter's head. The KGSK Brain promotes them to a
   first-class tool so SGP and GründachHeatStress aren't islands —
   they're sub-tools of an operating system the consultancy already
   uses every day.

---

## 2. Workflow-driven IA (information architecture)

Modern urban-climate-consultancy projects move through these stages.
The platform's left navigation must mirror them so a planner finds
the next-action without having to remember which tool to open.

```
┌───────────────────────────────────────────────────────────────────┐
│ STAGE                              | TOOL(S)              | PHASE │
├───────────────────────────────────────────────────────────────────┤
│ 1. Lead intake                     | KGSK Brain (CRM)   | next  │
│ 2. Site analysis                   | KGSK Brain         |  now  │
│                                    | (Address/LoD2 hub) |       │
│ 3. Brief development               | KGSK Brain (Q&A)    | next │
│ 4. Concept design — feasibility    | SGP (Tab 1–4)       |  now  │
│                                    | GründachHeatStress  | next  │
│ 5. Detailed planning — engineering | SGP (Tab 5–7)       |  now  │
│ 6. Funding application             | (Förder-Check on   | live  │
│                                    | public kgsk.de —    |       │
│                                    | linked, not hosted) |       │
│ 7. Tender / contractor selection   | KGSK Brain (BOM)   | future │
│ 8. Build supervision               | KGSK Brain (photo) | future │
│ 9. Commissioning + handover        | SGP report bundle   |  now  │
│10. Operation + monitoring          | KGSK Brain (KPIs)  | future │
└───────────────────────────────────────────────────────────────────┘
```

**Today** the umbrella starts at stage 4 and ends at stage 9. Stages
1–3, 7–8, and 10 land in the **KGSK Brain** as it grows in. Stage 6
deliberately points *out* of the platform — to the existing
Förder-Check on the public kgsk.de site — and stays linked rather
than absorbed (see §5.1).

---

## 3. The Project record — single source of truth

Every tool reads from and writes to one canonical `project` record.
The shared columns travel; per-tool details live in tool-specific
sub-tables linked by `project_id`.

```
project                     ← top-level: id, name, status, owner
├── address                 ← geocoded, shared
├── building                ← LoD2 facts, shared
├── building_bestand        ← survey, captured by SGP, readable by all
│
├── solar_gruendach_plan    ← SGP's own table (roof, obstacles,
│                            absturz_choice, pv_layout, ballast)
├── heat_stress_simulation  ← GründachHeatStress sub-table
│                            (thermal_benefit_kwh_yr, microclimate_dT)
└── brain_*                 ← KGSK Brain tables: lead, brief,
                             regulation_note, sop_link,
                             cross_tool_condition
```

**Implementation** — already partly done in SGP:

- `building`, `building_bestand` are now shared schema (Phase 2.x).
- The Project record exists as `project` table with `building_id` FK.
- Adding a new tool means: a new sub-table + a new
  `app/api/routes/<tool>.py` + a new tab in the umbrella shell.

---

## 4. Shell + tool layout

```
                     ┌──────────────────────────────┐
                     │  GSKE³ Platform — Top Bar    │  ← single brand
                     │  Project: Roonstr. 20  ▼    │  ← project switcher
                     └──────────────────────────────┘
┌──────────────┬─────────────────────────────────┬───────────────────┐
│              │                                 │                   │
│ STAGE RAIL   │  TOOL CANVAS                    │  CONDITIONS       │
│              │  (whichever tool is             │  (cross-tool      │
│ ▣ Site       │   active for the current         │   regulatory      │
│ ▣ Concept    │   stage — SGP, GHS, FM)         │   conscience)     │
│ ◆ Detail     │                                  │                   │
│ ▣ Funding    │                                 │                   │
│ ▣ Handover   │                                 │                   │
│              │                                 │                   │
└──────────────┴─────────────────────────────────┴───────────────────┘
```

Differences vs today's SGP shell:

- **Stage rail (left)** is workflow-stage, not tool. Clicking a stage
  opens whichever tool serves that stage. SGP's eight tabs become a
  *sub-navigation* within the Concept + Detail stages.
- **Top-bar project switcher** is the only place `project_id` lives.
  Every tool reads it from URL state.
- **Cross-tool conditions panel** is the unification of SGP's
  conditions, GründachHeatStress flags, and Förder eligibility
  checks. One regulatory conscience for the whole project.

---

## 5. Brand hierarchy — GSKE³ Platform only

```
GSKE³ Platform  (the internal platform)
├── KGSK Brain              — knowledge core / project register / SOPs
├── SGP                      — Solar-Gründach Planner
└── GründachHeatStress        — thermal-benefit modelling
```

The KGSK Brain is both the **shell** (top bar, project switcher,
stage rail, cross-tool conditions, regulation library) and a tool in
its own right. SGP and GründachHeatStress are the two domain tools
mounted inside it.

Amin's separate consultancy work is a different brand, legal entity,
client base, and domain. It is intentionally not represented in this
document, in the GSKE³ launcher, or in any GSKE³ navigation surface.
Shared *technical infrastructure* (server, identity provider) is
fine; shared product surfaces are not. Project records never cross
that boundary.

BauKlimaCheck is also kept out of the GSKE³ launcher (different tool
context, see scope note at the top).

### 5.1 Why the Förder-Check stays on the public kgsk.de website

The Förder-Check is a **lead-generation tool**, not an internal one.
Its job is to convert a kgsk.de visitor into a qualified inquiry by
showing them which funding programmes might apply to their situation
— that's marketing-funnel work, not project-execution work.

Bringing it inside the Brain would:

- put it behind Authentik, killing the visitor → lead conversion;
- mix marketing analytics into the project-record schema;
- entangle the public site's release cadence with the internal
  platform's.

So the Brain **links** to the Förder-Check (stage 6 in the IA)
rather than absorbing it. When a planner reaches the funding stage
for a real project, the Brain offers a "open Förder-Check on
kgsk.de with this project's address pre-filled" action, and the
*outcome* (which programmes were applied for, amount, status) gets
written back into the project record via a small `foerder_outcome`
table. The Brain owns the project truth; kgsk.de owns the funnel.

**Open status check:** the Förder-Check was previously fronted by
Cloudflare and pointed at kgsk.de. With Cloudflare being phased out
of the sovereignty stack, **someone needs to confirm the Förder-Check
is still reachable and functional on kgsk.de today** (see §8).

---

## 6. Authentication & sovereignty

Single sign-on across all GSKE³ tools via **Authentik**, self-hosted
on the same Hetzner box that already runs SGP. Authentik is German-
hosted by the user / installer, supports OIDC, has a free tier
generous enough for a 50-user consultancy, and replaces the current
email-only KGSK Cloudflare Access setup.

Phase 2.x C-full sovereignty work (deferred from
`TICKET-2026-04-30-c-full-sovereignty.md`) folds into this same
migration: switch SGP from Cloudflare Access to Authentik when the
umbrella shell goes live.

---

## 7. Phased rollout — pragmatic, not big-bang

| Phase | Window | Scope |
|---|---|---|
| **2.x** | now → 2026-05 | Finish SGP feature completeness (Phase 3.x backlog: alt-layout, shading, tilt-optim). Keep current standalone shell. |
| **3.0** | 2026-06 | Extract the SGP shell into a generic GSKE³ Platform shell; SGP becomes one of many tools mounted in it. Add the project switcher + workflow-stage rail. |
| **3.1** | 2026-07 | Mount GründachHeatStress as the second domain tool. Define the shared `heat_stress_simulation` sub-table. Cross-tool conditions panel. |
| **3.2** | 2026-08 | Authentik replaces Cloudflare Access for SGP and the Brain shell. Verify the public Förder-Check on kgsk.de is reachable post-Cloudflare and add the deep-link from stage 6. |
| **3.3** | 2026-09 | KGSK Brain phase 1 — project register, regulation library, SOP/template store, cross-tool conditions feed. |
| **4.0** | 2026-Q4 | KGSK Brain phase 2 — lead-intake (stage 1), brief Q&A (stage 3), monitoring (stage 10). |

Reasons to phase rather than big-bang:
1. SGP must stay reachable for active KGSK projects throughout the
   migration — the brand domain `sgp.kgsk.de` continues to resolve.
2. Each tool gets de-risked individually before the next is folded in.
3. The platform shell evolves with real usage feedback rather than
   designing in a vacuum.

---

## 8. Decisions still open — flag for review

- **Which domain does the umbrella live on?** `gske3.de` (registers
  the GSKE³ brand) vs `intern.kgsk.de` (continues the existing KGSK
  internal-tool pattern). Recommend `gske3.de` because the Brain *is*
  the GSKE³ commercial product.
- **Authentik realm strategy** — single realm with role-based
  access, or one realm per tool? Single is simpler; per-tool gives
  finer-grained billing for future SaaS commercial model.
- **Public Förder-Check on kgsk.de — current uptime.** It was
  previously fronted by Cloudflare. With Cloudflare leaving the
  sovereignty stack, is the page still reachable and the wizard
  still functional? Verification step: `curl -s -o /dev/null -w
  "%{http_code}" https://kgsk.de/foerder-check` (or equivalent path)
  → must be 200, then click through the wizard end-to-end. If broken,
  decide whether to (a) restore via the new Caddy/Hetzner stack as
  a static site, or (b) re-host on Cloudflare Pages without Access
  in front (kept public). Recommend (a) — keeps the sovereignty
  story consistent.
- **Open-source posture.** SGP's code today is private. The umbrella
  could open-source the shell + parts of the KGSK Brain (regulation
  library has academic value) while keeping SGP and the engines
  proprietary. Hybrid model. Recommend open shell + closed engines.

---

## 9. What competitors look like — and where the umbrella wins

| Competitor | Their angle | GSKE³ Platform's edge |
|---|---|---|
| Optigrün's "myOptigrün" planner | Vendor-locked to Optigrün products | Vendor-neutral, comparison-first |
| ZinCo's planning suite | Solar-Gründach only | Solar-Gründach + thermal benefit + project register in one operating surface |
| Generic CAD plug-ins (Revit, AutoCAD plugins) | General-purpose, no regulation citations | German regulation citations on every output |
| Bauder's Solar-Gründach Konfigurator | Single-product calculator | Project-record continuity from feasibility → handover |
| McKinsey-style climate consultancies | Slide-deck deliverables, no live tool | Software-shipped insights, citable, repeatable |

The umbrella's defensibility isn't "more features per tool" — it's
**workflow continuity + regulatory citation + sovereign hosting**.
That's what no individual tool gives, and what no competitor in the
DACH market currently offers in one place.
