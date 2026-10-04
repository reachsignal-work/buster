# Buster system architecture

**Status:** planning · Wave 1 active (Data Gains)  
**Audience:** designers and implementers of the buster app

Buster is a site-acquisition and techno-economic planning system for AI data-center builds. A user frames a region and constraints. The system returns ranked parcels with evidence: months-to-power, industrial power price, water and cooling posture, tax and political risk, and a versioned viability score.

## Questions the product must answer

1. Where are industrial electricity, water, and cooling operating costs lowest for a high-density AI hall?
2. Which jurisdictions offer equipment sales-tax exemptions, PILOT or property-tax abatements, and financing advantage?
3. Where is hyperscale and infrastructure-fund interest concentrating outside saturated hubs?
4. How does a new build capture that demand: time-to-power, liquid-cooling readiness, and modular offtake?

Wave 1 stores and serves the evidence for (1) and (2). Interest maps and deal memos for (3) and (4) are Wave 2, and they read Wave 1 records. They do not invent missing grid or parcel facts.

## Planes

```text
Presentation
  Map (parcel polygons, transmission, score)
  Copilot and scenario filters
  Deal-memo export
        │
        ▼
Orchestration
  Buster-Core workflow
  Tool calls into the Data Gains API
        │
        ├── GeoSite
        ├── GridPower
        ├── PolicyTax
        └── ThermalCAPEX
        │
        ▼
Data Gains
  Extractors → normalizers → observation store → feature assembler → serve API
        │
        ▼
Sources
  RTO queues, EIA rates, parcel and constraint GIS,
  NOAA climate, water tariffs, statute and agenda text
```

## Agents

Each agent is a reasoning worker with a fixed tool list. It does not scrape sources itself. Scraping, licensing, and schema normalization live in Data Gains so two agents cannot disagree about the same parcel because they fetched different files.

| Agent | Reads | Produces |
|-------|--------|----------|
| GeoSite | Parcels, slope, flood, wetlands, fiber distance, residential buffer | Buildable acreage and constraint flags |
| GridPower | Substations, lines, queue rows, tariff observations | Distance to transmission, headroom estimate, months-to-power estimate, ¢/kWh |
| PolicyTax | Structured incentive records, agenda excerpts | Exemption and PILOT eligibility, moratorium risk with citation |
| ThermalCAPEX | Wet-bulb hours, water rate, cooling-topology assumptions | Free-cooling hours, PUE/WUE scenarios, TCO inputs |
| Buster-Core | The four agent outputs plus the canonical site record | Ranked shortlist and the narrative a memo will later export |

## Two clocks

**Observation clock.** Queues, tariffs, weather, and agendas update on their own cadence. Data Gains keeps each snapshot and the delta.

**Decision clock.** A site score is computed when a user runs a search, from the latest observations plus an explicit score-model version. Historical scores stay reproducible.

## What is in a site record

A canonical site is a GeoJSON Feature. Raw tables stay source-shaped. The feature is the join: one parcel, the nearest qualifying substation and line, the climate station used, the water provider, the incentive row, and a freshness block. Derived money fields (lease rate, IRR, capex per MW) are optional and absent until ThermalCAPEX has a published model version. Data Gains v1 does not invent them.

See [data-gains.md](data-gains.md) and [canonical-site.schema.json](../../contracts/canonical-site.schema.json).

## Geography packs

v1 acquisition search covers US parcels in:

- PJM, with the Midwest corridor (AEP Ohio and neighboring MISO counties) first
- MISO (Indiana, Iowa, and the upper Midwest as climate context)
- ERCOT (West Texas, Abilene, Panhandle)
- SPP as a queue and transmission overlay once the first three extractors are stable

Pacific Northwest, TVA, and the Nordics enter first as **benchmark observations** (power price, free-cooling hours, water posture) so a US shortlist can be compared. Parcel search outside the United States waits on a later geography pack.

## Trust rules

- Every numeric field that can change has `source_id` and `retrieved_at`.
- Restricted transmission data (FERC Form 715 CEII) is out of scope until a lawful access path exists. Public queue reports and public transmission geometry are the v1 grid picture.
- A missing source yields an empty field and a gap list on the feature. It does not yield a silent default from the ideation brief.
- Score weights are configuration, versioned with the model, and shown beside the score.

## Wave boundary

| Wave | Ships |
|------|--------|
| 1 Data Gains | Contracts, PostGIS, public grid and climate baselines, parcel connector, spatial join, read API |
| 2 Agents and UX | Orchestrator, four agents, map filters, three cooling scenarios, memo export |
| 3 Demand capture | Tenant and capital interest layer, powered-shell offer templates, offtake scenarios |

Epics: [epics-and-stories.md](../planning/epics-and-stories.md).
