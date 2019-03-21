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

## Design, ecosystem by ecosystem

### Gradle

TODO

### Maven

TODO

### npm

The `npm` utility (which comes with Node.js) is used to install packages and tools required for your Node.js development. `npm` supports Azure Artifacts feeds and external feeds using basic or token authentication.

For npm, the `use` syntax will create or update an `.npmrc` file at the root of the repo with credentials from the provided feed/service connection.

```yaml
steps:
- use: npm
  auth: MyGetFeedNpm
- script: npm ci
```

### NuGet

`NuGet.exe` and `dotnet` (which includes NuGet functionality) are used to install NuGet packages required for your .NET development. The supported authentication types are Azure Artifacts feeds and external feeds using basic or token authentication. Token authentication only works for push scenarios; for restore, you must either use a username and password or use a token (but supply it as the password in a basic authentication configuration).

For NuGet, the `use` syntax will ensure [artifacts-credprovider](https://github.com/Microsoft/artifacts-credprovider/) is available on the agent and configure environment variables with the provided feed/service connection information.

```yaml
steps:
- use: dotnet
  auth: MyGetFeedNuGetRestore
- script: dotnet restore && dotnet build # Uses MyGetFeedNuGetRestore
- script: dotnet pack
- use: dotnet
  auth: MyGetFeedNuGetPush
- script: dotnet nuget push # Uses MyGetFeedNuGetPush
```

### Python

#### Install (download)

The `pip` utility (which comes with Python) is used to install packages required by your Python scripts. `pip` supports Azure Artifacts feeds and external feeds using basic authentication.

For pip, the `use` syntax will create or update a `pip.ini`/`pip.conf` file at the root of the repo with credentials from the provided feed/service connection.

```yaml
steps:
- use: python
  auth: MyGemFuryFeed
- script: pip install requirements.txt
```

#### Upload (publish)

The `twine` package is used to publish packages to [PyPI](https://pypi.org) and Azure Artifacts feeds. `twine` supports Azure Artifacts feeds and external feeds using basic authentication.

For twine, the `use` syntax will create or update a `.pypirc` file at the root of the repo with credentials from the provided feed/service connection.

```yaml
steps:
- use: python
  auth: MyGemFuryFeed
- script: twine upload -r MyGemFuryFeed
```

## Design: Artifacts-scoped tokens

TODO: as part of this, can we get a separate token (that still represents Project Collection Build Service/Project Build Service) that is scoped to only access Artifacts resources? This reduces the impact of a leaked or stolen token.

## Transition plan from today's full-featured tasks

TODO

## FAQ

### How do I use feeds in a vendor-agnostic way?

If you'd prefer not to use the `auth` shortcut and instead manually construct your package client's configuration file (e.g. `nuget.config`, `.npmrc`, etc.), you can still access Azure Artifacts feeds. To do so:

1. Ensure that the appropriate build identity (likely `Project Collection Build Service`) has the correct level of access (likely `Reader` or `Contributor`, as desired) to your feed [using these instructions](/azure/devops/artifacts/feeds/feed-permissions#package-permissions-in-azure-pipelines).
2. Make `System.AccessToken` available to scripts and tasks by mapping it [using these instructions](variables.md#systemaccesstoken).
3. Construct your configuration file using a scripting or templating language of your choice.
