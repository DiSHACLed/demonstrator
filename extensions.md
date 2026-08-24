Extensions to the semantic model I'm assuming here...

- Logical connections (`tcs:Connection`) + handling components with multiple inputs/outputs

    A type that represents a *logical*, directed connection (an arrow) between two *different* components (`tcp:from` a "source" `tcp:to` a "target").
    Such a connection does not necessarily directly point to a "physical" connection.
    E.g. we can have a logical connection between components of different frameworks.

    As such, logical connections are an abstraction over;
    - "concrete" connections (e.g. between two ldio components, two rdfc compononts)
    - "implicit" connections between two semantic works services
    - "bridgeable" connections between components of different frameworks
    
    Structure (TODO write shacl contracts for this) ;

    + Subjects; just blank nodes
    + Objects of this (when only one input/output channel)
        * `tcs:from` (exactly *one*, object of type `tcs:InstancePipelineComponent`)
        * `tcs:to` (same)
    + Object when one needs to specify between different output/input channels (e.g. [shacl-processor](https://github.com/rdf-connect/shacl-processor-ts)):
        When defining instance, different channels will be linked to different blank nodes; these will then be used. See [rdfc.ttl](/catalogs/components/rdfc.ttl) and [final-scenario-b.ttl](/pipelines/final-scenario-b.ttl) for an example using shacl-processor component.
        Things to have in mind for whatever we end up with;
        * Things like `rdfc:outgoing` do not need te be unique across different components
        * We also want to distinguish between multiple instances of the same component (e.g. two instances of shacl-processor)

    Upon a first phase during compilation, these logical connections can be made into real connections; where applicable appropriate bridges would then be inserted.

    We model semantic works services as follows (a simplification that's sufficient for the services we use in the demo here);

    + input through reading of the central database only (at intervals/at deltas through the delta notifier)
    + output through writing to the central database

        So no direct flows; only implicit information flows.
        Validation just checks that connections agree upon the contracts.

- Proposal for configuring components (`tcs:wrappedConfigShape`, `tcs:WrappedPipelineConfig`, `tcs:unwrapConfig`);
    Goal of proposal; 
    1. low effort/responsibility from pipeline definer to configure pipeline
    2. no repeating (and possibly contradicting yourself; e.g. `tm:stream`...)
    3. be able to easily validate stuff about config
    
    Moreover; 
    + high effort/responsibility when you want to register a component in the catalog (but once done, it's done); you need to do;
        1. decide what's the minimal info required from the user to be able to configure the service as a shacl contract (registered with `dcat:hadRole` as `tcs:wrappedConfigShape`)
        2. write a sparql construct query (as object of `tcs:unwrapConfig`) that results in something satisfying `tcs:configShape` 
    + when using pipeline (creating instance), you simple get to configure your component with the simplified `tcs:WrappedPipelineConfig` (satisfying `tcs:wrappedConfigShape`).

- Shapes for instances with more than one input/output channels
    See [final-scenario-b.ttl](/pipelines/final-scenario-b.ttl)

- Handling special shapes...
    Lot's of components will impose interesting restrictions between input and output shapes...
    It's all too easy to mistake yourself

    Idea; 
    + take on lots of responsibility and do this once only (when registering component)
    + pipeline definers explicitly write down the shapes they expect upon instantiation/configuring; but they should not be trusted. What they have written down should be verified.

    Proposal; 
    - When components are annotated with fixed contracts (independent of config/surrounding components) nothing special needs to happen. Pipeline definers may possible explicitly write down shapes as well; if so, the validator just needs to verify subsumption with the ones from the catalog (covariance on input shapes).
    - When components cannot be annotated with fixed contracts (they depend on config/surrounding components; e.g. skolemizing, sparql-construct-transformer, shacl-processor..), pipeline definers have to manually annotate their instances. In this case, said annotations have to verified by the validator. What needs to be verified will be highly component-specific. The component in the catalog can simply refer to e.g. a specialized python function (lets call it a 'resolver' function) taking as parameters:
        + any possible config
        + the input shapes from the instance
        + the output shapes from the instance
    Some examples;
        + skolomizer; needs to replace blank node reqs with IRI's (maybe it's more subtle?); then check if the result is compatible with output shapes
        + sparql-construct-transformer; given input shape and construct query one can infer "strongest"(?) shape of result. This is a research question tackled [here](https://arxiv.org/abs/2402.08509) (if you read the abstract, it's almost uncanny how applicable it is here). 
        + shacl-processor; discussed at length already...
    
    Additional notes, such resolver functions need to be:
    - sound (i.e. if they say there is no problem, there should never be a problem; given any input data satisfying the input shape, the component should always churn out output data that satisfies the output shape).
    - not necessarily complete (in a first instance); they may crap out, or even report that there is a problem, or if in actuality the above property is satisfied. E.g. resolver for the skolomizer may simple reject any input shape that has the restriction of being a blank node..

- document somewhere how we encode json-ld apis in catalog (the following is what we decided upon in a meeting many a moon ago):

    ```
    ... a dcat:Distribution ;
        dct:title "ld-json api for..."@en ;
        dct:format "application/ld+json" ;  
        dcat:endpointURL "https://dishacled-api.azurewebsites.net/api/v1/source-a/current" .
    ```