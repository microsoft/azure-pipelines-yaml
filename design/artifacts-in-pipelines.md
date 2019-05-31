# Artifacts in Pipelines

Today, it can be challenging to use Artifacts in Pipelines, esp. when using packages from private feeds with nuget.exe, dotnet, npm, mvn, gradle, pip, twine, etc.
Frequently, these challenges center around authentication between the client and the service, both on customers’ machines and when customers are using build services like Azure Pipelines. In Q2, we’re going to tackle the experience of using and authenticating to Artifacts from within Azure Pipelines.

## Principles

Pipelines is moving to a configuration-as-code (YAML) model, which fundamentally changes many of the assumptions the Artifacts team made about using artifacts in Pipelines the last time we made major investments here.
Many of the tools we rely on have advanced the security options they make available.
And, we have learned a lot from customer feedback.
These principles are a distillation of those inputs.

### Support all methods of invoking supported tools

Pipelines should support customers invoking a supported (see later section) tool to do a package operation in whatever way the customer wishes to do so. For example, I should be able to restore NuGet packages:

- with dotnet restore, nuget restore, or msbuild /t:restore – the supported clients,
- regardless of whether I call those commands using:
  - a top-level YAML script e.g. `- script: dotnet restore`
  - a custom sh/ps/etc. script e.g. `- powershell: my-script.ps1`
  - our tasks e.g. `– NuGet@2: <parameters>`
- whether or not I’m running within a container job

### From scenario tasks to scripts and helper tasks

YAML is script-first. In other words, YAML prefers that customers use - script: steps for most operations and only lean on tasks (e.g. - AzureRM@2: steps) for complex operations that aren’t easy to script.
Accordingly we should deprecate the full-featured tasks (NuGet, npm) as well as the package management auth portions of the build system tasks (.NET Core, Visual Studio Build, MSBuild, Maven, Gradle).
For customers that continue to use the designer, we should continue to provide a good experience that does not have the limitations of the current tasks (which do not accept custom inputs and limit customers’ ability to control the interactions with artifacts).

### No temporary configuration files

We should we should avoid writing temporary configuration files at all costs.
The user should provide us a file into which we will inject tokens (preferred) or a file location at which we will create a new configuration file with the given inputs (less preferred).

### Proxy support is part of the min-bar

After too many DTSes, it has become clear that many of our customers running Server and/or using private agents are working behind corporate proxies.
Going forward, all Artifacts-produced tasks must support proxies (insofar as the underlying tool supports them) before we ship.
In addition to our tasks, we should enable setting up tools that support proxies with the pipeline’s proxy.

## Provenance support is part of the min-bar

We should ensure that any packages published using auth that was set up by any of these tasks have the appropriate build and commit provenance information.

## Design, ecosystem by ecosystem

### Maven

The `mvn` command line is used to build Java projects and can install and publish Maven packages. `mvn` supports Azure Artifacts feeds and external feeds using basic authentication.

The Maven Authenticate task accepts one or more Azure Artifacts feed names and one or more Maven-typed service connection names. If a pom.xml exists in the root of your repo, any Azure Artifacts feeds within your organization will be automatically authenticated. Example:

```yaml
- task: MavenAuthenticate@0
```

More TBD.

### npm

The `npm` utility (which comes with Node.js) is used to install packages and tools required for your Node.js development. `npm` supports Azure Artifacts feeds and external feeds using basic or token authentication.

The npm Authenticate task accepts one Azure Artifacts feed and one or more npm-typed service connection names. If a .npmrc exists in the root of your repo, any Azure Artifacts feeds within your organization will be automatically authenticated. Example:

```yaml
steps:
- task: npmAuthenticate@1
```

If no feeds are provided and the.npmrc file cannot be found in the root of your repo, this task will fail.

If you don't have a .npmrc and want the task to create one for you, you can provide a single Azure Artifacts feed or a single npm-typed service connection name. This will create an .npmrc with the `registry=` line set to the feed you provide. Examples:

```yaml
- task: npmAuthenticate@1
  inputs:
    artifactsFeed: codesharing-demo
```

 Use this to publish to npmjs.com.

