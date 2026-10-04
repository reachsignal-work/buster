# Epics and user stories

**Graph:** [`.aivcs/issuegraphs/data-gains.json`](../../.aivcs/issuegraphs/data-gains.json)  
**Requirements:** [data-gains.md](../requirements/data-gains.md)

Wave 1 is the build order. A story is ready when every story it depends on is done. Wave 2 and Wave 3 are sequenced so agent work starts only after `POST /sites:search` is real.

## Wave 1 — Data Gains

### DG-E1 Canonical contracts

Lock the feature schema and the observation envelopes before extractors multiply formats.

| Story | Statement | Acceptance | Requires |
|-------|-----------|------------|----------|
| DG-S1.1 | As an agent author, I want a versioned canonical site schema so every consumer reads the same feature. | Schema file validates the examples in `contracts/`. `gaps` and `observations` are required. | DG-REQ-1 |
| DG-S1.2 | As a pipeline author, I want typed envelopes for queue rows, tariffs, climate aggregates, water, and incentives. | Each envelope has source id, retrieved_at, and parser version. Queue `status_class` is the closed set in the architecture doc. | DG-REQ-5, DG-REQ-7 |
| DG-S1.3 | As an auditor, I want derived fields to name their method. | Schema rejects `estimated_ttp_months`, headroom MW, net acres, and any financial benchmark that lacks `method` or `model_version`. | DG-REQ-8, DG-REQ-10 |

### DG-E2 Spatial database

Stand up PostGIS and the public geometry baseline. No parcel vendor yet.

| Story | Statement | Acceptance | Requires |
|-------|-----------|------------|----------|
| DG-S2.1 | As an operator, I want the logical tables and GiST indexes from the architecture doc. | Migration applies on empty Postgres. Extensions: `postgis`. `vector` may wait for DG-E7. | DG-E1 |
| DG-S2.2 | As a planner, I want transmission lines and substations loaded from a public extract with a recorded license note. | Reload is idempotent (DG-REQ-19). Voltage is numeric kV. Features without voltage are stored and excluded from the ≥230 kV filter. | DG-S2.1, DG-REQ-20 |
| DG-S2.3 | As a planner, I want flood and wetland polygons available for overlay. | NFHL and NWI fixtures intersect a sample polygon and return overlap ratios. | DG-S2.1 |
| DG-S2.4 | As a planner, I want slope class from a DEM once parcels exist. | Depends on a parcel to measure. Story delivers the raster sampler and the 3% flag against a synthetic DEM (DG-REQ-11). | DG-S2.1 |

### DG-E3 Interconnection queues

One normalizer, three sources, history kept.

| Story | Statement | Acceptance | Requires |
|-------|-----------|------------|----------|
| DG-S3.1 | As a grid analyst, I want a PJM queue fetch mapped into queue rows. | Recorded Data Miner payload passes DG-REQ-5 and DG-REQ-6. Subscription key is environmental. | DG-S1.2 |
| DG-S3.2 | As a grid analyst, I want the current ERCOT large-load or GIS workbook mapped the same way. | Sheet and column map is written next to the parser after a live file is inspected. Drift fails closed. | DG-S1.2 |
| DG-S3.3 | As a grid analyst, I want the MISO planning queue on the same envelope. | Same acceptance as PJM, MISO fixture. | DG-S1.2 |
| DG-S3.4 | As an operator, I want `GET /sources` to show the last queue fetch. | DG-REQ-18 for these three sources. | DG-S3.1 |

SPP and CAISO reuse the normalizer as DG-S3.5 after the first three are stable. They are in the graph and not on the critical path.

### DG-E4 Retail tariffs

| Story | Statement | Acceptance | Requires |
|-------|-----------|------------|----------|
| DG-S4.1 | As a planner, I want monthly industrial retail averages by utility and state from EIA. | Field is `retail_industrial_average_cents_kwh` (DG-REQ-7). Failed fetch leaves the prior month (DG-REQ-2). | DG-S1.2 |

### DG-E5 Parcels and constraints

| Story | Statement | Acceptance | Requires |
|-------|-----------|------------|----------|
| DG-S5.1 | As a planner, I want parcel polygons for a watched county from the licensed API. | DG-REQ-9 and DG-REQ-20. Watched-county list is config, starting with one Midwest county and one ERCOT county. | DG-S2.1 |
| DG-S5.2 | As a planner, I want net buildable acres after wetland and floodway subtraction. | DG-REQ-10 on the county fixture. | DG-S5.1, DG-S2.3 |
| DG-S5.3 | As a planner, I want a residential-buffer flag at a configurable distance. | DG-REQ-12. Residential layer source is named in the story's design note before implementation; if no public layer is licensed, the flag stays in `gaps` rather than a guess. | DG-S5.1 |
| DG-S5.4 | As a planner, I want slope evaluated on those parcels. | DG-S2.4 wired to DG-S5.1 parcels. | DG-S2.4, DG-S5.1 |

### DG-E6 Climate and water

| Story | Statement | Acceptance | Requires |
|-------|-----------|------------|----------|
| DG-S6.1 | As a thermal planner, I want annual free-cooling hours for stations covering watched counties. | DG-REQ-13. Threshold is configuration, default documented in the model note. | DG-S1.2 |
| DG-S6.2 | As a planner, I want industrial water rate and moratorium status with citations for watched utilities. | DG-REQ-14. v1 may be a curated, cited table plus a loader, with scrapers following once the record shape is stable. | DG-S1.2 |

