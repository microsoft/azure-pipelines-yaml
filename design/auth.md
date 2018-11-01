# Use authenticated feeds in Azure Pipelines

**Status: PM spec**


Your build may require access to authenticated package sources (called 'feeds' in Azure Pipelines and [Azure Artifacts](https://docs.microsoft.com/azure/devops/artifacts)) either to restore private package dependencies or to publish your own packages. Although each tool handles authentication differently, the [`use` statement](use-statement.md) provides access to your authenticated feeds in a consistent way.

## Feed

`feeds` let you use authenticated package sources or repositories with a packaging tool like `nuget`, `pip`, or `mvn`. Use `feeds` in [`use` statements](use-statement.md) to provide connection URLs and auth tokens to those tools. Azure Artifacts feeds can be referenced by name directly in the `auth` key of the `use` statement.

### Schema

```yaml
resources:
  feeds:
  - feed: string  # an Azure Artifacts feed name or the URL of an external feed
    name: string # for external feeds, an identifier
    auth:
      token: string # token, required for token-based authentication
      username: string # username, required for basic authentication
      password: string # password, required for basic authentication
```

### Example

```yaml

resources:
  feeds:
  - feed: FabrikamFiber
  - feed: https://www.myget.org/feed/Packages/vscscode-test
    name: MyGetFeedPush
    auth:
      token: TODOBuildSecureVariableHere
  - feed: https://www.myget.org/feed/Packages/vscscode-test
    name: MyGetFeedRestore
    auth:
      username: jamal
      password: TODOBuildSecureVariableHere

```

## Scoping auth

TODO Need to think more here - how would I go about scoping the auth to a single command, rather than the whole pipeline? Can I `unuse` something?

## Tokens in files vs. tokens in env. variables

TODO Need to think more here.

## Examples of `use` and `feeds`

For example, to make packages from an Azure Artifacts feed available to `dotnet` and `npm`:

```yaml
steps:
- use: dotnet
  auth: FabrikamFiber
- use: npm
  auth: FabrikamFiber
- script: npm ci
- script: dotnet restore && dotnet build
```

To do the same for a MyGet feed with NuGet and npm packages:

```yaml
resources:
  feeds:
  - feed: https://www.myget.org/F/vscscode-test/api/v3/index.json
    name: MyGetFeedNuGetRestore
    username: jamal
    password: TODOBuildSecureVariableHere
  - feed: https://www.myget.org/F/vscscode-test/npm/
    name: MyGetFeedNpm
    token: TODOBuildSecureVariableHere
steps:
- use: dotnet
  auth: MyGetFeedNuGetRestore
- use: npm
  auth: MyGetFeedNpm
- script: npm ci
- script: dotnet restore && dotnet build
```

### Maven

TODO

### npm

The `npm` utility (which comes with Node.js) is used to install packages and tools required for your Node.js development. `npm` supports Azure Artifacts feeds and external feeds using basic or token authentication.

```yaml
resources:
  feeds:
  - feed: https://www.myget.org/F/vscscode-test/npm/
    name: MyGetFeedNpm
    token: TODOBuildSecureVariableHere
steps:
- use: npm
  auth: MyGetFeedNpm
- script: npm ci
```

### NuGet

`NuGet.exe` and `dotnet` (which includes NuGet functionality) are used to install NuGet packages required for your .NET development. The supported authentication types are Azure Artifacts feeds and external feeds using basic or token authentication. Token authentication only works for push scenarios; for restore, you must either use a username and password or use a token (but supply it as the password in a basic authentication configuration).

```yaml
resources:
  feeds:
  - feed: https://www.myget.org/F/vscscode-test/api/v3/index.json
    name: MyGetFeedNuGetRestore
    username: jamal
    password: TODOBuildSecureVariableHere
  - feed: https://www.myget.org/F/vscscode-test/api/v3/index.json
    name: MyGetFeedNuGetPush
    token: TODOBuildSecureVariableHere
steps:
- use: dotnet
  auth:
    - MyGetFeedNuGetRestore
    - MyGetFeedNuGetPush
- script: dotnet restore && dotnet build # Uses MyGetFeedNuGetRestore
- script: dotnet pack
- script: dotnet nuget push # Uses MyGetFeedNuGetPush
```

### Python

#### Install (download)

The `pip` utility (which comes with Python) is used to install packages required by your Python scripts. `pip` supports Azure Artifacts feeds and external feeds using basic authentication.

```yaml
resources:
  feeds:
  - feed: https://pypi.fury.io/some-gemfury-server/me/
    name: MyGemFuryFeed
    username: Foo
    password: TODOBuildSecureVariableHere
steps:
- use: python
  auth: MyGemFuryFeed
- script: pip install requirements.txt
```

#### Upload (publish)

The `twine` package is used to publish packages to [PyPI](https://pypi.org) and Azure Artifacts feeds. `twine` supports Azure Artifacts feeds and external feeds using basic authentication.

```yaml
resources:
  feeds:
  - feed: https://pypi.fury.io/some-gemfury-server/me/
    name: MyGemFuryFeed
    username: Foo
    password: TODOBuildSecureVariableHere
steps:
- use: python
  auth: MyGemFuryFeed
- script: twine upload -r MyGemFuryFeed
```

## FAQ

### How do I use feeds in a vendor-agnostic way?

If you'd prefer not to use the `feeds` shortcut and instead manually construct your package client's configuration file (e.g. `nuget.config`, `.npmrc`, etc.), you can still access Azure Artifacts feeds. To do so:

1. Ensure that the appropriate build identity (likely `Project Collection Build Service`) has the correct level of access (likely `Reader` or `Contributor`, as desired) to your feed [using these instructions](/azure/devops/artifacts/feeds/feed-permissions#package-permissions-in-azure-pipelines).
2. Make `System.AccessToken` available to scripts and task by mapping it [using these instructions](variables.md#systemaccesstoken).
3. Construct your configuration file using a scripting or templating language of your choice.
