# Artifacts in Pipelines

> This is a technical design document written for the initial implementation of artifacts in Azure Pipelines.
>
> For the current documentation go to [Azure Pipelines Artifacts documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/artifacts/artifacts-overview?view=azure-devops)

----

Today, it can be challenging to use Artifacts in Pipelines, esp. when using packages from private feeds with `nuget.exe`, `dotnet`, `npm`, `mvn`, `gradle`, `pip`, `twine`, etc. Frequently, these challenges center around authentication between the client and the service, both on customers’ machines and when customers are using build services like Azure Pipelines. In Q1, we’re going to tackle the experience of using and authenticating to Artifacts from within Azure Pipelines.

## Principles

Pipelines is moving to a configuration-as-code (YAML) model, which fundamentally changes many of the assumptions the Artifacts team made about using artifacts in Pipelines the last time we made major investments here. Many of the tools we rely on have advanced the security options they make available. And, we have learned a lot from customer feedback. These principles are a distillation of those inputs.

### Support all methods of invoking supported tools

Pipelines should enable customers to invoke supported (see later section) tools to do package operations in whatever way they wish. For example, I should be able to restore NuGet packages:

- with `dotnet restore`, `nuget restore`, or `msbuild /t:restore`,
- regardless of whether I call those commands using:
  - a top-level YAML script e.g. `- script: dotnet restore`
  - a custom sh/ps/etc. script e.g. `- powershell: my-script.ps1`
  - our tasks e.g. `– NuGet@2: <parameters>`
- whether or not I’m running within a container job

### From scenario tasks to scripts and helper tasks

YAML is script-first. In other words, YAML prefers that customers use - script: steps for most operations and only lean on tasks (e.g. - AzureRM@2: steps) for complex operations that aren’t easy to script. Accordingly we should deprecate the full-featured tasks (NuGet, npm) as well as the package management auth portions of the build system tasks (.NET Core, Visual Studio Build, MSBuild, Maven, Gradle). For customers that continue to use the designer, we should continue to provide a good experience that does not have the limitations of the current tasks (which do not accept custom inputs and limit customers’ ability to control the interactions with artifacts).

### Provenance support is part of the min-bar

We should ensure that any packages published using auth that was set up by any of these tasks have the appropriate build and commit provenance information.

### Proxy support is part of the min-bar

After too many DTSes, it has become clear that many of our customers running Server and/or using private agents are working behind corporate proxies.
Going forward, all Artifacts-produced tasks must support proxies (insofar as the underlying tool supports them) before we ship.
In addition to our tasks, we should enable setting up tools that support proxies with the agent's proxy.

### Implementation principles

1. Prefer credential providers, rather than configuration files
2. If credential providers are not available for the ecosystem, prefer environment variables over tokens written to files
3. If neither credential providers or environment variables are available for the ecosystem, prefer writing tokens into a configuration file in the build's workspace directory (a level above the sources directory)
4. If none of that is possible (i.e. if the tool requires a configuration file in a particular location), either write into the file the user has provided or create a new file if they didn't provide one
5. Finally, regardless of the implementation, use a post-task to clean up any credentials

We should avoid moving any pre-existing configuration file, which may contain relative paths that we'll break by moving, at all costs.

Runs of the authenticate tasks do not "stack": running the task again will overwrite the result of previous invocations of the task.

Whatever we implement here, we need to test inside container builds and ensure that it works correctly.

## Design

### Provenance

The task will add extra claims to the build token and then look for those claims on requests coming to the Artifacts service. We should not deprecate the existing URL-based provenance, which other build systems may use.

### Good, but not overly verbose, logging

Today, several of our tasks turn on verbose logging by default. These tasks should not, unless the user explicitly enables verbose logging for the build or step.

All tasks should provide clear and concise default logs that:

- Show every command we run and its regular verbosity output
- Show the envvars and/or files we read and wrote
  - Show the envvar and/or file contents during verbose logging, along with a warning that these files may include tokens
- The feeds/service connections we were able to match up correctly
- Any unused service connections - this should also throw an error
- If a tool installer was used, the version of the tool it provided and whether or not that tool is new enough to be compatible with the authenticate task

### Maven

The `mvn` command line is used to build Java projects and can install and publish Maven packages. `mvn` supports Azure Artifacts feeds and external feeds using basic authentication.

#### Inputs

- 0+ Azure Artifacts feed names/GUIDs
- 0+ Maven-typed service connection names/GUIDs
- Agent's proxy information, if available

#### Actions

##### Authentication