### DG-E7 Incentives and agendas

| Story | Statement | Acceptance | Requires |
|-------|-----------|------------|----------|
| DG-S7.1 | As a capital planner, I want structured incentive rows for the watched states. | DG-REQ-15 for Ohio and Texas first, then Indiana, Iowa, Alabama. | DG-S1.2 |
| DG-S7.2 | As a risk analyst, I want weekly agenda capture for a seed county list. | DG-REQ-16. Seed list lives in config and starts with counties already in the watch set. Embeddings are optional in this story; the flag, chunk, and URL are not. | DG-S7.1 |

### DG-E8 Assemble and serve

This is the Wave 1 exit.

| Story | Statement | Acceptance | Requires |
|-------|-----------|------------|----------|
| DG-S8.1 | As an agent, I want a spatial query that links parcels to lines and substations. | DG-REQ-3. SQL is covered by a test, not only by a hand-run comment. | DG-S2.2, DG-S5.1 |
| DG-S8.2 | As an agent, I want the assembler to attach tariff, climate, water, and incentive observations and to list gaps. | One fixture parcel produces a schema-valid feature. Missing water yields `gaps: ["water"]` and no invented rate. | DG-S8.1, DG-S4.1, DG-S6.1, DG-S7.1 |
| DG-S8.3 | As a client, I want `POST /sites:search` and `GET /sites/{id}` plus health. | Filters: bbox, min acres, min kV, max distance, optional max retail ¢/kWh. DG-REQ-4 default: no overall score. | DG-S8.2 |
| DG-S8.4 | As a product owner, I want a documented score-model hook that stays off by default. | Enabling `score-0.1.0` returns `overall_buster_score` and the version. Weights are a config file in-repo with a comment that they are provisional. | DG-S8.3, DG-S1.3 |

## Wave 2 — Agents and experience

Blocked by DG-S8.3. These stories define the product the ideation brief asked for. They are not scheduled inside Wave 1.

### APP-E1 Orchestrator and specialists

| Story | Statement | Acceptance |
|-------|-----------|------------|
| APP-S1.1 | As a user, I want Buster-Core to fan a search out to the four agents and merge one shortlist. | Each agent calls Data Gains. None fetches an RTO site directly. |
| APP-S1.2 | As a user, I want GeoSite to explain acreage, slope, flood, wetlands, and buffer in sentences tied to field names. | Every number in the narrative appears in the feature JSON. |
| APP-S1.3 | As a user, I want GridPower to explain voltage, distance, queue pressure, and retail price, and to say when months-to-power is not yet modeled. | Copy matches DG-REQ-8. |
| APP-S1.4 | As a user, I want PolicyTax to cite the statute or agenda URL for every exemption or moratorium claim. | No incentive sentence without `citation`. |
| APP-S1.5 | As a user, I want ThermalCAPEX to compare adiabatic air, direct-to-chip dry cooler, and immersion using the same weather station. | Three scenarios, one station id, PUE/WUE labeled with model version. |

### APP-E2 Map discovery

| Story | Statement | Acceptance |
|-------|-----------|------------|
| APP-S2.1 | As a user, I want to draw a box or pick an RTO footprint and see parcels that pass my filters. | Filters map onto `POST /sites:search`. |
| APP-S2.2 | As a user, I want sliders for minimum MW headroom (when a method exists), maximum retail ¢/kWh, and maximum months-to-power (when a model exists). | Unavailable filters are visible and disabled, with the gap name. |

### APP-E3 Deal memo

| Story | Statement | Acceptance |
|-------|-----------|------------|
| APP-S3.1 | As a user, I want a site export that includes the map extent, constraint summary, queue evidence, and tax citations. | Export contains the observation ids. It does not contain uncitable IRR. |
| APP-S3.2 | As a user, I want capex, lease, and IRR only after a finance model version is selected. | Same rule as DG-S1.3. |

## Wave 3 — Demand capture

These answer "where is the interest" and "how a new build captures it." They need Wave 1 facts and a Wave 2 memo, plus data Buster does not have yet.

| Story | Statement | Acceptance |
|-------|-----------|------------|
| APP-S4.1 | As a strategist, I want a sourced layer of announced hyperscale and infrastructure-fund campuses outside Northern Virginia and Silicon Valley. | Each point has a public source URL and an announced MW if published. |
| APP-S4.2 | As a developer, I want a powered-shell offer sketch: block size (20 MW to 100 MW), cooling topology, and the site's months-to-power inputs. | Sketch refuses to claim an 18-month delivery without an interconnection evidence record. |
| APP-S4.3 | As a developer, I want benchmark cards for the Pacific Northwest, upper Midwest, TVA, ERCOT, and the Nordics built only from ingested observations. | A card with no observation renders as "no coverage," not as the ideation range. |

## Suggested first sprint

1. DG-S1.1, DG-S1.2, DG-S1.3 — schema and envelopes  
2. DG-S2.1, DG-S2.2 — PostGIS and transmission geometry  
3. DG-S3.1 — PJM, because it is a JSON API and unblocks the normalizer  
4. DG-S8.1 — spatial link, even before tariffs, using parcels from a small fixture if the license is not signed yet  

DG-S5.1 starts in parallel once the parcel vendor is chosen. Until that contract is signed, spatial tests use generated polygons, and the README states that.
