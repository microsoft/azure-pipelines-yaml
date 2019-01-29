# Pipeline Caching

*** Status: Work in Progress ***

We have observed the need for Azure Pipelines to implement a dependency caching mechanism that allows the outputs of steps within a pipeline to be optimized (or even skipped) if a suitable cached version of the outputs of those steps already exists. Caching can be effective provided the cost of determining a cache hit and acquiring the contents of the cache is cheaper than the cost of producing the output again from scratch.

## Proposal

This proposal contains a number of interconnected elements. We have attempted to layer them in this document from the most primitive building blocks up to simplified YAML syntax to make caching "just work" in the majority of cases. Our approach to building a caching mechanism for Azure Pipelines will be to get the fundamental building blocks right and prove we can increase build performance and then evolve the YAML syntax once we know exactly what that syntax will need to describe.

### Restore and Save Cache Tasks

The basic functions of Pipeline Cache will be implemented by two tasks. The ```RestoreCache``` task and the ```SaveCache``` task. Both tasks will specify a collection of files and environment variables which can be used as a key to lookup the cache and identify a hit or a miss. Each task will also take a collection of paths which will be fetched from or stored in the cache associated with that key.

Consider the following Azure Pipelines pipeline defined in YAML:

```yaml
steps:
- script: yarn
- script: yarn build
```

In this example we are working on a React-based application which was bootstrapped using ```create-react-app```. The output of this command is a directory with about ~28,000 files including the ```node_modules``` directory. On a local developer machine the ```yarn``` command will trigger the installation of any missing packages. Typically this will execute quickly because all packages already exist on the developer machine. However in cloud-based CI environments where build machines are recycled after every build there is no persistent state which means that Yarn must download all packages from scratch.

Applying caching to this pipeline would involve adding a restore cache step and save cache step before and after the ```-script: yarn``` step.

```yaml
steps:
- task: RestoreCache@0
  inputs:
    keys: |
      package.json
      yarn.lock
    paths: |
      node_modules
- script: yarn
- task: SaveCache@0
  inputs:
    keys: |
      package.json
      yarn.lock
    paths: |
      node_modules
- script: yarn build
```

This is the most verbose version of the task syntax. We will provide shortcuts to correctly configure pipeline caching, but under the covers these are the tasks that will be emitted in to the pipeline job. When the job runs the ```RestoreCache``` task will hash the ```package.json``` and ```yarn.lock``` and combine it with some other caching elements such as operating system and a cache salt (for forced cache invalidation scenarios). It will then lookup the Pipeline Caching service using the fingerprint those files represent and download the cached content from Azure Artifact's blob store (if the content is present).

In either case the ```- script: yarn``` command will be executed, and in the case of a cache hit this will be a close to a no-op for that command. Restoring the cache will take some time and so it is important to make sure that the technique used for restoring the cache is appropriate for the scenario.

### Caching Strategies

When restoring from the cache it is important that the technique used to deliver content to disk is optimized for the content being handled. The size, number of files and environmental conditions can impact the most efficient approach. In the example above we sampled a number of different experiments to deliver the content of the ```node_modules``` folder - following from the scenario above.

| Experiment # | Execution Time | Approach                                                 |
| ------------ | -------------- | -------------------------------------------------------------- |
| 1            | 47 seconds     | ```yarn``` with clean cache                                    |
| 2            | 19 seconds     | ```yarn``` with dirty cache                                    |
| 3            | 47 seconds     | dedup file transfer with ```tar``` extraction                  |
| 4            | 22 seconds     | dedup file transfer with direct file placement                 |
| 5            | 44 seconds     | dedup file transfer with direct file placement into yarn cache |
| 6            | 42 seconds     | dedup file transfer with ```tar``` & gzip compression          |

The timings above are relative and performed in a local test environment. They are shown here to establish the impact that selecting different caching strategies can have on performance.

To establish a baseline of performance experiment #1 just executes the ```yarn``` command in a workspace with no ```node_modules``` folder and a clean Yarn cache - then in experiment #2 we similuate the local development workstation scenario by removing the ```node_modules``` folder by maintain the Yarn cache directory.

For experiment #3 we used a dedup file transfer mechanism built into Azure Artifacts (used behind the scenes by Universal Packages and Pipeline Artifacts). As part of the experiment we packed the node_modules folder into a tar file (without compression) which resulted in a 150MB file. We were able to transfer that file in about 19 seconds and then unpacking took about another 28 seconds - resulting in the same performance as a Yarn install on a clean cache (no improvement).

In experiment #4 we eliminated the tar file and allowed the dedup file transfer to directly place files on disk. This process took 22 seconds end to end which roughly halves the Yarn installation time.

In experiment #5 we instead cached the Yarn package cache directory instead of the local ```node_modules``` folder. The download of the packages into the Yarn cache directory took 22 seconds similar to experiment #4, however Yarn installation added 22 seconds for linking which meant it was roughly the same as a clean install. My comparison, running Yarn install with experiment #4 is close to a no-op for Yarn.

In order to explore the archive file scenario a little bit more we did experiment #6 where in addition to creating a ```.tar``` archive, we also added compression to the file (```.tgz```). Due to file size reductions the transfer time was reduced to about 7 seconds, but decompression and placing files using ```tar``` took 35 seconds.

Based on the above results it would appear that using our dedup file transfer mechanism with direct file placement would be the most performant option, however from internal experimentation at Microsoft we've observed that some larger code bases with hundreds of thousands of files in the ```node_modules``` directory do better with ```.tgz``` so we believe to service the breadth of caching requirements for Azure Pipelines users that we'll need to support a number of ways of placing files.

Other more elaborate caching mechanisms may also be available such as virtualized file systems with prefetch which may provide significant performance boosts particularly in sparse file access scenarios but introduce other issues which would need to make that approach strictly opt-in.

The caching strategy would default to  dedup file transfer mechanism but allow an option to override the caching strategy with another approach, for example"

#### Dedup Strategy Configuration
```yaml
steps:
- task: RestoreCache@0
  inputs:
    strategy: dedup
    keys: |
      package.json
      yarn.lock
    paths: |
      node_modules
- script: yarn
- task: SaveCache@0
  inputs:
    strategy: dedup
    keys: |
      package.json
      yarn.lock
    paths: |
      node_modules
- script: yarn build
```

#### TGZ Strategy Configuration
```yaml
steps:
- task: RestoreCache@0
  inputs:
    strategy: tgz
    keys: |
      package.json
      yarn.lock
    paths: |
      node_modules
- script: yarn
- task: SaveCache@0
  inputs:
    strategy: tgz
    keys: |
      package.json
      yarn.lock
    paths: |
      node_modules
- script: yarn build
```

### YAML sytnax

TODO:

### Cache Scoping

TODO:

### Cache Expiry

TODO: 

### Step Over Support

TODO: