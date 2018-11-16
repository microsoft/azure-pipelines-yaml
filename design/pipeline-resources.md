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
resources:        # types: pipelines | repositories | containers | packages
  pipelines:
  - pipeline: string  # identifier for the pipeline resource
    type: enum  # type of the pipeline source like AzurePipelines, Jenkins etc. In future this can extend to other source types.
    connection: string  # service connection to connect to the source
    source:
      name: string  # source defintion of the pipeline including the project i.e. projectName/Definition
      version: string  # version to pick the artifact, optional; defaults to Latest
      branch: string  # branch to pick the artiafct, optional; defaults to master branch
      tags: string # picks the artifacts on from the pipeline with given tag, optional; defaults to no tags.
```

### Examples

```yaml
resources:         
  pipelines:
  - pipeline: SmartHotel      
    type: AzurePipelines
    source: SmartHotel-CI # If your source definition is in current project and doesn't need to update default versions
```


### `downloadArtifact` for pipelines

Artifacts produced by `pipeline` resource are automatically downloaded and made available for all the jobs. However, in any of the jobs, you can choose to override and download only specific artifacts using `downloadArtifact` shortcut.


```yaml
- job: deploy_windows_x86_agent
  steps:
  - downloadArtifact: SmartHotel   # pipeline resource identifier.
    name: WebTier1  # artifact to download, optional; defaults to all the artifacts from the resource.
    patterns: '**/*.zip'  # mini match pattern to download specific files, optional; defaults to all files.
```

Or to avoid downloading any of the artifacts at all:

```yaml
- downloadArtifact: none
```


Refer to [download artifacts](https://github.com/Microsoft/azure-pipelines-yaml/blob/master/design/pipeline-artifacts.md#downloading-artifacts-downloadartifact) for more details.


## Resources: `repositories`

If you have multiple repositories from which you need to sync the code into your pipeline, you can consume using `repositories`. A repository can be another Azure Repo or any external repo like GitHub etc.


### Schema

```yaml
resources:          # types: pipelines | repositories | containers | packages
  repositories:
  - repository: string # identifier for the repository resource      
    type: enum # type of the repository source like AzureRepos, GitHub etc. In future this can extend to other source types
    connection: string # service connection to connect to the source, defaults to primary source connection
    source: string # source repository to fetch
    branch: string # branch to fetch the repo from, defauts to master.
    clean: boolean  # whether to fetch clean each time
    fetchDepth: number  # the depth of commits to ask Git to fetch
    lfs: boolean  # whether to download Git-LFS files
    submodules: true | recursive  # set to 'true' for a single level of submodules or 'recursive' to get submodules of submodules
    persistCredentials: boolean  # set to 'true' to leave the OAuth token in the Git config after the initial fetch
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

### `checkout` your repository

All the `repository` sources are automatically synced and made available for all the jobs in the pipeline. However, in any of the jobs, you can choose to override and sync only specific repository using `checkout` shortcut.


```yaml
- checkout: string  # identifier for your repository; for primary repository use the keyword self.
  clean: boolean  # whether to fetch clean each time
  fetchDepth: number  # the depth of commits to ask Git to fetch
  lfs: boolean  # whether to download Git-LFS files
  submodules: true | recursive  # set to 'true' for a single level of submodules or 'recursive' to get submodules of submodules
  persistCredentials: boolean  # set to 'true' to leave the OAuth token in the Git config after the initial fetch
```

### Example

```yaml
- checkout: secondaryRepo  
  clean: false
  fetchDepth: 5
  lfs: true
```

Or to avoid syncing any of the sources at all:

```yaml
- checkout: none
```

## Resources: `containers`

If you need to consume a container image as part of your CI/CD pipeline, you can achieve it using `containers`. A container can be an Azure Container Registy or any external Docker registry.

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
