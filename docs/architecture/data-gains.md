# Data Gains architecture

**Status:** planning · first implementation tract  
**Depends on:** nothing in-repo yet  
**Serves:** GeoSite, GridPower, PolicyTax, ThermalCAPEX

Data Gains continuously ingests geospatial, interconnection, tariff, climate, water, and regulatory inputs, normalizes them, and serves them through one read API. Agents consume that API. They do not keep private copies of queues or parcel files.

## Source landscape

Cadence is the target poll interval after the extractor is proven against the live source. Field maps are written per source during the story that builds the extractor. Ideation column names are hints, not a committed mapping.

| Domain | Source | Access | Target cadence | v1 note |
|--------|--------|--------|----------------|---------|
| Interconnection | PJM Data Miner 2 planning queue | REST, subscription key | Daily | Key from the secrets plane |
| Interconnection | ERCOT GIS and large-load reports | MIS download (XLSX/CSV) | Weekly | Confirm the current report name and tab before coding a sheet name |
| Interconnection | MISO planning queue | Portal CSV/JSON | Weekly | |
| Interconnection | SPP, CAISO, NYISO | Public queue files | Weekly | After the first three extractors share one normalizer |
| Transmission geometry | Public electric-line and substation layers (HIFLD or the current successor) | GeoJSON or shapefile | Quarterly | Confirm redistribution terms before load |
| Retail power price | EIA Open Data v2, retail sales (EIA-861 / 861M) | REST, API key | Monthly | This is retail industrial average, not nodal LMP |
| Wholesale price | ISO LMP (later) | ISO market data | Hourly | Not in the first spatial milestone |
| Parcels | Licensed parcel API (Regrid or equivalent) | GeoJSON / vector tiles | On demand, refresh monthly for watched counties | Commercial license required |
| Elevation | USGS 3DEP | Raster or derived slope | On demand per candidate | Slope gate is max grade under 3% |
| Flood | FEMA NFHL | GIS | Semi-annual | 100- and 500-year overlap |
| Wetlands | USFWS NWI | GIS | Semi-annual | Overlap percent of parcel |
| Fiber | FCC National Broadband Map; commercial fiber later | GIS | Semi-annual | Long-haul dark fiber is a later licensed layer |
| Climate | NOAA ISD / NCEI hourly | CSV or bulk | Daily append | Dry-bulb and wet-bulb for free-cooling hours |
| Water | Utility industrial rate cards, state water-board notices | PDF/HTML, some GIS | Quarterly | Often unstructured; store the citation |
| Incentives | State economic-development statutes (for example Texas 312/380, Ohio CRA) | HTML and statute text | Bi-weekly | Structured rows with citation, then embeddings |
| Political risk | County and city agenda feeds for a seeded county list | HTML, PDF, RSS | Weekly | Flag phrases such as moratorium, water allocation, noise ordinance |

## Pipelines

Three ingress styles share one observation log.

```text
Structured poll          Spatial ETL                 Document ingest
PJM, EIA, MISO CSV       Lines, parcels, FEMA, NWI   Agendas, statutes
        │                        │                          │
        ▼                        ▼                          ▼
  QueueProject            Geometry layers            PolicyDocument
  TariffObservation       Constraint overlays        IncentiveRecord
        │                        │                          │
        └────────────────────────┴──────────────────────────┘
                                 │
                                 ▼
                    Observation log (append-only)
                                 │
                                 ▼
                    Feature assembler (parcel-keyed)
                                 │
                                 ▼
                    Read API for agents and the map
```

**Structured poll.** HTTP or file fetch on a schedule. Parse into a typed observation. Reject the batch if the header row drifted, and keep the previous good snapshot queryable.

**Spatial ETL.** Load geometries in EPSG:4326. Build GiST indexes. Derived overlays (slope class, flood fraction, wetland fraction) are tables keyed by parcel id, recomputed when the parcel or the constraint layer changes.

**Document ingest.** Store the source URL, the text chunk, and a structured extraction when the extractor is confident. A moratorium flag requires a quoted span and a URL. Unparsed documents stay searchable and do not flip a boolean by themselves.

Workers are Rust services, matching the Cargo-oriented repository. Schedules are configuration. There are no shell entrypoints.

## Storage

| Store | Holds | Engine |
|-------|--------|--------|
| Spatial | Parcel polygons, lines, substations, constraint layers | PostGIS |
| Analytical | Queue snapshots, EIA rates, hourly weather, free-cooling aggregates | Columnar files plus query tables (Parquet as the interchange; Postgres for the serving projection) |
| Knowledge | Statute chunks, agenda excerpts, embeddings | Postgres `vector` beside the policy tables |
| Observation log | Every successful fetch: source id, bytes hash, retrieved_at, parser version | Append-only table |

v1 runs these in one Postgres instance with PostGIS and vector extensions so spatial joins and policy citations share transactions. A separate warehouse can come later if NOAA hourly volume requires it. The serving projection remains the parcel-keyed feature.

## Logical schema

Names are the planning contract. Physical columns land with story DG-S2.1.

