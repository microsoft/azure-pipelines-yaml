# Pipeline Artifacts YAML shortcut

**Status: Ready for dev design**

Pipeline Artifacts are the new way to move files between jobs and stages in your pipeline. They are hosted in Azure Artifacts and will eventually entirely replace FCS "Build Artifacts". Because moving files between jobs and stages is a crucial part of most CI/CD workflows, and because Pipeline Artifacts are expected to be the default way to do so, this spec proposes a YAML shortcut syntax for uploading and downloading artifacts. It also covers default download behaviors.

In this document, `artifact` refers specifically to uploading Pipeline Artifacts from the current pipeline. `download` refers to downloading Pipeline Artifact artifacts from the current pipeline and from other Azure Pipeline. Other pipeline systems (e.g. Jenkins) will be handled as [pipeline resources](pipeline-resources.md).

Artifacts are distinct from other `resources` types, including `containers`, `repositories`, `packages`, and `feeds`.

**Note**: although this spec describes the behavior of the upload/download keywords, the tasks behave similarly when inputs are missing or left default.

## Uploading artifacts: `upload`

`upload` is a shortcut for the [Upload Pipeline Artifact](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/publish-pipeline-artifact.md) task (formerly called "Publish Pipeline Artifacts"). It will upload files from the current job to be used in subsequent jobs or in other stages or Pipeline.

### Schema

```yaml
- upload: string # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to upload
  artifact: string # identifier for this artifact (no spaces allowed), see below for default
  prependPath: string # a directory path that will be prepended to all uploaded files
  seal: boolean # if true, finalizes the artifact so no more files can be added after this step
```

Default artifact name: picking a good default relies on understanding the difference between "job" and "phase".
A "phase" is technically what's created in the YAML pipeline; a job is the running embodiment of the phase.
What we often think of as the job name is actually the phase name.
The job is named "\_\_default" unless it's part of a multi-configuration.
This wouldn't be a great artifact name.
If the job is named `__default`, then the artifact should get the phase's name (what's literally written in YAML as `- job: <name>`.
Otherwise, the name should include both the phase and job name: `<phase>.<job>`.

### Example

```yaml
- upload:
  - **/bin/*
  - **/obj/*
  artifact: webapp
  seal: true
```

---

## Downloading artifacts: `download`

`download` is a shortcut for the [Download Pipeline Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/download-pipeline-artifact) task.

It will download artifacts uploaded from a previous job, stage, or from another pipeline.

### Artifact download location

#### YAML / unified pipelines
Artifacts are automatically downloaded to the pipeline's workspace by default. We introduce a new variable for unified pipelines: `$(Pipeline.Workspace)`. `Pipeline.Workspace` is one level up from `$(System.DefaultWorkingDirectory)`; on hosted, it corresponds to `c:\agent\_work\1\`.

