# More details on `authTo` in `use`

## Tokens in files vs. tokens in env. variables

TODO Need to think more here.

## Package type-specific examples of the `use` shortcut

To make packages from an Azure Artifacts feed available to `dotnet` and packages from a MyGet feed available to `npm`:

```yaml
steps:
- use: dotnet
  authTo: MyAzureArtifactsFeedName
- use: npm
  authTo: MyGetnpmConnection
- script: npm ci
- script: dotnet restore && dotnet build
```

### Maven

TODO

### npm

The `npm` utility (which comes with Node.js) is used to install packages and tools required for your Node.js development. `npm` supports Azure Artifacts feeds and external feeds using basic or token authentication.

```yaml
steps:
- use: npm
  auth: MyGetFeedNpm
- script: npm ci
```

### NuGet

`NuGet.exe` and `dotnet` (which includes NuGet functionality) are used to install NuGet packages required for your .NET development. The supported authentication types are Azure Artifacts feeds and external feeds using basic or token authentication. Token authentication only works for push scenarios; for restore, you must either use a username and password or use a token (but supply it as the password in a basic authentication configuration).

```yaml
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
steps:
- use: python
  auth: MyGemFuryFeed
- script: pip install requirements.txt
```

#### Upload (publish)

The `twine` package is used to publish packages to [PyPI](https://pypi.org) and Azure Artifacts feeds. `twine` supports Azure Artifacts feeds and external feeds using basic authentication.

```yaml
steps:
- use: python
  auth: MyGemFuryFeed
- script: twine upload -r MyGemFuryFeed
```

## FAQ

### How do I use feeds in a vendor-agnostic way?

If you'd prefer not to use the `authTo` shortcut and instead manually construct your package client's configuration file (e.g. `nuget.config`, `.npmrc`, etc.), you can still access Azure Artifacts feeds. To do so:

1. Ensure that the appropriate build identity (likely `Project Collection Build Service`) has the correct level of access (likely `Reader` or `Contributor`, as desired) to your feed [using these instructions](/azure/devops/artifacts/feeds/feed-permissions#package-permissions-in-azure-pipelines).
2. Make `System.AccessToken` available to scripts and tasks by mapping it [using these instructions](variables.md#systemaccesstoken).
3. Construct your configuration file using a scripting or templating language of your choice.
