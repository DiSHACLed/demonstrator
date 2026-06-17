# demonstrator

This repo sketches the semantic descriptions of the different parts in the demonstrator pipeline.

The demonstrator pipeline shows flood alerting system with multiple independent org’s & technolgies
Sensor → Data Catalog → Data Platform → Algorithm → Alerting → Dashboard

## Components

Prefixes used throughout this document:

```
@prefix : <http://example.org/> .
@prefix tcs: <https://w3id.org/tcs#> .
@prefix dcat: <http://www.w3.org/ns/dcat#> .
@prefix dct: <http://purl.org/dc/terms/> .
@prefix dcterms: <http://purl.org/dc/terms/> .
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix p-plan: <http://purl.org/net/p-plan#> .
@prefix rdfc: <https://w3id.org/rdf-connect#> .
@prefix mu: <http://mu.semte.ch/vocabularies/core/> .
@prefix schema: <https://schema.org/> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .
@prefix prov: <http://www.w3.org/ns/prov#> .
```

### The Catalog

The central repository. It doesn't just list URLs; it stores SHACL "Contracts" for every component. These define what a dataset holds or what a service requires (e.g., "I require a water-level input in CM"). Can be used to find the dataset(s) & services via search.

Datasets linked with shapes via dcterms:conformsTo (other options are possible according to [the specification](https://dishacled.github.io/discovery-specification/), does not matter if the discovery algorithm is used):

- [ ] Update https://dishacled-api.azurewebsites.net/api/v1/catalog with the services and shapes of datasets and services
- [ ] Make 1 shape for source A dataset and 1 shape for source B dataset instead of shapes on dataset and service level
- [ ] Is the assumption that pipeline component = dcat service accurate?

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
:ldioHttpInPoller a tcs:PipelineComponent ;
  dcat:qualifiedRelation [
        a dcat:Relationship;
        dcat:hadRole :configShape ;
        dcterms:relation :ldioHttpInPollerConfigShape .
    ] .
# Shape of the config of LdioHttpInPoller
:ldioHttpInPollerConfigShape a sh:NodeShape ;
  sh:target [
            a sh:SPARQLTarget ;
            sh:prefixes :prefixes ;
            sh:select """
                    SELECT ?this
                    WHERE {
                        ?step prov:specializationOf :ldioHttpInPoller .
                        ?step p-plan:hasInputVar ?this .
                        ?this a tcs:Config .
                        }
                  """ ;
              ] ;
        sh:property [
            sh:path tcs:embedded ;
            sh:node [
                a sh:NodeShape ;
                sh:property [
                  sh:path :url ;
                      sh:datatype xsd:string ;
                      sh:minCount 0 ;
                      sh:maxCount 1 ;
                      sh:message "URL may have max one value of type string." ;
                ], [
                  sh:path :interval ;
                      sh:datatype xsd:duration ;
                      sh:minCount 0 ;
                      sh:maxCount 1 ;
                      sh:message "Interval may have max one value of type xsd:duration." ;
                ] ;
            ] ;
        ] .

# This deviates from how RDF Connect currently describe processors
:thresholdMonitoringProcessor a tcs:PipelineComponent , rdfc:ThresholdMonitoringProcessor ;
  dcat:qualifiedRelation [
          a dcat:Relationship;
          dcat:hadRole :configShape ;
          dcterms:relation :thresholdMonitoringProcessorShape .
      ] ;
  dct:requires rdfc:NodeRunner .

:thresholdMonitoringProcessorShape a sh:NodeShape ;
  sh:target [
            a sh:SPARQLTarget ;
            sh:prefixes :prefixes ;
            sh:select """
                    SELECT ?this
                    WHERE {
                        ?step prov:specializationOf :thresholdMonitoringProcessor .
                        ?step p-plan:hasInputVar ?this .
                        ?this a tcs:Config .
                        }
                  """ ;
              ] ;
        sh:property [
            sh:path tcs:embedded ;
            sh:node [
                a sh:NodeShape ;
                sh:property [
                  sh:path :value ;
                      sh:datatype xsd:decimal ;
                      sh:minCount 1 ;
                      sh:maxCount 1 ;
                      sh:message "Threshold value must be provided of type decimal." ;
                ], [
                  sh:path :unit ;
                      sh:datatype xsd:IRI ;
                      sh:minCount 1 ;
                      sh:maxCount 1 ;
                      sh:message "Unit must be provided as URI." ;
                ] ;
            ] ;
        ] .

rdfc:NodeRunner a tcs:PipelineComponent;
    rdfs:label "Javascript Node Runner" ; 
    dct:requires rdfc:Orchestrator ;
    dcterms:conformsTo :NodeRunnerConfigShape .
 
rdfc:Orchestrator a tcs:PipelineComponent ;
    rdfs:label "RDF Connect Orchestrator" ;
    tcs:hasDefaultConfig [
        a tcs:Config, tcs:MicroServiceConfig ;
        tcs:literal """
  rdf-connect:
    container_name: rdf-connect
    image: rdf-connect:latest
    build: ../../resources/rdfc-docker
    volumes:
      - ./rdfc_pipeline.ttl:/workspace/pipeline/pipeline.ttl:ro
    environment:
      LOG_LEVEL: debug
    command: npx rdfc /workspace/pipeline/pipeline.ttl
"""
] .

# The processors of Semantic.Workshave shapes for input / output data as described [here](https://github.com/DiSHACLed/discovery-specification/blob/main/20250422142901-describing_microservices.md), see below this document
:loketErrorAlertProcessor a :PipelineComponent ;
  dct:requires :loketErrorAlertService .

:loketErrorAlertService a :PipelineComponent, mu:Microservice, dcat:Service ;
  rdfs:label "Service that is responsible for sending out alerts when errors are inserted in the store. Build using the MU Javascript template.";
  dcat:accessUrl <https://github.com/lblod/loket-error-alert-service> ;
  dcterms:conformsTo :loketErrorAlertServiceShape .

:loketErrorAlertServiceShape a sh:NodeShape ;
  sh:target [
            a sh:SPARQLTarget ;
            sh:prefixes :prefixes ;
            sh:select """
                    SELECT ?this
                    WHERE {
                        ?step prov:specializationOf :loketErrorAlertService .
                        ?step p-plan:hasInputVar ?this .
                        ?this a tcs:Config .
                        }
                  """ ;
              ] ;
        sh:property [
            sh:path tcs:embedded ;
            sh:node [
                a sh:NodeShape ;
                # TODO
            ] ;
        ] .
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
        a tcs:Config;
        tcs:embedded
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
        a tcs:Config;
        tcs:embedded
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
        a tcs:Config;
        tcs:embedded
        [
          :url "http://path-to-rdf-connect-channel"
        ]
    ] .
```

### RDF-Connect Service

Performs continuous threshold monitoring on the LDES stream. If a value exceeds a limit (e.g., >7m), it generates an Error message.

```
:demoThresholdMonitoringProcessor a :PipelineStep ;
  :toBeCarriedOutByProcessor rdfc:thresholdMonitoringProcessor ;
  p-plan:hasInputVar [
    a tcs:Config;
    tcs:embedded
    [
      :value "7"
      :unit <http://qudt.org/vocab/unit/CentiM>
    ]
  ] .
```

### SPARQL ingest service

The Error message must be inserted in a triple store in order that the Semantic Works service can further process.

```
:demoThresholdMonitoringProcessor a :PipelineStep ;
  :toBeCarriedOutByProcessor rdfc:SPARQLIngest ;
  p-plan:hasInputVar [
    a tcs:Config;
    tcs:embedded
    [
      :memberStream <in> ;
      :ingestConfig [
        :operationMode ""Replication"
      ]
    ]
  ] .
```

### Semantic Works Service(s)

Detects the alert and automatically triggers a notification service (e.g., an emergency email) sending out a couple of e-mails to emergency responders.

In practice, the lblod/loket-error-alert-service gets triggered by the Error where the subject is the service rdfc:thresholdMonitoringProcessor, and generates an Email in the triple store.

Then, the repencilio/deliver-email-service can be used to sent the e-mail(s).

```
:demoLoketErrorAlertProcessor a :PipelineStep ;
  :toBeCarriedOutByProcessor :loketErrorAlertService ;
  p-plan:hasInputVar [
    a tcs:Config;
    tcs:embedded
    [
      :EMAIL_FROM "test@domain.net" ;
      :EMAIL_TO "test@domain.net,123@domain.com"
    ] ;
    tcs:literal """
      {
        // URI Base to be used at data creation.
        "base": "http://lblod.data.gift",
        "service": {
          // URI Resource identifier of this service
          "uri": "http://lblod.data.gift/services/loket-error-alert-service"
        },
        // Candidate service that creates delta's of interest. If non is configured, any service is accepted.
        "creators": ["https://w3id.org/rdf-connect#thresholdMonitoringProcessor"],
        "email": {
          // Created emails will be placed here.
          "folder": 'http://data.lblod.info/id/mail-folders/2'
        },
        "graph": {
          // Graph were emails live.
          "email": "http://mu.semte.ch/graphs/system/email"
        }
      }
    """ .
  ] .

:demoDeliverEmailProcessor: a :PipelineStep ;
  :toBeCarriedOutByProcessor :deliverEmailService: ;
  p-plan:hasInputVar [
    a tcs:Config;
    tcs:embedded
    [
      :MAILBOX_URI 'http://data.lblod.info/id/mailboxes/1'
    ]
  ] .
```


### Elody for alert viz

Elody is used as a clear UI for visualizing incoming flood-detection alerts in a more interactive way than just e-mails. It maps the context of each alert, including its source, thresholds, and underlying contracts, to make the data flow tangible and intuitive for stakeholders during demonstrations.


### Elody for deployment (pipeline builder)

Acting as a "click-and-connect" interface, this component allows users to inspect SHACL contracts and orchestrate the connection between datasets and services to manage configurations & deployment.

- [ ] Elody for deployment = creating the pipeline definition file
- [ ] Where does the pipeline validation happen? Could be on two places: in the Elody UI, but also in the generator (where validation is focusing on the generated result)
- [ ] Who will create the validation library of a pipeline (going through the chain of pipeline processors and check output with input of previous processor, and use shape matching algorithm)
- [ ] Pipeline generator = building the Docker compose with all configuration

# Config versus data shape validation

- [ ] We have a shape construction algorithm for datasets but not for pipeline components/services (see below). We assume that every pipeline component needs its own strategy to generate input/output shapes.
- [ ] Currently, the focus of the pipeline generator and specification lies in the description of the configuration parameters of a pipeline component, so it can be listed in a user interface (Elody, SHACL UI...). We suggest to also include pipeline steps (with config and input/output shape of the pipeline component) in the catalogue, so extra reusability of pipeline steps and validation can be achieved.
- [ ] For validating the demonstrator that the pipeline breaks when the unit of the source changes: the threshold monitoring processor have value and unit configuration parameters. When a pipeline step with the threshold monitoring processor is configured, then an input shape can be generated. When the source shape changes, this should create a validation error. However, what if LDIO changes the shape or when we don't have input/output shapes of processors? Will we send a sample through the pipeline to validate?
- [ ] Idea for Semantic.works: extend Delta notifier with SHACL support

However, the demonstrator describes a use case where we need data shape validation: does the shape of the datasource conflict with the input/output combination of shapes of processors?
The :PipelineComponents described above are generic descriptions where the config is not materialized yet. A shape is provided for validating the config in a later stage.

Below, I will give some examples how the data shape approach looks for the different frameworks.
For Semantic Works service, the approach is focusing on the description of input/output data shapes: https://github.com/DiSHACLed/discovery-specification/blob/main/20250422142901-describing_microservices.md

```
:dishacled-catalogue dcat:service :reusableLoketErrorAlertProcessor .

:reusableLoketErrorAlertProcessor a :PipelineStep ;
  :toBeCarriedOutByProcessor :loketErrorAlertService ;
  dcat:qualifiedRelation [
        a dcat:Relationship;
        dcat:hadRole :inputShape ;
        dcterms:relation :errorShape
    ];
  dcat:qualifiedRelation [
        a dcat:Relationship;
        dcat:hadRole :outputShape ;
        dcterms:relation :MailShape
    ] .

:errorShape a sh:NodeShape ;
  sh:targetClass oslc:Error ;
  sh:property [
      sh:path dct:subject ;
      sh:datatype xsd:string ;
      sh:minCount 1 ;
      sh:maxCount 1 ;
  ] ;
  sh:property [
      sh:path oslc:message ;
      sh:datatype xsd:string ;
      sh:minCount 1 ;
      sh:maxCount 1 ;
  ] .

:mailShape a sh:NodeShape ;
  sh:targetClass nmo:Email ;
  sh:property [
      sh:name "folder" ;
      sh:path nmo:isPartOf ;
      sh:nodeKind sh:IRI ;
      sh:minCount 1 ;
      sh:maxCount 1 ;
  ] ;
  sh:property [
      sh:path nmo:messageFrom  ;
      sh:datatype xsd:string ;
      sh:minCount 1 ;
      sh:maxCount 1 ;
  ] ;
  sh:property [
      sh:path nmo:messageTo  ;
      sh:datatype xsd:string ;
      sh:minCount 1 ;
      sh:maxCount 1 ;
  ] .
```

