# Multiple repo checkout

## Scenarios

1. My code is in one repository, but I have tools / scripts / dependencies in another repo that I need to build my code.
2. My code is one repository, but my pipeline YAML is in another.
3. My code is split up across multiple repos.
4. I want to control the checkout location of my code.

## YAML

```yaml
resources:
  repositories:
  - repository: tools
    type: git
    name: ToolsRepo
  - repository: scripts
    type: github
    name: ContosoOrg/ScriptsRepo
    endpoint: GitHubApp1

steps:
- checkout: self
- checkout: tools
  root: buildTools
- checkout: scripts
```

In the above example, three repos will end up checked out.
`self` is the usual repo where the pipeline YAML was found.
It will be checked out in the default location, `Build.SourcesDirectory`.
`tools` is another repo in the same Azure DevOps project.
It will be checked out into `buildTools/`, a subdirectory of a new "resources" directory.
Finally, `scripts` is a repo on GitHub.
It will be checked out into its default location, `scripts/` (driven by the resource name) underneath the resources directory.

## Schema

```yaml
- checkout: none | self | resourceName
  clean: boolean      # whether to fetch clean each time
  fetchDepth: number  # the depth of commits to ask Git to fetch
  lfs: boolean        # whether to download Git-LFS files
  submodules: true | recursive  # set to 'true' for a single level of submodules or 'recursive' to get submodules of submodules
  persistCredentials: boolean   # set to 'true' to leave the OAuth token in the Git config after the initial fetch
  root: string        # directory to checkout the repo
```

## Scenario examples

### Code in one repository, tools in another repo

```yaml
TODO
```

### Code in one repository, pipeline YAML in another

```yaml
TODO
```

### Code split up across multiple repos

```yaml
TODO
```

### Control the checkout location of code

```yaml
TODO
```
