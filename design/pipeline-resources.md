# Resources in YAML

Any external service that is consumed as part of your pipeline is a resource.

An example of a resource can be another CI/CD pipeline that produces artifacts (say Azure pipelines, Jenkins etc.), code repositories (GitHub, Azure Repos, Git), container image registries (ACR, Docker hub etc.) or package feeds (Azure Artifact feed, Artifactory package etc.).  

## Why resources?

Resources are defined at one place and can be consumed anywhere in your pipeline. Resources provide you the full traceablity of the services consumed in your pipeline including the branch, version, tags, associated commits and work-items. You can fully automate your DevOps workflow by subscribing to trigger events on your resources.

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
 

An Azure Pipeline that produces artifacts can be consumed by defining a `pipelines` resource. `pipelines` is a dedicated resource only for Azure Pipelines. A new pipeline is triggered automatically whenever a new run of the `pipeline` resource is succesfully completed. However, you can provide specific conditions to override the `trigger`.

### Schema

```yaml
resources:        # types: pipelines | builds | repositories | containers | packages
  pipelines:
  - pipeline: string  # identifier for the pipeline resource
    connection: string  # service connection for pipelines from other Azure DevOps organizations
    project: string # project for the source; optional for current project
    source: string  # source defintion of the pipeline
    version: string  # the pipeline run number to pick the artifact, defaults to Latest pipeline successful across all stages
    branch: string  # branch to pick the artiafct, optional; defaults to master branch
    tag: string # picks the artifacts on from the pipeline with given tag, optional; defaults to any tag or no tag.
    trigger:     # Optional; Triggers are enabled by default.
      branches:  # branch conditions to filter the events, optional; Defaults to all branches.
        include: [ string ]  # branches to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # branches to discard the trigger events, optional; Defaults to none.
      stages: [ string ]  # trigger after completion of given stage, optional; Defaults to all stage completion.
      tags: [ string ]  # tags on which the trigger events are considered, optional; Defaults to any tag or no tag.
```

### Examples

You can consume artifacts from another azure pipeline from current project with simple syntax.

```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI # name of the pipeline source definition
```

You can consume a azure pipeline from another project.

```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    project: DevOpsProject
    source: SmartHotel-CI
    branch: releases/M142
```

### `trigger` for pipelines

Triggers are enabled by default unless expliciltly opted out.

You can disable the triggers on the `pipeline` resource.

```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: none
    
```

You can control which branches get the triggers with a simple syntax.

