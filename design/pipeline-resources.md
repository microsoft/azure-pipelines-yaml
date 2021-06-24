# Resources in YAML

Any external service that is consumed as part of your pipeline is a resource.

An example of a resource can be another CI/CD pipeline that produces artifacts (say Azure pipelines, Jenkins etc.), code repositories (GitHub, Azure Repos, Git), container image registries (ACR, Docker hub etc.) or package feeds (Azure Artifact feed, Artifactory package etc.).  

## Why resources?

Resources are defined at one place and can be consumed anywhere in your pipeline. Resources provide you the full traceability of the services consumed in your pipeline including the branch, version, tags, associated commits and work-items. You can fully automate your DevOps workflow by subscribing to trigger events on your resources.

Resources in YAML represent sources of types pipelines, builds, repositories, containers and packages. 


### Schema

```yaml
resources:
  pipelines: [ pipeline ]  
  builds: [ build ]
  repositories: [ repository ]
  containers: [ container ]
  packages: [ package ]
```

---

## Resources: `pipelines`
 

If you have an Azure Pipeline that produces artifacts, you can consume the artifacts by defining a `pipelines` resource. `pipelines` is a dedicated resource only for Azure Pipelines.

### Schema

```yaml
resources:        # types: pipelines | builds | repositories | containers | packages
  pipelines:
  - pipeline: string  # identifier for the pipeline resource
    connection: string  # service connection for pipelines from other Azure DevOps organizations
    project: string # project for the source; optional for current project
    source: string  # source definition of the pipeline
    version: string  # the pipeline run number to pick the artifact, defaults to Latest pipeline successful across all stages
    branch: string  # branch to pick the artifact, optional; defaults to master branch
    tags: string # picks the artifacts on from the pipeline with given tag, optional; defaults to no tags
```

### Examples

If you need to consume artifacts from another azure pipeline from the current project and if you don't require setting branch, version and tags etc., this can be shortened to:

```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI # name of the pipeline source definition
```

In case you need to consume a Pipeline from other project, then you need to include the project name while providing source name.

```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    project: DevOpsProject
    source: SmartHotel-CI
    branch: releases/M142
```

### `download` for pipelines

