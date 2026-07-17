# LDIO workbench — source-a enrichment pipeline

This folder configures an [LDIO](https://openldes.org/) (Linked Data Interactions Orchestrator) workbench that enriches water-level measurements from the DiSHACLed source-a API with the [SSN/SOSA](https://www.w3.org/TR/vocab-ssn/) vocabulary before handing them off to the RDF-Connect pipeline in `../RDFC/`.

## What it does

Every 10 seconds:

1. **`Ldio:HttpInPoller`** fetches JSON-LD from `https://dishacled-api.azurewebsites.net/api/v1/source-a/current`.
2. **`Ldio:JsonToLdAdapter`** parses the response into LDIO's internal RDF model. The API serves JSON-LD but advertises `Content-Type: application/json`, so `Ldio:RdfAdapter` (which needs a valid RDF MIME type) can't be used; this adapter accepts any content-type via `force-content-type: true` and re-applies the same JSON-LD context that the API embeds inline.
3. **`Ldio:SparqlConstructTransformer`** runs a SPARQL `CONSTRUCT` that remaps each `schema:PropertyValue` measurement onto `sosa:Observation` (with `sosa:madeBySensor`, `sosa:observedProperty`, `sosa:hasFeatureOfInterest`, `sosa:hasSimpleResult`, and a `qudt:QuantityValue` result carrying `qudt:unit unit:CentiM`).
4. **`Ldio:HttpOut`** POSTs the enriched graph as `application/ld+json` to `http://rdfc:9000/source-a`, where the RDF-Connect pipeline picks it up for threshold monitoring.

## Files

- `application.yml` — LDIO's Spring Boot framework config. Sole job: tell LDIO to auto-load pipeline files from `/ldio/pipelines`.
- `pipelines/config.yml` — the pipeline definition (input, transformer, output). Loaded on startup because it sits in the scanned directory.

Both are mounted into the `ldio-workbench` container by the top-level [`docker-compose.yml`](../docker-compose.yml).

## Running

From the repo root:

```
docker compose up -d ldio-workbench rdfc
```

LDIO exposes its admin/health endpoints on `http://localhost:8080/actuator/health`. Watch the pipeline in action with `docker compose logs -f ldio-workbench rdfc`.
