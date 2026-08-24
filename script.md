Goal; detailed timeline for demo, answering "what to show, how and when?"

# API + shapes for that api [data.ttl](catalog/data.ttl)

1. show source API

    https://dishacled-api.azurewebsites.net/api/v1/source-a/current

    refresh a couple of times

2. add API to dcat catalog (according to our convention)

    ```sh
    ./...
    ```

3. show sample of API (created with simple script)

4. add sample as another distribution of dcat dataset

    ```sh
    ./...
    ```

5. show how shapes are automatically generated for sample

6. copy shapes from sample to api

# Defining pipeline (step by step, with errors!)

***Goal: Simulate a setting where developer impatiently tries out a lot (and messes up), to showcase the power of the validator; each of these cases are flagged statically by the validator with an informative and easy-to-read error message.***

## Scenario A

### Initially

Quick reminder: scenario A is the one *without* dynamic validation. 
We assume the vendor follow best practices; they do not change the behavior of the API whilst keeping the endpoint URL constant.

1. Quickly show list of components [main](/components/main.ttl)

2. Naive start ([naive-polling-string.ttl](/pipelines/naive-polling-string.ttl))
    - polling a string (instead of dcat entity) 
    - show report validator on this file with nice error message
    - [naive-polling-sample.ttl](/pipelines/naive-polling-sample.ttl); again show nice error message

3. Correct polling [correct-polling.ttl](/pipelines/correct-polling.ttl)
    - configure with correct dcat entity (the one advertized as json-ld)
    - "how does this work? and how did the others fail?" show shacl contract of config

4. Continuing with invalid threshold component [invalid-threshold.ttl]
    - crtl+f "threshold" to find threshold monitoring component...
    - adding this component after the poller
    - showcase error of component
    - "why is this an error?"/"how does this work?" show;
        + contract of poller (inheriting contract of api)
        + conclusion; threshold component was written for...
        + even though "functioning" pipeline; deployment would never reach threshold!

5. Better searching with shacl-contracts...
    - search for components with sparql query ([](/searches/has-result.sparql))
        ```sh
        ... -f ./searches/has-result-mm.sparql
        ```
    - explain results
    - continuing with `threshold-cm` component
    - pre-existing error is gone

6. ...

### After API change

...

## Scenario B

Quick reminder: scenario B is the one *with* dynamic validation. 
We do not assume the vendor follow best practices; hence, we dynamically check the contract of the initial sample.

...


# Building valid pipelines with generator