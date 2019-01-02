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
  path: buildTools
- checkout: scripts
```

In the above example, three repos will end up checked out.
`self` is the usual repo where the pipeline YAML was found.
It will be checked out in the default location.
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
  path: string        # directory to checkout the repo
```

## New features needed

- Resources directory: $(Pipelines.ResourcesDirectory)
- Default source directory: $(Pipelines.DefaultSourcesDirectory)
- `self` checkout influences $(Pipelines.SourcesDirectory)

## Scenario examples

### Code in one repository, tools in another repo

```yaml
resources:
  repositories:
  - repository: tools
    type: git
    name: ToolsRepo

steps:
- checkout: self
- checkout: tools
  path: buildTools  # relative paths are assumed to be rooted at $(Pipelines.ResourcesDirectory)
```

### Code in one repository, pipeline YAML in another

```yaml
resources:
  repositories:
  - repository: code
    type: git
    name: CodeRepo
    # TODO: how does a push to CodeRepo trigger this pipeline?

steps:
- checkout: none  # Pipelines.SourcesDirectory will be unset
- checkout: code
  root: $(Pipelines.DefaultSourcesDirectory)
```

### Code split up across multiple repos

```yaml
resources:
  repositories:
  - repository: code2
    type: git
    name: moreCode
  - repository: code3
    type: git
    name: evenMoreCode

steps:
- checkout: self
  root: $(Pipelines.DefaultSourcesDirectory)/module1
- checkout: code2
  path: $(Pipelines.DefaultSourcesDirectory)/module2
- checkout: code3
  path: $(Pipelines.DefaultSourcesDirectory)/module3
```

### Control the checkout location of code

```yaml
steps:
- checkout: self
  path: $(Pipelines.DefaultSourcesDirectory)/PutMyCodeHere
- script: ./build.sh
  workingDirectory: $(Pipelines.SourcesDirectory)  # correctly set based on the self checkout step
```