The task generates a `settings.xml` in `${user.home}/.m2` with the provided feeds/service connections as [servers](https://maven.apache.org/settings.html#Servers). The feed or service connection name should be used as the `<id>`. For Azure Artifacts feeds, a `<username>` (which will be ignored) and `<password>` (the build's access token) should be provided. For service connections, these fields come directly from the service connection.

It will be up to users to ensure that in their `pom.xml`, the `<id>` matches the feed name and thus the generated `settings.xml`. The new connect to feed experience will generate correct `<id>` pom.xml snippets. To help users understand this, we should provide a message in the logs along the lines of "Note: The repository `<id>` in your `pom.xml` must match the name of your feed or service connection for this task to work correctly." We will also provide documentation (see last section) that covers how to manually construct a `settings.xml` using `SYSTEM_ACCESSTOKEN` if users need to manually control the `<id>`.

##### Proxy

If the agent's proxy information is available, the generated `settings.xml` should also include [proxy configuration](https://maven.apache.org/guides/mini/guide-proxies.html).

#### Edge cases (for documentation purposes)

The only way to change the location of `settings.xml` is via the [`mvn -s` parameter](https://stackoverflow.com/questions/25277866/maven-command-line-how-to-point-to-a-specific-settings-xml-for-a-single-command). Thus, Maven users running multiple agents on the same user profile will need to be aware that builds could clash. These users should use the documentation (see last section) that covers how to manually construct a `settings.xml` using `SYSTEM_ACCESSTOKEN` and pass it using `mvn -s`.

#### Example

```yaml
- task: MavenAuthenticate@0
  inputs:
    artifactsFeeds: codesharing-demo
    mavenServiceConnections: myMavenFeed
```

#### Task reference

```yaml
# Maven Authenticate
- task: MavenAuthenticate@0
  inputs:
    artifactsFeeds: # Optional, one or more Azure Artifacts feed names/GUIDs, comma-separated
    mavenServiceConnections: # Optional, one or more Maven-typed service connection names/GUIDs, comma-separated
```

#### Open questions

- Should we put envvars in the settings.xml, [like so](https://stackoverflow.com/questions/31251259/how-to-pass-maven-settings-via-environmental-vars)? Seems unnecessary since the settings.xml is already outside the workspace.
- Is "comma-separated" the norm for multiple inputs in YAML?

### npm

The `npm` utility (which comes with Node.js) is used to install packages and tools required for your Node.js development. `npm` supports Azure Artifacts feeds and external feeds using basic or token authentication.

#### Inputs

- 0-1 Azure Artifacts feed name/GUID
- 0+ npm-typed service connection names/GUIDs
- Agent's proxy information, if available

#### Actions

##### Authentication

The task looks for an `.npmrc` either in the root of the repo or at the user's provided location and sets appropriate [environment variables](https://docs.npmjs.com/misc/config#environment-variables) for each item provided. For example, if a user provides a service connection for npmjs with an authtoken, the task will set an environment variable:

`npm_config_//registry.npmjs.org/:_authToken=token`

If the task discovers or if a user provides an Azure Artifacts feed name, the task will set 3 environment variables:

`npm_config_//pkgs.dev.azure.com/codesharing-demo/_packaging/codesharing-demo/npm/registry/:username=codesharing-demo` 

`npm_config_//pkgs.dev.azure.com/codesharing-demo/_packaging/codesharing-demo/npm/registry/:_password=token-here`

`npm_config_//pkgs.dev.azure.com/codesharing-demo/_packaging/codesharing-demo/npm/registry/:email=npm requires email to be set but doesn't use the value`

##### Proxy

If the agent's proxy information is available, the task will set the `HTTPS_PROXY` and/or `HTTP_PROXY` variables as appropriate. We should also take the list of regexes that represent hosts that should be bypassed, check each regex against the registries found in the `.npmrc` file, and add any matches to `NO_PROXY`.

#### Examples

If an `.npmrc` exists in the root of the repo, the task will discover any Azure Artifacts feeds within your organization and set the corresponding environment variables to the build's access token. Example:

```yaml
steps:
- task: npmAuthenticate@1
```

If the `.npmrc` file cannot be found in the root of your repo, the task will fail.

The user can publish to npmjs.com by providing a service connection with an npmjs.com auth token and the npmjs.com publish URL.

```yaml
- task: npmAuthenticate@1
  inputs:
    npmServiceConnections: npmjsFeedConnection
```

Users can use Azure Artifacts feeds in other organizations or non-Azure Artifacts feeds (e.g. MyGet, npmjs.com, etc.) (either as the primary registry or as scopes, depending on what the checked-in `.npmrc` has configured) by providing one or more npm-typed service connections whose URLs match URLs in the .npmrc. Example:

```bash
# .npmrc
registry=https://pkgs.dev.azure.com/codesharing-demo/_packaging/codesharing-demo/npm/registry/
@myscope:registry=https://myget.org/npm/feed/connection
always-auth=true
```

```yaml
- task: npmAuthenticate@1
  inputs:
    npmServiceConnections: ArtifactsFeedInOtherOrganizationConnection, MyGetFeedConnection
```

Users can store the `.npmrc` in a subfolder. Example:

```yaml
- task: npmAuthenticate@1
  inputs:
    npmrc: path/to/my/.npmrc
```

#### Task reference

```yaml
# npm Authenticate
- task: npmAuthenticate@1
  inputs:
    artifactFeed: # Optional, one Azure Artifacts feed name
    npmServiceConnections: # Optional, one or more npm-typed service connection names/GUIDs, comma-separated
    npmrc: # Optional, path to your .npmrc file
```

### NuGet

`NuGet.exe`, `dotnet`, or `msbuild` (the latter two include NuGet functionality) are used to install NuGet packages required for your .NET development. All tools supports Azure Artifacts feeds and external feeds using basic authentication via the credential provider.

#### Inputs

- 0+ NuGet-typed service connection names/GUIDs
- forceReinstallCredentialProvider: Reinstall the credential provider even if already installed

##### Authentication

The tasks set appropriate [environment variables](https://github.com/Microsoft/artifacts-credprovider#environment-variables) to allow automatic authentication to all Azure Artifacts feeds within the organization and all external feeds for which service connections are provided, and ensures the credential provider is installed. To help users understand this, we should provide a message in the logs along the lines of "Note: This task sets up authentication via https://github.com/Microsoft/artifacts-credprovider, which requires NuGet 4.9.0+, dotnet 2.1.500+ or 2.2.100+, or msbuild 15.9.0+."

`VSS_NUGET_EXTERNAL_FEED_ENDPOINTS`, `VSS_NUGET_URI_PREFIXES`, and `VSS_NUGET_ACCESSTOKEN` are the environment variables we will set to configure the credential provider.

We will install the credential provider to the user directory (~/.nuget/plugins or similar) if not already installed. Once https://github.com/NuGet/Home/issues/8151 is resolved, we should consider using setting the variable `NUGET_PLUGIN_PATHS` to within the task directory instead of installing the credential provider.

##### Proxy

The task must support web proxies, but will not configure nuget/dotnet/msbuild to use the agent's proxy settings. The task documentation will discuss how to configure nuget/dotnet/msbuild to use a proxy.

#### Examples

With no inputs, the task will set the `VSS_NUGET_URI_PREFIXES` environment variables up with the build's access token. Example:

```yaml
steps:
- task: NuGetAuthenticate@0
```

Users can use Azure Artifacts feeds in other organizations or non-Azure Artifacts feeds (e.g. MyGet, NuGet.org, etc.) by providing one or more NuGet-typed service connections. Example:

```yaml
- task: NuGetAuthenticate@0
  inputs:
    nuGetServiceConnections: MyGetFeedConnection
```

Users can publish to NuGet.org or any non-Azure Artifacts source without this task. Instead, they can create a [secret variable](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#secret-variables) and run a standard push command:

```yaml
- script: dotnet nuget push -k $(nugetorg-api-key) my-package.1.0.0.nupkg
```

Users publishing to Azure Artifacts should use an instance of the task and also provide an API Key (any string) in the push command.

```yaml
- task: NuGetExeAuthenticate@0 # Or DotNetAuthenticate@0, or MSBuildAuthenticate@0
- script: dotnet nuget push -k az -s [artifacts-feed-URL] my-package.1.0.0.nupkg
```

#### Task reference

```yaml
# Authenticate nuget.exe, dotnet, and MSBuild with Azure Artifacts and optionally other repositories
- task: NuGetAuthenticate@0
  #inputs:
    #nuGetServiceConnections: MyOtherOrganizationFeed, MyExternalPackageRepository # Optional
    #forceReinstallCredentialProvider: false # Optional
```

### Python install (download)

The `pip` utility (which comes with Python) is used to install packages required by your Python scripts. `pip` supports Azure Artifacts feeds and external feeds using basic authentication.

#### Inputs

- 0+ Azure Artifacts feed names/GUIDs
- 0+ Python package download-typed service connection names/GUIDs
- Agent's proxy information, if available

#### Actions

##### Authentication

The task creates a new pip.ini/pip.conf file in the root of the workspace and sets the `PIP_CONFIG_FILE` to that path. The first feed in the following search order becomes the primary index URL; all other URLs become extra index URLs:

1. If `onlyAddExtraIndex` is set to true, no feed is set as the primary index URL
1. If any Azure Artifacts feeds are provided, the first one in the list
1. If no Azure Artifacts feeds are provided, the first service connection in the list

##### Proxy

If the agent's proxy information is available, the task will set the `HTTPS_PROXY` and/or `HTTP_PROXY` variables as appropriate.

#### Examples

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

```yaml
- task: PipAuthenticate@0
  inputs:
    pythonDownloadServiceConnections: codesharing-demo # Primary index
```

```yaml
- task: PipAuthenticate@0
  inputs:
    pythonDownloadServiceConnections:
      - codesharing-demo # Primary index
      - otherfeed # Extra index
```

Users can disable the primary index behavior and always add `extra-index-url`s by using the `onlyAddExtraIndex` input.

```yaml
- task: PipAuthenticate@0
  inputs:
    artifactFeeds: codesharing-demo # Added as extra index due to parameter below
    onlyAddExtraIndex: true
```

#### Task reference

```yaml
# Pip Authenticate
- task: PipAuthenticate@1
  inputs:
    artifactFeeds: # Optional, one or more Azure Artifacts feed names/GUIDs, comma-separated
    pythonDownloadServiceConnections: # Optional, one or more NuGet-typed service connection names/GUIDs, comma-separated
    onlyAddExtraIndex: # Optional, boolean defaults to false
```

### Python upload (publish)

The `twine` package is used to publish packages to [PyPI](https://pypi.org) and Azure Artifacts feeds. `twine` supports Azure Artifacts feeds and external feeds using basic authentication.

#### Inputs

- 0-1 Azure Artifacts feed names/GUIDs
- 0-1 Python package download-typed service connection names/GUIDs
- Agent's proxy information, if available

##### Authentication

The task creates a new .pypirc file in the root of the workspace and sets the `PYPIRC_PATH` to that path. We should remove the `publishPackageMetadata` option from v0 of the task and always publish metadata to Azure Artifacts feeds.

##### Proxy

[Twine doesn't appear to support proxies.](https://stackoverflow.com/questions/55186914/twine-upload-fails-when-using-a-proxy-server)

#### Examples

```yaml
- task: TwineAuthenticate@1
  inputs:
    artifactFeed: codesharing-demo
- script: twine upload -r codesharing-demo --config-file $(PYPIRC_PATH)
```

```yaml
- task: TwineAuthenticate@1
  inputs:
    pythonUploadServiceConnection: PyPIConnection
- script: twine upload -r PyPIConnection --config-file $(PYPIRC_PATH)
```

#### Task reference

```yaml
# Twine Authenticate
- task: TwineAuthenticate@1
  inputs:
    artifactFeed: # Optional, one feed name
    pythonUploadServiceConnection: # Optional, one Python package upload-typed service connection
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

PM and engineering should walk through this list and validate that it's complete for each package type.

### Phase 1: Prevent new usage

#### Full-featured tasks (NuGet and npm)

Once the tasks are released, we will mark the full-featured tasks (NuGet and npm) as deprecated so they don't appear in the catalog or intellisense.

#### Build system tasks (Visual Studio Build, MSBuild, Maven)

For Visual Studio Build and MSBuild, we should remove the warning that is shown when the "Restore NuGet Packages" checkbox is selected, and we should check the checkbox by default for new insertions of the task.

For Maven, we should release a new major version that removes the "Authenticate to built-in Maven feeds" checkbox.

For .NET Core, we should release a new major version that removes all auth logic from the commands we own and adds an appropriate message like "Note: to use authenticated feeds with this version of the .NET Core task, add a NuGet Authenticate task earlier in your pipeline."

### Phase 2: Draw down existing usage

Whenever the NuGet, npm, or .NET Core task fails, we should include in the error message a link to a doc that outlines the new workflow and provides guidance to help users switch. We should also work with CSS to provide guidance for helping users move forward to these tasks.

### Phase 3: Stop supporting old tasks

As usage decreases, we will eventually support the old tasks solely by asking users to migrate to the new tasks.

## Plan for keeping Tool Installer defaults and Cred Provider versions up to date

As part of this work, we will publish a set of minimum tool versions supported by Azure Artifacts, using our existing test framework and adding new tests as appropriate to prove out a set of supported tools. This will be published to the docs next to the new troubleshooting guides coming as part of the connect to feed update.

At the end of every sprint, we will evaluate new versions of tools and see if they remain compatible. If so, we will update the Tool Installer default versions and the version of the Credential Provider that ships with any of the tasks.

## FAQ

### How do I use feeds in a vendor-agnostic way?

For users who don't want to use these tasks and instead manually set up their client, we'll cover how to do so in the docs.
