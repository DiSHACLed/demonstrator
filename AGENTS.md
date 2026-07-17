# DiSHACLed — project context

> **How to use this file.** Read it at the start of every session on the DiSHACLed
> project. Update the **Session log** and **Roadmap** at the end of each session so
> the next one can resume without archaeology. This is repo-scoped (versioned in
> git) and captures the *whole* DiSHACLed project, not only this `demonstrator`
> repository.
>
> Companion **private notes** live in the user-scope memory at
> `/memories/dishacled-project.md` (local paths, personal gotchas). AI assistants
> should consult both.

---

## 1. What is DiSHACLed?

DiSHACLed is a research project about compiling high-level, semantically-annotated
pipeline specifications down to concrete, running data-integration pipelines. The
demonstrator scenario is **flood detection**: water-level measurements from a
sensor API are semantically enriched, checked against thresholds, and — on
violation — trigger email alerts via a mu-semtech stack.

The project is intentionally **larger than any single repository**. It comprises
a target pipeline (this repo), a generator that compiles specifications into that
pipeline (a sibling repo), the source data API, a demo frontend, and supporting
research artefacts (semantic model, survey, catalog).

## 2. Repositories & services

| Kind | Name | Purpose |
| --- | --- | --- |
| Repo | [`DiSHACLed/demonstrator`](https://github.com/DiSHACLed/demonstrator) *(this repo)* | Manually-built target pipeline: LDIO + RDF-Connect + semantic.works, in one docker-compose stack |
| Repo | [`thcarsten/toolchain-specification`](https://github.com/thcarsten/toolchain-specification) | Pipeline generator, catalog (`pipeline generator/data/catalog.ttl`), semantic model, survey |
| Service | `dishacled-api.azurewebsites.net` | Mock sensor API. Endpoint used: `/api/v1/source-a/current` (JSON-LD body served with `Content-Type: application/json`) |
| Service | `dishacled-frontend.azurewebsites.net` | Demo frontend (Azure-hosted) |
| _TODO_ | Any other DiSHACLed org repos | *(fill in — user has flagged the project is larger than the two repos above)* |
| _TODO_ | Research paper / thesis draft | *(fill in — location, current status)* |

## 3. Current status (2026-07-17)

- ✅ **End-to-end pipeline verified working** across all three frameworks.
  Water-level readings flow from the source-a API through LDIO enrichment,
  RDF-Connect threshold monitoring, and into the semantic.works triple store,
  producing `oslc:Error` entities and composed `nmo:Email` triples.
  See [README.md](README.md) for the run recipe and verification queries.
- ✅ LDIO workbench + `Ldio:JsonToLdAdapter` + SSN/SOSA-mapping SPARQL CONSTRUCT
  in place ([`LDIO/`](LDIO/)).
- ✅ RDF-Connect pipeline retargeted from `schema:PropertyValue` to
  `sosa:Observation` and `sosa:hasSimpleResult` ([`RDFC/pipeline.ttl`](RDFC/pipeline.ttl)).
- ⚠️ `berichtencentrum-deliver-email-service` doesn't actually send SMTP —
  composed `nmo:Email` triples aren't linked to the mail folder the sender
  polls. Deployment gap in the semantic.works wiring, not a pipeline bug.
- ⚠️ `RDFC/pipeline.ttl` `tm:max` is `300 cm`, a demo placeholder — real
  water-level readings (~1500 cm) fire on every poll. Realistic thresholds
  needed before shipping the demo.

## 4. Architecture at a glance

```
source-a API  ──►  LDIO workbench  ──►  RDF-Connect  ──►  semantic.works
(JSON-LD)          poll + enrich          SDS + threshold        Virtuoso + delta
                   → SSN/SOSA             → oslc:Error           → error-alert
                                                                 → email
```

- **LDIO (Linked Data Interactions Orchestrator)** — Spring Boot; components
  named `Ldio:*` and configured in YAML.
- **RDF-Connect** — Node.js; components identified by IRIs (`rdfc:`, `tm:`,
  local `proc:`) and wired via a Turtle pipeline definition.
- **semantic.works (mu-semtech)** — a stack of small services communicating
  through a shared triple store; behaviour driven by SPARQL updates + delta
  rules.

Details, verification queries, and the full component table are in
[README.md](README.md); LDIO-specific pipeline details are in
[LDIO/README.md](LDIO/README.md).

## 5. Vocabulary cheat-sheet

- **LDIO** — Linked Data Interactions Orchestrator (Spring Boot). Ships with
  `Ldio:HttpInPoller`, `Ldio:HttpIn`, `Ldio:RdfAdapter`, `Ldio:JsonToLdAdapter`,
  `Ldio:SparqlConstructTransformer`, `Ldio:HttpOut`, …
- **RDF-Connect (RDFC)** — streaming pipeline framework. Runners run
  language-specific processors. We use `NodeRunner` here.
- **SDS** — Semantic Data Stream. RDFC's internal envelope format for streaming
  triples between processors, produced by `rdfc:Sdsify`.
- **SSN/SOSA** — W3C ontologies for sensor observations
  (`sosa:Observation`, `sosa:hasFeatureOfInterest`, `sosa:hasResult`,
  `sosa:hasSimpleResult`, `sosa:madeBySensor`, `sosa:observedProperty`).
- **QUDT** — quantity/unit ontology. We use `qudt:QuantityValue`,
  `qudt:numericValue`, `qudt:unit`, and `unit:CentiM` (canonical mapping of
  UN/CEFACT code `CMT`).
- **oslc:Error** — Open Services for Lifecycle Collaboration error resource;
  the shape ThresholdMonitor emits on violation.
- **mu-semtech / semantic.works** — microservice ecosystem where services
  communicate by publishing SPARQL updates to a shared store. `mu-identifier`
  is the front-door SPARQL gateway; `mu-delta-notifier` is the pub/sub bus.

## 6. Working with Copilot on this project

### Starting a session

1. Open **this** folder (`demonstrator/`) as your workspace so AGENTS.md is
   auto-loaded.
2. If your task also touches `toolchain-specification`, either add that folder
   as a workspace root (`File → Add Folder to Workspace`) or reference the
   sibling directory explicitly.
3. Tell the agent (any AI assistant): *"Continuing DiSHACLed work — read
   AGENTS.md and your memory."*
4. State the concrete goal for the session.

### During the session

- **Prefer editing over rewriting.** The pipeline is tightly linked across
  three frameworks; a small change in one place often has visible effects in
  another (see the SSN/SOSA switch that touched LDIO, RDFC, and the README).
- **Verify each hop** when changing the schema or thresholds — a broken
  `typeFilter` will silently drop everything downstream (no error, just no
  output). See the end-to-end verification recipe in
  [README.md](README.md#verifying-end-to-end).
- **When editing `RDFC/pipeline.ttl`**, remember the RDFC docker image is
  built from context; a plain `docker compose restart` won't pick up TTL
  changes. Use `docker compose up -d --build --force-recreate rdfc`.
- **When editing `LDIO/pipelines/*.yml`**, a `docker compose restart
  ldio-workbench` is enough — the file is bind-mounted read-only into the
  running container.

### Ending a session

Update this file, specifically:

1. **§3 Current status** — record what changed, what now works, what regressed.
2. **§7 Roadmap** — check off / delete anything done; add newly-discovered
   items.
3. **§8 Session log** — one short paragraph capturing the intent, the outcome,
   the affected files, and any commit SHAs.

Commit AGENTS.md alongside the code changes so its history mirrors the project's.

## 7. Roadmap & open items

- [ ] Set realistic `tm:min` / `tm:max` for the water-level threshold monitor
  (currently `0` – `300` cm; real readings are ~1500 cm).
- [ ] Wire the composed `nmo:Email` triples to the mail folder polled by
  `berichtencentrum-deliver-email-service`, and configure SMTP so the last
  hop actually delivers.
- [ ] Add the LDIO service definition (image, ports, volumes) to
  `thcarsten/toolchain-specification`'s catalog so the pipeline generator can
  emit an `ldio-workbench` stanza automatically. Requires the workbench + starter
  service pair or a refactor to LDIO Pattern A1 — see private note
  `/memories/dishacled-ldio-a1-refactor.md`.
- [ ] Decide whether to keep the three separate compose files or merge
  `docker-compose.yml` + `docker-compose-sw.yml` (leaving only `.dev.yml` as
  overlay).
- [ ] _TODO — fill in project-wide items beyond the demonstrator repo._

## 8. Session log

Newest entries first. Keep entries short — link to commits for detail.

### 2026-07-17 — LDIO workbench + end-to-end validation

- **Goal.** Add an LDIO workbench that polls the source-a API, enriches each
  measurement into SSN/SOSA, and forwards to RDF-Connect. Retarget the
  RDF-Connect pipeline from `schema:PropertyValue` to `sosa:Observation`.
- **Outcome.** Full three-framework pipeline verified end-to-end: 23+
  `oslc:Error` triples and 5+ composed `nmo:Email` entities produced in
  Virtuoso within minutes of `docker compose up`.
- **Files touched.** New `LDIO/application.yml`, `LDIO/pipelines/config.yml`,
  `LDIO/README.md`; patched `RDFC/pipeline.ttl` (3-line SSN/SOSA switch);
  extended `docker-compose.yml` with an `ldio-workbench` service; refreshed
  `README.md`.
- **Commits on `development`.** `5501d15`, `eff8701`.
- **Notes.** Discovered that the source-a API returns
  `Content-Type: application/json` for JSON-LD bodies, so `Ldio:RdfAdapter`
  refuses it; workaround is `Ldio:JsonToLdAdapter` with
  `force-content-type: true` + inline `@context`. Also switched the
  `qudt:QuantityValue` bnode to a deterministic IRI (`<...:.../result>`) to
  avoid CONSTRUCT fan-out duplicates.