If no `path` is specified, either because of automatic download _or_ because you added a `download:` entry with no path, there are some decisions made automatically for you:
- Artifacts from the current pipeline each get their own directory, e.g. `$(Pipeline.Workspace)\myartifact` for an artifact named `myartifact`.
- Other pipelines each get their own directory, e.g. `$(Pipeline.Workspace)\mypipeline` for a pipeline whose ID is `mypipeline`. (Pipeline ID comes from the current pipeline's `resource` name.)
- Artifacts from other pipelines each get a directory within their pipeline's directory. `$(Pipeline.Workspace)\mypipeline\someartifact` for the `someartifact` artifact of the `mypipeline` pipeline.

Example:
```yaml
resources:
  pipelines:
  - pipeline: mypipe

jobs:
- job: makeartifact
  steps:
  - script: ./build.sh
  - upload: outputs/**/*

- job: use1artifact
  dependsOn: makeartifact
  steps:
  # by default, a download step is injected -- see later section
  - script: ls $(Pipeline.Workspace)
    # this listing shows a folder called `makeartifact`, since the prior job didn't specify a name

- job: use2artifacts
  dependsOn: makeartifact
  steps:
  - download: mypipe   # downloads all artifacts from `mypipe`
  - download: current  # must include this, since by including a download step, we don't get automatic behavior anymore
  - script: ls $(Pipeline.Workspace)
    # this listing shows two folders: `makeartifact` and `mypipe`
```

#### Controlling download path

If the `path` key is specified, it's a relative path from `$(Pipeline.Workspace)`. Directory names are not automatically injected by the pipeline anymore (but of course, directories present in the artifact itself are still used).

Example:
```yaml
jobs:
- job: makeartifact
  steps:
  - script: ./build.sh
  - upload: outputs/**/*

- job: useartifact
  dependsOn: makeartifact
  steps:
  - download: current
    path: foo
  - script: ls $(Pipeline.Workspace)
    # listing shows one folder, "foo"
```

#### Build and RM classic pipelines
No change to current behavior. Artifacts are downloaded to `$(System.DefaultWorkingDirectory)`, which is the sources folder on Build and the artifacts folder on RM.

**Note**: this means we need a new job message type "Pipeline" in addition to the existing "Build" and "Release".

### Schema

```yaml
- download: string # identifier for the pipeline resource from which to download artifacts, optional; "current" means the current pipeline, blank means all available pipelines (including current)
  artifact: string # identifier for the artifact to download; optional
  patterns: string # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  path: string # the directory in which to download files, defaults to $(Pipeline.Worksapce); if a relative path is provided, it will be rooted from $(Pipeline.Workspace)
  displayName: string # friendly name displayed in the UI
  name: string # identifier for this step (A-Z, a-z, 0-9, and underscore)
  condition: string
  continueOnError: boolean # 'true' if future steps should run even if this step fails; default to 'false'
  enabled: true # whether or not to run this ste; defaults to 'true'
  timeoutInMinutes: number
  env: { string: string } # list of environment varibles to add
```

### Examples

```yaml
- download: current
  artifact: webapp
- download: tools-pipeline
```

## More details about Pipeline Artifacts

### Default and named artifacts

If no name is specified, the uploaded artifact should be named the same as the job name. You can also create multiple artifacts, each with their own name.

All artifacts (including the default) are automatically downloaded to each subsequent job's workspace directory (`$(Pipeline.Workspace)`). You can limit the artifacts downloaded for any job by adding a `download` step to the beginning of the job. Once you use `download` to override and download a specific artifact, all automatic artifact download behavior is disabled and you need to specify any and all artifacts you intend to download in the job.

Or to avoid downloading any of the artifacts at all:

```yaml
- download: none
```

### Multi-upload artifacts

You can add files to any artifact multiple times in the same job, in different jobs in the same stage, and in different stages in the same pipeline, or you can `seal` an artifact at any time to prevent further files from being added.

## Variables

In addition to the new `$(Pipeline.Workspace)` variable, we introduce a variable per resource indicating where it was checked out on disk. For example:
```yaml
resources:
  pipelines:
  - pipeline: foo

jobs:
- job: maker
  steps:
  - script: ./build.sh
  - upload: dist/**/*

- job: consumer
  dependsOn: maker
  steps:
  - download:   # remember, blank indicates we download all artifacts from all pipelines
  - bash: |
      echo $(Pipeline.Resources.foo)   # pipeline variable syntax
      echo $PIPELINE_RESOURCES_FOO     # environment variable syntax
  - script: |
      echo $(Pipeline.Resources.current)  # current pipeline's artifact(s) go here
      echo $(Pipeline.Resources.maker)    # or to get the specific artifact produced by the previous job
```

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
      ./my-deploy-script.sh $(Pipeline.Workspace)/Build/
```

### Specify a custom location for a build artifact

You can control the location where artifacts are downloaded using the `path` key.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/*
- job: Deploy
  steps:
  - download:
    path: from-build/
  - script: |
      ./my-deploy-script.sh $(Pipeline.Workspace)/from-build/
```

### Add to an artifact multiple times

You can add files to both the default artifact and to named artifacts until the end of the pipeline or until an `upload` step is run with the `seal` key set to `true`.

```yaml
- job: build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/netcore/**/*
    prependPath: netcore/     # prefix these files with a directory in the artifact
  - task: VSBuild@1
    inputs:
      solution: MySolution.sln
  - upload: bin/Release/**/*  # this upload goes to the same artifact
    prependPath: netfx/       # but with a different prepended path
    seal: true                # and after this, nothing else can be added
- job: deployCore
  dependsOn: build
  steps:
  - script: |
      ./my-deploy-script.sh $(Pipeline.Workspace)/build/netcore/
```

### Upload a named artifact

You can give an artifact a name, and you can upload multiple named artifacts. All artifacts are downloaded unless you specify a `download` step to limit the artifacts that are downloaded.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/WebApp/*
    artifact: WebApp
  - upload: bin/MobileApp/*
    artifact: MobileApp
- job: Deploy
  steps:
  - download:
    name: WebApp
  - download:
    name: MobileApp
  - script: ./my-deploy-script.sh $(Pipeline.Workspace)/WebApp/
  - script: ./my-xamarin-script.sh $(Pipeline.Workspace)/MobileApp/
```

### Multiple artifact downloads with explicit path

If a download step covers multiple artifacts and no explicit path is given, they're each pulled into a separate directory. With an explicit path, that explicit path becomes a containing folder for the artifacts.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/WebApp/*
    artifact: WebApp
  - upload: bin/MobileApp/*
    artifact: MobileApp
- job: Deploy
  steps:
  - download: current
    path: Apps
  - script: ./my-deploy-script.sh $(Pipeline.Workspace)/Apps/WebApp/
  - script: ./my-xamarin-script.sh $(Pipeline.Workspace)/Apps/MobileApp/
```
