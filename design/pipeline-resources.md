# Resources in YAML

Any source that is consumed as part of your pipeline is a resource. Resources in YAML represent sources of types pipelines, repositories, containers and packages.

An example of a resource can be resources published by another CI/CD pipeline (say Azure pipelines, Jenkins etc.), code repositories (GitHub), container registry (ACR, Docker hub etc.) and package feeds (Azure artifact feed, Artifactor etc.).  

## Why resources?

Resources enable you to define the sources at one place and consume anywhere in your pipeline. Resources provide you the full traceablity of the sources consumed in your pipeline including branches, versions, tags, associated commits and work-items. 

### Schema

```yaml
resources:
  pipelines: [ pipeline ]  
  repositories: [ repository ]
  containers: [ container ]
  packages: [ package ]
```

---

## Resources: `pipelines`

If you have a pipeline that produces artifacts, you can consume the artifacts using `pipelines` resource. A pipeline can be another Azure DevOps pipeline or any external pipelines like Jenkins etc.

### Schema

```yaml
resources:          # types: pipelines | repositories | containers | packages
  pipelines:
  - pipeline: string # identifier for the pipeline resource      
    type: enum # type of the pipeline source like AzurePipelines, Jenkins etc. In future this can extend to other source types.
    connection: string # service connection to connect to the source
    source: string # source defintion of the pipeline that produces the artifacts
    project: string # project that contains the source definition, optional; defauts to current project.
    branch: string # branch to pick the artiafct, optional; defaults to master branch
    version: string # version to pick the artifact, optional; defaults to Latest
    tags: string # picks the artifacts on from the pipeline with given tag, optional; defaults to no tags.
```

### Examples

```yaml
resources:         
  pipelines:
  - pipeline: SmartHotel      
    type: AzurePipelines
    source: SmartHotel-CI
```


## Resources: `repositories`

If you have multiple repositories from which you need to fetch the code into your pipeline, you can consume using `repositories`. A repository can be another Azure Repo or any external repo like GitHub etc.

### Schema

```yaml
resources:          # types: pipelines | repositories | containers | packages
  repositories:
  - repository: string # identifier for the repository resource      
    type: enum # type of the repository source like AzureRepos, GitHub etc. In future this can extend to other source types
    connection: string # service connection to connect to the source, defaults to primary source connection
    source: string # source repository to fetch
    branch: string # branch to fetch the repo from, defauts to master.
    fetchDepth: number # depth from the tip of the branch to fetch commits, optional; defaults to all
    lfs: boolean # checking our Git-LFS modules, defaults to false
    sync: boolean 
```

### Examples

```yaml
resources:         
  repositories:
  - repository: secondaryRepo      
    type: GitHub
    connection: myGitHubConnection
    source: Microsoft/alphaworz
```

## Resources: `containers`

If you have need a container image as part of your CI/CD pipeline, you can consume using `containers`. A container can be an Azure Container Registy or any external Docker registry.

### Schema

```yaml
resources:          # types: pipelines | repositories | containers | packages
  containers:
  - container: string # identifier for the container resource      
    type: enum # type of the registry like ACR, Docker etc. 
    connection: string # service connection to connect to the image registry, defaults to ACR??
    image: string # container image name, Tag/Digest is optional; defaults to latest image
    options: string # arguments to pass to container at startup
    env: { string:string } # list of environment variables to add
```

### Examples

```yaml
resources:         
  containers:
  - container: devLinux      
    image: GitHub
    connection: myDockerRegistry
    source: Microsoft/alphaworz
```