All artifacts from the current pipeline and from all `pipeline` resources are automatically downloaded and made available at the beginning of each of the 'deployment' job. You can override this behavior: see [Pipeline Artifacts](pipeline-artifacts.md#default-and-named-artifacts) for more details. For regular 'job' artifacts are not automatically downloaded. You need to use `download` explicitly wherever needed.

```yaml
- job: deploy_windows_x86_agent
  steps:
  - download: SmartHotel   # pipeline resource identifier.
    artifact: WebTier1  # artifact to download, optional; defaults to all the artifacts from the resource.
    patterns: '**/*.zip'  # mini match pattern to download specific files, optional; defaults to all files.
```

Or to avoid downloading any of the artifacts at all:

```yaml
- download: none
```

Artifacts from the `pipeline` resource are downloaded to `$(PIPELINE.WORKSPACE)/<pipeline-identifier>/<artifact-identifier>` folder; see [artifact download location](https://github.com/Microsoft/azure-pipelines-yaml/blob/master/design/pipeline-artifacts.md#artifact-download-location) for more details.

## Resources: `builds`

If you have any external CI build system that produces artifacts, you can consume the artifacts by defining a `builds` resource. A `builds` resource can be any external CI systems like Jenkins, TeamCity, CircleCI etc.

### Schema

```yaml
resources:        # types: pipelines | builds | repositories | containers | packages
  builds:
  - build: string   # identifier for the build resource
    type: string   # the type of your build service like Jenkins, circleCI etc.
    connection: string   # service connection for your build service.
    source: string   # source definition of the build
    version: string   # the build number to pick the artifact, defaults to Latest successful build
    branch: string   # branch to pick the artifact; defaults to master branch
```
The inputs for the `build` resource can change based on the `type` of the build service (i.e. Jenkins, TeamCity etc.). The publisher of each extension defines the inputs for the resource. During the pipeline authoring time, when user defines the resource, it is validated based on the inputs defined in the extension.

### Examples

```yaml
resources:
  builds:
  - build: Spaceworkz
    type: Jenkins
    connection: MyJenkinsServer 
    source: SpaceworkzProj   # name of the jenkins source project
```

### `downloadBuild` for builds
You can use `downloadBuild` task to download the artifacts available as part of the `build` resource.

### Schema

```yaml
- downloadBuild: string # identifier for the resource from which to download artifacts
  artifact: string # identifier for the artifact to download; if left blank, downloads all artifacts associated with the resource provided
  patterns: string | [ string ] # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  path: string # relative path from $(PIPELINE.WORKSPACE) to download the artifacts
```

The inputs for `downloadBuild` macro is fixed for all the build resources. The automatic artifact download and overriding behavior of `downloadBuild` is same as the `download` macro used for [pipeline artifacts](https://github.com/Microsoft/azure-pipelines-yaml/blob/master/design/pipeline-artifacts.md#downloading-artifacts-download).


### Examples
You can customize the download behavior for each deployment or job.

```yaml
- job: deploy_windows_x86_agent
  steps:
  - downloadBuild: Spaceworkz   # build resource identifier.
    artifact: WebTier1  # artifact to download, optional; defaults to all the artifacts from the resource.
    patterns: '**/*.zip'  # mini match pattern to download specific files, optional; defaults to all files.
```

Or to avoid downloading any of the artifacts at all:

```yaml
- downloadBuild: none
```
Based on the type of build resource (Jenkins, TeamCity etc.) and the associated artifacts, appropriate task is used to download the artifacts in the job.

Artifacts from the `build` resource are downloaded to `$(PIPELINE.WORKSPACE)/<build-identifier>/` folder unless user specifies a path in which case artifacts are downloaded to the path provided. 

We provide full artifact traceability i.e which artifact is downloaded from which resource for every job in a pipeline.

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
    ref: string  # ref name to use, defaults to 'refs/heads/master'
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

Repos from the `repository` resources defined are automatically synced and made available for all the jobs in the pipeline, except deployment jobs. However, in any of the jobs, you can choose to override and sync only specific repository using `checkout` shortcut. 

Repos from `repository` resource and `self` repo are not automatically synced in 'deployment' jobs. If you required repo to be fetched in the deployment job, you need to explicitly `checkout`.

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


If you need to consume images from external docker registries as part of your pipeline, you can define a generic container resource. A generic container resource requires a Docker registry service connection.

### Schema
```yaml
resources:          # types: pipelines | repositories | containers | packages
  containers:
  - container: string # identifier for the container resource   
    connection: string # service connection (Docker registry) to connect to the image registry;
    image: string # container image name, Tag/Digest is optional; defaults to latest image
    options: string  # arguments to pass to container at startup
    env: { string: string }  # list of environment variables to add
    ports: [ string ] # ports to expose on the container
    volumes: [ string ] # volumes to mount on the container
```

### Examples

```yaml
resources:         
  containers:
  - container: smartHotel 
    connection: myDockerRegistry
    image: smartHotelApp 
```

We also provide a first class experience for Azure Container registry (ACR). You can create a container resource of type ACR. 

### Schema
```yaml
resources:          # types: pipelines | repositories | containers | packages
  containers:
  - container: string # identifier for the container resource      
    type: string # type of the registry like ACR, GCR etc. 
    subscription: string # Azure subscription (ARM service connection) for container registry;
    registry: string # registry for the container images
    image: string # container image name, Tag/Digest is optional; defaults to latest image
```
### Examples

```yaml
resources:         
  containers:
  - container: petStore
    type: ACR
    subscription: jPetsAzureSubscription 
    registry: myDockerRegistry
    image: jPetStoreImage 
```
ACR container resource enables you to use Azure service principal (ARM service connection) for authentication. You can disable admin user for the container registry in azure and enforce using service principal.
ACR container resource provides you with rich [triggers](https://github.com/microsoft/azure-pipelines-yaml/blob/master/design/pipeline-triggers.md#containers) experience with support for location based triggers and better traceability. 

Once you define a container as resource, container image metadata is passed to the pipeline in the form of variables. Information like image, registry and connection details are made accessible across all the jobs so that your kubernetes deploy tasks can extract the image pull secrets and pass it to the cluster.

## Resources: `packages`

If you need to consume a package from your package repository as part of your deployment you can define a `package` resource. It can be any package type like Nuget, NPM, Python or Gradle. And you can consume the package from any package repository like GitHub packages, Nuget org or NPMJS etc.

### Schema
```yaml
resources:        # types: pipelines | builds | repositories | containers | packages
  packages:
  - package: string   # identifier for the package resource
    type: string   # the type of your package like Nuget, NPM etc.
    connection: string   # service connection for your package repo.
    package: string   # package definition
    version: string   # optional; the version of the package to download, defaults to latest published vession.
    tag: string   # optional; branch to pick the artifact; defaults to master branch
```
The inputs for the `package` resource can change based on the `type` of the package (i.e. Nuget, NPM etc.). The publisher of each package type defines the inputs for the resource. 

### Examples

```yaml
resources:
  packages:
  - package: MyPackage
    type: Nuget
    connection: MyConnection # Here we support Nuget service connection and GitHub service connection 
    package: types/zookeper   # name of the package; For github packages it is repo/package
```

### `getPackage` for packages

All the package types defined in `package` resources can be downloaded as part of your pipeline jobs using the macro `getPackages`. Package resources are not automatically downloaded. You need to explicitly define this task in your job/deploy-job to download the package.

### Schema
```yaml
- getPackage: string # identifier for the package resource from which to download the package
  path: string # relative path from $(PIPELINE.WORKSPACE) to download the package
```

### Examples
```yaml
- job: deploy_windows_x86_agent
  steps:
  - getPackage: MyPackage   # package resource identifier.
```

packages from the `package` resource are downloaded to `$(PIPELINE.WORKSPACE)/<resource-identifier>/` folder. 
We provide full artifact traceability i.e which package is downloaded from which resource for every job in a pipeline.

# Triggers
Refer to the [spec](https://github.com/Microsoft/azure-pipelines-yaml/blob/master/design/pipeline-triggers.md) for resource level triggers.
