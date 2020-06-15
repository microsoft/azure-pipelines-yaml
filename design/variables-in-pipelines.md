# Pipeline variables

**STATUS**: This feature is cut.
Merging the PR for historical interest only.

To bring Build and Release into one "unified pipelines" model, we need to evolve the variables we expose.
The Build.\* and Release.\* variables no longer make sense.
We also have a unique opportunity to clean up other aspects of the variables we expose.
Though we can make some "breaking changes", we have to give customers, first-party tasks, and Marketplace tasks a clear and easy path forward.

Worth noting: "variables" is an overloaded term.
Azure Pipelines has a notion of "pipeline variables".
Most of these are automatically converted into environment variables on the agent.
The major exception is secrets, which must be manually mapped into the environment.
Also, individual steps can include environment-only variables in the `env` statement.

## Problems
* Build.\* and Release.\* variables aren't meaningful in unified pipelines.
* There are dozens of variables, some which are similar but not quite the same.
We're wary of adding too many more, since we've run into technical limitations before (size of environment block on some systems).
Having similar-but-different variables increases the burden on customers and task authors to understand the system.
* Some tasks operate differently based on whether they're "running in build" vs "running in RM".
Soon, that distinction won't exist.
* Despite the dozens of variables, we're missing some obvious useful ones.

## Solution

A full solution needs to include:
- desired future state
- a path to bring tasks forward without breaking them on older definitions
- a path to bring old definitions forward with minimal pain
- rationalize what variables appear in what scopes - currently confused between agent, orchestration time, and always available

### Kinds of existing variables

In the existing set of variables, we found:
- Agent compile-time values
- Agent config-time values
- Agent run-time values
- Source and artifact details
- URIs and identifiers for talking back to Azure Pipelines
- Run control options (under customer control)
- Feature flags (under product team control)

### What are good variables?

Since variables take up space both mentally and in the environment block, we want to be judicious about which ones we include.
The best variables are useful both in expressions (Pipelines-side) and scripts (user/runtime-side).
They're _commonly_ required in an ad-hoc script or a task we ship in the box.
Typically they're relevant to detecting or dealing with the environment.
For example, detecting that we're running in Azure Pipelines, that we're running in CI, or where on disk the pipeline workspace is rooted.

Some variables are less commonly needed.
These won't be propagated into the environment by default.
They'll be available in YAML via expression context.
We need to design features to make them available to the classic editor and task authors.

## New variables

### New namespace for variables

We'll introduce a new Pipeline.\* namespace for variables.
We'll bring forward the parts of Build.\* and Release.\* that make sense.
All variables are available under Pipeline.\* plus the industry-standard "CI=true" variable.
Some of these may be set conditionally, such as "only for pipelines triggered by a PR".
Many of the new series of variables are NOT in the environment by default and must be explicitly mapped in.

And those variables are...

| Variable              | Description | Expression context? | Environment? |
|-----------------------|-------------|---------------------|--------------|
| CI                    | Set to `true` to match industry expectation for CI systems | :x: | :heavy_check_mark:
| Pipeline.Provider     | Set to `Azure` to differentiate from other CI systems | :heavy_check_mark: | :heavy_check_mark:
| Pipeline.Workspace    | Root directory where all source, artifacts, etc will be placed | :heavy_check_mark: | :heavy_check_mark:
| Pipeline.Url          | https:// URL to pipeline definition | :heavy_check_mark: | :x:
| Pipeline.Run.Url      | https:// URL to pipeline run | :heavy_check_mark: | :x:
| Pipeline.Job.DisplayName | Matches what's in the UI | :heavy_check_mark: | :x:
| ...                   | Commit hash of target branch | :heavy_check_mark: | :heavy_check_mark:
| ...                   | Merge commit | :heavy_check_mark: | :heavy_check_mark:
| _TODO_

### Task migration

We'll audit in-box tasks for dependencies on "am I running in Build or Release?"
This behavior must be removed and replaced with correct behavior for a unified pipeline.
At least one (the VS Test task) depends on the System Host Type.
We'll introduce a new System Host Type, "Pipeline", which tasks can use to conditionally switch to new behavior.

We'll introduce a task.json construct to signify "Pipeline-aware".
Once there are no remaining dependencies on deprecated variables, tasks should declare they are Pipeline-aware.
In-box tasks will be required to do so by some date.
We'll run an outreach campaign to push Marketplace tasks to do the same.

### Pipeline migration

