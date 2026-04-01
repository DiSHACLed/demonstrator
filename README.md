# demonstrator

This repo sketches the semantic descriptions of the different parts in the demonstrator pipeline.

The demonstrator pipeline shows flood alerting system with multiple independent org’s & technolgies
Sensor → Data Catalog → Data Platform → Algorithm → Alerting → Dashboard

## Components

### The Catalog

The central repository. It doesn't just list URLs; it stores SHACL "Contracts" for every component. These define what a dataset holds or what a service requires (e.g., "I require a water-level input in CM"). Can be used to find the dataset(s) & services via search.

Datasets linked with shapes via dcterms:conformsTo (other options are possible according to [the specification](https://dishacled.github.io/discovery-specification/), does not matter if the discovery algorithm is used):

```
:dishacled-catalogue
  a dcat:Catalog;
  dct:title "Datasets and processors used in DiSHACLed demonstrator.";
  dcat:dataset :waterLevelsInCm, :waterLevelsInMm .

# Source dataset published by the mock JSON-LD APIs, but could also be others like LDES or SPARQL
:waterLevelsInCm a dcat:Dataset ;
  dcterms:conformsTo :waterLevelsInCmShape .

:waterLevelsInMm a dcat:Dataset ;
  dcterms:conformsTo :waterLevelsInMmShape .

# The mock JSON-LD APIs
:rawDataStreamInCm a dcat:DataService ;
  dcat:servesdataset :waterLevelsInCm ;
  # Optional, because dataset already provides this shape
  dcat:qualifiedRelation [
        a dcat:Relationship;
        dcat:hadRole :outputShape ;
        dcterms:relation :waterLevelsInCmShape
    ] .

:rawDataStreamInMm a dcat:Service ;
  dcat:servesdataset :waterLevelsInMm ;
  # Optional, because dataset already provides this shape
  dcat:qualifiedRelation [
        a dcat:Relationship;
        dcat:hadRole :outputShape ;
        dcterms:relation :waterLevelsInMmShape
    ] .

# Generic processor / pipeline components where config still needs to be configured
:ldioHttpInPoller a :PipelineComponent ;
  dcat:qualifiedRelation [
        a dcat:Relationship;
        dcat:hadRole :configShape ;
        dcterms:relation :ldioHttpInPollerConfigShape .
    ] .



# Instances of processor / pipeline components where the config is defined and input / output shape is clear

```


### The Shape construction algorithm

This component utilizes the DiSHACLed shape-generation tool to ‘automatically’ create SHACL profiles for the datasets. By integrating this generator (or validator if creation fails), the system ensures that all data adheres to predefined structural constraints and semantic standards from the outset.

The SHACL profile of the water levels in cm:

```
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dcterms: <http://purl.org/dc/terms/> .
@prefix ns1: <https://data.vlaanderen.be/ns/waterkwaliteit#> .
@prefix prov: <http://www.w3.org/ns/prov#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix schema: <https://schema.org/> .
@prefix sosa: <http://www.w3.org/ns/sosa/> .
@prefix time: <http://www.w3.org/2006/time#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
@prefix : <http://dishacled.org/> .

:waterLevelsInCmShape
    a sh:NodeShape ;
    sh:targetClass sosa:Observation ;

    sh:property [
        sh:path sosa:hasResult ;
        sh:node [
          a sh:NodeShape ;
          sh:targetClass schema:QuantitativeValue ;
      
          sh:property [
              sh:path schema:value ;
              sh:datatype xsd:decimal ;
              sh:minCount 1 ;
              sh:maxCount 1 ;
          ] ;
      
          sh:property [
              sh:path schema:unitText ;
              sh:datatype xsd:string ;
              sh:minCount 1 ;
              sh:maxCount 1 ;
          ] ;
      
          sh:property [
              sh:path schema:unitCode ;
              sh:nodeKind sh:IRI ;
              # <http://qudt.org/vocab/unit/MilliM> in case of :waterLevelsInMmShape
              sh:defaultValue <http://qudt.org/vocab/unit/CentiM> ; 
              sh:minCount 1 ;
              sh:maxCount 1 ;
          ] ;
        ] ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path sosa:observedProperty ;
        sh:dataType xsd:IRI ;        
        sh:minCount 1 ;
        sh:maxCount 1 ;
    ] ;
    sh:property [
        sh:path sosa:phenomenonTime ;
        sh:dataType xsd:dateTime ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
    ] .
```

### Raw Data Streams

Mocked JSON-LD API endpoints producing (mocked) real-time measurements of water levels. Source A: Publishes values in centimeters (cm). Source B: Publishes values in millimeters (mm). Nice to have: A visual slider to manually increase or decrease data produced at the source to induce a "flood" during the demo. Data adheres to realistic dataset, but is not produced by real sensor.

```
{
  "@context": {
    "sosa": "http://www.w3.org/ns/sosa/",
    "schema": "https://schema.org/",
    "xsd": "http://www.w3.org/2001/XMLSchema#"
  },
  "@id": "http://example.org/observation/1",
  "@type": "sosa:Observation",

  "sosa:hasResult": {
    "@type": "schema:QuantitativeValue",
    "schema:value": {
      "@value": 123.4,
      "@type": "xsd:decimal"
    },
    "schema:unitText": "cm",
    "schema:unitCode": {
      "@id": "http://qudt.org/vocab/unit/CentiM"
    }
  },

  "sosa:observedProperty": {
    "@id": "http://example.org/property/waterLevel"
  },

  "sosa:phenomenonTime": {
    "@value": "2026-04-01T10:15:00Z",
    "@type": "xsd:dateTime"
  }
}
```

We use the qualified relationship for services, because we want to differentiate between input and output. Here, the API only has an output shape.

### LDIO Service

Standardizes raw (real time only) API data into a unified, versioned LDES (Linked Data Event Stream), providing a reliable history for downstream services.

A `prov:generatedAtTime` and `dct:isVersionOf` is added to enrich the sensor observation as an LDES member:
```
{
  "@context": {
    "sosa": "http://www.w3.org/ns/sosa/",
    "schema": "https://schema.org/",
    "xsd": "http://www.w3.org/2001/XMLSchema#",
    "prov": "http://www.w3.org/ns/prov#",
    "dct": "http://purl.org/dc/terms/",
  },
  "@id": "http://example.org/observation/1/2026-04-01T10:17:00Z",
  "@type": "sosa:Observation",
  "prov:generatedAtTime": {
    "@value": "2026-04-01T10:17:00Z",
    "@type": "xsd:dateTime"
  },
  "dct:isVersionOf": "http://example.org/observation/1",
  "sosa:hasResult": {
    "@type": "schema:QuantitativeValue",
    "schema:value": {
      "@value": 123.4,
      "@type": "xsd:decimal"
    },
    "schema:unitText": "cm",
    "schema:unitCode": {
      "@id": "http://qudt.org/vocab/unit/CentiM"
    }
  },
  "sosa:observedProperty": {
    "@id": "http://example.org/property/waterLevel"
  },
  "sosa:phenomenonTime": {
    "@value": "2026-04-01T10:15:00Z",
    "@type": "xsd:dateTime"
  }
}
```

#### LDIO HttpInPoller

The config of the LDIO HttpInPoller will have a semantic description in the pipeline definition:

```
:demodioHttpInPoller a :PipelineStep ;
  :toBeCarriedOutByProcessor :ldioHttpInPoller ;
  p-plan:hasInputVar [
        a tc:Config;
        tc:embedded
        [
          :url "http://path-to-mock-api"
        ],
        [
          :interval "PT5M"
        ]
    ] .
```

#### SPARQL CONSTRUCT transformer

```
:demoLdioSparqlConstructTransformer a :PipelineStep ;
  :toBeCarriedOutByProcessor :ldioSparqlConstructTransformer ;
  p-plan:hasInputVar [
        a tc:Config;
        tc:embedded
        [
          :query """
            PREFIX dct: <http://purl.org/dc/terms/> .
            PREFIX prov: <http://www.w3.org/ns/prov#> .
            CONSTRUCT {
              ?versionedS ?p ?o ;
                  prov:generatedAtTime ?generatedAtTime ;
                  dct:isVersionOf ?s .
            } WHERE {
              ?s ?p ?o .
              BIND(URI(CONCAT(STR(?s), '/', STR(?now))) as ?versionedS)
              BIND (NOW() as ?generatedAtTime)
            }
          """
        ]
    ] .
```

#### HTTP Out

```
:demoLdioHttpOut a :PipelineStep ;
  :toBeCarriedOutByProcessor :ldioHttpOut ;
  p-plan:hasInputVar [
        a tc:Config;
        tc:embedded
        [
          :url "http://path-to-rdf-connect-channel"
        ]
    ] .
```

### RDF-Connect Service

Performs continuous threshold monitoring on the LDES stream. If a value exceeds a limit (e.g., >7m), it generates a Semantic Alert message.

```

```

### Semantic Works Service

Detects the alert and automatically triggers a notification service (e.g., an emergency email) sending out a couple of e-mails to emergency responders.

### Elody for alert viz

Elody is used as a clear UI for visualizing incoming flood-detection alerts in a more interactive way than just e-mails. It maps the context of each alert, including its source, thresholds, and underlying contracts, to make the data flow tangible and intuitive for stakeholders during demonstrations.



Elody for deployment: Acting as a "click-and-connect" interface, this component allows users to inspect SHACL contracts and orchestrate the connection between datasets and services to manage configurations & deployment.

# Config versus data shape validation

Currently, the focus of the pipeline generator and specification lies in the description of the configuration parameters of a pipeline component.
However, the demonstrator describes a use case where we need data shape validation.

## Example of instance pipeline component

For example, SSN/SOSA input -> LDES member SPARQL construct transformer
