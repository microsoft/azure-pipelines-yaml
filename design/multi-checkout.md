# Multi-checkout: checking out multiple repos

Customers have often asked for the ability to use multiple repositories.
Today, we offer a weak workaround: a script step with the additional `git clone`.
You lose access to things like our smart re-use of clones (when running on a private agent).
Also, we tell you how to re-use the `System.AccessToken` for other Azure Repos, but that doesn't generalize to other source providers.

This feature will add multi-repo checkout as a first-class citizen in YAML pipelines.
(Note: we already allow [repository resources](https://docs.microsoft.com/azure/devops/pipelines/yaml-schema?tabs=schema#repository-resource), but they're used only for YAML templates today.)

Goals:
- Allow fetching multiple repositories, regardless of source host
- Allow triggering pipelines from changes to any of the repositories
- Provide useful/sane defaults for common scenarios, while allowing flexibility for customers with very specific requirements
- Preserve back-compat with existing pipelines, tasks, ad-hoc scripts, and template inclusion for customers who don't opt into multi-checkout

Non-goals:
- Adding a second checkout step is perfectly seamless (customers who opt into multiple checkouts may have to alter other aspects of their pipelines)
- Alter our existing behaviors when there are 0 or 1 `checkout` steps
- Design for multiple repository triggers (look for a sibling document shortly)
- Change our security posture for additional Azure Repos (other features may address this in the future)

## Customer scenarios

Customers often have source, tools, scripts, or other collateral stored in a secondary repo.
Even a big mono-repo like the Azure DevOps codebase has adjunct repositories that could be used in a CI/CD pipeline.
Especially for teams working with microservices, it's common to require additional sidecar repos containing shared infrastructure or SDKs.

On the other hand, we have lots of customers using just a single repository in their pipeline, and they plan to stay that way.
Any features we build must respect the investment those customers have made in their pipelines.
Therefore we have this stake in the ground:

> We are in "multi-checkout" when the customer has explicitly asked for multiple repositories to be checked out.
> Zero checkout steps yields the same behavior as today (checkout `self` in a normal job, no checkout in a `deployment` job).
> One checkout (whether it's `self` or not) preserves existing single-repo behavior.

## Multi-checkout mode

_(Note: I'm just calling it a "mode" for purposes of discussion here. We won't document it as a "mode".)_

The agent will look at the job message and discover all `checkout` steps.

### Zero or one checkout step
0. If there are none at all, we'll preserve the existing default behavior, acting as if a `checkout: self` were the first step.
0. If there is exactly one `checkout: none`, then we'll preserve the existing behavior: no repositories are synced or checked out.
0. If there is exactly one `checkout` step that isn't `none`, then we'll use existing single-checkout behavior but check out that repository rather than `self`.

### More than one checkout step
0. If there are multiple checkout steps and one or more are `checkout: none`, this is an error.
It's unclear what the user meant for us to do, so we fail with a clear message.
0. Otherwise, we are in multi-checkout mode.

Notably, a user may request the same repo to be checked out multiple times at the same or different refs.
There are niche scenarios around code gen, trying out different versions of a tool, etc. which are interesting to support.
For purposes of source mapping (an agent feature to avoid re-fetching), we'll consider only the first `checkout` step mentioning a particular repo.
If it's used again, it's OK to re-fetch the same data.
(But this is not part of the contract, and we could change behavior in the future based on customer demand/feedback.)

### Behavior changes in multi-checkout mode

#### Checkout directory
By default, all repositories will be checked out as if they've been `git clone`d into the `Build.SourcesDirectory` directory.
Git uses the last part of the path, minus any trailing slashes, spaces, and the string `.git`.
(We have YAML syntax for selecting a different path, and if the user specifies that, we'll use their name.)
This changes the default directory for the `self` repo: it would be `$(Pipeline.Workspace)/s` if it were the only `checkout` step, but now it will be `$(Pipeline.Workspace)/s/<reponame>`.

The pipeline workspace directory has four well-known subdirectories:
- `s/` for sources
- `a/` for artifacts
- `b/` for generated binaries
- `testResults/` for test results files

We'll recommend not to use any of these as checkout paths, but we won't attempt to block them.

#### System variables
No change to the contents of system variablies like `Build.SourcesDirectory` or `System.DefaultWorkingDirectory`.
Those will remain `$(Pipeline.Workspace)/s`.

**NOTE**: if one or more `checkout`s sets an explicit path outside of `$(Pipeline.Workspace)/s`, it's ambiguous where the "root of source" should be.
Since we don't know the user's or the task's intent, we won't try to be clever and do something else.
We'll document this behavior so that it's learnable.

## Resource specification

The `resources` block already allows a `repositories` list.
Today, those repositories hold only YAML templates, so we can't blindly check them out automatically.
That's why an explicit `checkout` step is also required.

However, in multi-checkout scenarios, this leads to unnecessary verbosity.
First you have to mention the repository up top, give it a name, and then you check it out by name.
To combat this, we'll add an inline repository syntax.
Inline syntax will cover the `type`, `name`, and `ref` of a repository resource.
(If other changes are needed - such as `endpoint` or `trigger` - users must fall back to explicit resource syntax.)

## Supported repo types

Supported repo types will match what YAML supports:
- Azure Repos
- GitHub
- External Git (but not for the triggering repo)

If and when other providers are supported for YAML, they will be supported by this feature as well.

## Syntax examples

### Basic syntax with inline resource

The most basic, and probably most common form of multi-checkout:
```yaml
steps:
- checkout: self
- checkout: git://MyProject/MyToolsRepo
```

This grabs a tools repository from Azure Repos and checks out its default branch, typically `master`.
Assuming this is in a repository called "MySourceRepo", the resulting directory structure on a hosted agent would be:

```
_w/
   1/
     MySourceRepo/
         (files from MySourceRepo)
     MyToolsRepo/
         (files from MyToolsRepo)
```

### Basic syntax, GitHub source provider

The same syntax works for GitHub:

```yaml
steps:
- checkout: self
- checkout: github://MyGitHubOrg/MyToolsRepo
```

And this yields the same directory structure.

### Control directory structure

You can tell the `checkout` step where to put your repos.

```yaml
steps:
- checkout: self
  path: src
- checkout: github://MyGitHubOrg/MyToolsRepo
  path: toolzdir
```

```
_w/
   1/
     src/
         (files from MySourceRepo)
     toolzdir/
         (files from MyToolsRepo)
```

### Inline ref syntax

```yaml
steps:
- checkout: self
- checkout: git://MyProject/MyToolsRepo@features/mytools
```

By appending `@<ref>`, the agent can be instructed to check out a different ref.
In this case, it's assumed to be a branch called `features/mytools`.
Branches can also be prefixed with `refs/heads/` to make it unambiguous that it's a branch.

Other valid refs (and ref-like things):
- `refs/tags/<tagname>`
- commit ID

### Non-inline syntax

If you need an endpoint or other extended resource field, you must use explicit resource syntax.

```yaml
resources:
  repositories:
  - repository: tools
    type: github
    endpoint: MyServiceConnection

steps:
- checkout: self
- checkout: tools
```

### Other examples

This one is not a multi-checkout scenario, but is a new capability:
```yaml
steps:
- checkout: git://MyProject/MySourceInAnotherRepo
```

The contents of `MySourceInAnotherRepo` will be checked out into the typical `s/` directory.
All other behaviors will remain identical.

This one is an error:
```yaml
steps:
- checkout: none
- checkout: self  # error! `checkout: none` must be the only checkout step
```

### Triggers example

This one will trigger on the main repo with the YAML file as well as the `app` repo.

```yaml
trigger:
- master

resources:
  repositories:
  - repository: app
    type: github
    name: org1/repoA
    ref: master
    endpoint: 'GitHub endpoint'
    trigger:
    - master
    - release/*
```

The default refs for `self` and `app` repos will be the latest commit from `master` in the respective repos. But, you can change that when you manually start a run. Similarly, if this pipeline is triggered by a change to `app` repo, then your checkout tasks will pull that change for the `app` repo and the latest from `master` for `self` repo.