```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: 
    - releases/*
    - master
 ```
 
 You can specify the full name of the branch (for example, master) or a prefix-matching wildcard (for example, releases/*). You cannot put a wildcard in the middle of a value. For example, releases/*2018 is invalid.
 

You can specify the branches to include and exclude.

```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: 
      branches:
        include: 
        - releases/*
        exclude:
        - master
 ```
 
 You can specify which tags to control the triggers.
 
 ```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: 
      branches:
        include: 
        - releases/*
      tags: 
      - Production
      - Signed
 ```
 
If you don't want to wait until all the stages of the run are completed for the `pipeline` resource. You can provide the stage to be completed to trigger you pipeline. 

```yaml
resources:
  pipelines:
  - pipeline: SmartHotel
    source: SmartHotel-CI 
    trigger: 
      branches:
        include: master
      stages: 
      - QA
 ```
### `download` for pipelines

All artifacts from the current pipeline and from all `pipeline` resources are automatically downloaded and made available at the beginning of each job. You can override this behavior: see [Pipeline Artifacts](pipeline-artifacts.md#default-and-named-artifacts) for more details.

```yaml
- job: deploy_windows_x86_agent
  steps:
  - download: SmartHotel   # pipeline resource identifier.
    name: WebTier1  # artifact to download, optional; defaults to all the artifacts from the resource.
    patterns: '**/*.zip'  # mini match pattern to download specific files, optional; defaults to all files.
```

Or to avoid downloading any of the artifacts at all:

```yaml
- download: none
```

Artifacts from the `pipeline` resource are downloaded to `$PIPELINE.RESOURCESDIRECTORY/<pipeline-identifier>/<artifact-identifier>` folder; see [artifact download location](https://github.com/Microsoft/azure-pipelines-yaml/blob/master/design/pipeline-artifacts.md#artifact-download-location) for more details.




## Resources: `builds`

If you have any external CI build system that produces artifacts, you can consume the artifacts by defining a `build` resource. A `build` resource can be any external CI systems like Jenkins, TeamCity, CircleCI etc. And you can enable triggers on the `build` resource. 

### Schema

```yaml
resources:        # types: pipelines | builds | repositories | containers | packages
  builds:
  - build: string   # identifier for the build resource
    type: enum   # the type of your build service like Jenkins, circleCI etc.
    connection: string   # service connection for your build service.
    source: string   # source definition of the build
    version: string   # the build number to pick the artifact, defaults to Latest successful build
    branch: string   # branch to pick the artifact; defaults to master branch
    tag: string  # picks the artifacts from the build with given tag.
    trigger:   # Optional; Triggers are enabled by default.
      branches:  # branch conditions to filter the events, optional; Defaults to all branches.
        include: [ string ]  # branches to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # branches to discard the trigger events, optional; Defaults to none.  
```

The source specific properties like version, branch, tag etc. and the trigger definition can change based on the type of `build` resource. The above schema is for Jenkins.

### Examples

The inputs for the `build` resource can change based on the `type` of the build service.

```yaml
resources:
  builds:
  - build: Spaceworkz
    type: Jenkins
    connection: MyJenkinsServer 
    source: SpaceworkzProj   # name of the jenkins source project
```

### `trigger` for builds

Triggers are enabled by default unless expliciltly opted out.

To disable the triggers on the `build` resource:

```yaml
resources:
  builds:
  - build: Spaceworkz
    type: Jenkins
    connection: MyJenkinsServer 
    source: SpaceworkzProj   
    trigger: none
```

You can control which branches get the triggers.

```yaml
resources:
  builds:
  - build: Spaceworkz
    type: Jenkins
    connection: MyJenkinsServer 
    source: SpaceworkzProj    
    trigger: 
      branches:
        include: 
        - releases/*
        exclude:
        - master
 ```

### `downloadBuild` for builds

All artifacts from the defined `build` resources are automatically downloaded and made available at the beginning of each job. However, you can override this behavior using `downloadBuild` macro.

### Schema

```yaml
- downloadBuild: string # identifier for the resource from which to download artifacts
  name: string # identifier for the artifact to download; if left blank, downloads all artifacts associated with the resource provided
  patterns: string | [ string ] # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  root: string # the directory in which to download files, defaults to $PIPELINES_RESOURCESDIR
```

### Examples

```yaml
- job: deploy_windows_x86_agent
  steps:
  - downloadBuild: Spaceworkz   # build resource identifier.
    name: WebTier1  # artifact to download, optional; defaults to all the artifacts from the resource.
    patterns: '**/*.zip'  # mini match pattern to download specific files, optional; defaults to all files.
```

Or to avoid downloading any of the artifacts at all:

```yaml
- downloadBuild: none
```

Artifacts from the `build` resource are downloaded to `$PIPELINES_RESOURCESDIRECTORY/<build-identifier>/<artifact-identifier>` folder.



## Resources: `repositories`

If you have multiple repositories from which you need to sync the code into your pipeline, you can consume the repos by defining a `repository` resource. A repository can be another Azure Repo or any external repo like GitHub etc. 

You can define triggers for your `repository` resource. Repository triggers can be of two types.
Normal triggers: Events raised upon commit activity on the `repository` resource. By default, triggers are enabled on the `repository` resource.
PR triggers: Events raised when a pull request is raised to merge into `repository` resource. By default, pr triggers are disabled and you can to explicitly opt in. 

You can take advantage of these two triggers to run your production and testing pipelines.


### Schema

```yaml
resources:          # types: pipelines | repositories | containers | packages
  repositories:
  - repository: string # identifier for the repository resource      
    type: enum # type of the repository source like AzureRepos, GitHub etc. In future this can extend to other source types
    connection: string # service connection to connect to the source, defaults to primary source connection
    source: string  # source repository to fetch
    ref: string  # ref name to use, defaults to 'refs/heads/master'
    trigger:  # Optional; Triggers are enabled by default
      batch: boolean 
      branches:  # branch conditions to filter the events, optional; Defaults to all branches.
        include: [ string ]  # branches to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # branches to discard the trigger events, optional; Defaults to none.  
      paths:
        include: [ string ]  # file paths to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # file paths to discard the trigger events, optional; Defaults to none.  
    pr:        # Optional; pr triggers are disabled by default
      branches:  # branch conditions to filter the events, optional; Defaults to all branches.
        include: [ string ]  # branches to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # branches to discard the trigger events, optional; Defaults to none.  
      paths:
        include: [ string ]  # file paths to consider the trigger events, optional; Defaults to all branches.
        exclude: [ string ]  # file paths to discard the trigger events, optional; Defaults to none.
```

### Examples

You can consume a github repo as a `repository` resource.

```yaml
resources:         
  repositories:
  - repository: secondaryRepo      
    type: GitHub
    connection: myGitHubConnection
    source: Microsoft/alphaworz
```

### `trigger` in repositories

Triggers are enabled by default and any new change to your repo will trigger a new pipeline run automatically.

You can disable triggers on your repository.

```yaml
resources:         
  repositories:
  - repository: secondaryRepo      
    type: GitHub
    connection: myGitHubConnection
    source: Microsoft/alphaworz
    trigger: none
```

You can control which branches to get triggers with simple syntax.

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    trigger:
      - master
      - releases/*
```

You can specify branches and paths to include and exclude.

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    trigger:
      branches:
        include:
        - features/*
        exclude:
        - features/experimental/*
      paths:
        exclude:
        - README.md
```


If you have a lot of team members uploading changes often, then you might want to reduce the number of builds you're running. If you set batch to true, when a build is running, the system waits until the build is completed, then queues another build of all changes that have not yet been built.

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    trigger:
      batch: true
      branches:
        - master
```

### `pr` triggers for repositories
Unless you specify, `pr` triggers are disabled for your repository. 

You can enable pull request based pipeline runs.

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    pr: true
```

You can control the target branches for your pull request based pipeline runs by simple syntax.

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    pr: 
    - master
    - releases/*
```

You can specify the branches and file paths to include and exclude.

```yaml
resources:         
  repositories:
  - repository: myPHPApp      
    type: GitHub
    connection: myGitHubConnection
    source: ashokirla/phpApp
    pr:
      branches:
        include:
        - releases/*
        exclude:
        - releases/experimental/*
      paths:
        include:
        - web/*
        exclude:
        - web/README.md
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

The repo is checked-out to `$(PIPELINE.RESOURCESDIRECTORY)/<pipeline-identifier>/` folder.

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

`self` checkout directory: `$(PIPELINE.SOURCESDIRECTORY)`
other repositories' checkout directory: `$(PIPELINE.RESOURCESDIRECTORY)/<repository-identifier>/`



## Resources: `containers`

If you need to consume a container image as part of your CI/CD pipeline, you can achieve it using `container`. A container can be an Azure Container Registry or any external Docker registry. You can enable triggers on the `container` resource. A new pipeline run get triggered whenever an image is published.

### Schema

```yaml
resources:          # types: pipelines | repositories | containers | packages
  containers:
  - container: string # identifier for the container resource      
    type: enum # type of the registry like ACR, DockerHub etc. 
    connection: string # service connection to connect to the image registry, defaults to ACR??
    image: string # container image name, Tag/Digest is optional; defaults to latest image
    options: string # arguments to pass to container at startup
    env: { string:string } # list of environment variables to add
    trigger:
      tags:
        include: [ string ]  # image tags to consider the trigger events, optional; defaults to any new tag
        exclude: [ string ]  # image tags on discard the trigger events, option; defaults to none
      location: [ string ]  # geo-location the image published to; ACR specific setting.
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

Triggers are enabled by default on the `container` resource. When a new image gets published to your image registry, pipeline run starts automatically. You can disable the triggers on your `container`.

```yaml
resources:         
  containers:
  - container: smartHotel 
    type: Docker
    connection: myDockerRegistry
    image: smartHotelApp 
    trigger: none
```

You can specify the image tag format to get the trigger by simple syntax.
```yaml
resources:         
  containers:
  - container: smartHotel 
    type: Docker
    connection: myDockerRegistry
    image: smartHotelApp 
    trigger:
    - version-*
```

You can specify the image tags to include and exclude.
```yaml
resources:         
  containers:
  - container: smartHotel 
    type: Docker
    connection: myDockerRegistry
    image: smartHotelApp 
    trigger:
      tags:
        include:
        - version-*
        exclude:
        - version-2017*
```

If you have an ACR `container` resource, you can specify the geo location to get the triggers.

```yaml
repositories:
  containers:
  - container: MyACR    
    connection: RMPM
    registry: contosodemo
    image: Microsoft/alphaworz
    trigger: 
      tags:
        include: 
        - production*
      location: 
      - east-US
      - west-EU
```

Once you define a container as resource, container image metadata passed to the pipeline in the form of variables. Information like image, registry and connection details are made accessible across all the jobs so that your kubernetes deploy tasks can extract the image pull secrets and pass it to the cluster.

