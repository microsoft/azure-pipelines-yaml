# Pipeline Artifacts YAML shortcut

**Status: Signed off**

Pipeline Artifacts are the new way to move files between jobs and stages in your pipeline. They are hosted in Azure Artifacts and will eventually entirely replace FCS "Build Artifacts". Because moving files between jobs and stages is a crucial part of most CI/CD workflows, and because Pipeline Artifacts are expected to be the default way to do so, this spec proposes a YAML shortcut syntax for uploading and downloading artifacts.

In this document, `artifact` refers specifically to uploading Pipeline Artifacts from the current pipeline. `download` refers to downloading Pipeline Artifact artifacts from the current pipeline and from other Azure Pipelines. Other pipeline systems (e.g. Jenkins) will be handled as [pipeline resources](pipeline-resources.md).

Artifacts are distinct from other `resources` types, including `containers`, `repositories`, `packages`, and `feeds`.

## Uploading artifacts: `upload`

`artifact` is a shortcut for the [Upload Pipeline Artifact](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/publish-pipeline-artifact.md) task (formerly called "Publish Pipeline Artifacts"). It will upload files from the current job to be used in subsequent jobs or in other stages or pipelines.

### Schema

```yaml
- upload: string | [ string ]  # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to upload
  name: string # identifier for this artifact (no spaces allowed), defaults to 'default'
  prependPath: string # a directory path that will be prepended to all uploaded files
  seal: boolean # if true, finalizes the artifact so no more files can be added after this step
```

### Example

```yaml
- upload:
  - **/bin/*
  - **/obj/*
  name: webapp
  seal: true
```

---

## Downloading artifacts: `download`

`download` is a shortcut for two tasks:

- The [Download Pipeline Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/download-pipeline-artifact) task
- The [Download Fileshare Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/download-fileshare-artifacts) task

It will download artifacts uploaded from a previous job or stage or from another pipeline. Artifacts are downloaded either to `$PIPELINES_RESOURCESDIR` or to the directory specified in `root`.

### Schema

```yaml
- download: string # identifier for the pipeline resource from which to download artifacts, optional; defaults to `self`, which represents the current pipeline
  name: string # identifier for the artifact to download; if left blank, downloads all artifacts associated with the resource provided
  patterns: string | [ string ] # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  root: string # the directory in which to download files, defaults to $PIPELINES_RESOURCESDIR
```

### Examples

```yaml
- download:
  name: webapp
- download: tools-pipeline
```

## More details about Pipeline Artifacts

### Default and named artifacts

Every Pipeline run has a default artifact (named `default`). You can also create multiple artifacts, each with their own name. All artifacts (including `default`) are automatically downloaded to each subsequent job's resources directory (`$(Pipelines.ResourcesDir)`). You can limit the artifacts downloaded for any job by adding a `download` step to the beginning of the job. Once you use `download` to override and download a specific artifact, all automatic artifact download behavior is disabled and you need to specify any and all artifacts you intend to download in the job.

Or to avoid downloading any of the artifacts at all:

```yaml
- download: none
```

### Multi-upload artifacts

You can add files to any artifact multiple times in the same job, in different jobs in the same stage, and in different stages in the same pipeline, or you can `seal` an artifact at any time to prevent further files from being added.

## Examples

### Upload a build artifact and use it in a deployment

This is a simple pipeline that include a build job and a deployment job. The built artifact is provided to the deployment job using the default Pipeline Artifact.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/*
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
  - upload: bin/*
- job: Deploy
  steps:
  - download:
    root: $(Pipelines.SourcesDir)/from-build/
  - script: |
      TODO-some-cool-deploy-script-here $(Pipelines.SourcesDir)/from-build/bin/
```

### Add to an artifact multiple times

You can add files to both the default artifact and to named artifacts until the end of the pipeline or until an `upload` step is run with the `seal` key set to `true`.

```yaml
- job: Build .NET Core
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/*
    prependPath: netcore/
- job: Build .NET Framework
  steps:
    - task: VSBuild@1
      inputs:
        solution: MySolution.sln
    - upload: bin/*
      prependPath: netfx/
      seal: true
- job: Deploy .NET Core
  steps:
  - script: |
      TODO-some-cool-deploy-script-here $(Pipelines.ResourcesDir)/default/netcore/bin/
```

### Upload a named artifact

You can give an artifact a name, and you can upload multiple named artifacts. All artifacts are downloaded unless you specify a `download` step to limit the artifacts that are downloaded.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/WebApp/*
    name: WebApp
  - upload: bin/MobileApp/*
    name: MobileApp
- job: Deploy
  steps:
  - download: WebApp
  - download: MobileApp
  - script: TODO-some-cool-deploy-script-here $(Pipelines.ResourcesDir)/WebApp/bin/
  - script: TODO-xamarin-magic $(Pipelines.ResourcesDir)/MobileApp/
```