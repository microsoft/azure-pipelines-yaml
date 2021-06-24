# Pipeline Artifacts YAML shortcut

**Status: Under development**

Pipeline Artifacts are the new way to move files between jobs and stages in your pipeline. They are hosted in Azure Artifacts and will eventually entirely replace "Build Artifacts" (which are stored in the File Container Service). Because moving files between jobs and stages is a crucial part of most CI/CD workflows, and because Pipeline Artifacts are expected to be the default way to do so, this spec proposes a YAML shortcut syntax for uploading and downloading artifacts. It also covers default download behaviors.

In this document, `artifact` refers specifically to uploading Pipeline Artifacts from the current pipeline. `download` refers to downloading Pipeline Artifact artifacts from the current pipeline and from other Azure Pipeline. Other pipeline systems (e.g. Jenkins) will be handled as [pipeline resources](pipeline-resources.md).

Artifacts are distinct from other `resources` types, including `containers`, `repositories`, `packages`, and `feeds`.

> Note: although this spec describes the behavior of the 'publish' and 'download' keywords, the corresponding tasks behave similarly with the same inputs/configuration.

## Publishing artifacts: `publish`

`publish` is a shortcut for the [Publish Pipeline Artifact](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/publish-pipeline-artifact) task. It will upload files from the current job to be used in subsequent jobs or in other stages or pipelines.

> Note: a previous version of this design referred to this task as "upload". To avoid introducing a new term, we will continue using "publish" when referring to this operation.

### Schema

```yaml
steps:
- publish: string # Required. A minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to upload
  artifact: string # Optional. Identifier for this artifact (no spaces allowed), see below for default
  prependPath: string # a directory path that will be prepended to all uploaded files
  seal: boolean # if true, finalizes the artifact so no more files can be added after this step
```

#### Default artifact name

It is recommeneded to name all published artifacts to avoid having its name change when the parent job's name or phase name changes.

Picking a good default relies on understanding the difference between "job" and "phase".
A "phase" is technically what's created in the YAML pipeline; a job is the running embodiment of the phase.
What we often think of as the job name is actually the phase name.
The job is named "\_\_default" unless it's part of a multi-configuration.
This wouldn't be a great artifact name.
If the job is named `__default`, then the artifact should get the phase's name (what's literally written in YAML as `- job: <name>`.
Otherwise, the name should include both the phase and job name: `<phase>.<job>`.

### Examples

Create an artifact named 'webapp' which includes all files under a 'bin' or 'obj' folder:

```yaml
steps:
- publish:
  - **/bin/*
  - **/obj/*
  artifact: webapp
```

---

## Downloading artifacts: `download`

`download` is a shortcut for the [Download Pipeline Artifact](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/download-pipeline-artifact) task. It will download artifacts uploaded from a previous job, stage, or from another pipeline.

> Note: a future version of this task should support downloading "build" artifacts --- i.e. artifacts published with the "Publish Build Artifacts" task --- in addition to "pipeline" artifacts.

### Schema

```yaml
steps:
- download: string - the pipeline to download from (`current` or a resource id) or the word `none`
  artifact: string - optional name of the artifact to download; otherwise all artifacts are downloaded
  patterns: string - optional minimatch patterns of files to download from the artifact(s); defaults to **/*
  path: string - the destination path for where to download the artifact(s), defaults to $(Pipeline.Workspace)
```

### Automated download behavior (deployment jobs)

In a typical CI/CD pipeline, one or more jobs might produce artifacts that get deployed by a special type of job: a [deployment jobs](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/deployment-jobs?view=azure-devops). In a deployment job, all artifacts produced in the current pipeline (and/or any artifacts from referenced pipelines) are automatically downloaded at the start of the job.

To bypass this behavior, an explicit `download` step (or task) can be added. Similarly, you can leave out `artifact`, `patterns`, or `path` properties to get some automatic behaviors (described below).

Or to avoid downloading any of the artifacts at all:

```yaml
steps:
- download: none
```

### Choosing which pipelines and artifacts to download

One `download` step will download from one pipeline, either the `current` pipeline or a pipeline's resource ID.

If `artifact` is not present, the step will download all artifacts that are part of the pipeline. Otherwise, the step will only download the specified artifact.

### Download location

#### In YAML / unified pipelines

Artifacts are automatically downloaded to the pipeline's workspace by default, which is represented by the `$(Pipeline.Workspace)` variable. `Pipeline.Workspace` is typically one level above `$(System.DefaultWorkingDirectory)`. This is the "shallowest" place where artifacts will be placed by default.

The spec that follows is lengthy because it covers details and edge cases. But for customers, it should be easy to understand:

1. If no "path" is specified, you get a directory per artifact from the current pipeline. You also get a directory per pipeline containing a directory per artifact from other pipelines.

2. As soon as you put a `download` step, you have to be explicit about all pipelines you want to download from. You still get  automatic directory layout.

3. You may mention a specific artifact, in which case only the mentioned artifact is downloaded to the automatic directory layout. If you also mention a `path`, the contents of the artifact are downloaded directly to the path provided.

