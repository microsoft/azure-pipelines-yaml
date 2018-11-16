# Pipeline Caching

*** Status: Work in Progress ***

We have observed the need for Azure Pipelines to implement a caching mechanism that allows the outputs of steps within the pipeline to be skipped if a suitable cached version of the output already exists.

Provided the cost of determining a cache hit and acquiring the contents of the cache is cheaper than the cost of producing the output again from scratch then the pipeline's performance will be increased.

Caching can introduce many complexities into build pipelines and our goal is to strike the balance between "correct" and "good enough".

## Proposal

Think of this proposal contains a number of interelated elements. I've tried to build it up sequentially bug sometimes you'll need to read further along to understand why something is the way it is.

### The basics & task integration

At the end of the day, when work is done on an agent as part of a pipeline, it is represented as a _job_ when a set of _steps_ which are implemented by tasks. For many pipelines the YAML syntax obscures the underlying "tasky" nature of Azure Pipelines, but they are there nonetheless. As a result I'm starting with how pipeline caching will integrate with the underlying task infrastructure.

The first thing to understand is that detecting a cache hit or miss will be performend by the agent worker that was spawned to service a job. As the worker prepares to execute each task it will check to see whether any given task has caching enabled and if so it will determine the key and check for a cache it. If a hit is detected it will set about restoring the cached content into the pipeline execution environment. Exactly how that content is restored will be discussed a bit later.

Customers that are using the task-designer experience will be able to take advantage of pipeline caching. Cache control properties will be available in a _Caching_ group under each task in the designer.

![Picture of Azure Pipelines task designer with Caching group](images/pipeline-caching-task-editor.jpg)

TODO: Expand on Use cache and Populate cache and how Key and Capture fields are used.

### YAML expression

TODO:

```yaml
steps:
  - task: Npm@0
    cache: true
    inputs:
      command: restore
```

Resolves to...

```yaml
steps:
  - task: Npm@0
    cache:
      key: ./package.json
      capture: ./node_modules
    inputs:
      command: restore
```

```yaml
steps:
  - task: Npm@0
    cache:
      key:
        - ./package.json
        - /etc/os-release
      capture:
        - ./node_modules
    inputs:
      command: restore
```


### Use statement interaction

TODO: We need to figure out whether it makes sense for this to interact with the ```use:``` statement in any way. The [use statement](use-statement.md) is designed to be prepended to script blocks (for example):

```yaml
- use: node
  version: 8
- script:
    | npm install
    | npm run build
```

The idea is that all scipts following the ```use:``` block will use Node.js 8.x. So how would this apply to caching? Perhaps we could adopt a syntax as follows:

```yaml
- use: node
  version: 8
- use: cache
  key: ./package.json
  capture:
   - ./node_modules
   - ./build
- script:
    | npm install
    | npm run build
```

### Key expressions

Selecting the right key for the cache is critical in order to produce _correct_ or event _good enough_ results. For example if a Node.js module is designed to work across multiple platforms and takes native dependencies that need to be compiled using _node-gyp_ (for example) then the cache will need to include sufficient platform information to differentiate caches on Windows, macOS and Linux and sub-flavors for really specific native dependencies.

### Capture expressions

TODO:

### Caching strategies

TODO: Why do we need multiple caching strategies.

#### Zip

TODO:

#### Virtualize

TODO:

#### Proxy

TOOD:

### Sidecar processes

TODO: Why do we need a process per caching strategy.

### Security model

TODO: Rights composition...

### Retention & extension

TODO: Default seven days

### Cache management

TODO:

### Outer caching vs. inner caching

TODO: