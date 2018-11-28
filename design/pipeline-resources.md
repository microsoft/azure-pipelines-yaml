# Resources in YAML

Any external service that is consumed as part of your pipeline is a resource. 

An example of a resource can be another CI/CD pipeline that produces artifacts (say Azure pipelines, Jenkins etc.), code repositories (GitHub, Azure Repos, Git), container image registries (ACR, Docker hub etc.) or package feeds (Azure Artifact feed, Artifactory package etc.).  

## Why resources?

Resources are defined at one place and can be consumed anywhere in your pipeline. Resources provide you the full traceablity of the services consumed in your pipeline including the branch, version, tags, associated commits and work-items. You can fully automate your DevOps workflow by subscribing to trigger events on your resources.

Resources in YAML represent sources of types pipelines, repositories, containers and packages.


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

If you have a pipeline that produces artifacts, you can consume the artifacts by defining a `pipelines` resource. A pipeline can be another Azure DevOps pipeline or any external pipelines like Jenkins etc.


### Schema

```yaml
resources:        # types: pipelines | repositories | containers | packages
  pipelines:
  - pipeline: string  # identifier for the pipeline resource
    type: enum  # type of the pipeline source like azurePipelines, Jenkins etc. 
    connection: string  # service connection to connect to the source
    project: string # project for the source; optional for current project  
    source: string  # source defintion of the pipeline
    version: string  # version to pick the artifact, optional; defaults to Latest
    branch: string  # branch to pick the artiafct, optional; defaults to master branch
    tags: string # picks the artifacts on from the pipeline with given tag, optional; defaults to no tags
```

The inputs starting from `project` to `tags` are provider `type` specific properties. The above schema is for the `type`: `azurePipelines`. The inputs may change for other provider types like Jenkins etc.


### Examples

If you need to consume artifacts from another azure pipeline from the current project and if you dont require setting branch, version and tags etc., this can be shortened to:

```yaml
resources:         
  pipelines:
  - pipeline: SmartHotel      
    source: SmartHotel-CI  # name of the pipeline source definition
```

By default, the `type` is taken as `azurePipelines` unless specified otherwise.


In case you need to consume an azure pipeline from other project, then you need to include the project name while providing source name.

```yaml
resources:         
  pipelines:
  - pipeline: SmartHotel   
    project: DevOpsProject
    source: SmartHotel-CI  
    branch: releases/M142
```

### `downloadArtifact` for pipelines

Artifacts from the `pipeline` resource are automatically downloaded and made available for all the jobs. However, in any of the jobs, you can choose to override and download only specific artifacts using `downloadArtifact` shortcut. Once you use `downloadArtifact` to override and download a specific artifact, automatic artifact download behavior is removed and you need to specify all the artifacts you intend to download in the job.


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

Artifacts from the `pipeline` resource are downloaded to `$PIPELINES_RESOURCESDIR/<pipeline-identifier>/` folder.

## Resources: `repositories`

If you have multiple repositories from which you need to sync the code into your pipeline, you can consume the repos by defining a `repositories` resource. A repository can be another Azure Repo or any external repo like GitHub etc.


### Schema

```yaml
resources:          # types: pipelines | repositories | containers | packages
  repositories:
  - repository: string # identifier for the repository resource      
    type: enum # type of the repository source like AzureRepos, GitHub etc. In future this can extend to other source types
    connection: string # service connection to connect to the source, defaults to primary source connection
    source: string  # source repository to fetch
    refs: string  # ref name to use, defaults to 'refs/heads/master'
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

Repos from the `repository` resources defined are automatically synced and made available for all the jobs in the pipeline. However, in any of the jobs, you can choose to override and sync only specific repository using `checkout` shortcut. 

```yaml
- checkout: string  # identifier for your repository; for primary repository use the keyword self.
  clean: boolean  # whether to fetch clean each time
  fetchDepth: number  # the depth of commits to ask Git to fetch
  lfs: boolean  # whether to download Git-LFS files
  submodules: true | recursive  # set to 'true' for a single level of submodules or 'recursive' to get submodules of submodules
  persistCredentials: boolean  # set to 'true' to leave the OAuth token in the Git config after the initial fetch
  root: string        # directory to checkout the repo
```

The repo is checked-out to `$PIPELINES_RESOURCESDIR/<pipeline-identifier>/` folder.

### Example

```yaml
- checkout: secondaryRepo  
  clean: false
  fetchDepth: 5
  lfs: true
```

You can sync your primary repository as `self`.
```yaml
- checkout: self  
```

Or to avoid syncing any of the repository resources use:

```yaml
- checkout: none
```

When you use `checkout` to sync a specific repository resource, all the other repositories are not synced. 

`self` checkout directory: `$PIPELINES_SOURCESDIR`
other repositories' checkout directory: `$PIPELINES_RESOURCESDIR/<repository-identifier>/`

## Resources: `containers`

If you need to consume a container image as part of your CI/CD pipeline, you can achieve it using `containers`. A container can be an Azure Container Registry or any external Docker registry.

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
  - container: smartHotel 
    type: Docker
    connection: myDockerRegistry
    image: smartHotelApp 
```

Once you define a container as resource, container image metadata passed to the pipeline in the form of variables. Information like image, registry and connection details are made accessible across all the jobs so that your kubernetes deploy tasks can extract the image pull secrets and pass it to the cluster.

# Triggers
<TODO>
Proposal for resource level triggers will be taken care in the next iteration.
