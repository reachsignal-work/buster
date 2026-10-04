# Buster

Agentic site-acquisition platform for AI data-center developers. Buster answers where power, water, cooling, incentives, and tenant demand line up, then packages that evidence for planners and capital.

Code name: **buster**.

## Start here

Wave 1 is the **Data Gains** layer: ingest, normalize, and serve the facts the agents will reason over. Product agents and the map UI sit on that layer and are planned, not built, in this transaction.

| Document | What it decides |
|----------|-----------------|
| [System architecture](docs/architecture/system.md) | Planes, agents, and what Data Gains does and does not compute |
| [Data Gains architecture](docs/architecture/data-gains.md) | Sources, pipelines, storage, and the canonical site contract |
| [Data Gains requirements](docs/requirements/data-gains.md) | Numbered requirements and acceptance |
| [Epics and user stories](docs/planning/epics-and-stories.md) | Wave 1 build order and Wave 2 backlog |
| [Issue graph](.aivcs/issuegraphs/data-gains.json) | Same plan in machine-readable form |
| [Canonical site schema](contracts/canonical-site.schema.json) | Versioned feature payload |

## Wave 1 outcome

A planner can ask, inside a bounding box, for parcels that sit near high-voltage transmission, and receive a canonical site record whose power, land, climate, water, and incentive fields each cite a source and a retrieval time.

Benchmark ranges in the ideation brief are hypotheses. They are replaced by observations once a pipeline lands.