```yaml
- task: npmAuthenticate@1
  inputs:
    externalFeeds: npmjsFeedConnection
```

If you want to use Azure Artifacts feeds in other organizations or non-Azure Artifacts feeds (e.g. MyGet, npmjs.com, etc.) to access scoped packages, provide one or more npm-typed service connections whose URLs match URLs in your .npmrc. Example:

```bash
# .npmrc
registry=https://pkgs.dev.azure.com/codesharing-demo/_packaging/codesharing-demo/npm/registry/
@myscope:registry=https://myget.org/npm/feed/connection
always-auth=true
```

```yaml
- task: npmAuthenticate@1
  inputs:
    externalFeeds: MyGetFeedConnection
```

If you store your .npmrc in a subfolder, you can tell the task. Example:

```yaml
- task: npmAuthenticate@1
  inputs:
    npmrc: path/to/my/.npmrc
```

Task reference:

```yaml
# npm Authenticate
- task: npmAuthenticate@1
  inputs:
    artifactFeed: # Optional, one feed name
    externalFeeds: # Optional, one npm-typed service connection or a YAML list of the same
    npmrc: # Optional, path to your .npmrc file
```

### NuGet

`NuGet.exe` and `dotnet` (which includes NuGet functionality) are used to install NuGet packages required for your .NET development. The supported authentication types are Azure Artifacts feeds and external feeds using basic or token authentication. Token authentication only works for push scenarios; for restore, you must either use a username and password or use a token (but supply it as the password in a basic authentication configuration).

The NuGet Authenticate task accepts one or more Azure Artifacts feed names and one or more NuGet-typed service connection names. If a NuGet.config and a solution, packages.config, or csproj exists in the root of your repo, any Azure Artifacts feeds within your organization will be automatically authenticated. Example:

```yaml
- task: NuGetAuthenticate@0
```

If no feeds are provided and either the project/solution file or the NuGet.config file cannot be found in the root of your repo, this task will fail.

If you want to use Azure Artifacts feeds in other organizations or non-Azure Artifacts feeds (e.g. MyGet, NuGet.org, etc.), provide one or more NuGet-typed service connections. Example:

```yaml
- task: NuGetAuthenticate@0
  inputs:
    externalFeeds: MyGetFeedConnection
```

If you store your NuGet.config and/or solution, packages.config or csproj file in a subfolder, you can tell the task. Example:

```yaml
- task: NuGetAuthenticate@0
  inputs:
    projectFile: path/to/my/project.sln
    nugetConfig: path/to/my/nuget.config
```

If you don't have a NuGet.config, the task will generate one for you with any combination of Azure Artifacts feeds and service connections you provide.

```yaml
- task: NuGetAuthenticate@0
  inputs:
    artifactFeeds: MyFeed
    externalFeeds: MyGetFeedConnection
```

Task reference:

```yaml
# NuGet Authenticate
- task: NuGetAuthenticate@0
  inputs:
    artifactFeeds: # Optional, one feed name or a YAML list of feed names
    externalFeeds: # Optional, one NuGet-typed service connection or a YAML list of the same
    nugetConfig: # Optional, path to your NuGet.config file
    projectFile: # Optional, path to your solution, packages.config, or csproj
```

### Python

#### Install (download)

The `pip` utility (which comes with Python) is used to install packages required by your Python scripts. `pip` supports Azure Artifacts feeds and external feeds using basic authentication.

The Pip Authenticate task accepts one or more Azure Artifacts feed names and one or more Python package download-typed service connection names. If only one Azure Artifacts feed is provided, it is assumed to be the primary index URL (this makes upstreams work correctly). If multiple Azure Artifacts feeds are provided, the first one is assumed to be the primary index URL. Examples:

```yaml
- task: PipAuthenticate@0
  inputs:
    artifactFeeds: codesharing-demo # Primary index
```

```yaml
- task: PipAuthenticate@0
  inputs:
    artifactFeeds:
      - codesharing-demo # Primary index
      - otherfeed # Extra index
```

The same is true if no Azure Artifacts feeds are provided and only one service connection is provided.

