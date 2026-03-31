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

:waterLevelsInCm a dcat:Dataset ;
  dcterms:conformsTo :waterLevelsInCmShape .

:waterLevelsInMm a dcat:Dataset ;
  dcterms:conformsTo :waterLevelsInMmShape .
```


### The Shape construction algorithm

This component utilizes the DiSHACLed shape-generation tool to ‘automatically’ create SHACL profiles for the datasets. By integrating this generator (or validator if creation fails), the system ensures that all data adheres to predefined structural constraints and semantic standards from the outset.

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

### LDIO Processor

Standardizes raw (real time only) API data into a unified, versioned LDES (Linked Data Event Stream), providing a reliable history for downstream services.

### RDF-Connect Service

Performs continuous threshold monitoring on the LDES stream. If a value exceeds a limit (e.g., >7m), it generates a Semantic Alert message.


### Semantic Works Service

Detects the alert and automatically triggers a notification service (e.g., an emergency email) sending out a couple of e-mails to emergency responders.

### Elody for alert viz

Elody is used as a clear UI for visualizing incoming flood-detection alerts in a more interactive way than just e-mails. It maps the context of each alert, including its source, thresholds, and underlying contracts, to make the data flow tangible and intuitive for stakeholders during demonstrations.



Elody for deployment: Acting as a "click-and-connect" interface, this component allows users to inspect SHACL contracts and orchestrate the connection between datasets and services to manage configurations & deployment.