We'll introduce a "compat mode" flag on the Pipeline level.
All existing classic editor pipelines and all YAML pipelines will default to "compat mode".
When a pipeline runs in compat mode, both the old and new namespace variables are injected.
This way, older tasks and scripts are not broken, but the new world is available.
In non-compat mode, only the Pipeline.\* variables are injected -- not Build.\* or Release.\*.

If any task in a pipeline is not Pipeline-aware, the flag cannot be unset on the pipeline.
Once the pipeline is free of non-Pipeline-aware tasks, it becomes a user option to change.
Eventually, we'll run an outreach campaign to instruct users to update their pipelines to turn off compat mode.
This may include injecting warnings in the pipeline run.

When a pipeline has compat mode turned off, non-Pipeline-aware tasks cannot be added.
In the classic editor, we give immediate feedback, and for YAML, we throw a YAML-compile-time failure.

For the classic editor, the compat mode flag is a UI checkbox.
Newly created pipelines will default to having compat mode off.
**NOTE**: we need to invest in a way to map context variables into the environment on a job- and task-level.
Otherwise there's no way for classic editor pipelines to use the new variables.

For YAML, it's a `version: 2` keyword at the root level of the file.
YAML v2 may also introduce other breaking changes; those are [documented elsewhere](https://github.com/Microsoft/azure-pipelines-yaml/pull/92).

We must give task authors a way to access the context-only variables.
This is probably a command-line utility that's carried by the agent (and available separately for development).

## Variable scope

Almost all variables under Pipeline\.* are available for template expansion and plan construction time.
Pipeline.Workspace and Pipeline.Agent\.* are only available on the agent.

## Appendix: Existing variables

### Agent
| Agent. | Example data | Keep, cut, or rename | Notes |
|--------|--------------|----------------------|-------|
| .BuildDirectory | D:\a\1 | rename | replace with Pipeline.Workspace
| .DeploymentGroupId | 1 | **TODO** | only in deployment group jobs
| .Diagnostic | true | keep | non-existent by default
| .DisableLogPlugin.TestFilePublisherPlugin | true | CUT | replace with feature flag
| .DisableLogPlugin.TestResultLogPlugin | true | CUT | replace with feature flag
| .HomeDirectory | C:\agents\2.148.2 | CUT
| .ID | 3 | keep
| .JobName | Job | CUT | replace with a richer job status mechanism
| .JobStatus | Succeeded | CUT | replace with a richer job status mechanism (completely cut the back-compat lower-cased version "agent.jobstatus") |
| .MachineName | fv-az379 | keep
| .Name | Hosted Agent | keep
| .OS | Windows_NT | keep
| .OSArchitecture | X64 | keep
| .ReleaseDirectory | D:\a\r1\a | CUT | _RM only_
| .RetainDefaultEncoding | false | CUT | feature flag
| .RootDirectory | D:\a | keep
| .ServerOMDirectory | C:\agents\2.148.2\externals\vstsom | CUT
| .TempDirectory | D:\a\\_temp | CUT | use operating system temp construct instead
| .ToolsDirectory | C:/hostedtoolcache/windows | **TODO**
| .Version | 2.148.2 | keep
| .WorkFolder | D:\a | keep

### Build
| Build. | Example data | Keep, cut, or rename | Notes |
|--------|--------------|----------------------|-------|
| .ArtifactStagingDirectory | D:\a\1\a | CUT
| .BinariesDirectory | D:\a\1\b | CUT
| .BuildID | 1174 | rename | **TODO**
| .BuildNumber | 20190401.7 | rename | **TODO**
| .BuildURI | vstfs:///Build/Build/1174 | CUT
| .Clean | true | CUT | already deprecated
| .ContainerID | 2713905 | **TODO**
| .DefinitionID | 14 | CUT | same as System.DefinitionID |
| .DefinitionName | playground | CUT | same as System.DefinitionName
| .DefinitionVersion | 8 | CUT | used in telemetry, but doesn't seem useful
| .QueuedBy | Microsoft.VisualStudio.Services.TFS | rename | **TODO**
| .QueuedByID | 00000002-0000-8888-8000-000000000000 | rename | **TODO**
| .ProjectID | 0807fc91-4393-482d-9e23-defdbb7d0857 | rename | **TODO** set for RM with a build artifact
| .ProjectName | Test1 | rename | **TODO** set for RM with a build artifact
| .Reason | IndividualCI | rename | **TODO**
| .Repository.Clean | False | CUT
| .Repository.Git.SubmoduleCheckout | False | CUT
| .Repository.ID | 80ab696a-644c-497e-84fd-b1d98c470825 | CUT
| .Repository.LocalPath | D:\a\1\s | rename | **TODO**
| .Repository.Name | container | rename | **TODO**
| .Repository.Provider | TfsGit | rename | **TODO**
| .Repository.Tfvc.Workspace | ws_12_8 | rename | **TODO**
| .Repository.URI | https://mattc-demo@dev.azure.com/mattc-demo/Test1/_git/container | rename | **TODO**
| .RequestedFor | Matt Cooper | rename | **TODO**
| .RequestedForEmail | macoope@microsoft.com | rename | **TODO**
| .RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | rename | **TODO**
| .SourceTfvcShelveset | my_shelveset | rename | **TODO**
| .SourceBranch | refs/heads/master | rename | **TODO**
| .SourceBranchName | master | CUT | actual scenario is replaced by **TODO**
| .SourcesDirectory | D:\a\1\s | CUT
| .SourceVersion | 5d7a52ce5a6e5f3cd1a8d1ee36fd2dd3ae731121 | rename | **TODO**
| .SourceVersionAuthor | Matt Cooper | rename | **TODO**
| .SourceVersionMessage | my commit msg | rename | **TODO**
| .StagingDirectory | D:\a\1\a | CUT
| .TriggeredBy.BuildId | | rename | **TODO**
| .TriggeredBy.DefinitionId | | rename | **TODO**
| .TriggeredBy.DefinitionName | | rename | **TODO**
| .TriggeredBy.BuildNumber | | rename | **TODO**
| .TriggeredBy.ProjectID | | rename | **TODO**
| .Type | Build | **TODO** | set for RM with a build artifact

### Release
| Release. | Example data | Keep, cut, or rename | Notes |
|----------|--------------|----------------------|-------|
| .Artifacts.{artifact_id}.BuildID | 1174 | rename 
| .Artifacts.{artifact_id}.BuildNumber | 20190401.7 | rename 
| .Artifacts.{artifact_id}.BuildURI | vstfs:///Build/Build/1174 | CUT
| .Artifacts.{artifact_id}.DefinitionID | 14 |  rename 
| .Artifacts.{artifact_id}.DefinitionName | playground |  rename 
| .Artifacts.{artifact_id}.ProjectID | 0807fc91-4393-482d-9e23-defdbb7d0857 | rename 
| .Artifacts.{artifact_id}.ProjectName | Test1 | rename 
| .Artifacts.{artifact_id}.PullRequest.TargetBranch | refs/heads/master | rename 
| .Artifacts.{artifact_id}.PullRequest.TargetBranchName | master | rename 
| .Artifacts.{artifact_id}.Repository.ID | 80ab696a-644c-497e-84fd-b1d98c470825 | rename 
| .Artifacts.{artifact_id}.Repository.Name | container | rename 
| .Artifacts.{artifact_id}.Repository.Provider | TfsGit | rename 
| .Artifacts.{artifact_id}.RequestedFor | Matt Cooper | rename 
| .Artifacts.{artifact_id}.RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | rename 
| .Artifacts.{artifact_id}.SourceBranch | refs/heads/master | rename 
| .Artifacts.{artifact_id}.SourceBranchName | master | rename 
| .Artifacts.{artifact_id}.SourceVersion | 5d7a52ce5a6e5f3cd1a8d1ee36fd2dd3ae731121 | rename 
| .Artifacts.{artifact_id}.Type | `Build`, `Jenkins`, `TeamCity`, `Git`, ... | rename 
| .AttemptNumber | 1 | rename 
| .DefinitionEnvironmentID | 2 | **TODO**
| .DefinitionID | 2 | CUT | same as System.DefinitionID 
| .DefinitionName | My Release Pipeline | cut | same as System.DefinitionName
| .DeploymentID | 3 | rename 
| .Deployment.RequestedFor | Matt Cooper | rename 
| .Deployment.RequestedForEmail | *** | rename 
| .Deployment.RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | rename 
| .Deployment.StartTime | 2019-04-02 12:39:52Z | rename 
| .DeployPhaseID | 3 | rename 
| .EnvironmentID | 5 | CUT
| .EnvironmentName | My First Stage | rename
| .Environments.{stage_name}.Status | InProgress | CUT
| .EnvironmentURI | vstfs:///ReleaseManagement/Environment/5 | CUT
| .PrimaryArtifactSourceAlias | _playground | CUT
| .Reason | Manual | rename
| .ReleaseDescription |  | CUT
| .ReleaseID | 5 | rename
| .ReleaseName | Release-2 | rename
| .ReleaseURI | vstfs:///ReleaseManagement/Release/5 | CUT
| .ReleaseWebURL | https://dev.azure.com/mattc-demo/0807fc91-4393-482d-9e23-defdbb7d0857/_release?releaseId=5&_a=release-summary | CUT
| .RequestedFor | Matt Cooper | rename
| .RequestedForEmail | *** | rename
| .RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | rename
| .SkipArtifactsDownload | False | CUT | replace with feature flag
| .TriggeringArtifact.Alias |  | CUT

### System
| System. | Example data | Keep, cut, or rename | Notes |
|---------|--------------|----------------------|-------|
| System | `build`, `release` | CUT | _this isn't System.System -- just `System`_
| .AccessToken | (access token) | CUT | replace with **TODO**
| .ArtifactsDirectory | D:\a\1\a | CUT
| .CollectionID | bb420569-6e91-4163-ab87-1a5b192fd50c | CUT
| .CollectionURI | https://dev.azure.com/mattc-demo/ | rename | **TODO**
| .Culture | en-US | CUT
| .Debug | false | keep | non-existent by default
| .DefaultWorkingDirectory | D:\a\1\s | CUT
| .DefinitionID | 14 | rename | Pipeline.ID
| .DefinitionName | playground | rename | Pipeline.Name
| .EnableAccessToken | `SecretVariable`, `False` | CUT
| .HostType | `build`, `release`, `deployment` | CUT
| .IsScheduled | False | CUT
| .JobAttempt | 1 | rename | Pipeline.Job.Attempt
| .JobDisplayName | Job | rename | Pipeline.Job.DisplayName
| .JobID | 12f1170f-54f2-53f3-20dd-22fc7dff55f9 | CUT | used in telemetry and in 2 downlevel tasks
| .JobIdentifier | Job.__default | CUT
| .JobName | __default | CUT
| .JobParallelismTag | Public | CUT
| .JobPositionInPhase | 1 | rename | **TODO**
| .ParallelExecutionType | None | CUT
| .PhaseDisplayName | Job | CUT
| .PhaseID | 3a3a2a60-14c7-570b-14a4-fa42ad92f52a | CUT
| .PhaseName | Job | CUT
| .PipelineStartTime | 2019-04-01 20:03:55+00:00 | rename | **TODO**
| .PlanID | 5e42ea59-b1aa-4240-88cf-a443c0ac38d7 | CUT
| .PullRequest.IsFork | False | rename | **TODO**
| .PullRequest.PullRequestId | 17 | rename | **TODO** _only for Azure Repos?_
| .PullRequest.PullRequestNumber | | rename | **TODO** _only for GitHub?_
| .PullRequest.SourceBranch | refs/heads/users/raisa/new-feature | rename | **TODO**
| .PullRequest.SourceRepositoryURI | https://dev.azure.com/ouraccount/_git/OurProject | rename | **TODO**
| .PullRequest.TargetBranch | refs/heads/master | rename | **TODO**
| .ServerType | Hosted | CUT
| .TaskDefinitionsURI | https://dev.azure.com/mattc-demo/ | CUT
| .TaskDisplayName | Bash | rename | **TODO**
| .TaskInstanceId | efa2bfe1-554a-50c8-79b6-ef106ad3c7c2 | CUT
| .TaskInstanceName | `Bash`, `04bff6ce5394c41b0c048826688170a` | CUT
| .TeamFoundationCollectionURI | https://dev.azure.com/mattc-demo/ | CUT | replace with **TODO**
| .TeamFoundationServerURI | https://dev.azure.com/mattc-demo/ | CUT | replace with **TODO**
| .TeamProject | Test1 | rename | **TODO**
| .TeamProjectID | 0807fc91-4393-482d-9e23-defdbb7d0857 | CUT
| .TimelineID | 5e42ea59-b1aa-4240-88cf-a443c0ac38d7 | rename | **TODO**
| .TotalJobsInPhase | 1 | rename | **TODO**
| .WorkFolder | D:\a | CUT | same as Agent.WorkFolder

### Misc
| Variable | Example data | Keep, cut, or rename | Notes |
|----------|--------------|----------------------|-------|
| Common.TestResultsDirectory | D:\a\1\TestResults | CUT
| Endpoint.URL.SystemVSSConnection | https://dev.azure.com/mattc-demo/ | CUT
| RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | CUT | only set for RM
| Task.DisplayName | Bash | CUT
| TF_BUILD | True | CUT
