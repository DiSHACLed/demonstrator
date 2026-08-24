# Main Goals

- a [script](./script.md) with a detailed description of how a demo would look like

- the semantic pipeline definitions in script (written by end user) are all *"logical"*; i.e. there are no explicit "bridges" between the frameworks anymore (this amounts to what would be supported by the "toddler" pipeline generator)

- script provides a testbed for how validation should behave

- proposal for semantic description of configs (and how they should be compiled down)

- better configs (for polling API, semantic works services...)

# Notes

All extensions to semantic model/discovery spec/pipeline generator/... are described [here](./extensions.md).

# Why these goals

- verification on such a pipeline is easier; we don't need to worry about these bridges

- is in line with the claimed "interoperability"

# How could this fit with `data`-folder of pipeline generator

Pre-processing stage in compiler that compiles these pipelines to something like [this](https://github.com/thcarsten/toolchain-specification/blob/main/pipeline%20generator/data/pipeline_definition.ttl) 

# Non-goals

- components with generic variables

- fully support SW (we make crude simplification that works when considering the two services that we are using here)

# Files

- [script.ttl](./script.md)
- [extensions.md](./extensions.md): proposed extensions
- [searches](/searches/): sparql queries used in demo to find stuff through discovery service...
- [pipelines](/pipelines/): all the pipelines show in demonstrator (many of them impartial/invalid to demonstrate capabilities of validator)
- [catalogs/](/catalogs/)
    + [data.ttl](/catalogs/data.ttl)
        Dcat catalog with with water measurements dateset containing the following distributions
        * `ex:water-measurements-dist`: Tom's [source api](https://dishacled-api.azurewebsites.net/api/v1/source-a/current) (advocated as json-ld)
        * `ex:water-measurements-sample`: A fictitious [sample](./sample-a-20260824-15m.ttl) we took from that API (with the idea of adjusting the API such that the sample can actually be real)
    + [components/](/catalogs/components/)
        Catalog of components taken from Thomas [here](https://github.com/thcarsten/toolchain-specification/tree/main/pipeline%20generator/data) but adjusted and removed everything which would not be needed for demonstrator.
- ...