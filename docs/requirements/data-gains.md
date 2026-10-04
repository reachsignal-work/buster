# Data Gains requirements

**Status:** draft for Wave 1  
**Architecture:** [data-gains.md](../architecture/data-gains.md)

Requirements use `DG-REQ-n`. Stories in the epic plan trace to these ids. A requirement is met only when its acceptance checks pass against a live or recorded source fixture that was captured from the real file format, not from a hand-written happy-path row.

## Product

### DG-REQ-1 Provenance

Every served metric that originates outside Buster includes `source_id`, `retrieved_at`, and the parser or assembler version that produced it.

**Acceptance:** `GET /sites/{id}` for a seeded parcel returns those three fields for transmission distance and for any tariff attached. A metric with no observation is listed in `gaps` and is omitted from the body.

### DG-REQ-2 No silent defaults

The service must not substitute ideation benchmark ranges when a fetch fails or a county is uncovered.

**Acceptance:** With the EIA extractor disabled, search results omit `industrial_power_cost_cents_kwh` and list `tariff` in `gaps`. They do not return 4.2–7.2 cent placeholders.

### DG-REQ-3 Search

A client can submit a bounding box and receive parcels that intersect the box, meet a minimum acreage, and lie within a given distance of a transmission line at or above a given voltage.

**Acceptance:** For a fixture county with one 300-acre parcel 2 km from a 345 kV line and one 300-acre parcel 20 km from any line, a query with max distance 5 km and min voltage 230 returns only the first parcel.

### DG-REQ-4 Rank inputs, not a hidden score

v1 search returns the inputs used for ranking (distance, voltage, acres, constraint fractions, tariff when present, free-cooling hours when present). A single `overall_buster_score` is returned only when a score-model version is configured. The response names that version.

**Acceptance:** Default deployment has no score model. Responses have no `overall_buster_score`. Enabling model `score-0.1.0` adds the number and the version string.

## Grid

### DG-REQ-5 Queue normalization

PJM, ERCOT, and MISO extracts each map into the normalized queue row. Raw status text is stored. `status_class` is one of `active`, `withdrawn`, `suspended`, `in_service`, `unknown`.

**Acceptance:** A recorded PJM payload, a recorded ERCOT workbook, and a recorded MISO file each produce rows. A renamed column fails the batch with a drift error and leaves the previous snapshot in place.

### DG-REQ-6 Queue history

Each successful queue fetch is addressable by source, retrieval time, and content hash.

**Acceptance:** Two fetches of an unchanged file share a hash and do not duplicate project identity. A changed megawatt value inserts a new observation and keeps the prior one.

### DG-REQ-7 Retail versus wholesale

Industrial price from EIA is labeled `retail_industrial_average`. The field name and the API must not call it a wholesale or LMP price.

**Acceptance:** Schema and API examples use `retail_industrial_average_cents_kwh`. No served field is named as wholesale until an LMP extractor exists.

### DG-REQ-8 Time-to-power honesty

`estimated_ttp_months` is absent until a documented model version computes it from stored inputs.

**Acceptance:** After queues and substations load, and before any TTP model is configured, site features include queue counts and distance and omit `estimated_ttp_months`.

## Land and environment

### DG-REQ-9 Parcel geometry

Licensed parcel polygons are stored in EPSG:4326 with APN, state, and county.

**Acceptance:** A parcel inserted from the vendor fixture returns the same APN and a valid polygon. Area used for `gross_acres` is computed geodetically and stored, not typed in by hand.

### DG-REQ-10 Buildable acreage

Net buildable acreage subtracts mapped floodway and wetland overlap from gross acreage, and records the method version.

**Acceptance:** A parcel fixture that is 10% wetland reports net acres lower than gross by that fraction, within a stated tolerance, and records `buildable-0.1.0`.

### DG-REQ-11 Slope gate

Where a 3DEP-derived slope raster covers the parcel, `topography_slope_max_pct` is the maximum slope on the parcel. Parcels at or above 3% are flagged `slope_exceeds_gate`.

**Acceptance:** A synthetic DEM fixture with a known 2% and a known 5% parcel flags only the 5% parcel.

### DG-REQ-12 Residential buffer

A parcel with a dense residential polygon within 1,000 ft of its boundary is flagged `residential_buffer_conflict`. The distance threshold is a parameter.

**Acceptance:** Fixtures at 800 ft and 1,500 ft flag only the 800 ft case when the parameter is 1,000 ft.

## Climate, water, incentives

### DG-REQ-13 Free-cooling hours

For each climate station in the watched set, the pipeline stores annual hours at or below a configured wet-bulb threshold, with station id, year, and threshold.

**Acceptance:** A one-year hourly fixture with a known count of hours below 50°F wet-bulb matches that count. The site feature points at the station id used.

### DG-REQ-14 Water

A water observation includes provider name, rate, unit, currency, the as-of date printed on the tariff, and a URL or document hash. `water_moratorium_active` is true only with a citation URL and an excerpt.

**Acceptance:** A rate card fixture loads the numeric rate and URL. A moratorium boolean without a URL is rejected at write time.

### DG-REQ-15 Incentives

An incentive record includes jurisdiction, incentive kind (`sales_tax_exemption`, `pilot`, `property_abatement`, `other`), a statute or program citation, and optional term years and percent. Unknown kinds use `other` plus the source label.

**Acceptance:** Ohio and Texas fixture pages each produce at least one record whose citation string is non-empty. The assembler attaches the record by state and county when both match, otherwise by state, and sets `incentive_match` to `county` or `state`.

### DG-REQ-16 Agenda watch

For each county in the seed list, a weekly job records newly seen agenda documents. A risk label is applied only when a pattern matches and the chunk plus URL are stored.

**Acceptance:** A fixture agenda containing "data center moratorium" creates a policy chunk with `moratorium_risk_flag` true. An agenda with no match is stored with the flag false.

## Platform

### DG-REQ-17 Secrets

API keys and license tokens are read from the environment or a secrets mount. They are not in source, fixtures, or info logs.

**Acceptance:** Repository search finds no key material. A request log line for an upstream call does not contain the credential header.

### DG-REQ-18 Health and freshness catalog

`GET /sources` lists each configured source, last success time, last error, and parser version.

**Acceptance:** After one successful EIA fetch and one forced parser failure, the catalog shows both the success timestamp and the error string.

### DG-REQ-19 Idempotent reload

Re-running a spatial load for an unchanged file does not duplicate geometries.

**Acceptance:** Loading the same substation extract twice leaves one row per source feature id.

### DG-REQ-20 Licensed redistribution

Parcel vendor payloads and any restricted grid extracts are not committed to the repository. Fixtures used in tests are either generated or small excerpts allowed by the license, noted in the fixture README.

**Acceptance:** CI fixtures directory contains a notice per third-party file. Full nationwide parcel dumps are absent.

## Explicitly deferred

These are real product needs and are not Wave 1 acceptance gates:

- Nodal LMP and congestion
- FERC CEII power-flow cases
- Behind-the-meter generation siting
- Nordic and Iberian parcel search
- Sovereign-capital "spore" interest tracking
- IRR, lease rate, and capex-per-MW models
- PDF deal-memo export
- Natural-language copilot

They are tracked as Wave 2 and Wave 3 stories so the data model does not block them.
