# Multiple repo checkout

## Scenarios

1. My code is in one repository, but I have tools / scripts / dependencies in another repo that I need to build my code.
2. My code is one repository, but my pipeline YAML is in another.
3. My code is split up across multiple repos.

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
It will be checked out in a default location.
`tools` is another repo in the same Azure DevOps project.
It will be checked out into `buildTools/`.
(See [the checkout path](checkout-path.md) spec for more on that feature.)
Finally, `scripts` is a repo on GitHub.
It will be checked out into its default location, `scripts/` (driven by the resource name) underneath the sources directory.

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

## Layout on disk and variables

`$(Build.SourcesDirectory)` (and someday, `$(Pipeline.SourcesDirectory)` always points to `$(Agent.WorkDir)\s`.
In single-checkout, the repo is checked out directly there.

In multi-checkout scenarios, it serves as the root for multiple repos.
`self` goes in `$(Build.SourcesDirectory)\self`.
A repository called `tools` goes in `$(Build.SourcesDirectory)\tools` by default.

The user can override the paths for any repo including `self`.
The path is relative to `$(Build.SourcesDirectory)`.
Supporting absolute paths or paths not rooted in the sources directory is a non-goal.
This leads to problems running multiple agents on a machine, plus mysterious and hard to troubleshoot permissions issues.

We also introduce a new series of variables, but only in the new `Pipeline` namespace.
`$(Pipeline.SourcesDirectory.self)` points to the actual path of the `self` repo.
`$(Pipeline.SourcesDirectory.<repoResourceName>)` points to the actual path of the repo resource with that name.

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
  path: buildTools  # relative paths rooted at $(Build.SourcesDirectory)
```

### Code in one repository, pipeline YAML in another

```yaml
resources:
  repositories:
  - repository: code
    type: git
    name: CodeRepo
    # not shown: triggers

steps:
- checkout: none
- checkout: code
  path: ''  # puts the code directly into $(Agent.WorkDir)\s as if this were a single-checkout of self
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
  path: module1
- checkout: code2
  path: module2
- checkout: code3
  path: module3
```