- `parcels` — APN, state, county, owner (when licensed), gross acres, boundary
- `transmission_lines` — voltage kV, owner, geometry
- `substations` — name, RTO if known, max voltage, geometry
- `parcel_constraints` — slope max percent, flood overlap, wetland overlap, residential-buffer violation, computed_at
- `queue_projects` — normalized interconnection rows (see below)
- `queue_snapshots` — raw file pointer and hash per fetch
- `tariff_observations` — utility, state, sector, cents per kWh, period, source
- `climate_stations` — station id, geometry
- `free_cooling_hours` — station, year, hours under the published wet-bulb threshold, model version
- `water_observations` — provider, rate, unit, moratorium flag, citation
- `incentive_records` — jurisdiction, exemption kind, years, percent, statute citation
- `policy_documents` — jurisdiction, url, chunk, embedding, risk labels
- `site_features` — assembled canonical feature, assembler version, input observation ids

### Normalized queue row

RTO files do not share columns. The normalizer emits:

| Field | Meaning |
|-------|---------|
| `queue_id` | Source queue or INR identifier |
| `rto` | PJM, ERCOT, MISO, SPP, CAISO, NYISO |
| `poi_name` | Substation or point of interconnection as published |
| `state`, `county` | When the source provides them |
| `capacity_mw` | Requested megawatts |
| `study_status` | Source status string, preserved |
| `status_class` | `active`, `withdrawn`, `suspended`, `in_service`, `unknown` |
| `target_in_service` | Date if parsed, else null |
| `fuel_or_load` | Generation type or `large_load` when the report distinguishes load |
| `retrieved_at` | Fetch time |
| `parser_version` | Code version that mapped the columns |

`status_class` is the only closed vocabulary. The raw status string is kept so a mapper change can be audited.

## Spatial join

The first join that makes a parcel useful:

1. Candidate parcels with gross acreage at or above a request threshold (default 100 acres).
2. Transmission lines at or above a voltage floor (default 230 kV) within a distance (default 5 km, user-overridable).
3. Nearest substation among those lines' endpoints or the substation point layer.
4. Constraint filters: slope, floodway, wetland fraction, residential buffer.
5. Attach the latest tariff for the utility territory when a territory polygon exists; otherwise attach the state industrial average and mark the match as `state_fallback`.
6. Attach the nearest climate station's free-cooling hours.
7. Attach incentive and water rows by state and county.

Months-to-power is **not** a single formula in v1. The API returns the inputs a later model will use: distance, voltage, count of active queue projects associated with the same POI name, and the source's stated in-service dates. A numeric `estimated_ttp_months` appears only after GridPower publishes a model version. Until then the field is omitted.

Headroom in megawatts is the same: publish it only with a stated method (`nameplate_minus_active_queue` or better). A bare integer with no method is forbidden.

## Canonical feature

Assembler output validates against [canonical-site.schema.json](../../contracts/canonical-site.schema.json).

Freshness is a list, not one timestamp. Grid data can be a day old while zoning is a month old. Clients display the oldest input that the score used.

```json
{
  "site_id": "SITE-US-PJM-AEP-089",
  "assembler_version": "0.1.0",
  "geometry": { "type": "Polygon", "coordinates": [] },
  "properties": {
    "jurisdiction": { "state": "OH", "county": "Licking", "rto_iso": "PJM" },
    "land_metrics": { "gross_acres": 340.5 },
    "grid_metrics": { "transmission_voltage_kv": 345 },
    "gaps": ["water_rate", "estimated_ttp_months"],
    "observations": [
      { "domain": "transmission", "source_id": "hifld-lines", "retrieved_at": "2026-10-01T00:00:00Z" }
    ]
  }
}
```

Financial benchmarks from the ideation sample (`projected_unlevered_irr_pct`, lease rate, capex per MW) belong to ThermalCAPEX in Wave 2. The schema reserves `financial_benchmarks` and requires `model_version` when the object is present.

## Read API

Serving is a small HTTP API the agents call. v1 operations:

| Operation | Behavior |
|-----------|----------|
| `GET /healthz` | Process and database reachable |
| `POST /sites:search` | Bounding box or RTO filter, min acres, min voltage, max distance, optional max cents/kWh when a tariff is attached |
| `GET /sites/{site_id}` | One assembled feature |
| `GET /sources` | Last successful fetch per source |

Search is spatial (`ST_DWithin`, `ST_Intersects`) plus attribute filters. Results include `gaps` so the UI can show an incomplete site instead of hiding it.

## Failure behavior

- A failed fetch does not delete the last good snapshot.
- A parser that sees an unknown header fails the batch and emits a drift error naming the header.
- Secrets (EIA key, PJM key, parcel license token) are injected at runtime. They never appear in the repository, in logs, or in the observation payload.
- Rate limits back off. They do not fall through to fixture rows presented as live data.

## Build order

The epic sequence in [epics-and-stories.md](../planning/epics-and-stories.md) is the build order: contract and database, public transmission geometry, one queue extractor, climate aggregate, parcel license, spatial search, then incentives. The map and agents wait until `POST /sites:search` returns real features for at least one watched county.
