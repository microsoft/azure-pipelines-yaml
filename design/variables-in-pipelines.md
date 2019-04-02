# Pipeline variables

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

Ideally we also take this opportunity to rationalize what variables appear in what scopes.
Current scopes are agent, orchestration time, and available 
- (more?)

### What are good variables?

Since variables take up space both mentally and in the environment block, we want to be judicious about which ones we include.
Guidance on what makes a "good" variable to include:
- Useful both in expressions (Pipelines-side) and scripts (user/runtime-side)
- _Commonly_ required in an ad-hoc script or a task we ship in the box
- Relevant to detecting or dealing with the environment (e.g. that we're running in Azure Pipelines, that we're running in CI, where on disk the pipeline workspace is rooted)

Existing variables:

| Current variable | Example data | Keep, cut, or rename |
|------------------|--------------|----------------------|
| Agent.BuildDirectory | D:\a\1 |
| Agent.DisableLogPlugin.TestFilePublisherPlugin | true |
| Agent.DisableLogPlugin.TestResultLogPlugin | true |
| Agent.HomeDirectory | C:\agents\2.148.2 |
| Agent.ID | 3 |
| Agent.JobName | Job |
| Agent.JobStatus | Succeeded | _Note: cut the back-compat lower-cased version "agent.jobstatus"_ |
| Agent.MachineName | fv-az379 |
| Agent.Name | Hosted Agent |
| Agent.OS | Windows_NT |
| Agent.OSArchiecture | X64 |
| Agent.RetainDefaultEncoding | false |
| Agent.RootDirectory | D:\a |
| Agent.ServerOMDirectory | C:\agents\2.148.2\externals\vstsom |
| Agent.TempDirectory | D:\a\_temp |
| Agent.ToolsDirectory | C:/hostedtoolcache/windows |
| Agent.Version | 2.148.2 |
| Agent.WorkFolder | D:\a |
| Build.ArtifactStagingDirectory | D:\a\1\a |
| Build.BinariesDirectory | D:\a\1\b |
| Build.BuildID | 1174 |
| Build.BuildNumber | 20190401.7 |
| Build.BuildURI | vstfs:///Build/Build/1174 |
| Build.ContainerID | 2713905 |
| Build.DefinitionName | playground |
| Build.DefinitionVersion | 8 |
| Build.QueuedBy | Microsoft.VisualStudio.Services.TFS |
| Build.QueuedByID | 00000002-0000-8888-8000-000000000000 |
| Build.Reason | IndividualCI |
| Build.Repository.Clean | False |
| Build.Repository.Git.SubmoduleCheckout | False |
| Build.Repository.ID | 80ab696a-644c-497e-84fd-b1d98c470825 |
| Build.Repository.LocalPath | D:\a\1\s |
| Build.Repository.Name | container |
| Build.Repository.Provider | TfsGit |
| Build.Repository.URI | https://mattc-demo@dev.azure.com/mattc-demo/Test1/_git/container |
| Build.RequestedFor | Matt Cooper |
| Build.RequestedForEmail | macoope@microsoft.com |
| Build.RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b |
| Build.SourceBranch | refs/heads/master |
| Build.SourceBranchName | master |
| Build.SourcesDirectory | D:\a\1\s |
| Build.SourceVersion | 5d7a52ce5a6e5f3cd1a8d1ee36fd2dd3ae731121 |
| Build.SourceVersionAuthor | Matt Cooper |
| Build.SourceVersionMessage | printenv | sort |
| Build.StagingDirectory | D:\a\1\a |
| Common.TestResultsDirectory | D:\a\1\TestResults |
| Endpoint.URL.SystemVSSConnection | https://dev.azure.com/mattc-demo/ |
| System | build |
| System.ArtifactsDirectory | D:\a\1\a |
| System.CollectionID | bb420569-6e91-4163-ab87-1a5b192fd50c |
| System.CollectionURI | https://dev.azure.com/mattc-demo/ |
| System.Culture | en-US |
| System.Debug | false |
| System.DefaultWorkingDirectory | D:\a\1\s |
| System.DefinitionID | 14 |
| System.DefinitionName | playground |
| System.EnableAccessToken | SecretVariable |
| System.HostType | build |
| System.IsScheduled | False |
| System.JobAttempt | 1 |
| System.JobDisplayName | Job |
| System.JobID | 12f1170f-54f2-53f3-20dd-22fc7dff55f9 |
| System.JobIdentifier | Job.__default |
| System.JobName | __default |
| System.JobParallelismTag | Public |
| System.PhaseDisplayName | Job |
| System.PhaseID | 3a3a2a60-14c7-570b-14a4-fa42ad92f52a |
| System.PhaseName | Job |
| System.PipelineStartTime | 2019-04-01 20:03:55+00:00 |
| System.PlanID | 5e42ea59-b1aa-4240-88cf-a443c0ac38d7 |
| System.PullRequest.IsFork | False |
| System.ServerType | Hosted |
| System.TaskDefinitionsURI | https://dev.azure.com/mattc-demo/ |
| System.TaskDisplayName | Bash |
| System.TaskInstanceId | efa2bfe1-554a-50c8-79b6-ef106ad3c7c2 |
| System.TaskInstanceName | Bash |
| System.TeamFoundationCollectionURI | https://dev.azure.com/mattc-demo/ |
| System.TeamFoundationServerURI | https://dev.azure.com/mattc-demo/ |
| System.TeamProject | Test1 |
| System.TeamProjectID | 0807fc91-4393-482d-9e23-defdbb7d0857 |
| System.TimelineID | 5e42ea59-b1aa-4240-88cf-a443c0ac38d7 |
| System.WorkFolder | D:\a |
| Task.DisplayName | Bash |
| TF_BUILD | True |