```yaml
- task: PipAuthenticate@0
  inputs:
    externalFeeds: codesharing-demo # Primary index
```

```yaml
- task: PipAuthenticate@0
  inputs:
    externalFeeds:
      - codesharing-demo # Primary index
      - otherfeed # Extra index
```

The first Azure Artifacts feed provided always takes precedence as the primary index.

Task reference:

```yaml
# Pip Authenticate
- task: PipAuthenticate@1
  inputs:
    artifactFeeds: # Optional, one feed name or a YAML list of feed names
    externalFeeds: # Optional, one Python package download-typed service connection or a YAML list of the same
```

#### Upload (publish)

The `twine` package is used to publish packages to [PyPI](https://pypi.org) and Azure Artifacts feeds. `twine` supports Azure Artifacts feeds and external feeds using basic authentication.

The Twine Authenticate task accepts one Azure Artifacts feed name or one Python package upload-typed service connection name. Examples:

```yaml
- task: TwineAuthenticate@1
  inputs:
    artifactFeed: codesharing-demo
- script: twine upload -r codesharing-demo --config-file $(PYPIRC_PATH)
```

```yaml
- task: TwineAuthenticate@1
  inputs:
    externalFeed: PyPIConnection
- script: twine upload -r PyPIConnection --config-file $(PYPIRC_PATH)
```

We should remove the `publishPackageMetadata` option and always do this for publishes to Azure Artifacts feeds.

Task reference:

```yaml
# Twine Authenticate
- task: TwineAuthenticate@1
  inputs:
    artifactFeed: # Optional, one feed name
    externalFeed: # Optional, one Python package upload-typed service connection
```

## Transition plan from today's full-featured tasks

To ensure we can provide the best support to customers, we need to implement a plan to get existing builds moved forward from the old tasks. Here's what that looks like.

### Phase 0: Release new tasks with plentiful documentation and samples

First, we need to build and release all of the new tasks and update all of the docs and YAML templates to show how to achieve common scenarios with the new tasks. Documentation should show scenarios like:

- Publish package(s) to the registry of record (nuget.org, npmjs.com, Maven Central)
- Install public packages through an Azure Artifacts feed with an upstream source to the registry of record
- Install public and private packages through an Azure Artifacts feed
- Install private packages through a service connection
- Publish packages to an Azure Artifacts feed
- Publish packages to a service connection
- Install private packages from inside a container job
- Publish packages from inside a container job

### Phase 1: Prevent new usage

#### Full-featured tasks (NuGet and npm)

Once the tasks are released, we will mark the full-featured tasks (NuGet and npm) as deprecated so they don't appear in the catalog or intellisense.

#### Build system tasks (Visual Studio Build, MSBuild, Maven)

For Visual Studio Build and MSBuild, we should remove the warning that is shown when the "Restore NuGet Packages" checkbox is selected, and we should check the checkbox by default for new insertions of the task.

For Maven, we should release a new major version that removes the "Authenticate to built-in Maven feeds" checkbox.

### Phase 2: Draw down existing usage

Whenever the NuGet or npm task fails, we should include in the error message a link to a doc that outlines the new workflow and provides guidance to help users switch. We should also work with CSS to provide guidance for helping users move forward to these tasks.

### Phase 3: Stop supporting old tasks

TODO: Need to understand Pipelines policy on when we can stop supporting old task versions and how we communicate that.

## FAQ

### How do I use feeds in a vendor-agnostic way?

If you'd prefer not to use the `auth` shortcut and instead manually construct your package client's configuration file (e.g. `nuget.config`, `.npmrc`, etc.), you can still access Azure Artifacts feeds. To do so:

1. Ensure that the appropriate build identity (likely `Project Collection Build Service`) has the correct level of access (likely `Reader` or `Contributor`, as desired) to your feed [using these instructions](/azure/devops/artifacts/feeds/feed-permissions#package-permissions-in-azure-pipelines).
2. Make `System.AccessToken` available to scripts and tasks by mapping it [using these instructions](variables.md#systemaccesstoken).
3. Construct your configuration file using a scripting or templating language of your choice.
