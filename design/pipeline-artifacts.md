# Pipeline Artifacts YAML shortcut

**Status: PM spec**

Pipeline Artifacts are the new way to move files between jobs and stages in your pipeline. They are hosted in Azure Artifacts and will eventually entirely replace FCS "Build Artifacts". Because moving files between jobs and stages is a crucial part of most CI/CD workflows, and because Pipeline Artifacts are expected to be the default way to do so, this spec proposes a YAML shortcut syntax for publishing and downloading artifacts.

In this document, `artifact` and `downloadArtifact` refer specifically to Pipeline Artifacts. Artifacts are distinct from `resources`, which are various kinds of inputs like `pipelines`, `containers`, `repositories`, and `packages`. Ashok is driving the YAML spec for these inputs, and he and I (amullans) are in sync.

## Proposal: artifact

`artifact` is a shortcut for the [Publish Pipeline Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/publish-pipeline-artifact.md) task. It will publish files from the current job to be used in subsequent jobs or in other stages or pipelines.

### Schema

```yaml
- artifact: string | [ string ]  # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to publish
  name: string # identifier for this artifact (no spaces allowed), defaults to 'default'
  prependPath: string # a directory path that will be prepended to all published files
  seal: boolean # if true, finalizes the artifact so no more files can be added after this step
```

### Example

```yaml
- artifact:
  - **/bin/*
  - **/obj/*
  name: webapp
  seal: true
```

---

## Proposal: downloadArtifact

`downloadArtifact` is a shortcut for the [Download Pipeline Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/download-pipeline-artifact.md) task. It will download artifacts published from a previous job or stage or from another pipeline. Artifacts are downloaded either to `$PIPELINES_RESOURCESDIR` or to the directory specified in `root`.

### Option 1: fewer lines

#### Schema

```yaml
- downloadArtifact: string # identifier for the artifact to download, optional; defaults to 'default'
  patterns: string | [ string ] # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  root: string # the directory in which to download files, defaults to $PIPELINES_RESOURCESDIR

- downloadPipeline: string # identifier for the pipeline to download
  artifacts: string | [ string ] # identifier(s) for the artifacts to download
  patterns: string | [ string ] # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  version: string # the run/version of the pipeline to get artifacts from
```

#### Example

```yaml
- downloadArtifact: webapp
- downloadPipeline: P2
  artifacts: p1
  version: 2
```

### Option 2: fewer keywords

#### Schema

```yaml
- downloadArtifact: string # identifier for the pipeline from which to download artifacts, optional; defaults to 'self'
  name: string # identifier for the artifact to download
  patterns: string | [ string ] # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  root: string # the directory in which to download files, defaults to $PIPELINES_RESOURCESDIR
  version: string # the run/version of the pipeline to get artifacts from
```

### Option 3: multi-artifact in one step

#### Schema

```yaml
- downloadArtifacts: string # identifier for the pipeline from which to download artifacts, optional; defaults to 'self'
  version: string # the run/version of the pipeline to get artifacts from
  - artifact: string # identifier for the artifact to download
    patterns: string | [ string ] # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
    root: string # the directory in which to download files, defaults to $PIPELINES_RESOURCESDIR
```

## More details about Pipeline Artifacts

### Default and named artifacts

Every Pipeline run has a default artifact (named `default`). You can also create multiple artifacts, each with their own name. All artifacts (including `default`) are automatically downloaded to each subsequent job's resources directory (`$(Pipelines.ResourcesDir)`). You can limit the artifacts downloaded by adding a `downloadArtifact` step to the beginning of a job.

### Multi-publish artifacts

You can add files to any artifact multiple times in the same job, in different jobs in the same stage, and in different stages in the same pipeline, or you can `seal` an artifact at any time to prevent further files from being added.

## Examples

### Publish a build artifact and use it in a deployment

This is a simple pipeline that include a build job and a deployment job. The built artifact is provided to the deployment job using the default Pipeline Artifact.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - artifact: bin/*
- job: Deploy
  steps:
  - script: |
      TODO-some-cool-deploy-script-here $(Pipelines.ResourcesDir)/default/bin/
```

### Specify a custom location for a build artifact

You can control the location where artifacts are downloaded using the `downloadRoot` key.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - artifact: bin/*
- job: Deploy
  steps:
  - downloadArtifact:
    downloadRoot: $(Pipelines.SourcesDir)/from-build/
  - script: |
      TODO-some-cool-deploy-script-here $(Pipelines.SourcesDir)/from-build/bin/
```

### Add to an artifact multiple times

You can add files to both the default artifact and to named artifacts until the end of the pipeline or until a `artifacts` step is run with the `seal` key set to `true`.

```yaml
- job: Build .NET Core
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - artifact: bin/*
    prependPath: netcore/
- job: Build .NET Framework
  steps:
    - task: VSBuild@1
      inputs:
        solution: MySolution.sln
    - artifact: bin/*
      prependPath: netfx/
      seal: true
- job: Deploy .NET Core
  steps:
  - script: |
      TODO-some-cool-deploy-script-here $(Pipelines.ResourcesDir)/default/netcore/bin/
```

### Publish a named artifact

You can give an artifact a name, and you can publish multiple named artifacts. All artifacts are downloaded unless you specify a `downloadArtifact` step to limit the artifacts that are downloaded.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - artifact: bin/WebApp/*
    name: WebApp
  - artifact: bin/MobileApp/*
    name: MobileApp
- job: Deploy
  steps:
  - downloadArtifact: WebApp
  - downloadArtifact: MobileApp
  - script: TODO-some-cool-deploy-script-here $(Pipelines.ResourcesDir)/WebApp/bin/
  - script: TODO-xamarin-magic $(Pipelines.ResourcesDir)/MobileApp/
```