##### With no `path`

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
  - publish: outputs/**/*

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

##### With `path`

The value of `path` can be either a relative or absolute path. Relative paths are resolved relative to `$(Pipeline.Workspace)`. 

If `artifact` is specified, the contents of the artifact are placed in the specified target path (no sub-directory is created), unless the target path equals `$(Pipeline.Workspace)`, in which case a sub-directory is created.

If no artifact is specified, a sub-directory is created for each artifact regardless of whether the resolved target path is `$(Pipeline.Workspace)` or not.

Example:
```yaml
jobs:
- job: job1
  steps:
  - publish: outputs/**/*
    artifact: WebTier
  - publish: daata/**/*
    artifact: DBTier

- job: job2
  dependsOn: job1
  steps:
  - download: current
    path: foo
  - script: ls $(Pipeline.Workspace)
    # listing shows one folder, "foo"
```
In this case the artifact contents for get downloaded to `$(Pipeline.Workspace)/foo/WebTier/` and `$(Pipeline.Workspace)/foo/DBTier/` respectively (separate directories under "foo" for each artifact).

#### Build and RM classic pipelines

No change to current behavior. Artifacts are downloaded to `$(System.DefaultWorkingDirectory)`, which is the sources folder on Build and the artifacts folder on RM.

### Partial downloads

If `patterns` is absent, all files from the artifact are downloaded.
If present, it's a newline-separated list of minimatch patterns for what files to download.

### Full schema

```yaml
- download: string # pipeline ID, `current`, or `none`; required
  artifact: string # identifier for the artifact to download; optional unless `path` is specified
  patterns: string # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  path: string # the relative directory in which to download files, rooted from $(Pipeline.Workspace)
  
  # these are common to all steps
  displayName: string # friendly name displayed in the UI
  name: string # identifier for this step (A-Z, a-z, 0-9, and underscore)
  condition: string
  continueOnError: boolean # 'true' if future steps should run even if this step fails; default to 'false'
  enabled: true # whether or not to run this ste; defaults to 'true'
  timeoutInMinutes: number
  env: { string: string } # list of environment variables to add
```

One instance of `- download` has to translate into exactly one underlying task, which may require changes to how the Download Artifacts task operates.

### Examples

```yaml
- download: current
  artifact: webapp
- download: tools-pipeline
```

## More details about Pipeline Artifacts

### Default and named artifacts

If no name is specified, the published artifact gets a default name.
See the section above on `publish` for details.
You can also create multiple artifacts, each with their own name.

All artifacts (including the default) are automatically downloaded to each subsequent job's workspace directory (`$(Pipeline.Workspace)`). You can limit the artifacts downloaded for any job by adding a `download` step to the beginning of the job. Once you use `download` to override and download a specific artifact, all automatic artifact download behavior is disabled and you need to specify any and all artifacts you intend to download in the job.

### Adding more files to an artifact

You can add files to any artifact multiple times in the same job, in different jobs in the same stage, and in different stages in the same pipeline, or you can `seal` an artifact at any time to prevent further files from being added.

### Variables

In addition to the new `$(Pipeline.Workspace)` variable, we introduce a variable per resource indicating where it was checked out on disk. For example:

```yaml
resources:
  pipelines:
  - pipeline: foo

jobs:
- job: maker
  steps:
  - script: ./build.sh
  - publish: dist/**/*

- job: consumer
  dependsOn: maker
  steps:
  # implicit download of all current and other pipelines' artifacts
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
  - publish: bin/*
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
  - publish: bin/*
- job: Deploy
  steps:
  - download: current
    artifact: Build
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
  - publish: bin/netcore/**/*
    prependPath: netcore/     # prefix these files with a directory in the artifact
  - task: VSBuild@1
    inputs:
      solution: MySolution.sln
  - publish: bin/Release/**/*  # this upload goes to the same artifact
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
  - publish: bin/WebApp/*
    artifact: WebApp
  - publish: bin/MobileApp/*
    artifact: MobileApp
- job: Deploy
  steps:
  - download: current
    name: WebApp
  - download: current
    name: MobileApp
  - script: ./my-deploy-script.sh $(Pipeline.Workspace)/WebApp/
  - script: ./my-xamarin-script.sh $(Pipeline.Workspace)/MobileApp/
```

### Multiple artifact downloads with explicit path

If a download step covers multiple artifacts and no explicit path is given, they're each pulled into a separate directory.
With an explicit path, the artifact is put exactly there.
This gives the user final say over on-disk layout.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)

  - publish: bin/WebApp/*
    artifact: WebApp

  - publish: bin/MobileApp/*
    artifact: MobileApp

- job: Deploy
  steps:
  - download: current
    artifact: WebApp
    path: Apps/Web
  
  - download: current
    artifact: MobileApp
    path: Apps/iOS
    
  - script: ./my-deploy-script.sh $(Pipeline.Workspace)/Apps/Web
  - script: ./my-xamarin-script.sh $(Pipeline.Workspace)/Apps/iOS
```
