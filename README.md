# demonstrator

For the demonstrator we manually set up a target pipeline, which gives us a concrete goal the [pipeline generator](https://github.com/thcarsten/toolchain-specification/tree/main/pipeline%20generator) should compile to. 
The scenario is the following: Our pipeline receives water levels from an [API-endpoint](https://dishacled-frontend.azurewebsites.net/). This data is enriched with the [LDIO workbench](https://openldes.org/) and forwarded to [RDF-Connect](https://rdf-connect.github.io/). In RDF-Connect, water levels are continuously checked against a fixed threshold, to detect flooding. In case of flooding, RDF-Connect forwards this information to [semantic.works](https://abb-vlaanderen.gitbook.io/abb/development/architecture/semantic-works-application-framework)-components. It does so by inserting triples to a triple store. An error-alert service is triggered by this placement of triples, causing an email to be send out to emergency services.
Concretely, the pipeline pipes data through the following components:

This api is the starting point: https://dishacled-frontend.azurewebsites.net/

| Component | Purpose |
| ----- | ----- |
| ldio:HttpInPoller | Fetches data from the API |
| ldio:RdfAdapter | Parses string to internal Linked Data representation |
| ldio:SparqlConstructTransformer | Enriches the data (to be discussed) |
| ldio:HttpOut | Sends data out via http |
| rdfc:HttpIn | Receives data via http |
| rdfc:thresholdMonitoringProcessor | Continuously checks flooding |
| rdfc:SparqlIngest | Inserts triples to semantic.works triplestore to indicate flooding |
| sw:loket-error-alert-service | Is alerted by flooding and creates email |
| sw:deliver-email-service | Sends out email to emergency services |


# commands

```
docker compose \\ 
    -f docker-compose.yml \\
    -f docker-compose-sw.yml \\
    -f docker-compose-sw.dev.yml \\
    up -d --build
```

```
curl -X POST localhost:9000/source-a -d@test.jsonld
```