Necessary new variables:

| New variable | Description | Special notes |
|--------------|-------------|---------------|
| CI | Set to "true" to match industry expectation for CI systems | Environment only, not available in expressions
| AZURE_PIPELINES | Set to "True" to differentiate from other CI systems | Environment only, not available in expressions
| Pipeline.Workspace | Root directory where all source, artifacts, etc will be placed
| Pipeline.Run.Url | https:// URL to pipeline run | [requested](https://twitter.com/_a__w_/status/1102802095474827264)
| Pipeline.Url | https:// URL to pipeline definition
| Pipeline.JobDisplayName | Matches what's in the UI
| _TODO_

### New namespace for variables

We'll introduce a new Pipeline.\* namespace for variables.
We'll bring forward the parts of Build.\* and Release.\* that make sense.
*TODO: define what those are.*
This is the desired future state: all relevant variables are available under Pipeline.\*, Agent.\*, or System.\*.
Some of these may be set conditionally, such as "only for pipelines triggered by a PR".

### Task migration

We'll audit in-box tasks for dependencies on "am I running in Build or Release?"
This behavior must be removed and replaced with correct behavior for a unified pipeline.
At least one (the VS Test task) depends on the System Host Type.
We'll introduce a new System Host Type, "Pipeline", which tasks can use to conditionally switch to new behavior.
*TODO: validate that this is a sane strategy. Right now it's a proposal.*

We'll introduce a task.json construct to signify "Pipeline-aware".
Once there are no remaining dependencies on deprecated variables, tasks should declare they are Pipeline-aware.
In-box tasks will be required to do so by some date.
We'll run an outreach campaign to push Marketplace tasks to do the same.

### Pipeline migration

We'll introduce a "compat mode" flag on the Pipeline level.
All existing designer pipelines and all YAML pipelines will default to "compat mode".
When a pipeline runs in compat mode, both the old and new namespace variables are injected.
This way, older tasks and scripts are not broken, but the new world is available.
In non-compat mode, only the Pipeline.\* variables are injected -- not Build.\* or Release.\*.

If any task in a pipeline is not Pipeline-aware, the flag cannot be unset on the pipeline.
Once the pipeline is free of non-Pipeline-aware tasks, it becomes a user option to change.
Eventually, we'll run an outreach campaign to instruct users to update their pipelines to turn off compat mode.
This may include injecting warnings in the pipeline run.

When a pipeline has compat mode turned off, non-Pipeline-aware tasks cannot be added.
In the designer, we give immediate feedback, and for YAML, we throw a YAML-compile-time failure.

For the designer, the compat mode flag is a UI checkbox.
Newly created pipelines will default to having compat mode off.

For YAML, it's a `version: 2` keyword at the root level of the file.
YAML v2 may also introduce other breaking changes; those are [documented elsewhere](https://github.com/Microsoft/azure-pipelines-yaml/pull/92).

## Variable scope

We'll ensure everything under System.\* is available at orchestration time, including for pipeline run numbers.
Variables under Pipeline.\* will also be available at orchestration time including run numbers.
*TODO: Ensure this is possible.*
Agent.\* variables are never available at orchestration time.
