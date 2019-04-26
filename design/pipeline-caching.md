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
    key: |
      package.json
      yarn.lock
    path: node_modules
- script: yarn
- task: SaveCache@0
  inputs:
    key: |
      package.json
      yarn.lock
    path: node_modules
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

Selecting the cache strategy would simply be a matter of specifying a strategy on the task inputs (for both the restore and save tasks, restore shown below):

```yaml
steps:
- task: RestoreCache@0
  inputs:
    strategy: dedup
    key: |
      package.json
      yarn.lock
    path: node_modules
```

### YAML syntax

The example task usage above uses the explicit task references. We also want to provide a streamlined syntax for caching, applied to the example above.

```yaml
steps:
- restoreCache: yarn.lock
  path: node_modules
- script: yarn
- saveCache:
  key: yarn.lock
  path: node_modules
- script: yarn build
```

This more compact syntax just reduces some of the noise around using the caching task. Additionally we are planning to integrate caching into the the ```use:``` syntax. This will look like the following:

```yaml
steps:
- use: node
  cache: true
- script: |
    yarn
    yarn build
    yarn test
```

The idea behind the ```use:``` syntax is that given what we are using (in this case ```node```) we make some automatic decisions about what and how to cache. This logic will use various heuristics to make the decision about what to cache, for example, it will look for the presence of a ```yarn.lock``` file or a ```package-lock.json``` to determine where the ```node_modules``` path may be located.

Other ```use:``` statements will have similar heuristics to pick the best defaults.

Note that the ```use:``` syntax will inject the cache save step at the end of the build process which will not always be desirable. In those cases developers can fall back to specifying the ```restoreCache:``` and ```saveCache:``` YAML statements.

### Cache Scoping

Getting scoping right is important for maximising cache hits and but also avoiding the cache becoming an attack vector to insert malicous code into official/master builds. A cache scope will be defined by the pipeline ID, the Git repos and refs that are being built and any other special considerations - such as whether the build is for a particular pull request. For regular builds on a branch (non PR scenario) a single scope will be used and the job will have read/write access to this scope.

For PR builds it is a little bit more complex because we want the PR build to be able to use the scope for the source or target branch of it is available. So i nthe PR scenario the job will have access to three scopes, one that represents the target branch, one that represents the source branch (these will both be R/O) and a third which represents the PR specifically. The PR scope will be tried first, then the source branch and then the target branch. In general topic-branch workflows the first build on a PR branch will be a cache miss on PR scope, the source branch will also likely be a miss but the third scope should be a hit (the target branch). The reason for these precedence rules is to allow for the most specific cache to be used.

When saving the cache, the entry will only be associated with one scope. The task will attempt to save the cache against the entries in precedence order. It is expected though that the first entry in the precendence list will have read/write access.

### Cache Expiry

Cache lifetime will be best effort. Our underlying storage will generally keep content for 7 days before it becomes a candidate for eviction, but we won't initially make a guarantee here. We will evaluate the effectiveness of cache durations and listen to community feedback.

### Step Over Support

In the interests of correctness we will not skip build steps by default based on a cache hit. However we will provide a way to emit a variable which can be used in subsequent steps to skip a task if a cache is hit. The usage is as follows:

```yaml
steps:
- restoreCache: yarn.lock
  path: node_modules
  skipVariable: cache.skipyarn
- script: yarn
  condition: eq(variables['cache.skipyarn'], 'sourcehit')
```

The value inserted into the variable specified by ```skipVariable:``` will change depending on whether there was a cache hit or miss, and what kind of hit it was. Example values are:

* sourcehit; used to signal that there is a cache hit on the source branch in a PR.
* targethit; used to signal that there is a cache hit on the target branch in a PR.
* hit; used to signal that there is a cache hit on the current branch (non PR build scenario).
* upstreamhit; used to signal that there is a cache hit from an upstream branch (applies to branch builds and PR builds).