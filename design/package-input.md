# `package` resource

**Status: WIP, not ready for review**

### Schema

```yaml
inputs:
  packages:
  - package: string # the package ID (for Azure Artifacts) or the file spec (for Artifactory)
    feed: string # the name of a feed or service connection
    name: string # optional identifier
    type: string # nuget, npm, maven, etc. - only need this for Azure Artifacts feeds
    trigger:
      defaultVersion: string # 'latest', a specific version, or leave blank to specify at creation
```

### Example

