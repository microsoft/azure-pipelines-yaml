# Moving variables forward

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
* There are many dozens of variables, some which are similar but not quite the same.
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

| Current variable | Keep, cut, or rename |
|------------------|----------------------|
| _TODO_

Necessary new variables:

| New variable | Description | Special notes |
|--------------|-------------|---------------|
| CI | Set to "True" to match industry expectation for CI systems | Environment only, not available in expressions
| AZURE_PIPELINES | Set to "True" to differentiate from other CI systems | Environment only, not available in expressions
| Pipeline.Workspace | Root directory where all source, artifacts, etc will be placed
| Pipeline.Url | https:// URL to pipeline definition | [requested](https://twitter.com/_a__w_/status/1102802095474827264)
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

For YAML, it's a `version: 1` keyword at the root level of the file.
YAML v1 may also introduce other breaking changes; those need to be documented elsewhere.

## Variable scope

We'll ensure everything under System.\* is available at orchestration time, including for pipeline run numbers.
Variables under Pipeline.\* will also be available at orchestration time including run numbers.
*TODO: Ensure this is possible.*
Agent.\* variables are never available at orchestration time.
